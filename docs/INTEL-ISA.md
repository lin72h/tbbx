# INTEL-ISA

## Purpose

This note records the current reverse-engineering findings about Intel-specific
ISA and platform-feature usage in Intel's proprietary TCM runtime.

The key question is narrow:

- does Intel TCM depend on Intel-only hybrid-core and hardware-feedback
  features in a way that TBBX must reproduce for phase 1?

This note separates hard evidence from inference.

## What Was Inspected

- Linux Intel TCM binary:
  - `../nx/tcm-1.2.0-intel_589-linux64/lib/libtcm.so.1.2.0`
- Linux bundled `hwloc`:
  - `../nx/tcm-1.2.0-intel_589-linux64/lib/libhwloc.so.15.5.4`
- Windows Intel TCM binary:
  - `../nx/tcm-1.2.0-intel_583-win64/Library/bin/tcm.dll`
- Windows bundled `hwloc`:
  - `../nx/tcm-1.2.0-intel_583-win64/Library/bin/libhwloc-15.dll`
- local `hwloc` source tree:
  - `../nx/hwloc-2.13.0`

Inspection methods used:

- exported symbol inspection
- import table inspection
- binary string scanning
- disassembly spot checks around CPU-feature probing
- source correlation against local `hwloc` cpukinds support

## Hard Findings

### 1. Intel TCM is hybrid-core aware

This is well-supported.

The Intel TCM binaries import `hwloc` CPU-kind APIs:

- `hwloc_cpukinds_get_nr`
- `hwloc_cpukinds_get_info`

The bundled Intel `hwloc` binaries also contain explicit hybrid-core
vocabulary:

- `CoreType`
- `IntelAtom`
- `IntelCore`
- `HWLOC_CPUKINDS_RANKING`
- `coretype+frequency`
- `forced_efficiency`

That means Intel TCM is not treating the machine as a flat homogeneous CPU set.
It has access to a hybrid-core classification model through `hwloc`.

### 2. There is no current evidence of a direct HFI / Thread Director dependency

This is the most important negative finding.

Across the inspected Intel TCM and bundled `hwloc` binaries, no direct evidence
was found for:

- `intel_hfi`
- `Thread Director`
- `hardware feedback`
- `/sys/devices/system/cpu/intel_hfi`
- Windows CPU-set efficiency APIs such as `GetSystemCpuSetInformation`

So the current evidence does not support the claim that Intel TCM depends on a
special direct Thread Director / HFI integration path.

### 3. There is no current evidence that Intel TCM executes newer Intel-only
instruction extensions such as WAITPKG, UINTR, LKGS, or FRED

The binaries were checked for actual instruction usage and related feature
terms.

No executed instruction use was found for:

- `UMONITOR`
- `UMWAIT`
- `TPAUSE`
- `UINTR`
- `UIRET`
- `SENDUIPI`
- `LKGS`
- `FRED`
- `ERETU`
- `ERETS`
- `RDMSR`
- `WRMSR`

Some feature-name strings do appear in the Linux `libtcm.so.1.2.0` binary,
including:

- `WAITPKG`
- `UINTR`
- `HRESET`
- `WRMSRNS`

But those strings appear inside a much larger CPU-feature name table alongside
many unrelated ISA extensions. That pattern is consistent with Intel runtime
CPU-feature enumeration support code, not with proof that TCM executes those
instructions.

So the current result is:

- feature names exist in the binary;
- actual execution of those instructions is not evidenced.

### 4. The visible CPU probing is compatible with normal topology detection and
generic Intel runtime dispatch

`libtcm.so.1.2.0` contains `cpuid` instructions and Intel CPU-feature probing,
but that alone is not evidence of a TCM-specific hardware-feedback path.

What is visible is consistent with:

- Intel runtime CPU dispatch
- cache sizing and feature detection
- ordinary x86 topology and capability probing

The stronger hybrid-core signal still comes from the `hwloc` cpukinds path, not
from a unique TCM-only ISA interface.

### 5. The bundled `hwloc` model aligns with public `hwloc` hybrid support

The local `hwloc-2.13.0` source tree shows that hybrid-core classification is a
normal part of the public `hwloc` model:

- `include/hwloc/cpukinds.h`
- `hwloc/topology-linux.c`
- `tests/hwloc/x86/Intel-CPUID.1A-1p2co2t.output`

That model uses public signals such as:

- x86 CPUID hybrid/core-type information
- frequency data
- Linux capacity heuristics where available

This reinforces the conclusion that Intel TCM's visible hybrid awareness is
likely grounded in `hwloc` and ordinary platform topology inputs, not in a
private mandatory HFI dependency.

### 6. WAITPKG is visibly used in open-source oneTBB, but that is separate from
proprietary TCM

This distinction matters.

The local oneTBB sources explicitly detect WAITPKG and HYBRID support:

- `src/tbb/misc.cpp`

And oneTBB's scheduler backoff uses `_tpause` when WAITPKG is available:

- `src/tbb/scheduler_common.h`

So WAITPKG is a real part of the broader TBB runtime story.

But that is upstream oneTBB scheduler behavior, not evidence that Intel's
proprietary TCM broker depends on WAITPKG internally.

## Best Current Interpretation

The current evidence supports this reading:

1. Intel TCM is hybrid-aware.
2. That awareness is exposed through `hwloc` CPU kinds and related topology
   metadata.
3. Intel TCM does not currently show direct executed use of newer Intel-only
   instructions such as WAITPKG, UINTR, LKGS, or FRED.
4. Intel TCM may benefit indirectly from OS-exposed platform information on
   some systems.
5. But there is no current evidence that direct HFI / Thread Director support
   is a required secret dependency for TCM itself.

This is an inference, not a proof of absence. It means:

- direct HFI use is not evidenced by the binaries we inspected;
- direct WAITPKG / UINTR / LKGS / FRED execution is not evidenced in the TCM
  binaries we inspected;
- hybrid-core awareness is evidenced clearly;
- the safest design assumption for TBBX is to treat HFI as optional and
  non-blocking until contrary evidence appears.

## TBBX Design Consequences

### Phase 1

Phase 1 should not depend on:

- Intel Thread Director
- HFI
- WAITPKG
- UINTR
- LKGS
- FRED
- any Intel-only kernel or firmware feedback path

Phase 1 should depend on:

- public TCM ABI compatibility
- `hwloc` topology
- conservative CPU-kind support where available

That is enough for a credible TCM replacement.

### Later phases

If future evidence shows useful HFI-like signals on supported platforms, they
should be treated as:

- optional lower-layer hints
- below the TCM broker
- never as a requirement of the TCM ABI itself

For TBBX, that means such signals would belong with substrate-informed policy
in later phases, not with the phase-1 compatibility surface.

## Bottom Line

The current reverse-engineering result is:

- Intel TCM is clearly hybrid-core aware.
- Intel TCM is not currently evidenced as directly HFI / Thread Director
  dependent.
- Intel TCM is not currently evidenced as executing WAITPKG, UINTR, LKGS, or
  FRED instructions internally.

That is a very favorable result for TBBX because it means:

- the visible "magic" is mostly topology-aware software policy;
- Intel-only ISA feedback is not currently a proven blocker;
- `hwloc` remains the right public foundation for phase 1.
