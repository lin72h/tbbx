# Changelog

All notable changes to TBBX documentation and planning are recorded here.

## Unreleased

### Added

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
- [Project roadmap](ROADMAP.md) covering research freeze, standalone broker,
  provider SPI, and GCDX-informed phases.
- [Changelog](CHANGELOG.md) for ongoing project tracking.

### Changed

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
- [Terminology note](docs/TBBX-terminology.md) updated to distinguish shared
  topology infrastructure from GCDX-specific mechanism ownership.
