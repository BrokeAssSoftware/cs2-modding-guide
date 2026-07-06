---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Case study: Advanced Road Naming"
case_study: advanced-road-naming
mod: "Advanced Road Naming"
dossier: ../../vice-and-order-research/mods/dossiers/advanced-road-naming/
repo_commit: 559e72cdb3180e3e71869643367e094a22eed988
source_version: "~1.5.10f1 (advanced-road-naming@559e72c; date-pinned, static source only)"
last_reverified: "2026-07-04"
diataxis: explanation
techniques: [E, Q, W, V, I, B]
technique_applicability: [infrastructure, ui, core]
status: source-verified
Created: 2026-07-04
Updated: 2026-07-04
Owners:
  - codex
---

# Advanced Road Naming - case study

> A road-labelling mod that adds a full custom tool - click a chain of waypoints,
> auto-path the segments between them, drag waypoints to edit, and write route
> numbers onto the real road label - built entirely from ECS systems and vanilla
> `NameSystem` / `OverlayRenderSystem` seams, with **no Harmony patch anywhere**.
> A clean tour of how a tooling + UI + persistence mod stitches together purely
> through system scheduling.

All code claims below are cited to `repo/<path>#Lxx` at pinned commit
`559e72cdb3180e3e71869643367e094a22eed988` (commit subject "Routes Overhaul"),
surfaced through the dossier at
`../../vice-and-order-research/mods/dossiers/advanced-road-naming/`.

> Drift trap: the checked-out working tree of this dossier is **ahead** of the
> pin at `f31d04379b12e2092660fbde4db1f53dab61e0f0` ("1.7 Update"). Every line
> number here was resolved with `git show 559e72c:<path>`, not the working tree;
> re-verify against the pin, not `repo/` on disk.

## What it does / why it's instructive

Advanced Road Naming lets a player rename road segments and assign route numbers
(highways, transit-style corridors) that persist across saves. Its interesting
half is the *tool*: instead of a one-click-per-segment picker, it exposes a
multi-waypoint route tool - you click a start segment, click further segments,
and the mod runs a breadth-first search across the road graph to auto-fill every
segment between consecutive waypoints, then draws the whole route as an overlay
you can revise by dragging waypoints before committing.

It is instructive because it is a **maximal "polite" mod**: it reaches deep into
core game state (aggregated road entities, the vanilla `NameSystem` custom-name
store, the `AggregateSystem` rebuild) yet never patches a single method. Every
integration point is a vanilla ECS seam - a serialization phase, a system update
order, a `NameSystem` API, an `OverlayRenderSystem.Buffer`. It is therefore a
good reference for the whole family of "add a tool + UI + saved data" mods where
Harmony is *not* required, only correct scheduling. It also shows the cost of
that politeness: a large amount of defensive code fighting the vanilla aggregate
lifecycle (split/re-merge timing, deferred name reapply after load).

## Architecture at a glance

Everything is registered in one place, `Mod.OnLoad`, which schedules ten system
hooks (across eight distinct systems) into specific `SystemUpdatePhase` slots and
applies **zero** Harmony patches (repo/Mod.cs#L54-L64):

```csharp
updateSystem.UpdateAt<SegmentMetadataSystem>(SystemUpdatePhase.Deserialize);
updateSystem.UpdateAt<SegmentMetadataSystem>(SystemUpdatePhase.Serialize);
updateSystem.UpdateAfter<SegmentMetadataSystem, AggregateSystem>(SystemUpdatePhase.ModificationEnd);
updateSystem.UpdateBefore<RoadAggregateProtectionSystem, AggregateSystem>(SystemUpdatePhase.ModificationEnd);
updateSystem.UpdateAt<RoadRouteToolSystem>(SystemUpdatePhase.ToolUpdate);
updateSystem.UpdateAt<RoadRouteToolTooltipSystem>(SystemUpdatePhase.UITooltip);
updateSystem.UpdateAt<RoadRouteToolUISystem>(SystemUpdatePhase.UIUpdate);
updateSystem.UpdateAfter<RoadSelectionInfoSectionSystem>(SystemUpdatePhase.UIUpdate);
updateSystem.UpdateAt<RoadRouteOverlayGeometrySystem>(SystemUpdatePhase.Rendering);
updateSystem.UpdateAt<RoadRouteHighlightSystem>(SystemUpdatePhase.Rendering);
```

The absence of Harmony is confirmed at the build level: the `.csproj` references
`Game`, the `Colossal.*` assemblies, `cohtml.Net`, and Unity modules only - no
`0Harmony` / `Lib.Harmony` reference exists at this pin
(repo/AdvancedRoadNaming.csproj#L53-L80; a full-file grep for "harmony" returns
zero hits).

Roles by phase:

- **Persistence** - `SegmentMetadataSystem` is scheduled at both
  `SystemUpdatePhase.Deserialize` and `SystemUpdatePhase.Serialize` so its
  `IDefaultSerializable` hooks run inside the save pipeline
  (repo/Mod.cs#L54-L55).
- **Aggregate lifecycle** - the same system also runs `UpdateAfter<..,
  AggregateSystem>` in `ModificationEnd`, and a tiny helper
  `RoadAggregateProtectionSystem` runs `UpdateBefore<.., AggregateSystem>` in the
  same phase, so the mod can guard its managed aggregates on both sides of the
  vanilla rebuild (repo/Mod.cs#L56-L57).
- **Tool** - `RoadRouteToolSystem` runs at `ToolUpdate`; its tooltip and UI
  bridges (`RoadRouteToolTooltipSystem`, `RoadRouteToolUISystem`) run at
  `UITooltip` / `UIUpdate`, and `RoadSelectionInfoSectionSystem` runs at
  `UIUpdate` as well (repo/Mod.cs#L58-L61).
- **Overlay** - `RoadRouteOverlayGeometrySystem` builds route geometry and
  `RoadRouteHighlightSystem` emits it into the overlay buffer, both at
  `Rendering` (repo/Mod.cs#L62-L63).

## Techniques demonstrated

- [ECS serializable save data](../how-to/recipes/ecs-serializable-savedata.md)
  (family E) - `SegmentMetadataSystem` is
  `GameSystemBase, IDefaultSerializable` with a versioned format
  (`SaveVersion = 6`) and hand-rolled `Serialize`/`Deserialize` that walk the
  in-memory repository, writing each record's segment entity, base-name snapshot,
  custom name, route-number list, timestamps, flags and placement, then a second
  block of saved routes; `Deserialize` refuses any `version > SaveVersion`
  (repo/Systems/SegmentMetadataSystem.cs#L19-L21,
  repo/Systems/SegmentMetadataSystem.cs#L3257-L3277,
  repo/Systems/SegmentMetadataSystem.cs#L3281-L3293).
- [Custom names via NameSystem](../how-to/recipes/namesystem-custom-names.md)
  (family Q) - the system caches the vanilla `NameSystem`
  (`World.GetOrCreateSystemManaged<NameSystem>()`,
  repo/Systems/SegmentMetadataSystem.cs#L70) and writes the resolved label with
  `_nameSystem.SetCustomName(nameEntity, safeName)` inside `SetAuthoritativeName`,
  then tags `CustomName` / `Updated` / `BatchesUpdated` for refresh; it reads back
  via `_nameSystem.TryGetCustomName(...)` / `GetRenderedLabelName(...)` and clears
  child-edge names with `SetCustomName(edge, null)` so an aggregated road's label
  lives on one owner entity
  (repo/Systems/SegmentMetadataSystem.cs#L1655-L1666,
  repo/Systems/SegmentMetadataSystem.cs#L1690-L1691,
  repo/Systems/SegmentMetadataSystem.cs#L850-L855).
- [N-waypoint path tool](../how-to/recipes/tool-nwaypoint-path.md) (family W) -
  `RoadRouteToolSystem : ToolBaseSystem` commits one waypoint per click
  (`TryAddHoveredWaypoint` -> `_selectionController.TryAddWaypoint`,
  repo/Systems/RoadRouteToolSystem.cs#L270-L291,
  repo/Systems/RoadRouteToolSystem.cs#L818-L829). The controller then rebuilds
  the selected-segment set by pathing between consecutive waypoints
  (repo/Services/RouteSelectionController.cs#L167,
  repo/Services/RouteSelectionController.cs#L455-L462). The path itself is a
  breadth-first search over the road graph in `RoadNetworkPathingService.FindPath`,
  which walks `Edge.m_Start` / `Edge.m_End` nodes and their `ConnectedEdge`
  buffers up to `maxDepth`, reconstructing the segment chain when it reaches the
  target (repo/Services/RoadNetworkPathingService.cs#L40-L110).
- [Drag-select / drag-to-edit gesture](../how-to/recipes/tool-drag-select.md)
  (family V) - waypoints are editable by a press-drag-release gesture. On
  `applyAction.WasPressedThisFrame()` the tool calls
  `_selectionController.TryBeginEditFromHover()`, which grabs the hovered waypoint
  in `Move` mode (or a route-line insertion point in `Insert` mode); the route
  follows the cursor while held; on `applyAction.WasReleasedThisFrame()` the tool
  calls `CommitActiveEdit()` to move/insert the waypoint and re-path
  (repo/Systems/RoadRouteToolSystem.cs#L751-L752,
  repo/Systems/RoadRouteToolSystem.cs#L796-L803,
  repo/Systems/RoadRouteToolSystem.cs#L818-L826,
  repo/Services/RouteSelectionController.cs#L77-L100,
  repo/Services/RouteSelectionController.cs#L102-L133).
- [Overlay render-pipeline drawing](../how-to/recipes/render-pipeline-overlay.md)
  (family I) - two systems split "compute" from "draw".
  `RoadRouteOverlayGeometrySystem` builds `Bezier4x3` curves and `float3` nodes
  (the element types of its pooled lists,
  repo/Systems/RoadRouteOverlayGeometrySystem.cs#L16-L22) for the
  active/preview/saved/hover routes each frame while the tool is running
  (repo/Systems/RoadRouteOverlayGeometrySystem.cs#L46-L61).
  `RoadRouteHighlightSystem` then acquires the vanilla overlay buffer with
  `_overlayRenderSystem.GetBuffer(out var jobHandle)`, completes the handle, and
  emits the geometry via `buffer.DrawCurve(...)` and `buffer.DrawCircle(...)`
  (repo/Systems/RoadRouteHighlightSystem.cs#L42-L49,
  repo/Systems/RoadRouteHighlightSystem.cs#L57-L70,
  repo/Systems/RoadRouteHighlightSystem.cs#L95-L111).
- [ECS system replacement & ordering](../how-to/recipes/ecs-system-replacement-ordering.md)
  (family B) - all core game integration is scheduling, not patching. The mod
  brackets the vanilla `AggregateSystem` in `ModificationEnd`
  (`RoadAggregateProtectionSystem` before it, `SegmentMetadataSystem` after it)
  so it can protect its managed aggregates ahead of the rebuild and reconcile
  after (repo/Mod.cs#L56-L57). `RoadAggregateProtectionSystem.OnUpdate` is a
  one-line delegate into the metadata system's
  `ProtectModAggregatesBeforeVanilla()`
  (repo/Systems/RoadAggregateProtectionSystem.cs#L15-L18,
  repo/Systems/SegmentMetadataSystem.cs#L97-L115).

## Key decisions & tradeoffs

- **No Harmony, pure ECS.** The mod never patches game code; it only registers
  systems into vanilla phases and calls public vanilla APIs (`NameSystem`,
  `OverlayRenderSystem.Buffer`, `AggregateSystem` ordering). This is the single
  biggest design decision and the reason it survives patches better than a
  decompile-and-patch mod: there are no method signatures to re-bind
  (repo/Mod.cs#L54-L64; repo/AdvancedRoadNaming.csproj#L53-L80).
- **Own the aggregate, borrow the label.** A named road is a vanilla *aggregate*
  of edges; the mod writes the custom name onto the aggregate owner and actively
  clears child-edge custom names so the vanilla label resolver shows one name
  (repo/Systems/SegmentMetadataSystem.cs#L1655-L1691). It uses `BatchesUpdated`
  on child edges to refresh render batches *without* feeding `AggregateSystem` an
  `Updated` edge - an explicit choice to avoid retriggering the vanilla rebuild
  (repo/Systems/SegmentMetadataSystem.cs#L1671-L1674).
- **Versioned, defensive save format.** `Serialize` runs `CleanupOrphanedMetadata`
  first and stamps `SaveVersion`; `Deserialize` clears all in-memory state, reads
  the version, and bails on anything newer than it understands rather than
  mis-parsing (repo/Systems/SegmentMetadataSystem.cs#L3259-L3260,
  repo/Systems/SegmentMetadataSystem.cs#L3281-L3293). Names are not written during
  load; a deferred post-load reapply is queued instead (see pitfalls).
- **Compute/draw split for the overlay.** Geometry is built in one `Rendering`
  system into pooled lists and consumed by a second `Rendering` system that only
  touches the overlay buffer, keeping the buffer critical section (acquire ->
  `Complete()` -> draw) small (repo/Systems/RoadRouteHighlightSystem.cs#L57-L70).
- **BFS, not the game pathfinder.** Segment auto-fill uses the mod's own
  bounded breadth-first search over `ConnectedEdge` adjacency
  (repo/Services/RoadNetworkPathingService.cs#L40-L110) rather than the simulation
  path engine - simpler, deterministic, and depth-capped, at the cost of ignoring
  road hierarchy/cost.

## Pitfalls / upstream-watch

- **NameSystem write timing after load.** The mod does not write names during
  `Deserialize`; it queues a delayed reapply and only writes once the world is
  judged safe. `OnUpdate` gates all live aggregate maintenance behind
  `IsSafeForLiveAggregateMaintenance()` and runs `ProcessDeferredPostLoadNameReapply`
  before anything else (repo/Systems/SegmentMetadataSystem.cs#L81-L95). The retry
  budget is finite (`DeferredNameReapplyMaxAttempts = 10`,
  repo/Systems/SegmentMetadataSystem.cs#L28); a save that never reaches the "safe"
  state within that budget would drop the reapply. Any change to the aggregate or
  name lifecycle in a new CS2 build is the thing to re-verify here.
- **Aggregate split/re-merge race.** Renaming forces the mod to split and re-merge
  vanilla aggregates, and it carries a whole state machine of stability checks
  (`AggregateStabilityStableChecksRequired = 3`,
  `AggregateStabilityMaxReapplyAttempts = 3`,
  repo/Systems/SegmentMetadataSystem.cs#L22-L25) plus the
  before/after-`AggregateSystem` bracketing. This is fragile by nature - it is
  timing against a system the mod does not control
  (repo/Mod.cs#L56-L57).
- **Working-tree drift (upstream-watch).** The dossier's checked-out tree is at
  `f31d043` "1.7 Update", ahead of this pin `559e72c` "Routes Overhaul". Anyone
  re-deriving claims must read at the pin; the newer tree has diverged (the pin
  subject alone signals a routes-subsystem rewrite between them). Treat every
  line number here as pin-scoped.
- **Single-owner label assumption.** Because the mod deliberately nulls child-edge
  custom names to keep one visible label
  (repo/Systems/SegmentMetadataSystem.cs#L1688-L1691), another mod that also
  writes edge-level custom names on the same road can be clobbered on the next
  reapply - a multi-writer conflict on the vanilla `NameSystem` store.

Needs Verification (in-game): the actual on-screen appearance of the route
overlay and label refresh, and whether the deferred-reapply retry budget is ever
exhausted on very large saves - both require the running game and cannot be
confirmed from source.

## Source pointers

- Dossier:
  `../../vice-and-order-research/mods/dossiers/advanced-road-naming/`
  (index/source/modding/guide + notes).
- Repo @ `559e72cdb3180e3e71869643367e094a22eed988` (pin subject "Routes
  Overhaul"; working tree drifts to `f31d043`), key files:
  - `repo/Mod.cs` - single-place system scheduling, no Harmony.
  - `repo/Systems/SegmentMetadataSystem.cs` - `IDefaultSerializable` save data +
    `NameSystem` writes + aggregate lifecycle.
  - `repo/Systems/RoadRouteToolSystem.cs`,
    `repo/Services/RouteSelectionController.cs` - N-waypoint tool + drag-edit.
  - `repo/Services/RoadNetworkPathingService.cs` - BFS segment auto-fill.
  - `repo/Systems/RoadRouteOverlayGeometrySystem.cs`,
    `repo/Systems/RoadRouteHighlightSystem.cs` - overlay compute + draw.
  - `repo/Systems/RoadAggregateProtectionSystem.cs` - before-`AggregateSystem`
    guard.
