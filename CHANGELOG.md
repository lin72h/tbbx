# Changelog

All notable changes to TBBX documentation and planning are recorded here.

## Unreleased

### Added

- [PlanA architecture note](docs/TBBX-PlanA.md) documenting the
  local fallback `libtcm.so.1` path completely.
- [PlanB architecture note](docs/TBBX-PlanB.md) documenting the native
  `permit_manager` contingency path completely.
- [PlanC architecture note](docs/TBBX-PlanC.md) documenting the layered path:
  upstream/open TCM on top of native GCDX / TWQ / future `hmp` machinery.
- [Native permit-manager design note](docs/TBBX-native-permit-manager-design.md)
  documenting the concrete `tbbx_permit_manager` seam and implementation
  blueprint.
- [TBBX terminology note](docs/TBBX-terminology.md) to stabilize the `TBBX` /
  `TCM` / `GCDX` vocabulary.
- [INTEL-ISA note](docs/INTEL-ISA.md) capturing the current ISA-specific
  finding:
  - hybrid-core awareness is evidenced through `hwloc` CPU kinds
  - direct HFI / Thread Director dependence is not currently evidenced
  - direct WAITPKG / UINTR / LKGS / FRED execution is not currently evidenced
- [FreeBSD HMP/HFI strategy note](docs/TBBX-freebsd-hmp-hfi-strategy.md)
  capturing:
  - current FreeBSD review-series scope and status
  - the replacement boundary versus `hwloc2`
  - why native HMP/HFI is a strong future TBBX differentiator
- [TCM platform findings note](docs/TBBX-tcm-platform-findings.md) capturing:
  - Intel TCM package inspection
  - 13-symbol shipped ABI surface
  - `hwloc` dependency and FreeBSD relevance
  - current phase-1 boundary
- [Project roadmap](ROADMAP.md) covering PlanC-preferred sequencing, the
  upstream TCM port path, shared provider SPI work, and later hybrid /
  pressure integration.
- [Changelog](CHANGELOG.md) for ongoing project tracking.

### Changed

- [Roadmap](ROADMAP.md) updated to distinguish:
  - `PlanA` as the local fallback TCM reimplementation path
  - `PlanB` as the native contingency path
  - `PlanC` as the preferred layered path
  - shared provider-boundary rules required by all three
- [PlanC architecture note](docs/TBBX-PlanC.md) added as the comprehensive
  design for layering upstream/open TCM above native GCDX / TWQ / future
  `hmp` machinery.
- [PlanA architecture note](docs/TBBX-PlanA.md) updated to clarify that
  `PlanA` is now the fallback local reimplementation path if upstream/open TCM
  is delayed or unsuitable.
- [PlanB architecture note](docs/TBBX-PlanB.md) updated to clarify that
  `PlanB` is the native contingency path, not the active first move while
  PlanC is being evaluated.
- [Native permit-manager design note](docs/TBBX-native-permit-manager-design.md)
  added as the concrete `PlanB` implementation blueprint.
- [Roadmap](ROADMAP.md) updated again to shift the current project focus from
  immediate PlanB work to PlanC: wait for upstream TCM source, port it on
  FreeBSD, then layer native pressure and capacity inputs underneath it.
- [PlanC architecture note](docs/TBBX-PlanC.md) strengthened to:
  - treat PlanC as the active path and PlanA as its local fallback
  - promote the missing-provider-seam risk to a first-class concern
  - define the provider SPI as a translation layer, not a raw pass-through
  - keep per-QoS details below the bridge
- [PlanC architecture note](docs/TBBX-PlanC.md) updated again to:
  - prefer snapshot-first provider SPI in v1
  - use `struct_size`-first ABI hygiene
  - add the October 2026 reassessment checkpoint
- [Roadmap](ROADMAP.md) updated to move provider SPI work earlier and run it in
  parallel with waiting for upstream TCM source.
- [Immediate TCM integration strategy](docs/oneTBB-tcm-immediate-integration.md)
  updated to make its role as the `PlanA` compatibility path explicit.
- [Long-term strategy](docs/oneTBB-platform-long-term-strategy.md) updated to
  distinguish the immediate TCM compatibility seam from the long-term native
  `permit_manager` seam.
- [Immediate TCM integration strategy](docs/oneTBB-tcm-immediate-integration.md)
  updated to:
  - use `GCDX` terminology
  - distinguish standalone phase 1 from provider-backed phase 1.5
  - record the 13-symbol compatibility target
  - state `hwloc` as the canonical phase-1 topology layer
- [TCM sidecar analysis](docs/oneTBB-tcm-sidecar-analysis.md) updated with:
  - Intel binary and packaging findings
  - `hwloc`-based design implications
  - FreeBSD `hwloc2` capability notes
  - stronger phase-1 broker guidance
- [Long-term strategy](docs/oneTBB-platform-long-term-strategy.md) updated to
  treat topology discovery as shared infrastructure rather than a purely
  GCDX-local concern.
- [TCM platform findings note](docs/TBBX-tcm-platform-findings.md) updated with
  a dedicated Intel ISA finding and link to `INTEL-ISA.md`.
- [INTEL-ISA note](docs/INTEL-ISA.md) expanded with a broader extension scan
  and an explicit distinction between proprietary TCM internals and
  open-source oneTBB WAITPKG usage.
- [Roadmap](ROADMAP.md) updated to treat FreeBSD-native HMP/HFI as the future
  preferred source of hybrid capacity and dynamic scores once a usable
  user-space boundary exists.
- [Immediate TCM integration strategy](docs/oneTBB-tcm-immediate-integration.md)
  and [Roadmap](ROADMAP.md) updated to lock in the provider-abstraction rule:
  broker policy should not depend directly on `hwloc2`, even though `hwloc2`
  remains the initial phase-1 implementation.
- [Terminology note](docs/TBBX-terminology.md) updated to distinguish shared
  topology infrastructure from GCDX-specific mechanism ownership.
