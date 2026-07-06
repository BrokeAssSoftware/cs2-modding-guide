---
FrontmatterVersion: 1
DocumentType: Guide
Title: "ECS/DOTS Fundamentals (writing the OnUpdate body)"
Summary: "The DOTS/ECS model an agent needs to write a CS2 system's OnUpdate body correctly - queries, ComponentLookup, EntityCommandBuffer sync points, DynamicBuffer, and the Burst/IJobChunk parallel model - grounded in real mod source."
diataxis: explanation
source_version: "1.6.0f1 (realistic-path-finding@50645fa; date-pinned, static source only)"
last_reverified: "2026-07-04"
status: source-verified
canonical_mods:
  - realistic-path-finding@50645fa6a078181365e36a42e2e27b96699bf02a
  - traffic-tool-essentials@10973595ac9ed37f47ca24eb788d4421dc295fa0
  - magic-mail@6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47
  - magic-garbage-truck@1b6a478753e1ef4e43ac9b90d567f3d7183c7be2
  - realistic-jobsearch@7a096b2ab974bb03cc4cf0937f250bf1d7671f31
  - anarchy@a6311e898d20a775368668b234aaa32f06e3e1eb
  - advanced-road-naming@559e72cdb3180e3e71869643367e094a22eed988
technique_applicability: [core, simulation]
Created: 2026-07-04
Updated: 2026-07-05
Owners:
  - codex
References:
  - Label: System Scheduling (where the system runs)
    Path: ./system-scheduling.md
  - Label: Lifecycle and Initialization (where OnCreate/OnUpdate sit)
    Path: ./mod-lifecycle.md
  - Label: Recipe - Burst IJobChunk (the parallel-jobs how-to)
    Path: ../how-to/recipes/burst-ijobchunk.md
  - Label: Performance and Terminology (reference)
    Path: ../reference/performance-and-terminology.md
---

# ECS/DOTS Fundamentals (writing the OnUpdate body)

The rest of the handbook tells you *where* a system runs - which
[`SystemUpdatePhase`](./system-scheduling.md) it belongs to and *when* in the
[lifecycle](./mod-lifecycle.md) `OnCreate`/`OnUpdate` fire. This page is about the
other half: *what you write inside the body*. Cities: Skylines II is built on Unity
DOTS (Data-Oriented Tech Stack), so every simulation touch - reading a citizen's
state, adding a component, mutating a building's resource buffer - goes through a
small set of ECS primitives. An agent that knows the phase model but not these
primitives will write an `OnUpdate` that compiles and silently corrupts the world.

This is a concept page. It explains *why each primitive exists and when to reach for
it*, grounded in real cited source. The step-by-step of scheduling a Burst job lives
in the how-to companion [`recipes/burst-ijobchunk.md`](../how-to/recipes/burst-ijobchunk.md);
the vocabulary lookup lives in [Performance and Terminology](../reference/performance-and-terminology.md).

## Entity, component, system

DOTS separates *data* from *behaviour*. An **entity** is just an id - a citizen, a
car, a lane, a prefab. A **component** is a plain struct of data attached to an
entity (`Resident`, `CarCurrentLane`, `Resources`). A **system** is code that runs
each simulation tick over the entities matching some shape, reading and writing their
components. Your mod's system derives from `GameSystemBase` (CS2's `SystemBase`
wrapper) and does all its work in `OnUpdate`:

```csharp
public partial class RPFResidentAISystem : GameSystemBase
{
    protected override void OnCreate() { /* build queries, resolve systems */ }
    protected override void OnUpdate() { /* the per-tick body - this page */ }
}
```
[realistic-path-finding repo/RealisticPathFinding/Systems/RPFResidentAISystem.cs#L54](../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Systems/RPFResidentAISystem.cs), commit `50645fa`

The `partial` keyword matters: Unity's source generator emits the other half of the
class (the `__TypeHandle` field, lookup plumbing, and `IJobChunk` boilerplate) at
compile time. You never write those by hand - you *declare* the shapes and let the
generator wire them.

## EntityQuery: declaring the shape you act on

You never loop over "all entities". You declare a **query** - a set of components an
entity must have (`All`), must not have (`None`), or may have (`Any`) - and the
system only touches matching entities. Build queries once in `OnCreate`, not per
frame. Car Congestion's EWMA sampler builds two, using the exclude (`None`) filter to
split "cars that already have my state" from "cars that still need it":

```csharp
_carsMissingStateQ = GetEntityQuery(new EntityQueryDesc
{
    All  = new[] { ComponentType.ReadOnly<CarCurrentLane>() },
    None = new[] { ComponentType.ReadOnly<LaneSampleState>() }
});
_carsWithLaneQ = GetEntityQuery(new EntityQueryDesc
{
    All = new[] { ComponentType.ReadWrite<LaneSampleState>(),
                  ComponentType.ReadOnly<CarCurrentLane>() }
});
```
[realistic-path-finding repo/RealisticPathFinding/Systems/CarCongestionEwmaSystem.cs#L69-L79](../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Systems/CarCongestionEwmaSystem.cs), commit `50645fa`

The `None` list is the exclude filter - the same idea as an `Exclude<T>`/`WithNone<T>`.
It is how you carve out a subset without disabling anyone else, and it is the
backbone of the query-narrowing takeover shape (see
[System Replacement](./system-replacement.md)). Two common variants of the same idea:

- The positional `GetEntityQuery(...)` overload takes `ComponentType.ReadWrite<T>()`
  / `ComponentType.ReadOnly<T>()` for `All` and `ComponentType.Exclude<T>()` for
  `None` inline. RPF's resident query excludes group members and dead/temp/stumbling
  entities this way: `GetEntityQuery(ComponentType.ReadWrite<Resident>(), ...,
  ComponentType.Exclude<GroupMember>(), ComponentType.Exclude<Deleted>(),
  ComponentType.Exclude<Temp>(), ComponentType.Exclude<Stumbling>())`
  ([RPFResidentAISystem.cs#L88](../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Systems/RPFResidentAISystem.cs), commit `50645fa`).
- Magic Mail queries operational post facilities with an `All` of `PrefabRef` +
  `PostFacility` + `Resources` and a `None` of `Destroyed`/`Deleted`/`Temp`
  ([magic-mail repo/Systems/MagicMailSystem.cs#L84-L98](../../vice-and-order-research/mods/dossiers/magic-mail/repo/Systems/MagicMailSystem.cs), commit `6fb3d2b`).

### RequireForUpdate: skip the tick when nothing matches

Pair a query with `RequireForUpdate(query)` in `OnCreate` and the engine skips your
`OnUpdate` entirely when the query is empty - free early-out, no per-frame guard:

```csharp
RequireForUpdate(_carsWithLaneQ);
```
[realistic-path-finding repo/RealisticPathFinding/Systems/CarCongestionEwmaSystem.cs#L100](../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Systems/CarCongestionEwmaSystem.cs), commit `50645fa`

This is the standard gate: a system that owns post facilities or zone-restricted
lines should not run in a city that has none. Magic Mail and Traffic Tool Essentials
both call `RequireForUpdate` on their primary query for exactly this reason
([MagicMailSystem.cs#L100](../../vice-and-order-research/mods/dossiers/magic-mail/repo/Systems/MagicMailSystem.cs), commit `6fb3d2b`;
[DepotZoneDispatchSystem.cs#L294](../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Systems/DepotZone/DepotZoneDispatchSystem.cs), commit `1097359`).
See [Conditional Execution](./conditional-execution.md) for the broader gating model.

## ComponentLookup<T>: random access to another entity's component

A query iterates the entities *you* own. But mid-iteration you often need a component
off some *other* entity - the prefab a car points at, the lane an entity sits on.
That is what `ComponentLookup<T>` is for: a random-access handle keyed by `Entity`.
Fetch it in `OnUpdate` (or refresh a cached one), passing a `bool` that declares
read-only vs read-write access:

```csharp
_carLaneLk = GetComponentLookup<CarCurrentLane>(true);   // true  = read-only
_curveLk   = GetComponentLookup<Curve>(true);
_prefabLk  = GetComponentLookup<PrefabRef>(true);
_roadDataLk= GetComponentLookup<RoadData>(true);
```
[realistic-path-finding repo/RealisticPathFinding/Systems/CarCongestionEwmaSystem.cs#L122-L125](../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Systems/CarCongestionEwmaSystem.cs), commit `50645fa`

The read-only vs read-write distinction is not cosmetic - it is what the job
scheduler uses to decide which systems can run in parallel. Declaring RW when you
only read needlessly serialises against every other reader. Traffic Tool Essentials
refreshes its whole lookup set each frame and is explicit about which are writable:

```csharp
m_TransportLineLookup = GetComponentLookup<TransportLine>(false);  // RW
m_LineZoneLookup      = GetComponentLookup<LineDepotZoneAssignment>(true);  // RO
m_PrefabRefLookup     = GetComponentLookup<PrefabRef>(true);       // RO
m_ServiceDispatchLookup = GetBufferLookup<ServiceDispatch>(false); // RW buffer lookup
```
[traffic-tool-essentials repo/TrafficToolEssentials/Systems/DepotZone/DepotZoneDispatchSystem.cs#L1465-L1473](../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Systems/DepotZone/DepotZoneDispatchSystem.cs), commit `1097359`

A lookup is only valid for the frame it was fetched. If you cache the handle in a
field across frames, you must call `.Update(this)` on it each `OnUpdate` before use so
it re-points at the current chunk layout - the alternative to re-fetching with
`GetComponentLookup` as these mods do. The source-generated systems express the same
cached-handle+refresh pattern through `__TypeHandle` fields threaded with
`CheckedStateRef` (RPF's job assembles ~80 such lookups in its `OnUpdate`,
[RPFResidentAISystem.cs#L185-L243](../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Systems/RPFResidentAISystem.cs), commit `50645fa`).

### The `.Update(this)` staleness Heisenbug

Even a correctly-refreshed cached handle can lie to you. `.Update(this)` only re-points
the handle at the current chunk layout - it does *not* force the job dependency chain to
resolve. When the data you are about to read was written by a job in a *different*
`SystemGroup` and that cross-group dependency has not settled, the handle can hand you
pre-write-back chunk data: intermittent, timing-dependent wrong reads that evaporate
under a debugger - a Heisenbug. Traffic Tool Essentials hit exactly this and worked
around it by dropping `.Update(this)` entirely and re-fetching with `GetComponentLookup`
every frame, which forces a fresh lookup guaranteed to read current chunk data:

```csharp
// V422.30: GetComponentLookup instead of Update(this) - known ECS Heisenbug pattern.
m_CustomTrafficLightsLookup = GetComponentLookup<CustomTrafficLights>(false);
m_TrafficLightsLookup       = GetComponentLookup<Game.Net.TrafficLights>(false);
m_SyncGroupLookup           = GetBufferLookup<SyncGroup>(false); // RW
```
[traffic-tool-essentials repo/TrafficToolEssentials/Systems/TrafficLightSystems/Simulation/SyncGroupSystem.cs#L513-L521](../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Systems/TrafficLightSystems/Simulation/SyncGroupSystem.cs), commit `1097359`

The lesson is not "never cache handles" - it is that `.Update(this)` fixes chunk-layout
staleness, not *dependency* staleness. When a lookup crosses system-group boundaries and
you cannot fully trust the chain, a fresh `GetComponentLookup` is cheaper than chasing
the race.

## EntityCommandBuffer: deferring structural change to a sync point

The single most important rule of an `OnUpdate` body: **do not add, remove, or create
components/entities while you iterate**. Adding or removing a component changes an
entity's archetype - a **structural change** - which moves it to a different memory
chunk and invalidates every query iterator and `ComponentLookup` currently in flight.
Doing it mid-loop corrupts iteration or crashes.

The fix is to *record* the change into an **EntityCommandBuffer (ECB)** and let a
**barrier system** play it back at a well-defined **sync point** (usually end of
frame), after all jobs have finished. You get the ECB from a barrier:

```csharp
// V418.64: All changes via ECB to avoid mixing direct writes with deferred commands.
// Direct ComponentLookup writes + ECB DestroyEntity in same frame caused native crash.
var ecb = m_EndFrameBarrier.CreateCommandBuffer();
```
[traffic-tool-essentials repo/TrafficToolEssentials/Systems/DepotZone/DepotZoneDispatchSystem.cs#L394-L399](../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Systems/DepotZone/DepotZoneDispatchSystem.cs), commit `1097359`

That inline comment is a real war story: TTE hit a native crash from *mixing* direct
`ComponentLookup` writes with deferred `ecb.DestroyEntity` in the same frame, and the
fix was to route **all** structural change through the ECB so nothing mutates the
world until playback. This is the discipline the primitive exists to enforce.

Inside a parallel job you cannot share a plain ECB, so you take a
`ParallelWriter` and pass the chunk index as a sort key. RPF's resident job records
every add/remove this way, and the system registers its job with the barrier so
playback waits for the job to finish:

```csharp
m_CommandBuffer = this.m_EndFrameBarrier.CreateCommandBuffer().AsParallelWriter()
// ... later, inside ResidentTickJob.Execute:
//   this.m_CommandBuffer.AddComponent<PathOwner>(unfilteredChunkIndex, entity, default);
// ... after scheduling:
this.m_EndFrameBarrier.AddJobHandleForProducer(jobHandle2);
```
[realistic-path-finding repo/RealisticPathFinding/Systems/RPFResidentAISystem.cs#L303, #L868, #L311](../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Systems/RPFResidentAISystem.cs), commit `50645fa`

`AddJobHandleForProducer` is the contract that makes deferral safe: it tells the
barrier "do not play back my buffer until this job's handle completes." Skip it and
the barrier can replay a half-written command buffer. See
[Dependency Strategy](./dependency-strategy.md) for how these handles chain.

The ECB is not the only deferred-write barrier in CS2. Several engine systems expose
sibling command-buffer APIs that follow the same record-now / play-back-at-a-sync-point
contract under different names - notably `IconCommandSystem`, whose `IconCommandBuffer`
queues notification-icon add/removes from inside a job. You create it from the system,
hand it to the job struct, and register the job with the same producer handshake:

```csharp
IconCommandBuffer iconBuffer = m_IconCommandSystem.CreateCommandBuffer();
JobHandle cleanHandle = new MagicJob { /* ... */ m_IconCommandBuffer = iconBuffer }
    .ScheduleParallel(m_GarbageProducerQuery, Dependency);
m_IconCommandSystem.AddCommandBufferWriter(cleanHandle);   // = AddJobHandleForProducer
```
[magic-garbage-truck repo/Systems/TotalMagicSystem.cs#L91, #L100-L104](../../vice-and-order-research/mods/dossiers/magic-garbage-truck/repo/Systems/TotalMagicSystem.cs), commit `1b6a478`

`AddCommandBufferWriter` is the IconCommandSystem analogue of the barrier's
`AddJobHandleForProducer` - the same "do not replay until my job finishes" guarantee.
Whenever you see a `CreateCommandBuffer` / `Add...Writer` pair, you are looking at the
ECB pattern wearing a different name.

## DynamicBuffer<T>: per-entity variable-length data

Some data is a *list* per entity, not a single struct - a building's stored
`Resources`, an entity's `PathElement` steps. That is a **DynamicBuffer<T>**: a
resizable array living on the entity. You get it by entity from the `EntityManager`
(or a `BufferLookup<T>`), then index it like an array. Magic Mail reads and rewrites
a post facility's resource buffer directly:

```csharp
if (!entityManager.HasBuffer<Resources>(postEntity)) { /* warn + skip */ }
DynamicBuffer<Resources> resources = entityManager.GetBuffer<Resources>(postEntity);
```
[magic-mail repo/Systems/MagicMailSystem.cs#L155-L161](../../vice-and-order-research/mods/dossiers/magic-mail/repo/Systems/MagicMailSystem.cs), commit `6fb3d2b`

Reading is a plain scan; writing back a mutated element is the important part - buffer
elements are value types, so you must assign the whole struct back to its slot (or
`.Add` a new one), you cannot mutate in place:

```csharp
value.m_Amount = (int)newAmount;
resources[i] = value;        // write the mutated struct back to its slot
// ...
resources.Add(new Resources { m_Resource = resource, m_Amount = amount });
```
[magic-mail repo/Systems/MagicMailSystem.cs#L459-L466](../../vice-and-order-research/mods/dossiers/magic-mail/repo/Systems/MagicMailSystem.cs), commit `6fb3d2b`

Note that *editing* buffer contents (like this) is a normal component write, not a
structural change - it needs no ECB. Only adding/removing the buffer *type* from an
entity is structural. `HasBuffer`-guard before `GetBuffer` so a missing buffer is a
safe skip, exactly as Magic Mail does above.

## Tag components: zero-size coordination signals

Not every component carries data. A **tag component** is an empty struct - zero bytes -
whose *presence on an entity is the message*. Because adding or removing one flips the
entity's archetype (a structural change), a tag is precisely what one system drops so a
*different* system's query picks the entity up next tick. It is the ECS substrate for
cross-system coordination without a shared field or an event bus. Anarchy's
override-prevention feature is built almost entirely from tags: `PreventOverride` is an
empty `IComponentData` marker, while `TransformRecord`/`HeightRangeRecord` snapshot just
the fields needed to restore an object later:

```csharp
public struct PreventOverride : IComponentData, IQueryTypeParameter, IEmptySerializable { }
```
[anarchy repo/Anarchy/Components/PreventOverride.cs#L13](../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/Components/PreventOverride.cs), commit `a6311e8`
(companion records: `TransformRecord.cs#L15`, `HeightRangeRecord.cs#L12`, same commit).

The consuming system queries for its tag plus whatever engine state it must react to,
then edits the world through an ECB. `PreventOverrideSystem` matches entities carrying
both `PreventOverride` and the engine's own `Overridden` tag, strips `Overridden`, and
stamps `Updated` so the pipeline refreshes them:

```csharp
buffer.RemoveComponent<Overridden>(needToPreventOverrideQueryEntities);
buffer.AddComponent<Updated>(needToPreventOverrideQueryEntities);
```
[anarchy repo/Anarchy/Systems/OverridePrevention/PreventOverrideSystem.cs#L64-L65](../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/Systems/OverridePrevention/PreventOverrideSystem.cs), commit `a6311e8`

`Updated` and `BatchesUpdated` are two of the engine's own coordination tags, and the
difference between them is a real idiom worth knowing. `Updated` tells CS2's
`AggregateSystem`/naming pipeline to *reprocess* an entity; `BatchesUpdated` only asks
for a **render/label batch refresh**. Advanced Road Naming exploits the gap: after a
rename it stamps *both* on the owning name entity, but on each child edge it stamps
`BatchesUpdated` **alone** - forcing the label to redraw without feeding `AggregateSystem`
an `Updated` edge that would re-run aggregation:

```csharp
EnsureRefreshTag<Updated>(nameEntity);          // reprocess the name owner
EnsureRefreshTag<BatchesUpdated>(nameEntity);
// ... per child edge:
// BatchesUpdated refreshes label/render batches without feeding AggregateSystem an Updated edge.
EnsureRefreshTag<BatchesUpdated>(edge);
```
[advanced-road-naming repo/Systems/SegmentMetadataSystem.cs#L1660-L1661, #L1671-L1673](../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Systems/SegmentMetadataSystem.cs), commit `559e72c`

Reaching for `BatchesUpdated` without `Updated` is how you trigger a *cosmetic* refresh
without paying for - or accidentally re-triggering - a full simulation reprocess.
Knowing which tag means "recompute me" versus "just redraw me" is the difference between
a cheap label update and an aggregation storm.

## Burst + jobs: the parallel model

`GameSystemBase` gives you the whole city's entities; touching them one-at-a-time on
the main thread does not scale. DOTS runs the work as **jobs** compiled by **Burst**
(a SIMD-optimising, GC-free native compiler) and scheduled across worker threads.
Mark the job struct `[BurstCompile]` and pick a job shape:

- **IJobChunk** - lowest level: you get an `ArchetypeChunk` and iterate its entities
  yourself. Maximum control, most boilerplate. RPF's resident AI is a decompiled
  vanilla clone and uses it:

```csharp
[BurstCompile]
private struct ResidentTickJob : IJobChunk
{
    [ReadOnly] public EntityTypeHandle m_EntityType;
    // ... ~80 lookups/handles ...
    public EntityCommandBuffer.ParallelWriter m_CommandBuffer;
}
```
[realistic-path-finding repo/RealisticPathFinding/Systems/RPFResidentAISystem.cs#L564-L565](../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Systems/RPFResidentAISystem.cs), commit `50645fa`

- **IJobEntity** - higher level: the source generator matches an `Execute` signature
  to a query and iterates for you. Far less boilerplate - reach for this first. Car
  Congestion's sampler declares the components it wants (`ref` = RW, `in` = RO) and
  the generator builds the loop:

```csharp
[BurstCompile]
partial struct SampleJob : IJobEntity
{
    public float DeltaTime;
    [WriteOnly] public NativeQueue<Sample>.ParallelWriter Out;
    // Runs for entities that have both LaneSampleState and CarCurrentLane
    void Execute(Entity v, ref LaneSampleState st, in CarCurrentLane cur) { /* ... */ }
}
```
[realistic-path-finding repo/RealisticPathFinding/Systems/CarCongestionEwmaSystem.cs#L257-L266](../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Systems/CarCongestionEwmaSystem.cs), commit `50645fa`

You then **schedule** the job against a query. `ScheduleParallel` fans it across
worker threads and returns a `JobHandle` you store in `Dependency` so the next system
waits on you. RPF schedules its `IJobChunk` fully parallel over two queries and
chains the handles:

```csharp
JobHandle dependsOn = jobData.ScheduleParallel<ResidentTickJob>(this.m_CreatureQuery,
    JobHandle.CombineDependencies(this.Dependency, jobHandle1));
JobHandle jobHandle2 = jobData.ScheduleParallel<ResidentTickJob>(this.m_GroupCreatureQuery, dependsOn);
this.Dependency = jobHandle2;
```
[realistic-path-finding repo/RealisticPathFinding/Systems/RPFResidentAISystem.cs#L306-L308, #L313](../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Systems/RPFResidentAISystem.cs), commit `50645fa`

### The key tradeoff: parallel schedule vs main-thread run

Not every body should be a parallel job. `ScheduleParallel` is fastest but comes with
strict rules: Burst code cannot touch managed objects, and every write must be
thread-safe (ECB parallel writer, `NativeQueue.ParallelWriter`, etc.). When your
follow-up work *is* inherently main-thread - draining a queue, calling a managed
`PrefabSystem`, doing `EntityManager` structural changes immediately - you schedule
the parallel part, then `.Complete()` the handle to sync back to the main thread and
finish there. Car Congestion does exactly this: parallel sampling, then complete and
drain on the main thread:

```csharp
Dependency = job.ScheduleParallel(_carsWithLaneQ, Dependency);
Dependency.Complete(); // we'll drain queue on main thread right after
```
[realistic-path-finding repo/RealisticPathFinding/Systems/CarCongestionEwmaSystem.cs#L157-L158](../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Systems/CarCongestionEwmaSystem.cs), commit `50645fa`

The choice, in order of preference:

| Approach | When | Cost |
| --- | --- | --- |
| `IJobEntity` + `ScheduleParallel` | pure data work over many entities, thread-safe writes | fastest; strictest rules |
| `IJobChunk` + `ScheduleParallel` | need chunk-level control or a vanilla clone | fastest; most boilerplate |
| Schedule then `.Complete()` + main-thread finish | follow-up needs managed APIs / immediate structural change | serialises the tail; simple |
| `.Run()` / `.WithoutBurst()` | tiny entity counts, or you must call managed code inside | no parallelism; use sparingly |

`Needs Verification (in-game)`: the actual per-frame speedup of parallel scheduling
versus a main-thread `.Run()` is workload- and hardware-dependent and only observable
in a running city; treat the ordering above as a correctness/complexity guide, not a
measured benchmark.

## Reaching a system's private ECS state from a Harmony patch

Sometimes the entity data you need lives in a *private* field of a vanilla system and no
query or lookup exposes it. A Harmony patch can reach in, but *how* it reaches matters
for allocation cost and job safety. Realistic Job Search patches
`PathfindSetupSystem.CompleteSetup` to re-rank a job seeker's pathfind targets, and it
binds the system's private `NativeList` and `JobHandle` fields once via
`AccessTools.FieldRefAccess` - a cached, strongly-typed `ref` accessor - rather than
per-call `Traverse` or raw `FieldInfo.GetValue`:

```csharp
static readonly AccessTools.FieldRef<PathfindSetupSystem, NativeList<PathfindSetupSystem.SetupListItem>> _setupListRef =
    AccessTools.FieldRefAccess<PathfindSetupSystem, NativeList<PathfindSetupSystem.SetupListItem>>("m_SetupList");
static readonly AccessTools.FieldRef<PathfindSetupSystem, Unity.Jobs.JobHandle> _setupDepsRef =
    AccessTools.FieldRefAccess<PathfindSetupSystem, Unity.Jobs.JobHandle>("m_SetupDependencies");
```
[realistic-jobsearch repo/RealisticJobSearch/Patches/Patch_CompleteSetup_FilterJobSeekerTargets.cs#L27-L31](../../vice-and-order-research/mods/dossiers/realistic-jobsearch/repo/RealisticJobSearch/Patches/Patch_CompleteSetup_FilterJobSeekerTargets.cs), commit `7a096b2`

`FieldRefAccess` resolves the field offset once and returns a `ref` to the live field,
so reads and writes are in-place and allocation-free; `Traverse`/`FieldInfo` re-resolve
and box on every call. Two more ECS-specific disciplines live in the same prefix:

- **Force the producing jobs to finish before you touch the native container.** The
  prefix reads `m_SetupList` *before* the original method runs its own completion, so it
  first completes the system's stored dependency by hand - otherwise it races the fill
  jobs still writing that list: `ref var setupDeps = ref _setupDepsRef(__instance);
  setupDeps.Complete();`
  ([same file#L52-L53](../../vice-and-order-research/mods/dossiers/realistic-jobsearch/repo/RealisticJobSearch/Patches/Patch_CompleteSetup_FilterJobSeekerTargets.cs), commit `7a096b2`).
- **Edit native storage in place, no allocations.** It rewrites the target lists through
  no-alloc `UnsafeList<PathTarget>` helpers that move `.Length` and assign `.ElementAt(idx)`
  rather than allocating a fresh list - the same value-type write-back discipline as a
  `DynamicBuffer`, applied to raw native storage
  ([same file#L179-L200](../../vice-and-order-research/mods/dossiers/realistic-jobsearch/repo/RealisticJobSearch/Patches/Patch_CompleteSetup_FilterJobSeekerTargets.cs), commit `7a096b2`).

The takeaway: patching *into* the ECS from Harmony still obeys ECS rules - complete the
dependency before reading a native list, and mutate native storage in place - only now
the compiler is not enforcing them for you.

## Putting it together: the shape of an OnUpdate

1. Early-out (let `RequireForUpdate` gate you; add a `SimulationGuard.IsGameLoading()`
   check if you touch simulation state - both RPF systems do,
   [CarCongestionEwmaSystem.cs#L119](../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Systems/CarCongestionEwmaSystem.cs), commit `50645fa`).
2. Refresh the `ComponentLookup`s you will read/write this frame.
3. Do main-thread prep (init newly-seen entities, read a config singleton).
4. Fill a job struct with handles + lookups + an ECB `ParallelWriter`, `ScheduleParallel`.
5. Either store the handle in `Dependency` (fully parallel) or `.Complete()` and finish
   on the main thread.
6. Register the ECB producer with its barrier (`AddJobHandleForProducer`) so deferred
   structural changes replay safely at the sync point.

## See also

- Concept: [System Scheduling](./system-scheduling.md) - which phase this body runs in
  and why ordering only works within a phase.
- Concept: [Lifecycle and Initialization](./mod-lifecycle.md) - where `OnCreate`/
  `OnUpdate` sit and when queries/lookups are valid.
- Concept: [Dependency Strategy](./dependency-strategy.md) - how `JobHandle`/
  `Dependency` chaining keeps parallel work correct.
- Concept: [System Replacement](./system-replacement.md) - the query-narrowing (`None`
  filter) takeover shape.
- How-to: [Burst IJobChunk](../how-to/recipes/burst-ijobchunk.md) - the step-by-step
  of scheduling a parallel job (recipe).
- Reference: [Performance and Terminology](../reference/performance-and-terminology.md)
  - the vocabulary lookup (chunk, archetype, sync point, structural change).
