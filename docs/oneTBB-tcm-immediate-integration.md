# oneTBB Immediate TCM Integration

## Purpose

This document describes the immediate oneTBB integration path: provide a
`libtcm.so.1`-compatible service layer now, and explain exactly how that layer
can take advantage of the existing GCDX / TWQ / `pthread_workqueue` work
without turning oneTBB into a dispatch runtime.

## Immediate Goal

The immediate goal is not full long-term oneTBB substrate replacement.

The immediate goal is:

1. make oneTBB usable on this platform quickly;
2. reuse the existing kernel/user-space work wherever it already solves the
   same low-level problem;
3. keep the oneTBB-facing seam narrow and controlled.

That makes TCM the right phase-1 integration boundary.

## Why TCM First

oneTBB already has the seam:

- `tcm.h` defines the public contract;
- `tcm_adaptor.cpp` dynamically loads `libtcm.so.1`;
- `threading_control.cpp` already switches to the TCM adaptor when it is
  available.

So the immediate platform story should be:

1. provide `libtcm.so.1`;
2. implement only the semantics oneTBB actually depends on first;
3. back that implementation with platform-native pressure and concurrency
   signals where possible.

Phase-1 activation should be explicit, not ambient.

Because oneTBB will try to load `libtcm.so.1` whenever it is present, the
broker should be able to decline connection cleanly when coordination is not
enabled for the process. The easiest phase-1 shape is an activation gate in
`tcmConnect`, for example via an environment variable or equivalent platform
switch.

That activation decision is decisive:

- if `tcmConnect` returns success, oneTBB commits to the TCM path;
- if `tcmConnect` returns an error, oneTBB falls back to its internal `market`
  path.

There is no later fallback boundary inside the same process.

## Export Surface And Compatibility Boundary

The public oneTBB adaptor resolves 11 TCM symbols at load time, and all of them
must be present for initialization to succeed.

Intel's current shipped TCM binaries expose two additional compatibility
exports:

- `tcmRuntimeVersion`
- `tcmRuntimeInterfaceVersion`

So the practical phase-1 compatibility target is a 13-symbol surface:

1. the 11 symbols oneTBB resolves directly;
2. the 2 extra version-query exports shipped by Intel's current Linux and
   Windows binaries.

Those extra exports are functions, not data objects. oneTBB does not currently
resolve them itself, but exporting them keeps the platform library closer to a
drop-in replacement for Intel's shipped TCM runtime.

## Immediate Layering

The clean phase-1 architecture is:

```text
application
  -> oneTBB API
  -> oneTBB scheduler / arenas / tasking semantics
  -> oneTBB tcm_adaptor
  -> libtcm.so.1 (platform implementation)
  -> platform coordination broker
```

The crucial point is that oneTBB talks to `libtcm.so.1`, not to `libdispatch`
queues, and not directly to `_pthread_workqueue_*`.

Phase 1.5 then extends the lower half of the picture:

```text
libtcm.so.1 broker
  -> private pressure provider in libthr / pthread_workqueue
  -> kernel TWQ mechanism
```

## Permit-Centric Broker Model

The broker must be modeled around permits, not around a single client-wide
budget object.

From the oneTBB adaptor behavior:

1. `tcmConnect(...)` is called once per adaptor instance, producing one client
   ID;
2. each arena later issues its own `tcmRequestPermit(...)`;
3. each permit carries its own demand and lifecycle;
4. renegotiation is callback-driven per permit via the permit-specific
   `callback_arg`.

So the actual allocation problem in phase 1 is:

- given one client with N permits, and potentially other clients with their own
  permits later, distribute a total CPU budget across active permits.

That means the broker should store:

- a client table for ownership and disconnect lifecycle;
- a permit table as the actual allocation unit.

## How TCM Can Benefit From The Existing TWQ / pthread_workqueue Work

The broker can benefit from the main lane in several concrete ways.

### 1. Reuse TWQ as the source of real pressure information

The existing main effort already knows things a process-wide coordination broker
cares about:

- how many workers were requested;
- how many workers were admitted;
- how many workers are active;
- how many workers are blocked or recently blocked;
- when narrowing is appropriate.

That is exactly the kind of information a TCM-style broker can use to decide
how much concurrency should still be granted to oneTBB.

In other words:

- TWQ solves "what is the real runtime pressure and available headroom?";
- TCM can consume that answer when computing oneTBB permits.

### 2. Reuse the libthr bridge as the export point, not libdispatch as an API

The clean place to expose dispatch/TWQ pressure to `libtcm.so.1` is not the
public `libdispatch` API.

The cleaner place is a small private user-space provider interface near the
existing `libthr` / `pthread_workqueue` bridge, because that is already where
the runtime-to-kernel accounting boundary lives.

That keeps the dependency direction clean:

- `libtcm.so.1` depends on a platform pressure provider;
- it does not depend on dispatch queue internals.

### 3. Reuse the admission philosophy already proven in the main lane

The main lane already uses a real admission and narrowing model rather than a
naive "active threads < ncpu" heuristic.

The immediate broker should reuse that philosophy:

- account for real runnable pressure;
- discount short stalls and transient blocking;
- avoid oversubscription;
- respect the difference between demand and admitted execution.

This is better than inventing a second simplified policy engine just for TCM.

### 4. Reuse platform observability

The broker should reuse the same style of proof used by the main lane:

- explicit snapshots;
- visible pressure effects;
- no silent fallback;
- controlled test workloads.

That makes TCM integration debuggable instead of magical.

### 5. Reuse topology and capacity knowledge where possible

If the platform already has reliable knowledge about CPUs, capacity, or
placement constraints from the main effort or shared topology code, the TCM
broker should consume that instead of constructing its own competing view.

## Concrete TCM-to-TWQ Data Flow

This is the immediate path that makes sense.

### Step 1: oneTBB publishes demand to `libtcm.so.1`

Through the existing TCM adaptor, oneTBB tells the broker:

- minimum desired software threads;
- maximum desired software threads;
- optional constraints;
- thread participation updates.

In oneTBB's current shape, that happens as one connected client with multiple
permits, typically one permit per arena.

### Step 2: the broker computes a process-wide budget

The broker starts with:

- machine capacity;
- process-local runtime state;
- direct TCM client demand.

### Step 3: the broker incorporates dispatch/TWQ pressure

The broker then asks a platform pressure provider for the current dispatch/TWQ
view of pressure, such as:

- active workers by bucket;
- recently blocked workers by bucket;
- requested versus admitted work;
- whether the dispatch lane is already in a narrowed state.

This lets the broker answer:

- how much of the machine is already effectively committed to the dispatch/TWQ
  lane?
- how much concurrency headroom remains for oneTBB right now?

### Step 4: the broker derives oneTBB permit grants

From there the broker computes:

- oneTBB permit state;
- allowed concurrency;
- optional later placement constraints.

In phase 1, those grants should be computed per permit, not as a single
client-wide number.

### Step 5: oneTBB updates arena concurrency

The existing oneTBB TCM adaptor already knows how to:

- receive a callback;
- call `tcmGetPermitData`;
- update arena concurrency from the returned permit.

That is why TCM is the practical immediate seam.

## Recommended Immediate Provider Boundary

The cleanest immediate integration is:

1. keep `libtcm.so.1` as the oneTBB-facing broker;
2. add a small private pressure-provider SPI below it;
3. implement that SPI using the existing `libthr` / TWQ bridge or kernel
   snapshot path.

In other words:

- oneTBB-facing boundary: TCM
- platform-pressure boundary: TWQ snapshot/provider SPI

This avoids coupling TCM to `libdispatch` internals while still reusing the
main agent's work.

## Suggested Private Provider Shape

The right near-term boundary is not polling-only and not event-only.

It should be a **hybrid provider SPI** exposed from the existing `libthr` /
`pthread_workqueue` side rather than from `libdispatch` itself:

1. event notification so the broker does not have to poll blindly;
2. versioned snapshot query so the broker can read a coherent state after an
   event.

For example:

