---
FrontmatterVersion: 1
DocumentType: Guide
Title: "How-to: Sub-element removal (props, trees, sub-nets)"
diataxis: how-to
source_version: "~1.5.x (better-bulldozer@4408466; date-pinned, static source only)"
last_reverified: "2026-07-04"
canonical_mods:
  - better-bulldozer@4408466f226db811159d92859479ae1e1c28ba06
technique_applicability: [tooling, content]
status: source-verified
Created: 2026-07-03
Updated: 2026-07-03
Owners:
  - codex
---

# Sub-element removal (props, trees, sub-nets)

> Delete props, trees, decals, sub-buildings, or sub-lanes that belong to a parent
> asset - without destroying the parent - by tagging targets with `Deleted` through a
> tool command buffer, and (optionally) recording the removal so it survives regrowth
> and save/load.

## Problem
A placed building, road, or network node owns a `DynamicBuffer<SubObject>` (and often
sub-lanes) of child entities: fences, hedges, street lights, branding props, surface
decals. You want a tool that removes one of those children while leaving the parent
intact. Two things make this non-trivial: you must not mutate ECS during the raycast
pass (structural changes mid-frame corrupt the tool pipeline), and some sub-elements
are **regenerated** by the game the moment you delete them, so a one-shot delete does
not stick.

## Solution
Do the deletion through an `EntityCommandBuffer` obtained from a barrier, never with
direct `EntityManager` calls during the tool update. The recipe has three layers:

1. **Preview** - tag hovered targets with `Highlighted` + `BatchesUpdated` so the
   player sees what will go.
2. **Delete** - on apply, enumerate the parent's `DynamicBuffer<SubObject>` and
   `buffer.AddComponent<Deleted>(child)` for each target. `Deleted` is the vanilla
   marker the cleanup systems act on.
3. **Persist (optional)** - for elements the game regrows, record the removed prefab in
   a serializable buffer on the owner (`PermanentlyRemovedSubElementPrefab`) and run a
   background system that re-deletes any regenerated copy via a frame-delay marker
   (`DeleteInXFrames`) on a `ModificationEndBarrier`.

## Steps & Code

### 1. Delegate the prefab/tool identity to the vanilla bulldozer

If you are extending the bulldozer, share its `toolID` and prefab so the toolbar button
and keybinds stay vanilla while your logic runs:

```csharp
public override string toolID => m_BulldozeToolSystem.toolID;
public override PrefabBase GetPrefab()          => m_BulldozeToolSystem.GetPrefab();
```
Source: `../../../vice-and-order-research/mods/dossiers/better-bulldozer/repo/BetterBulldozer/Tools/SubElementBulldozerTool.cs#L62-L85` (@4408466f226db811159d92859479ae1e1c28ba06)

### 2. Get a command buffer from a barrier - not `EntityManager`

Grab a `ToolOutputBarrier` in `OnCreate` and create a fresh command buffer each update.
All structural edits go on this buffer; the barrier plays it back at a safe point:

```csharp
// OnCreate
m_ToolOutputBarrier = World.GetOrCreateSystemManaged<ToolOutputBarrier>();

// during the tool update
EntityCommandBuffer buffer = m_ToolOutputBarrier.CreateCommandBuffer();
```
Source: `../../../vice-and-order-research/mods/dossiers/better-bulldozer/repo/BetterBulldozer/Tools/SubElementBulldozerTool.cs#L168, #L306` (@4408466f226db811159d92859479ae1e1c28ba06)

### 3. Preview targets with `Highlighted` + `BatchesUpdated`

Before applying, mark the hovered sub-objects so the player sees the selection. Adding
`BatchesUpdated` forces a re-render of the affected entity:

```csharp
foreach (Game.Objects.SubObject subObject in dynamicBuffer)
{
    buffer.AddComponent<Highlighted>(subObject.m_SubObject);
    buffer.AddComponent<BatchesUpdated>(subObject.m_SubObject);
}
```
Source: `../../../vice-and-order-research/mods/dossiers/better-bulldozer/repo/BetterBulldozer/Tools/SubElementBulldozerTool.cs#L338-L340` (@4408466f226db811159d92859479ae1e1c28ba06)

