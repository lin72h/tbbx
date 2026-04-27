# TBBX Consumption Of GCDX M15 Pressure Provider

Date: 2026-04-27

## Purpose

This note records the TBBX-side response to the GCDX M15 pressure-provider
coordination milestone in:

- `../wip-codex54x/m15-tbbx-tcm-pressure-provider-coordination.md`
- `../wip-codex54x/m15-pressure-provider-bundle-smoke.md`

The goal is to freeze how TBBX should consume the GCDX pressure bundle and how
that bundle should project into the TCM synthetic reserve-permit design.

## Executive Decision

Accept the GCDX boundary as-is.

GCDX owns the lower provider line and keeps the namespace:

- `twq_pressure_provider_*`

TBBX must not ask GCDX to expose:

- `tbbx_*` names below the line
- TCM permit counts
- TCM permit states
- TCM callbacks
- reserve-permit hints
- TCM policy vocabulary

TBBX consumes aggregate pressure facts and performs all TCM projection above
the provider line.

## What GCDX Delivered

GCDX now has a callable pressure-provider bundle lane:

```text
session
  -> view
  -> observer
  -> tracker
  -> bundle
```

The important checked-in C surface is:

- `csrc/twq_pressure_provider_adapter.h`
- `csrc/twq_pressure_provider_session.h`
- `csrc/twq_pressure_provider_observer.h`
- `csrc/twq_pressure_provider_tracker.h`
- `csrc/twq_pressure_provider_bundle.h`

The bundle type is:

```c
struct twq_pressure_provider_bundle_v1;
```

The callable flow is:

```c
twq_pressure_provider_bundle_init_v1(&bundle);
twq_pressure_provider_bundle_prime_v1(&bundle);
twq_pressure_provider_bundle_poll_v1(&bundle);
```

The bundle is not yet a public ABI. It is a stable preview artifact suitable
for early TBBX adapter work.

The future live SPI should be platform-owned, not TBBX-owned. The target shape
is:

```c
struct _pthread_workqueue_pressure_snapshot_v1 {
    size_t struct_size;
    uint32_t version;
    uint32_t _pad0;
    uint64_t generation;
    uint64_t timestamp_ns;

    uint32_t total_workers;
    uint32_t idle_workers;
    uint32_t nonidle_workers;

    uint32_t requested_workers;
    uint32_t admitted_workers;
    uint32_t blocked_workers;
    uint32_t unblocked_workers;
    uint32_t narrowed_events;

    uint32_t reserved[6];
};
```

This live struct belongs in a libthr / pthread-workqueue-owned private platform
header. It must not live in a TBBX header, because that would make the
mechanism layer depend on a consumer.

The live provider should keep the GCDX convention of `size_t struct_size`.
TBBX may copy data into any private projection type it wants above the provider
line, but the provider-facing structure should follow the native
`twq_pressure_provider_*` width convention.

`struct_size` is the primary ABI gate. `version` is informational. Consumers
should branch on whether `struct_size` covers a complete field, not on the
version number. Caller and provider both zero the struct before filling known
fields. `generation == 0` means no pressure data is available yet.

The cumulative event counters are `uint32_t` in this v1 shape and may wrap.
Consumers must compute deltas with wrapping-safe unsigned subtraction:

```c
uint32_t delta = (uint32_t)(current - previous);
```

The live v1 layout is intended for the LP64 FreeBSD runtime shape. An ILP32
consumer would need an explicitly reviewed ABI variant because `size_t` width
is part of the struct layout.

The M15 bundle is the preview input. The live snapshot is the later SPI target.
Both carry the same semantic rule: current non-idle workers are the current
consumption signal; cumulative counters are diagnostic or trigger inputs.

## Relevant Bundle Fields

The current TBBX adapter should primarily consume:

```text
bundle.current_view.generation
bundle.current_view.monotonic_time_ns
bundle.current_view.total_workers_current
bundle.current_view.idle_workers_current
bundle.current_view.nonidle_workers_current
bundle.current_view.active_workers_current
bundle.current_view.request_backlog_total
bundle.current_view.block_backlog_total
bundle.current_view.should_narrow_true_total
bundle.current_view.pressure_visible
bundle.current_quiescent
bundle.current_narrow_feedback
bundle.generation_contiguous
bundle.monotonic_increasing
```

The observer and tracker are useful for validation and later hysteresis:

```text
bundle.observer.max_nonidle_workers_current
bundle.observer.max_request_backlog_total
bundle.observer.max_block_backlog_total
bundle.tracker.nonidle_rises
bundle.tracker.nonidle_falls
bundle.tracker.quiescent_rises
bundle.tracker.quiescent_falls
```

## Critical Projection Rule

