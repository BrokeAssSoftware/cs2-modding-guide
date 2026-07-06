---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: ECS structural cascade deletion of an entity + its sub-graph"
recipe: ecs-cascade-deletion
technique_family: "AF - ECS structural cascade deletion of an entity + its sub-graph"
diataxis: how-to
source_version: "~1.6.0f1 (abandoned-building-remover-deviance-fix@a515bfe; date-pinned, static source only)"
last_reverified: "2026-07-05"
status: source-verified
canonical_mods:
  - abandoned-building-remover-deviance-fix@a515bfe588965cbc988cba6a974242f66e5386b7
technique_applicability: [core, simulation]
Created: 2026-07-05
Updated: 2026-07-05
Owners:
  - codex
---

# ECS structural cascade deletion of an entity + its sub-graph

> Delete an entity **and** its whole owned sub-graph (sub-areas, sub-nets,
> sub-lanes) by tagging each with `Deleted`, so vanilla cleanup runs and no orphaned
> infrastructure is left dangling.

## Problem
You want to remove a composite object - a building that owns lots, fences, driveways,
parking lanes, and pathfinding lanes - and have it disappear *cleanly*. If you tag
only the top-level entity `Deleted`, the child infrastructure it references through
its `SubArea` / `SubNet` / `SubLane` buffers is orphaned: the building is gone but its
lots and lanes persist as unreachable ghosts. You need to delete the parent **and**
every child it owns, in one sweep, without hand-rolling refund/economy/notification
logic.

## Solution
Do not destroy entities directly. Add the `Deleted` tag to the parent **and** to every
child listed in its owned buffers, then let the game's own end-of-frame cleanup
consume the tags. `Deleted` is an idempotent marker: adding it is a structural change
deferred through an `EntityCommandBuffer` (or `EntityManager`), and the vanilla
deletion/cleanup systems that run afterward handle refunds, notifications, and the
actual archetype teardown. The whole recipe is: query the parents, walk each owned
buffer (`SubArea`, `SubNet`, `SubLane`), tag every entry, then tag the parent. Because
you only ever *add a tag* and never *read child state*, the order you visit the
buffers in does not matter.

## Steps & Code

### 1. Query the parents, excluding already-deleted/temp entities

Build an `EntityQuery` over the identifying components (`Abandoned` + `Building`) and
exclude `Deleted` and `Temp` so you never re-process an entity mid-teardown or touch
a placement preview:

```csharp
_abandonedBuildingQuery = GetEntityQuery(new EntityQueryDesc()
{
    All = new ComponentType[]
    {
        ComponentType.ReadOnly<Abandoned>(),
        ComponentType.ReadOnly<Building>()
    },
    None = new ComponentType[]
    {
        ComponentType.ReadOnly<Deleted>(),
        ComponentType.ReadOnly<Temp>()
    }
});
```
Source: `../../../vice-and-order-research/mods/dossiers/abandoned-building-remover-deviance-fix/repo/AbandonedBuildingRemoverSystem.cs#L28-L40` (@a515bfe588965cbc988cba6a974242f66e5386b7)

`RequireForUpdate(_abandonedBuildingQuery)` then keeps `OnUpdate` idle until at least
one parent matches
(`../../../vice-and-order-research/mods/dossiers/abandoned-building-remover-deviance-fix/repo/AbandonedBuildingRemoverSystem.cs#L49`,
@a515bfe588965cbc988cba6a974242f66e5386b7).

### 2. (Managed path) Walk each owned buffer, tag every child, then tag the parent

The straightforward path iterates the query on the main thread through
`EntityManager`. Guard every buffer read with `TryGetBuffer` so an entity with no lots
or no lanes is a safe skip, tag each child entry, and tag the parent **last**:

```csharp
if (EntityManager.TryGetBuffer<SubArea>(entity, false, out var subareas))
    foreach (var subArea in subareas)
        EntityManager.AddComponent<Deleted>(subArea.m_Area);

if (EntityManager.TryGetBuffer<SubNet>(entity, false, out var subnets))
    foreach (var net in subnets)
        EntityManager.AddComponent<Deleted>(net.m_SubNet);

if (EntityManager.TryGetBuffer<SubLane>(entity, false, out var sublanes))
    foreach (var lane in sublanes)
        EntityManager.AddComponent<Deleted>(lane.m_SubLane);

EntityManager.AddComponent<Deleted>(entity);
```
Source: `../../../vice-and-order-research/mods/dossiers/abandoned-building-remover-deviance-fix/repo/AbandonedBuildingRemoverSystem.cs#L73-L100` (@a515bfe588965cbc988cba6a974242f66e5386b7)

