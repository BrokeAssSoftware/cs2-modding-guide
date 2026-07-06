---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: Durable write to vanilla network components (survives uninstall)"
recipe: durable-network-component-write
technique_family: "AN - Durable write to vanilla network components (survives uninstall)"
diataxis: how-to
source_version: "1.6.0f1 (road-speed-adjuster@e0c0c0b; date-pinned, static source only)"
last_reverified: "2026-07-05"
status: source-verified
canonical_mods:
  - road-speed-adjuster@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed
technique_applicability: [infrastructure, core]
Created: 2026-07-05
Updated: 2026-07-05
Owners:
  - codex
---

# Durable write to vanilla network components (survives uninstall)

> Change a live per-segment network value (a lane speed limit) by writing the game's
> OWN net component on the live lane entity - not a prefab - so the game's built-in
> entity serialization bakes it into the save and the edit outlives the mod.

## Problem
You want to change a network value on ONE placed segment, not on every instance of a
prefab, and you want that change to be part of the save the same way a vanilla edit is
- present after save/load, and still present if the player later removes your mod.
Prefab-field override (family A) is the wrong tool here: it mutates the shared template
and its effect disappears the instant your mod stops re-applying it. This recipe writes
the vanilla component that already lives on the live entity, so the game persists it for
you.

## Solution
Resolve the live lane entities under an edge (its `SubLane` buffer), read each lane's
vanilla `Game.Net.CarLane` / `Game.Net.TrackLane` struct, overwrite ONLY the speed
fields, and `SetComponentData` it back. Because those are the game's own components on
the game's own entities, the game's entity serialization writes them into the save with
no work from you. Keep a private modded marker component (`CustomSpeed`) on the edge to
carry the target value and drive re-apply/overlay, and mirror the value into a JSON
side-car keyed by entity so you can reset later. The durable part - the actual speed
limit - lives entirely in vanilla data, which is what lets it survive uninstall.

## Steps & Code

### 1. Tag the edges you changed and re-apply on `Updated`

The apply system queries edges that carry your marker AND the vanilla `Updated` tag,
then schedules a parallel job. `Updated` is the game's "this entity was regenerated"
signal, so this fires exactly when the game rebuilds an edge's lanes (which would
otherwise wipe your speed back to the prefab default):

```csharp
_entitiesToRestoreQuery = GetEntityQuery(new EntityQueryDesc
{
    All = new[] { ComponentType.ReadOnly<CustomSpeed>() },
    Any = new[] { ComponentType.ReadOnly<Updated>() }
});
_commandBufferSystem = World.GetOrCreateSystemManaged<EndSimulationEntityCommandBufferSystem>();
```
Source: `../../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/Systems/RoadSpeedApplySystem.cs#L24-L36` (@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed)

### 2. Re-apply the stored speed, then strip `Updated`

Inside the `IJobChunk`, for each entity that still has the marker, push its speed back
onto the lanes and remove `Updated` via the command buffer so the entity is not
re-processed next frame:

```csharp
if (this.EntityManager.HasComponent<CustomSpeed>(entity))
{
    var customSpeed = this.EntityManager.GetComponentData<CustomSpeed>(entity);
    SetSpeed(entity, customSpeed.m_Speed);
    CommandBuffer.RemoveComponent<Updated>(unfilteredChunkIndex, entity);
}
```
Source: `../../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/Systems/RoadSpeedApplySystem.cs#L78-L84` (@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed)

> **Source oddity - `[BurstCompile]` + `EntityManager` together.** In this mod `RestoreSpeedJob`
> is declared `[BurstCompile] IJobChunk` (`RoadSpeedApplySystem.cs#L55`) yet carries an
> `EntityManager` field and calls `EntityManager.HasComponent`/`GetComponentData`/`GetBuffer`
> inside `Execute` (the snippets above and in Steps 3-4). Burst cannot compile managed
> `EntityManager` access, so this is a real tension - the job runs against the managed path for
> those calls, not truly Burst-compiled. For a genuinely Burst-compiled job, drop the
> `EntityManager` field and read/write via `ComponentLookup<CarLane>` / `BufferLookup<SubLane>`
> handles refreshed each frame (see [ECS fundamentals](../../explanation/ecs-fundamentals.md)).
> The code above is faithful to source; treat "Burst-compiled" here as aspirational.

### 3. Walk the `SubLane` buffer and convert to CS2 speed units

The value stored on the marker is km/h. CS2 speed fields are NOT m/s and NOT km/h -
they are **2x m/s**, so the conversion from km/h is `/ 1.8` (km/h -> m/s is `/ 3.6`,
then x2). Do NOT reassign the buffer element - only touch the lane entity it points to:

```csharp
// Convert km/h to game units (2x m/s) -> divide by 1.8, not 3.6
float speedGameUnits = speedKmh / 1.8f;

var subLanes = this.EntityManager.GetBuffer<SubLane>(entity);
for (int i = 0; i < subLanes.Length; i++)
    SetSpeedSubLane(entity, subLanes[i].m_SubLane, speedGameUnits);
    // DON'T reassign subLanes[i] - it breaks connectivity!
```
Source: `../../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/Systems/RoadSpeedApplySystem.cs#L92-L105` (@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed)

### 4. Write ONLY the speed fields on the vanilla lane component (flags-preserving)

This is the durable write. Snapshot `m_Flags`, set the two speed fields, restore
`m_Flags`, and commit with `SetComponentData` onto the LIVE lane entity. Preserving the
flags is the discipline that keeps this a surgical edit of vanilla data rather than a
corruption of it:

```csharp
if (this.EntityManager.HasComponent<CarLane>(laneEntity))
{
    var carLane = this.EntityManager.GetComponentData<CarLane>(laneEntity);
    var originalFlags = carLane.m_Flags;          // snapshot
    carLane.m_DefaultSpeedLimit = speedGameUnits;  // ONLY modify speed fields
    carLane.m_SpeedLimit        = speedGameUnits;
    carLane.m_Flags = originalFlags;               // restore
    this.EntityManager.SetComponentData(laneEntity, carLane);
}
else if (this.EntityManager.HasComponent<Game.Net.TrackLane>(laneEntity))
{
    var trackLane = this.EntityManager.GetComponentData<Game.Net.TrackLane>(laneEntity);
    trackLane.m_SpeedLimit = speedGameUnits;
    this.EntityManager.SetComponentData(laneEntity, trackLane);
}
```
Source: `../../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/Systems/RoadSpeedApplySystem.cs#L112-L136` (@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed)

`CarLane` and `TrackLane` are `Game.Net` components (`using Game.Net;`,
`Systems/RoadSpeedApplySystem.cs#L8`). Because they are vanilla, the game serializes
them into the save on its own - you do not register anything for that.

### 5. Carry the target value in a modded marker component

The marker is a modded `IComponentData` added to the EDGE entity, holding the target
speed. It also implements `ISerializable` so the author's intent is for it to ride along
in the game's entity save:

```csharp
public struct CustomSpeed : IComponentData, IQueryTypeParameter, IEquatable<CustomSpeed>, ISerializable
{
    public float m_Speed;      // Speed in KPH
    public float m_SpeedMPH;   // Speed in MPH (for NA lane markings)
    // ...
    public void Serialize<TWriter>(TWriter writer) where TWriter : IWriter
    { writer.Write(m_Speed); writer.Write(m_SpeedMPH); }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/Components/CustomSpeed.cs#L7-L27` (@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed)

The tool adds it and stamps the value when the player edits a segment:

```csharp
if (!EntityManager.HasComponent<CustomSpeed>(targetEdge))
    EntityManager.AddComponent<CustomSpeed>(targetEdge);
var customSpeed = new CustomSpeed(speedKmh);
EntityManager.SetComponentData(targetEdge, customSpeed);
```
Source: `../../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/Systems/RoadSpeedToolSystem.cs#L432-L438` (@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed)

### 6. Mirror to a JSON side-car for reset (not for re-apply)

Alongside the ECS write, the mod stores `(defaultSpeed, currentSpeed)` per edge in a
per-city JSON file. This is a RESET record: it remembers the pre-edit vanilla value so a
segment can be restored. Its storage system is registered at the `Deserialize` phase but
only *initializes* the file for the loaded city - nothing iterates the JSON at load to
re-push speeds, so the side-car is inert with respect to applying edits:

```csharp
PersistentSpeedStorage.StoreRoadSpeed(targetEdge.Index, originalSpeed, speedKmh);
```
Source: `../../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/Systems/RoadSpeedToolSystem.cs#L429` (@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed)

Reset reads the stored default, writes it back onto the vanilla lane (same `/ 1.8`
conversion), removes the marker, and drops the JSON row:

```csharp
float speedGameUnits = originalSpeed.Value / 1.8f;   // restore the pre-edit value
// ... SetComponentData(laneEntity, carLane/trackLane) ...
EntityManager.RemoveComponent<CustomSpeed>(entity);
PersistentSpeedStorage.RemoveRoad(entity.Index);
```
Source: `../../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/Systems/ClearCustomSpeedsSystem.cs#L94-L127` (@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed)

Register the apply system at `ModificationEnd` and the save/reset system at `Deserialize`
so writes land after the game finishes regenerating networks each frame:
Source: `../../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/Mod.cs#L36-L42` (@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed)

## Pitfalls & gotchas

- **The `/ 1.8` unit is CS2-specific and easy to get wrong.** Speed fields are `2x m/s`,
  not m/s and not km/h. From km/h the divisor is `1.8`, not `3.6`; the source calls this
  out inline (`RoadSpeedApplySystem.cs#L92-L94`). Both the apply path (L94) and the reset
  path (`ClearCustomSpeedsSystem.cs`, L94 in the reset block) use `/ 1.8f`. Use the wrong
  divisor and every speed is off by 2x.

- **You MUST preserve `m_Flags`.** `CarLane` carries behavior flags alongside the speed
  fields. The canonical write snapshots `m_Flags` before the edit and restores it after,
  with the explicit note "ONLY modify speed fields"
  (`RoadSpeedApplySystem.cs#L116-L124`). Overwrite the struct without preserving flags and
  you clobber lane behavior. `TrackLane` here only has `m_SpeedLimit` touched
  (`#L131-L135`).

- **Never reassign the `SubLane` buffer element.** Write through to the lane entity
  (`subLanes[i].m_SubLane`), not back into the buffer slot. The source flags this
  explicitly: "DON'T reassign subLanes[i] - it breaks connectivity!"
  (`RoadSpeedApplySystem.cs#L104`).

- **Regeneration resets your write - that is what the `Updated` re-apply is for.** When
  the game rebuilds an edge's lanes it re-derives speeds from the prefab, discarding your
  live write. The apply system re-pushes the stored value on any edge that has both
  `CustomSpeed` and `Updated` (`RoadSpeedApplySystem.cs#L24-L84`). If your marker is gone,
  the re-apply cannot fire.

- **Side-car keyed by `Entity.Index` is fragile across sessions.** The JSON stores rows
  under `targetEdge.Index` (an `int`), e.g. `StoreRoadSpeed(targetEdge.Index, ...)`
  (`RoadSpeedToolSystem.cs#L429`). Entity indices are not stable identifiers across
  save/reload or between machines; a reused index can point the reset record at the wrong
  segment. This side-car is reset-only, so the blast radius is a bad "reset to default",
  but it is a real correctness hazard for any load-time re-apply you might add.

- **Persistence model - what is proven vs. what is not.** Proven from source: the durable
  value is written into the vanilla `Game.Net.CarLane`/`TrackLane` components on live
  entities (`RoadSpeedApplySystem.cs#L112-L136`); the `CustomSpeed` marker is a modded
  `IComponentData` that also implements `ISerializable` (`CustomSpeed.cs#L7-L27`); the JSON
  side-car is initialized at load but never iterated to re-apply
  (`RoadSpeedSaveDataSystem.cs`, `PersistentSpeedStorage` only `Initialize`/`Save`).
  **Needs Verification (in-game):** that the vanilla-component edits actually survive
  after the mod is uninstalled; whether the game's entity serializer keeps or drops the
  modded `CustomSpeed` component across save/load and after uninstall; and whether the
  game re-tags edited edges with `Updated` on load. These are runtime-serializer behaviors
  not decidable from this source alone.

## Variations

- **Reset one segment instead of clearing all.** The tool's per-segment reset reads the
  stored default off the side-car, writes it back onto the vanilla lane, and removes the
  marker (`RoadSpeedToolSystem.cs#L499`, `#L531-L538`) - the same durable-write shape run
  with the pre-edit value.

- **Any per-instance vanilla net field, not just speed.** The technique generalizes to
  any `Game.Net` component field on a lane/edge/node entity: read the vanilla struct off
  the live entity, mutate the target field, preserve the rest, `SetComponentData`. The
  durability comes from the field living in a vanilla component - the speed limit is just
  the worked example here.

- **Prefab-level default instead of per-segment.** If you want to change the value for
  EVERY segment of a road type rather than one placed segment, that is the opposite
  trade-off - a prefab-data edit that does NOT survive uninstall and must be re-applied
  each load. See [prefab-field-override](prefab-field-override.md).

## See also
- Related recipes: [prefab-field-override](prefab-field-override.md) (family A - the
  non-durable, all-instances counterpart), [json-sidecar-persistence](json-sidecar-persistence.md)
  (the reset/side-car layer used here).
- Reference: [serialization](../../explanation/serialization.md) (vanilla vs. modded
  component persistence - the "why" behind survives-uninstall).
- Case studies demonstrating it: [road-speed-adjuster](../../case-studies/road-speed-adjuster.md).

## Sources
- Canonical mods (dossier + repo):
  - `road-speed-adjuster` @e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed - `repo/Systems/RoadSpeedApplySystem.cs`, `repo/Components/CustomSpeed.cs`, `repo/Systems/RoadSpeedToolSystem.cs`, `repo/Systems/ClearCustomSpeedsSystem.cs`, `repo/Mod.cs`
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding
