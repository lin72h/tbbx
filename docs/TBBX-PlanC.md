# TBBX PlanC

## Purpose

`TBBX-PlanC` is the active layered path.

It assumes the upstream Thread Composability Manager (TCM) source becomes
available and is portable enough to run on FreeBSD, but it also assumes the
existing GCDX / `pthread_workqueue` / TWQ / future `hmp(4)` machinery remains
the stronger native execution substrate.

Under that assumption, PlanC keeps TCM as the user-space coordination layer
and makes it ride on top of the platform machinery already being built for
GCDX.

In short:

- upstream/open TCM stays the coordination broker
- GCDX / TWQ stays the mechanism layer
- TCM consumes platform pressure and capacity facts through provider
  boundaries
- oneTBB, OpenMP, and future runtimes consume TCM normally

## Core Definition

`PlanC` means:

1. preserve the oneTBB API for applications;
2. preserve the oneTBB `tcm_adaptor` path and `libtcm.so.1` loading model;
3. prefer upstream/open TCM over a local TCM reimplementation;
4. keep TWQ / `pthread_workqueue` / scheduler feedback as the native
   execution mechanism;
5. add a one-way bridge so platform pressure and capacity facts can inform TCM
   grant computation;
6. keep GCDX independent underneath, whether or not TCM is present.

This is not:

- replacing TWQ with TCM;
- moving TCM into the kernel;
- making GCDX depend on TCM;
- rewriting oneTBB around a native `permit_manager` first.

## Why PlanC Exists

PlanC exists because two things are now true at once.

First, the oneTBB side is moving toward open-source TCM:

- the RFC was merged in commit
  [430e2bcb](/Users/me/wip-gcd-tbb-fx/nx/oneTBB/rfcs/proposed/coordinate_cpu_resources_use/readme.org)
- the RFC says TCM will be an independent subproject, buildable with
  `TCM_BUILD`, distributed separately, and licensed under Apache 2.0 with LLVM
  exception
- the RFC also says TCM is meant to remain a singleton shared library, not a
  static component

Second, the platform already has stronger native machinery below that layer:

- kernel-backed worker admission and backpressure through TWQ
- `libthr` / `pthread_workqueue` bridge work
- future FreeBSD-native hybrid-capacity work through `hmp(4)` / HFI
- existing pressure/accounting thinking in GCDX

That creates a synthesis path:

- do not replace TCM if upstream gives us the real thing
- do not throw away the platform machinery we already built
- layer them

PlanC should therefore be read as:

- the active path if upstream TCM lands and ports cleanly
- not a third seam
- and not a replacement for PlanA's TCM seam so much as the upstream-first
  version of it plus the later platform bridge

## Architectural Reading

The right reading is:

- TWQ is mechanism
- TCM is policy

That distinction is not cosmetic.

TWQ deals in:

- real threads
- admission
- blocking and unblocking
- backpressure and narrowing
- scheduler feedback

TCM deals in:

- permits
- requested minimum and maximum concurrency
- granted concurrency ceilings
- renegotiation
- process-wide runtime coordination

So the correct relationship is not "TWQ or TCM."

It is:

- TWQ below
- TCM above

## Architecture

The `PlanC` stack is:

```text
application
  -> oneTBB / OpenMP / future runtimes
  -> oneTBB tcm_adaptor or equivalent runtime-side TCM client
  -> libtcm.so.1 (upstream/open TCM on FreeBSD)
  -> provider interfaces / bridge layer
  -> GCDX / libthr / pthread_workqueue / TWQ / hmp
  -> FreeBSD kernel scheduler and machine state
```

The important split is:

```text
TCM
  -> topology provider
  -> capacity provider
  -> pressure provider
  -> GCDX / TWQ / hmp facts
```

That bridge is one-way upward:

- GCDX informs TCM
- TCM does not control GCDX's kernel mechanisms

Layered more sharply:

- Layer A: runtime clients
- Layer B: TCM coordination
- Layer C: provider bridge / translation layer
- Layer D: native mechanism

The critical layer is Layer C. It must translate native mechanism facts into
TCM-appropriate policy inputs. It must not become a raw pass-through of TWQ
internals.

## What PlanC Preserves

PlanC preserves:

- the upstream oneTBB TCM seam
- the shared-library singleton model of TCM
- oneTBB's existing `tcm_adaptor`
- oneTBB fallback to `market` when TCM is unavailable or disabled
- the possibility of supporting other TCM-aware runtimes later

It also preserves the platform's own lower-layer investment:

- TWQ admission and backpressure
- `pthread_workqueue` bridge work
- future `hmp(4)` user-space export
- kernel-observed pressure and capacity facts

## What PlanC Reuses

PlanC should reuse the platform machinery in the following order.

### Pressure facts

The most important thing TCM can reuse is real pressure information:

- active workers
- admitted versus requested workers
- blocked or recently blocked workers
- narrowing / backpressure state

These are stronger than runtime self-reports alone.

