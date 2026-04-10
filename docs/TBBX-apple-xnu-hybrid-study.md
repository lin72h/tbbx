# TBBX Apple XNU Hybrid Scheduler Study

Date: 2026-04-10

## Purpose

Study how Apple XNU models heterogeneous CPUs and how its scheduler handles
performance and efficiency cores, with emphasis on lessons that matter for
`TBBX`, `TCM`, `GCDX`, and future FreeBSD `hmp(4)` integration.

This note is based on the local source tree:

- `../nx/apple-opensource-xnu`

## Executive Summary

Apple's XNU solution is not a generic user-space topology library. It is a
kernel-native hybrid scheduling system with three distinct layers:

1. Static topology and perf-level exposure
2. Dynamic performance-control policy
3. Scheduler execution and migration machinery

The important design lesson is that Apple keeps these layers separate.

- Static hardware classes are exposed publicly as `hw.nperflevels` and
  `hw.perflevelN.*`.
- Dynamic steering is handled through perfcontrol / CLPC and scheduler callouts.
- The scheduler uses per-cluster processor sets, thread-group
  recommendations, and explicit asymmetric spill / steal rules.

The strongest architectural lesson is not the static perf-level view. It is the
separate dynamic perfcontrol plane that can change recommendations at runtime
and force prompt scheduler reaction.

For `TBBX`, the main architectural lesson is not to copy XNU's exact APIs, but
to copy its separation of concerns:

- one provider for topology facts
- another provider for dynamic hybrid capacity / preference signals
- scheduler / runtime policy above both

## 1. Static Hardware Model

XNU has an explicit kernel notion of CPU cluster type:

- `CLUSTER_TYPE_SMP`
- `CLUSTER_TYPE_E`
- `CLUSTER_TYPE_P`

Source:

- `osfmk/arm/machine_routines.h:258-264`

On Apple ARM systems, cluster type is discovered from the CPU topology data and
the `cluster-type` property:

- `'E'` maps to `CLUSTER_TYPE_E`
- `'P'` maps to `CLUSTER_TYPE_P`

Source:

- `osfmk/arm64/machine_routines.c:1212-1217`

XNU also assigns user-visible names to those types:

- `Efficiency`
- `Performance`

Source:

- `osfmk/arm/machine_routines_common.c:65-69`

## 2. Public User-Space Topology View

XNU exposes hybrid topology to user space through `hw.nperflevels` and
`hw.perflevelN.*`.

Important details:

- perf levels are ordered in descending performance
- on ARM/ARM64, `perflevel0` corresponds to `P`, `perflevel1` to `E`
- each perf level exposes CPU counts, cache geometry, CPUs-per-cache, and
  cluster name

Sources:

- `bsd/kern/kern_mib.c:239-265`
- `bsd/kern/kern_mib.c:309-430`
- `bsd/kern/kern_mib.c:998-1028`

This is a clean design choice:

- user space gets a stable typed view of heterogeneity
- it does not get raw scheduler internals
- dynamic placement remains elsewhere

## 3. Processor-Set Based Scheduler Model

On AMP systems, XNU builds per-cluster processor sets (`psets`) and uses those
as the scheduler's unit of placement and balancing.

During bootstrap, it initializes cluster IDs and cluster-typed psets from the
machine topology:

- boot pset gets the boot cluster's type and cluster ID
- `sched_num_psets` matches the number of clusters

Source:

- `osfmk/kern/processor.c:290-313`

The scheduler tracks, per CPU, the currently running thread's:

- recommended pset type
- thread group
- perfcontrol class

Source:

- `osfmk/kern/processor.c:587-600`

This means Apple's scheduler is cluster-aware at the core accounting level, not
just at the final CPU-pick stage.

## 4. Eligibility: Who Is Allowed To Use P-Cores?

XNU does not simply say "high priority goes to P-cores." It has an eligibility
layer above the scheduler.

In the classic AMP scheduler path, `recommended_pset_type(thread)` decides
whether a thread is P-recommended or E-recommended. The decision uses:

1. explicit cluster binding override
2. QoS policy for background / utility
3. thread-group recommendation
4. default kernel-vs-user fallback

Important behavior:

- bound threads keep their bound cluster
- background and utility may be forced to E unless perfcontrol says "follow
  group"
- default user-space recommendation is P
- default `kernel_task` recommendation is E

Source:

- `osfmk/kern/processor.c:1904-1955`

The AMP scheduler then applies asymmetric eligibility:

- P-recommended threads may run on either P or E clusters
- E-recommended threads may run on E only

Source:

- `osfmk/kern/sched_amp.c:693-700`

