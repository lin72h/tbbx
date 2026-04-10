# oneTBB Immediate TCM Integration

## Purpose

This document describes the immediate oneTBB integration path: provide a
`libtcm.so.1`-compatible service layer now, and explain exactly how that layer
can take advantage of the existing TWQ / `pthread_workqueue` / `libdispatch`
work without turning oneTBB into a dispatch runtime.

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

## Immediate Layering

The clean phase-1 architecture is:

```text
application
  -> oneTBB API
  -> oneTBB scheduler / arenas / tasking semantics
  -> oneTBB tcm_adaptor
  -> libtcm.so.1 (platform implementation)
  -> platform coordination broker
  -> TWQ / pthread_workqueue / libthr pressure provider
  -> kernel TWQ mechanism
```

The crucial point is that oneTBB talks to `libtcm.so.1`, not to `libdispatch`
queues, and not directly to `_pthread_workqueue_*`.

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

The most practical near-term shape is a **snapshot-style private SPI** exposed
from the existing `libthr` / `pthread_workqueue` side rather than from
`libdispatch` itself.

For example:

```c
struct twq_pressure_snapshot_v1 {
    uint32_t version;
    uint32_t ncpu;
    uint64_t timestamp_ns;

    uint32_t requested_by_bucket[6];
    uint32_t admitted_by_bucket[6];
    uint32_t active_by_bucket[6];
    uint32_t recently_blocked_by_bucket[6];
    uint32_t idle_by_bucket[6];

    uint32_t narrow_hint_mask;
};

int _pthread_workqueue_pressure_snapshot_np(struct twq_pressure_snapshot_v1 *out);
```

The exact field names can change, but the idea should stay the same:

- expose the data already closest to the runtime-to-kernel boundary;
- keep the SPI private and platform-native;
- let `libtcm.so.1` consume snapshots instead of peering into dispatch.

### Why snapshot-first is the best phase-1 shape

- it is simple to implement;
- it is easy to test against existing counters and probes;
- it keeps dependencies one-way;
- it does not require dispatch to become a first-class TCM client yet.

### How `libtcm.so.1` should use the snapshot

At minimum, the broker can:

1. estimate how much concurrency is already effectively occupied by the
   dispatch/TWQ lane;
2. treat admitted and active workers as stronger consumption signals than raw
   requests;
3. treat recently blocked workers as partially consuming capacity, in the same
   spirit as the main lane's admission model;
4. reduce oneTBB permit grants when dispatch pressure is high;
5. relax those grants again when TWQ pressure falls.

That gives the TCM broker a direct way to benefit from the main agent's kernel
and `pthread_workqueue` work without turning dispatch into a dependency surface
for oneTBB.

## Immediate Implementation Phases

### Phase 1: Minimal standalone broker

Implement:

- full exported TCM symbol set;
- process-local client and permit tracking;
- simple fair-share concurrency grants;
- explicit `INACTIVE` handling;
- thread registration accounting;
- callback-driven grant refresh.

At this stage, the broker can work even before TWQ pressure is wired in.

### Phase 1.5: TWQ-informed broker

Add:

- a pressure-provider interface below the broker;
- dispatch/TWQ pressure snapshots as an input to grant decisions;
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
