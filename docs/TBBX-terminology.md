# TBBX Terminology

## Purpose

This note fixes the project vocabulary for future design documents so that
runtime layers and ownership boundaries stay stable.

## Names

### `oneTBB`

The upstream open-source codebase.

Use this when referring to:

- the upstream project;
- public source files;
- upstream scheduler and arena behavior;
- upstream portability and bug-fix work.

### `full TBB`

The effective vendor-complete stack:

- `oneTBB`
- proprietary TCM implementation
- vendor-distributed runtime combination

Use this when talking about the shipped complete runtime story, not the
open-source tree alone.

### `TBBX`

This project.

Use this when referring to:

- our platform-native continuation built from `oneTBB`;
- our long-term runtime strategy;
- our platform-specific broker and substrate integration work.

### `GCDX`

Our platform-native continuation of the GCD / `pthread_workqueue` / TWQ line.

Use this when referring to:

- the kernel-backed execution substrate;
- `libthr` / `pthread_workqueue` integration;
- admission, pressure, narrowing, and lifecycle mechanism.

### `TCM`

The immediate oneTBB-facing coordination seam and broker contract.

Use this when referring to:

- permits;
- runtime-level concurrency budgets;
- renegotiation callback flow;
- the `libtcm.so.1` compatibility layer.

## Layer Vocabulary

The preferred three-layer model is:

1. `TBBX` / oneTBB runtime layer
   - scheduler, arenas, tasking semantics
2. `TCM` broker layer
   - inter-runtime budgets, permit lifecycle, renegotiation
3. `GCDX` substrate layer
   - TWQ, `pthread_workqueue`, pressure signals, admission mechanism

## Ownership Summary

### oneTBB / TBBX runtime layer owns

- intra-runtime scheduling
- arena-level allotment
- task stealing and scheduler invariants
- TBB-facing semantics

### TCM broker layer owns

- inter-runtime concurrency ceilings
- permit creation and lifetime
- granted concurrency values
- runtime-level renegotiation

### GCDX substrate layer owns

- worker admission
- pressure and backpressure signals
- block/yield compensation
- worker lifecycle
- topology/capacity mechanism that is specific to the GCDX lane

### Shared infrastructure across TBBX and GCDX

- canonical topology discovery and constraint vocabulary
- `hwloc`-backed CPU mask and NUMA representation
- cross-runtime capacity facts that should not be rederived independently

## Short Rule

Use this sentence as the default mental model:

> GCDX has the stronger execution mechanism, oneTBB has the stronger
> coordination interface, and TCM is where they meet.

## Anti-Confusion Rules

Do not say:

- "TBBX is just libdispatch/GCD"
- "TCM is the kernel boundary"
- "oneTBB is only interface clues"

Prefer:

- "TBBX reuses GCDX substrate where the semantics overlap"
- "TCM is the user-space coordination boundary"
- "oneTBB keeps its runtime identity above the broker"
