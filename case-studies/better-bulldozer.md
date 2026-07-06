---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Case study: Better Bulldozer"
case_study: better-bulldozer
mod: "Better Bulldozer (75250)"
dossier: ../../vice-and-order-research/mods/dossiers/better-bulldozer/
repo_commit: 4408466f226db811159d92859479ae1e1c28ba06
source_version: "n/a - no pinned source"
last_reverified: "2026-07-04"
diataxis: explanation
techniques: [V, Y, P, A]
technique_applicability: [tooling, ui]
status: source-verified
Created: 2026-07-03
Updated: 2026-07-03
Owners:
  - codex
---

# Better Bulldozer - case study

> Better Bulldozer extends the vanilla bulldozer instead of replacing it: two Harmony
> patches retarget the stock tool's raycast at markers, lanes, areas, and moving
> objects; two custom `ToolBaseSystem` tools borrow the vanilla bulldozer's own
> `toolID` and prefab; and every destructive edit is deferred across frames through
> `DeleteInXFrames` on a `ToolOutputBarrier` so other systems see clean transitions.
> It is the reference example of "augment a vanilla tool" plus event-driven,
> reversible demolition.

All code claims below are cited to `repo/<path>#Lxx` at pinned commit
`4408466f226db811159d92859479ae1e1c28ba06` (branch `master`, AssemblyVersion 1.3.18,
built for game v1.5.7f1, Paradox modVersion 25), surfaced through the dossier at
`../../vice-and-order-research/mods/dossiers/better-bulldozer/`. An earlier dossier pass
wrongly claimed the mod avoided Harmony in favour of phase scheduling; that is
corrected here - it uses **both** (`repo/BetterBulldozer/BetterBulldozerMod.cs#L104-L105`).

## What it does / why it's instructive

Better Bulldozer (author yenyang, Paradox ModId 75250) turns the vanilla bulldozer into
a precision demolition tool. It adds toolbar toggles to raycast invisible markers,
network lanes, surfaces/areas, and vehicles/cims/animals; a Sub-Element Bulldozer for
peeling props, trees, lights, and sub-networks off buildings; a radius/single tool for
wiping moving entities; and settings-driven automation that strips fences, hedges,
branding props, and manicured grass as they spawn - each reversible through a Restore
pass.

It is instructive for four reasons:

1. **Augment, do not replace, the vanilla tool.** The core behaviors are added by
   Harmony-patching the vanilla bulldozer's raycast, and by two custom tools that
   delegate `toolID`/`GetPrefab`/`TrySetPrefab` to the stock `BulldozeToolSystem`. The
   toolbar button and keybinds stay vanilla while the mod injects filters and selection
   logic - a much smaller surface than shipping a replacement tool.
2. **Harmony and deterministic scheduling used together.** The mod pins ~15 ECS systems
   into explicit `SystemUpdatePhase` slots *and* applies two Harmony patches. They are
   not alternatives; each does what the other cannot.
3. **Frame-deferred, barrier-scoped deletion.** Nothing is destroyed inline. A
   `DeleteInXFrames` countdown on a `ToolOutputBarrier` command buffer - debounced on
   owner regeneration - lets other systems watching for `Deleted` see clean transitions.
4. **Reversibility as a first-class concern.** Removed sub-elements are recorded in
   versioned `ISerializable` buffers so a Restore/Safely-Remove pass can rebuild them,
   and so removals survive save/reload.

## Architecture at a glance

`BetterBulldozerMod.OnLoad` loads settings and localization, applies Harmony patches,
then schedules every system into a specific update lane
(`repo/BetterBulldozer/BetterBulldozerMod.cs#L76-L130`):

```csharp
m_Harmony = new Harmony("Mods_Yenyang_Better_Bulldozer");
m_Harmony.PatchAll();
...
updateSystem.UpdateAt<BetterBulldozerUISystem>(SystemUpdatePhase.UIUpdate);
updateSystem.UpdateAt<SubElementBulldozerTool>(SystemUpdatePhase.ToolUpdate);
updateSystem.UpdateAt<HandleDeleteInXFramesSystem>(SystemUpdatePhase.ToolUpdate);
updateSystem.UpdateAt<AutomaticallyRemoveFencesAndHedges>(SystemUpdatePhase.ModificationEnd);
updateSystem.UpdateAt<CleanUpOwnerRecordsSystem>(SystemUpdatePhase.Deserialize);
updateSystem.UpdateAt<HandleUpdateNextFrameSystem>(SystemUpdatePhase.Modification5);
```
(`repo/BetterBulldozer/BetterBulldozerMod.cs#L104-L128`)

The lanes are load-bearing: UI in `UIUpdate`, interactive tools and the delete/restore
handlers in `ToolUpdate`, spawn-time automation in `ModificationEnd`, deferred
re-tagging in `Modification5`, and owner-record cleanup in `Deserialize`.

### Two Harmony patches retarget the vanilla bulldozer's raycast

`BulldozeToolSystemInitializeRaycastPatch` is a **Postfix** on
`BulldozeToolSystem.InitializeRaycast`. Because it runs *after* the vanilla setup, it
overwrites the `ToolRaycastSystem` mask fields per the UI's selected target - so the
stock bulldozer itself gains marker/area/lane/moving-object modes with no custom tool
(`repo/BetterBulldozer/Patches/BulldozeToolSystemInitializeRaycastPatch.cs#L20-L94`):

```csharp
else if (betterBulldozerUISystem.SelectedRaycastTarget == RaycastTarget.Lanes)
{
    toolRaycastSystem.typeMask = TypeMask.Net;
    toolRaycastSystem.netLayerMask = Layer.Fence | Layer.LaneEditor;
    toolRaycastSystem.raycastFlags |= RaycastFlags.Markers | RaycastFlags.EditorContainers;
}
```
(`repo/BetterBulldozer/Patches/BulldozeToolSystemInitializeRaycastPatch.cs#L58-L63`)

`ToolBaseSystemGetRaycastResultPatch` is a **Prefix** on the two-out-parameter overload
`ToolBaseSystem.GetRaycastResult(out Entity, out RaycastHit)`. It runs only for the
Lanes/Vanilla targets and returns `false` (skipping the vanilla method) when a hit must
be rejected by the active vanilla-filter set - classifying hits by walking
`PrefabRef -> SubMesh -> MeshData.m_State & MeshFlags.Decal` for decals and by component
presence (Building/Tree/Plant/Object+Static) otherwise
(`repo/BetterBulldozer/Patches/ToolBaseSystemGetRaycastResultPatch.cs#L20-L132`). The
two patches target internal vanilla signatures, which is the mod's main breakage vector
on a base-game refactor.

### Custom tools borrow the vanilla bulldozer

`RemoveVehiclesCimsAndAnimalsTool` and `SubElementBulldozerTool` both extend
`ToolBaseSystem` but delegate identity to the vanilla bulldozer so the toolbar and
keybinds stay vanilla (`repo/BetterBulldozer/Tools/RemoveVehiclesCimsAndAnimalsTool.cs#L32-L74`):

```csharp
public override string toolID => m_BulldozeToolSystem.toolID;
public override PrefabBase GetPrefab() => m_BulldozeToolSystem.GetPrefab();
public override bool TrySetPrefab(PrefabBase prefab)
{
    if (m_BetterBulldozerUISystem.VCAToolActive && prefab is BulldozePrefab bulldozePrefab)
    {
        m_BulldozeToolSystem.prefab = bulldozePrefab;
        return true;
    }
    return false;
}
```

`BetterBulldozerUISystem.OnGameLoadingComplete` inserts both custom tools at index 0 of
`m_ToolSystem.tools` so they win tool resolution
(`repo/BetterBulldozer/Systems/BetterBulldozerUISystem.cs#L365-L368`). The VCA tool runs
Burst entity queries against moving/parked objects and issues deletions through a
`ToolOutputBarrier` (`repo/BetterBulldozer/Tools/RemoveVehiclesCimsAndAnimalsTool.cs#L35`).

### Deletion is deferred and debounced across frames

`HandleDeleteInXFramesSystem` runs a Burst `IJobChunk` over
`WithAll<DeleteInXFrames, Owner>().WithNone<Temp, Deleted>()` on the `ToolOutputBarrier`.
Per entity: if the **owner currently has `Updated`** (it is regenerating) the countdown
is reset to 30 - a debounce, not a plain timer - otherwise when it reaches zero the entity
gets `Deleted`, else the counter decrements
(`repo/BetterBulldozer/Systems/HandleDeleteInXFramesSystem.cs#L42-L108`):

```csharp
if (m_UpdatedLookup.HasComponent(ownerNativeArray[i].m_Owner))
    buffer.SetComponent(entityNativeArray[i], new DeleteInXFrames { m_FramesRemaining = 30 });
else if (deleteInXFrames.m_FramesRemaining <= 0)
    buffer.AddComponent<Deleted>(entityNativeArray[i]);
else { deleteInXFrames.m_FramesRemaining--; buffer.SetComponent(...); }
```
(`repo/BetterBulldozer/Systems/HandleDeleteInXFramesSystem.cs#L91-L106`)

The spawn-time automation feeds the same component: `AutomaticallyRemoveFencesAndHedges`
runs a Burst `GatherSubLanesJob`, then a `HandleDeleteInXFramesJob` enqueues
`DeleteInXFrames` through the `ModificationEndBarrier` so decorative networks are stripped
right after they spawn (`repo/BetterBulldozer/Systems/AutomaticallyRemoveFencesAndHedgesSystem.cs#L164-L190`).

### Removals persist and are reversible

`OwnerRecord` and `PermanentlyRemovedSubElementPrefab` implement `ISerializable` with a
leading version int, so "permanently removed" sub-elements survive save/reload and the
Restore/Safely-Remove systems can rebuild them
(`repo/BetterBulldozer/Components/OwnerRecord.cs#L13-L44`):

```csharp
public void Serialize<TWriter>(TWriter writer) where TWriter : IWriter
{ writer.Write(1); writer.Write(m_Owner); }
```

`DeleteInXFrames` itself is a plain `IComponentData` (not serialized), so an in-flight
countdown does not survive a reload - only the record of what was removed does.

## Techniques demonstrated

- [Tool drag-select + Highlighted components](../how-to/recipes/tool-drag-select.md)
  (family V) - two `ToolBaseSystem` derivatives add Single/Radius selection over the
  vanilla bulldozer. `RemoveVehiclesCimsAndAnimalsTool` maintains moving/parked queries
  and a UI-driven radius, deleting via `ToolOutputBarrier`
  (`repo/BetterBulldozer/Tools/RemoveVehiclesCimsAndAnimalsTool.cs#L32-L74`); the radius
  triggers step by 1 below 10 and by 10 above it
  (`repo/BetterBulldozer/Systems/BetterBulldozerUISystem.cs#L329-L350`).
- [Event-driven mod-state write (ModificationEnd / ToolOutputBarrier)](../how-to/recipes/event-driven-modificationend.md)
  (family Y) - deletions are never inline. `HandleDeleteInXFramesSystem` (ToolUpdate,
  `ToolOutputBarrier`) debounces on owner regeneration
  (`repo/BetterBulldozer/Systems/HandleDeleteInXFramesSystem.cs#L42-L108`); automation
  runs at `ModificationEnd` and enqueues through the `ModificationEndBarrier`
  (`repo/BetterBulldozer/Systems/AutomaticallyRemoveFencesAndHedgesSystem.cs#L164-L190`);
  `HandleUpdateNextFrameSystem` re-tags at `Modification5`
  (`repo/BetterBulldozer/BetterBulldozerMod.cs#L128`).
- [UISystemBase / React binding](../how-to/recipes/uisystembase-react-binding.md)
  (family P) - `BetterBulldozerUISystem` (a custom `ExtendedUISystemBase`) owns the
  binding surface: `ValueBinding`s for `RaycastTarget`, `AreasFilter`, `MarkersFilter`,
  `SelectedVanillaFilters`, and `SelectionRadius`, plus `TriggerBinding`/`CreateTrigger`
  routes for every toolbar button, mirrored one-for-one by the React module
  (`repo/BetterBulldozer/Systems/BetterBulldozerUISystem.cs#L302-L351`).
