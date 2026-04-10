# TBBX PlanA

## Purpose

`TBBX-PlanA` is the local fallback path.

It keeps the upstream oneTBB runtime shape intact and makes oneTBB usable on
this platform by providing a platform-native `libtcm.so.1` implementation when
upstream/open TCM source is unavailable, too delayed, or too awkward to port
cleanly.

This is the right path when the goal is:

- preserve stock oneTBB behavior as much as possible;
- preserve the TCM seam but not depend on upstream timing;
- keep oneTBB modifications minimal;
- retain a controlled local implementation when upstream TCM is not yet
  practical.

It is no longer the preferred first move while waiting for upstream TCM, and it
is no longer the preferred final architecture if PlanC works.

## Core Definition

`PlanA` means:

1. preserve the oneTBB API for applications;
2. preserve oneTBB scheduler and arena internals;
3. preserve the oneTBB `tcm_adaptor` path;
4. replace the missing proprietary TCM runtime with our own
   `libtcm.so.1`;
5. use platform-native mechanism below that broker where the semantics line up.

That makes PlanA a fallback compatibility implementation strategy, not the
primary architectural destination.

In short:

- app contract: oneTBB API
- oneTBB internal seam: unchanged TCM adaptor
- platform work: user-space TCM broker plus kernel-assisted providers

## Why PlanA Exists

PlanA exists because oneTBB already has a clean factory seam:

- [threading_control.cpp](/Users/me/wip-gcd-tbb-fx/nx/oneTBB/src/tbb/threading_control.cpp)
- [tcm_adaptor.cpp](/Users/me/wip-gcd-tbb-fx/nx/oneTBB/src/tbb/tcm_adaptor.cpp)

Today oneTBB does this:

1. initialize the TCM adaptor if possible;
2. try to connect to `libtcm.so.1`;
3. if that succeeds, use the TCM path;
4. otherwise fall back to `market`.

That makes `libtcm.so.1` the lowest-risk local seam when upstream/open TCM is
not yet available as real source.

## Architecture

The `PlanA` stack is:

```text
application
  -> oneTBB API
  -> oneTBB scheduler / arenas / tasking semantics
  -> oneTBB tcm_adaptor
  -> libtcm.so.1 (TBBX implementation)
  -> TCM broker
  -> provider interfaces
  -> platform mechanism
```

The lower part of the stack is intentionally split:

```text
TCM broker
  -> topology provider
  -> capacity provider
  -> pressure provider
  -> kernel / libthr / pthread_workqueue / hmp / TWQ
```

The broker remains user space.

The kernel remains mechanism.

## What PlanA Preserves

PlanA preserves:

- oneTBB API surface
- oneTBB `permit_manager` factory shape
- oneTBB `tcm_adaptor` logic
- oneTBB arena lifecycle
- oneTBB scheduler and work-stealing semantics
- oneTBB fallback to `market`

That fallback is important:

- if `tcmConnect` declines, oneTBB still works through `market`;
- if the platform broker has early bugs, the repo still has a known escape
  hatch.

## What PlanA Builds

PlanA builds a platform-native `libtcm.so.1` that:

- exports the full 13-symbol Intel-compatible surface;
- owns client and permit lifecycle;
- computes permit grants;
- delivers renegotiation callbacks;
- consumes topology, pressure, and later hybrid-capacity signals from platform
  providers.

It is not trying to mimic Intel's hidden implementation exactly.

It is trying to satisfy the public ABI and the semantics oneTBB actually uses.

## PlanA Boundaries

### User-space broker owns

- `tcmConnect` / `tcmDisconnect`
- permit creation, update, deactivate, release
- grant computation
- per-permit state
- callback delivery
- policy over topology / pressure / capacity inputs

### Platform providers own

- topology facts
- NUMA and CPU-mask information
- hybrid capacity and score information when available
- pressure and narrowing signals
- TWQ admission facts

### Kernel / GCDX mechanism owns

- worker admission
- thread state observations
- blocking and wake/park facts
- hybrid hardware telemetry
- HMP / HFI facts

### oneTBB owns

- task scheduling
- arena semantics
- intra-runtime worker allotment inside the runtime

## Phase Breakdown

### Phase 1: Standalone broker

Goals:

