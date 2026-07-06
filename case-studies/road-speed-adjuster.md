---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Case study: Road Speed Adjuster"
case_study: road-speed-adjuster
mod: "Road Speed Adjuster"
dossier: ../../vice-and-order-research/mods/dossiers/road-speed-adjuster/
repo_commit: e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed
source_version: "1.6.0f1 (road-speed-adjuster@e0c0c0b; date-pinned, static source only)"
last_reverified: "2026-07-05"
diataxis: explanation
techniques: [AN, V, I, F, P]
technique_applicability: [infrastructure, ui, core]
status: source-verified
Created: 2026-07-05
Updated: 2026-07-05
Owners:
  - codex
Summary: How Road Speed Adjuster writes custom speed limits directly into vanilla CarLane/TrackLane components so they survive save/load and even uninstall, and the three-layer persistence model (durable net write + transient ECS marker + reset-only JSON side-car) that makes that durability both powerful and asymmetric.
---

# Road Speed Adjuster - case study

> The canonical example of a *durable* vanilla-network write: instead of shadowing
> speed limits in a mod-owned store, it mutates the game's own `CarLane`/`TrackLane`
> speed fields (flags preserved) so CS2's built-in net serialization saves them -
> which means the change survives save/load AND survives the mod being uninstalled.
> That single decision drives everything else: a three-layer persistence model, a
> reset-only JSON side-car, and a set of asymmetries you have to design around.

All code claims below are cited to `repo/<path>#Lxx` at pinned commit
`e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed` (author DanielVNZ, `mod.json`
version 1.0.0, commit "removed obsolite code"),
surfaced through the dossier at
`../../vice-and-order-research/mods/dossiers/road-speed-adjuster/`. Repo paths are
relative to that dossier's `repo/` tree. Line numbers were confirmed by
`git show <pin>:<path>` at this commit.

## What it does / why it's instructive

Road Speed Adjuster (RSA) lets the player select road, rail, tram, subway, or
waterway segments with a custom drag tool and set a new speed limit on them via a
slider docked in the selected-info panel. A floating text overlay shows the new
limit while the tool is active.

It is instructive not for any one technique but for a *design commitment* and its
consequences. Most "override a vanilla value" mods keep the override in a mod-owned
component and re-apply it every frame/tick so removing the mod cleanly reverts the
world. RSA does the opposite: it writes the new speed straight into the vanilla
`CarLane.m_SpeedLimit` / `TrackLane.m_SpeedLimit` fields, touching only the speed
and explicitly preserving the lane flags. Because those are stock components, CS2's
own network serialization persists them - so the edit is *durable*: it is in the
save with or without the mod, and it stays after uninstall. To make that safe and
reversible, RSA layers three different persistence mechanisms with three different
lifetimes, and that layering (plus the asymmetries it creates) is the real lesson.

## Architecture at a glance

`Mod.OnLoad` registers six systems, each into a deliberately different
`SystemUpdatePhase` (`repo/Mod.cs#L36-L51`):

- `RoadSpeedSaveDataSystem` -> `SystemUpdatePhase.Deserialize` - initialises the
  per-city JSON side-car when a save loads (`repo/Mod.cs#L36`).
- `RoadSpeedToolSystem` -> `SystemUpdatePhase.ToolUpdate` - the custom drag-select
  tool (`repo/Mod.cs#L39`).
- `RoadSpeedApplySystem` -> `SystemUpdatePhase.ModificationEnd` - re-applies the
  durable write onto freshly rebuilt sublanes (`repo/Mod.cs#L42`).
- `ClearCustomSpeedsSystem` -> `SystemUpdatePhase.ModificationEnd` - services the
  "Clear All" settings action (`repo/Mod.cs#L45`).
- `RoadSpeedToolUISystem` -> `SystemUpdatePhase.UIUpdate` - the in-panel React
  slider, an `InfoSectionBase` (`repo/Mod.cs#L48`).
- `SpeedLimitRenderSystem` -> `SystemUpdatePhase.Rendering` - the direct-mesh text
  overlay (`repo/Mod.cs#L51`).

