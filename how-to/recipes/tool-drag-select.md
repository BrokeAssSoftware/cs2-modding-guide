---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: Tool drag-select + Highlighted components"
recipe: tool-drag-select
technique_family: "V - Tool drag-select + Highlighted components"
diataxis: how-to
source_version: "~1.5.2f1 (road-speed-adjuster@e0c0c0b; date-pinned, static source only)"
last_reverified: "2026-07-04"
status: source-verified
canonical_mods:
  - road-speed-adjuster@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed
  - anarchy@a6311e898d20a775368668b234aaa32f06e3e1eb
  - better-bulldozer@4408466f226db811159d92859479ae1e1c28ba06
  - advanced-road-naming@559e72cdb3180e3e71869643367e094a22eed988
technique_applicability: [ui, infrastructure]
Created: 2026-07-04
Updated: 2026-07-04
Owners:
  - codex
---

# Tool drag-select + Highlighted components

> Build a custom `ToolBaseSystem` that click-drags to multi-select world entities,
> paints each one with the vanilla `Highlighted` overlay, and finalizes the set on
> mouse-up - no Harmony, no custom shaders.

## Problem
You want the player to point at things in the world (road segments, tracks, markers),
drag across several of them, and have your mod act on the whole selection at once -
with the same glowing overlay vanilla tools use for hover/selection. You need three
things wired together: a raycast that only hits the entities you care about, a
per-frame accumulator that grows the selection while the mouse is held, and the
`Highlighted` overlay toggled on and off as the set changes.

## Solution
Subclass `ToolBaseSystem`. In `InitializeRaycast` narrow `m_ToolRaycastSystem` to the
layers you want (e.g. `TypeMask.Net` + a `netLayerMask`). Let `base.OnCreate()` build
the `applyAction` input for you, then read that action's edge/held state in `OnUpdate`:
`WasPressedThisFrame` starts a drag, `IsPressed` keeps adding the raycast-hit entity to
a `HashSet<Entity>`, `WasReleasedThisFrame` finalizes. For each entity you add or drop,
add `Highlighted` **and** `BatchesUpdated` - the overlay is a batched render component,
so without `BatchesUpdated` the glow never repaints. On cancel (`cancelAction`) or tool
stop, strip `Highlighted` from everything with a query. Road Speed Adjuster's
`RoadSpeedToolSystem` is the cleanest end-to-end example of exactly this shape.

## Steps & Code

### 1. Subclass `ToolBaseSystem` and hold the selection state

The selection lives in a `HashSet<Entity>` (dedupes hits during a fast drag); a
committed `List<Entity>` is the finalized result. A single `m_HoverEntity` tracks the
pre-drag hover glow so it can be cleared cleanly.

```csharp
public partial class RoadSpeedToolSystem : ToolBaseSystem
{
    public const string kToolID = "Road Speed Tool";

    private EntityQuery m_RoadQuery;
    private readonly List<Entity> m_SelectedRoads = new();
    private readonly HashSet<Entity> m_TempSelection = new();

    private bool m_IsDragging = false;
    private Entity m_HoverEntity = Entity.Null;

    public override string toolID => kToolID;
```
Source: `../../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/Systems/RoadSpeedToolSystem.cs#L26-L39` (@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed)

### 2. Let `base.OnCreate()` build `applyAction`, then enable it on start

Do not construct the input action yourself. `ToolBaseSystem.OnCreate()` initializes
`applyAction`; you just call `base.OnCreate()` after your own setup, and flip
`applyAction.shouldBeEnabled = true` in `OnStartRunning` so it fires while the tool is
active.

```csharp
protected override void OnCreate()
{
    Enabled = false;
    m_UISystem = World.GetOrCreateSystemManaged<RoadSpeedToolUISystem>();
    // ... build m_RoadQuery ...
    base.OnCreate();   // initializes applyAction for us
}

protected override void OnStartRunning()
{
    base.OnStartRunning();
    m_SelectedRoads.Clear();
    m_TempSelection.Clear();
    m_IsDragging = false;
    applyAction.shouldBeEnabled = true;   // exactly like Better Bulldozer
}
```
Source: `../../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/Systems/RoadSpeedToolSystem.cs#L76-L115` (@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed)

### 3. Narrow the raycast in `InitializeRaycast`

Override `InitializeRaycast`, call `base`, then set `typeMask`, `netLayerMask`, and
`raycastFlags` on `m_ToolRaycastSystem`. This is what limits your hits to the entity
class you multi-select. Road Speed Adjuster targets nets on several transport layers:

```csharp
public override void InitializeRaycast()
{
    base.InitializeRaycast();

    m_ToolRaycastSystem.typeMask = TypeMask.Net;
    m_ToolRaycastSystem.netLayerMask =
        Layer.Road | Layer.TrainTrack | Layer.TramTrack | Layer.SubwayTrack | Layer.Waterway;
    m_ToolRaycastSystem.raycastFlags = RaycastFlags.SubElements | RaycastFlags.Markers;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/Systems/RoadSpeedToolSystem.cs#L145-L153` (@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed)

### 4. Highlight on hover - add `Highlighted` AND `BatchesUpdated`

Before a drag starts, glow whatever the ray currently hits. When the hovered entity
changes, drop the old glow and add the new one. Note that **every** toggle of
`Highlighted` is paired with `BatchesUpdated` - see the pitfall below.

```csharp
// Add highlight to new hover
if (!EntityManager.HasComponent<Highlighted>(currentPoint.m_OriginalEntity))
{
    EntityManager.AddComponent<Highlighted>(currentPoint.m_OriginalEntity);
    EntityManager.AddComponent<BatchesUpdated>(currentPoint.m_OriginalEntity);
}
m_HoverEntity = currentPoint.m_OriginalEntity;
```
Source: `../../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/Systems/RoadSpeedToolSystem.cs#L206-L212` (@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed)

### 5. Accumulate into the `HashSet` while the button is held

`WasPressedThisFrame` opens a drag (clearing the temp set and the leftover hover glow);
`IsPressed` keeps adding the current hit if it is a valid `Edge` and not already in the
set. The `HashSet.Contains` check both dedupes and gates the (cost-y) component add.

```csharp
// Continue adding segments while dragging
if (m_IsDragging && applyAction.IsPressed())
{
    if (hasHit && currentPoint.m_OriginalEntity != Entity.Null
        && EntityManager.HasComponent<Edge>(currentPoint.m_OriginalEntity))
    {
        if (!m_TempSelection.Contains(currentPoint.m_OriginalEntity))
        {
            m_TempSelection.Add(currentPoint.m_OriginalEntity);
            if (!EntityManager.HasComponent<Highlighted>(currentPoint.m_OriginalEntity))
                EntityManager.AddComponent<Highlighted>(currentPoint.m_OriginalEntity);
            EntityManager.AddComponent<BatchesUpdated>(currentPoint.m_OriginalEntity);
        }
    }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/Systems/RoadSpeedToolSystem.cs#L272-L289` (@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed)

### 6. Finalize on release; cancel with the cancel action

`WasReleasedThisFrame` commits the `HashSet` into the `List` and hands it to the UI.
`cancelAction.WasPressedThisFrame()` (ESC) reverts to the default tool. Both are read
in the same `OnUpdate`.

```csharp
// Finalize selection when mouse is released
if (m_IsDragging && applyAction.WasReleasedThisFrame())
{
    m_IsDragging = false;
    FinalizeSelection();          // m_SelectedRoads.AddRange(m_TempSelection)
}

// Cancel with ESC
if (cancelAction != null && cancelAction.WasPressedThisFrame())
{
    m_ToolSystem.activeTool = m_DefaultToolSystem;
    return inputDeps;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/Systems/RoadSpeedToolSystem.cs#L291-L307` (@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed)

### 7. Clear all highlights on stop with a single query

On `OnStopRunning` (and whenever you clear the selection) strip the overlay from every
entity that still carries it - a query-wide `RemoveComponent` + `AddComponent<BatchesUpdated>`,
not a per-entity loop.

```csharp
private void RemoveAllHighlights()
{
    var query = GetEntityQuery(ComponentType.ReadOnly<Highlighted>());
    EntityManager.AddComponent<BatchesUpdated>(query);
    EntityManager.RemoveComponent<Highlighted>(query);
}
```
Source: `../../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/Systems/RoadSpeedToolSystem.cs#L312-L318` (@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed)

## Pitfalls & gotchas

- **Toggling `Highlighted` without `BatchesUpdated` = no visual change.** `Highlighted`
  is consumed by the batched rendering pass; the overlay only repaints when the entity
  is also flagged `BatchesUpdated`. Every add and every remove in the canonical source
  is paired: hover-add
  (`../../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/Systems/RoadSpeedToolSystem.cs#L208-L209`),
  hover-remove
  (`...RoadSpeedToolSystem.cs#L200-L201`), and the query-wide clear
  (`...RoadSpeedToolSystem.cs#L316-L317`). Forget it and the entity is logically
  highlighted but looks unchanged until something else dirties its batch.

- **Read `applyAction`, don't build it.** The action is created by
  `base.OnCreate()`; if you skip the base call or try to construct input yourself,
  `applyAction` is null and every `WasPressedThisFrame`/`IsPressed`/`WasReleasedThisFrame`
  throws. Enable it in `OnStartRunning` and disable in `OnStopRunning`
  (`...RoadSpeedToolSystem.cs#L114-L130`).

- **Dedupe hits with the `HashSet`, not the `List`.** During a fast drag `OnUpdate`
  sees the same segment on consecutive frames; the `HashSet.Contains` guard
  (`...RoadSpeedToolSystem.cs#L276`) stops redundant component adds. Committing to a
  `List` only at finalize keeps the public result order-stable.

- **Filter the raycast or you select the wrong thing.** The whole selection is only as
  precise as `typeMask`/`netLayerMask`. With `TypeMask.Net` you get segments; a hover
  hit is still validated with `HasComponent<Edge>` before it counts
  (`...RoadSpeedToolSystem.cs#L190`), because the ray can return sub-elements you do not
  want in the set.

- **`Entity.Index` is not a stable key across sessions.** This mod separately persists
  edits keyed by `Entity.Index` and rebuilds entities as `new Entity { Index = ..., Version = 1 }`
  (`...RoadSpeedToolSystem.cs#L576`). That is fragile across save/reload and unrelated to
  the drag-select itself - keep your selection set in terms of live `Entity` handles for
  the duration of the tool only.

- **Whether the overlay color/intensity is configurable** and how it composites with
  vanilla tool highlights is a runtime concern not visible in this source:
  `Needs Verification (in-game)`.

## Variations

- **Broad multi-layer bulldoze filter (Better Bulldozer).** The same skeleton with a
  much wider raycast and a `m_HighlightedQuery` maintained via an
  `EntityCommandBuffer`. Its `InitializeRaycast` switches `typeMask` and `netLayerMask`
  per sub-tool (nets + static objects + utility lines), e.g.:

  ```csharp
  m_ToolRaycastSystem.typeMask = TypeMask.Net | TypeMask.StaticObjects;
  m_ToolRaycastSystem.netLayerMask = Layer.Road | Layer.Taxiway | Layer.SubwayTrack
      | Layer.PublicTransportRoad | Layer.TrainTrack | Layer.TramTrack
      | Layer.PowerlineHigh | Layer.PowerlineLow;
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/better-bulldozer/repo/BetterBulldozer/Tools/SubElementBulldozerTool.cs#L105-L106` (@4408466f226db811159d92859479ae1e1c28ba06)

  It clears the overlay through a cached query rather than a fresh one each stop
  (`AddComponent<BatchesUpdated>(m_HighlightedQuery)` then
  `RemoveComponent<Highlighted>(m_HighlightedQuery)`,
  `SubElementBulldozerTool.cs#L239-L240`) and adds highlights via an ECB
  (`SubElementBulldozerTool.cs#L326-L329`).

- **Radius vs. per-object selection mode (Anarchy).** Anarchy's components tool swaps
  the raycast target based on a UI selection mode - `TypeMask.Terrain` for a radius
  brush, `TypeMask.StaticObjects` for click selection - showing the raycast is where you
  choose "area" vs "single":

  ```csharp
  if (m_UISystem.CurrentSelectionMode == SelectionMode.Radius)
      m_ToolRaycastSystem.typeMask = TypeMask.Terrain;
  else
      m_ToolRaycastSystem.typeMask = TypeMask.StaticObjects;
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/Systems/AnarchyComponentsTool/AnarchyComponentsToolSystem.cs#L97-L103` (@a6311e898d20a775368668b234aaa32f06e3e1eb)

  It maintains its highlight set through an `m_HighlightedQuery`
  (`AnarchyComponentsToolSystem.cs#L286-L289`, `#L508-L513`).

- **Single-layer, single-pick tool (Advanced Road Naming).** When you only ever act on
  one entity class and do not need a drag set, keep the raycast to one layer and read
  the same apply/cancel edges. ARN's route tool masks to `Layer.Road` only:

  ```csharp
  m_ToolRaycastSystem.typeMask = TypeMask.Net;
  m_ToolRaycastSystem.netLayerMask = Game.Net.Layer.Road;
  m_ToolRaycastSystem.raycastFlags = RaycastFlags.Markers | RaycastFlags.ElevateOffset
      | RaycastFlags.SubElements | RaycastFlags.OutsideConnections;
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Systems/RoadRouteToolSystem.cs#L154-L156` (@559e72cdb3180e3e71869643367e094a22eed988)

  It reads the same `applyAction.WasPressedThisFrame()` / `WasReleasedThisFrame()` edges
  (`RoadRouteToolSystem.cs#L751-L752`) but skips the `HashSet` accumulation entirely.
  (Note: ARN's working HEAD drifts ahead of the pin; the citation is at `559e72c`.)

## See also
- Reference: [raycast filters](../tooling/raycast-filters.md) - the `typeMask` /
  `netLayerMask` / `raycastFlags` catalogue behind step 3.
- Related recipes: [tool N-waypoint path](tool-nwaypoint-path.md) (accumulating ordered
  points instead of an unordered set); [UISystemBase React binding](uisystembase-react-binding.md)
  (surfacing the finalized selection to a panel, as `FinalizeSelection` does here).
- Case studies demonstrating it: [better-bulldozer](../../case-studies/better-bulldozer.md).

## Sources
- Canonical mods (dossier + repo):
  - `road-speed-adjuster` @e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed - `repo/Systems/RoadSpeedToolSystem.cs`
  - `anarchy` @a6311e898d20a775368668b234aaa32f06e3e1eb - `repo/Anarchy/Systems/AnarchyComponentsTool/AnarchyComponentsToolSystem.cs`
  - `better-bulldozer` @4408466f226db811159d92859479ae1e1c28ba06 - `repo/BetterBulldozer/Tools/SubElementBulldozerTool.cs`
  - `advanced-road-naming` @559e72cdb3180e3e71869643367e094a22eed988 - `repo/Systems/RoadRouteToolSystem.cs`
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding
