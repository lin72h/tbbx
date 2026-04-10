# oneTBB TCM Sidecar Analysis

## Executive Summary

The local oneTBB sources support the handoff's main conclusion: TCM is not a
replacement for TWQ, and TWQ is not a substitute for TCM.

They live at different layers:

- TWQ is a kernel-backed execution-feedback and worker-admission mechanism for
  a specific runtime path.
- TCM is a user-space process-wide coordination broker that lets multiple
  runtimes negotiate how much concurrency each one should use.

For this platform, the cleanest path to oneTBB support is a hybrid
compatibility approach, closest to "Option C" from the handoff:

1. ship a real `libtcm.so.1` compatibility surface so unmodified oneTBB can
   use its existing adaptor;
2. implement only the TCM semantics oneTBB actually depends on first;
3. keep the implementation user-space and process-local;
4. reuse the existing TWQ/libdispatch work as a policy input, not as a direct
   replacement for TCM's broker role.

The key practical point is that oneTBB's current dependency on TCM is narrower
than the full public API. That makes a phase-1 compatibility library realistic.

## Alignment With Main Integration Strategy

This sidecar analysis matches the stronger repo-level framing in
[oneTBB-integration-strategy.md](/Users/me/wip-gcd-tbb-fx/wip-codex54x/oneTBB-integration-strategy.md):

1. oneTBB is a real platform support target, not just an adjacent curiosity;
2. oneTBB should benefit from the platform-native implementation family that
   already exists here;
3. the main TWQ / `libdispatch` lane should not become dependent on hidden TCM
   internals;
4. the right reuse direction is:
   - platform mechanism and policy underneath;
   - oneTBB consuming that through a compatibility-facing service surface above.

That shared conclusion matters because it clarifies the relationship between
the TBB/TCM sidecar work and the main `libdispatch` work:

- the goal is not to make oneTBB ride directly on `libdispatch`;
- the goal is not to make TWQ/libdispatch depend on oneTBB's hidden engine;
- the goal is to let both sit in the same implementation family, with TCM
  acting as a user-space coordination contract and TWQ remaining the
  kernel-backed execution-feedback mechanism.

## The Core Asymmetry

The asymmetry is real, and it should be stated precisely:

- this platform already has the stronger low-level execution mechanism;
- oneTBB already has a sophisticated upper-layer scheduler plus a usable
  coordination seam.

That means the most productive reuse direction is:

1. reuse the platform's TWQ / `libdispatch`-era mechanism and policy work
   underneath;
2. reuse oneTBB/TCM as both:
   - a contract and compatibility target, and
   - a proven upper-layer scheduling model whose internal runtime semantics
     should be respected;
3. let oneTBB benefit from the platform implementation, not the other way
   around.

The phrasing also matters. The platform is not really trying to provide "a TBB
service." The more accurate description is:

1. oneTBB keeps its own scheduler, arenas, and worker model;
2. the platform provides a TCM-compatible coordination service surface;
3. oneTBB consumes that surface to obtain concurrency guidance;
4. that service can in turn reuse platform-native mechanism and pressure
   signals.

### 1. Mechanism asymmetry

The main lane already has real mechanism:

- kernel-backed admission and pressure tracking;
- userland `_pthread_workqueue_*` bridging;
- staged `libdispatch` on the real TWQ path;
- proven backpressure and narrowing behavior.

By contrast, the public oneTBB tree mainly exposes:

- `tcm.h` contract surface;
- `tcm_adaptor.cpp` loader and glue;
- the points where permit grants become arena concurrency.

So there is relatively little production mechanism to pull from oneTBB into the
main TWQ lane.

### 2. Layer asymmetry

TWQ and TCM solve related problems at different layers:

- TWQ is execution-facing mechanism;
- TCM is policy-facing coordination above schedulers.

That is why reuse should flow upward from our mechanism into a compatibility
layer, not downward from the TCM API into kernel design.

### 3. Scheduler-ownership asymmetry

`libdispatch` and oneTBB are not interchangeable runtimes.

`libdispatch` owns a dispatch/workqueue style scheduler.
oneTBB owns arenas, permit-manager logic, and intrusive scheduler invariants.