```c
typedef enum {
    TWQ_PRESSURE_EVENT_CHANGED = 1,
    TWQ_PRESSURE_EVENT_RISING,
    TWQ_PRESSURE_EVENT_RELAXED
} twq_pressure_event_t;

typedef void (*twq_pressure_notify_f)(void* ctx, twq_pressure_event_t event,
                                      uint64_t generation);

struct twq_pressure_snapshot_v1 {
    uint32_t version;
    uint32_t ncpu;
    uint64_t generation;
    uint64_t timestamp_ns;

    uint32_t total_requested;
    uint32_t total_admitted;
    uint32_t total_active;
    uint32_t total_recently_blocked;
    uint32_t total_idle;

    /* Explicitly defined as TWQ priority/QoS buckets, not vague "buckets". */
    uint32_t requested_by_bucket[6];
    uint32_t admitted_by_bucket[6];
    uint32_t active_by_bucket[6];
    uint32_t recently_blocked_by_bucket[6];

    uint32_t narrow_hint_mask;
};

int _pthread_workqueue_register_pressure_listener_np(twq_pressure_notify_f fn,
                                                     void* ctx);
int _pthread_workqueue_get_pressure_snapshot_np(struct twq_pressure_snapshot_v1 *out);
```

The exact shape can change, but the boundary principles should stay the same:

- expose the data already closest to the runtime-to-kernel boundary;
- keep the SPI private and platform-native;
- make the semantic meaning of each bucket explicit;
- let `libtcm.so.1` react to change events, then pull a coherent snapshot.

### ABI hygiene for the provider SPI

The provider SPI should be versioned in semantics, not only in field names.

That means:

- the snapshot call should accept the consumer's structure size;
- the provider must fill only the fields that fit in that size;
- unknown fields must be zeroed, not left uninitialized;
- the consumer must check `version` and any bucket-count field before reading
  bucket arrays;
- the callback registration path should tolerate a newer provider and an older
  broker without undefined behavior.

This matters because `libtcm.so.1` and the `libthr` provider may evolve on
different schedules.

### Why hybrid event + snapshot is the best phase-1.5 shape

- pure polling introduces avoidable staleness;
- pure events do not give the broker a coherent state to reason over;
- a hybrid model matches the existing broker-to-runtime callback shape well;
- it keeps dependencies one-way;
- it does not require dispatch to become a first-class TCM client yet.

### How `libtcm.so.1` should use the provider

At minimum, the broker can:

1. treat total admitted and active workers as stronger consumption signals than
   raw requests;
2. treat recently blocked workers as partially consuming capacity, in the same
   spirit as the main lane's admission model;
3. use TWQ priority/QoS bucket information as a policy input where it matters,
   without depending on dispatch internals;
4. reduce oneTBB permit grants when TWQ pressure is rising or sustained;
5. relax those grants again when pressure falls.

That gives the TCM broker a direct way to benefit from the main agent's kernel
and `pthread_workqueue` work without turning dispatch into a dependency surface
for oneTBB.

## Callback Reentrancy Protocol

The broker must specify the renegotiation protocol explicitly before
implementation.

Required rules:

1. the broker must not hold its main mutable state lock while invoking the
   TCM callback;
2. the broker must first commit the new grant state into broker-owned storage;
3. the callback should be delivered asynchronously or via deferred notification
   after state commit;
4. it must be legal for callback-driven oneTBB code to re-enter the broker via
   `tcmGetPermitData(...)` and later `tcmRequestPermit(...)`;
5. callback delivery must be per changed permit, not merely per client.

This is the most dangerous correctness boundary in the phase-1 design. It must
be treated as a protocol, not as an implementation detail.

## Immediate Implementation Phases

### Phase 1: Minimal standalone broker

Implement:

- 13 exported symbols matching the current Intel 1.2.0 surface;
- process-local client and permit tracking;
- permit-centric fair-share concurrency grants;
- explicit `INACTIVE` handling;
- thread registration accounting;
- `hwloc` as the canonical topology layer, but only for PU counting and
  constraint capture in phase 1;
- activation gate in `tcmConnect`;
- callback-driven grant refresh;
- no dependency on TWQ pressure yet.

The grant model should stay deliberately simple in phase 1:

- treat `INACTIVE` versus "not inactive" as the only semantically important
  state distinction for oneTBB;
- satisfy per-permit mandatory floors first;
- divide remaining capacity across active permits;
- do not attempt to model richer TCM policy features yet.

When mandatory floors exceed available budget, phase 1 should use an explicit
conflict rule rather than silently overcommitting:

1. preserve at least a minimal non-zero floor for each active permit when
   possible;
2. reduce mandatory floors proportionally when total requested floors exceed
   available budget;
3. never exceed the computed total CPU budget in order to "honor" every floor
   simultaneously.

The exact proportional rule can stay simple in phase 1, but it must be written
down and tested.

At this stage, the broker can work even before TWQ pressure is wired in.

### Phase 1.5: TWQ-informed broker

Add:

- a pressure-provider interface below the broker;
- event notification plus coherent TWQ pressure snapshot as input to grant
  decisions;
- reuse of existing admission and narrowing ideas from the main lane.

This is the first point where oneTBB meaningfully benefits from the other main
agent's kernel/user-space work.

### Phase 2: richer cross-runtime coordination

Add later if justified:

- topology-sensitive grants;
- better state modeling;
- stronger coordination across more than one TCM-aware runtime;
- explicit treatment of dispatch as a first-class broker participant if that
  becomes useful.

## What The Immediate Integration Must Not Do

### Do not make oneTBB depend on dispatch APIs

oneTBB should not:

- enqueue its work onto dispatch queues as the main design;
- use `libdispatch` as its scheduler replacement;
- depend on dispatch queue semantics for correctness.

### Do not duplicate admission logic

The broker should not invent a separate oversubscription model when the TWQ
lane already has a better one.

It can start simpler for phase 1, but it should converge toward the same
platform-native pressure philosophy rather than drifting into a second
unrelated policy engine.

### Do not confuse inter-runtime and intra-runtime arbitration

The broker should not:

- decide how oneTBB divides its granted budget across arenas;
- replace oneTBB market-style allotment logic;
- make per-task or per-arena scheduling decisions.

Its authority is the runtime-level budget ceiling, not oneTBB's internal
scheduler policy.

### Do not over-model TCM semantics in phase 1

Phase 1 should not assume more semantic richness than oneTBB currently uses.

Specifically:

- do not build elaborate `PENDING` / `IDLE` transitions unless workload
  evidence requires them;
- do not depend on immediate permit fill through the final `tcmRequestPermit`
  parameter, because oneTBB currently passes `nullptr` and reads grants later
  via `tcmGetPermitData(...)`;
- do not treat `stale`, `exclusive`, or `rigid_concurrency` as mandatory for
  correctness in the first broker.

## Validation And CI Requirements

The market fallback path must stay healthy even after `libtcm.so.1` exists.

At minimum, CI should exercise two configurations:

1. `libtcm.so.1` present and TCM explicitly enabled:
   - proves oneTBB is using the broker path
2. `libtcm.so.1` present and TCM explicitly disabled:
   - proves oneTBB still falls back cleanly to `market`

If the second configuration fails, it should be treated as a blocker. The
fallback path is not optional engineering debt; it is the escape hatch while
the broker matures.

### Do not move TCM into the kernel

TCM is the wrong kernel boundary for phase 1.

The kernel should continue to expose TWQ-like mechanism.
The TCM broker should remain user-space and consume those mechanism signals.

## Immediate Engineering Checklist

The immediate TCM effort should answer these concrete questions:

1. what private provider interface should expose dispatch/TWQ pressure to
   `libtcm.so.1`?
2. which TWQ signals are the minimum useful inputs for oneTBB permit
   computation?
3. how should dispatch/TWQ pressure map onto a reduced oneTBB concurrency
   grant?
4. what oneTBB workloads should validate that permit reductions actually change
   arena behavior as expected?
5. how do we prove the broker is using the platform-native path and not just a
   fallback fair-share heuristic?

## Immediate Bottom Line

The immediate integration path is:

1. support oneTBB through a `libtcm.so.1` compatibility layer;
2. keep that broker in user space;
3. let it consume TWQ / `pthread_workqueue` pressure through a small private
   provider boundary;
4. reuse the main lane's kernel/user-space work as low-level pressure and
   admission input;
5. keep oneTBB's scheduler and runtime semantics above that broker.

That is the shortest path to oneTBB support that is both practical now and
consistent with the longer-term shared-substrate strategy.
