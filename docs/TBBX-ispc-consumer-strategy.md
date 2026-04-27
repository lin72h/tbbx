# TBBX ISPC Consumer Strategy

## Status

This note captures the current `ISPC` / `ISPCRT` relationship to `TBBX` and
defines the recommended first-consumer strategy.

Date context:

- evaluated against the local `ispc` clone in `../nx/ispc`
- checked against FreeBSD ports in `/usr/ports/devel/ispc` and
  `/usr/ports/devel/onetbb`
- written on April 11, 2026

## Executive Summary

`ISPC` is an important early consumer of `TBB`, but the real integration point
is not the compiler front-end. It is the `ISPCRT` CPU runtime.

The important findings are:

- `ISPCRT` supports three CPU tasking models today: `TBB`, `OpenMP`, and
  `Threads`.
- On non-Windows, non-Apple platforms, `ISPCRT` defaults to `TBB`.
- The current `TBB` coupling is shallow:
  - build-time selection of the `TBB` task model
  - runtime use of `tbb::parallel_for`
- `ISPCRT` also exposes a public callback seam,
  `ispcrtSetTaskingCallbacks()`, that can override its default
  `ISPCLaunch` / `ISPCAlloc` / `ISPCSync` behavior.

That leads to the right strategy:

- treat `ISPCRT` as the first important external `TBBX` consumer
- validate the normal `TBB` path first
- defer any `GCDX` / `TWQ` callback integration until after plain `TBBX`
  consumer validation works

## Core Finding

`ISPC` is not deeply entangled with `TBB`.

`ISPCRT` CPU tasking is the real consumer, and even there the current
dependency is narrow:

- build-time default task model selection in
  `ispcrt/detail/cpu/CMakeLists.txt`
- `TBB` task execution in `ispcrt/detail/cpu/ispc_tasking.cpp`
- public override hook in `ispcrt.h` / `ispcrt.cpp`

This is good news for `TBBX`:

- it makes `ISPC` a realistic first consumer target
- it reduces the amount of oneTBB-specific surface we need to exercise
- it gives a clean fallback if early `TBBX` work is unstable

## Source Facts

### 1. `ISPCRT` supports multiple CPU tasking models

The CPU runtime defines:

- `OpenMP`
- `TBB`
- `Threads`

and defaults to `TBB` on platforms that are neither Windows nor Apple.

Source:

- `ispcrt/detail/cpu/CMakeLists.txt`

Important lines:

- `set(ISPCRT_BUILD_TASK_MODELS "OpenMP;TBB;Threads")`
- default `ISPCRT_BUILD_TASK_MODEL = "TBB"` on non-Windows, non-Apple

Implication:

- on FreeBSD, `ISPCRT` currently wants `TBB` by default
- but `TBB` is not the only supported mode

### 2. The current shipped `TBB` path uses `parallel_for`

The CMake path for `ISPCRT_BUILD_TASK_MODEL=TBB` defines
`ISPC_USE_TBB_PARALLEL_FOR`, not `ISPC_USE_TBB_TASK_GROUP`.

That means the active `TBB` implementation is the `parallel_for` branch.

Source:

- `ispcrt/detail/cpu/CMakeLists.txt`
- `ispcrt/detail/cpu/ispc_tasking.cpp`

Implication:

- the immediate `TBBX` compatibility target is narrow
- this is not a deep `task_arena` / scheduler-internals consumer

### 3. `ISPCRT` exposes a public tasking override seam

`ISPCRT` lets applications override the built-in CPU tasking callbacks via:

- `ispcrtSetTaskingCallbacks()`

and the runtime then prefers those callbacks over the default dynamically
loaded `ISPCLaunch_cpu` / `ISPCAlloc_cpu` / `ISPCSync_cpu` symbols.

Source:

- `ispcrt/ispcrt.h`
- `ispcrt/ispcrt.cpp`

Implication:

- there is a real future path to route `ISPCRT` CPU tasking through a
  platform-native provider
- that path exists independently of `TBB`
- but using it would bypass the plain "`ISPC` as a `TBBX` consumer" story

### 4. `ISPC` still documents generic task-runtime integration

The upstream `ispc` documentation continues to define the real task-runtime
contract in terms of:

- `ISPCAlloc`
- `ISPCLaunch`
- `ISPCSync`

and explicitly says these can interoperate with an existing task system.

Source:

- `docs/ispc.rst`

Implication:

- the callback seam is not accidental
- it is part of the intended `ISPC` runtime model

### 5. Upstream messaging about TBB is stronger than the current source reality

`ReleaseNotes.txt` includes language like:

- `ispcrt does not depend on OpenMP runtime anymore, but requires TBB`

But the current source still clearly supports:

- `OpenMP`
- `Threads`

So the precise reading should be:

- current upstream defaults and packaging strongly prefer `TBB`
- the source tree does not make `TBB` the only viable CPU tasking model

This distinction matters for `TBBX` planning.

## What Makes ISPC A Good First Consumer

`ISPCRT` is a strong first consumer for `TBBX` because it exercises real
external use without immediately forcing the hardest coordination problems.

Why it is attractive:

- it is an important upstream project in the Intel/oneAPI ecosystem
- its default CPU path on FreeBSD/Linux wants `TBB`
- its current `TBB` usage is shallow enough to make bring-up tractable
- it gives us a real compatibility target outside the oneTBB tree

Why it is safer than some later consumers:

- it does not currently require `TCM`
- it does not currently require deep oneTBB internals
- it can fall back to `Threads` if the `TBBX` path is not ready yet

## FreeBSD Packaging Reality

The FreeBSD ports tree already models most of the ordinary downstream
assumptions that matter here.

### `ispc` port

The local FreeBSD `ispc` port:

- depends on `devel/onetbb`
- disables `ISPCRT_BUILD_GPU`
- disables `ISPCRT_BUILD_TESTS`
- disables `ISPC_INCLUDE_EXAMPLES`
- still packages:
  - `libispcrt.so`
  - `libispcrt_device_cpu.so`
  - `ispcrt` CMake package files

This is close to the exact CPU-only bring-up shape we want for early `TBBX`
consumer validation.

### `onetbb` port

The local FreeBSD `onetbb` port already provides the package surface that
`ISPCRT` expects:

- compatibility headers in `include/tbb/`
- `TBBConfig.cmake` under `lib/cmake/TBB`
- imported-target packaging for `find_package(TBB)`
- ordinary shared-library naming through `libtbb.so`

Implication:

- `TBBX` should preserve the ordinary FreeBSD oneTBB consumer surface
- phase-1 `ISPC` validation should not require custom `ISPCRT` patches just to
  find or link the library

## Relationship To PlanC

`ISPC` consumer work is related to `PlanC`, but it is not blocked on `PlanC`.

Important distinction:

- `PlanC` is about upstream/open `TCM` sitting above native FreeBSD machinery
- `ISPCRT` today is mostly just a `TBB` API consumer

Therefore:

- we do not need upstream/open `TCM` source to start validating `ISPC`
  against `TBBX`
- early `ISPC` validation can run against ordinary oneTBB / `TBBX`
  behavior, including `market` fallback
- later, when `PlanC` matures, `ISPC` will benefit indirectly because its
  `TBB` runtime use will sit on top of a stronger coordination substrate

So the dependency direction is:

- `ISPC` helps validate `TBBX`
- `PlanC` later improves multi-runtime coexistence underneath `TBBX`

## Recommended Strategy

### Phase I0: Freeze the consumer reading

Treat these as the stable findings:

- `ISPCRT` CPU runtime is the first important `TBBX` consumer target
- FreeBSD/Linux default tasking model is `TBB`
- current active `TBB` path is `parallel_for`
- `ispcrtSetTaskingCallbacks()` is a future native seam, not the first move

### Phase I1: Validate the plain `TBBX` consumer path first

The first practical goal should be:

- build `ISPCRT` CPU runtime with `ISPCRT_BUILD_TASK_MODEL=TBB`
- point it at `TBBX`
- confirm that CPU task launches and sync work correctly

This is the right first consumer test because it validates:

- external headers / library discovery
- ABI compatibility for the used `TBB` surface
- runtime behavior of the `parallel_for`-based tasking path

Concrete downstream expectation:

- `find_package(TBB COMPONENTS tbb)` works
- imported target `TBB::tbb` exists
- `include/tbb/tbb.h` and `include/tbb/parallel_for.h` are present
- ordinary shared-library discovery still finds `libtbb.so`

### Phase I2: Keep `Threads` as the bring-up fallback

If early `TBBX` bring-up is unstable, `ISPCRT_BUILD_TASK_MODEL=Threads`
remains the escape hatch.

This is useful for:

- separating general `ISPCRT` CPU issues from `TBBX` issues
- keeping a working FreeBSD baseline while TBBX matures

## Practical Validation Targets

### Best small CPU smoke test

The smallest real `ISPCRT` CPU consumer appears to be the `examples/xpu/simple`
lane with `--cpu`.

That is useful because it exercises the end-to-end CPU runtime path through
`ISPCRT`, even though it is not a strong multi-task stress test.

### Better later multi-task checks

The more meaningful task-using kernels are in:

- `examples/xpu/noise`
- `examples/xpu/mandelbrot`
- `examples/xpu/aobench`

But they are not the cleanest first FreeBSD CPU-only bring-up targets because
their surrounding drivers are less focused on a CPU-only validation path.

So the right order is:

