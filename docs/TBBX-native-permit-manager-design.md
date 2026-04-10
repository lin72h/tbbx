# TBBX Native permit_manager Design

## Purpose

This document defines the concrete `PlanB` control boundary: a native
`tbbx_permit_manager` that preserves the TBB API and the oneTBB scheduler
identity above it while replacing the Intel-shaped TCM path underneath.

It is the implementation blueprint for the preferred TBBX architecture.

## Source Seam

The native seam is defined by the existing oneTBB interfaces:

- [permit_manager.h](/Users/me/wip-gcd-tbb-fx/nx/oneTBB/src/tbb/permit_manager.h)
- [pm_client.h](/Users/me/wip-gcd-tbb-fx/nx/oneTBB/src/tbb/pm_client.h)
- [threading_control.cpp](/Users/me/wip-gcd-tbb-fx/nx/oneTBB/src/tbb/threading_control.cpp)
- [market.cpp](/Users/me/wip-gcd-tbb-fx/nx/oneTBB/src/tbb/market.cpp)
- [tcm_adaptor.cpp](/Users/me/wip-gcd-tbb-fx/nx/oneTBB/src/tbb/tcm_adaptor.cpp)
- [arena.cpp](/Users/me/wip-gcd-tbb-fx/nx/oneTBB/src/tbb/arena.cpp)

Those files show the real contract we must satisfy:

1. `threading_control_impl::make_permit_manager(...)` selects a
   `permit_manager` implementation.
2. `threading_control_impl` wires that implementation to the
   `thread_request_serializer`.
3. each `arena` gets one `pm_client`.
4. `adjust_demand(...)` and `set_active_num_workers(...)` are the control
   inputs.
5. arena allotment changes flow through `arena::set_allotment(...)` or
   `arena::update_concurrency(...)`.

## Core Decision

`TBBX-PlanB` treats TCM as internal design guidance, not as a public ABI.

We keep the good parts of the TCM design:

- permit-per-arena accounting;
- explicit inactive versus active grant semantics;
- fair-share arbitration with honest handling of mandatory floor conflicts;
- thread participation accounting;
- provider-driven renegotiation.

We do not keep the TCM ABI baggage:

- no `libtcm.so.1` hot-path dependency;
- no 13-symbol compatibility surface;
- no `dlopen` / `dlsym` boundary;
- no Intel version-query exports;
- no user-space callback ABI shaped like TCM.

## Preserved External Contract

The preserved contract is:

- the TBB API exposed to applications;
- oneTBB scheduler semantics above the `permit_manager` seam;
- arena lifecycle and demand accounting;
- thread-dispatcher interaction through the existing thread-request observer.

The new contract is not:

- binary compatibility with Intel TCM;
- an unchanged oneTBB source tree;
- reuse of Intel's proprietary internal layering.

## Native Stack

The `PlanB` stack is:

```text
application
  -> TBB / oneTBB API
  -> TBBX runtime
  -> tbbx_permit_manager
  -> provider interfaces
  -> kernel-assisted shared substrate
```

The lower half is:

```text
tbbx_permit_manager
  -> topology provider
  -> capacity provider
  -> pressure provider
  -> FreeBSD kernel / libthr / pthread_workqueue / hmp / TWQ
```

## Required permit_manager Behavior

The native implementation must satisfy the five `permit_manager` entry points.

### `create_client(arena&)`

Create one native client object per arena.

The native client should:

- retain a reference to the owning `arena`;
- expose `register_thread()` / `unregister_thread()` hooks;
- store or reference the native permit record for that arena;
- start with zero grant and inactive state.

### `register_client(pm_client*, d1::constraints&)`

Publish the arena into the native manager.

This method should:

- capture initial constraints;
- place the permit into the manager's client tables;
- initialize any provider-facing topology metadata;
- leave the permit inactive until demand becomes nonzero.

### `unregister_and_destroy_client(pm_client&)`

Remove the arena from the manager, retire any pending provider state, and
destroy the native client object safely.

This method must:

- recalculate global grants after removal;
- avoid use-after-free if a provider-triggered recalculation races with
  destruction;
- keep the fallback story deterministic while the native path is maturing.

### `set_active_num_workers(int soft_limit)`

Update the runtime-wide soft limit and trigger grant recomputation.

This is the runtime-visible cap that interacts with the provider-derived
machine budget.

### `adjust_demand(pm_client&, int mandatory_delta, int workers_delta)`

Update the arena request, recalculate grants, and propagate the resulting
aggregate worker delta to the thread-request observer.

This method is the most important control path.

It must:

1. call `pm_client::update_request(...)`;
2. update manager state using the resulting min/max values;
3. recompute grants across all active permits;
4. apply any arena grant changes;
5. notify the thread-request observer of the net runtime-wide change.

## Internal Object Model

The native implementation should be organized around these objects.

### `tbbx_permit_manager`

Process-local coordination object responsible for:

- owning all native permit records;
- holding provider handles;
- tracking the current soft limit;
- recalculating grants;
- applying grant changes to arenas;
- reporting aggregate worker deltas upward.

Phase 1 locking can use one manager mutex.

### `tbbx_pm_client : pm_client`

Per-arena client object.

Responsibilities:

- hold the owning arena reference already provided by `pm_client`;
- track native permit state for that arena;
- implement `register_thread()` / `unregister_thread()` accounting;
- expose helper methods like `apply_grant(unsigned)`.

For `PlanB`, the native permit record may be embedded in this object rather
than stored in a separate broker-only handle table.

### `tbbx_permit`

Conceptual per-arena grant object.

Recommended fields:

- owner pointer to `tbbx_pm_client`;
- priority level;
- min workers;
- max workers;
- current granted workers;
- inactive flag;
- registered thread count;
- copied or normalized constraints;
- provider-facing CPU mask / NUMA / class hints;
- generation or retirement flag for safe destruction.

## Grant Semantics

The native manager should preserve these semantics.

### Unit of allocation

The allocation unit is one permit per arena.

### Inactive versus active

The important semantic distinction is:

- inactive permit: arena must observe zero grant;
- active permit: arena may receive nonzero grant subject to budget.

Inactive should remain an explicit state, not just a synonym for
"temporarily starved."

### Mandatory floors

Mandatory floor conflicts must be handled honestly.

The rule is:

- try to satisfy mandatory floors first;
- if the remaining budget cannot satisfy every floor, do not pretend
  otherwise;
- resolve the conflict with a stable fair policy inside the affected priority
  bucket;
- never exceed the effective budget.

### Priority

`pm_client::priority_level()` and `set_top_priority(...)` are part of the real
oneTBB seam and should be preserved.

Phase 1 native policy should therefore keep market-style priority buckets:

1. highest active priority bucket gets serviced first;
2. lower buckets consume only the remaining budget;
3. `set_top_priority(true)` should be applied to the current highest active
   bucket, consistent with market behavior.

Within a single priority bucket, fair-share policy should be used.

## Recommended Phase-1 Grant Algorithm

Phase-1 native policy should be simple, explicit, and provider-agnostic.

Inputs:

- runtime soft limit from `set_active_num_workers(...)`;
- machine budget from the topology provider;
- current demand from all permits;
- optional flat capacity multiplier from the capacity provider;
- optional reserve from the pressure provider.

Recommended computation:

1. compute `effective_budget` as the minimum of:
   - topology budget;
   - runtime soft limit, except preserve the existing oneTBB mandatory-worker
     escape hatch when the runtime soft limit is zero.
2. subtract any provider-specified reserve or unavailable capacity.
3. iterate permits by priority bucket.
4. inside each bucket:
   - remove permits with `max_workers == 0` from the active allocation set;
   - try to satisfy mandatory floors first;
   - distribute remaining budget proportionally by expandable demand;
   - clamp each permit to its `max_workers`.
5. update each arena grant and aggregate the total delta.

This algorithm should be O(clients), not O(clients^2).

## Arena Update Rule

The native manager should standardize on explicit grant-delta accounting.

Recommended rule:

- use a helper like `apply_grant(new_grant)` on `tbbx_pm_client`;
- implement it with `arena::update_concurrency(...)`;
- accumulate the sum of grant deltas across all affected permits;
- call `notify_thread_request(total_delta)` once per recomputation, outside the
  manager lock.

This is better than treating demand deltas and grant deltas as the same thing.
It also supports provider-driven renegotiation cleanly.

## Thread Accounting

Native thread accounting should be preserved even though there is no TCM ABI.

`tbbx_pm_client::register_thread()` and `unregister_thread()` should:

- update per-permit active-thread accounting;
- optionally maintain thread-local current-permit association;
- provide observability for idle-versus-busy permits;
- remain cheap enough for normal worker paths.

Phase 1 does not need sophisticated idle policy, but it should keep the
accounting hook.

## Provider Interfaces

The native manager must not reach directly into concrete platform sources.

It should consume three provider interfaces.

### Topology provider

Provides:

- total CPU budget;
- CPU masks and affinity facts;
- NUMA shape;
- static class or cluster facts.

Initial implementation may use `hwloc2` or FreeBSD-native sysctl / `cpuset`
glue, but the manager must not depend on one concrete source.

### Capacity provider

Provides:

- per-CPU or per-class capacity values;
- hybrid-performance / efficiency ranking;
- throttling or reduced-capacity flags;
- generation-based updates.

Long-term preferred source:

- native FreeBSD `hmp(4)` / HFI export when a usable user-space boundary exists

### Pressure provider

Provides:

- admitted versus active worker facts;
- pressure or narrowing state;
- runtime reserve guidance;
- generation-based change notifications.

Long-term preferred source:

- GCDX / `pthread_workqueue` / TWQ lower-layer provider

## Concurrency And Recalculation Model

Phase 1 should prefer a simple and defensible model.

Recommended rule:

1. provider callbacks never mutate permit state directly;
2. they mark provider state dirty or enqueue a recalculation request;
3. grant recomputation runs in the manager context under the manager mutex;
4. arena updates and `notify_thread_request(...)` happen after the new grant
   set has been computed;
5. no provider callback or external source is invoked while the manager lock is
   held unless the operation is known to be local and non-reentrant.

This keeps the native path debuggable and avoids importing the TCM callback
deadlock problem into a different form.

## What PlanB Borrows From TCM

Borrow:

- permits as the coordination unit;
- inactive-state discipline;
- fair-share thinking;
- per-runtime coordination as a separate concern from task scheduling;
- thread participation accounting;
- explicit grant recomputation on meaningful state changes.

Do not borrow:

- the C ABI;
- the dynamic-link loading path;
- the Intel export/version surface;
- the assumption that `hwloc` must remain the permanent concrete dependency;
- the assumption that FreeBSD must imitate Intel's packaging story.

## Relationship To PlanA

PlanA and PlanB should share provider concepts and grant vocabulary, but not
the same top-level seam.

PlanA uses:

- oneTBB `tcm_adaptor`
- `libtcm.so.1`
- a user-space TCM broker

PlanB uses:

- a patched `make_permit_manager(...)`
- native `tbbx_permit_manager`
- no mandatory TCM ABI at runtime

PlanA remains useful for:

- compatibility testing;
- reference comparison;
- learning and validating TCM-shaped semantics.

PlanB is the preferred TBBX production shape.

## Phase Breakdown

### Phase B1: Minimal native path

Build:

- `tbbx_permit_manager`
- `tbbx_pm_client`
- flat topology provider
- flat fair-share grant algorithm
- `market` fallback switch in the factory

Do not build yet:

- deep hybrid policy;
- HMP dependence;
- TWQ pressure dependence;
- TCM compatibility surface in the hot path.

### Phase B2: Provider-backed native path

Add:

- explicit topology/capacity/pressure providers;
- provider-triggered recalculation;
- richer observability;
- safer retirement handling for concurrent destruction.

### Phase B3: Shared-substrate optimization

Add:

- native FreeBSD hybrid-capacity provider;
- GCDX-informed pressure reserve;
- stronger multi-runtime coexistence over the shared substrate.

## Validation Requirements

The native path is healthy only if:

- key oneTBB workloads preserve application-visible behavior;
- arena creation and destruction remain race-free;
- aggregate thread-request deltas stay consistent with actual grant changes;
- provider-triggered renegotiation is deadlock-free;
- `market` fallback remains available until confidence is high.

## Bottom Line

The right `PlanB` design is:

- preserve the TBB API;
- use `permit_manager` as the real seam;
- treat TCM as internal design guidance;
- keep kernel-assisted mechanism below provider interfaces;
- keep user-space runtime policy in `tbbx_permit_manager`.

That is the native TBBX architecture that should be designed first and then
implemented.
