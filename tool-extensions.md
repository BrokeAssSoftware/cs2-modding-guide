# Tool Extensions

Cities: Skylines II exposes the low-level tool systems that power the bulldozer, net, area, and object tools. This reference captures reusable patterns so Vice & Order can deliver advanced placement overrides, filtered demolition, and bespoke in-game tooling.

## Filtering Tool Raycasts

_Reference implementation: `BetterBulldozer/Tools/SubElementBulldozerTool.cs`, `BetterBulldozer/Systems/BetterBulldozerUISystem.cs`._

1. **Wrap the vanilla tool** by deriving from `ToolBaseSystem`. In `InitializeRaycast()` configure which entities can be hit:
   - `m_ToolRaycastSystem.typeMask` - choose the high-level categories (`TypeMask.Net`, `TypeMask.StaticObjects`, `TypeMask.Area`, etc).
   - `m_ToolRaycastSystem.netLayerMask` - limit the net layers (for example `Layer.MarkerPathway`, `Layer.PowerlineLow`).
   - `m_ToolRaycastSystem.utilityTypeMask` - restrict utility sub-types such as `UtilityTypes.SewagePipe` or `UtilityTypes.HighVoltageLine`.
   - `m_ToolRaycastSystem.raycastFlags` - enable special cases like `RaycastFlags.Markers` or `RaycastFlags.UpgradeIsMain`.

2. **Expose filter toggles** through `ValueBinding<int>` (or `ValueBindingHelper<T>`) in a UI system. When the player changes a filter, store the enum value and force the tool to re-run `InitializeRaycast()` - toggling the active tool via `ToolSystem` is a simple way to do this.

3. **Show hidden markers** while marker-only filters are active by setting `RenderingSystem.markersVisible = true`. Cache the previous state and restore it when the player leaves the mode. If you temporarily require hidden `AreaTypeMask.Spaces`, add the flag to `activeTool.requireAreas` with `SetMemberValue` and remove it afterward.

4. **Persist defaults** in your `ModSetting` so the bulldozer (or your custom tool) remembers the last filter the player selected.

5. **Stack with vanilla filters** by reading the base bulldozer flags through `ToolUISystem` bindings. This lets custom filters combine with Colossal's "buildings/roads/surfaces" switches instead of replacing them outright.

## Sub-Element Removal and Bulk Operations

_Reference implementation: `BetterBulldozer/Tools/SubElementBulldozerTool.cs`, `BetterBulldozer/Components/PermanentlyRemovedSubElementPrefab.cs`._

1. **Reuse the bulldozer prefab**: forward `GetPrefab()` and `TrySetPrefab()` to the active `BulldozeToolSystem` so the tool stays in sync with vanilla UI selections.

2. **Query sub-objects** via the `DynamicBuffer<Game.Objects.SubObject>` on the main entity and entity queries that target specific prefab families (trees, props, decals, hedge nets, and so on). Cache both the owning entities and their prefab entities.

3. **Highlight selections** by adding `Highlighted` and `BatchesUpdated` components with an `EntityCommandBuffer`. Store the highlighted entities in a `NativeList<Entity>` so you can un-highlight or delete them later in the frame.

4. **Delete safely** by enqueueing structural changes on the `ToolOutputBarrier`. A simple pattern is to tag each entity with a lightweight component (for example `DeleteInXFrames`) that a follow-up system consumes, ensuring removal happens after the current tool update.

5. **Track permanent removals** using a component such as `PermanentlyRemovedSubElementPrefab` that records which prefabs were stripped. Provide a reset button that re-spawns the recorded prefabs to restore the asset.

6. **Respect safety toggles**: add settings that disallow risky operations (e.g., removing network pieces) and verify the toggle before scheduling the deletion commands.

## Placement Validation and Error Overrides

_Reference implementation: `Anarchy`._

1. **Patch validation** with Harmony to filter out specific `ToolError` entries from `NetToolSystem`, `ObjectToolSystem`, or `BulldozeToolSystem`. Only drop errors (overlap, tight curve, shoreline, etc.) when the player has explicitly enabled the override.

2. **Share tool state** through a central service that watches `ToolSystem.activeTool` and exposes toggle state to both the UI (button icon, shortcut) and the patched methods.

3. **Add safeguards** so overrides disable themselves in scenarios the player should not touch (for example, automatically turning off anarchy while tree brushing).

4. **Adjust placement elevation** by intercepting the placement data (line/object tool command structs) and modifying their elevation fields before submission. Bind keyboard shortcuts (arrow keys, Shift+E, etc.) to change the offset and provide a reset action.

5. **Surface warnings** whenever validation is disabled - flashing icons, tooltips, or status text reduce accidental misuse and make it easy to confirm when the tool has returned to vanilla behavior.

## Transform Gizmos and Snap-to-Surface

_Reference implementation: `ExtraDetailingTools`._

1. **Listen for selection** events and display a transform panel that edits the selected entity's position, rotation, and scale. Synchronise fields with `ValueBinding<float>` so the Gameface UI stays up to date.

2. **Persist adjustments** by writing the new transform values back through the `ToolOutputBarrier` (either to a `Transform` or `TransformFrame` component). Quantise or clamp the values according to the player's chosen increment.

3. **Toggle surface snapping** by setting `ObjectToolSystem.snapToSurface` (or the equivalent for the active tool) and saving the preference in your settings file. Decal placement can also be snapped by changing the `RaycastFlags` used when constructing placement commands.

4. **Publish curated asset menus** by registering extra tabs through `ToolRegistry.AppendTab` (for surfaces, decals, net lanes). Selecting an entry calls `TrySetPrefab()` on the underlying tool, letting players access curated content directly from your mod UI.

## Shared Dependency Patterns

- **Shared libraries**: when multiple modules rely on the same helpers (for example ExtraLib), ship a dedicated dependency mod, declare it in `mod.json`, and fail gracefully if it is missing.
- **Icon packages**: use Unified Icon Library to reference SVGs through `coui://uil/<Style>/<Icon>.svg` instead of bundling duplicates.
- **Localization helpers**: integrate I18n Everywhere by dropping locale JSON files under a `lang/` directory; the dependency injects them into the localization manager automatically.

Documenting these patterns once lets Vice & Order reuse them across future tooling work without rediscovering the implementation details.
