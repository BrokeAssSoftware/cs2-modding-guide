---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: System skeleton (GameSystemBase / SystemBase)"
recipe: system-template
technique_family: "Foundational - system skeleton (underlies families L, S)"
diataxis: how-to
source_version: "1.6.0f1 (magic-mail@6fb3d2b; date-pinned, static source only)"
last_reverified: "2026-07-04"
canonical_mods:
  - magic-mail@6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47
  - realistic-path-finding@50645fa6a078181365e36a42e2e27b96699bf02a
  - traffic-tool-essentials@10973595ac9ed37f47ca24eb788d4421dc295fa0
technique_applicability: [core, simulation]
status: source-verified
Created: 2026-07-03
Updated: 2026-07-03
Owners:
  - codex
---

# System skeleton (GameSystemBase / SystemBase)

> A minimal, well-behaved CS2 ECS system: cache queries and system handles in
> `OnCreate`, gate with `RequireForUpdate`, do work in `OnUpdate`, throttle with
> `GetUpdateInterval`, and (optionally) offload to a Burst job with `ScheduleParallel`.

## Problem

You need a DOTS system to hold your mod's simulation logic, but a naive system either
never runs, runs every tick when it only needs to run occasionally, or rebuilds its
queries and lookups each frame. You want the standard shape a source-verified CS2 mod
uses so your system is cheap, gated, and safe to schedule.

This recipe covers the *body* of a system. Scheduling it into the loop is a separate
step - see [Register a system into the update loop](system-registration.md) - and the
cadence/scheduling *concepts* live in
[System Scheduling](../../explanation/system-scheduling.md).

## Solution

Derive from `Game.GameSystemBase` (the CS2 base that exposes `GetUpdateInterval` /
`GetUpdateOffset`; plain Unity `SystemBase` also works but lacks those overrides). Then:

1. **`OnCreate`** - build every `EntityQuery`, `ComponentLookup`, and sibling
   `SystemHandle` *once*, and call `RequireForUpdate` so the system is skipped entirely
   when its inputs are absent.
2. **`OnUpdate`** - do the work; for parallelism, fill a `[BurstCompile]` job struct and
   `ScheduleParallel` it against the cached query.
3. **`GetUpdateInterval`** (optional) - return the tick count between runs so the system
   throttles itself instead of running every simulation tick.

## Steps & Code

### 1. Periodic GameSystemBase skeleton (Magic Mail)

`MagicMailSystem` is a compact end-to-end example: a `GameSystemBase` that throttles
itself, builds one query in `OnCreate`, gates on it, and resolves a sibling vanilla
system once.

```csharp
public partial class MagicMailSystem : GameSystemBase
{
    private EntityQuery m_PostFacilitiesQuery;

    private const int UpdatesPerDay = 32;   // once per ~45 in-game minutes

    public override int GetUpdateInterval(SystemUpdatePhase phase)
    {
#if DEBUG
        return 256;                    // vanilla PostFacilityAISystem interval, for debugging
#else
        return 262144 / UpdatesPerDay; // 8192 ticks; a day is 262144 ticks
#endif
    }

    public override int GetUpdateOffset(SystemUpdatePhase phase) => 48; // slot before vanilla
```
Source: `../../../vice-and-order-research/mods/dossiers/magic-mail/repo/Systems/MagicMailSystem.cs#L33-L74` (@6fb3d2b)

`OnCreate` builds the query with an explicit `EntityQueryDesc` (`All` + `None`) and gates
on it, then caches the vanilla stats system:

```csharp
protected override void OnCreate()
{
    base.OnCreate();

    m_PostFacilitiesQuery = GetEntityQuery(new EntityQueryDesc
    {
        All  = new[] { ComponentType.ReadOnly<PrefabRef>(),
                       ComponentType.ReadOnly<Game.Buildings.PostFacility>(),
                       ComponentType.ReadWrite<Resources>() },
        None = new[] { ComponentType.ReadOnly<Destroyed>(),
                       ComponentType.ReadOnly<Deleted>(),
                       ComponentType.ReadOnly<Temp>() },
    });

    RequireForUpdate(m_PostFacilitiesQuery);   // skip the whole system when no facilities
    TryResolveMailAccumulationSystem();        // cache a sibling system handle once
}
```
Source: `../../../vice-and-order-research/mods/dossiers/magic-mail/repo/Systems/MagicMailSystem.cs#L79-L102` (@6fb3d2b)

The sibling system is cached with `World.GetExistingSystemManaged<T>()` (use
`GetOrCreateSystemManaged<T>()` when the target must be instantiated if absent):

```csharp
m_MailAccumulationSystem = World.GetExistingSystemManaged<MailAccumulationSystem>();
```
Source: `../../../vice-and-order-research/mods/dossiers/magic-mail/repo/Systems/MagicMailSystem.cs#L482` (@6fb3d2b)

`OnUpdate` then reads the cached query on the main thread (`ToEntityArray(Allocator.Temp)`)
and processes facilities (`#L110` onward). For a main-thread system that is the whole
skeleton: cache in `OnCreate`, throttle via `GetUpdateInterval`, work in `OnUpdate`.

### 2. Cache sibling systems + query in OnCreate (Traffic Tool Essentials)

`PatchedTrafficLightSystem` shows the same `OnCreate` discipline at larger scale -
several sibling systems resolved with `GetOrCreateSystemManaged<T>()`, one query cached
and gated:

```csharp
public override int GetUpdateInterval(SystemUpdatePhase phase) => 4;

protected override void OnCreate()
{
    base.OnCreate();
    m_SimulationSystem = base.World.GetOrCreateSystemManaged<SimulationSystem>();
    m_SyncGroupSystem  = base.World.GetOrCreateSystemManaged<SyncGroupSystem>();
    m_EndFrameBarrier  = base.World.GetOrCreateSystemManaged<EndFrameBarrier>();
    m_TimeSystem       = base.World.GetOrCreateSystemManaged<TimeSystem>();
    m_TrafficLightQuery = GetEntityQuery(
        ComponentType.ReadWrite<TrafficLights>(), ComponentType.ReadOnly<UpdateFrame>(),
        ComponentType.Exclude<Deleted>(), ComponentType.Exclude<Destroyed>(),
        ComponentType.Exclude<Temp>());
    RequireForUpdate(m_TrafficLightQuery);
}
```
Source: `../../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Systems/TrafficLightSystems/Simulation/PatchedTrafficLightSystem.cs#L866-L880` (@1097359)

Its `OnUpdate` applies a shared-component filter to the cached query and schedules the
Burst job against it (`#L907-L908`). Note the modern CS2 pattern: query handles are
resolved through `GetComponentTypeHandle` / `GetComponentLookup` at update time and
passed into the job by value (this file uses the compiler-generated `InternalCompilerInterface`
form because it is a decompiled clone; the hand-written equivalent is in step 3).

### 3. Burst job body: IJobChunk (Realistic Path Finding)

For parallel work, put the loop in a `[BurstCompile]` `IJobChunk` and `ScheduleParallel`
it against the cached query. `BicycleOwnerLimiterSystem` is a clean hand-written example:

```csharp
protected override void OnCreate()
{
    base.OnCreate();
    m_EndFrameBarrier = World.GetOrCreateSystemManaged<EndFrameBarrier>();
    m_CitizenQuery = GetEntityQuery(
        ComponentType.ReadOnly<Citizen>(), ComponentType.ReadWrite<BicycleOwner>());
    RequireForUpdate(m_CitizenQuery);
}

protected override void OnUpdate()
{
    var job = new LimitBicycleOwnersJob
    {
        EntityType       = GetEntityTypeHandle(),
        CitizenType      = GetComponentTypeHandle<Citizen>(true),
        BicycleOwnerType = GetComponentTypeHandle<BicycleOwner>(false),
        // ... settings ...
        CommandBuffer    = m_EndFrameBarrier.CreateCommandBuffer().AsParallelWriter(),
    };
    Dependency = job.ScheduleParallel(m_CitizenQuery, Dependency);
    m_EndFrameBarrier.AddJobHandleForProducer(Dependency);   // register the writer
}

[BurstCompile]
private struct LimitBicycleOwnersJob : IJobChunk
{
    [ReadOnly] public EntityTypeHandle EntityType;
    [ReadOnly] public ComponentTypeHandle<Citizen> CitizenType;
    public ComponentTypeHandle<BicycleOwner> BicycleOwnerType;
    public EntityCommandBuffer.ParallelWriter CommandBuffer;

    public void Execute(in ArchetypeChunk chunk, int unfilteredChunkIndex,
                        bool useEnabledMask, in v128 chunkEnabledMask)
    {
        var entities = chunk.GetNativeArray(EntityType);
        var owners   = chunk.GetNativeArray(ref BicycleOwnerType);
        // ... per-entity loop over chunk.Count ...
    }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Systems/BicycleOwnerLimiterSystem.cs#L25-L110` (@50645fa)
(`ScheduleParallel` at `#L66`, producer handle at `#L67`, `IJobChunk` struct at `#L71`,
the four-argument `Execute` at `#L83`.)

Two things worth copying: pass `Dependency` in and assign it back so the job scheduler
chains correctly, and register the command buffer's writer with
`AddJobHandleForProducer(Dependency)` so playback waits on your job.

