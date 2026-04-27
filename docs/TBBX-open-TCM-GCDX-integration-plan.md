# TBBX Open TCM And GCDX Integration Plan

Date: 2026-04-27

## Purpose

This note updates `PlanC` now that the official open-source TCM work exists as
oneTBB PR #2061 and is cloned locally in:

- `../nx/oneTBB-TCM`

The goal is to decide how the real upstream TCM implementation helps TBBX, how
it should integrate with GCDX / `pthread_workqueue` / TWQ, and whether any TCM
code or concepts should move into the lower kernel mechanism.

## Current Upstream Status

The upstream PR is real but not merged yet.

As of 2026-04-27, GitHub reports:

- PR: `uxlfoundation/oneTBB#2061`
- title: `[TCM] Open source Thread Composability Manager`
- state: open
- head branch: `open-source-tcm`
- head SHA: `7bc8577403df32b69b25157caad45677a8552861`
- changed files: 66
- additions: 15355
- review comments: 5
- mergeable: true
- mergeable state: unstable

Local evidence:

- `../nx/oneTBB-TCM/thread_composability_manager/` exists
- TCM is a standalone CMake project
- TCM version is `1.5.0`
- the main implementation is in
  `../nx/oneTBB-TCM/thread_composability_manager/src/tcm.cpp`
- the public C ABI is in
  `../nx/oneTBB-TCM/thread_composability_manager/include/tcm.h`

This changes the project from RFC-era planning to source-based planning.

## Executive Verdict

Open-source TCM directly validates `PlanC`.

It eliminates most of the reverse-engineering guesswork around:

- permit states
- permit lifecycle
- callback delivery
- fair-balance negotiation
- nested permits
- thread registration semantics
- topology and CPU-kind usage
- package/build shape

It does not justify moving TCM into TWQ or the kernel.

The correct integration is:

```text
oneTBB / OpenMP / future runtimes
  -> libtcm.so.1
  -> TCM permit and grant policy
  -> FreeBSD provider layer
  -> GCDX / pthread_workqueue / TWQ pressure facts
  -> kernel mechanism
```

The strongest implementation idea is to represent external TWQ pressure in
TCM's permit economy as a private synthetic reserve permit created by a
FreeBSD pressure adapter through the public TCM API. That lets TCM reuse its
own negotiation machinery to reduce grants when TWQ pressure rises, without
making GCDX depend on TCM and without adding permit vocabulary to the kernel.

## What The Real TCM Implementation Contains

### Public ABI

The open TCM API exports the expected oneTBB-facing functions:

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

`version.h` also declares compatibility version functions:

- `tcmGetVersion`
- `tcmRuntimeVersion`
- `tcmRuntimeInterfaceVersion`

This means TBBX no longer needs to guess the ABI surface from the proprietary
binary. The source confirms it.

### Data Model

The public types confirm:

- `TCM_PERMIT_STATE_VOID`
- `TCM_PERMIT_STATE_INACTIVE`
- `TCM_PERMIT_STATE_PENDING`
- `TCM_PERMIT_STATE_IDLE`
- `TCM_PERMIT_STATE_ACTIVE`
- `request_as_inactive`
- `rigid_concurrency`
- `exclusive`
- `stale`
- `min_sw_threads`
- `max_sw_threads`
- CPU constraints using `hwloc_bitmap_t`
- high-level constraints using NUMA id, core type id, and threads-per-core

This is exactly the model we had inferred, but now the edge cases are visible.

### oneTBB Integration

The updated oneTBB adaptor still loads `libtcm.so.1` dynamically on Unix. It
resolves 11 symbols and falls back when TCM is unavailable.

Important observed behavior:

- oneTBB creates one TCM client for the adaptor.
- oneTBB creates a TCM permit per arena client.
- arenas start by requesting inactive permits.
- `TCM_PERMIT_STATE_INACTIVE` maps to zero arena concurrency.
- every non-inactive permit maps to the granted concurrency integer.
- `tcm_adaptor::set_active_num_workers(int)` is empty.
- TCM does not create or admit worker threads.

This confirms again that TCM is a coordination broker, not a worker execution
mechanism.

### Grant Engine

The main implementation is `ThreadComposabilityFairBalance`.

The grant engine:

- stores pending, idle, and active permits in ordered containers
- tracks `available_concurrency`
- processes pending permits first
- then distributes unused resources to the least unhappy active permits
- treats idle permits as negotiable resources
- avoids callback invocation while holding the main data mutex
- merges repeated callback notifications
- uses a per-permit epoch to make permit data copies stable

This is valuable code. It is not just an API shim.

### Topology

TCM has a concrete `system_topology` wrapper around hwloc.

It currently:

- loads hwloc topology at library load time
- duplicates topology during TCM initialization
- reads process CPU affinity through `hwloc_get_cpubind`
- converts CPU masks to NUMA masks
- parses NUMA nodes
- parses hybrid CPU kinds through `hwloc_cpukinds_get_nr` and
  `hwloc_cpukinds_get_info`
- falls back to one homogeneous core type if CPU-kind parsing fails

This confirms that `hwloc` is not incidental. It is deeply embedded in the
current TCM topology path.

### Linux cgroup Support

The source has a Linux-only `cgroup_info` helper.

It reads:

- `/proc/self/mounts`
- `/proc/self/cgroup`
- `/sys/fs/cgroup`
- cgroup v1 `cpu.cfs_quota_us` / `cpu.cfs_period_us`
- cgroup v2 `cpu.max`

The core `tcm.cpp` includes this only under `#if __linux__`.

This is useful for FreeBSD because it shows the exact kind of platform budget
hook upstream is willing to have: a platform-specific source can reduce
`process_concurrency` before TCM computes the resource pool.

FreeBSD should not copy cgroups. FreeBSD should add its own provider path.

## What The Source Changes For TBBX

### It eliminates the local TCM clone as the main path

`PlanA` as a local reimplementation is now only a fallback.

We should not rebuild the TCM state machine while upstream source exists under
a permissive license.

### It makes PlanC executable

The old PlanC assumption was:

- wait for upstream source
- port it
- then add native pressure/capacity inputs

The first assumption is now true enough to act on:

- the source exists in PR form
- it is not merged yet
- but it is real code and can be inspected

### It gives us an exact inspection target

Before source, the provider boundary was abstract.

Now the real inspection and future-extension targets are visible:

- `ThreadComposabilityManagerData` constructor
- `process_concurrency`
- `available_concurrency`
- `platform_resources(process_concurrency)`
- `try_satisfy_request`
- `renegotiate_permits`
- `system_topology`
- Linux `cgroup_info` as precedent for platform-specific budget input

For v1 pressure integration, these do not have to become patch targets. A
FreeBSD adapter can create a normal TCM client and a normal permit through the
public API. Source changes are only needed later if we want a tighter
upstreamable provider abstraction or lower-latency event integration.

### It exposes the main structural risk

There is no obvious provider abstraction yet.

The current code directly uses:

- `system_topology::instance()`
- `hwloc_bitmap_*`
- `hwloc_cpukinds_get_info`
- `available_concurrency`

So integrating GCDX pressure cleanly probably requires a local FreeBSD patch
or an upstreamable provider refactor.

That is acceptable if we keep the patch narrow.

## FreeBSD Port Surface

The standalone configure probe failed in this environment because `hwloc` is
not installed:

```text
Cannot find HWLOC: HWLOC >= 2.0 required.
```

Local checks also found no `hwloc-info`, no `pkg-config hwloc`, and no
`/usr/local/include/hwloc.h`.

This does not prove a FreeBSD source problem. It only proves this workspace is
missing the required dependency.

Expected FreeBSD work:

- add or depend on `devel/hwloc2`
- verify CMake can find `hwloc` through the FreeBSD package
- check whether the current hardening flags are accepted by FreeBSD clang/lld
- run TCM tests with `TCM_ENABLE=1`
- verify non-Linux cgroup tests skip cleanly
- verify oneTBB can load `libtcm.so.1`

