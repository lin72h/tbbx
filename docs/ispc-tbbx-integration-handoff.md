# ISPC TBBX Integration Handoff

## Purpose

This note is for a dedicated sidecar agent that will focus on `ISPC` as the
first important downstream consumer of `TBBX`.

The goal is not to re-open the whole `PlanA` / `PlanB` / `PlanC` strategy. The
goal is to get productive quickly on the concrete consumer problem:

1. determine exactly how `ISPC` / `ISPCRT` consumes `TBB` today;
2. identify the smallest real `TBBX` compatibility surface needed for `ISPC`;
3. separate plain "`ISPCRT` uses `TBBX`" validation from later native-tasking
   redesign ideas;
4. produce a concrete onboarding path for making `ISPC` a real FreeBSD-side
   validation target for `TBBX`.

This note is intentionally scoped so the sidecar agent can start from the
current repo state without reconstructing prior research from scratch.

## Explicit Mission

The sidecar agent should treat this as a downstream-consumer integration task.

The mission is:

1. map the exact `TBB` API surface that `ISPCRT` CPU tasking uses today;
2. confirm whether `ISPC`'s current FreeBSD/Linux path truly needs `TBB`, or
   only defaults to it;
3. determine the most realistic first build-and-run path for `ISPCRT` against
   `TBBX`;
4. identify the minimum test matrix that distinguishes:
   - `ISPCRT` general CPU-tasking issues
   - `TBBX` compatibility issues
5. evaluate the public tasking-callback seam as a later native-integration
   direction, but do not let that replace the plain consumer-validation lane.

## What Is Already Settled

Treat these as current working conclusions unless local source inspection
proves otherwise:

1. The real `ISPC` consumer seam is `ISPCRT` CPU tasking, not the compiler
   front-end.
2. `ISPCRT` supports three CPU tasking models:
   - `TBB`
   - `OpenMP`
   - `Threads`
3. On non-Windows, non-Apple platforms, `ISPCRT` defaults to `TBB`.
4. The current active `TBB` implementation is the `parallel_for` path, not a
   deeper task-arena or scheduler-internals path.
5. `ISPCRT` exposes a public override seam:
   `ispcrtSetTaskingCallbacks()`.
6. That callback seam is important, but it is a later native-tasking
   exploration, not the first consumer milestone.
7. `ISPC` consumer work is not blocked on upstream/open `TCM`.
8. The first useful question is:
   does `ISPCRT` build and run correctly against `TBBX` as an ordinary
   downstream `TBB` consumer?

## Separation From The Main Lane

The main repo's current strategy is still centered on:

1. `PlanC` as the active path for oneTBB resource coordination:
   upstream/open `TCM` above native FreeBSD machinery;
2. early provider SPI design while waiting for upstream TCM source;
3. `PlanA` as fallback if upstream/open `TCM` does not land or does not fit;
4. `PlanB` as the native contingency path if the TCM seam later proves too
   limiting.

The `ISPC` consumer task is adjacent, not a replacement.

The sidecar agent should not assume:

1. `ISPC` validation needs upstream/open `TCM` first;
2. `ISPCRT` should be rewritten around `GCDX` immediately;
3. callback-based native tasking is automatically better than plain `TBB`
   consumption;
4. proving a native `ISPCRT` tasking path is the same as proving `TBBX`
   compatibility.

The useful outcome is a clean separation:

1. plain downstream `TBBX` consumer validation first;
2. optional native tasking exploration later.

## Current Repo Context

Start with these local project notes:

1. [TBBX-ispc-consumer-strategy.md](/Users/me/wip-gcd-tbb-fx/wip-tbb-gpt54x/docs/TBBX-ispc-consumer-strategy.md)
2. [TBBX-PlanC.md](/Users/me/wip-gcd-tbb-fx/wip-tbb-gpt54x/docs/TBBX-PlanC.md)
3. [ROADMAP.md](/Users/me/wip-gcd-tbb-fx/wip-tbb-gpt54x/ROADMAP.md)
4. [CHANGELOG.md](/Users/me/wip-gcd-tbb-fx/wip-tbb-gpt54x/CHANGELOG.md)
5. [oneTBB-platform-long-term-strategy.md](/Users/me/wip-gcd-tbb-fx/wip-tbb-gpt54x/docs/oneTBB-platform-long-term-strategy.md)

Also check the local FreeBSD packaging reality:

6. [/usr/ports/devel/ispc/Makefile](/usr/ports/devel/ispc/Makefile)
7. [/usr/ports/devel/ispc/pkg-plist](/usr/ports/devel/ispc/pkg-plist)
8. [/usr/ports/devel/onetbb/Makefile](/usr/ports/devel/onetbb/Makefile)
9. [/usr/ports/devel/onetbb/pkg-plist](/usr/ports/devel/onetbb/pkg-plist)

Very short current summary:

1. `PlanC` remains the active TCM strategy while waiting for upstream source.
2. `ISPC` / `ISPCRT` is now treated as the first important external `TBBX`
   consumer target.
3. The current recommendation is:
   - validate plain `TBBX` consumption first
   - keep `Threads` as the control case
   - defer callback-based native tasking integration

## Local ISPC Paths To Read First

Use the local clone, not the network.

Start here:

1. [CMakeLists.txt](/Users/me/wip-gcd-tbb-fx/nx/ispc/ispcrt/detail/cpu/CMakeLists.txt)
2. [ispc_tasking.cpp](/Users/me/wip-gcd-tbb-fx/nx/ispc/ispcrt/detail/cpu/ispc_tasking.cpp)
3. [ispcrt.h](/Users/me/wip-gcd-tbb-fx/nx/ispc/ispcrt/ispcrt.h)
4. [ispcrt.cpp](/Users/me/wip-gcd-tbb-fx/nx/ispc/ispcrt/ispcrt.cpp)
5. [ispcrt/CMakeLists.txt](/Users/me/wip-gcd-tbb-fx/nx/ispc/ispcrt/CMakeLists.txt)
6. [ispc.rst](/Users/me/wip-gcd-tbb-fx/nx/ispc/docs/ispc.rst)
7. [ReleaseNotes.txt](/Users/me/wip-gcd-tbb-fx/nx/ispc/docs/ReleaseNotes.txt)
8. [examples/common/tasksys.cpp](/Users/me/wip-gcd-tbb-fx/nx/ispc/examples/common/tasksys.cpp)

Most important local facts:

1. `ispcrt/detail/cpu/CMakeLists.txt` shows the supported tasking models and
   the FreeBSD/Linux default:
   - `OpenMP`
   - `TBB`
   - `Threads`
   - default `TBB` on non-Windows, non-Apple
2. The `TBB` path defines `ISPC_USE_TBB_PARALLEL_FOR`, not
   `ISPC_USE_TBB_TASK_GROUP`.
3. `ispcrt/detail/cpu/ispc_tasking.cpp` implements the active `TBB` path with
   `tbb::parallel_for`.
4. `ispcrt.h` exposes `ispcrtSetTaskingCallbacks()`.
5. `ispcrt.cpp` prefers caller-supplied callbacks over the built-in
   `ISPCLaunch_cpu` / `ISPCAlloc_cpu` / `ISPCSync_cpu` symbols.
6. `docs/ispc.rst` still documents the generic three-function task-runtime
   contract for custom task systems.
7. `ReleaseNotes.txt` speaks more strongly about `TBB` than the current source
   actually enforces, so build-system statements should be checked against code,
   not accepted blindly.
8. the FreeBSD `ispc` port already models a CPU-only lane:
   - depends on `devel/onetbb`
   - disables GPU and ISPCRT tests
   - still packages `libispcrt.so` and `libispcrt_device_cpu.so`
9. the FreeBSD `onetbb` port already provides the ordinary consumer surface:
   - `include/tbb/*`
   - `TBBConfig.cmake`
   - `libtbb.so`

## Source Evidence To Anchor On

These are the strongest concrete anchors and should be cited in any report:

1. [CMakeLists.txt](/Users/me/wip-gcd-tbb-fx/nx/ispc/ispcrt/detail/cpu/CMakeLists.txt#L4)
   shows the supported tasking models.
2. [CMakeLists.txt](/Users/me/wip-gcd-tbb-fx/nx/ispc/ispcrt/detail/cpu/CMakeLists.txt#L8)
   through [CMakeLists.txt](/Users/me/wip-gcd-tbb-fx/nx/ispc/ispcrt/detail/cpu/CMakeLists.txt#L13)
   show the default model selection.
3. [CMakeLists.txt](/Users/me/wip-gcd-tbb-fx/nx/ispc/ispcrt/detail/cpu/CMakeLists.txt#L33)
   through [CMakeLists.txt](/Users/me/wip-gcd-tbb-fx/nx/ispc/ispcrt/detail/cpu/CMakeLists.txt#L78)
   show the active `TBB` build path and `TBB::tbb` linkage.
4. [ispc_tasking.cpp](/Users/me/wip-gcd-tbb-fx/nx/ispc/ispcrt/detail/cpu/ispc_tasking.cpp#L866)
   through [ispc_tasking.cpp](/Users/me/wip-gcd-tbb-fx/nx/ispc/ispcrt/detail/cpu/ispc_tasking.cpp#L889)
   show the active `tbb::parallel_for` path.
5. [ispcrt.h](/Users/me/wip-gcd-tbb-fx/nx/ispc/ispcrt/ispcrt.h#L80)
   through [ispcrt.h](/Users/me/wip-gcd-tbb-fx/nx/ispc/ispcrt/ispcrt.h#L86)
   show the public callback types and `ispcrtSetTaskingCallbacks()`.
6. [ispcrt.cpp](/Users/me/wip-gcd-tbb-fx/nx/ispc/ispcrt/ispcrt.cpp#L167)
   through [ispcrt.cpp](/Users/me/wip-gcd-tbb-fx/nx/ispc/ispcrt/ispcrt.cpp#L243)
   show the override-vs-default callback loading logic.
7. [ispc.rst](/Users/me/wip-gcd-tbb-fx/nx/ispc/docs/ispc.rst#L4964)
   onward documents the generic `ISPCAlloc` / `ISPCLaunch` / `ISPCSync`
   runtime contract.
8. [ReleaseNotes.txt](/Users/me/wip-gcd-tbb-fx/nx/ispc/docs/ReleaseNotes.txt#L1047)
   and [ReleaseNotes.txt](/Users/me/wip-gcd-tbb-fx/nx/ispc/docs/ReleaseNotes.txt#L1104)
   show the public callback addition and the stronger "requires TBB" wording.
9. [/usr/ports/devel/ispc/Makefile](/usr/ports/devel/ispc/Makefile#L13)
   and [/usr/ports/devel/ispc/pkg-plist](/usr/ports/devel/ispc/pkg-plist#L23)
   show the current FreeBSD CPU-only packaging shape.
10. [/usr/ports/devel/onetbb/pkg-plist](/usr/ports/devel/onetbb/pkg-plist#L110)
    through [/usr/ports/devel/onetbb/pkg-plist](/usr/ports/devel/onetbb/pkg-plist#L161)
    show the downstream-consumer package surface that `ISPCRT` expects.

## Working Model

The strongest current reading is:

1. `ISPCRT` wants `TBB` by default on FreeBSD/Linux.
2. Its current `TBB` dependency is shallow enough that it is a good first
   `TBBX` compatibility target.
3. The public callback seam exists for custom task systems, but using it would
   bypass the ordinary downstream-consumer validation story.
4. A good `TBBX` FreeBSD port should match the ordinary oneTBB package surface
   closely enough that `ISPCRT` can keep using `find_package(TBB)` and
   `TBB::tbb` without special-case logic.

That means the clean progression is:

1. validate `ISPCRT` against `TBBX` as a normal `TBB` consumer;
2. use `Threads` as the control/fallback path;
3. only later ask whether `ISPCRT` should consume native `GCDX` / `TWQ`
   machinery directly.

## What The Sidecar Agent Should Produce

The most useful outputs are:

1. a precise inventory of the `TBB` API surface actually used by `ISPCRT`
   today;
2. a concrete FreeBSD build matrix for:
   - `ISPCRT_BUILD_TASK_MODEL=TBB`
   - `ISPCRT_BUILD_TASK_MODEL=Threads`
   - optional `OpenMP` comparison
3. a statement of whether `TBBX` consumer validation requires:
   - only `TBB::tbb`
   - any extra packaging assumptions
   - any extra ABI/version assumptions
4. a recommended first-run validation plan:
   - which examples/tests to use
   - what successful behavior looks like
   - what failures would point to `TBBX` vs `ISPCRT` vs generic build problems
5. a later-stage evaluation of the callback seam:
   - whether it can sensibly host a `GCDX`-backed tasking implementation
   - what would be gained
   - what would be lost
6. a recommendation for a tiny downstream-consumer CI lane that configures
   `ispcrt` in CPU-only `TBB` mode against `TBBX`

## Questions The Sidecar Agent Should Answer

1. What exact `TBB` symbols and headers does `ISPCRT` consume in the active
   FreeBSD/Linux path?
2. Can current `ISPCRT` be built cleanly against a FreeBSD `oneTBB` / `TBBX`
   package with only the standard `TBB::tbb` target?
3. Which `ISPC` examples or tests actually exercise the CPU tasking path
   through `ISPCRT` in a way useful for `TBBX` validation?
4. Does `ISPCRT` impose any assumptions about `TBB` packaging, SONAME, or
   CMake package layout that a FreeBSD `TBBX` port must match?
5. Is `ispcrtSetTaskingCallbacks()` truly process-global in a way that would
   complicate later native integration?
6. If we later route `ISPCRT` through native callbacks, does that still serve
   the project goal of validating `TBBX`, or does it become a different
   project entirely?
7. Does the current FreeBSD `TBB` packaging surface already tell us what
   `TBBX` must preserve for stock downstream consumers?

## Negative Guidance

The sidecar agent should avoid these low-value rabbit holes at the start:

1. Do not redesign `ISPCRT` around `GCDX` before validating the ordinary
   `TBB` consumer path.
2. Do not conflate callback-based custom tasking with `TBBX` compatibility.
3. Do not assume the compiler front-end is the main integration seam; it is
   not.
4. Do not treat `ReleaseNotes.txt` as more authoritative than the current
   source tree.
5. Do not burn time on `TCM` for this task; `ISPC` consumer validation is not
   blocked on `PlanC`.
6. Do not over-focus on hypothetical long-term elegance before proving the
   simple path.

## Suggested First Concrete Steps

1. Reconfirm the active `TBB` path from local source.
2. Enumerate the exact `TBB` surface used by `ISPCRT`.
3. Identify the smallest example or test that exercises `ISPCRT` CPU tasking.
4. Draft a FreeBSD-side build plan for `ISPCRT` with:
   - `TBB`
   - `Threads`
5. Use the `Threads` build as the control case for later `TBBX` debugging.
6. Only after that, sketch the callback-based native-tasking path as a
   separate appendix.
7. Add a tiny downstream-consumer CI recommendation for the parent `TBBX`
   lane:
   - `ISPCRT_BUILD_TASK_MODEL=TBB`
   - `ISPCRT_BUILD_GPU=OFF`
   - `ISPCRT_BUILD_TESTS=OFF`

## Bottom Line

The dedicated agent should treat `ISPC` as a practical downstream `TBBX`
consumer, not as an excuse to immediately bypass `TBB`.

The first milestone is:

1. prove that `ISPCRT` can consume `TBBX` through its ordinary `TBB` path;
2. keep `Threads` as the control and fallback path;
3. postpone `ispcrtSetTaskingCallbacks()` and native `GCDX` integration until
   after that plain-consumer story is working.

That is the shortest path to a useful answer for the project.
