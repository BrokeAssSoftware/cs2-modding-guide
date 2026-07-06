---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Case study: Anarchy"
case_study: anarchy
mod: "Anarchy (74604)"
dossier: ../../vice-and-order-research/mods/dossiers/anarchy/
repo_commit: a6311e898d20a775368668b234aaa32f06e3e1eb
source_version: "n/a - no pinned source"
last_reverified: "2026-07-04"
diataxis: explanation
techniques: [A, D, F, R, X, Y, O, S, V]
technique_applicability: [simulation, platform]
status: source-verified
Created: 2026-07-01
Updated: 2026-07-01
Owners:
  - codex
---

# Anarchy - case study

> Anarchy grants placement freedom not by patching each tool, but by flipping flag
> bits on shared vanilla `ToolErrorData` prefabs - a whole-game override that new tools
> inherit for free. That single decision, plus six narrowly-scoped Harmony patches, is
> the reason this ~1M-subscriber mod survives Colossal's rapid patch cadence.

## What it does / why it's instructive

Anarchy (yenyang, mod 74604) lets players suppress Cities: Skylines II placement
error checks on demand - overlap, steep slope, short distance, small area, and (since
the Bridges & Ports update) port access and clearance - while keeping safety-critical
checks such as pathfinding intact. On top of the toggle it layers network composition
overrides (constant slope, forced ground/elevated/tunnel, inline retaining-wall/quay/
tree upgrades), object elevation with transform locks, a bespoke components tool, and a
public mod-to-mod bridge.

It is an instructive teaching example for three reasons:

1. **Breadth without patch churn.** The core capability - the anarchy toggle - is
   implemented with *zero* per-tool Harmony patches. It edits shared prefab data
   through ECS, so any current or future tool that emits an error prefab is covered
   automatically. This is the cleanest available demonstration of "prefer ECS data
   edits over Harmony where the game already models the thing you want to change."
2. **Disciplined use of Harmony where ECS cannot reach.** The six Harmony patches it
   *does* ship are orthogonal to the error toggle and each targets a concern ECS
   cannot express (raycast layers, elevation range, unique-building tracking, toolbar
   wiring). It is a good model of scoping Harmony to the minimum.
3. **Longevity engineering.** Guardrails (moveable-bridge detection), a public bridge
   API, embedded-resource localization, and a version-tolerant default-error matrix
   are all visible responses to a real, fast-moving upstream game.

## Architecture at a glance

`AnarchyMod.OnLoad` is the single entrypoint: it loads settings, registers keybinds,
calls `m_Harmony.PatchAll()`, then schedules ~25 ECS systems into explicit
`SystemUpdatePhase` slots so the mod layers over vanilla tool execution deterministically
(`repo/Anarchy/AnarchyMod.cs#L125-L155`). The load-bearing scheduling:

```
updateSystem.UpdateAt<DisableToolErrorsSystem>(SystemUpdatePhase.Modification5);
updateSystem.UpdateAt<EnableToolErrorsSystem>(SystemUpdatePhase.ModificationEnd);
updateSystem.UpdateAt<ModifyNetCompositionDataSystem>(SystemUpdatePhase.Modification3);
updateSystem.UpdateAt<NetworkAnarchyUISystem>(SystemUpdatePhase.UIUpdate);
updateSystem.UpdateAt<AnarchyComponentsToolSystem>(SystemUpdatePhase.ToolUpdate);
```
(`repo/Anarchy/AnarchyMod.cs#L132-L151`)

### The anarchy toggle is a flag-flip, not per-tool patches

`DisableToolErrorsSystem` queries every prefab carrying `ToolErrorData`, asks the UI for
its allowlist via `GetAllowableErrorTypes()`, and OR-sets `ToolErrorFlags.DisableInGame`
and `ToolErrorFlags.DisableInEditor` on each matching prefab through a `ModificationBarrier5`
command buffer (`repo/Anarchy/Systems/ErrorChecks/DisableToolErrorsSystem.cs#L94-L113`):

```csharp
if (errorTypesToDisable.Contains(toolErrorData.m_Error) && /* not already disabled */)
{
    toolErrorData.m_Flags |= ToolErrorFlags.DisableInGame;
    toolErrorData.m_Flags |= ToolErrorFlags.DisableInEditor;
    buffer.SetComponent(currentEntity, toolErrorData);
}
```

