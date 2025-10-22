# Raycast Filters For Tools

Reference mod: `BetterBulldozer/Tools/SubElementBulldozerTool.cs`, `BetterBulldozer/Systems/BetterBulldozerUISystem.cs`.

Use this pattern when you need a tool (bulldozer, object, net, area) to only interact with specific entity families such as invisible markers, surfaces, or standalone net lanes.

## 1. Extend The Tool

Derive a system from `ToolBaseSystem` and override `InitializeRaycast()`:

- Set `m_ToolRaycastSystem.typeMask` to the categories you want (`TypeMask.Net`, `TypeMask.StaticObjects`, `TypeMask.Area`, etc).
- Restrict the network layers with `m_ToolRaycastSystem.netLayerMask` (for example `Layer.MarkerPathway`, `Layer.PowerlineLow`).
- Narrow utilities through `m_ToolRaycastSystem.utilityTypeMask` (`UtilityTypes.SewagePipe`, `UtilityTypes.HighVoltageLine`, etc).
- Toggle special cases via `m_ToolRaycastSystem.raycastFlags` (e.g., `RaycastFlags.Markers`, `RaycastFlags.UpgradeIsMain`).

Re-run `InitializeRaycast()` whenever the filter changes so the raycaster applies the new mask immediately.

## 2. Bind Filter Options To The UI

- Create one or more `ValueBinding<int>` (or `ValueBindingHelper<TEnum>`) inside an `ExtendedUISystemBase`.
- When the player selects a new filter, store the enum value in your settings object and force the tool pipeline to rebuild. Toggling the active tool via `ToolSystem` is a reliable way to do this.
- Surface tooltips so other developers understand which mask combination each filter represents.

## 3. Manage Hidden Markers

When targeting invisible paths, markers, or temp geometry:

- Cache `RenderingSystem.markersVisible`, set it to `true` while the filter is active, and restore the cached value afterwards.
- If your active tool normally requires `AreaTypeMask` flags, temporarily add `AreaTypeMask.Spaces` with `SetMemberValue("requireAreas", updatedMask)` so the raycast still succeeds. Remove the flag when the filter is cleared.

## 4. Persist Defaults

- Store the last selected filter in your `ModSetting`.
- Call `ApplyAndSave()` when the player changes the binding so the bulldozer (or your custom tool) restores the choice on next load.

## 5. Stack With Vanilla Filters

The vanilla bulldozer already exposes filters (buildings, roads, surfaces). Read and update the underlying bitmask through `ToolUISystem` bindings so your custom filters combine with the base game instead of replacing it. This keeps the UX predictable and avoids conflicting category settings.