1. a small CPU smoke target first
2. a tiny dedicated CPU-only multi-task harness later if needed
3. only then broader example coverage

## Verified Local Check

I re-ran the shortest local validation in this workspace.

### `Threads` control case

This succeeded:

```sh
cmake -S ../nx/ispc/ispcrt -B build/ispcrt-threads \
  -DCMAKE_BUILD_TYPE=Release \
  -DISPCRT_BUILD_CPU=ON \
  -DISPCRT_BUILD_TASKING=ON \
  -DISPCRT_BUILD_TASK_MODEL=Threads \
  -DISPCRT_BUILD_GPU=OFF \
  -DISPCRT_BUILD_TESTS=OFF
cmake --build build/ispcrt-threads -j2
```

Observed result:

- `ispcrt`
- `ispcrt_static`
- `ispcrt_device_cpu`

all built successfully.

### `TBB` lane in this environment

This failed exactly at `TBB` discovery:

```sh
cmake -S ../nx/ispc/ispcrt -B build/ispcrt-tbb \
  -DCMAKE_BUILD_TYPE=Release \
  -DISPCRT_BUILD_CPU=ON \
  -DISPCRT_BUILD_TASKING=ON \
  -DISPCRT_BUILD_TASK_MODEL=TBB \
  -DISPCRT_BUILD_GPU=OFF \
  -DISPCRT_BUILD_TESTS=OFF
```

Observed result:

- configuration reached `detail/cpu/CMakeLists.txt`
- it selected `TBB` tasking as expected
- it stopped on `TBB is not found!`

That supports the current reading:

- ordinary FreeBSD logic is not the blocker here
- package availability and consumer-facing `TBB` packaging are the first
  compatibility issue to solve

### Phase I3: Defer callback-based native integration

Only after the plain `TBBX` consumer path is working should we consider:

- `ispcrtSetTaskingCallbacks()`
- a `GCDX` / `TWQ`-aware tasking provider under `ISPCRT`

That later path may be valuable, but it is a different goal:

- it is not "prove `ISPC` consumes `TBBX`"
- it is "teach `ISPCRT` to consume native platform tasking"

Those are not the same project and should not be mixed in the first pass.

## What To Avoid

Do not start by patching `ISPC` to bypass `TBB`.

That would throw away the value of `ISPC` as an external consumer signal.

Do not start by wiring `ispcrtSetTaskingCallbacks()` into `GCDX`.

That may become interesting later, but it would answer a different question:

- can `ISPCRT` use a native task system?

before answering the simpler and more important one:

- does `TBBX` work as a real downstream `TBB` provider?

Do not overread the release notes as a hard architectural requirement that
`ISPCRT` can only run with `TBB`.

The source still supports alternatives, and that fallback is strategically
useful.

## Recommended Next Steps

1. Treat `ISPCRT` CPU tasking as the first formal `TBBX` downstream consumer.
2. Record the exact `TBB` surface currently exercised:
   - `tbb::parallel_for`
   - `TBB::tbb`
   - associated headers only
3. Plan an early build-and-run matrix:
   - `ISPCRT_BUILD_TASK_MODEL=TBB`
   - `ISPCRT_BUILD_TASK_MODEL=Threads`
   - optional `OpenMP` comparison if useful
4. Use the `Threads` build as the non-`TBB` control case.
5. Defer `ispcrtSetTaskingCallbacks()` exploration until after plain `TBBX`
   consumer validation is complete.

## Concrete Asks For TBBX

For the first `ISPC` milestone, `TBBX` should preserve ordinary consumer
expectations rather than asking `ISPCRT` to bypass `TBB`.

Highest-value near-term work:

1. keep FreeBSD `TBBConfig.cmake` / `TBB::tbb` compatibility airtight
2. keep `include/tbb/` compatibility headers complete enough for:
   - `tbb/tbb.h`
   - `tbb/parallel_for.h`
3. keep ordinary shared-library discovery simple for downstream CMake users
4. avoid packaging or ABI surprises that break a stock `find_package(TBB)`
   consumer
5. optionally add a tiny downstream-consumer CI target that configures
   `ispcrt` with:
   - `ISPCRT_BUILD_TASK_MODEL=TBB`
   - `ISPCRT_BUILD_GPU=OFF`
   - `ISPCRT_BUILD_TESTS=OFF`

## Bottom Line

`ISPC` is the right first important external consumer to focus on, but the
important seam is `ISPCRT` CPU tasking, not the compiler front-end.

The immediate `TBBX` path should be conservative:

- validate `ISPCRT` against `TBBX` as a normal `TBB` consumer first
- keep `Threads` as the control and fallback path
- treat callback-based native tasking as a later exploration, not the first
  milestone

That is the cleanest way to use `ISPC` to validate `TBBX` without mixing the
consumer story with a separate native-tasking redesign.