The source already avoids Linux cgroup compilation on non-Linux systems.

## How TCM Should Use GCDX / TWQ

### Direction

The direction is one-way upward:

```text
GCDX / TWQ facts -> provider snapshot -> TCM grant policy
```

Not:

```text
TCM -> GCDX / TWQ control
```

GCDX must not include `tcm.h`, must not know permit states, and must not have
`if (tcm_present)` branches.

### Pressure Provider

GCDX should expose an aggregate pressure snapshot from libthr or an adjacent
private platform library.

The first version should be snapshot-first, not callback-first:

```c
struct _pthread_workqueue_pressure_snapshot_v1 {
    size_t struct_size;
    uint32_t version;
    uint32_t _pad0;
    uint64_t generation;
    uint64_t timestamp_ns;

    /* Current gauges at capture time. */
    uint32_t total_workers;
    uint32_t idle_workers;
    uint32_t nonidle_workers;

    /* Cumulative event counters since the provider's base snapshot. */
    uint32_t requested_workers;
    uint32_t admitted_workers;
    uint32_t blocked_workers;
    uint32_t unblocked_workers;
    uint32_t narrowed_events;

    uint32_t reserved[6];
};
```

Rules:

- `struct_size` is the ABI versioning gate.
- `version` is informational; consumers branch on `struct_size`, not on
  `version`.
- The provider fills only the fields the caller's size can hold.
- Caller and provider both zero the struct before filling known fields.
- A field is present only if `struct_size` covers the whole field.
- The provider exposes aggregate pressure facts only.
- `generation == 0` means no pressure data is available yet.
- `total_workers`, `idle_workers`, and `nonidle_workers` are current gauges.
- `requested_workers`, `admitted_workers`, `blocked_workers`,
  `unblocked_workers`, and `narrowed_events` are cumulative event counters
  since the provider base snapshot.
- Cumulative `uint32_t` counters may wrap; consumers compute deltas with
  unsigned wrapping subtraction, for example
  `delta = (uint32_t)(current - previous)`.
- Per-QoS bucket details stay below this line for v1.
- CPU count belongs to the topology provider, not the pressure provider.
- Capacity fractions belong to TCM policy, not the pressure provider.
- Pressure-state enums belong to TCM policy, not the pressure provider.
- No TCM vocabulary appears in this structure.
- v1 is intended for the LP64 FreeBSD runtime shape. An ILP32 consumer would
  need an explicitly reviewed ABI variant because `size_t` width is part of
  the struct layout.

Possible private entry point:

```c
int __pthread_workqueue_pressure_snapshot_np(
    struct _pthread_workqueue_pressure_snapshot_v1 *snapshot);
```

The struct and entry point should use platform-neutral naming. TBBX-specific
names belong only in the TCM-side adapter that consumes this interface.

The struct definition should live in a libthr / pthread-workqueue-owned private
platform header, not in a TBBX header. Otherwise libthr would need to include a
consumer header to implement the mechanism interface, which would invert the
dependency.

The native provider should keep GCDX's existing `size_t struct_size`
convention. If TBBX wants a private projection struct with `uint32_t
struct_size`, that projection must stay above the provider line and must not
become the platform wire format.

Generation and timestamp are live-ABI requirements. The current GCDX M15
bundle has generation and monotonic-time fields for the callable preview lane.
The final pressure SPI still needs provider-owned generation and
`CLOCK_MONOTONIC` timestamp capture at the live snapshot point, rather than an
extractor-generated sequence.

TBBX derives backlog facts above the provider line:

```text
request_backlog = max(requested_workers - admitted_workers, 0)
block_backlog   = max(blocked_workers - unblocked_workers, 0)
```

Backlog is useful for saturation detection, renegotiation triggers, and later
hysteresis. It must not directly size the reserve permit in v1.

### TCM-Side Consumer

For v1, TBBX should consume this provider from a FreeBSD pressure sidecar
adapter that lives above the provider line and links to the public TCM API.