### Topology and capacity facts

PlanC should also reuse:

- CPU count
- NUMA shape
- static cluster or CPU-kind information
- future native hybrid capacity and throttling signals

In the near term, `hwloc2` may still be the easiest topology source.
Longer-term, native FreeBSD `hmp(4)` export should become the better hybrid
capacity source.

### Observability discipline

PlanC should reuse the same style of debugging and validation already used in
the GCDX lane:

- explicit snapshots
- generation counters
- measurable pressure effects
- no magical or invisible policy edges

## What PlanC Does Not Simplify

PlanC does not materially simplify the kernel TWQ core.

It does not replace:

- kernel admission logic
- scheduler feedback paths
- worker lifecycle management
- blocking compensation
- QoS-aware worker supply
- `_pthread_workqueue_*` bridging

So the reuse is not symmetrical.

The correct direction is:

- GCDX powers TCM
- TCM does not simplify the kernel core

## What PlanC Buys

PlanC's benefits are:

- upstream/open TCM becomes usable on FreeBSD with minimal oneTBB disturbance
- oneTBB integration stays aligned with upstream architecture
- cross-runtime coordination becomes available without inventing a local TCM
  clone
- the platform's kernel-informed pressure facts can make TCM smarter than a
  pure self-reporting model
- `PlanB` can remain deferred unless upstream TCM proves insufficient

This is the cleanest path if:

- there is no deadline
- upstream TCM source really lands
- FreeBSD porting effort remains moderate

## What PlanC Costs

PlanC's costs are:

- dependence on upstream TCM release timing
- dependence on upstream TCM architecture choices
- continued shared-library singleton model
- possible friction if TCM internals assume Linux-oriented `hwloc` or build
  conventions
- possible lack of a clean upstream provider seam for injecting platform
  pressure and capacity facts
- risk that the FreeBSD port becomes "upstream TCM plus a growing local
  patchset"
- less direct ownership than `PlanB`

It is therefore upstream-aligned, not fully platform-owned.

## Preconditions

PlanC only becomes executable when the actual TCM source lands.

As of April 10, 2026:

- the RFC is merged
- the `thread_composability_manager/` source tree is not yet present in the
  local oneTBB clone

So today PlanC is a strategy, not yet an implementation task.

## FreeBSD Porting Assumptions

PlanC assumes the following will likely be true:

- `hwloc` remains the main dependency story
- pthreads and standard C++17 build cleanly
- there are no deep Linux-only kernel dependencies in the core arbitration
  engine
- TCM can degrade gracefully when FreeBSD hybrid-capacity information is weak

The main likely weak point is heterogeneous-core quality:

- TCM is expected to use `hwloc` `cpukinds`
- FreeBSD `cpukinds` is currently weaker than native OS score sources
- hybrid policy quality may lag until `hmp(4)` gets a user-space ABI

That should affect quality, not basic viability.

## Biggest Risks

### Risk 1: upstream TCM may not expose a clean provider seam

The RFC confirms independence and `hwloc` dependence, but it does not confirm a
pluggable provider architecture. If the real source bakes `hwloc` calls and
grant-policy assumptions directly into the arbitration engine, then the FreeBSD
pressure bridge will require local patching rather than clean injection.

This is the most important structural risk in PlanC.

### Risk 2: upstream timeline is still outside project control

The RFC is merged, but the actual source tree is not in the local clone yet.
PlanC is therefore a bet on upstream timing as well as upstream architecture.

Recommended checkpoint:

- reassess upstream landing status in October 2026
- if real source is still absent by then, revisit PlanB formally

### Risk 3: `hwloc` `cpukinds` quality on FreeBSD is only a partial answer

TCM may port and function correctly while still having weak heterogeneous-core
policy on FreeBSD until native `hmp(4)` export exists.

### Risk 4: the bridge can become a maintenance burden

If upstream TCM requires invasive local modifications to consume platform
pressure, then PlanC stops being a clean layering story and starts becoming a
FreeBSD-specific fork against upstream TCM.

## Provider Boundary In PlanC

PlanC still requires explicit provider boundaries.

The upstream TCM code should not become a place where concrete platform facts
are scattered through arbitrary `#ifdef`s if we can avoid it.

The clean shape is:

- topology provider
- capacity provider
- pressure provider

With concrete sources such as:

- `hwloc2`
- FreeBSD-native sysctl / `cpuset`
- future `kern.hmp.*` or equivalent
- private `libthr` / `pthread_workqueue` pressure SPI

If upstream TCM is not already structured this way, the FreeBSD port should
still preserve this separation locally and treat that divergence as an explicit
porting risk, not an incidental detail.

## Pressure Provider SPI

The defining extra step in PlanC, beyond plain upstream TCM porting, is the
pressure provider SPI.

This should be read as a provider interface, not as bidirectional coupling.
The bridge term should be reserved for the `libthr` / `pthread_workqueue`
glue that implements the SPI.

The provider SPI should be:

- private
- one-way
- provider-like
- versioned
- snapshot-first in v1
- optionally event-assisted later if polling proves too stale

It should expose enough for TCM to make smarter budget decisions, for example:

- total active workers
- total admitted workers
- total blocked workers
- aggregate pressure state
- aggregate consumed capacity
- generation counter and timestamp

It should not expose:

- libdispatch queue semantics
- raw kernel scheduler internals
- direct worker control hooks for TCM
- per-QoS bucket detail

Per-QoS detail should stay below the provider SPI. The provider should
translate it
into aggregate pressure and capacity signals rather than leaking GCDX-internal
vocabulary upward into TCM.

Recommended shape:

```c
struct tbbx_pressure_snapshot_v1 {
    uint32_t struct_size;
    uint32_t version;
    uint64_t generation;
    uint64_t timestamp_ns;

    uint32_t ncpu;
    uint32_t total_consumed;
    uint32_t total_requested;
    uint32_t pressure_state;
    uint32_t reserved[8];
};
```

Version by `struct_size` first. The caller passes the size it understands. The
provider fills only what fits. That is the safest initial ABI hygiene model.

The point is to inform TCM, not to make TCM a scheduler.

## Phase Breakdown

### Phase C0: Wait And Monitor

Goals:

- watch for actual TCM source landing
- inspect the upstream source tree and build system immediately when it lands
- verify whether the real source matches the RFC-level expectations

### Phase C0.5: Freeze The Provider Boundary In Parallel

Goals:

- design the topology, capacity, and pressure provider interfaces before
  upstream TCM source lands
- validate those interfaces against existing GCDX machinery
- keep the provider boundary usable by both PlanC and PlanB

This is the one part of PlanC that should not wait on upstream timing.

### Phase C1: Upstream TCM Port On FreeBSD

Goals:

- build upstream/open TCM on FreeBSD
- satisfy its dependencies
- ship it as `libtcm.so.1`
- verify that oneTBB uses it through the existing adaptor path

This is plain PlanA, but now sourced from upstream rather than reimplemented
locally.

### Phase C2: Validate Standalone Upstream TCM

Goals:

- run oneTBB through upstream TCM on FreeBSD
- validate fallback to `market`
- document real FreeBSD limitations
- measure whether TCM alone already gives acceptable coordination

### Phase C3: Add The GCDX Pressure Bridge

Goals:

- expose private pressure facts upward from TWQ / `pthread_workqueue`
- let TCM consume them through a provider boundary
- measure whether kernel-informed pressure materially improves grant quality

This is the distinctive PlanC step.

### Phase C4: Add Native Hybrid-Capacity Input

Goals:

- feed future `hmp(4)` user-space capacity data into the TCM side
- reduce dependence on weak `hwloc` `cpukinds` heuristics on FreeBSD
- improve hybrid-core grant quality on asymmetric systems

### Phase C5: Reassess PlanB

Goals:

- decide whether upstream/open TCM plus native platform inputs is enough
- decide whether direct native `permit_manager` work is still justified

PlanB should only be revived if PlanC shows concrete architectural limits.

## Relationship To PlanA

PlanC includes PlanA, but is not an independent seam.

PlanA is:

- upstream TCM seam
- `libtcm.so.1`
- oneTBB adaptor path

PlanC is:

- PlanA with upstream source and native pressure/capacity inputs from the
  platform below it

So PlanA is the fallback local implementation of the same seam.
PlanC is the active upstream-first version of that seam plus the platform
provider SPI.

## Relationship To PlanB

PlanB remains the native divergence path.

PlanC should be preferred over PlanB when:

- upstream TCM lands and ports cleanly
- the shared-library singleton model is acceptable
- the provider bridge gives TCM the platform data it needs
- upstream alignment matters more than owning the entire coordination layer

PlanB should take over only if:

- upstream TCM proves too rigid
- upstream TCM cannot consume platform facts cleanly enough
- the shared-library model becomes a liability
- the runtime needs tighter integration than TCM permits

## When PlanC Is The Right Choice

PlanC is the right choice when:

- there is no urgent deadline
- upstream TCM source is available
- FreeBSD porting effort is tolerable
- upstream alignment is strategically valuable
- the platform already has meaningful lower-layer machinery worth reusing

## What PlanC Must Not Do

PlanC must not:

- make GCDX depend on TCM
- put permit semantics into the kernel
- collapse policy and mechanism into one state machine
- make TCM the owner of worker creation or scheduler feedback
- assume upstream TCM automatically solves hybrid quality on FreeBSD

## Bottom Line

`TBBX-PlanC` is the layered synthesis path:

- port upstream/open TCM to FreeBSD
- keep it as the user-space coordination broker
- let it ride on top of the platform's existing GCDX / TWQ / future `hmp`
  machinery
- add one-way pressure and capacity bridges upward
- keep PlanB in reserve only if upstream TCM later proves limiting

If upstream TCM lands cleanly, PlanC is the most conservative and elegant
architecture to try first.