Clear the highlight each frame the query is non-empty so stale highlights do not stick
(`AddComponent<BatchesUpdated>(m_HighlightedQuery); RemoveComponent<Highlighted>(m_HighlightedQuery)`,
`SubElementBulldozerTool.cs#L239-L240`).

### 4. On apply, enumerate `SubObject` and tag `Deleted`

The delete itself: read the parent's `DynamicBuffer<Game.Objects.SubObject>` and add the
vanilla `Deleted` marker to each child (and, if appropriate, the parent entity):

```csharp
if (EntityManager.TryGetBuffer(currentEntity, false,
        out DynamicBuffer<Game.Objects.SubObject> dynamicBuffer))
{
    foreach (Game.Objects.SubObject subObject in dynamicBuffer)
    {
        if (subObject.m_SubObject != Entity.Null)
            buffer.AddComponent<Deleted>(subObject.m_SubObject);
    }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/better-bulldozer/repo/BetterBulldozer/Tools/SubElementBulldozerTool.cs#L700-L713` (@4408466f226db811159d92859479ae1e1c28ba06)

For sub-networks the mod tags the edge's start/end nodes `Deleted` and marks connected
edges `Updated` so the network re-stitches cleanly
(`SubElementBulldozerTool.cs#L649-L695`).

### 5. Record permanent removals so regrowth stays gone

Some sub-elements are re-spawned by the game. To make removal stick, add a serializable
buffer to the owner and record each removed prefab in it:

```csharp
[InternalBufferCapacity(0)]
public struct PermanentlyRemovedSubElementPrefab
    : IBufferElementData, IQueryTypeParameter, ISerializable
{
    public Entity m_RecordEntity;   // -> an entity holding OwnerRecord + PrefabRef
}
```
Source: `../../../vice-and-order-research/mods/dossiers/better-bulldozer/repo/BetterBulldozer/Components/PermanentlyRemovedSubElementPrefab.cs#L16-L23` (@4408466f226db811159d92859479ae1e1c28ba06)

The tool ensures the buffer exists on the owner, then records each removed prefab
against a small record entity carrying `OwnerRecord` + `PrefabRef`
(`SubElementBulldozerTool.cs#L718-L730`).

### 6. Re-delete regenerated copies with a frame-delay marker

A background system watches for the recorded prefab regrowing and tags it for delayed
deletion. `DeleteInXFrames` is a countdown; a handler decrements it and adds `Deleted`
when it hits zero, all on a barrier command buffer:

```csharp
public struct DeleteInXFrames : IComponentData, IQueryTypeParameter
{
    public int m_FramesRemaining;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/better-bulldozer/repo/BetterBulldozer/Components/DeleteInXFrames.cs#L11-L16` (@4408466f226db811159d92859479ae1e1c28ba06)

```csharp
// HandleDeleteInXFramesSystem: countdown then delete
DeleteInXFrames deleteInXFrames = deleteInXFramesNativeArray[i];
if (deleteInXFrames.m_FramesRemaining <= 0)
    buffer.AddComponent<Deleted>(entityNativeArray[i]);
else
    deleteInXFrames.m_FramesRemaining--;
```
Source: `../../../vice-and-order-research/mods/dossiers/better-bulldozer/repo/BetterBulldozer/Systems/HandleDeleteInXFramesSystem.cs#L97-L104` (@4408466f226db811159d92859479ae1e1c28ba06)

The system that catches regenerated prefabs runs on a `ModificationEndBarrier` (correct
phase for post-simulation structural edits) and stamps `DeleteInXFrames { m_FramesRemaining = 30 }`
(`RemoveRegeneratedSubelementPrefabsSystem.cs#L54, #L569-L572`).

## Pitfalls & gotchas

- **Never call `EntityManager.DestroyEntity`/`AddComponent` during the tool pass.**
  Structural changes mid-frame corrupt the tool pipeline. Enqueue everything on a
  barrier's `EntityCommandBuffer`; the barrier plays it back at the right phase. Better
  Bulldozer uses `ToolOutputBarrier` for interactive edits and `ModificationEndBarrier`
  for the background regrowth sweep - match the barrier to the phase your system runs in.
- **Regeneration defeats a one-shot delete.** Fences, hedges, and branding props re-spawn.
  Without the record-and-re-delete layer (steps 5-6) the element reappears within a few
  frames. The two-phase `PermanentlyRemovedSubElementPrefab` + `DeleteInXFrames` dance
  exists precisely to beat this.
- **`Deleted` is a marker, not an instant destroy.** Adding `Deleted` schedules vanilla
  cleanup; the entity lives for the rest of the frame. Do not assume the child is gone
  the line after you tag it.
- **Deleting the parent vs the child.** Tagging `Deleted` on a `SubObject` removes the
  child; tagging it on `currentEntity` removes the parent too. Better Bulldozer guards
  the parent delete behind extension/node checks
  (`SubElementBulldozerTool.cs#L711`) - be deliberate about which you tag.
- **Not every sub-element is safe to remove.** Service upgrades, network connectors, and
  extensions can break the owner. Gate risky categories behind settings/UI toggles and
  validate before enqueuing (the mod checks `HasComponent<Extension>` and an
  `AllowRemovingExtensions` setting, `SubElementBulldozerTool.cs#L700, #L711`).
- Whether a given regenerated prefab is caught reliably across all asset types is
  `Needs Verification (in-game)`; the countdown value (30 frames) is an empirical
  choice, not a guarantee.

## Variations

- **Undo / restore to factory.** Because removals are recorded against the owner
  (`PermanentlyRemovedSubElementPrefab` -> record entity with `OwnerRecord` + `PrefabRef`),
  a reset action can iterate the records and re-spawn the original prefabs. Better
  Bulldozer ships restore systems (`RestoreFencesAndHedgesSystem`, `SafelyRemoveSystem`)
  that do exactly this.
- **Automatic (non-interactive) pruning.** The same `DeleteInXFrames` +
  `ModificationEndBarrier` machinery drives "always remove fences/branding on spawn"
  systems - no tool involved, just a query that stamps the delay marker on matching new
  entities.
- **Delete sub-lanes and network sub-elements**, not just objects: enumerate
  `DynamicBuffer<SubLane>` and tag `Deleted` on `m_SubLane`, marking touched edges
  `Updated` (`SubElementBulldozerTool.cs#L412-L413, #L649-L695`).

## See also
- Sibling tooling how-tos: [raycast filters](raycast-filters.md) (how the tool finds
  the sub-element to delete), [transform gizmos](transform-gizmos.md),
  [validation overrides](validation-overrides.md).
- Reference: [technique index](../../technique-index.md) - family Y (event-driven
  ModificationEnd / ToolOutputBarrier) and family V (tool drag-select + Highlighted).
- Explanation: [multi-phase scheduling](../../explanation/multi-phase-scheduling.md)
  (why the barrier phase matters), [serialization](../../explanation/serialization.md)
  (persisting the removal record across save/load).

## Sources
- Canonical mods (dossier + repo):
  - `better-bulldozer` @4408466f226db811159d92859479ae1e1c28ba06 -
    `repo/BetterBulldozer/Tools/SubElementBulldozerTool.cs`,
    `repo/BetterBulldozer/Components/PermanentlyRemovedSubElementPrefab.cs`,
    `repo/BetterBulldozer/Components/DeleteInXFrames.cs`,
    `repo/BetterBulldozer/Systems/HandleDeleteInXFramesSystem.cs`,
    `repo/BetterBulldozer/Systems/RemoveRegeneratedSubelementPrefabsSystem.cs`
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding
