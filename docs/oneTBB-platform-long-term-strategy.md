# oneTBB Platform Long-Term Strategy

## Purpose

This document describes the long-term strategy for supporting oneTBB on this
platform without carrying a second unrelated concurrency stack next to the
existing TWQ / `pthread_workqueue` / `libdispatch` work.

The target is not "replace oneTBB with `libdispatch`."

The target is:

1. preserve the oneTBB API and the parts of the oneTBB runtime that are
   genuinely oneTBB-specific;
2. gradually substitute duplicated low-level runtime machinery with a shared
   platform-native substrate where the semantics line up;
3. let oneTBB and `libdispatch` benefit from the same underlying effort rather
   than building two separate worker-admission and pressure systems.

One important correction of emphasis:

- oneTBB is not merely a source of "interface clues";
- it is also a sophisticated upper-layer runtime with real scheduler policy,
  arena semantics, and intra-runtime worker allotment logic.

The asymmetry is therefore more precise when stated this way:

- the platform has the stronger low-level concurrency substrate;
- oneTBB has the stronger existing upper-layer scheduling model and a clean
  coordination seam.

A shorter way to say it is:

- GCDX has the stronger execution mechanism;
- oneTBB has the stronger coordination interface;
- TCM is where they meet.

One important refinement now that `PlanB` exists:

- TCM is the immediate compatibility seam;
- `permit_manager` is the preferred long-term native seam for TBBX.

## Core Position

The platform should think in terms of **shared substrate plus runtime-specific
upper layers**.

That means:

- shared low-level mechanism should be implemented once;
- oneTBB-specific scheduler behavior should remain oneTBB-specific;
- `libdispatch`-specific scheduler behavior should remain `libdispatch`-specific.

So the long-term strategy is a **gradual internal substitution strategy**:

1. keep the public oneTBB API;
2. keep oneTBB task-scheduler semantics where they matter;
3. replace overlapping lower-level concurrency-governance machinery with
   platform-native implementations over time.

## The Shared-Substrate Model

The architectural split should look like this.

### Shared platform substrate

This is the part that should be reused across runtimes:

- kernel-backed worker admission;
- pressure and backpressure accounting;
- block/yield/recent-stall compensation;
- worker park/wake lifecycle management;
- CPU-capacity and topology accounting;
- observability and validation harnesses;
- process-wide coordination inputs.

This substrate is where the existing TWQ / `libthr` / staged `libdispatch`
effort is already strongest.

One nuance from the TCM binary and `hwloc` research:

- topology discovery should be treated as shared infrastructure, not as a
  GCDX-private abstraction;
- `hwloc` is the most plausible canonical cross-runtime topology layer for
  TBBX, because Intel's own TCM uses it and FreeBSD already has a working
  backend through `hwloc2`.

### oneTBB-specific upper layer

This is the part that should remain oneTBB-shaped:

- task scheduler behavior;
- arena semantics;
- permit-manager-facing concurrency decisions above the shared substrate;
- work stealing and scheduler invariants;
- task isolation, cancellation, and context semantics;
- flow graph and algorithm-facing behavior.

This upper layer includes real runtime intelligence, not just API surface.
TBBX should treat it as a consumer of lower-level budgets and signals, not as
an empty shell to be replaced.

### `libdispatch`-specific upper layer

This is the part that should remain dispatch-shaped:

- queue semantics;
- dispatch priorities and execution model;
- dispatch source / timer / continuation behavior;
- Swift-facing executor interaction.

## What "Gradually Swapping Parts Of TBB" Should Mean

The phrase "swap part of TBB into our libdispatch-optimized counterpart" is
directionally right but should be made precise.

The platform should **not** swap oneTBB into a dispatch runtime.

The platform **should** gradually swap oneTBB's duplicated low-level runtime
machinery into shared platform-native counterparts where the overlap is real.

In practical terms, the replacement candidates are:

1. concurrency budgeting and permit arbitration;
2. worker-admission policy inputs;
3. thread lifecycle and blocking-compensation support;
4. topology/capacity accounting;
5. sleep/wake and pressure observability;
6. multi-runtime coordination support.

The non-candidates are:

1. the oneTBB public API;
2. arena and scheduler semantics;
3. the oneTBB tasking model;
4. oneTBB-specific algorithm behavior.

## Progressive Replacement Roadmap

The clean way to do this is in stages.

### Stage 0: Optional compatibility-first oneTBB support

Goal:

- support oneTBB with minimal disturbance when a compatibility lane is useful.

Approach:

- provide a `libtcm.so.1`-compatible service surface;
- let unmodified or minimally modified oneTBB use its existing TCM adaptor;
- keep most oneTBB internals intact.

This is the lowest-risk entry point, but no longer the preferred destination.

### Stage 1: Native permit-manager seam

Goal:

- make `permit_manager` the native TBBX control boundary.

Approach:

- replace the TCM adaptor as the primary internal path with a native
  `tbbx_permit_manager`;
- preserve the TBB API and oneTBB scheduler identity above that seam;
- keep platform topology, capacity, and pressure behind provider interfaces.

At this stage, oneTBB still keeps its own scheduler, but TBBX stops treating
TCM compatibility as the permanent architecture.

Critical invariant:

- the shared substrate owns low-level mechanism;
- the native permit manager owns runtime-visible budgets;
- oneTBB continues to own intra-runtime allotment across its arenas.

Those are different decisions and should not be collapsed into one layer.

### Stage 2: Shared worker-provisioning support where semantics align

Goal:

- share more of the expensive runtime plumbing without flattening runtime
  identity.

Approach:

- reuse platform-native worker lifecycle machinery;
- reuse common sleep/wake and blocking-compensation infrastructure;
- reuse common topology and CPU-capacity views.

This does **not** mean "dispatch workers run TBB tasks" by default.
It means the lower-level worker-management substrate becomes common where
possible.

Stopping rule:

- replace mechanism;
- do not replace oneTBB scheduler policy.

If a proposed replacement starts to absorb market-style arena allotment,
priority policy, or scheduler invariants, it has crossed the boundary and
should stop.

### Stage 3: oneTBB-specific optimization on top of the shared substrate

Goal:

- exceed mere compatibility.

Approach:

- add TBB-specific optimizations that exploit the shared platform substrate;
- improve arena budgeting, topology-sensitive grants, and interaction with
  other runtimes;
- preserve oneTBB-visible semantics while improving platform performance.

This is where the effort becomes more than a compatibility layer.

### Stage 4: Productized platform oneTBB story

Goal:

- treat oneTBB as a first-class supported runtime on the platform.

Approach:

- ship a maintained broker and runtime integration path;
- add oneTBB workloads to the platform validation lane;
- keep behavior regressions visible through the same observability story used
  by the main TWQ / `libdispatch` effort.

## What The Shared Substrate Should Reuse From The Current Main Effort

The current main effort already has valuable low-level assets.

### 1. Kernel-backed admission and narrowing

These are already the strongest mechanism pieces in the repo.

oneTBB support should reuse:

- admission thinking;
- pressure accounting;
- narrowing/backpressure philosophy;
- worker lifecycle signals.

It should not build another unrelated policy engine if the same low-level
problem is already solved here.

### 2. User-space bridge and integration discipline

The custom `libthr` and staged-runtime integration work already show how to
connect a user-space runtime to the TWQ kernel substrate.

That integration discipline should be reused for oneTBB support, even when the
oneTBB seam is TCM rather than direct `_pthread_workqueue_*`.

### 3. Observability and validation

The main lane already has:

- TWQ counters;
- dispatch probes;
- Swift integration;
- controlled runtime matrices.

oneTBB support should reuse the same engineering style:

- small probes;
- pressure-sensitive validation;
- no silent fallback;
- explicit proof that the platform-native path is active.

## What Should Remain Separate

Elegance here depends on keeping the right things separate.

### oneTBB should not become dispatch

Do not:

- map oneTBB tasks directly onto dispatch queues as the primary design;
- force arena semantics into queue-width semantics;
- assume dispatch priorities and oneTBB scheduler policy are interchangeable.

### oneTBB market-style policy should remain oneTBB-owned

Do not:

- move arena allotment policy into the shared substrate;
- let the broker make per-arena scheduling decisions;
- let dispatch QoS vocabulary become the internal policy language of oneTBB.

The broker may cap oneTBB globally, but oneTBB should still decide how to
spend that budget within its own runtime.

### The main TWQ lane should not depend on hidden TCM internals

Do not:

- redirect kernel ABI design around a hidden Intel engine;
- treat the public TCM contract as if it dictated kernel structure;
- block TWQ progress waiting for deeper TCM reverse engineering.

### Shared substrate should remain below runtime identity

The rule is:

- share mechanism below;
- preserve runtime semantics above.

## Long-Term Design Rule

For every prospective oneTBB integration feature, ask:

1. is this shared low-level mechanism?
2. or is this oneTBB-specific scheduler behavior?

If it is shared mechanism:

- build it once in the platform substrate.

If it is oneTBB-specific behavior:

- keep it in the oneTBB-facing layer.

That is the rule that keeps the result elegant instead of collapsing into
either duplication or forced unification.

## Long-Term Bottom Line

The long-term strategy is:

1. preserve the oneTBB API and runtime identity;
2. use TCM compatibility as the first support seam;
3. gradually replace duplicated lower-level machinery with shared
   platform-native implementations;
4. treat the existing TWQ / `libdispatch` work as the foundation of that
   shared substrate;
5. only add oneTBB-specific lower-level code where the shared substrate does
   not cover a real TBB requirement.

That gives the platform:

- a first-class oneTBB story;
- one coherent low-level concurrency substrate;
- less duplicated engineering effort;
- and a cleaner path to optimization than either "keep two stacks forever" or
  "turn oneTBB into dispatch."
