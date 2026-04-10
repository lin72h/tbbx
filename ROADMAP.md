# Roadmap

## Current Status

TBBX is still in architecture-freeze and research consolidation.

What is done:

- oneTBB TCM seam analysis
- Intel TCM package and binary inspection
- Intel ISA / hybrid-core pass:
  - hybrid awareness is evidenced through `hwloc` cpukinds
  - direct HFI / Thread Director dependence is not currently evidenced
  - direct WAITPKG / UINTR / LKGS / FRED execution is not currently evidenced
- `hwloc` and FreeBSD capability review
- FreeBSD HMP / HFI review-series analysis:
  - native replacement candidate for the hybrid-ranking slice
  - not yet a full replacement for `hwloc2` topology
- high-level TBBX / TCM / GCDX layering

What is next:

- freeze the phase-1 TCM spec
- implement a standalone `libtcm.so.1`

## Principles

- Keep oneTBB's scheduler and arena semantics intact.
- Implement TCM as a user-space broker, not a kernel ABI.
- Use `hwloc2` as the canonical topology layer.
- Do not assume Intel HFI / Thread Director is a phase-1 requirement unless
  later evidence proves otherwise.
- Do not assume WAITPKG or other newer Intel-only ISA features are a phase-1
  requirement for TCM unless later evidence proves otherwise.
- Treat FreeBSD-native HMP/HFI as the preferred future source of hybrid
  capacity and dynamic scores, once a usable user-space boundary exists.
- Reuse GCDX / `pthread_workqueue` pressure only through a private lower-layer
  provider boundary.
- Keep phase 1 standalone before adding GCDX-informed budgeting.

## Milestones

### M0: Research And Spec Freeze

Status: in progress

Goals:

- document public ABI and shipped Intel surface
- freeze the current Intel ISA finding:
  - hybrid-aware through `hwloc`
  - no direct HFI dependency evidenced
- freeze the current FreeBSD-native hybrid strategy:
  - replace `hwloc2` for hybrid ranking later
  - keep `hwloc2` for general topology in phase 1
- freeze the 13-symbol export target
- freeze the callback, permit, and activation-gate rules
- freeze the phase-1 / phase-1.5 split

Exit criteria:

- findings note, roadmap, and changelog are present
- TCM phase-1 scope is explicit and stable

### M1: Standalone ABI Skeleton

Status: next

Goals:

- build `libtcm.so.1` on FreeBSD
- export all 13 TCM symbols
- make oneTBB load and connect through the TCM path
- keep explicit fallback to `market`

Exit criteria:

- oneTBB resolves the library successfully when enabled
- oneTBB falls back cleanly when disabled

### M2: Permit Lifecycle And Fair-Share Broker

Status: planned

Goals:

- client table and permit table
- fair-share grant engine
- explicit `INACTIVE` handling
- thread registration accounting
- deferred callback delivery

Exit criteria:

- permit lifecycle works under concurrent arena creation/destruction
- oneTBB test coverage matches the market path closely enough to begin hardening

### M3: Correctness Hardening

Status: planned

Goals:

- deadlock and reentrancy testing
- stale-handle handling policy
- leak and race testing
- activation-gate validation

Exit criteria:

- stress tests run clean
- fallback path remains healthy in CI

### M4: Performance And Topology Hardening

Status: planned

Goals:

- benchmark standalone broker against market fallback
- profile grant operations
- validate `hwloc2` topology quality on target FreeBSD hardware
- document any FreeBSD `cpukinds` limits

Exit criteria:

- no unacceptable single-runtime regression
- topology assumptions are documented, not implicit

### M5: Provider SPI Freeze

Status: planned

Goals:

- design a private GCDX pressure provider SPI below the broker
- keep it in `libthr` / `pthread_workqueue`, not GCD queue APIs
- settle the hybrid event-plus-snapshot shape

Exit criteria:

- provider ABI is documented and versioned
- no layer confusion between TCM budgeting and GCDX admission

### M6: GCDX-Informed Broker

Status: planned

Goals:

- consume GCDX / TWQ pressure snapshots
- reduce or relax TCM grants from real platform pressure
- validate mixed oneTBB + GCDX workloads

Exit criteria:

- mixed-runtime oversubscription is measurably reduced
- broker remains stable and debuggable

### M7: Broader TBBX Integration

Status: planned

Goals:

- richer topology-sensitive policy where justified
- stronger cross-runtime coordination
- upstreamable oneTBB portability fixes where appropriate

Exit criteria:

- TBBX has a maintained, validated oneTBB story on top of the shared substrate

## Near-Term Focus

The immediate focus is M0.

Specifically:

1. keep phase 1 narrow
2. avoid implementation drift into GCDX-dependent behavior too early
3. treat the findings note as the baseline for all early code decisions