This asymmetry matters. XNU is not modeling "P" and "E" as two equal buckets.
It models P recommendation as an upper capability class and E recommendation as
a strict restriction.

## 5. Thread Groups And Perfcontrol

Thread groups are a first-class object in XNU's scheduling model.

Relevant public signals:

- thread group flags such as `EFFICIENT`, `APPLICATION`, `CRITICAL`,
  `BEST_EFFORT`, `GAME_MODE`
- thread group recommendation

Source:

- `osfmk/kern/thread_group.h:63-113`
- `osfmk/kern/thread_group.h:149`
- `osfmk/kern/thread_group.c:1172-1179`
- `osfmk/kern/thread_group.c:1249-1257`

Thread-group recommendation changes are wired straight into scheduler behavior.
In the AMP path, a recommendation change to `P` immediately bounces the thread
group off E-cores:

Source:

- `osfmk/kern/sched_amp.c:681-689`

For the newer Edge scheduler path, perfcontrol / CLPC can set a preferred
cluster per QoS within a thread group rather than a single global class.

Source:

- `osfmk/kern/sched_clutch.c:1468-1498`
- `osfmk/kern/thread_group.c:1270-1366`

This is a much richer model than static core classification. It means Apple
separates:

- "what the hardware is"
- "what this workload should prefer right now"

## 6. Coalitions Influence P-Core Eligibility

The hybrid decision is not just thread-local or process-local. Apple also ties
it to a higher-level resource-management concept.

The XNU documentation says:

- jetsam coalitions are used by the scheduler and CLPC to determine P-core
  eligibility
- the thread group of a jetsam coalition led by a P-core capable process is
  allowed to use P-cores

Source:

- `doc/observability/coalitions.md:124-130`

This is a major architectural point:

- Apple inserts an entitlement / policy layer between raw hardware classes and
  scheduling
- not every workload is equally entitled to P-core access

For `TBBX`, that is a useful lesson. A future FreeBSD-native hybrid policy may
need a notion of workload entitlement or runtime class above raw `hmp(4)` score
data.

## 7. The Edge Scheduler: Preferred Clusters, Migration, And Spill Rules

XNU's newer "Edge" scheduler generalizes the original AMP design.

Key properties from the design doc:

- per-thread-group preferred cluster recommendations
- preferred cluster can vary by QoS bucket
- scheduler may migrate if the preferred cluster is busy
- migration decisions use an edge matrix between clusters
- steal and rebalance are explicit first-class operations
- long-running AMP workloads use "stir-the-pot" swaps to avoid permanent slow
  core stragglers

Sources:

- `doc/scheduler/sched_clutch_edge.md:182-288`
- `doc/scheduler/sched_clutch_edge.md:288-289`

Important default initialization from the implementation:

- `P -> P`: free spill
- `E -> E`: free spill
- `P -> E`: weighted spill
- `E -> P`: no spill

Source:

- `osfmk/kern/sched_clutch.c:6128-6140`
- `doc/scheduler/sched_clutch_edge.md:210-214`

That is a strong signal that Apple's default policy is asymmetric. But the more
important point is that this is only the initial directed policy shape. CLPC is
allowed to update edge permissions and weights dynamically at runtime. So the
real XNU lesson is not "hardcode asymmetric spill rules." It is "build the
infrastructure for dynamic directional policy and let a separate control plane
drive it."

## 8. Immediate Migration On Policy Change

XNU does not always wait for the next ordinary scheduling point to react to a
new recommendation.

In the Edge scheduler:

- CLPC may update the preferred cluster per QoS
- the scheduler updates those preferences immediately
- it can also migrate running and runnable threads of the thread group right
  away via IPIs and rebalance logic

Source:

- `osfmk/kern/sched_clutch.c:5968-6035`

This matters because it shows Apple treats hybrid policy changes as live control
signals, not just static hints.

## 9. QoS Width And Default Core-Class Policy

The classic AMP scheduler has explicit maximum parallelism policy by QoS.

Notable default behavior:

- realtime width is effectively limited to P-cluster width
- utility defaults to E only
- background / maintenance default to E only
- runtime policy can unlock broader placement

Source:

- `osfmk/kern/sched_amp_common.c:469-519`

This is another example of Apple not treating heterogeneity as a pure topology
problem. Width policy depends on workload class.

## 10. Perfcontrol As A Separate Dynamic Control Plane

Beyond topology sysctls, XNU exposes a separate perfcontrol-related scheduler
plane:

- `kern.sched_recommended_cores`
- a debug/development sysctl to update recommended cores

Source:

- `bsd/kern/kern_sysctl.c:2711-2739`

The stackshot path also respects perfcontrol recommendations:

- CPUs determine whether they are eligible to participate based on perfcontrol
  recommendation
- on AMP systems, a suitable P-core is chosen as the "main" CPU

Source:

- `doc/observability/mt_stackshot.md:16-27`

This reinforces the same split:

- perf levels describe the machine
- perfcontrol describes current preferred use of the machine

## 11. What XNU Is Doing Architecturally

The Apple pattern can be summarized as:

1. Kernel owns heterogenous topology
2. Kernel exports a simple typed user-space view of perf levels
3. A performance controller computes dynamic cluster preferences
4. Scheduler owns pset-local execution, migration, steal, rebalance, and swaps
5. Workload grouping and entitlement sit above the raw hardware model

This is not a `hwloc`-style architecture.

It is closer to:

- static topology API
- dynamic capacity / preference provider
- scheduler policy engine

## 12. Lessons For TBBX

### 12.1 Do Not Collapse Topology And Dynamic Capacity

Apple keeps static hardware description and dynamic placement policy separate.
`TBBX` should do the same.

That argues for:

- one provider for topology / masks / NUMA
- one provider for native dynamic hybrid capacity / score data
- broker / runtime policy above both

### 12.2 A Native Kernel Signal Source Is More Valuable Than Static Heuristics

Apple's interesting behavior is not merely that it knows P versus E. The
valuable part is that perfcontrol can change recommendations dynamically and the
scheduler can react immediately.

That strongly supports the current `TBBX` direction:

- `hwloc2` is acceptable for phase 1 topology
- future FreeBSD-native `hmp(4)` / HFI-like data is the more strategic source
  for dynamic hybrid policy

### 12.3 Hybrid Policy Is Asymmetric

Apple's default scheduler behavior is not "spread evenly across all cores."
It is asymmetric:

- E-native work stays on E
- P-preferred work may spill down if necessary
- migration policy is directional and weighted

This is a useful warning for `TBBX`: hybrid policy should probably not be a
single symmetric score table with no concept of preferred degradation
direction. But the XNU lesson is still dynamic policy, not a frozen table. If
`TBBX` later adopts asymmetric spill rules, they should come from the dynamic
hybrid-capacity provider and broker policy interface rather than from
hardcoded Apple-style constants.

### 12.4 Eligibility Matters

Apple makes P-core eligibility a higher-level policy decision involving thread
groups and coalitions. That suggests a future `TBBX` design might eventually
need:

- runtime class
- workload class
- policy / entitlement inputs

above pure hardware capacity.

### 12.5 Migration And Rebalance Need An Explicit Policy Layer

XNU does not rely on passive "best CPU" selection alone. It has:

- live recommendation changes
- running-thread rebalance
- runnable-thread migration
- directional steal
- stir-the-pot swaps

For `TBBX`, this is a reminder that hybrid policy is not only about initial
grant sizing. It also affects migration and fairness behavior once work is
already running.

## 13. What TBBX Should Borrow And What It Should Not

### Borrow

- The separation between static topology and dynamic policy
- The idea of a small, typed public hardware view
- Dynamic directional spill / steal reasoning
- Immediate response to meaningful recommendation changes
- Workload-group-level policy rather than purely per-thread policy

### Do Not Borrow Literally

- XNU's exact sysctl names
- XNU's CLPC-specific interfaces
- XNU's full scheduler design
- Apple-specific coalition and game-mode policy

`TBBX` should take architecture lessons, not mimic Apple's ABI.

## 14. TBBX Interpretation

The closest TBBX analogue to XNU's split would be:

- `hwloc2`
  static topology / masks / NUMA / general machine shape
- future FreeBSD `hmp(4)` / HFI-like provider
  dynamic hybrid capacity / throttling / performance-efficiency signals
- `TCM` broker
  runtime-level policy and concurrency budgeting
- `GCDX`
  low-level mechanism and pressure feedback

In other words:

- `hwloc2` is closer to XNU's `hw.perflevel*` layer than to CLPC
- native FreeBSD `hmp(4)` would be closer to XNU's dynamic hybrid control
  substrate
- `TCM` is where `TBBX` should combine those sources into runtime-facing
  decisions

## Bottom Line

Apple's XNU solution confirms a design direction that already fits `TBBX`:

- keep topology and dynamic hybrid policy separate
- treat hybrid scheduling as a native platform concern
- do not ask a generic topology library to become the long-term policy oracle

The strongest direct lesson is:

`TBBX` should continue to use `hwloc2` for topology in phase 1, but it should
leave a clean provider seam for a future FreeBSD-native hybrid-capacity source,
because that is where Apple's real advantage lives too.