Preferred packaging:

```text
libtbbx_twq_bridge.so
```

This sidecar should:

- link to `libtcm.so.1`
- call the private TWQ pressure SPI
- create and update the reserve permit through the public TCM API
- load without breaking processes that do not use oneTBB
- degrade gracefully if `libtcm.so.1`, TCM enablement, or the pressure SPI is
  unavailable

It should not live inside the FreeBSD TCM port for v1, because that would make
pressure integration a local patch to upstream TCM. It should not be a
separate process, because IPC is unnecessary for process-local pressure
sampling.

The one unresolved v1 packaging detail is trigger ownership. A sidecar with no
thread and no TCM/oneTBB patch needs an invocation point. For the first
prototype, the test harness may call the adapter explicitly. A production
version can later choose a narrow TCM/oneTBB hook, symbol interposition, or a
small polling thread, but that choice should be driven by M5/M6 measurements.

The pressure snapshot field set should be treated as frozen for v1. The
delivery mechanism can stay provisional until the mixed-runtime baseline proves
that pressure integration is worth carrying.

If this later becomes an upstream TCM provider, the natural source locations
are:

```text
thread_composability_manager/src/freebsd/pressure_info.h
```

or better, under a more general provider abstraction if upstream accepts it:

```text
thread_composability_manager/src/platform/pressure_provider.h
```

The provider-side data should answer:

- how many external workers are currently non-idle
- which worker/request/blocking counters changed since the base snapshot
- when the snapshot last changed

The adapter should use that data to change TCM permit demand, not to operate
threads.

## Preferred Integration Mechanism: Synthetic Reserve Permit

The cleanest way to integrate TWQ pressure into the current TCM implementation
is to model external TWQ consumption as an adapter-created synthetic reserve
permit.

### Why

TCM already knows how to:

- reserve resources for a permit
- make other permits negotiate downward
- notify oneTBB when a grant changes
- release resources and renegotiate upward later

If TWQ pressure is represented as a private permit, TCM can reuse that existing
machinery instead of adding a second capacity-shrink algorithm.

### Shape

The reserve participant can be built with the public TCM API:

```c
static tcm_client_id_t reserve_client;
static tcm_permit_handle_t reserve_permit;
static uint32_t last_reserve_demand;
static bool reserve_connected;
static bool reserve_active;  /* Adapter-side non-zero demand flag. */

static tcm_result_t
reserve_noop_callback(tcm_permit_handle_t permit, void *arg,
    tcm_callback_flags_t flags)
{
    (void)permit;
    (void)arg;
    (void)flags;
    return TCM_RESULT_SUCCESS;
}

void
freebsd_pressure_adapter_init(void)
{
    if (tcmConnect(reserve_noop_callback, &reserve_client) !=
        TCM_RESULT_SUCCESS) {
        reserve_connected = false;
        return;
    }

    reserve_connected = true;
    reserve_active = false;
    reserve_permit = NULL;
    last_reserve_demand = 0;
}

void
freebsd_pressure_adapter_update(uint32_t external_nonidle,
    uint32_t platform_concurrency)
{
    if (!reserve_connected)
        return;

    uint32_t demand = external_nonidle;
    if (demand > platform_concurrency)
        demand = platform_concurrency;
    if (demand == last_reserve_demand)
        return;

    if (demand == 0) {
        if (reserve_permit != NULL && reserve_active)
            (void)tcmDeactivatePermit(reserve_permit);
        reserve_active = false;
        last_reserve_demand = 0;
        return;
    }

    tcm_permit_request_t req = TCM_PERMIT_REQUEST_INITIALIZER;
    req.min_sw_threads = demand;
    req.max_sw_threads = demand;
    req.flags.rigid_concurrency = 1;

    if (tcmRequestPermit(reserve_client, req, NULL, &reserve_permit, NULL) ==
        TCM_RESULT_SUCCESS) {
        reserve_active = true;
        last_reserve_demand = demand;
    }
}

void
freebsd_pressure_adapter_shutdown(void)
{
    if (!reserve_connected)
        return;

    if (reserve_permit != NULL) {
        (void)tcmReleasePermit(reserve_permit);
        reserve_permit = NULL;
    }

    (void)tcmDisconnect(reserve_client);
    reserve_connected = false;
    reserve_active = false;
    last_reserve_demand = 0;
}
```

TCM sees this as a normal client with a normal permit. The permit is
"synthetic" only from the adapter's perspective: it represents external TWQ
consumption rather than a real TCM-aware runtime.

The reserve client must be separate from oneTBB's TCM client and must use a
non-null no-op callback. TCM stores callbacks by client and asserts before
invocation; `tcmConnect(NULL, ...)` is not safe for a permit that may appear in
renegotiation callback sets.

The reserve request should set `rigid_concurrency = 1`. External TWQ pressure
is a fact, not a negotiable preference. If 3 CPUs are already consumed by TWQ
workers, TCM should not negotiate that reserve down to make another TCM permit
happier.

The `reserve_active` flag is an adapter-side non-zero-demand flag, not an
authoritative copy of TCM's internal permit state. It is reliable for the v1
deactivation path because the reserve permit is unconstrained and rigid. If a
future version relaxes `rigid_concurrency`, tests should query
`tcmGetPermitData` when they need actual reserve state.

The request should not use `request_as_inactive` for normal non-zero demand
updates. `request_as_inactive` is useful only if the adapter chooses to
pre-create an inactive placeholder permit; the active reserve path should
request the current non-zero demand directly.

TCM uses the same `tcmRequestPermit` entry point for first request and
re-request. If a higher rigid reserve demand cannot be satisfied immediately,
the reserve permit can temporarily become pending while normal TCM
renegotiation settles. That is expected behavior and should be visible in the
M6 reserve-demand and permit-state trace.

When TWQ pressure rises:

1. The adapter reads the pressure snapshot.
2. The adapter increases the reserve permit demand through `tcmRequestPermit`.
3. The normal negotiation path takes resources from idle or negotiable active
   permits.
4. TCM invokes normal client callbacks.
5. oneTBB arenas reduce concurrency through the existing adaptor path.

When TWQ pressure relaxes:

1. The adapter lowers non-zero reserve permit demand through
   `tcmRequestPermit`.
2. Resources return to `available_concurrency`.
3. TCM renegotiates pending/unsatisfied permits.
4. oneTBB arenas receive larger grants if demand exists.

When TWQ pressure drops to zero, the adapter should call
`tcmDeactivatePermit`, not `tcmRequestPermit` with `min=max=0`. That matches
oneTBB's own adaptor lifecycle: zero demand deactivates the permit while
preserving the handle for future re-request.

At sidecar shutdown, explicitly call `tcmReleasePermit` before
`tcmDisconnect`. `tcmDisconnect` unregisters the client and can clean up client
permits, but explicit release matches oneTBB's own adaptor discipline and makes
the reserve lifecycle auditable.

### TCM enablement lifecycle

The adapter must treat TCM availability as optional.

Source review shows `tcmConnect` returns an error when `TCM_ENABLE` is not
enabled. The adapter must check the return value, leave reserve demand at zero,
and continue without affecting oneTBB fallback behavior.

Deployment rule:

- tests that intend to exercise TCM should set `TCM_ENABLE=1` before loading
  oneTBB, TCM, or the sidecar
- tests that intentionally disable TCM should set `TCM_ENABLE=0` to avoid
  confusing suggestion messages from TCM development-environment heuristics
- the sidecar should not mutate `TCM_ENABLE` after TCM may already have
  initialized

### Why this is better than mutating `available_concurrency` directly

A direct capacity ceiling is deceptively hard.

If all resources are already granted and TWQ pressure rises, simply reducing
`available_concurrency` is not enough because the resources are not in the
available pool. They are in active permits.

A synthetic reserve permit turns external pressure into a normal TCM
stakeholder. That means existing code can negotiate active permits down.

### Why this still keeps GCDX independent

The synthetic permit exists only above the provider line.

The FreeBSD pressure adapter may include `tcm.h` and request a TCM permit.
GCDX exports worker pressure facts only. It does not include `tcm.h`, does not
request a TCM permit, does not receive a TCM callback, and does not know the
reserve permit exists.

This preserves the PlanC invariant.

### Oscillation control

The reserve permit must not blindly renegotiate on every transient sample if
TWQ pressure oscillates quickly.

The v1 adapter should start conservatively:

- poll only during TCM permit request / renegotiation paths
- clamp reserve demand to TCM platform concurrency
- update the reserve permit only when the computed demand changes
- keep pressure-provider failure non-fatal
- add no smoothing until raw reserve-demand behavior has been measured

If workloads show grant oscillation, add smoothing above the provider line:

- first, quantize reserve demand to small worker-count buckets
- generation-based hold-down before releasing reserve demand
- small rate limit on reserve-demand changes
- hysteresis using observer/tracker transition summaries

This smoothing belongs in the TCM-side adapter, not in GCDX.

### Trigger latency

Polling only during TCM request or renegotiation paths is the right v1
implementation, but it has a known limitation: no TCM event means no pressure
poll. If GCD pressure spikes while oneTBB has no permit activity, the reserve
permit can remain stale until the next TCM event or timer poll.

That is acceptable for v1 because it keeps the boundary simple and one-way. If
mixed workloads show stale pressure response, v2 should add event-assisted
notification where the provider only signals "generation changed" and the
adapter still pulls a coherent snapshot.

## What Should Not Be Done

### Do not put TCM code into the kernel

The TCM source uses:

- C++17
- hwloc
- C callbacks into user runtimes
- process-local singleton state
- `std::mutex`
- STL containers
- dynamic allocation
- user-space environment variables
- shared-library packaging

That is the wrong shape for FreeBSD kernel code.

Moving this into TWQ would not simplify TWQ. It would import a user-space
policy engine into a kernel mechanism.

### Do not make pthread_workqueue depend on TCM

`pthread_workqueue` should remain a native worker-admission mechanism.

It should not include:

- TCM headers
- permit handles
- TCM states
- grant negotiation
- TCM callbacks

### Do not expose raw TWQ internals to TCM

TCM should not see:

- raw scheduler run queues
- individual thread state transitions
- libdispatch queue internals
- per-QoS bucket layout in v1
- kernel narrowing implementation details

TCM should see aggregate pressure facts.

### Do not replace TCM with TWQ

TWQ does not coordinate oneTBB and OpenMP permits.

TCM does not admit kernel workers.

They are complementary, not substitutes.

## What Can Be Borrowed Into GCDX

TCM code should not move into TWQ, but TCM concepts can improve GCDX-side
design and tests.

Useful concepts:

- explicit state machines
- generation/epoch-based coherent snapshots
- callback delivery outside core locks
- fair-share test cases
- active/idle distinction
- rigid versus negotiable demand
- fuzzing and constrained-request tests

For GCDX, the most useful borrowing is not the permit API. It is the discipline:

- snapshot state coherently
- expose aggregate facts
- keep callbacks out of kernel locks
- test pathological contention and reentrancy

## Implementation Plan

### N1: Build Open TCM On FreeBSD

Tasks:

- install or provide `devel/hwloc2`
- configure TCM standalone with tests enabled
- build `libtcm.so.1`
- run the TCM test suite
- verify cgroup tests skip cleanly on non-Linux

Expected early issue:

- this workspace currently lacks hwloc, so configure cannot proceed yet

Exit criteria:

- `libtcm.so.1` builds on FreeBSD
- core TCM tests pass
- FreeBSD-specific build failures are documented or fixed

### N2: Make oneTBB Load Open TCM

Tasks:

- build oneTBB from the PR branch or a matching oneTBB tree
- ensure oneTBB dynamically finds `libtcm.so.1`
- run with `TCM_ENABLE=1`
- verify fallback to `market` when TCM is absent or disabled
- validate simple oneTBB workloads through the existing adaptor