- [Prefab-field / component-data override](../how-to/recipes/prefab-field-override.md)
  (family A) - the family-A idea (re-assert values the game reads) appears here as the
  Harmony Postfix overwriting `ToolRaycastSystem` mask fields (`typeMask`,
  `netLayerMask`, `raycastFlags`, `areaTypeMask`) from live UI state
  (`repo/BetterBulldozer/Patches/BulldozeToolSystemInitializeRaycastPatch.cs#L37-L93`),
  and as command-buffer component writes (`DeleteInXFrames`, `Deleted`, `OwnerRecord`)
  onto placed entities (`repo/BetterBulldozer/Systems/HandleDeleteInXFramesSystem.cs#L91-L106`).
  Note the scope: Better Bulldozer overrides *live tool-system fields and entity
  components*, not the shared-prefab authoring data of the canonical family-A recipe -
  see the recipe for the `PrefabSystem.AddComponentData` form.

See the [technique index](../technique-index.md) for the family ledger. Related
explanation pages: [multi-phase scheduling](../explanation/multi-phase-scheduling.md),
[system scheduling](../explanation/system-scheduling.md),
[UI <-> C# communication](../explanation/ui-cs-communication.md), and
[serialization](../explanation/serialization.md).

## Key decisions & tradeoffs

- **Postfix retarget vs. custom raycast per mode.** Overwriting the vanilla
  `ToolRaycastSystem` masks in a Postfix means the *stock* bulldozer gains marker/lane/
  area/moving-object targeting with no replacement tool
  (`repo/BetterBulldozer/Patches/BulldozeToolSystemInitializeRaycastPatch.cs#L20-L94`).
  The cost is binding to an internal method name - the documented breakage vector on a
  base-game refactor.
- **Borrow the vanilla `toolID` vs. register a new tool.** Delegating identity keeps the
  toolbar button and keybinds vanilla and avoids reimplementing prefab wiring
  (`repo/BetterBulldozer/Tools/RemoveVehiclesCimsAndAnimalsTool.cs#L46-L74`); the price
  is inserting the custom tools ahead of vanilla in `m_ToolSystem.tools` so they win
  resolution (`repo/BetterBulldozer/Systems/BetterBulldozerUISystem.cs#L365-L368`).
- **Deferred, debounced deletion vs. immediate destroy.** Routing deletions through
  `DeleteInXFrames` on a barrier - and resetting the countdown while the owner is
  `Updated` - keeps the mod compatible with systems watching for `Deleted`/`Updated` and
  avoids deleting a sub-element mid-regeneration
  (`repo/BetterBulldozer/Systems/HandleDeleteInXFramesSystem.cs#L91-L106`).
- **Versioned ISerializable records vs. fire-and-forget.** Persisting `OwnerRecord` /
  `PermanentlyRemovedSubElementPrefab` (each with a leading version int) makes removals
  reversible and reload-safe, at the cost of a `Deserialize`-phase cleanup pass
  (`CleanUpOwnerRecordsSystem`) to avoid orphaned buffers
  (`repo/BetterBulldozer/Components/OwnerRecord.cs#L13-L44`,
  `repo/BetterBulldozer/BetterBulldozerMod.cs#L123`).
- **Safety toggles default-on; automation default-off.** `AllowRemovingSubElementNetworks`
  / `AllowRemovingExtensions` ship enabled as guardrails you can lock off, while the
  spawn-time automation ships disabled and auto-enables the matching Restore system when
  turned off in-game (per `BetterBulldozerModSettings`,
  `repo/BetterBulldozer/Settings/BetterBulldozerModSettings.cs#L35-L99`).

## Pitfalls / upstream-watch

- **Harmony signature fragility.** Both patches bind internal vanilla methods -
  `BulldozeToolSystem.InitializeRaycast` and the two-out-parameter
  `ToolBaseSystem.GetRaycastResult` - so a base-game refactor of either silently breaks
  the affected modes and forces a rebuild (as at v1.3.12 and v1.3.18)
  (`repo/BetterBulldozer/Patches/ToolBaseSystemGetRaycastResultPatch.cs#L20`).
- **In-flight countdowns do not survive reload.** `DeleteInXFrames` and `UpdateNextFrame`
  are plain `IComponentData`; only `OwnerRecord` / `PermanentlyRemovedSubElementPrefab`
  are serialized, so a delete scheduled just before a save is dropped on reload
  (`repo/BetterBulldozer/Systems/HandleDeleteInXFramesSystem.cs#L42-L48`).
- **Marker-visibility clobber with co-loaded mods.** The UI system caches
  `RenderingSystem.markersVisible` only on entry into marker mode and replays it on exit,
  so a mod that flips markers mid-mode (for example Anarchy) can be overwritten on exit
  (`repo/BetterBulldozer/Systems/BetterBulldozerUISystem.cs#L302-L312`). `Needs
  Verification (in-game)`: the exact interop outcome when both mods toggle markers.
- **Removing sub-element networks/extensions can break connectivity.** The README warns
  this despite the safety work; the `AllowRemovingSubElementNetworks` /
  `AllowRemovingExtensions` toggles exist to lock it off - keep "disable risky categories
  unless you know what you're doing" framing
  (`repo/BetterBulldozer/Settings/BetterBulldozerModSettings.cs#L35-L44`).
- **Automation perf on megacities.** Fences/hedges/branding automation runs Burst gather
  jobs at `ModificationEnd`, and the VCA radius delete runs a Burst job at `ToolUpdate`;
  large selections or dense saves are the profiling targets
  (`repo/BetterBulldozer/Systems/AutomaticallyRemoveFencesAndHedgesSystem.cs#L140-L190`).
  `Needs Verification (in-game)`: frame-time under a 100 m radius on a 5k+ sub-lane save
  (the dossier's QA plan is unexecuted, requiring a running CS2 session).
- **Post-load regeneration race (Traffic Mod interop).** Networks with custom
  intersection lane connections regenerate sub-elements on load;
  `RemoveRegeneratedSubelementPrefabsSystem` waits `PostLoadDelayFrames = 10`, reflects
  for `Traffic.Components.ModifiedConnections`, and re-tags affected entities via
  `UpdateNextFrame` - the reflection branch is skipped entirely when the Traffic Mod is
  absent (`repo/BetterBulldozer/Systems/RemoveRegeneratedSubelementPrefabsSystem.cs#L29-L46`).

## Source pointers

- Dossier: `../../vice-and-order-research/mods/dossiers/better-bulldozer/`
  (index / source / modding / guide + notes).
- Repo @ `4408466f226db811159d92859479ae1e1c28ba06` (branch `master`), key files:
  - `repo/BetterBulldozer/BetterBulldozerMod.cs` - Harmony `PatchAll` + phase scheduling.
  - `repo/BetterBulldozer/Patches/BulldozeToolSystemInitializeRaycastPatch.cs`,
    `.../ToolBaseSystemGetRaycastResultPatch.cs` - the two raycast patches.
  - `repo/BetterBulldozer/Systems/BetterBulldozerUISystem.cs` - binding surface + tool
    reorder.
  - `repo/BetterBulldozer/Tools/RemoveVehiclesCimsAndAnimalsTool.cs`,
    `.../SubElementBulldozerTool.cs` - vanilla-delegating custom tools.
  - `repo/BetterBulldozer/Systems/HandleDeleteInXFramesSystem.cs`,
    `.../AutomaticallyRemoveFencesAndHedgesSystem.cs` - deferred/debounced deletion.
  - `repo/BetterBulldozer/Components/OwnerRecord.cs`,
    `.../PermanentlyRemovedSubElementPrefab.cs` - versioned ISerializable records.
- Dependency: Lib.Harmony 2.2.2 (PackageReference) + Unified Icon Library (modId 74417);
  I18n Everywhere dropped in v1.3.11. Storefront modId 75250, modVersion 25 (1.3.18).
