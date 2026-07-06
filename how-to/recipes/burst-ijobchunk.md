---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: Burst IJobChunk / IJobEntity jobs (DOTS parallelism)"
recipe: burst-ijobchunk
technique_family: "S - Burst IJobChunk / IJobEntity jobs (DOTS parallelism)"
diataxis: how-to
source_version: "1.6.0f1 (realistic-path-finding@50645fa; date-pinned, static source only)"
last_reverified: "2026-07-04"
status: source-verified
canonical_mods:
  - realistic-path-finding@50645fa6a078181365e36a42e2e27b96699bf02a
  - traffic-tool-essentials@10973595ac9ed37f47ca24eb788d4421dc295fa0
  - time2work-realistic-trips@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39
  - anarchy@a6311e898d20a775368668b234aaa32f06e3e1eb
  - write-everywhere@13c70eb04e6bed152257c516a982455148f591a5
  - abandoned-building-remover-deviance-fix@a515bfe588965cbc988cba6a974242f66e5386b7
  - road-speed-adjuster@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed
technique_applicability: [core, simulation]
Created: 2026-07-04
Updated: 2026-07-05
Owners:
  - codex
---

# Burst IJobChunk / IJobEntity jobs (DOTS parallelism)

> Write a Burst-compiled job that runs your per-entity logic across worker threads
> over an `EntityQuery`, and chain its `JobHandle` into the ECS dependency graph so
> it stays thread-safe.

## Problem
You have per-entity work to do every simulation tick - sampling every car, ticking
every resident's AI, tallying every vehicle - and doing it on the main thread in an
`Entities.ForEach` or a plain loop is too slow. CS2's simulation runs on Unity DOTS,
so the fast path is a **Burst-compiled job** scheduled in parallel over the chunks of
an `EntityQuery`, with its `JobHandle` woven into the system's `Dependency` chain so
the ECS safety system serializes conflicting reads/writes for you.

## Solution
Declare a `[BurstCompile]` job struct - `IJobChunk` when you want the raw chunk (full
control, `ComponentTypeHandle`s, enabled-mask), or `IJobEntity` when you just want a
per-entity `Execute(...)` with the components as parameters. Populate its fields
(type handles, `ComponentLookup`s, config values, `ParallelWriter`s) in `OnUpdate`,
then `ScheduleParallel(query, Dependency)` and assign the returned `JobHandle` back to
`this.Dependency`. Register any `EntityCommandBuffer` producer with its barrier. Use
`Schedule` (single background thread) or `.Run` (main thread, still Burst-optimized)
when the work is order-sensitive or you must read the result the same frame.
realistic-path-finding is the reference here: it uses **both** job shapes in one mod.

## Steps & Code

### 1. Declare the job struct with `[BurstCompile]` and `IJobChunk`

`IJobChunk` gives you the whole `ArchetypeChunk`. Its fields are the handles and
lookups the job reads/writes; mark read-only ones `[ReadOnly]` so the safety system
lets them run alongside other readers.

```csharp
[BurstCompile]
private struct ResidentTickJob : IJobChunk
{
    [ReadOnly]
    public EntityTypeHandle m_EntityType;
    [ReadOnly]
    public ComponentTypeHandle<CurrentVehicle> m_CurrentVehicleType;
    // ... more [ReadOnly] handles ...
    public ComponentTypeHandle<Game.Creatures.Resident> m_ResidentType;  // read-write
    public ComponentTypeHandle<Creature> m_CreatureType;
```
Source: `../../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Systems/RPFResidentAISystem.cs#L564-L582` (@50645fa6a078181365e36a42e2e27b96699bf02a)

### 2. In `Execute`, pull the component arrays off the chunk and loop

`IJobChunk.Execute` hands you the chunk, a chunk index (use it to seed per-chunk
randoms and as the command-buffer sort key), and the enabled-mask. Call
`chunk.GetNativeArray(ref handle)` per component, then index all arrays by `i`:

```csharp
public void Execute(
  in ArchetypeChunk chunk,
  int unfilteredChunkIndex,
  bool useEnabledMask,
  in v128 chunkEnabledMask)
{
    NativeArray<Entity> nativeArray1 = chunk.GetNativeArray(this.m_EntityType);
    NativeArray<PrefabRef> nativeArray2 = chunk.GetNativeArray<PrefabRef>(ref this.m_PrefabRefType);
    NativeArray<Creature> nativeArray3 = chunk.GetNativeArray<Creature>(ref this.m_CreatureType);
    NativeArray<Game.Creatures.Resident> nativeArray4 = chunk.GetNativeArray<Game.Creatures.Resident>(ref this.m_ResidentType);
    // ... one GetNativeArray per component, then: for (int index = 0; index < nativeArray1.Length; ++index) ...
```
Source: `../../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Systems/RPFResidentAISystem.cs#L811-L824` (@50645fa6a078181365e36a42e2e27b96699bf02a)

### 3. Fill the job in `OnUpdate` and `ScheduleParallel`, then chain the handle

Assign the handles/lookups/settings, `ScheduleParallel(query, deps)`, and feed the
result forward. Note the pattern: the **second** schedule's dependency is the
**first** job's returned handle, so the two passes run in sequence; the final handle
is registered with the command-buffer barrier and written back to `this.Dependency`:

```csharp
JobHandle dependsOn = jobData.ScheduleParallel<RPFResidentAISystem.ResidentTickJob>(
    this.m_CreatureQuery, JobHandle.CombineDependencies(this.Dependency, jobHandle1));
jobData.m_GroupMember = true;
JobHandle jobHandle2 = jobData.ScheduleParallel<RPFResidentAISystem.ResidentTickJob>(
    this.m_GroupCreatureQuery, dependsOn);
this.m_EndFrameBarrier.AddJobHandleForProducer(jobHandle2);
this.Dependency = jobHandle2;
```
Source: `../../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Systems/RPFResidentAISystem.cs#L306-L313` (@50645fa6a078181365e36a42e2e27b96699bf02a)

`JobHandle.CombineDependencies` merges two upstream handles into one; always pass a
job's `deps` and always assign the last handle back to `this.Dependency` so the next
system waits on you.

### 4. Prefer `IJobEntity` when you just want per-entity args

The same mod's congestion sampler uses `IJobEntity`: no chunk plumbing, no
`GetNativeArray`. You write `Execute` with the components as parameters (`ref` = write,
`in` = read-only) and Burst/source-gen builds the chunk loop for you:

```csharp
[BurstCompile]
partial struct SampleJob : IJobEntity
{
    public float DeltaTime;
    public float MinEmitSec;
    [WriteOnly] public NativeQueue<Sample>.ParallelWriter Out;

    // Runs for entities that have both LaneSampleState and CarCurrentLane
    void Execute(Entity v, ref LaneSampleState st, in CarCurrentLane cur)
    {
        Entity laneNow = cur.m_Lane;
        if (laneNow == Entity.Null || laneNow == st.Lane) { st.ElapsedSec += DeltaTime; return; }
        if (st.Lane != Entity.Null && st.ElapsedSec >= MinEmitSec)
            Out.Enqueue(new Sample { Owner = st.Lane, TravelSec = st.ElapsedSec });
        st.Lane = laneNow; st.ElapsedSec = 0f;
    }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Systems/CarCongestionEwmaSystem.cs#L257-L287` (@50645fa6a078181365e36a42e2e27b96699bf02a)

An `IJobEntity` struct must be `partial` (source-gen writes the `IJobChunk` half).
Its query is inferred from the `Execute` parameters, so you schedule it the same way:
`Dependency = job.ScheduleParallel(query, Dependency);`
Source: `../../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Systems/CarCongestionEwmaSystem.cs#L157` (@50645fa6a078181365e36a42e2e27b96699bf02a)

### 5. Choose the scheduler: `ScheduleParallel` vs `Schedule` vs `.Run`

- **`ScheduleParallel(query, deps)`** - splits the query's chunks across worker
  threads. The safety system rejects it if two parallel writers can touch the same
  data; that is the guarantee. Both examples above use it.
- **`Schedule(deps)` / `Schedule(len, batch)`** - one background thread; use when the
  work must not race itself. traffic-tool-essentials schedules an `IJobParallelFor`
  over an entity array with a batch size, then blocks the same frame:

  ```csharp
  // V418.35: Schedule AND Complete in same frame - no memory leak possible!
  job.Schedule(entities.Length, 64).Complete();
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Systems/Transit/VehicleStatsSystem.cs#L772-L773` (@10973595ac9ed37f47ca24eb788d4421dc295fa0)

- **`.Run()`** - runs on the main thread, but still Burst-compiled (fast). Use for a
  small aggregation whose result you need immediately. traffic-tool-essentials runs a
  Burst `IJob` this way, and marks it `CompileSynchronously = true` so there is no
  slow managed first-run:

  ```csharp
  [BurstCompile(CompileSynchronously = true)]
  public struct AggregateVehicleStatsJob : IJob
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Systems/Transit/VehicleStatsSystem.cs#L128-L129` (@10973595ac9ed37f47ca24eb788d4421dc295fa0)

  ...scheduled with `aggregateJob.Run();`
  Source: `../../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Systems/Transit/VehicleStatsSystem.cs#L871` (@10973595ac9ed37f47ca24eb788d4421dc295fa0)

## Pitfalls & gotchas

- **Cached type handles / lookups MUST be refreshed every `OnUpdate`.** If you store
  `ComponentTypeHandle<T>` or `ComponentLookup<T>` as system fields (instead of
  fetching them fresh each frame), they carry a stale version and Burst will read
  garbage or trip the safety system. Call `.Update(...)` on every one before you
  assign it into the job. traffic-tool-essentials refreshes each lookup at the top of
  its update:

  ```csharp
  // Update all lookups
  m_PersonalCarLookup.Update(this);
  m_TaxiLookup.Update(this);
  m_DeliveryTruckLookup.Update(this);
  m_CarTrailerLookup.Update(this);
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Systems/Transit/VehicleStatsSystem.cs#L728-L732` (@10973595ac9ed37f47ca24eb788d4421dc295fa0)

  The overload differs by system base type: a managed `SystemBase` takes
  `.Update(this)` (as above); an unmanaged `ISystem` takes `.Update(ref state)`. The
  anarchy and realistic-path-finding examples avoid the trap differently - they fetch
  handles fresh in `OnUpdate` (`SystemAPI.GetComponentTypeHandle<...>()` /
  source-gen), so there is nothing stale to refresh.

- **`[BurstCompile]` is often wrapped in `#if BURST` on purpose.** Two of the
  canonical mods gate the attribute behind a compile symbol so they can build a
  **non-Burst** configuration - Burst-compiled code is far harder to step through in a
  debugger, and disabling it surfaces exceptions with real stack traces. anarchy does
  this on its culling job:

  ```csharp
  #if BURST
          [BurstCompile]
  #endif
          private struct VerifyVisibleJob : IJobChunk
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/Systems/OverridePrevention/PreventCullingSystem.cs#L156-L159` (@a6311e898d20a775368668b234aaa32f06e3e1eb)

  write-everywhere wraps even the `using Unity.Burst;` the same way
  (`BelzontWE/Templates/WEPrefabTemplateFilterJob.cs#L12-L23`). Ship with `BURST`
  defined; drop it only to debug. traffic-tool-essentials leans the other way - it
  keeps Burst on and marks its synchronously-run aggregation job
  `CompileSynchronously = true` so first use has no managed fallback
  (`VehicleStatsSystem.cs#L124-L129`).

- **Disabling `[BurstCompile]` is not a free debug switch - it can leak
  `JobTempAlloc`.** The `#if BURST` gate above lets you build without Burst for real
  stack traces, but Burst does more than speed up code. traffic-tool-essentials turned
  Burst *off* on a parallel job, found it leaked temp allocations, and re-enabled it -
  leaving the reason in a comment right above the attribute:

  ```csharp
  // V284: BurstCompile RE-ENABLED - disabling caused memory leaks (JobTempAlloc)
  [BurstCompile]
  public struct UpdateTrafficLightsJob : IJobChunk
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Systems/TrafficLightSystems/Simulation/PatchedTrafficLightSystem.cs#L31-L35` (@10973595ac9ed37f47ca24eb788d4421dc295fa0)

  Keep the toggle for occasional debugging, but ship (and normally run) with Burst on:
  a non-Burst build can behave differently, temp-allocator lifetime included.

- **`ScheduleParallel` enforces read/write safety - respect it.** The safety system
  will throw at schedule time if a parallel job could write the same component two
  threads reach at once. That is why write-access fields are plain and read-access
  fields are `[ReadOnly]` (see step 1): mislabel a written component `[ReadOnly]` and
  you corrupt data; over-claim write access and the scheduler serializes or rejects
  you. Fields that legitimately alias (a component you also read elsewhere in the
  same job) need `[NativeDisableContainerSafetyRestriction]` - realistic-path-finding
  applies it to `m_HumanType` and `m_TargetType`
  (`RPFResidentAISystem.cs#L583-L587`); use it only when you have proven the access is
  actually safe.

- **Writing entities from a parallel job needs a `ParallelWriter` + a barrier + a sort
  key.** You cannot call `EntityManager` from a job. Create an
  `EntityCommandBuffer.ParallelWriter`, pass `unfilteredChunkIndex` as the sort key,
  and register the job with the barrier's producer list. anarchy's culling job adds a
  component this way (`buffer.AddComponent<Updated>(unfilteredChunkIndex, currentEntity)`,
  `PreventCullingSystem.cs#L186`) after `AddJobHandleForProducer`
  (`PreventCullingSystem.cs#L152`). Skipping `AddJobHandleForProducer` lets the buffer
  play back before the job finishes - a race.

- **Exact worker-thread scheduling, chunk-split sizes, and real-world speedup are
  runtime behaviour.** The code proves the *shape* (parallel, Burst, dependency-
  chained) but the measured cost/benefit on a given city is `Needs Verification
  (in-game)`. The "~0.05ms for 50k vehicles" and "~10x faster" figures are the mod
  authors' own comments, not something provable from source
  (`VehicleStatsSystem.cs#L741, #L861`).

## Variations

- **`IJobParallelFor` over a `NativeArray<Entity>` instead of a query.** When you have
  already materialized an entity array (e.g. `query.ToEntityArray`), an
  `IJobParallelFor` with `Execute(int index)` and `ComponentLookup`s is simpler than
  chunk plumbing:

  ```csharp
  [BurstCompile]
  public struct CollectVehicleStatsJob : IJobParallelFor
  {
      [ReadOnly] public NativeArray<Entity> VehicleEntities;
      [ReadOnly] public ComponentLookup<Game.Vehicles.Taxi> TaxiLookup;
      [WriteOnly] public NativeArray<VehicleStatsRawData> Results;
      public void Execute(int index) { var entity = VehicleEntities[index]; /* ... */ }
  }
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Systems/Transit/VehicleStatsSystem.cs#L40-L58` (@10973595ac9ed37f47ca24eb788d4421dc295fa0)

  The trade-off: random `ComponentLookup` access is slower than the sequential
  `GetNativeArray` you get inside `IJobChunk`, but you avoid declaring every type
  handle.

- **Hand-written `IJobChunk` (not source-gen).** anarchy's `VerifyVisibleJob` is a
  clean, non-decompiled example of the full chunk shape - `GetNativeArray`, a
  `for (i < chunk.Count)` loop, and a `ParallelWriter` command buffer - scheduled with
  a single `ScheduleParallel(m_CullingInfoQuery, Dependency)`
  (`PreventCullingSystem.cs#L151, #L173-L189`). Read it if the decompiled
  realistic-path-finding source is hard to follow.

- **Split one job across two queries with mutated state between schedules.**
  realistic-path-finding reuses the *same* struct instance for two passes, flipping a
  `bool` field (`m_GroupMember`) between the two `ScheduleParallel` calls so a single
  job body handles both solo and grouped creatures (step 3). Cheaper than two structs.

- **Single-threaded `IJob` over a materialized chunk list (no `IJobChunk`, no
  `ScheduleParallel`).** When the per-entity work is cheap-but-structural (marking
  sub-buildings / sub-nets / sub-lanes `Deleted`) and parallelism buys nothing,
  abandoned-building-remover skips the chunk-split entirely. It snapshots the query
  into a `NativeList<ArchetypeChunk>` with `ToArchetypeChunkListAsync`, hands that plus
  a plain (non-parallel) `EndFrameBarrier` command buffer to a `[BurstCompile] IJob`,
  and `Schedule`s it on one background thread:

  ```csharp
  job.m_entityCommandBuffer = _endFrameBarrier.CreateCommandBuffer();
  job.m_abandonedBuildingsChunk = _abandonedBuildingQuery.ToArchetypeChunkListAsync(
      World.UpdateAllocator.ToAllocator, out _);
  JobHandle handle = job.Schedule(Dependency);
  _endFrameBarrier.AddJobHandleForProducer(handle);
  Dependency = handle;
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/abandoned-building-remover-deviance-fix/repo/AbandonedBuildingRemoverSystem.cs#L57-L64` (@a515bfe588965cbc988cba6a974242f66e5386b7)

  The `IJob.Execute` walks the chunk list itself (`for i < list.Length` ->
  `chunk.GetNativeArray(handle)` -> `for j < array.Length`) and writes through a plain
  `EntityCommandBuffer` - no `.ParallelWriter`, no sort key, because one thread plays
  the buffer back in order.
  Source: `../../../vice-and-order-research/mods/dossiers/abandoned-building-remover-deviance-fix/repo/AbandonedBuildingRemoverSystem.cs#L111-L127` (@a515bfe588965cbc988cba6a974242f66e5386b7)

- **Tag-driven reapply: gate on `Updated`, re-write, then strip `Updated`.** To keep a
  custom value alive across the engine's own recalculations, road-speed-adjuster
  queries edges that carry its `CustomSpeed` component AND the engine's `Updated` tag
  (`All = CustomSpeed`, `Any = Updated`), re-applies the stored speed, then removes
  `Updated` through the `ParallelWriter` so the edge settles until the game touches it
  again:

  ```csharp
  if (this.EntityManager.HasComponent<CustomSpeed>(entity)) {
      var customSpeed = this.EntityManager.GetComponentData<CustomSpeed>(entity);
      SetSpeed(entity, customSpeed.m_Speed);
      CommandBuffer.RemoveComponent<Updated>(unfilteredChunkIndex, entity);
  }
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/Systems/RoadSpeedApplySystem.cs#L78-L84` (@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed)

  `Updated` is the ECS "this changed this frame" marker; using it as the query gate
  means the job only touches edges the engine just rebuilt, and stripping it stops
  re-processing next frame. Query at `RoadSpeedApplySystem.cs#L24-L34`, scheduled with
  `ScheduleParallel` at `#L49`.

- **Stagger per-entity work across frames with a frame-hash gate.** When a job visits
  many entities every frame but each only needs an occasional refresh, spread the load
  by processing an entity only when the low bits of its index match the low bits of the
  frame counter. write-everywhere skips an unchanged entity unless
  `(entity.Index & MASK) == (frameCount & MASK)`, so ~1/32 of entities refresh per
  frame (mask `0x1f`):

  ```csharp
  if (m_geomEntitiesLastFrame.Contains(entity)
      && (entity.Index & WEConstants.RENDERER_FRAME_CHECK_MASK) != (frameCount & WEConstants.RENDERER_FRAME_CHECK_MASK)
      && (!isAtWeEditor || entity != m_selectedEntity)) {
      m_unmodifiedEntities.Add(entity); return;
  }
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/write-everywhere/repo/BelzontWE/Systems/WEPreCullingSystem.cs#L295` (@13c70eb04e6bed152257c516a982455148f591a5)

  `RENDERER_FRAME_CHECK_MASK = 0x1f` is a 1/32 duty cycle
  (`../../../vice-and-order-research/mods/dossiers/write-everywhere/repo/BelzontWE/Utils/WEConstants.cs#L25`);
  `frameCount` is passed into the job as a field (`WEPreCullingSystem.cs#L202`, set from
  `UnityEngine.Time.frameCount` at `#L123`). Larger mask = spread over more frames
  (cheaper, staler); smaller = fresher, costlier. A live editor-selected entity is
  exempted so it always updates.

## See also
- Explanation: [ECS fundamentals](../../explanation/ecs-fundamentals.md) (the
  primitives: entities, components, chunks, queries),
  [system scheduling](../../explanation/system-scheduling.md) (where `Dependency`
  chains fit in the frame).
- Reference: [performance and terminology](../../reference/performance-and-terminology.md).
- Case study demonstrating it:
  [realistic-path-finding](../../case-studies/realistic-path-finding.md).

## Sources
- Canonical mods (dossier + repo):
  - `realistic-path-finding` @50645fa6a078181365e36a42e2e27b96699bf02a - `repo/RealisticPathFinding/Systems/RPFResidentAISystem.cs`, `repo/RealisticPathFinding/Systems/CarCongestionEwmaSystem.cs`
  - `traffic-tool-essentials` @10973595ac9ed37f47ca24eb788d4421dc295fa0 - `repo/TrafficToolEssentials/Systems/Transit/VehicleStatsSystem.cs`
  - `anarchy` @a6311e898d20a775368668b234aaa32f06e3e1eb - `repo/Anarchy/Systems/OverridePrevention/PreventCullingSystem.cs`
  - `write-everywhere` @13c70eb04e6bed152257c516a982455148f591a5 - `repo/BelzontWE/Templates/WEPrefabTemplateFilterJob.cs`, `repo/BelzontWE/Systems/WEPreCullingSystem.cs`, `repo/BelzontWE/Utils/WEConstants.cs`
  - `abandoned-building-remover-deviance-fix` @a515bfe588965cbc988cba6a974242f66e5386b7 - `repo/AbandonedBuildingRemoverSystem.cs`
  - `road-speed-adjuster` @e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed - `repo/Systems/RoadSpeedApplySystem.cs`
  - `time2work-realistic-trips` @d42921ffb1f6bbbf2c2b086bb17727bf0f637c39 (family S coverage; DOTS scheduling)
- Official/community references (link out, do not duplicate):
  https://docs.unity3d.com/Packages/com.unity.entities@latest ,
  https://cs2.paradoxwikis.com/Modding