### 4. IJobChunk vs IJobEntity (be honest about which)

Not every RPF job is `IJobChunk`. Several systems in the same mod use `IJobEntity`, whose
`Execute` takes the components directly instead of a chunk. `CarCongestionEwmaSystem`'s
sampler is one:

```csharp
[BurstCompile]
partial struct SampleJob : IJobEntity
{
    public float DeltaTime;
    [WriteOnly] public NativeQueue<Sample>.ParallelWriter Out;

    void Execute(Entity v, ref LaneSampleState st, in CarCurrentLane cur) { /* ... */ }
}
// scheduled the same way:
Dependency = job.ScheduleParallel(_carsWithLaneQ, Dependency);
```
Source: `../../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Systems/CarCongestionEwmaSystem.cs#L257-L266` (@50645fa)
(`ScheduleParallel` at `#L157`.)

`IJobEntity` is terser for per-entity work; `IJobChunk` gives you the raw chunk and the
enabled-component mask, which you need for manual chunk iteration or enableable
components. Both `ScheduleParallel` against a cached query - pick per job, do not assume
one form. The dedicated write-up of the Burst body is the queued
[`burst-ijobchunk`](burst-ijobchunk.md) recipe.

## Pitfalls & gotchas

- **A system with no `RequireForUpdate` runs every eligible tick even when idle.** Gating
  with `RequireForUpdate<T>()` (or `RequireForUpdate(query)`) is cheaper than an early
  `return` inside `OnUpdate` because the whole system is skipped.
- **Rebuilding queries/lookups per frame.** Build `EntityQuery`, `ComponentLookup`, and
  sibling `SystemHandle` handles once in `OnCreate`; refresh `ComponentLookup` with
  `.Update(this)` inside `OnUpdate`, do not re-`GetComponentLookup`.
- **`GetExistingSystemManaged` can return null.** Magic Mail wraps it in try/catch and
  warns rather than assuming the sibling exists. Use `GetOrCreateSystemManaged` when the
  target must exist; check for null when it might not.
- **`ScheduleParallel` requires no structural changes in the job.** Structural edits go
  through an `EntityCommandBuffer.ParallelWriter` (as in step 3), and you must call
  `AddJobHandleForProducer(Dependency)` on the barrier or playback races your job.
- **Forgetting to chain `Dependency`.** Pass the incoming `Dependency` into
  `ScheduleParallel` and assign the returned handle back to `Dependency`; dropping it
  breaks the job dependency graph.
- **`GameSystemBase` vs `SystemBase`.** `GetUpdateInterval` / `GetUpdateOffset` are
  CS2's `GameSystemBase` additions. A plain Unity `SystemBase` has no cadence override
  and runs every tick of its phase. `Needs Verification (in-game)`: exact tick-to-time
  mapping for a given interval on a non-default day-length setting.

## Variations

- **Main-thread system** - do the work directly in `OnUpdate` (Magic Mail step 1); fine
  for small entity counts or once-per-interval passes.
- **One-shot / apply-on-change** - `GetUpdateInterval => 1` plus `Enabled = false`, enable
  to run once then disable. See the queued
  [`periodic-updateinterval-system`](periodic-updateinterval-system.md) recipe and
  [System Scheduling](../../explanation/system-scheduling.md#tuning-cadence-getupdateinterval-and-getupdateoffset).
- **Parallel `IJobChunk`** - raw chunk access + enabled mask (RPF step 3).
- **Parallel `IJobEntity`** - terser per-entity form (RPF step 4).

## See also

- Related recipes: [Register a system into the update loop](system-registration.md)
  (scheduling the skeleton you build here),
  [ECS system replacement via ordering](ecs-system-replacement-ordering.md)
  (using a custom system to take over vanilla behaviour), `burst-ijobchunk` (queued -
  the Burst job body in depth).
- Concept: [System Scheduling](../../explanation/system-scheduling.md)
  (phases, `GetUpdateInterval`/`GetUpdateOffset`),
  [Multi-Phase Scheduling](../../explanation/multi-phase-scheduling.md).
- Reference: [Technique Index](../../technique-index.md) (families L, S).

## Sources

- Canonical mods (dossier + repo, pinned commits):
  - `magic-mail` @6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47
  - `realistic-path-finding` @50645fa6a078181365e36a42e2e27b96699bf02a
  - `traffic-tool-essentials` @10973595ac9ed37f47ca24eb788d4421dc295fa0
- Official/community references (link out, do not duplicate):
  - Unity DOTS systems / IJobChunk / IJobEntity: https://docs.unity3d.com/Packages/com.unity.entities@latest
  - CS2 modding wiki: https://cs2.paradoxwikis.com/Modding
