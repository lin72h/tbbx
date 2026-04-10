# TBBX PlanB

## Purpose

`TBBX-PlanB` is the native contingency path.

It assumes the public contract that matters is the TBB API to applications,
not the proprietary Intel TCM architecture underneath.

Under that assumption, `PlanB` replaces the Intel-shaped TCM boundary with a
platform-native `permit_manager` implementation while preserving the TBB API
and the oneTBB scheduler identity where it matters.

This is no longer the preferred first move while PlanC is being evaluated, but
it remains the cleanest native divergence path if the TCM seam later proves too
rigid.

## Core Definition

`PlanB` means:

1. preserve the TBB / oneTBB API for applications;
2. preserve oneTBB scheduler and arena semantics above the control boundary;
3. modify oneTBB internals where needed;
4. replace the `tcm_adaptor` / `market` choice with a native
   `tbbx_permit_manager`;
5. let that native permit manager consume platform-native kernel-assisted
   providers directly;
6. treat TCM concepts as internal design guidance, not as a public ABI we
   ship.

In short:

- app contract: preserved
- internal seam: `permit_manager`
- long-term goal: native TBBX runtime over shared platform mechanism

## Why PlanB Exists

The real oneTBB seam is not only `tcm.h`.

The more important seam is the `permit_manager` factory in
[threading_control.cpp](/Users/me/wip-gcd-tbb-fx/nx/oneTBB/src/tbb/threading_control.cpp#L64),
because both `tcm_adaptor` and `market` are just `permit_manager`
implementations.

Relevant sources:

- [permit_manager.h](/Users/me/wip-gcd-tbb-fx/nx/oneTBB/src/tbb/permit_manager.h)
- [pm_client.h](/Users/me/wip-gcd-tbb-fx/nx/oneTBB/src/tbb/pm_client.h)
- [threading_control.cpp](/Users/me/wip-gcd-tbb-fx/nx/oneTBB/src/tbb/threading_control.cpp#L64)
- [market.cpp](/Users/me/wip-gcd-tbb-fx/nx/oneTBB/src/tbb/market.cpp)

If we are willing to patch oneTBB internally, then the compatibility broker is
no longer the only viable seam.

The cleaner seam becomes:

- `permit_manager`

That makes TCM useful as a design reference, but no longer the only available
production boundary.

## Architecture

The `PlanB` stack is:

```text
application
  -> TBB / oneTBB API
  -> TBBX runtime (scheduler / arenas / tasking semantics)
  -> tbbx_permit_manager
  -> provider interfaces
  -> kernel-assisted platform mechanism
```

The lower part becomes:

```text
tbbx_permit_manager
  -> topology provider
  -> capacity provider
  -> pressure provider
  -> kernel / libthr / pthread_workqueue / hmp / TWQ
```

There is no mandatory `libtcm.so.1` in the hot path.

## What PlanB Replaces

PlanB replaces:

- `tcm_adaptor` as the primary control path;
- Intel-shaped permit and callback compatibility as the primary internal model;
- `market` as the only native fallback implementation.

It does not replace:

- the application-facing TBB API;
- the scheduler semantics above the `permit_manager` boundary;
- arenas, tasking, and work-stealing identity unless there is a separate
  reason to do so.

It also replaces the assumption that TCM compatibility must remain the default
internal architecture on FreeBSD.

## The Native Seam

The oneTBB factory today is conceptually:

```text
if TCM works:
    use tcm_adaptor
else:
    use market
```

PlanB changes that to:

```text
if TBBX native mode is enabled:
    use tbbx_permit_manager
else:
    keep a compatibility or market fallback
```

So `PlanB` is not "throw away oneTBB."

It is:

- swap the internal concurrency-governance seam
- while preserving the upper-layer runtime identity

## What tbbx_permit_manager Should Own

The native permit manager should own:

- arena registration and deregistration
- demand tracking per arena
- active worker limit handling
- grant computation
- notification of thread requests upward
- direct arena concurrency updates through the existing permit-manager path

Internally, it should borrow the good parts of the TCM model:

- permit-per-arena accounting;
- explicit inactive versus active grant semantics;
- fair-share grant computation with mandatory floors handled honestly;
- thread participation accounting;
- provider-driven renegotiation.

It should not carry the TCM C ABI, the `dlopen` boundary, or the Intel export
surface.

The control flow is simpler than TCM:

- no `dlopen`
- no 13-symbol compatibility surface
- no `tcmConnect`
- no TCM callback ABI
- no separate broker bookkeeping only needed for compatibility

## What PlanB Should Consume From The Platform

PlanB should consume platform-native mechanism through explicit provider
interfaces.

### Topology provider

Provides:

- CPU count
- NUMA shape
- CPU masks and affinity facts
- static cluster/class information

Phase 1 native implementation can still use `hwloc2` here.

### Capacity provider

Provides:

- hybrid capacity data
- per-CPU or per-class scores
- throttling or effective-capacity signals

Long-term target:

- native FreeBSD `hmp(4)` / HFI-backed provider when a usable user-space
  boundary exists

### Pressure provider

Provides:

- TWQ / `pthread_workqueue` pressure
- admitted vs active worker facts
- narrowing or backpressure state
- generation-based updates

Long-term target:

- GCDX-aware shared concurrency substrate below TBBX

## PlanB Boundaries

### User-space TBBX runtime owns

- oneTBB/TBBX scheduler semantics
- arenas
- task stealing
- runtime-visible concurrency policy
- conversion of provider facts into arena-level grants

### Providers own

- machine facts
- hybrid capacity facts
- runtime pressure facts

### Kernel / shared substrate owns

- worker admission
- pressure tracking
- hybrid telemetry
- scheduling facts the kernel uniquely observes
- future shared runtime registration if ever added

This is the same mechanism/policy split as GCDX, just with a different user
space consumer.

## Advantages

PlanB's strengths are:

- simpler hot path than TCM compatibility;
- no Intel-shaped compatibility baggage in the steady state;
- smaller long-term code surface to maintain;
- more direct use of platform-native providers;
- more elegant long-term architecture if we truly own the TBBX runtime;
- aligns naturally with a shared kernel-backed concurrency substrate.

It remains the cleaner native architecture if internal modification is allowed
and the TCM seam later becomes the constraint rather than the convenience.

## Costs

PlanB's costs are:

- requires oneTBB/TBBX source modifications;
- reduces the "drop-in stock oneTBB" story;
- weakens the automatic fallback to `market` unless we keep it deliberately;
- requires maintaining a localized TBBX fork or patch set;
- raises the bar for rebasing to upstream oneTBB.

These costs are real but localized.

The key source reason they are acceptable is that the seam is small:

- `permit_manager` interface is narrow;
- factory logic is localized;
- `pm_client` contract is small;
- arena update path already exists.

## What PlanB Must Not Do

PlanB must not:

- kernelize Intel-shaped TCM;
- turn `tbbx_permit_manager` into an ad hoc sysctl parser with no abstraction;
- collapse topology, hybrid score, and pressure into one monolithic source;
- replace oneTBB scheduler semantics with GCDX queue semantics;
- force the kernel to own arena-level policy.

The right lesson is still:

- kernel below
- policy above

not:

- push everything downward

## Fallback Story

PlanB should still keep an escape hatch.

Recommended shape:

- native `tbbx_permit_manager` as primary path
- `market` kept as fallback during development and stabilization
- optional continued `PlanA` compatibility path if that remains strategically
  useful

This keeps PlanB from becoming brittle in its early life.

## Phase Breakdown

### Phase B0: Prepare the seam

Goals:

- freeze the `permit_manager`-based architecture;
- isolate provider interfaces;
- keep `market` available as fallback;
- decide what parts of PlanA logic are reusable directly.

### Phase B1: Native permit-manager prototype

Goals:

- add `tbbx_permit_manager`;
- wire the factory to use it in controlled builds;
- implement flat fair-share grants first;
- validate arena lifecycle and demand behavior.

Characteristics:

- topology facts may still come from `hwloc2`;
- no deep hybrid policy required yet;
- no Intel compatibility surface on the hot path.

### Phase B2: Provider-backed native path

Goals:

- integrate explicit topology, capacity, and pressure providers;
- consume GCDX pressure;
- later consume native FreeBSD hybrid-capacity data;
- keep grant computation independent from concrete sources.

### Phase B3: Native shared-substrate optimization

Goals:

- make the TBBX runtime a first-class consumer of the shared kernel-backed
  substrate;
- coordinate better with GCDX without becoming GCDX;
- use platform-native hybrid and pressure data directly in the native runtime
  policy path.

## Relationship To PlanA

PlanA and PlanB are not mutually exclusive in the short term.

The clean project sequencing is:

1. define PlanB as the preferred target;
2. use PlanA only where a compatibility check or reference implementation is
   useful;
3. keep provider boundaries compatible with both paths.

So:

- PlanC is the active layered path
- PlanA is the fallback local TCM path
- PlanB is the contingency if the TCM seam later proves limiting

## When PlanB Is The Right Choice

PlanB is the right choice when:

- we only need to preserve the TBB API for applications;
- internal oneTBB changes are acceptable;
- long-term maintainability matters more than upstream TCM alignment;
- the TCM shared-library singleton model becomes a constraint;
- the team wants one coherent substrate across TBBX and GCDX.

## When PlanB Is Premature

PlanB is premature when:

- we still do not understand oneTBB's demand / concurrency behavior well
  enough;
- upstream/open TCM has not yet been evaluated on FreeBSD;
- the provider and kernel-signal story is still being frozen;
- there is no concrete evidence yet that the TCM seam is the problem.

In that case, PlanA should remain the active path until those risks are lower.

## Validation Requirements

PlanB should be considered healthy only if:

- TBBX API behavior remains compatible for key workloads;
- the native permit manager stays correct under arena churn;
- `market` fallback remains available until confidence is high;
- provider boundaries remain stable and composable;
- mixed TBBX + GCDX workloads improve rather than regress.

## Exit Criteria

PlanB becomes the primary architecture when:

- the native permit-manager path is stable;
- its provider interfaces are frozen;
- it clearly reduces long-term complexity versus the compatibility broker;
- application-facing behavior is preserved;
- the team no longer needs `libtcm.so.1` as the default internal seam.

## Bottom Line

`TBBX-PlanB` is the right escape hatch once we only need to preserve the TBB
API for applications and we have concrete evidence that the TCM seam is holding
the platform back.

It treats `permit_manager`, not TCM, as the real long-term seam.

That makes PlanB the native contingency path, with PlanC tried first and PlanA
kept as the local fallback if upstream TCM itself is unavailable or unsuitable.