The child entity lives in the buffer element's field: `SubArea.m_Area`,
`SubNet.m_SubNet`, `SubLane.m_SubLane`. You never touch geometry or economy here -
adding `Deleted` is the entire operation; vanilla systems do the rest.

### 3. (Burst path) Same scrub through an EndFrameBarrier command buffer

For the Burst-compiled variant you cannot use `EntityManager` inside a job, so switch
to `BufferLookup` reads and an `EntityCommandBuffer` from the `EndFrameBarrier`. Set
the job up in `OnUpdate`; note the query is captured as a chunk list via
`ToArchetypeChunkListAsync` and the lookups are requested writable (`false`):

```csharp
AbandonedBuildingRemoverJob job = default;
job.m_entityTypeHandle = SystemAPI.GetEntityTypeHandle();
job.m_entityCommandBuffer = _endFrameBarrier.CreateCommandBuffer();
job.m_abandonedBuildingsChunk =
    _abandonedBuildingQuery.ToArchetypeChunkListAsync(World.UpdateAllocator.ToAllocator, out _);
job.m_subLaneLookup = GetBufferLookup<SubLane>(false);
job.m_subNetLookup  = GetBufferLookup<SubNet>(false);
job.m_subAreaLookup = GetBufferLookup<SubArea>(false);
JobHandle handle = job.Schedule(Dependency);
_endFrameBarrier.AddJobHandleForProducer(handle);
Dependency = handle;
```
Source: `../../../vice-and-order-research/mods/dossiers/abandoned-building-remover-deviance-fix/repo/AbandonedBuildingRemoverSystem.cs#L55-L64` (@a515bfe588965cbc988cba6a974242f66e5386b7)

`_endFrameBarrier` is resolved once in `OnCreate` via
`World.GetOrCreateSystemManaged<EndFrameBarrier>()`
(`../../../vice-and-order-research/mods/dossiers/abandoned-building-remover-deviance-fix/repo/AbandonedBuildingRemoverSystem.cs#L43`,
@a515bfe588965cbc988cba6a974242f66e5386b7).

### 4. Inside the job: iterate chunks, tag children then parent

The job body is a plain single-threaded `IJob` (not `IJobChunk`, not
`ScheduleParallel`). It loops the captured chunks, pulls the entity array off each
chunk, and for each entity does the same `TryGetBuffer` -> tag-children -> tag-parent
sequence as the managed path - only through `BufferLookup` and the command buffer:

```csharp
var entity = nativeArray[j];
if (m_subAreaLookup.TryGetBuffer(entity, out var subAreas))
    foreach (var entry in subAreas)
        m_entityCommandBuffer.AddComponent<Deleted>(entry.m_Area);

if (m_subLaneLookup.TryGetBuffer(entity, out var subLanes))
    foreach (var entry in subLanes)
        m_entityCommandBuffer.AddComponent<Deleted>(entry.m_SubLane);

if (m_subNetLookup.TryGetBuffer(entity, out var subNet))
    foreach (var entry in subNet)
        m_entityCommandBuffer.AddComponent<Deleted>(entry.m_SubNet);

m_entityCommandBuffer.AddComponent<Deleted>(nativeArray[j]);
```
Source: `../../../vice-and-order-research/mods/dossiers/abandoned-building-remover-deviance-fix/repo/AbandonedBuildingRemoverSystem.cs#L128-L153` (@a515bfe588965cbc988cba6a974242f66e5386b7)

### 5. Throttle the sweep

This is structural churn, so it does not need to run every frame. The system returns a
long update interval during game simulation:

```csharp
public override int GetUpdateInterval(SystemUpdatePhase phase) =>
    phase == SystemUpdatePhase.GameSimulation ? 16 : 1;
```
Source: `../../../vice-and-order-research/mods/dossiers/abandoned-building-remover-deviance-fix/repo/AbandonedBuildingRemoverSystem.cs#L107` (@a515bfe588965cbc988cba6a974242f66e5386b7)

## Pitfalls & gotchas

