# TBBX FreeBSD HMP/HFI Strategy

## Purpose

This note records the strategic significance of the current FreeBSD hybrid-CPU
work for `TBBX`.

The core question is:

- can FreeBSD's emerging native hybrid-capacity and Intel HFI / Thread
  Director work replace the `hwloc2`-based hybrid feature path for `TBBX`?

As of **April 10, 2026**, the answer is:

- **yes for the hybrid-ranking and dynamic score layer**
- **no for the full `hwloc2` topology role**

That distinction is the entire point of this note.

## Executive Verdict

The FreeBSD `HMP` / `intelhfi` work is potentially one of the strongest
platform differentiators for `TBBX`.

It is important because it moves hybrid-core knowledge from:

- user-space heuristics and inferred topology

toward:

- kernel-native capacity
- kernel-native dynamic performance / efficiency scores
- scheduler-facing heterogeneous placement signals

That is more valuable to `TCM` than `hwloc`'s current FreeBSD hybrid story.

But the replacement boundary must be stated accurately:

1. this work can realistically replace `hwloc2` for **hybrid-core ranking and
   dynamic score sourcing**
2. this work does **not** yet replace `hwloc2` for:
   - CPU masks
   - affinity APIs
   - NUMA and memory locality
   - cache/package/socket topology traversal
   - general user-space topology enumeration

So the right TBBX strategy is not:

- replace `hwloc2` entirely

It is:

- replace the **hybrid feature slice** of `hwloc2` with a FreeBSD-native
  provider when that interface becomes usable and stable

## Why This Matters So Much For TBBX

The current TCM research already established:

- Intel's proprietary TCM is clearly hybrid-aware
- the visible hybrid awareness comes through `hwloc` CPU kinds and related
  topology metadata
- direct HFI / Thread Director dependence is not currently evidenced in the
  proprietary TCM binary itself

That means the hybrid policy problem is still open for `TBBX`.

`hwloc2` gives a portable answer, but on FreeBSD it is still the weaker kind of
answer:

- partly heuristic
- partly CPUID-derived
- not obviously tied to live platform performance state

The FreeBSD-native HMP/HFI work aims at a better answer:

- actual kernel-known heterogeneous capacity
- actual HFI / Thread Director-backed score updates
- actual scheduler-facing placement semantics

If that line matures, it gives `TBBX` something stronger than "we copied the
portable Linux path."

It gives:

- a native FreeBSD hybrid-capacity substrate that `TCM` can consume

That is a real selling point.

## The Relevant Review Series

Local patchset directory:

- [freebsd-hybrid-patchsets](/Users/me/wip-gcd-tbb-fx/nx/freebsd-hybrid-patchsets)

Online review pages:

- [D44453](https://reviews.freebsd.org/D44453)
- [D44454](https://reviews.freebsd.org/D44454)
- [D44455](https://reviews.freebsd.org/D44455)
- [D44456](https://reviews.freebsd.org/D44456)
- [D54674](https://reviews.freebsd.org/D54674)

### Review state as of April 10, 2026

- [D44453](https://reviews.freebsd.org/D44453): **Closed**
- [D44454](https://reviews.freebsd.org/D44454): **Accepted**
- [D44455](https://reviews.freebsd.org/D44455): **Needs Revision**
- [D44456](https://reviews.freebsd.org/D44456): **Needs Revision**
- [D54674](https://reviews.freebsd.org/D54674): **Needs Review**

This matters because it tells us the low-level groundwork is farther along than
the full architecture.

## What Each Review Actually Does

### D44453: low-level x86/MSR/CPUID groundwork

Local diff:

- [D44453.diff](/Users/me/wip-gcd-tbb-fx/nx/freebsd-hybrid-patchsets/D44453.diff)

This patch adds:

- Intel HFI / Thread Director-related CPUID bits
- Intel HFI / Thread Director-related MSR definitions
- package thermal MSR definitions used to observe HFI updates

This is the architectural ground truth layer.

It does not provide policy, but it enables the rest of the stack to talk about
real Intel HFI / Thread Director hardware.

### D44454: APIC thermal interrupt path

Local diff:

- [D44454.diff](/Users/me/wip-gcd-tbb-fx/nx/freebsd-hybrid-patchsets/D44454.diff)

This patch wires up:

- local APIC thermal interrupt entry points
- local APIC thermal handler registration
- enable/disable hooks for thermal interrupt consumers

That matters because the proposed Intel HFI driver uses thermal interrupts to
notice updated HFI data.

### D44455: CPU-group score plumbing

Local diff:

- [D44455.diff](/Users/me/wip-gcd-tbb-fx/nx/freebsd-hybrid-patchsets/D44455.diff)

This patch adds score storage to the SMP topology structure:

- `cpu_group.cg_score[class][capability]`

This is the first attempt to let topology carry heterogeneous
performance/efficiency data directly.

Strategically, this is the part most likely to be superseded by `hmp(4)`,
because `D54674` proposes a cleaner machine-independent abstraction.

### D44456: Intel HFI / Thread Director provider driver

Local diff:

- [D44456.diff](/Users/me/wip-gcd-tbb-fx/nx/freebsd-hybrid-patchsets/D44456.diff)

Online summary:

- [D44456 summary](https://reviews.freebsd.org/D44456)

This patch adds:

- a kernel `intelhfi` driver
- HFI table mapping
- HFI / Thread Director enablement through MSRs
- thermal-interrupt-driven table refresh
- copying dynamic scores into kernel CPU topology structures
- a sysctl dump of the HFI / ITD table

This is the most concrete proof that FreeBSD can source dynamic Intel hybrid
scores natively rather than guessing them in user space.

Important limitation:

- the current driver is still tightly Intel-specific and kernel-facing
- it is not yet a stable user-space-consumable abstraction for TBBX

### D54674: machine-independent `hmp(4)` framework

Local diff:

- [D54674.diff](/Users/me/wip-gcd-tbb-fx/nx/freebsd-hybrid-patchsets/D54674.diff)

Online summary:

- [D54674 summary](https://reviews.freebsd.org/D54674)

This is the most important patch in the set.

It introduces:

- `hmp(4)` as a machine-independent heterogeneous multiprocessor framework
- normalized per-CPU capacity
- normalized per-class / per-capability scores
- common scheduler-facing APIs such as:
  - `hmp_highest_capacity_cpu(...)`
  - `hmp_best_cpu(...)`
- an architecture/provider split:
  - providers feed scores
  - scheduler consumes them

The review summary states:

- `hmp(4)` sits between scheduler and provider
- providers may include Intel ITD, device tree, and ACPI CPPC
- the current `coredirector(4)` / `intelhfi` patch is only an example and does
  **not yet use the `hmp` API**

That is the right long-term shape for TBBX.

## What `hwloc2` Gives TBBX Today

`hwloc2` currently gives `TBBX` a portable user-space answer for:

- CPU masks
- complete CPU set enumeration
- affinity queries and binding
- NUMA topology and nodesets
- cache/package/core traversal
- CPU-kind discovery API
- some hybrid ranking metadata such as:
  - `CoreType`
  - `IntelAtom`
  - `IntelCore`
  - efficiency ordering

Relevant public API:

- [cpukinds.h](/Users/me/wip-gcd-tbb-fx/nx/hwloc-2.13.0/include/hwloc/cpukinds.h)

The weakness on FreeBSD is not that `hwloc` is unusable.

The weakness is that FreeBSD hybrid ranking through `hwloc` is still mostly:

- CPUID-backed
- heuristic-backed
- not obviously driven by live kernel-known HFI score updates

That is the exact slice where FreeBSD-native HMP/HFI work is stronger.

## What FreeBSD HMP/HFI Can Replace

If this work matures, it can replace `hwloc2` for:

### 1. Hybrid capacity ranking

The kernel can expose:

- per-CPU capacity
- highest-capacity CPU selection

That is better than asking `hwloc` to infer hybrid order from core type and
frequency heuristics alone.

### 2. Dynamic hybrid score updates

This is the biggest upgrade.

`hwloc` can describe heterogeneous kinds.
It is not designed to be the canonical live runtime score feed for FreeBSD
thread placement.

`intelhfi` + `hmp` aim to provide:

- dynamic score update paths
- class/capability-specific performance/efficiency values
- scheduler-consumable live state

That is much closer to the kind of signal a TCM broker would want when choosing
hybrid-sensitive permit policy.

### 3. Native scheduler alignment

A FreeBSD-native heterogeneous-capacity framework can align:

- kernel scheduling
- thread placement
- future `GCDX` policy
- future `TBBX`/`TCM` broker policy

around one score vocabulary.

That is architecturally cleaner than having:

- kernel one way
- `hwloc` another way
- TCM heuristics a third way

## What FreeBSD HMP/HFI Cannot Replace Yet

This is the crucial constraint.

Even if the review series lands, it still does not replace `hwloc2` for:

### 1. General topology enumeration

`hwloc2` gives:

- package/core/PU tree walking
- cache hierarchy
- object discovery APIs
- topology export/import

The FreeBSD patchset does not currently provide a user-space topology library
with equivalent breadth.

### 2. NUMA and memory locality

`TCM` and `TBBX` still need:

- NUMA node IDs
- CPU-node locality
- memory binding and nodesets

The HMP/HFI work is not a replacement for that.

### 3. Affinity and mask manipulation APIs

`hwloc` gives portable CPU-set helpers and binding APIs.

The patchset does not yet offer an equivalent user-space abstraction that TCM
could adopt as a drop-in topology/mask layer.

### 4. Stable user-space ABI

This is the most important practical gap.

The patchset currently gives:

- kernel fields
- kernel helpers
- a driver sysctl dump

It does not yet give:

- a stable libc/libthr/userland API specifically designed for TBBX or TCM

Without that, it is not ready to replace `hwloc` at the broker boundary.

## The Correct Replacement Boundary

The correct replacement boundary is:

- **replace `hwloc2` for hybrid ranking and dynamic heterogeneous capacity**
- **do not replace `hwloc2` for general topology yet**

Said another way:

- keep `hwloc2` as the topology and mask layer
- let FreeBSD-native HMP/HFI become the hybrid score provider

That is the elegant split.

## Recommended TBBX Strategy

### Phase 1

Keep the current plan:

- use `hwloc2` directly for topology
- build standalone `libtcm.so.1`
- do not block phase 1 on HMP/HFI landing

Reason:

- `hwloc2` already works
- the FreeBSD-native HMP/HFI path is still under review and not yet a stable
  user-space interface

### Phase 1.5

Design a provider seam below the broker that is capable of consuming:

- hybrid capacity
- dynamic performance/efficiency scores

without assuming the source.

That provider can initially be backed by:

- `hwloc2` static hybrid information

and later by:

- FreeBSD-native HMP/HFI signals

This avoids baking `hwloc` heuristics into the broker permanently.

### Immediate design rule

The broker should not call `hwloc2` directly from grant policy code.

Instead, `TBBX` should lock in a provider boundary now:

- a topology/capacity provider interface above concrete data sources
- `hwloc2` as the first provider implementation
- FreeBSD `hmp(4)` as a later provider or score source

The important consequence is architectural, not cosmetic:

- the grant engine should consume abstract capacity/topology queries
- not concrete `hwloc` calls embedded in broker policy paths

That keeps phase 1 simple while preserving a clean future path to:

- static topology from `hwloc2`
- dynamic hybrid scores from native FreeBSD HMP/HFI

The rule is:

- abstract the provider now
- keep `hwloc2` as the initial implementation
- add HMP later without refactoring the broker core

### Phase 2

When a usable FreeBSD-native interface exists, prefer:

- HMP/HFI for hybrid capacity and dynamic score data
- `hwloc2` for topology, masks, NUMA, and binding

At that point the TCM broker can compute better hybrid-aware policy from:

- FreeBSD-native kernel scores
- not `hwloc`'s inferred ranking alone

### Long term

The ideal end state is:

1. `hwloc2` remains a portable topology compatibility layer
2. FreeBSD HMP/HFI becomes the authoritative native heterogeneous-capacity
   signal source
3. `GCDX` and `TBBX` both consume the same native hybrid score substrate

That is exactly the kind of platform coherence that makes TBBX interesting.

## What TBBX Should Ask For From This Work

For this path to become truly useful to TBBX, the missing piece is not more raw
kernel internals.

The missing piece is a small stable user-space boundary.

The most useful future interface would expose:

- system-wide class count
- capability count and bitmap
- per-CPU static capacity
- per-CPU per-class/per-capability dynamic scores
- generation counter or timestamp for updates

Preferably through:

- a stable sysctl family for discovery and debugging
- and/or a small `libthr`/private SPI for efficient runtime consumption

That would let `TCM` consume native FreeBSD hybrid information without
re-implementing kernel logic or scraping debug dumps.

Until that exists, `TBBX` should treat HMP/HFI as an architectural target, not
an immediate runtime dependency.

## Architecture Consequence For TCM

This matters specifically for `TCM`.

The TCM broker does not need a full topology engine at every decision point.
It mainly needs:

- capacity view
- hybrid ranking / score view
- pressure view

So the natural TBBX architecture becomes:

- `hwloc2` provides topology objects and constraints
- FreeBSD HMP/HFI provides native heterogeneous capacity/scores
- `GCDX` / `pthread_workqueue` provides pressure and admission signals
- `TCM` combines them into permit policy

That is a better architecture than forcing `hwloc` to be both:

- topology library
- hybrid score oracle
- live runtime policy source

## Risks And Constraints

### 1. Review risk

The current review states are mixed.

As of April 10, 2026:

- prerequisites are in better shape than the full provider architecture
- the user-space-consumable story is not complete

TBBX should design for this path, but not hard-depend on its immediate landing.

### 2. Intel-specific provider risk

`intelhfi` is Intel-specific.

`hmp(4)` is the real cross-architecture abstraction.

TBBX should align with:

- `hmp` as the stable conceptual layer

not with:

- the raw Intel driver as the permanent interface

### 3. Duplication risk

If TBBX consumes:

- `hwloc` hybrid ranks
- and FreeBSD HMP/HFI ranks

without defining source precedence, policy becomes confused.

The rule should be:

- if native HMP/HFI scores are available and trusted, they override `hwloc`
  hybrid ranking
- `hwloc` remains responsible for topology shape, not authoritative dynamic
  hybrid scoring

## Bottom Line

This FreeBSD work is strategically important enough to deserve explicit TBBX
attention.

The correct statement is:

- **FreeBSD HMP/HFI is a strong candidate to replace `hwloc2`'s hybrid feature
  slice**
- **it is not yet a full replacement for `hwloc2` as a topology layer**

That makes it one of the best long-term TBBX differentiators:

- phase 1 remains portable and `hwloc`-based
- later TBBX can become genuinely FreeBSD-native for hybrid policy
- the result can be better than the generic Linux-style path because it uses
  kernel-native heterogeneous-capacity information instead of heuristics alone

That is the selling point worth building around.