For the initial TCM reserve-permit projection, use:

```text
current external worker consumption = current_view.nonidle_workers_current
```

Do not use `request_backlog_total` or `block_backlog_total` as the reserve
demand.

Reason:

- `nonidle_workers_current` is the contract's current pressure signal.
- `request_backlog_total` is derived from cumulative request/admission deltas
  since the session base.
- `block_backlog_total` is derived from cumulative block/unblock deltas since
  the session base.
- `pressure_visible` can remain true after the system is quiescent because
  backlog totals can remain nonzero.

The baseline bundle confirms this: final state can be quiescent with
`current_nonidle_workers_current == 0`, while `current_pressure_visible` is
still true due to request backlog history.

So v1 projection must be current-consumption based, not historical-pressure
based.

For a future live `_pthread_workqueue_pressure_snapshot_v1`, the equivalent
projection input is:

```text
snapshot.nonidle_workers
```

For saturation detection, the adapter should derive:

```text
request_backlog = max(snapshot.requested_workers - snapshot.admitted_workers, 0)
block_backlog   = max(snapshot.blocked_workers - snapshot.unblocked_workers, 0)
```

For the current M15 bundle preview, the equivalent inputs are already present
as:

```text
bundle.current_view.request_backlog_total
bundle.current_view.block_backlog_total
```

Those backlog values are useful for distinguishing saturation, oversubscription
and quiescence in the adapter, but they do not directly size the reserve permit
in v1.

## Initial Reserve Demand Formula

The initial TBBX/TCM adapter should compute:

```c
uint32_t
tbbx_external_reserve_demand_v1(
    const struct twq_pressure_provider_bundle_v1 *bundle,
    uint32_t tcm_platform_concurrency)
{
    if (bundle == NULL)
        return 0;

    if (!bundle->generation_contiguous || !bundle->monotonic_increasing)
        return 0;

    if (bundle->current_quiescent)
        return 0;

    uint64_t nonidle = bundle->current_view.nonidle_workers_current;
    if (nonidle > tcm_platform_concurrency)
        nonidle = tcm_platform_concurrency;

    return (uint32_t)nonidle;
}
```

This formula is deliberately conservative.

It only reserves capacity for workers that are currently non-idle in the GCDX
lane. It does not reserve capacity for historical backlog.

## How To Use Backlog And Transitions

In v1:

- `request_backlog_total` is a pressure-history signal.
- `block_backlog_total` is a pressure-history signal.
- `pressure_visible` is a trigger signal.
- tracker rises/falls are validation and possible hysteresis inputs.
- observer max values are benchmark and tuning inputs.

They should not directly become TCM reserve demand.

Possible later use:

- trigger a fresh TCM renegotiation when pressure first becomes visible
- delay reserve release by a short generation window if nonidle oscillates
- tune hysteresis thresholds for sustained pressure workloads

But v1 should keep the reserve demand tied to current non-idle workers.

## Oscillation Control

The reserve permit should not churn on every transient pressure fluctuation.

The initial adapter should:

- poll during TCM request / renegotiation paths
- clamp demand to TCM platform concurrency
- update the reserve permit only when demand changes
- treat provider errors as reserve demand zero
- subtract TBBX-owned TWQ workers later if oneTBB ever starts using TWQ
  directly and GCDX adds a runtime-origin tag
- add no smoothing until raw reserve-demand behavior has been measured

If this causes grant oscillation, smoothing should be added above the provider
line:

- first, quantize reserve demand to small worker-count buckets
- hold reserve demand for a small number of generations after non-idle falls
- rate-limit large demand swings
- use tracker rise/fall summaries as hysteresis inputs

This smoothing is TCM-side policy. It must not be pushed into GCDX.

## TCM-Side Adapter Shape

TBBX should add a TCM-side adapter with this shape:

```text
libtbbx_twq_bridge.so
  -> owns twq_pressure_provider_bundle_v1
  -> connects a private TCM reserve client through tcmConnect
  -> polls bundle before permit request / renegotiation
  -> computes external reserve demand
  -> updates a private reserve permit through tcmRequestPermit
```

The adapter owns translation from pressure facts to TCM policy. For v1 this
does not require TCM source changes; the reserve participant is a normal TCM
client and permit created through the public API.

The preferred package shape is a sidecar library, not a TCM source patch and
not a separate process. The sidecar may include `tcm.h`; GCDX may not.

The trigger path is deliberately left as a prototype decision. A no-thread
sidecar needs some invocation point: explicit calls from the test harness,
symbol interposition, a narrow TCM/oneTBB hook, or later a small polling
thread. M5/M6 measurements should decide which one is worth carrying.

Treat trigger ownership as the top implementation risk for the bridge. The
prototype should use explicit harness polling. Production should prefer a
narrow oneTBB-side hook near TCM demand adjustment or event-assisted generation
notification. Avoid a trigger that makes libthr, TWQ, or GCDX TCM-aware.

The v1 field set should be frozen now. The delivery mechanism can remain
provisional until the mixed-runtime baseline proves that pressure integration
is worth carrying.

GCDX remains pressure-only.

## Synthetic Reserve Permit Update

The reserve permit remains the preferred TCM-side model:

```text
external reserve demand N
  -> private adapter-owned TCM reserve permit min=N max=N
  -> normal TCM negotiation
  -> normal callbacks to oneTBB / other clients
```

The synthetic permit must be:

- private to the FreeBSD pressure adapter
- created through the public TCM API
- connected through a separate TCM client with a non-null no-op callback
- not represented in GCDX
- updated only from the TBBX/TCM adapter projection
- requested with `rigid_concurrency = 1`

If demand changes from `N` to `M`, TCM should re-request the reserve permit and
then run normal renegotiation.

If demand drops to zero, the adapter should call `tcmDeactivatePermit` rather
than re-requesting `min=max=0`. The permit handle should remain valid so a
later non-zero demand can call `tcmRequestPermit` on the existing handle.

On sidecar shutdown, explicitly call `tcmReleasePermit` before
`tcmDisconnect`. `tcmDisconnect` can clean up the client's permits, but
explicit release keeps the reserve lifecycle aligned with oneTBB's adaptor
discipline.

An adapter-side `reserve_active` flag should be treated as a non-zero-demand
flag, not as an authoritative copy of TCM permit state. It is reliable for v1
because the reserve request uses `rigid_concurrency = 1`; if v2 relaxes that,
tests should query `tcmGetPermitData` for actual reserve state.

The adapter must also treat `tcmConnect` failure as normal. If `TCM_ENABLE` is
not enabled, reserve demand is zero and TBBX continues without pressure
coordination.

Do not rename the provider fields to `delta_*` in v1. The provider has no
per-consumer session argument in the live function shape, so it should expose
cumulative counters. The adapter derives deltas between its own samples using
wrapping-safe arithmetic.

The important boundary is that the adapter may include `tcm.h`; GCDX may not.
The adapter is above the provider line, so using the public TCM API there does
not contaminate TWQ, libthr, or libdispatch.

## Trigger Latency

Polling during TCM request or renegotiation paths is enough for v1, but it is
not instantaneous. If GCD/TWQ pressure changes while no TCM permit event is
happening, the reserve permit can remain stale until the next TCM event or
timer poll.

That limitation should be measured before adding event machinery. The v2 event
shape should still be one-way: provider says "generation changed"; the adapter
pulls a coherent snapshot and updates the reserve permit if needed.

## Adapter Validation Rules

Before using a bundle sample, TBBX should require:

- `bundle.version == TWQ_PRESSURE_PROVIDER_BUNDLE_VERSION`
- `bundle.struct_size == sizeof(bundle)`
- source session/view/observer/tracker versions are stable
- `bundle.generation_contiguous != 0`
- `bundle.monotonic_increasing != 0`
- `bundle.sample_count > 0`

If validation fails:

- ignore the pressure sample
- set reserve demand to zero
- do not fail TCM connection
- do not affect oneTBB fallback behavior

Pressure data is an optimization input, not a correctness dependency.

## Confirmed Boundary

The GCDX agent explicitly confirmed:

- no `tcm.h` below the provider line
- no TCM permit handles or permit states in TWQ / `pthread_workqueue`
- no `if (tcm_present)` branches in GCDX mechanism code
- no synthetic reserve permit in the GCDX repo
- aggregate pressure facts only for this provider version

TBBX should treat this as the correct contract.

## Immediate TBBX Tasks

1. Add a TCM-side adapter design note or source stub for consuming
   `twq_pressure_provider_bundle_v1`.
2. Build a replay/projection tool that reads the bundle JSON artifact and emits
   reserve-demand samples.
3. Validate that final quiescent bundle states project to reserve demand zero,
   even if `pressure_visible` remains true.
4. Add a public-API synthetic reserve-permit prototype after the upstream TCM
   port builds.
5. Keep all pressure-provider failure modes non-fatal.

## Bottom Line

The GCDX M15 response matches PlanC.

TBBX should consume the GCDX bundle as a pressure-only artifact and project it
into TCM policy above the provider line. The first reserve-permit projection
must use `nonidle_workers_current` as the current external consumption signal.
Backlog and pressure-visible fields are useful for triggering, validation, and
future hysteresis, but they must not directly size the reserve permit in v1.