`EnableToolErrorsSystem` clears the same bits (`&= ~ToolErrorFlags.DisableInGame` /
`DisableInEditor`) at `ModificationEnd` to restore vanilla checks when anarchy is off,
skipping `ExceedsLotLimits` and a "do not re-enable for editor" set
(`repo/Anarchy/Systems/ErrorChecks/EnableToolErrorsSystem.cs#L80-L95`). Because the flags
live on *shared vanilla prefabs*, every tool consumes them - the mod never touches an
individual tool's placement code. A separate branch suppresses the "Already Exists"
notification prefab when `AllowPlacingMultipleUniqueBuildings` is set
(`repo/Anarchy/Systems/ErrorChecks/DisableToolErrorsSystem.cs#L75-L91`).

### Six Harmony patches, all orthogonal to the toggle

`PatchAll()` applies exactly six patches, none of which touch the error checks:

- `NetToolSystem_InitializeRaycast` - postfix that ORs Pathway/TrainTrack/
  PublicTransportRoad/TramTrack/SubwayTrack layers into the replace-mode raycast mask
  (`repo/Anarchy/Patches/NetToolSystem_InitializeRaycast.cs#L31-L48`).
- `ToolUISystemGetElevationRangePatch` - expands the reported elevation range.
- `UniqueAssetTrackingSystemIsPlacedUniqueAssetPatch` and
  `UniqueAssetTrackingSystemOnCreatePatch` - allow duplicate unique buildings.
- `ToolbarUISystemActivatePrefabToolPatch` and `ToolbarUISystemOnUpdatePatch` - wire
  the Anarchy Components Tool into the vanilla toolbar.

(`repo/Anarchy/Patches/` at commit a6311e8.)

### Network composition read from a bitmask

`NetworkDefinitionSystem.OnUpdate` reads the `NetworkComposition` bitfield on
`NetworkAnarchyUISystem` and early-outs unless ConstantSlope / Ground / Tunnel /
Elevated is set; otherwise it rewrites `NetCourse` elevations before the vanilla
`NetToolSystem` finalizes geometry (`repo/Anarchy/Systems/NetworkAnarchy/NetworkDefinitionSystem.cs#L66-L114`):

```csharp
if ((m_UISystem.NetworkComposition & NetworkAnarchyUISystem.Composition.ConstantSlope) != ...ConstantSlope
    && (m_UISystem.NetworkComposition & ...Ground) != ...Ground
    && (m_UISystem.NetworkComposition & ...Tunnel) != ...Tunnel
    && (m_UISystem.NetworkComposition & ...Elevated) != ...Elevated)
{
    return;
}
```

Side upgrades (Quay/RetainingWall/Trees/BikeLane/BikeRestriction) are mapped onto
`CompositionFlags.Side` in `TempNetworkSystem`
(`repo/Anarchy/Systems/NetworkAnarchy/TempNetworkSystem.cs#L57-L63`).

### Clearance guard for moveable bridges

`ModifyNetCompositionDataSystem` (scheduled at `Modification3`) scans the active
prefab's `SubObject` buffer for `MoveableBridgeData`; when found - or when anarchy is
off, or a non-net tool is active - it resets any in-flight override and returns early so
Bridges & Ports lift bridges keep their vanilla animation/clearance data
(`repo/Anarchy/Systems/ClearanceViolation/ModifyNetCompositionDataSystem.cs#L78-L104`).

### Persistence: XML per-error, not a JSON profile

General toggles and keybinds persist through the `[FileLocation("Mods_Yenyang_Anarchy")]`
settings asset. The per-error three-state values persist *separately*: one
`<ErrorType>.xml` file per error, written by `XmlSerializer` under
`ModsData/<AnarchyMod.Id>/ErrorChecks/` (`repo/Anarchy/Systems/Common/AnarchyUISystem.cs#L243-L244`,
`#L671-L724`). If a value equals the default, the file is deleted rather than written
(`repo/Anarchy/Systems/Common/AnarchyUISystem.cs#L676-L691`). This is XML, not JSON.

### AnarchyBridge public API

`AnarchyBridge` is a `public static` class other mods call to register a
`ToolBaseSystem` into Anarchy's compatible-tool list or to add/remove Anarchy /
TransformLock components on entities (`repo/Anarchy/Bridge/AnarchyBridge.cs#L17-L45`).
`CopyAnarchyComponentsSystem` (scheduled at `Modification2`) integrates Move It Alpha
copy/paste of those components (`repo/Anarchy/AnarchyMod.cs#L153`).

## Techniques demonstrated

- [Harmony redirector / reverse-patch bridge](../how-to/recipes/harmony-redirector-reverse-patch.md)
  (family D) - six scoped Harmony patches (postfixes plus toolbar wiring) that augment
  vanilla tools without owning their logic
  (`repo/Anarchy/Patches/NetToolSystem_InitializeRaycast.cs#L31-L48`).
- [Reflect into private fields via Harmony Traverse](../how-to/recipes/harmony-traverse-private-fields.md)
  (family X) - `UpgradesManager.Install()` uses `Traverse.Create(...).Field<...>(...)` to
  pull `GameManager.instance`'s private `m_World` and the `PrefabSystem`'s private
  `m_Prefabs` list so it can clone/register Extended Road Upgrades prefabs; the raycast
  patch uses the same trick to grab `m_ToolRaycastSystem`
  (`repo/Anarchy/ExtendedRoadUpgrades/UpgradesManager.cs#L94-L115`,
  `repo/Anarchy/Patches/NetToolSystem_InitializeRaycast.cs#L38`).
- [External JSON side-car persistence](../how-to/recipes/json-sidecar-persistence.md)
  (family F) - Anarchy is the **XML variant**: per-error state is serialized to one
  `<ErrorType>.xml` file per error under `ModsData/.../ErrorChecks/` via `XmlSerializer`,
  with default-valued files deleted rather than stored
  (`repo/Anarchy/Systems/Common/AnarchyUISystem.cs#L671-L724`).
- [Network composition bitmasks](../how-to/recipes/network-composition-bitmasks.md)
  (family R) - the `NetworkComposition` and side-upgrade bitfields drive slope/elevation
  clamping and inline upgrades
  (`repo/Anarchy/Systems/NetworkAnarchy/NetworkDefinitionSystem.cs#L69-L72`,
  `repo/Anarchy/Systems/NetworkAnarchy/TempNetworkSystem.cs#L57-L63`).
- [Event-driven mod-state write (ModificationEnd / ToolOutputBarrier)](../how-to/recipes/event-driven-modificationend.md)
  (family Y) - flags are set at `Modification5` and restored at `ModificationEnd`, and
  entity tags are applied before vanilla cleanup runs
  (`repo/Anarchy/AnarchyMod.cs#L132-L136`,
  `repo/Anarchy/Systems/ErrorChecks/EnableToolErrorsSystem.cs#L80-L95`).
- [Localization multi-locale registration](../how-to/recipes/localization-helper.md)
  (family O) - non-English locales load from embedded `Anarchy.l10n.<localeID>.json`
  resources via `GetSupportedLocales()`; the hard I18n Everywhere dependency was dropped
  in v1.7.10 (`repo/Anarchy/AnarchyMod.cs#L170-L214`).
- [Tool drag-select + Highlighted components](../how-to/recipes/tool-drag-select.md)
  (family V) - the Anarchy Components Tool retargets `ToolRaycastSystem` masks/flags by
  selection mode (Radius vs object) and tier bits for radius/object mass selection
  (`repo/Anarchy/Systems/AnarchyComponentsTool/AnarchyComponentsToolSystem.cs#L94-L115`).

Related but supporting (not the primary lesson here): prefab-field override (family A) via
the `PreventOverride`/`TransformRecord`/`HeightRangeRecord` tag components, and Burst
IJobChunk work (family S) inside the components tool's `ToolUpdate` jobs.

## Key decisions & tradeoffs

- **Flag-bits on shared prefabs vs. per-tool patches.** The pivotal choice. Editing
  `ToolErrorData.m_Flags` on shared vanilla prefabs means one system covers every tool,
  and future tools inherit anarchy with no code change - dramatically lowering the
  per-patch maintenance surface versus writing a Harmony patch for each placement
  validator (`repo/Anarchy/Systems/ErrorChecks/DisableToolErrorsSystem.cs#L94-L113`). The
  cost is that it mutates *shared, global* prefab state, which is why a paired
  `EnableToolErrorsSystem` must run every `ModificationEnd` to restore the bits and why
  the on/off state has to be applied and reverted rather than scoped to one placement.
- **ECS guard vs. Harmony for exotic assets.** For moveable bridges the mod uses a
  lightweight ECS query on the `SubObject` buffer rather than a Harmony patch, keeping
  vanilla authoritative for assets it does not understand
  (`repo/Anarchy/Systems/ClearanceViolation/ModifyNetCompositionDataSystem.cs#L78-L104`).