Exit criteria:

- oneTBB uses open TCM through the existing adaptor
- no oneTBB scheduler changes are required
- permit lifecycle and callback behavior survive FreeBSD stress tests

### N3: Establish The Mixed-Runtime Baseline

This is the critical experiment before building pressure integration.

First run single-runtime baselines:

- GCD only
- oneTBB only
- GCD + oneTBB with TCM off

Then run mixed GCD + oneTBB workloads under these conditions:

- A: TCM off
- B: TCM on, no pressure adapter
- C: TCM on, pressure adapter enabled later

For N3, only A and B are required.

Measure:

- peak process thread count
- context switches
- throughput
- oneTBB arena concurrency changes
- GCDX pressure bundle samples

Decision criteria:

- If B materially improves over A, open TCM is already valuable and the
  pressure adapter is an optimization.
- If B does not improve over A because GCD/TWQ pressure dominates, the
  pressure adapter is on the critical path.
- If the workload is not reproducible, do not tune policy yet.
- If condition A already shows acceptable thread counts, context switches, and
  throughput, TCM pressure integration may be solving a non-problem.

Exit criteria:

- measured baseline exists before adding the pressure adapter
- the project knows whether pressure integration is required or optional for
  the first useful result

### N4: Add The Pressure Adapter If N3 Shows Need

Tasks:

- consume `twq_pressure_provider_bundle_v1` above the provider line
- project `nonidle_workers_current` into reserve demand
- create a private reserve client and reserve permit through the public TCM API
- update reserve demand with `tcmRequestPermit`
- call `tcmDeactivatePermit` when reserve demand drops to zero
- treat `tcmConnect` failure and missing `TCM_ENABLE=1` as non-fatal
- keep polling-only v1 unless latency measurements prove it stale
- measure raw reserve-demand behavior before adding smoothing

Exit criteria:

- condition C improves mixed-runtime behavior over condition B
- GCDX remains fully independent underneath
- no TCM source changes are required for v1
- `TCM_ENABLE` lifecycle and load-order assumptions are documented

### Later: Event-Assisted Pressure

Add notification only if snapshot polling is too stale.

Notification should mean "snapshot generation changed." The adapter still
pulls a coherent snapshot after notification. Do not build an event-stream
state machine.

### Later: Native Hybrid Capacity

Add future FreeBSD `hmp(4)` user-space capacity input when available.

Goals:

- replace weak hwloc CPU-kind ranking for hybrid-capacity decisions
- keep topology traversal in hwloc or a topology provider
- keep dynamic capacity behind a separate provider
- improve TCM grant quality on hybrid P-core/E-core hardware

## GCDX Boundary Answers

The GCDX-side review resolves the main provider-boundary questions:

- The pressure snapshot should live in libthr or an adjacent private platform
  library, not in a TBBX header and not as a TCM-specific API.
- The provider can expose aggregate worker/request/block/narrowing facts from
  existing TWQ accounting without adding TCM concepts.
- v1 cannot distinguish external GCDX pressure from future TBBX-owned TWQ
  workers. If oneTBB later uses TWQ directly, add a runtime-origin tag below
  the provider line and keep the TCM adapter as the translator.
- Provider generation and monotonic timestamp are part of the live target. The
  current M15 preview has a callable-session generation suitable for adapter
  prototyping.
- The provider should expose numeric facts only. `pressure_state` and
  `consumed_capacity_1024` are adapter-side projections.
- Coherent bounded-lock snapshots are preferable to lock-free incoherent
  reads.
- v1 should remain polling-only. Event notification is a later optimization
  that says only "generation changed."

## GCDX M15 Response

The GCDX M15 response confirms the PlanC boundary.

GCDX will keep the lower provider surface in the
`twq_pressure_provider_*` namespace and will not expose TCM vocabulary below
the provider line. TBBX is responsible for projecting those aggregate facts
into any TCM-side policy object, including the synthetic reserve permit.

The current GCDX preview artifact is:

```text
twq_pressure_provider_bundle_v1
```

It combines:

- callable session state
- current aggregate view
- observer summary
- transition tracker summary

The TBBX-side consumption rule is documented in:

- `docs/TBBX-GCDX-M15-pressure-provider-consumption.md`

The critical projection rule is:

- use `current_view.nonidle_workers_current` as the current external worker
  consumption signal
- do not size the reserve permit from `request_backlog_total`,
  `block_backlog_total`, or `pressure_visible`

Those backlog fields are cumulative from the session base and can remain
nonzero after the final state is quiescent. They are useful for diagnostics,
triggering, and later hysteresis, but not as the direct reserve size in v1.

## Source-Based Review Refinements

The full TCM source review adds three concrete corrections to this plan:

1. The reserve permit does not require TCM grant-engine changes in v1. Create a
   private client and permit with `tcmConnect` and `tcmRequestPermit`; TCM sees
   a normal permit with `min=max=external_nonidle_workers`.
2. Zero reserve demand should deactivate the reserve permit with
   `tcmDeactivatePermit`, not re-request `min=max=0`.
3. `tcmConnect` failure is normal when `TCM_ENABLE` is not enabled. The adapter
   must degrade to reserve demand zero and leave oneTBB fallback behavior
   unchanged.
4. Trigger latency is the first known limitation. Polling on TCM activity can
   miss pure GCD/TWQ pressure transitions until the next TCM event or timer.
   Event-assisted generation notification is the v2 fix, not a v1 dependency.
5. The mixed-runtime baseline must come before pressure integration. Measure
   TCM-off versus TCM-on/no-adapter first so the project knows whether the
   pressure adapter is required or merely an optimization.

## Stop Rules

Switch away from the current PlanC pressure-adapter path if any of these become
true:

- TCM cannot be made to build on FreeBSD because of hard `hwloc` or platform
  assumptions.
- Upstream TCM remains unmerged or abandoned by October 2026.
- The uncoordinated mixed-runtime baseline already shows no meaningful
  oversubscription problem.
- The N3 workload cannot reliably create simultaneous GCD/TWQ and oneTBB
  contention.
- Mixed-runtime runs with the pressure adapter show no improvement over the
  TCM-only baseline.
- Reserve-permit oscillation cannot be controlled with adapter-side smoothing.
- The TCM grant engine behaves pathologically with a dynamic background
  reserve permit, such as callback storms or permit starvation.
- A production trigger mechanism cannot be designed without violating the
  provider-line boundary.
- Upstream TCM requires a downward control path into libthr, TWQ, or GCDX.
- TCM enablement or load-order requirements create deployment friction larger
  than the bridge value.
- Maintaining the FreeBSD pressure adapter costs more than the value it
  provides; use this as the trigger to reassess PlanB.

## Current Top Risk

The top implementation risk is no longer the TCM API shape. It is trigger
ownership.

The sidecar has no natural production invocation point if it owns no thread and
does not patch TCM or oneTBB. The prototype should use explicit harness polling
because it is deterministic. A production bridge should only add a trigger
after M5/M6 proves value. Candidate production triggers, in order of
preference:

1. a narrow oneTBB-side hook near TCM demand adjustment;
2. event-assisted generation notification from the pressure provider;
3. a small sidecar polling thread if the overhead is measured and acceptable.

Avoid symbol interposition unless no cleaner hook exists. Avoid any trigger
that requires libthr, TWQ, or GCDX to know about TCM.

## Bottom Line

The open-source TCM PR is highly useful.

It should replace reverse-engineering as the source of truth for TCM behavior.
It should become the upstream-aligned PlanC broker on FreeBSD once ported.

It should not be incorporated into `pthread_workqueue` or the kernel.

The right integration is to let GCDX export pressure facts upward and let a
FreeBSD TCM pressure adapter translate those facts into permit-grant changes.
The best current design is a private adapter-created reserve permit backed by
a GCDX/TWQ pressure snapshot. That gives us integration, performance, and reuse
without violating the mechanism/policy boundary.