Data flow: the tool collects a selection and hands it to the UI system; the UI
system (or the tool) writes both the vanilla lane fields *and* a transient
`CustomSpeed` marker component, and records a default+current entry in the JSON
side-car; `RoadSpeedApplySystem` watches for `Updated` edges carrying `CustomSpeed`
and re-stamps the lane speed so a network rebuild mid-session cannot silently drop
the override; the render system draws the overlay for any edge carrying
`CustomSpeed` while the tool is active.

### The three-layer persistence model

This is the heart of the mod. The same logical fact ("this road is 80 km/h") is
represented in three places with three lifetimes:

1. **Durable vanilla component (survives save/load + uninstall).** The new speed is
   written into stock `CarLane`/`TrackLane` fields. CS2 serializes those components
   as part of the network, so the value is baked into the save independent of the
   mod.
2. **Transient `CustomSpeed` ECS marker (session-scoped).** A mod component that
   flags "this edge has an override" - it drives the re-apply query, the overlay
   query, and the panel's current-value readback. It is not re-created from the JSON
   at load.
3. **Reset-only JSON side-car (persists on disk, not auto-applied).** A per-city
   file recording each road's *default* speed (for reset) plus its current speed.
   Nothing reads it back to re-apply at load; it exists so "reset to original" still
   works later.

`CustomSpeed` itself implements `IComponentData`, `IQueryTypeParameter`, and
`ISerializable`, with real `Serialize`/`Deserialize` bodies
(`repo/Components/CustomSpeed.cs#L7-L33`). But the mod registers no serialization
system that persists it: the file that was meant to be the ECS save component,
`Components/RoadSpeedSaveData.cs`, is an empty namespace with no type
(`repo/Components/RoadSpeedSaveData.cs#L1-L7`), and `SpeedDataManager`'s own comment
still claims data "is persisted via the RoadSpeedSaveData component" while actually
being two in-memory dictionaries cleared per session
(`repo/Data/SpeedDataManager.cs#L6-L12`). The `Deserialize`-phase handler only
initialises the JSON file - it never re-applies speeds or re-adds markers
(`repo/Systems/RoadSpeedSaveDataSystem.cs#L32-L50`). The net effect is that after a
reload the durable lane values are still present, but the overlay and reset paths
(which key off the `CustomSpeed` marker + a live selection) stay inert until the
road is re-selected with the tool. Whether the `ISerializable` marker actually
round-trips through a CS2 save is a runtime question - see the Needs-Verification
note below.

## Techniques demonstrated

- [Durable network component write](../how-to/recipes/durable-network-component-write.md)
  (family AN) - the restore job mutates only the speed fields of the vanilla lanes
  and re-assigns the cached flags verbatim, and deliberately does NOT re-assign the
  `SubLane` buffer element (a comment warns that reassigning it "breaks
  connectivity"):

  ```csharp
  // repo/Systems/RoadSpeedApplySystem.cs#L109-L137 (SetSpeedSubLane)
  if (this.EntityManager.HasComponent<CarLane>(laneEntity))
  {
      var carLane = this.EntityManager.GetComponentData<CarLane>(laneEntity);
      var originalFlags = carLane.m_Flags;           // preserve flags EXPLICITLY
      carLane.m_DefaultSpeedLimit = speedGameUnits;  // ONLY modify speed fields
      carLane.m_SpeedLimit = speedGameUnits;
      carLane.m_Flags = originalFlags;
      this.EntityManager.SetComponentData(laneEntity, carLane);
  }
  else if (this.EntityManager.HasComponent<Game.Net.TrackLane>(laneEntity))
  {
      var trackLane = this.EntityManager.GetComponentData<Game.Net.TrackLane>(laneEntity);
      trackLane.m_SpeedLimit = speedGameUnits;
      this.EntityManager.SetComponentData(laneEntity, trackLane);
  }
  ```

  The `/1.8` conversion is the CS2 unit quirk: game speed is stored as `2 * m/s`, so
  km/h -> game is `/3.6` (to m/s) then `*2`, i.e. `/1.8`; reading back multiplies by
  `1.8` (`repo/Systems/RoadSpeedApplySystem.cs#L92-L94`;
  `repo/Systems/RoadSpeedToolSystem.cs#L403`). The re-apply itself is a
  `[BurstCompile] IJobChunk` (`RestoreSpeedJob`) filtered to edges that carry both
  `CustomSpeed` and `Updated`, so a mid-session lane rebuild cannot silently drop
  the override - the durable write is *verified and re-stamped* rather than assumed
  (`repo/Systems/RoadSpeedApplySystem.cs#L24-L34`,
  `repo/Systems/RoadSpeedApplySystem.cs#L55-L87`). See also
  [verify-and-reapply-override](../how-to/recipes/verify-and-reapply-override.md).

- [Custom drag-select tool](../how-to/recipes/tool-drag-select.md) (family V) -
  `RoadSpeedToolSystem : ToolBaseSystem` raycasts the Net type mask across five net
  layers and paints a `Highlighted` selection as the player drags:

  ```csharp
  // repo/Systems/RoadSpeedToolSystem.cs#L145-L152 (InitializeRaycast)
  m_ToolRaycastSystem.typeMask = TypeMask.Net;
  m_ToolRaycastSystem.netLayerMask = Layer.Road | Layer.TrainTrack | Layer.TramTrack
                                   | Layer.SubwayTrack | Layer.Waterway;
  m_ToolRaycastSystem.raycastFlags = RaycastFlags.SubElements | RaycastFlags.Markers;
  ```

  Drag lifecycle is driven by `applyAction.WasPressedThisFrame()` /
  `IsPressed()` / `WasReleasedThisFrame()`, accumulating a `HashSet<Entity>` and
  toggling `Highlighted` + `BatchesUpdated` per edge; release calls
  `FinalizeSelection`, which hands the aggregate + edge list to the UI system
  (`repo/Systems/RoadSpeedToolSystem.cs#L182-L347`).

- [Direct-mesh render overlay](../how-to/recipes/render-pipeline-overlay.md)
  (family I) - rather than the overlay buffer, `SpeedLimitRenderSystem :
  GameSystemBase` hooks `RenderPipelineManager.beginContextRendering`
  (`repo/Systems/SpeedLimitRenderSystem.cs#L91`) and issues `Graphics.DrawMesh` per
  camera (`repo/Systems/SpeedLimitRenderSystem.cs#L207-L239`). It borrows the shared
  `TextMeshPro` instance from `OverlayRenderSystem.GetTextMesh()`, bakes a Unity
  `Mesh` from the generated `TMP_MeshInfo` (vertices/triangles/uvs/colors), clones
  the SDF material and copies the font-atlas parameters, then caches one mesh per
  speed value (`repo/Systems/SpeedLimitRenderSystem.cs#L249-L341`). Rendering is
  gated to when the tool is active and overlays aren't hidden
  (`repo/Systems/SpeedLimitRenderSystem.cs#L164-L174`), with a `sqrt`-based
  distance scale so the text grows as the camera pulls back
  (`repo/Systems/SpeedLimitRenderSystem.cs#L214-L224`).

- [JSON side-car persistence](../how-to/recipes/json-sidecar-persistence.md)
  (family F) - `PersistentSpeedStorage` writes one file per city under
  `.../ModsData/RoadSpeedAdjuster/<city>.json`
  (`repo/Data/PersistentSpeedStorage.cs#L28-L30`,
  `repo/Data/PersistentSpeedStorage.cs#L48-L55`), auto-saving on every store
  (`repo/Data/PersistentSpeedStorage.cs#L84-L104`). Each entry is keyed by the raw
  `Entity.Index` of the edge (`StoreRoadSpeed(targetEdge.Index, ...)`,
  `repo/Systems/RoadSpeedToolSystem.cs#L429`) and stores both the pre-edit default
  (for reset) and the current speed. This is a *fragile* key - see pitfalls.

- [Info-panel React binding](../how-to/recipes/uisystembase-react-binding.md)
  (family P) - `RoadSpeedToolUISystem : ExtendedInfoSectionBase` (an
  `InfoSectionBase`) is `[UpdateInGroup(typeof(SelectedInfoUISystem))]` and inserts
  itself with `m_InfoUISystem.AddMiddleSection(this)`
  (`repo/Systems/RoadSpeedToolUISystem.cs#L24-L56`). The `ExtendedInfoSectionBase`
  helper wraps Colossal's binding API into terse `CreateBinding` / `CreateTrigger`
  factories (`repo/Extensions/ExtendedInfoSectionBase.cs#L20-L105`). The panel binds
  a float speed value and an `APPLY_SPEED` trigger whose handler clamps to
  `[5, 240]` km/h before delegating to the tool
  (`repo/Systems/RoadSpeedToolUISystem.cs#L66-L74`,
  `repo/Systems/RoadSpeedToolUISystem.cs#L337-L344`). The slider control itself
  reuses the vanilla `Slider` through the `VanillaComponentResolver`
  (`repo/RoadSpeedAdjuster/src/slider/slider.tsx#L1-L13`), so RSA styles the panel
  with the game's own widget rather than a bespoke one (see also
  [vanilla-ui-augmentation](../how-to/recipes/vanilla-ui-augmentation.md)).