- build `libtcm.so.1`;
- export all required symbols;
- get oneTBB to run through the TCM path;
- keep phase 1 standalone and debuggable.

Characteristics:

- no GCDX pressure dependency yet;
- topology via `hwloc2` behind a provider interface;
- flat fair-share grant logic;
- explicit activation gate in `tcmConnect`.

### Phase 1.5: Provider-backed broker

Goals:

- add private pressure and capacity providers below the broker;
- consume TWQ / `pthread_workqueue` pressure;
- later consume FreeBSD native hybrid-capacity data.

Characteristics:

- broker remains user-space;
- providers remain explicit boundaries;
- GCDX influence is informative, not API-collapsing.

### Phase 2: Hardened compatibility path

Goals:

- stress and validation against oneTBB workloads;
- stronger mixed-runtime coexistence with GCDX;
- richer topology-sensitive policy where justified.

This is still compatibility-first, not yet the native `PlanB` architecture.

## Provider Model

PlanA should already enforce the same provider split that later matters for
`PlanB`.

The broker must not reach directly into `hwloc2` or future HMP internals.

Instead, all external data should enter through internal provider interfaces.

Minimal conceptual split:

- topology provider
- capacity provider
- pressure provider

Phase 1 can implement topology with `hwloc2` and leave the other providers as
flat or absent.

The important rule is that the broker policy does not become permanently tied
to one concrete source.

## Advantages

PlanA's strengths are:

- minimal oneTBB modification;
- uses a seam oneTBB already expects;
- easy to validate against upstream behavior;
- keeps `market` fallback alive;
- lets us learn oneTBB's demand and permit cycle safely;
- easier upstream rebases during the early phase.

It is the best fallback and comparison path.

## Costs

PlanA's costs are:

- compatibility baggage;
- 13-symbol exported surface to maintain;
- `tcm_adaptor` indirection;
- callback semantics and reentrancy complexity;
- broker bookkeeping that exists only because the TCM ABI exists;
- an architecture that is excellent for compatibility but not ideal as the
  final platform shape.

So PlanA should be treated as:

- an optional compatibility path;
- a bootstrap and validation path;
- and a reference implementation of the TCM-shaped semantics.

It should not be mistaken for the inevitable final architecture.

## When PlanA Is The Right Choice

PlanA is the right choice when:

- upstream TCM source has not landed yet;
- upstream TCM source lands but does not port cleanly enough;
- the team still wants a local compatibility implementation;
- fallback to `market` is strategically important;
- the team wants a compatibility oracle without depending on upstream timing.

## When PlanA Stops Being Enough

PlanA stops being enough when:

- upstream/open TCM lands and ports cleanly enough that local reimplementation
  becomes redundant;
- the indirection and compatibility baggage becomes more expensive than the
  upstream path;
- the native platform path is clearer than the local compatibility path.

That transition point is exactly where `TBBX-PlanB` starts to dominate.

## Validation Requirements

PlanA should be considered healthy only if:

- oneTBB runs through the TCM path when enabled;
- oneTBB still falls back cleanly to `market` when disabled;
- single-runtime regressions stay acceptable;
- broker callback behavior is deadlock-free;
- mixed GCDX + oneTBB oversubscription is measurably controlled once provider
  integration is added.

## Exit Criteria

PlanA is complete when:

- `libtcm.so.1` exists and is stable on FreeBSD;
- oneTBB is validated through the TCM seam;
- the provider boundaries under the broker are frozen;
- the project understands enough of oneTBB's runtime behavior to either:
  - keep PlanA as a supported compatibility path, or
  - replace it internally with `PlanB`.

## Relationship To PlanB

PlanA is not a dead end.

It is the fallback path that can still teach us:

- permit-manager behavior
- arena demand updates
- callback timing
- grant semantics
- how oneTBB responds to changing concurrency

Those are exactly the inputs needed to build `PlanB` well.

So the intended relationship is:

- PlanC first if upstream/open TCM lands and ports cleanly
- PlanA if upstream/open TCM is delayed or unsuitable
- PlanB only if the TCM seam itself later proves limiting

## Bottom Line

`TBBX-PlanA` remains valuable because it preserves a controlled local route to
the TCM seam if upstream timing or upstream source structure turns out to be a
problem.

But it is a fallback path now: local compatibility, comparison, and safety net,
not the preferred primary strategy.
