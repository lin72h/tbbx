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
- source-based review of official open TCM PR #2061:
  - real `thread_composability_manager/` source is present in the PR branch
  - TCM is a standalone `libtcm.so.1` project with `hwloc` dependency
  - the core grant engine is visible and no longer has to be guessed
  - no clean provider seam is obvious yet
  - GCDX pressure is best represented in TCM's permit economy as a private
    adapter-created reserve permit fed by a one-way pressure snapshot and the
    public TCM API
  - v1 reserve integration should be a sidecar adapter, not a TCM source patch
  - zero reserve demand should use `tcmDeactivatePermit`
  - the reserve client needs a non-null no-op callback and
    `rigid_concurrency = 1`
  - `TCM_ENABLE` lifecycle must be handled as an optional feature, not a hard
    dependency
- GCDX M15 pressure-provider coordination review:
  - lower provider surface remains `twq_pressure_provider_*`
  - GCDX/TWQ/libthr stay TCM-blind
  - TBBX consumes the bundle artifact above the provider line
  - v1 reserve-demand projection uses `nonidle_workers_current`
  - future live pressure SPI uses platform-neutral
    `_pthread_workqueue_pressure_snapshot_v1`
  - native provider structs keep GCDX's `size_t struct_size` convention
  - request/block backlog are adapter-derived signals, not reserve size
- concrete native design note for `tbbx_permit_manager`
- comprehensive PlanC design note
- open TCM / GCDX integration plan
- GCDX M15 pressure-provider consumption note
- ISPC / ISPCRT consumer-seam analysis:
  - `ISPCRT` CPU runtime is the real `TBB` consumer
  - current active `TBB` path is narrow and based on `parallel_for`
  - `ispcrtSetTaskingCallbacks()` is the future native override seam
  - FreeBSD ports already model the relevant CPU-only packaging shape

What is next:

- freeze `PlanC` as the preferred layered strategy
- track upstream TCM PR #2061 until merge
- make upstream TCM work on FreeBSD from the PR source first
- design provider SPI work in parallel with PR review and the FreeBSD TCM port
- consume the GCDX M15 pressure-provider bundle shape on the TBBX side
- build a reserve-demand projection harness before modifying TCM internals
- freeze the pressure snapshot field set now, but keep the delivery mechanism
  provisional until the mixed-runtime baseline
- measure TCM-off versus TCM-on/no-adapter mixed-runtime behavior before
  building pressure integration
- include GCD-only and oneTBB-only baselines before judging mixed-runtime
  pressure behavior
- keep `PlanA` as fallback if upstream TCM is delayed or unsuitable
- keep `PlanB` as the contingency if the TCM seam later proves limiting
- treat `ISPC` / `ISPCRT` as the first important downstream `TBBX` consumer
  validation target

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
- Keep downstream consumer validation separate from native-tasking redesign.
- Validate plain `TBBX` consumption before using downstream callback seams to
  bypass `TBB`.

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
- the `ISPC` / `ISPCRT` consumer strategy note exists

### M1: Upstream TCM Source Landing Assessment

Status: in progress

Goals:

- monitor oneTBB PR #2061 until it merges
- inspect the real build system, source layout, and dependency surface
- verify whether the real code matches the RFC-level expectations
- identify any immediate FreeBSD blockers
- identify whether provider injection can be upstreamed cleanly

Exit criteria:

- PR source is reviewed against FreeBSD and GCDX needs
- FreeBSD porting surface is understood well enough to estimate Phase C1
- the provider-seam risk is documented from source, not inferred

### M2: Shared Provider SPI Freeze

Status: planned

Goals:

- design private topology/capacity/pressure provider SPIs below the runtime
- validate those interfaces against existing GCDX machinery while PR review and
  FreeBSD port work proceed
- keep pressure integration in `libthr` / `pthread_workqueue`, not GCD queue
  APIs
- settle the event-plus-snapshot shape for future consumers
- keep pressure snapshots platform-owned and distinguish current gauges from
  cumulative event counters
- keep backlog and pressure-state interpretation above the provider line

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

### M5: Mixed-Runtime Baseline Without Pressure Adapter

Status: planned

Goals:

- measure GCD-only and oneTBB-only resource behavior first
- run mixed GCD + oneTBB workloads before adding pressure integration
- compare TCM disabled against TCM enabled with no pressure adapter
- measure peak thread count, context switches, throughput, oneTBB arena
  concurrency, and GCDX pressure samples
- decide whether TCM alone is already useful or whether GCDX pressure is on
  the critical path

Exit criteria:

- baseline data exists before the pressure adapter is built
- pressure integration is classified as required or optional from measurement

### M6: PlanC Pressure Adapter Integration

Status: planned

Goals:

- expose TWQ / `pthread_workqueue` pressure upward through a private provider
  SPI if M5 shows it is needed
- let a FreeBSD TCM pressure adapter consume those facts without owning worker
  creation
- create a private synthetic reserve permit through the public TCM API to
  represent external TWQ pressure
- package the first adapter as a sidecar such as `libtbbx_twq_bridge.so`
- use a separate TCM reserve client with a non-null no-op callback
- request the reserve permit with `rigid_concurrency = 1`
- consume `twq_pressure_provider_bundle_v1` above the provider line
- size the initial reserve demand from `current_view.nonidle_workers_current`,
  not from cumulative backlog counters
- validate `TCM_ENABLE` and `tcmConnect` failure handling
- use `tcmDeactivatePermit` when reserve demand drops to zero
- call `tcmReleasePermit` before `tcmDisconnect` at sidecar shutdown
- use explicit harness polling as the prototype trigger
- measure raw reserve-demand behavior before adding smoothing
- document polling-only trigger latency and add event-assisted pressure only if
  measurements require it
- measure whether kernel-informed pressure improves TCM grant quality

Exit criteria:

- GCDX remains fully independent underneath
- pressure-adapter runs improve mixed-runtime behavior over the M5 TCM-only
  baseline
- no TCM source changes are required for v1

Stop or defer M6 if the M5 no-TCM baseline already shows no meaningful
oversubscription problem to solve.

Stop or redesign M6 if the production trigger path cannot be implemented
without making libthr, TWQ, or GCDX aware of TCM.

### M7: PlanC Native Hybrid-Capacity Input

Status: planned

Goals:

- add future FreeBSD-native hybrid-capacity providers when a stable user-space
  ABI exists
- reduce dependence on weak `hwloc` `cpukinds` heuristics on FreeBSD
- improve hybrid-core grant quality on asymmetric systems

Exit criteria:

- TCM can consume stronger native hybrid facts on FreeBSD
- policy remains layered and provider-driven

### M8: PlanB Reassessment

Status: planned

Goals:

- decide whether upstream/open TCM plus native platform inputs is sufficient
- identify any concrete reasons to switch to the native `permit_manager` seam
- only revive `PlanB` if PlanC shows real architectural limits
- trigger this review early if upstream TCM has still not merged by October
  2026

Exit criteria:

- the project has an explicit keep-PlanC or move-to-PlanB decision

### M9: Broader TBBX Integration

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
6. use `ISPC` / `ISPCRT` as the first external consumer target for plain
   `TBBX` validation, separate from later callback-based native tasking work
7. preserve ordinary downstream `find_package(TBB)` / `TBB::tbb` compatibility
   so the `ISPCRT` consumer lane does not require custom patches