## Key decisions & tradeoffs

- **Write live net data, not the prefab, for uninstall-survival.** The whole point
  is that the value lives in the serialized network, not in a mod component. Writing
  the prefab's default speed would only affect newly placed segments and would
  revert on uninstall; writing the live `CarLane`/`TrackLane` bakes the change into
  *this* save's existing roads (`repo/Systems/RoadSpeedApplySystem.cs#L109-L137`).
- **Flags-preserving, speed-only mutation.** Every write caches `m_Flags` and
  restores it verbatim, and never re-assigns the `SubLane` buffer element, to avoid
  corrupting lane connectivity or lane semantics while changing one float
  (`repo/Systems/RoadSpeedApplySystem.cs#L104-L124`).
- **Three lifetimes on purpose.** Durability comes from the vanilla write; the ECS
  marker is a cheap session-scoped query handle for overlay + re-apply; the JSON
  side-car is the *only* record of the pre-edit default, and it is deliberately
  reset-only (never re-applied at load) so the mod does not fight CS2's own restored
  values (`repo/Systems/RoadSpeedSaveDataSystem.cs#L32-L50`).
- **Reset is a first-class, explicit path.** Because the write is durable, "undo"
  cannot be "remove the mod component" - it must actively restore the stored default
  and then drop the marker. Both the per-selection reset and the settings-driven
  "Clear All" do exactly that, pulling the default from the memory cache or JSON
  before writing it back and removing `CustomSpeed`
  (`repo/Systems/RoadSpeedToolSystem.cs#L478-L544`,
  `repo/Systems/ClearCustomSpeedsSystem.cs#L47-L140`).
- **Reuse vanilla UI over custom widgets.** The slider is the game's own `Slider`
  via `VanillaComponentResolver`, and the panel is a standard middle info-section,
  so the feature reads as native (`repo/RoadSpeedAdjuster/src/slider/slider.tsx#L1-L13`).

## Pitfalls / upstream-watch

- **Uninstall asymmetry - you must "Clear All" first.** Durability cuts both ways:
  a durable write with no owning mod is unrecoverable through the mod. If you
  uninstall RSA while roads still carry custom speeds, the speeds remain baked into
  the save and there is no in-game control left to revert them. The intended
  workflow is the settings action `ClearAllCustomSpeeds`, which routes through
  `ClearCustomSpeedsSystem` to restore defaults *before* removal
  (`repo/sETTING.cs#L45-L52`, `repo/sETTING.cs#L82-L102`;
  `repo/Systems/ClearCustomSpeedsSystem.cs#L47-L127`).