- **XML per-error files vs. one JSON blob.** Splitting each error's three-state value
  into its own XML file (and deleting default-valued files) keeps the on-disk footprint
  minimal and makes a reset to default a file-delete, at the cost of many small files
  (`repo/Anarchy/Systems/Common/AnarchyUISystem.cs#L671-L724`).
- **No I18n Everywhere hard dependency since v1.7.10.** Locales moved to embedded
  resources loaded at runtime, so the sole hard dependency is Unified Icon Library; I18n
  Everywhere is only a README-documented soft dependency
  (`repo/Anarchy/AnarchyMod.cs#L170-L214`).
- **Public bridge as a first-class extension point.** Exposing `AnarchyBridge` lets other
  mods opt their tools into anarchy and manage components without reflection into
  Anarchy's internals (`repo/Anarchy/Bridge/AnarchyBridge.cs#L17-L45`).
- **Keybind = Ctrl+A.** The toggle binds to `BindingKeyboard.A` with `ctrl: true`; reset
  elevation is Alt+R, elevation step Alt+E, and PageUp/PageDown for elevation, with
  optional `[SettingsUIBindingMimic]` bindings that piggyback the vanilla "Change
  Elevation" shortcut (`repo/Anarchy/Settings/AnarchyModSettings.cs#L256-L312`). There is
  no AZERTY/Ctrl+Q auto-remap - that earlier claim was a fabrication and is not in source.

## Pitfalls / upstream-watch

- **Traverse reflection is name-fragile.** `UpgradesManager` reaches for the private
  `m_World` and `m_Prefabs` field names; a Colossal rename silently breaks ERU prefab
  injection (`repo/Anarchy/ExtendedRoadUpgrades/UpgradesManager.cs#L94-L115`). GitHub #34
  ("invisible quays") is the observed failure mode. Re-verify after every game patch.
- **Default-error matrix drift.** New `Game.Tools.ErrorType` enums must be added to
  `DefaultErrorChecks` or players get silent no-ops; `NoPortAccess` /
  `NotEnoughClearance` were added for Bridges & Ports (both default `WithAnarchy`)
  (`repo/Anarchy/Systems/Common/AnarchyUISystem.cs#L66-L67`,
  `repo/Anarchy/Settings/LocaleEN.cs#L138-L139`).
- **Shared-prefab flags must be reverted.** Because the toggle mutates global prefab
  state, a failure to run `EnableToolErrorsSystem` would leave checks disabled game-wide;
  the `ModificationEnd` scheduling is load-bearing
  (`repo/Anarchy/AnarchyMod.cs#L133`).
- **Transit stops are out of scope by design.** `AnarchyPlopSystem` explicitly excludes
  `Game.Routes.BusStop`/`ShipStop`/`TakeoffLocation`/`TaxiStand`, so GitHub #4 (bus stops
  on invisible car paths) is unimplemented net-new work, not a shipped feature
  (`repo/Anarchy/Systems/OverridePrevention/AnarchyPlopSystem.cs#L95-L98`). `Needs
  Verification`: exact line range not re-read at a6311e8 in this pass; taken from dossier.
- **Anarchy-family hotkey conflicts.** Stacking Time & Weather Anarchy / Anarchy 24
  fights over Ctrl+A and the flaming Chirper; rebind `ToggleAnarchy`
  (`repo/Anarchy/Settings/AnarchyModSettings.cs#L256-L258`).
- **Mega-city performance.** The components tool runs Burst jobs at `ToolUpdate`; large
  brush radii / low `PropRefreshFrequency` can spike frame time. `Needs Verification`:
  the megacity telemetry (100 m @ 30-frame -> ~5 ms) is an internal estimate from
  `notes/telemetry-20251110-megacity.md`, not a live capture.

## Source pointers

- Dossier: `../../vice-and-order-research/mods/dossiers/anarchy/`
  (index / source / modding / guide + notes)
- Repo @ `a6311e898d20a775368668b234aaa32f06e3e1eb` (branch master, "Merge PR #48
  PatchForGameV1.5.9", fetched 2026-06-29)
- Storefront: mod 74604, version 39 / userModVersion 1.7.24; live API requiredVersion
  1.6.* (in-repo PublishConfiguration records GameVersion 1.5.*).