So the right relationship is not:

- make oneTBB call `libdispatch` directly;
- or make oneTBB "become dispatch."

The right relationship is:

- keep oneTBB's scheduler intact;
- give it concurrency budgets through a TCM-facing broker;
- let that broker be informed by the same platform-native machinery family as
  the dispatch lane.

Explicit authority boundary:

- the broker owns inter-runtime budgets;
- oneTBB owns intra-runtime allotment across its own arenas.

Those are different arbitration levels and should remain separate.

### 4. Validation asymmetry

The main lane is already the stronger validation vehicle for core mechanism:

- dispatch probes;
- TWQ counters;
- Swift integration.

oneTBB support should therefore be treated as:

- an additional compatibility and validation consumer of the platform model,
  not
- the foundation of the platform mechanism itself.

### Practical implication

The asymmetric but coherent design is:

1. main TWQ / `libdispatch` work remains the native kernel-backed mechanism
   lane;
2. a TCM-compatible user-space service sits above it;
3. oneTBB consumes that service with minimal platform-specific change;
4. future multi-runtime coordination can be added without collapsing oneTBB
   into `libdispatch` or redirecting the kernel ABI around TCM.

## Primary Evidence Anchors

These are the highest-value local files and functions behind the analysis:

- `src/tbb/tcm.h`
  - public TCM contract: states, flags, request structure, exported functions
- `src/tbb/tcm_adaptor.cpp`
  - `tcm_link_table`
  - `tcm_client::init`
  - `tcm_client::request_permit`
  - `tcm_client::actualize_permit`
  - `tcm_adaptor::adjust_demand`
  - `renegotiation_callback`
- `src/tbb/threading_control.cpp`
  - `make_permit_manager`
- `src/tbb/market.cpp`
  - fallback permit-manager behavior and internal allotment model
- `src/tbb/arena.cpp`
  - `request_workers`
  - `update_request`
  - `update_concurrency`
- `src/tbb/task_dispatcher.h`
  - thread registration and unregistration around the dispatch loop
- `src/tbb/dynamic_link.cpp`
  - all requested symbols must resolve before the adaptor is considered loaded
- `rfcs/proposed/coordinate_cpu_resources_use/readme.org`
  - TCM as a runtime coordination layer for CPU resource sharing
- `doc/main/tbb_userguide/appendix_B.rst`
  - user-facing description of TCM as an oversubscription-avoidance layer
- unpacked Intel TCM package trees in `../nx`
  - exported symbol shape, bundled dependencies, and platform packaging
- unpacked `hwloc-2.13.0` source plus FreeBSD `devel/hwloc2` port metadata
  - topology, affinity, NUMA, and `cpukinds` support quality on FreeBSD

## Binary And Platform Findings

The local binary and packaging pass materially strengthens the architecture.

### 1. Intel TCM is a cross-platform user-space runtime

The unpacked Intel TCM package trees confirm current binaries for:

- Linux x86_64
- Windows x86_64

Older 1.1.x packages also exist in Intel's package feed for 32-bit Linux and
Windows, but the current 1.2.0 packages we verified are 64-bit.

### 2. Intel TCM uses `hwloc` as its actual topology layer

The current Linux package bundles `libhwloc.so.15`, and the current Windows
package bundles `libhwloc-15.dll`.

The Linux and Windows binaries both reference:

- `hwloc_topology_*`
- `hwloc_bitmap_*`
- `hwloc_get_cpubind`
- `hwloc_cpukinds_get_nr`
- `hwloc_cpukinds_get_info`

That makes `hwloc` part of the real TCM design, not a hypothetical helper.

### 3. Intel TCM does not appear to rely on a Linux-only kernel control API

The Linux binary links against:

- `libhwloc.so.15`
- standard libc / libstdc++ / `libdl` / `libm` / `libgcc`

Its imported/runtime-visible behavior includes:

- pthread locking / TLS
- `sched_yield`
- `hwloc` topology traversal

It does not visibly depend on a Linux-specific runtime-control API comparable to
`pthread_workqueue`.

### 4. FreeBSD `hwloc2` already covers the important phase-1 topology needs

The local `hwloc-2.13.0` source tree and FreeBSD `devel/hwloc2` port confirm:

- CPU affinity support through `cpuset_setaffinity` / `cpuset_getaffinity`
- NUMA discovery through `vm.phys_locality`, `vm.phys_segs`, and `vm.ndomains`
- memory-domain binding through `cpuset_setdomain` / `cpuset_getdomain`
- last-CPU-location queries through `sysctl` and thread identity

The main caveat is `cpukinds` quality:

- the API exists on FreeBSD;
- on x86 it relies largely on CPUID-backed heuristics rather than strong
  OS-native efficiency reporting;
- so heterogeneous-core policy should stay conservative until proven on target
  hardware.

## Strong Source-Level Findings

### 1. oneTBB already treats TCM as a pluggable permit manager

`threading_control_impl::make_permit_manager()` picks `tcm_adaptor` if the TCM
surface loads and connects successfully; otherwise it falls back to the normal
`market` permit manager.

That is the strongest architectural clue in the tree:

- oneTBB does not need to be redesigned around TCM;
- it already has an abstraction seam for external resource governance;
- the natural compatibility point is the existing TCM adaptor, not a oneTBB
  fork.

### 2. oneTBB always tries to load `libtcm.so.1`

At one-time initialization, oneTBB calls `tcm_adaptor::initialize()`, which
uses the runtime dynamic loader to resolve `libtcm.so.1`.

That means a platform `libtcm.so.1` implementation is enough to get oneTBB onto
its TCM path without patching oneTBB itself.

### 3. oneTBB requires the full exported symbol set, even if it does not use all
of it

The dynamic-link table in `tcm_adaptor.cpp` asks for:

- `tcmConnect`
- `tcmDisconnect`
- `tcmRequestPermit`
- `tcmGetPermitData`
- `tcmReleasePermit`
- `tcmIdlePermit`
- `tcmDeactivatePermit`
- `tcmActivatePermit`
- `tcmRegisterThread`
- `tcmUnregisterThread`
- `tcmGetVersionInfo`

`dynamic_link.cpp` resolves every requested symbol before it marks the library
as initialized. So a minimum viable compatibility library must export the full
surface, even if some functions are temporary stubs.

Binary inspection tightens that requirement slightly:

- Intel's current Linux and Windows TCM packages export two additional version
  functions, `tcmRuntimeVersion` and `tcmRuntimeInterfaceVersion`.

oneTBB does not currently resolve them through `tcm_adaptor.cpp`, but a
drop-in platform replacement should still export all 13 symbols shipped by the
current Intel runtime.

### 4. The hot path oneTBB actually depends on is narrower than the API

From the adaptor and surrounding control flow, oneTBB materially depends on:

- `tcmConnect` / `tcmDisconnect`
- `tcmRequestPermit`
- `tcmGetPermitData`
- `tcmDeactivatePermit`
- `tcmReleasePermit`
- `tcmRegisterThread`
- `tcmUnregisterThread`

It does not currently call:

- `tcmIdlePermit`
- `tcmActivatePermit`

It only reads a narrow slice of returned permit data:

- one concurrency value
- permit state
- `flags.stale`

It does not currently consume:

- returned CPU masks
- callback flags
- per-request priority beyond the default initializer value
- `exclusive`
- `rigid_concurrency`

This materially lowers the bar for phase 1.

### 5. oneTBB only distinguishes `INACTIVE` from "not inactive"

`actualize_permit()` converts permit data into arena concurrency and forces the
effective concurrency to zero only when the state is `TCM_PERMIT_STATE_INACTIVE`.

From oneTBB's point of view:

- `INACTIVE` is semantically significant;
- `PENDING`, `IDLE`, and `ACTIVE` are not distinguished in the current path.

That implies a first implementation does not need a sophisticated public state
machine to support oneTBB correctly.

### 6. oneTBB uses TCM as an upper-bound governor, not as its worker scheduler

The adaptor updates arena concurrency. The existing oneTBB worker-dispatch
machinery still runs underneath that permit manager decision.

This is critical:

- TCM does not replace oneTBB's worker scheduler;
- it constrains the concurrency budget that oneTBB should use.

That makes TCM conceptually closer to policy and budget arbitration than to
kernel thread management.

### 7. Thread registration is real, but phase-1 semantics can stay modest

oneTBB calls `register_thread()` and `unregister_thread()` when a thread enters
or leaves the task-dispatch loop for an arena. That gives the broker visibility
into actual worker participation.

However, oneTBB itself only asserts success. A phase-1 broker can use thread
registration as accounting and observability rather than as a hard admission
mechanism.

## Reverse-Engineered TCM Pseudo-Spec For oneTBB

This is not a full reconstruction of Intel's hidden implementation. It is the
minimum source-grounded spec that oneTBB appears to rely on.

### Core model

TCM is best modeled as:

1. a process-local broker;
2. with multiple runtime clients;
3. where each client owns one or more permits;
4. and each permit carries a currently granted concurrency budget plus optional
   placement information;
5. with asynchronous renegotiation when the broker changes a grant.

For oneTBB specifically, the practical first model is:

- one connected client per `tcm_adaptor`;
- multiple permits under that client, typically one per arena;
- permit-level allocation and renegotiation.

### `tcmConnect`

Meaning:

- register a runtime client with the broker;
- provide a callback that TCM can invoke when a permit should be refreshed;
- return a client identifier.

Minimum oneTBB-facing semantics:

- client registration succeeds or fails atomically;
- callback remains valid until disconnect;
- oneTBB can assume the returned client ID is stable for the life of the
  adaptor.

Important phase-1 consequence:

- `tcmConnect` is also the activation gate;
- if it succeeds, oneTBB commits to the TCM path;
- if it fails, oneTBB falls back to `market`.

### `tcmDisconnect`

Meaning:

- unregister the client;
- release all broker-side state associated with that client;
- trigger reallocation for any remaining clients.

### `tcmRequestPermit`

Meaning:

- create or update a permit for a client;
- publish demand as a min/max software-thread request;
- optionally attach topology constraints and policy hints;
- return a stable permit handle.

Minimum oneTBB-facing semantics:

- repeated requests for the same permit must preserve a usable handle;
- requests may change `min_sw_threads`, `max_sw_threads`, and constraints over
  time;
- `request_as_inactive` must be honored at least enough to keep initial arena
  concurrency at zero.

Practical oneTBB nuance:

- the last `tcmRequestPermit(...)` parameter allows immediate permit fill;
- oneTBB currently passes `nullptr` there and fetches grant state later via
  `tcmGetPermitData(...)`;
- phase 1 can therefore treat immediate fill as optional.

### `tcmGetPermitData`

Meaning:

- return the current grant for a permit.

Minimum oneTBB-facing semantics:

- fill `permit->concurrencies[0]` if `size >= 1`;
- fill `permit->state`;
- fill `permit->flags.stale`;
- do not require `cpu_masks` to be non-null.

### `tcmDeactivatePermit`

Meaning:

- keep the permit object but mark it inactive;
- make the client's effective allowed concurrency become zero.

Minimum oneTBB-facing semantics:

- after deactivate, `tcmGetPermitData` must report either:
  - state `INACTIVE`, or
  - state `INACTIVE` plus any numerical concurrency that oneTBB will mask to
    zero anyway.

### `tcmReleasePermit`

Meaning:

- destroy a permit and release any budget it held.

### `tcmRegisterThread` / `tcmUnregisterThread`

Meaning:

- tell the broker that the current thread is actively participating under a
  permit, then later stops participating.

Minimum oneTBB-facing semantics:

- registration should associate the current thread with a permit handle;
- unregistration can rely on thread-local state because it has no handle
  parameter;
- phase 1 can treat these as accounting hooks rather than hard scheduling
  control.

### `tcmIdlePermit` / `tcmActivatePermit`

Meaning:

- richer state transitions on an existing permit.

For oneTBB specifically:

- these symbols must exist;
- a phase-1 compatibility layer can implement them conservatively because
  oneTBB does not currently call them.

### Callback contract

Meaning:

- TCM notifies a client that a permit may have changed;
- the client re-reads the real state through `tcmGetPermitData`.

Minimum oneTBB-facing semantics:

- callback delivery can be coarse;
- callback flags are advisory only for oneTBB;
- if the broker sets `flags.stale`, it should ensure another callback follows.

In practice, a simple broker can avoid setting `stale` at all by serializing
updates cleanly.

## How oneTBB Uses TCM Step By Step

### Startup

1. oneTBB global initialization attempts to load `libtcm.so.1`.
2. if the library and all requested symbols resolve, `threading_control`
   constructs `tcm_adaptor`;
3. otherwise oneTBB falls back to its internal `market`.

### Client and permit creation

1. `tcm_adaptor` calls `tcmConnect(...)`;
2. each arena gets a `tcm_client`;
3. `register_client()` calls `init(...)`;
4. `init(...)` issues an initial `tcmRequestPermit(...)` with:
   - `min_sw_threads = 0`
   - `max_sw_threads = 0`
   - `request_as_inactive = 1`
5. optional CPU constraints are attached only if arena constraints require them
   and an affinity mask is available.

This means the first broker should think in terms of:

- one client connection;
- multiple arena-backed permits beneath it;
- per-permit budgets, not only per-client budgets.

### Demand growth

1. the arena detects work and calls `request_workers(...)`;
2. `threading_control` forwards that to the permit manager;
3. `tcm_adaptor::adjust_demand(...)` updates the arena's min/max request via
   `client.update_request(...)`;
4. if `max_workers() == 0`, it deactivates the permit;
5. otherwise it reissues `tcmRequestPermit(...)` with the new demand;
6. it then immediately calls `actualize_permit()`;
7. `actualize_permit()` reads the grant via `tcmGetPermitData(...)`;
8. the arena's allowed concurrency is updated from the permit.

### Demand shrink

When an arena goes out of work, it reduces mandatory and worker demand through
the same `adjust_demand(...)` path. If the resulting `max_workers()` becomes
zero, the adaptor uses `tcmDeactivatePermit(...)`.

### Asynchronous renegotiation

If TCM changes the grant later, it invokes the callback supplied at connect
time. oneTBB's callback ignores the callback flags and simply calls
`actualize_permit()` again.

### Thread participation

When a thread enters the task-dispatch loop for an arena, oneTBB registers it
with the permit. When it leaves, oneTBB unregisters it.

That means the broker can observe real participation, not only requested
concurrency.

## Minimum Viable TCM Feature Subset For oneTBB On This Platform

The minimum viable subset is smaller than the public API surface.

### Symbols that must exist

All symbols in oneTBB's dynamic-link table must be exported:

- `tcmConnect`
- `tcmDisconnect`
- `tcmRequestPermit`
- `tcmGetPermitData`
- `tcmReleasePermit`
- `tcmIdlePermit`
- `tcmDeactivatePermit`
- `tcmActivatePermit`
- `tcmRegisterThread`
- `tcmUnregisterThread`
- `tcmGetVersionInfo`

### Semantics that appear required for correctness

- stable client registration
- stable permit handles
- request update by min/max concurrency
- explicit inactive state
- grant retrieval through `tcmGetPermitData`
- asynchronous callback path
- thread registration bookkeeping
- cleanup on release and disconnect

### Semantics that appear optional or deferrable in phase 1

- non-trivial use of returned CPU masks
- topology-aware placement enforcement
- meaningful `priority` policy beyond default
- `exclusive`
- `rigid_concurrency`
- distinct `PENDING` and `IDLE` handling for oneTBB
- sophisticated `stale` generation handling
- rich `IdlePermit` / `ActivatePermit` behavior

## TCM Versus TWQ

| Question | TCM | TWQ |
| --- | --- | --- |
| Primary layer | User-space broker | Kernel-backed mechanism |
| Scope | Process-wide, multi-runtime | Runtime-to-kernel control path |
| Main object | Permit / grant | Worker admission / return / narrow |
| Input signal | Runtime demand and thread participation | Blocking, unblocking, yield, return, priority bucket load |
| Output | Allowed concurrency budget | Admit more workers or narrow existing ones |
| Scheduler ownership | Runtime still schedules its own work | Kernel informs worker lifecycle and pressure |
| Placement model | Public CPU masks and topology constraints | Current project mainly QoS bucket admission; placement is future work |
| Cross-runtime coordination | Explicitly yes | Not by itself |
| Hidden Intel policy dependency | Real but partly avoidable for oneTBB | Not relevant |

### Closest conceptual mappings

These are analogies, not equivalences:

- `tcmRequestPermit(min,max)` is analogous to a runtime declaring worker demand.
- a TCM grant is analogous to a concurrency budget.
- the renegotiation callback is analogous to a "re-check demand under new
  pressure" signal.
- `RegisterThread` / `UnregisterThread` are analogous to visibility into active
  worker participation.

### Important non-overlap

These should stay explicit:

- TCM does not replace kernel scheduling hooks.
- TCM does not replace worker admission in the dispatch/TWQ lane.
- TWQ does not solve process-wide arbitration across independent runtimes.
- oneTBB-on-TCM does not imply oneTBB-on-libdispatch.

## What Existing TWQ Work Can And Cannot Be Reused

### What can be reused

The current project already has something valuable that a TCM broker can lean
on:

- a real notion of bounded concurrency under pressure;
- a mature distinction between demand and admission;
- kernel-backed feedback for the dispatch lane;
- real measurements and instrumentation around pressure, narrowing, and worker
  lifecycle.

That work is reusable as **policy input**.

The most realistic reuse is:

1. keep TCM broker logic in user space;
2. let the broker compute grants for TCM-aware runtimes;
3. let the broker optionally consume TWQ/libdispatch pressure as one of the
   inputs that reduces the process-wide budget available to TCM clients.

This is also the cleanest reading of the relation between the sidecar's TBB
work and the main agent's `libdispatch` work:

- `libdispatch` remains one important native execution path whose pressure
  signals can inform the broker;
- oneTBB remains a separate runtime that should consume a coordination service,
  not `libdispatch` APIs directly;
- the shared platform story is underneath both, not by collapsing one runtime
  into the other.

### What cannot be reused directly

The existing TWQ/libdispatch machinery cannot simply be treated as a TCM
implementation because:

- oneTBB still owns its own worker threads and scheduler;
- TWQ today is tied to the `libthr`/`libdispatch` path;
- the current kernel ABI is about thread lifecycle and feedback, not about
  exporting a public multi-runtime broker contract.

So the reuse boundary is real, but it is above the kernel and below the
runtime-specific scheduler.

## Recommendation

Implement a **real `libtcm.so.1` compatibility layer** with a **platform-local
user-space broker**, backed by a **hybrid policy design**:

1. phase 1 gives oneTBB the narrow TCM semantics it already expects;
2. the internal broker remains platform-specific and small;
3. the broker optionally integrates TWQ/libdispatch pressure later without
   changing the public TCM ABI.

This is effectively Option C from the handoff, but with a strong bias toward a
small first implementation.

### Why this is better than a oneTBB-specific patch

- it matches the seam oneTBB already provides;
- it avoids patching oneTBB's threading-control architecture;
- it leaves the door open to other runtimes later;
- it reuses the existing public adaptor rather than creating a new local fork;
- it keeps future cross-runtime coordination possible.

### Why this should stay user-space

TCM is a broker contract, not a kernel ABI problem.

Moving it into the kernel would:

- blur the responsibility boundary between policy and mechanism;
- over-couple a generic coordination service to one runtime path;
- make experimentation much harder than it needs to be.

The kernel should keep doing TWQ things. The broker should sit above that.

## Suggested Phase-1 Broker Design

### Scope

- process-local singleton inside `libtcm.so.1`
- no daemon
- no kernel ABI changes required
- enough semantics for oneTBB first
- `hwloc` as the canonical topology layer

### Internal objects

- client table keyed by `tcm_client_id_t`
- permit table keyed by `tcm_permit_handle_t`
- thread-local current permit binding for `UnregisterThread`
- generation counter or dirty flag for callback scheduling
- shared `hwloc` topology handle for PU counting and later constraint handling

### Phase-1 arbitration model

Use a simple fair-share governor:

1. compute a process-wide CPU budget from available PUs discovered through
   `hwloc`;
2. optionally subtract externally known TWQ/libdispatch pressure later;
3. satisfy active permits' mandatory floor first (`min_sw_threads`);
4. distribute the remainder across requested headroom (`max - min`);
5. clamp each grant to its requested max;
6. notify permits whose grant or state changed.

This is intentionally simpler than hidden Intel policy, but it is already
useful and explainable.

It also mirrors the spirit of oneTBB's own `market` allotment logic instead of
inventing a completely unrelated heuristic.

### Phase-1 state policy

A practical first mapping is:

- `INACTIVE` when explicitly deactivated or requested as inactive
- `ACTIVE` when granted concurrency is non-zero
- `IDLE` optionally when granted concurrency is non-zero but no threads are
  currently registered
- `PENDING` unused initially

Because oneTBB only distinguishes `INACTIVE` from the rest, this is enough to
start.

### Phase-1 CPU constraint policy

Treat placement constraints conservatively:

- accept and store them;
- use `hwloc` as the canonical parser and storage model;
- enforce only what is already easy and correct;
- allow pure concurrency-only behavior when masks are unavailable.

This matches oneTBB's own current behavior, where it only attaches constraints
if affinity information exists.

### Phase-1 callback policy

- synchronous recompute under a broker mutex
- asynchronous callback invocation after state change
- no `stale` flag unless genuinely necessary

That keeps the implementation small and avoids forcing oneTBB into the stale
retry path.

## How To Reuse TWQ Work Without Crossing Layers

The right long-term shape is:

1. TWQ continues to govern dispatch worker admission and pressure;
2. `libtcm.so.1` governs cross-runtime budget arbitration in user space;
3. a small platform hook can let the broker observe dispatch pressure;
4. the broker can then lower oneTBB's grants when dispatch is already consuming
   the CPU budget.

That is the clean complement:

- TWQ remains mechanism for the dispatch lane;
- TCM becomes policy for the multi-runtime lane.

## What Should Not Be Copied From TCM Into This Repo

The handoff asked for explicit negative guidance. The main points are:

- Do not replace `_pthread_workqueue_should_narrow()` with a user-space TCM
  heuristic.
- Do not move TWQ worker admission policy into a generic TCM broker.
- Do not treat TCM as evidence that kernel-backed feedback is unnecessary.
- Do not force oneTBB onto `libdispatch` just because both care about
  concurrency budgets.
- Do not delay oneTBB support waiting for a perfect reconstruction of Intel's
  hidden arbitration logic.

## Risks And Open Questions

### Risk: hidden Intel heuristics are richer than the public surface suggests

True, but oneTBB's actual dependency surface is much narrower than that hidden
engine. A correct-enough broker can still provide meaningful support.

### Risk: process-wide coordination is only partial if dispatch does not report
into the broker

Also true. A phase-1 `libtcm.so.1` can still coordinate TCM-aware runtimes, but
it should not overclaim coordination with non-participating runtimes.

### Risk: default-on behavior may change oneTBB unexpectedly

Because oneTBB always attempts to load TCM, a platform implementation should be
careful about activation policy. Matching the existing "disabled by default"
story from oneTBB documentation is likely safer than silently forcing every
oneTBB process onto the broker path.

### Open question: where should dispatch pressure enter the broker?

The cleanest answer is likely a small user-space-facing provider hook near the
existing `libthr`/TWQ lane, not a kernel-level TCM ABI.

## Bottom Line

The main design answer is now fairly clear:

- oneTBB support should target `libtcm.so.1`, not a local oneTBB rewrite;
- the required phase-1 TCM semantics are narrow enough to implement;
- the implementation should stay user-space and process-local;
- the current TWQ/libdispatch work should be reused as policy input and future
  coordination signal, not as a direct substitute for the TCM broker.

That gives the platform a credible path to oneTBB support while preserving the
clean separation between:

- kernel-backed runtime feedback, and
- user-space multi-runtime coordination.