- **`Entity.Index` is a fragile persistence key.** The JSON side-car keys entries by
  `Entity.Index` alone. Indices are recycled when entities are destroyed and
  recreated, so a stored default can end up associated with a different edge across
  sessions. The alternate clear path even *reconstructs* entities by assuming
  `Version = 1` (`new Entity { Index = roadEntry.Key, Version = 1 }`), which is not a
  safe assumption (`repo/Systems/RoadSpeedToolSystem.cs#L576`;
  `repo/Data/PersistentSpeedStorage.cs#L84-L104`).
- **Overlay + reset go inert after reload until re-select.** Nothing re-applies the
  JSON or re-adds `CustomSpeed` on load (`repo/Systems/RoadSpeedSaveDataSystem.cs#L32-L50`),
  so a freshly loaded city shows the durable speeds but no overlay, and the wired
  "Clear All" queries live `CustomSpeed` components - if the markers did not persist,
  it finds nothing to clear even though the lanes still carry overrides. Re-selecting
  the road re-establishes the marker.
- **Dead ECS-save scaffolding.** `RoadSpeedSaveData.cs` is an empty type and
  `SpeedDataManager`'s "persisted via the RoadSpeedSaveData component" comment is
  stale; the real persistence is the durable write + JSON, not an ECS save component
  (`repo/Components/RoadSpeedSaveData.cs#L1-L7`; `repo/Data/SpeedDataManager.cs#L6-L12`).
- **CS2 net-serialization dependency.** The entire durability guarantee rests on CS2
  continuing to serialize `CarLane.m_SpeedLimit` / `TrackLane.m_SpeedLimit` the way
  it does today. A change to how the game stores or recomputes lane speed on load
  (e.g. re-deriving speed from prefab at deserialize) would break the persistence
  silently. The mod is also unmaintained relative to the moving game.
- **Multi-writer conflicts.** Any other mod that also writes lane speed limits, or
  rebuilds lanes without an `Updated` tag RSA can see, can overwrite or desync RSA's
  values; RSA's own re-apply only fires on `Updated` edges carrying `CustomSpeed`
  (`repo/Systems/RoadSpeedApplySystem.cs#L24-L34`).

Needs Verification (in-game): whether the `CustomSpeed` marker (which implements
`ISerializable`, `repo/Components/CustomSpeed.cs#L23-L33`) actually round-trips
through a CS2 save given that the mod registers no serialization system for it;
consequently, whether the overlay and the wired "Clear All" work immediately after a
reload or only after re-selecting each road. The durable *lane* values persisting is
expected from the vanilla write, but the marker's save/load survival cannot be
confirmed from source alone.

## Source pointers

- Dossier: `../../vice-and-order-research/mods/dossiers/road-speed-adjuster/`
  (`index.md`, `source.md`, `modding.md`, `guide.md`, `notes/`).
- Repo @ `e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed`, key files:
  - `repo/Systems/RoadSpeedApplySystem.cs` - durable flags-preserving lane write +
    Burst re-apply job (family AN).
  - `repo/Systems/RoadSpeedToolSystem.cs` - `ToolBaseSystem` drag-select, apply/reset
    (family V), JSON key usage.
  - `repo/Systems/SpeedLimitRenderSystem.cs` - direct-mesh TMP overlay via
    `beginContextRendering` (family I).
  - `repo/Data/PersistentSpeedStorage.cs` - per-city JSON side-car (family F).
  - `repo/Systems/RoadSpeedToolUISystem.cs`, `repo/Extensions/ExtendedInfoSectionBase.cs`,
    `repo/RoadSpeedAdjuster/src/slider/slider.tsx` - info-panel React slider (family P).
  - `repo/Components/CustomSpeed.cs`, `repo/Components/RoadSpeedSaveData.cs`,
    `repo/Data/SpeedDataManager.cs`, `repo/Systems/RoadSpeedSaveDataSystem.cs` -
    the three-layer persistence model and its dead ECS-save scaffolding.
  - `repo/Systems/ClearCustomSpeedsSystem.cs`, `repo/sETTING.cs` - the mandatory
    "Clear All" reversal path and its wiring.
  - `repo/Mod.cs` - system registration and `SystemUpdatePhase` scheduling.
