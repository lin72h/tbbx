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
- Apple XNU hybrid scheduler study:
  - confirms the static-topology versus dynamic-control-plane split
  - reinforces the provider-boundary rule for future hybrid policy
- high-level TBBX / TCM / GCDX layering
- explicit architecture split:
  - `PlanA` = local fallback reimplementation of the TCM seam
  - `PlanB` = native `permit_manager` contingency path
  - `PlanC` = upstream/open TCM riding on top of native GCDX / TWQ / future
    `hmp` machinery
- concrete native design note for `tbbx_permit_manager`
- comprehensive PlanC design note

What is next:

- freeze `PlanC` as the preferred layered strategy
- design provider SPI work in parallel with waiting for upstream TCM source
- make upstream TCM work on FreeBSD first once source lands
- keep `PlanA` as fallback if upstream TCM is delayed or unsuitable
- keep `PlanB` as the contingency if the TCM seam later proves limiting

## Principles

- Keep oneTBB's scheduler and arena semantics intact above the control seam.
- Keep kernel-assisted mechanism below user-space runtime policy.
- Treat TWQ as mechanism and TCM as coordination policy.
- Prefer `PlanC` if upstream/open TCM lands cleanly.
- Treat `PlanA` as the fallback local implementation of the TCM seam.
- Keep `PlanB` available as the native divergence path if PlanC proves too
  rigid.
- Keep policy behind abstract provider boundaries so no concrete topology or
  capacity source becomes permanent by accident.
- Use `hwloc2` as an acceptable early topology provider, not as a mandatory
  permanent dependency.
- Treat FreeBSD-native HMP/HFI as the preferred future source of hybrid
  capacity and dynamic scores once a usable user-space boundary exists.
- Reuse GCDX / `pthread_workqueue` pressure only through a private lower-layer
  provider boundary.
- Do not put Intel-shaped TCM in the kernel.
- Do not make GCDX depend on TCM.
- Do not let TCM vocabulary leak below the provider line.
- No TCM headers or permit vocabulary inside GCDX / TWQ code.

## Milestones

### M0: Research And Spec Freeze

Status: in progress

Goals:

- freeze the current Intel ISA finding:
  - hybrid-aware through `hwloc`
  - no direct HFI dependency evidenced
- freeze the current FreeBSD-native hybrid strategy:
  - replace `hwloc2` for hybrid ranking later
  - keep `hwloc2` or equivalent topology providers for early phases
- freeze the architecture split:
  - `PlanA` local fallback reimplementation
  - `PlanB` native `permit_manager` contingency
  - `PlanC` layered TCM-over-GCDX strategy
- freeze the provider-abstraction rule:
  - concrete topology/capacity/pressure sources stay behind explicit provider
    boundaries
- freeze the relationship between user-space runtime policy and kernel
  mechanism

Exit criteria:

- findings note, roadmap, and changelog are present
- `PlanA`, `PlanB`, and `PlanC` are all documented with clear boundaries
- the native `tbbx_permit_manager` and `PlanC` design notes exist

### M1: Upstream TCM Source Landing Assessment

Status: next

Goals:

- monitor the oneTBB tree for the actual `thread_composability_manager/`
  source landing
- inspect the real build system, source layout, and dependency surface
- verify whether the real code matches the RFC-level expectations
- identify any immediate FreeBSD blockers
- set an explicit October 2026 reassessment checkpoint if source still has not
  landed

Exit criteria:

- actual source is present and reviewed
- FreeBSD porting surface is understood well enough to estimate Phase C1

### M2: Shared Provider SPI Freeze

Status: planned

Goals:

- design private topology/capacity/pressure provider SPIs below the runtime
- validate those interfaces against existing GCDX machinery while waiting for
  upstream TCM source
- keep pressure integration in `libthr` / `pthread_workqueue`, not GCD queue
  APIs
- settle the event-plus-snapshot shape for future consumers

Exit criteria:

- provider ABI is documented and versioned
- the provider boundary is usable by both PlanC and PlanB

### M3: PlanC Upstream TCM Port On FreeBSD

Status: planned

Goals:

- build upstream/open TCM on FreeBSD
- satisfy `hwloc` / compiler / build assumptions
- ship it as `libtcm.so.1`
- make oneTBB use it through the existing adaptor path

Exit criteria:

- oneTBB can load TCM on FreeBSD
- fallback to `market` still works when TCM is unavailable or disabled

### M4: Standalone Upstream TCM Validation

Status: planned

Goals:

- validate oneTBB behavior through upstream/open TCM
- stress permit lifecycle and callback behavior
- document FreeBSD limitations, especially around `hwloc` `cpukinds`
- establish whether plain upstream TCM is already sufficient for initial use

Exit criteria:

- standalone TCM path is stable enough to trust as a baseline
- limitations are explicit rather than guessed

### M5: PlanC Pressure Provider SPI Integration

Status: planned

Goals:

- expose TWQ / `pthread_workqueue` pressure upward through a private provider
  SPI
- let upstream/open TCM consume those facts without owning worker creation
- measure whether kernel-informed pressure improves TCM grant quality

Exit criteria:

- GCDX remains fully independent underneath
- mixed-runtime oversubscription is measurably reduced

### M6: PlanC Native Hybrid-Capacity Input

Status: planned

Goals:

- add future FreeBSD-native hybrid-capacity providers when a stable user-space
  ABI exists
- reduce dependence on weak `hwloc` `cpukinds` heuristics on FreeBSD
- improve hybrid-core grant quality on asymmetric systems

Exit criteria:

- TCM can consume stronger native hybrid facts on FreeBSD
- policy remains layered and provider-driven

### M7: PlanB Reassessment

Status: planned

Goals:

- decide whether upstream/open TCM plus native platform inputs is sufficient
- identify any concrete reasons to switch to the native `permit_manager` seam
- only revive `PlanB` if PlanC shows real architectural limits
- trigger this review early if upstream TCM source has still not landed by
  October 2026

Exit criteria:

- the project has an explicit keep-PlanC or move-to-PlanB decision

### M8: Broader TBBX Integration

Status: planned

Goals:

- richer topology-sensitive policy where justified
- stronger cross-runtime coordination
- upstreamable oneTBB portability fixes where appropriate

Exit criteria:

- TBBX has a maintained, validated oneTBB story on top of the shared
  substrate

## Near-Term Focus

The immediate focus is M0, M1, and M2 for PlanC.

Specifically:

1. freeze PlanC as the preferred layered strategy
2. monitor upstream TCM source landing closely
3. design and validate the provider boundary in parallel with the upstream wait
4. keep provider boundaries compatible with both PlanC and PlanB
5. avoid premature PlanB implementation before upstream TCM is evaluated