- **Idempotent tagging is what makes order irrelevant - lean on it.** The two paths in
  this mod visit the child buffers in *different* orders: the managed path is
  `SubArea -> SubNet -> SubLane`
  (`AbandonedBuildingRemoverSystem.cs#L73-L98`) while the Burst path is
  `SubArea -> SubLane -> SubNet`
  (`AbandonedBuildingRemoverSystem.cs#L129-L151`, both
  @a515bfe588965cbc988cba6a974242f66e5386b7). This is harmless precisely because each
  step only **adds** the `Deleted` tag and never **reads** child state. If your cascade
  read a child's data (position, capacity, "is this already scheduled") and branched on
  it, visit order would suddenly matter. Keep the operation a pure idempotent set-add
  and you get order-independence for free.

- **A child reachable from two parents gets tagged twice - benign.** If two buildings
  both list the same sub-lane, both sweeps add `Deleted` to it. Adding a component that
  is already present is a no-op set operation, so double-tagging costs nothing and
  corrupts nothing. Do **not** add "already-tagged?" bookkeeping to avoid it; that is
  the fragile-read pattern the idempotent design exists to avoid.

- **ECB deferral means mid-sweep churn can delete a resurrected entity.** Both paths
  defer the structural change (the managed path through `EntityManager` during
  iteration, the Burst path through the `EndFrameBarrier` command buffer played back at
  end of frame). The query snapshot is captured before playback, so if an entity is
  repaired/un-abandoned between capture and playback it is still tagged `Deleted` from
  the stale snapshot. Exact timing of when a repair would race this sweep is
  `Needs Verification (in-game)`.

- **`ToArchetypeChunkListAsync` allocator lifetime.** The chunk list is allocated from
  `World.UpdateAllocator.ToAllocator`
  (`AbandonedBuildingRemoverSystem.cs#L58`,
  @a515bfe588965cbc988cba6a974242f66e5386b7), a rewindable per-update allocator - you
  do **not** dispose it yourself, but its contents are only valid for this update, so
  the job must complete this frame (it does: the handle feeds
  `AddJobHandleForProducer`). Do not stash that list across frames.

- **Single-threaded by design.** The job is `IJob` scheduled with `job.Schedule(...)`
  (`AbandonedBuildingRemoverSystem.cs#L62`,
  @a515bfe588965cbc988cba6a974242f66e5386b7), not `IJobChunk`/`ScheduleParallel`. All
  three buffer lookups are requested writable (`GetBufferLookup<...>(false)`), and
  writable lookups across parallel workers invite aliasing hazards; keeping it
  single-threaded sidesteps that. For the volumes here (abandoned buildings per
  16-tick sweep) that is the right trade.

- **You are trusting vanilla cleanup to finish the job.** This recipe only tags; it
  never issues refunds, notifications, or archetype teardown. That those vanilla
  `Deleted`-consuming systems actually run and fully collect the sub-graph is
  `Needs Verification (in-game)` - the source proves the tags are added, not what the
  game does with them.

## Variations

- **Managed vs Burst is a compile switch, not two designs.** The same class ships both
  paths behind `#if USE_BURST`
  (`AbandonedBuildingRemoverSystem.cs#L54-L104`,
  @a515bfe588965cbc988cba6a974242f66e5386b7): `EntityManager.AddComponent<Deleted>` on
  the main thread, or `EntityCommandBuffer.AddComponent<Deleted>` from a job. Start with
  the managed path (simpler to reason about); move to the Burst/ECB path only when the
  per-frame entity count justifies it.

- **Different owned-buffer set.** The three buffers here (`SubArea`, `SubNet`,
  `SubLane`) are what a *building* owns. A different composite entity exposes a
  different set of "sub-" buffers; the pattern is identical - enumerate whatever owned
  buffers the parent carries and tag each entry. Confirm the buffer types your target
  actually holds before naming them.

- **Different identifying query.** Swap the `All` set (`Abandoned` + `Building`) for
  whatever selects your deletion targets; keep the `None` = `Deleted` + `Temp`
  exclusion so you never re-enter teardown or touch previews.

## See also
- Related recipes: [event-driven ModificationEnd](event-driven-modificationend.md)
  (reacting to structural changes at the right barrier).
- Tooling: [sub-element removal](../tooling/sub-element-removal.md) (removing owned
  child elements as a tool operation).
- Reference: [ECS fundamentals](../../explanation/ecs-fundamentals.md) (queries,
  command buffers, `Deleted` semantics).
- Case study demonstrating it:
  [abandoned-building-remover](../../case-studies/abandoned-building-remover.md).

## Sources
- Canonical mods (dossier + repo):
  - `abandoned-building-remover-deviance-fix` @a515bfe588965cbc988cba6a974242f66e5386b7 - `repo/AbandonedBuildingRemoverSystem.cs`
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding
