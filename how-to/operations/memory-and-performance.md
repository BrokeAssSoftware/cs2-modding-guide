---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Manage Memory and Performance in a CS2 Mod"
Summary: How to keep a DOTS/ECS mod cheap and leak-free - match the allocator to the lifetime, dispose every native collection, keep allocations out of per-frame paths, use Burst jobs correctly, and profile before you optimize.
diataxis: how-to
source_version: "1.6.0f1 (realistic-path-finding@50645fa; date-pinned, static source only)"
last_reverified: "2026-07-04"
canonical_mods:
  - realistic-path-finding@50645fa6a078181365e36a42e2e27b96699bf02a
  - write-everywhere@13c70eb04e6bed152257c516a982455148f591a5
  - traffic-tool-essentials@10973595ac9ed37f47ca24eb788d4421dc295fa0
technique_applicability: [core, simulation]
status: source-verified
Created: 2026-07-03
Updated: 2026-07-05
Owners:
  - codex
References:
  - Label: System Scheduling (phases, cadence, RequireForUpdate)
    Path: ../../explanation/system-scheduling.md
  - Label: "Recipe: System skeleton (Burst job body, ScheduleParallel)"
    Path: ../recipes/system-template.md
  - Label: Security and Stability (never throw in OnUpdate, no per-frame spam)
    Path: ./security-and-stability.md
  - Label: Technique Index
    Path: ../../technique-index.md
---

# Manage Memory and Performance in a CS2 Mod

Cities: Skylines II runs on Unity DOTS/ECS. Your mod's systems share a hot simulation
loop with the whole city, and DOTS *native collections* (`NativeArray`, `NativeList`,
`NativeQueue`, `NativeParallelHashMap`, `BlobAssetReference`, `NativeSlice`) live in
unmanaged memory the garbage collector will not clean up for you. You own their lifetime.
Get that wrong and you leak memory, corrupt data across a job boundary, or stall every
frame with allocations.

This page is a set of habits to apply as you write systems, and a review checklist for
contributions. It assumes the system shape from
[Recipe: System skeleton](../recipes/system-template.md) and the scheduling model in
[System Scheduling](../../explanation/system-scheduling.md).

---

## 1. Match the allocator to the lifetime

Every native collection takes an `Allocator`. Pick it by how long the data must live -
picking wrong is either a leak or a use-after-free:

- **`Allocator.Temp`** - one frame / one scope, freed automatically at the end of the
  frame. Use it for main-thread scratch such as `ToEntityArray` inside `OnUpdate`. Wrap it
  in a `using` so it disposes even on an early return.
- **`Allocator.TempJob`** - lives across a scheduled job, up to ~4 frames; you must dispose
  it after the job completes (or hand the dispose to the job handle).
- **`Allocator.Persistent`** - lives until you dispose it. Use it for state a system keeps
  between updates; allocate once in `OnCreate` and dispose in `OnDestroy`.

Realistic Path Finding's congestion system shows the two long-lived vs. per-frame extremes
in one file. It allocates its cross-frame state as `Persistent` in `OnCreate`:

```csharp
_sampleQueue    = new NativeQueue<Sample>(Allocator.Persistent);
_lastDensityAdd = new NativeParallelHashMap<Entity, float>(4096, Allocator.Persistent);
_agg            = new NativeParallelHashMap<Entity, Agg>(4096, Allocator.Persistent);
```
Source: `../../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Systems/CarCongestionEwmaSystem.cs#L81-L83` (@50645fa)

...but its per-frame scratch is `Temp`, scoped with `using` so it is released the moment
the block ends:

```csharp
using (var newCars = _carsMissingStateQ.ToEntityArray(Allocator.Temp))
{
    foreach (var v in newCars) { /* ... init component ... */ }
}
// ... later, another one-frame scratch array:
using var lanes = _agg.GetKeyArray(Allocator.Temp);
```
Source: `../../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Systems/CarCongestionEwmaSystem.cs#L138-L145`, `#L176` (@50645fa)

A collection whose data must survive a `ScheduleParallel` uses `TempJob`. RPF's turn-bias
system allocates the job's output queues as `TempJob` right before scheduling:

```csharp
var turnResults = new NativeQueue<TurnResult>(Allocator.TempJob);
var densResults = new NativeQueue<DensityResult>(Allocator.TempJob);
```
Source: `../../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Systems/CarTurnAndHierarchyBiasSystem.cs#L189-L190` (@50645fa)

## 2. Dispose every native collection you allocate

The allocator determines *how* you release it, but you must always release it. Three
idioms cover almost every case:

- **`Persistent` -> guarded dispose in `OnDestroy`.** Check `IsCreated` first so a
  half-initialized system does not throw on teardown:

  ```csharp
  protected override void OnDestroy()
  {
      if (_sampleQueue.IsCreated)    _sampleQueue.Dispose();
      if (_lastDensityAdd.IsCreated) _lastDensityAdd.Dispose();
      if (_agg.IsCreated)            _agg.Dispose();
      base.OnDestroy();
  }
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Systems/CarCongestionEwmaSystem.cs#L103-L109` (@50645fa)

- **`Temp` -> `using`.** As in section 1: the `using` statement disposes at scope exit,
  including on an early `return` or an exception.

- **`TempJob` -> dispose after the job, or hand the dispose to the handle.** Dispose it
  directly once you have `Complete()`d, or pass the job handle to `Dispose(jobHandle)` so
  the disposal is scheduled after the job:

  ```csharp
  // after draining the queues on the main thread:
  turnResults.Dispose();
  densResults.Dispose();
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Systems/CarTurnAndHierarchyBiasSystem.cs#L276-L278` (@50645fa)

Build in Debug: the DOTS safety system logs a warning naming any native collection that
was allocated and never disposed. Treat that warning as a failed build, not noise.

## 3. Keep allocations out of per-frame paths

Allocating inside `OnUpdate` (or any per-tick method) means allocating tens of times a
second. Move the work out:

- **Cache handles in `OnCreate`.** Build `EntityQuery`, `ComponentLookup`, and sibling
  `SystemHandle` handles once in `OnCreate`; refresh a `ComponentLookup` in `OnUpdate` with
  `.Update(this)` rather than calling `GetComponentLookup` again. This is the standard
  system skeleton - see [Recipe: System skeleton](../recipes/system-template.md).
- **Gate the whole system with `RequireForUpdate`.** A system whose required query or
  singleton is absent is skipped entirely - cheaper than allocating and early-returning
  inside `OnUpdate`. See [System Scheduling](../../explanation/system-scheduling.md).
- **Throttle periodic work with `GetUpdateInterval`.** Do not run a once-per-in-game-hour
  pass every simulation tick.
- **Reuse a `Persistent` collection instead of reallocating.** RPF's congestion system
  keeps one `Persistent` aggregator and calls `.Clear()` each pass rather than allocating a
  fresh map per frame (`CarCongestionEwmaSystem.cs`, the `_agg.Clear()` before the drain
  loop).
- **Stagger per-entity work across frames.** When a check must eventually run for every
  entity but need not run for *every* entity *every* frame, gate it on a cheap bitmask of
  the entity index against the frame counter. Write Everywhere's pre-culling job only
  re-tests an unchanged entity when its index and the frame count agree in their low five
  bits, spreading the per-entity cost over ~32 frames instead of paying it all at once:

  ```csharp
  if (m_geomEntitiesLastFrame.Contains(entity)
      && (entity.Index & WEConstants.RENDERER_FRAME_CHECK_MASK)
         != (frameCount & WEConstants.RENDERER_FRAME_CHECK_MASK)
      && (!isAtWeEditor || entity != m_selectedEntity))
  {
      m_unmodifiedEntities.Add(entity);   // skip re-render this frame
      return;
  }
  ```
  `RENDERER_FRAME_CHECK_MASK` is `0x1f` (31), so the index/frame comparison partitions
  entities into 32 frame-buckets. Source:
  `../../../vice-and-order-research/mods/dossiers/write-everywhere/repo/BelzontWE/Systems/WEPreCullingSystem.cs#L295-L299`,
  `../../../vice-and-order-research/mods/dossiers/write-everywhere/repo/BelzontWE/Utils/WEConstants.cs#L25` (@13c70eb)

## 4. Use Burst jobs correctly

Parallel work goes in a `[BurstCompile]` job (`IJobChunk` or `IJobEntity`) scheduled with
`ScheduleParallel` against a cached query. The full, cited job body - dependency chaining,
`EntityCommandBuffer.ParallelWriter`, `AddJobHandleForProducer` - lives in
[Recipe: System skeleton](../recipes/system-template.md#3-burst-job-body-ijobchunk-realistic-path-finding).
The performance-relevant rules:

- **Mark inputs `[ReadOnly]`.** A job that only reads a `ComponentTypeHandle` /
  `ComponentLookup` should declare it `[ReadOnly]` so the scheduler can run it in parallel
  with other readers instead of serializing on a false write dependency.
- **No structural changes inside a parallel job.** Add/remove component and create/destroy
  entity go through an `EntityCommandBuffer.ParallelWriter`, played back later by a barrier
  system.
- **Chain `Dependency`.** Pass the incoming `Dependency` into `ScheduleParallel` and assign
  the returned handle back to `Dependency`; dropping it breaks the dependency graph and
  causes races or redundant `Complete()` stalls.
- **Register the writer.** Call `AddJobHandleForProducer(Dependency)` on the command
  buffer's barrier so playback waits on your job.
- **Do not force a `Complete()` unless you need the result on the main thread this frame.**
  An unnecessary `Complete()` throws away the parallelism you scheduled for.

**Do not capture an `Entity` or `ComponentLookup` into an async closure.** Copy the values
you need into local fields of the job struct before scheduling; a captured reference read
on a worker thread after the source has moved is a data race, not a value read.

- **Keep `[BurstCompile]` on, and never use `Allocator.Temp` inside a parallel job.** Two
  related leaks show up in Traffic Tool Essentials' patched traffic-light job. Disabling
  `[BurstCompile]` on the job leaked `JobTempAlloc` allocations, so the attribute was
  re-enabled with a comment recording exactly that:

  ```csharp
  // V284: BurstCompile RE-ENABLED - disabling caused memory leaks (JobTempAlloc)
  [BurstCompile]
  public struct UpdateTrafficLightsJob : IJobChunk
  ```
  And a per-chunk scratch `NativeList` had to move off `Allocator.Temp` - a `Temp`
  allocation made on a worker thread inside a parallel job leaks rather than freeing at
  frame end, so it uses `Allocator.TempJob` instead:

  ```csharp
  // V284: CRITICAL FIX - Allocator.Temp causes memory leak in parallel jobs!
  // Must use Allocator.TempJob for worker thread allocations
  NativeList<Entity> laneSignals = new NativeList<Entity>(30, Allocator.TempJob);
  ```
  Source:
  `../../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Systems/TrafficLightSystems/Simulation/PatchedTrafficLightSystem.cs#L31-L35`,
  `#L118-L120` (@1097359)

## 5. Profile before you optimize

Measure first - do not guess which system is hot.

- **Unity Profiler.** Attach it to a running game and read the ECS/systems view to find
  which system dominates a tick and where allocations happen. `ProfilerMarker` scopes
  around a suspect code path give you named samples in that view (`Unity.Profiling`; see
  the [Unity Profiler docs](https://docs.unity3d.com/Manual/Profiler.html)).
- **Run against a stress save.** Profile the heavy [regression saves](./logging-and-debugging.md)
  that actually exercise your feature, not an empty map - a per-entity cost is invisible at
  small entity counts.
- **Throttle verbose work in Release.** A common tactic (seen in RPF and Magic Mail) is a
  `#if DEBUG` branch that runs a system far more often while debugging and throttles it in
  Release via `GetUpdateInterval`. Keep diagnostics and profiling markers out of the shipped
  hot path.

> `Needs Verification (in-game)`: the actual per-frame cost, allocation counts, and the
> tick-to-wall-clock mapping of a given `GetUpdateInterval` depend on the machine, the save,
> and the current game build. Confirm any specific number by profiling the running game;
> do not quote a figure you have not measured.

## Pitfalls & gotchas

- **Leaked `Persistent` collection.** Allocated in `OnCreate`, never disposed in
  `OnDestroy` - leaks across every load/unload cycle. The Debug safety system names it.
  A real instance in the wild: Write Everywhere's `WEPreCullingSystem` initializes three
  `Persistent` collections as fields - a `NativeQueue<WERenderData>` and two
  `NativeParallelHashSet<Entity>` - but its `OnDestroy` disposes only `m_availToDraw` and
  leaks all three. Source:
  `../../../vice-and-order-research/mods/dossiers/write-everywhere/repo/BelzontWE/Systems/WEPreCullingSystem.cs#L46-L57` (@13c70eb)
- **Disposing a `TempJob` before its job finishes.** Dispose after `Complete()`, or use
  `Dispose(jobHandle)` so the free is scheduled after the job.
- **Re-`GetComponentLookup` each frame** instead of caching in `OnCreate` and `.Update(this)`
  in `OnUpdate`.
- **Per-frame `ToEntityArray(Allocator.Temp)` on a huge query** when a Burst job over the
  query would do - `Temp` is cheap but the main-thread loop is not.
- **Throwing out of `OnUpdate`.** An exception on the hot path fires every frame; contain
  it (see [Security and Stability](./security-and-stability.md)).
- **Assuming `GetExistingSystemManaged<T>()` is non-null.** It can return null; guard it or
  use `GetOrCreateSystemManaged<T>()` when the target must exist.

## See also

- Concept: [System Scheduling](../../explanation/system-scheduling.md) - phases, cadence,
  and `RequireForUpdate` gating.
- Recipe: [System skeleton](../recipes/system-template.md) - the cited Burst job body and
  `OnCreate`/`OnUpdate` discipline this page depends on.
- Operations: [Security and Stability](./security-and-stability.md) - never throw or spam
  the log on a per-frame path; [Testing and Recovery](./testing-and-recovery.md) - profiling
  regression saves before release.
- Reference: [Technique Index](../../technique-index.md) (families L, M, S).

## Sources

- Canonical mods (dossier + repo, pinned commit):
  `realistic-path-finding` @50645fa6a078181365e36a42e2e27b96699bf02a;
  `write-everywhere` @13c70eb04e6bed152257c516a982455148f591a5 (native-collection leak +
  frame-hash staggering, `WEPreCullingSystem.cs` / `WEConstants.cs`);
  `traffic-tool-essentials` @10973595ac9ed37f47ca24eb788d4421dc295fa0 (Burst/JobTempAlloc
  and worker-thread `Temp` leaks, `PatchedTrafficLightSystem.cs`).
- Official references (link out, do not duplicate): Unity DOTS allocators / native
  collections - https://docs.unity3d.com/Packages/com.unity.collections@latest ; Unity
  Profiler - https://docs.unity3d.com/Manual/Profiler.html
