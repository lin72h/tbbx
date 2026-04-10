# TBBX TCM Platform Findings

## Purpose

This note records the current evidence about Intel's shipped TCM runtime, the
public oneTBB seam, and the FreeBSD `hwloc2` capability that matters for TBBX.

It is meant to freeze the research baseline before implementation.

## What Was Inspected

- public oneTBB sources:
  - `src/tbb/tcm.h`
  - `src/tbb/tcm_adaptor.cpp`
  - `src/tbb/threading_control.cpp`
- Intel unpacked TCM package trees:
  - `../nx/tcm-1.2.0-intel_589-linux64`
  - `../nx/tcm-1.2.0-intel_583-win64`
- local `hwloc` sources:
  - `../nx/hwloc-2.13.0`
- local FreeBSD port metadata:
  - `/usr/ports/devel/hwloc2`
- Intel ISA / hybrid-core findings:
  - `docs/INTEL-ISA.md`

## Findings

### 1. The public oneTBB seam is real and sufficient

oneTBB already has a binary seam for TCM:

- on Unix it loads `libtcm.so.1`
- on Windows it loads `tcm.dll`
- if connection fails, oneTBB falls back to `market`

That makes a real `libtcm.so.1` implementation the clean phase-1 integration
target for TBBX.

### 2. Intel ships TCM as a separate binary runtime

Current Intel packages verified locally:

- Linux x86_64:
  - `tcm-1.2.0-intel_589`
- Windows x86_64:
  - `tcm-1.2.0-intel_583`

Older 1.1.x packages also appear in Intel's package feed for 32-bit Linux and
Windows, but the current verified 1.2.0 runtime is 64-bit on both platforms.

No current macOS TCM package was found in the same Intel feed.

### 3. Intel's shipped TCM surface is 13 exported symbols

The public oneTBB adaptor resolves 11 symbols directly:

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

Intel's current binaries export two additional version-query functions:

- `tcmRuntimeVersion`
- `tcmRuntimeInterfaceVersion`

Design consequence:

- phase 1 should export all 13 symbols, not only the 11 oneTBB resolves today

### 4. Intel TCM is a user-space broker, not a Linux-only kernel trick

The Linux `libtcm.so.1.2.0` binary depends on:

- `libhwloc.so.15`
- `libdl`
- `libstdc++`
- `libm`
- `libgcc_s`
- `libc`

Observed imports / strings show:

- pthread locking and TLS
- `sched_yield`
- `hwloc` topology and bitmap APIs
- no visible Linux-specific control plane analogous to `pthread_workqueue`

Design consequence:

- the missing proprietary part looks like a portable user-space coordination
  library, not a hidden Linux scheduler engine

### 5. Intel TCM really uses `hwloc`

Both the Linux and Windows packages bundle `hwloc`:

- Linux: `libhwloc.so.15`
- Windows: `libhwloc-15.dll`

The TCM binaries reference:

- `hwloc_topology_*`
- `hwloc_bitmap_*`
- `hwloc_get_cpubind`
- `hwloc_cpukinds_get_nr`
- `hwloc_cpukinds_get_info`

The bundled Intel `hwloc` appears to be around 2.7.2, which is notably older
than the local FreeBSD port and source tree.

Design consequence:

- `hwloc` should be the canonical topology layer for TBBX TCM
- TBBX does not need a custom topology abstraction before phase 1

### 6. FreeBSD `hwloc2` already covers the important phase-1 needs

The FreeBSD backend in `hwloc` supports:

- CPU affinity binding via `cpuset_setaffinity` / `cpuset_getaffinity`
- thread and process binding queries
- NUMA discovery through `vm.phys_locality`, `vm.phys_segs`, and `vm.ndomains`
- memory-domain binding via `cpuset_setdomain` / `cpuset_getdomain`
- last-CPU-location queries

Design consequence:

- FreeBSD already has enough topology and affinity support for a phase-1 TCM
  broker

### 7. `cpukinds` on FreeBSD should be treated conservatively

`hwloc` exposes the `cpukinds` API on FreeBSD, but native efficiency ranking is
not as strong there as on Windows or some Linux systems.

Practical implication:

- phase 1 should not depend on `cpukinds` for correctness
- heterogeneous-core policy should remain optional and conservative

### 8. oneTBB's immediate semantic needs are still narrow

The public adaptor still appears to rely mainly on:

- client connection
- permit create/update/release
- current granted concurrency
- explicit `INACTIVE` handling
- thread registration accounting
- permit data query

The richer public TCM vocabulary exists, but phase 1 still does not need to
fully reconstruct Intel's hidden internal policy.

### 9. Intel TCM is hybrid-aware, but direct HFI dependence is not evidenced

The inspected Intel TCM and bundled `hwloc` binaries clearly show hybrid-core
awareness through `hwloc` CPU kinds and `CoreType` metadata.

However, the current binary pass did not find direct evidence for:

- `intel_hfi`
- `Thread Director`
- `hardware feedback`
- a special Intel-only HFI control path

Design consequence:

- TBBX phase 1 should treat hybrid awareness as useful and real, but should
  not assume direct HFI or Thread Director support is required

The dedicated ISA note is in:

- `docs/INTEL-ISA.md`

### 10. Newer Intel-only instruction execution is not currently evidenced in
TCM

The current Intel TCM binary pass did not find executed use of:

- WAITPKG (`UMONITOR`, `UMWAIT`, `TPAUSE`)
- UINTR / `UIRET`
- `LKGS`
- FRED instructions

Important nuance:

- some ISA feature-name strings do exist in the Linux `libtcm.so.1.2.0`
  binary, but they appear as part of a much larger Intel runtime CPU-feature
  name table
- that is weaker evidence than actual instruction use

Separate note:

- open-source oneTBB itself does have explicit WAITPKG detection and `_tpause`
  usage in scheduler backoff, but that is not the same thing as proprietary
  TCM internals

## Strategic Consequences

These findings support the current TBBX direction:

1. `libtcm.so.1` remains the right phase-1 seam.
2. TCM should remain a user-space coordination broker.
3. `hwloc2` should be used directly as the topology layer on FreeBSD.
4. Phase 1 should be standalone and fair-share.
5. GCDX / `pthread_workqueue` should feed the broker later through a private
   provider SPI, not by replacing the broker.

## Phase-1 Boundary

Phase 1 should do:

- 13-symbol ABI-compatible TCM surface
- process-local broker
- client and permit lifecycle
- explicit `INACTIVE` support
- fair-share grants from available PUs
- `hwloc` PU counting and conservative constraint capture
- thread registration accounting
- activation gate in `tcmConnect`

Phase 1 should not do:

- TWQ pressure integration yet
- strong placement enforcement
- `cpukinds`-driven grant policy
- deep reconstruction of Intel-only hidden heuristics

## Bottom Line

The proprietary Intel TCM runtime looks replaceable.

The strongest reason is not guesswork anymore:

- the oneTBB seam is public and stable enough;
- the shipped binaries look like a portable `hwloc`-based user-space broker;
- FreeBSD already has the `hwloc2` support needed for a first implementation;
- GCDX can later improve broker decisions through pressure input without
  becoming the broker itself.
