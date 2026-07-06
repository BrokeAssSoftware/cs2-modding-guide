---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: Prefab cloning / new-prefab synthesis"
recipe: prefab-clone-synthesis
technique_family: "AC - Prefab cloning / new-prefab synthesis"
diataxis: how-to
source_version: "~1.5.x (specialized-industrial-zones@8d9153c; date-pinned, static source only)"
last_reverified: "2026-07-05"
status: source-verified
canonical_mods:
  - specialized-industrial-zones@8d9153c3fda72e3b79a133bee6c83f9ba7df8b5a
  - anarchy@a6311e898d20a775368668b234aaa32f06e3e1eb
technique_applicability: [content, core]
Created: 2026-07-05
Updated: 2026-07-05
Owners:
  - codex
---

# Prefab cloning / new-prefab synthesis

> Register a brand-new prefab at runtime by cloning a vanilla template, mutating the
> clone's authoring components, and adding it to `PrefabSystem` - so the game treats
> it as a first-class buildable/zonable/upgradeable object it never shipped with.

## Problem
You need a prefab the game does not have: a new specialized zone type, a new road
upgrade mode, a new marker. Editing an existing prefab's fields (families A / J -
[prefab-field override](prefab-field-override.md)) is not enough, because those
recipes mutate the *shared* template every instance already uses; you cannot make a
*new*, separately-selectable object that way. You want an additional prefab that
coexists with the vanilla one, carries its own name/icon/UI entry, and survives
savegame round-trips.

## Solution
Take a loaded vanilla prefab as a template, produce a **copy** of it, give the copy a
unique name, mutate the copy's authoring components (`ZoneProperties`,
`BuildingProperties`, `UIObject`, `PlaceableNetData`, ...), then hand it to
`PrefabSystem.AddPrefab` so the game bakes it into ECS like any other prefab. Two
copy strategies exist in the wild: `PrefabBase.Clone(newName)` (deep managed clone -
Specialized Industrial Zones) and `ScriptableObject.CreateInstance<T>()` +
`AddComponentFrom` per component (manual rebuild - Anarchy's Extended Road Upgrades).
Both enumerate their template list by reflecting `PrefabSystem.m_Prefabs`. The load-
bearing detail is **save-safety**: attach an `ObsoleteIdentifiers` component listing
every legacy name the clone ever had, or a rename breaks existing saves.

## Steps & Code

### 1. Register the system to run BEFORE `PrefabSystem`

Your synthesis must happen before vanilla `PrefabSystem` bakes prefabs into entities,
so order the system explicitly:

```csharp
updateSystem.UpdateBefore<SpecializedZoningSystem, PrefabSystem>(SystemUpdatePhase.MainLoop);
```
Source: `../../../vice-and-order-research/mods/dossiers/specialized-industrial-zones/repo/src/SpecializedIndustryZones/Mod.cs#L27` (@8d9153c3fda72e3b79a133bee6c83f9ba7df8b5a)

### 2. Enumerate clone sources by reflecting `PrefabSystem.m_Prefabs`

There is no public "list every loaded prefab" API, so both canonical mods reflect the
private `m_Prefabs` list. SIZ snapshots it in `OnCreate` and indexes the zones it will
clone from:

```csharp
_allPrefabs = typeof(PrefabSystem)
    .GetField("m_Prefabs", System.Reflection.BindingFlags.NonPublic | System.Reflection.BindingFlags.Instance)
    ?.GetValue(_prefabSystem) as List<PrefabBase>
    ?? throw new Exception("Could not access m_Prefabs field in PrefabSystem, likely broken by game update");

_initialZones = [.._allPrefabs.OfType<ZonePrefab>()];
```
Source: `../../../vice-and-order-research/mods/dossiers/specialized-industrial-zones/repo/src/SpecializedIndustryZones/SpecializedZoningSystem.cs#L48-L53` (@8d9153c3fda72e3b79a133bee6c83f9ba7df8b5a)

### 3. Clone the template with a NEW unique name

`PrefabBase.Clone(newName)` produces a deep managed copy carrying the same component
set. The name **must** differ from the source, or the clone collides with the vanilla
prefab (SIZ explicitly warns and drops a clone that failed to rename - see Pitfalls):

```csharp
private static T Clone<T>(string specID, T source, SpecializedZoneSpec spec)
    where T : PrefabBase
{
    var newName = GetUpdatedName(specID, source.name, spec);
    var clone = (T)source.Clone(newName);
    return clone;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/specialized-industrial-zones/repo/src/SpecializedIndustryZones/SpecializedZoningSystem.cs#L414-L420` (@8d9153c3fda72e3b79a133bee6c83f9ba7df8b5a)

### 4. Mutate the clone's authoring components

Fetch components off the *clone* and edit them; you are shaping a new object, not the
template. SIZ narrows the zone's allowed resources and swaps its edge colour and icon:

```csharp
var zone = Clone(specID, sourceZone, spec);
zone.m_Edge = spec.Color;
// ... (unique-name guard omitted) ...
var zoneProps = zone.GetComponent<ZoneProperties>();
if (combinedFilter.ManufacturedResources != null)
    zoneProps.m_AllowedManufactured = [.. combinedFilter.ManufacturedResources.Intersect(zoneProps.m_AllowedManufactured)];
if (combinedFilter.StoredResources != null)
    zoneProps.m_AllowedStored = [.. combinedFilter.StoredResources.Intersect(zoneProps.m_AllowedStored)];
if (spec.IconUri != null)
    zone.GetComponent<UIObject>().m_Icon = spec.IconUri;
```
Source: `../../../vice-and-order-research/mods/dossiers/specialized-industrial-zones/repo/src/SpecializedIndustryZones/SpecializedZoningSystem.cs#L285-L306` (@8d9153c3fda72e3b79a133bee6c83f9ba7df8b5a)

### 5. Attach `ObsoleteIdentifiers` so old saves still resolve the clone

This is the step that separates a robust clone from one that corrupts saves. When your
naming scheme changes across mod versions, savegames reference the *old* clone name.
`AddOrGetComponent<ObsoleteIdentifiers>()` and append every legacy name so the loader
maps the old identity onto the new prefab:

```csharp
var obsoleteIdentifiers = newPrefab.AddOrGetComponent<ObsoleteIdentifiers>();
// legacy name from the 0.2.1 rename, mapped onto this clone's type
var legacyName = sourcePrefab.name.Replace("Industrial", $"Specialized Industrial {spec.Name}");
var legacyPrefabInfo = new PrefabIdentifierInfo { m_Name = legacyName, m_Type = newPrefab.GetType().Name };
if (obsoleteIdentifiers.m_PrefabIdentifiers == null)
    obsoleteIdentifiers.m_PrefabIdentifiers = [legacyPrefabInfo];
else
    obsoleteIdentifiers.m_PrefabIdentifiers = [.. obsoleteIdentifiers.m_PrefabIdentifiers, legacyPrefabInfo, /* ...remapped existing ids... */];
```
Source: `../../../vice-and-order-research/mods/dossiers/specialized-industrial-zones/repo/src/SpecializedIndustryZones/SpecializedZoningSystem.cs#L452-L491` (@8d9153c3fda72e3b79a133bee6c83f9ba7df8b5a)

### 6. Register the clone with `AddPrefab` (or `UpdatePrefab` on re-provision)

`AddPrefab` inserts a *new* prefab; `UpdatePrefab` replaces one you already added.
SIZ tracks which specialised IDs it has provisioned and branches accordingly:

```csharp
if (!_provisionedZones.TryGetValue(specID, out var zone))
{
    zone = CreateZonePrefab(specID, spec, baseZone, combinedFilter);
    success = _prefabSystem.AddPrefab(zone);          // brand-new prefab
}
else
{
    var replacementZone = CreateZonePrefab(specID, spec, baseZone, combinedFilter);
    _prefabSystem.UpdatePrefab(replacementZone);      // already registered -> replace
    success = true;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/specialized-industrial-zones/repo/src/SpecializedIndustryZones/SpecializedZoningSystem.cs#L124-L135` (@8d9153c3fda72e3b79a133bee6c83f9ba7df8b5a)

### 7. Deferred ECS data: add `IComponentData` after the prefab is baked

`AddPrefab` registers the managed prefab, but some behaviour lives in *baked* ECS
`IComponentData` that does not exist until the game processes the prefab. Anarchy's
Extended Road Upgrades therefore splits synthesis into two phases: `Install` clones +
`AddPrefab`, then an `onGameLoadingComplete` handler reads and rewrites the baked
`PlaceableNetData` on each clone:

```csharp
clonedGrassUpgradePrefabData.m_SetUpgradeFlags = upgradeMode.m_SetUpgradeFlags;
clonedGrassUpgradePrefabData.m_UnsetUpgradeFlags = upgradeMode.m_UnsetUpgradeFlags;
if (upgradeMode.IsUnderground)
    clonedGrassUpgradePrefabData.m_PlacementFlags |= Game.Net.PlacementFlags.UndergroundUpgrade;
prefabSystem.AddComponentData(clonedGrassUpgradePrefab, clonedGrassUpgradePrefabData);
```
Source: `../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/ExtendedRoadUpgrades/UpgradesManager.cs#L317-L329` (@a6311e898d20a775368668b234aaa32f06e3e1eb)

## Pitfalls & gotchas

- **A clone that failed to rename is silently dropped.** `Clone` derives the new name
  from a string-replace (`GetUpdatedName`); if the source name matches none of the
  expected substrings the "new" name equals the old one. SIZ detects `zone.name ==
  sourceZone.name`, logs a warning, and returns `null` rather than register a
  colliding prefab
  (`../../../vice-and-order-research/mods/dossiers/specialized-industrial-zones/repo/src/SpecializedIndustryZones/SpecializedZoningSystem.cs#L288-L291`,
  @8d9153c3fda72e3b79a133bee6c83f9ba7df8b5a). Design your naming so a rename can never
  no-op.

- **Skip `ObsoleteIdentifiers` and you break existing saves on any rename.** The clone
  is persisted by name. If v2 of your mod renames the clone, a city saved under v1
  references a prefab that no longer exists unless the new prefab lists the old name in
  `ObsoleteIdentifiers.m_PrefabIdentifiers` (step 5). This is the whole reason the mod
  carries a "// This is required due to the rename in 0.2.1" block
  (`.../SpecializedZoningSystem.cs#L456`).

- **Filter sets are INTERSECT-only - you can narrow a vanilla prefab, never widen it.**
  Every mutation is `spec.Set.Intersect(zoneProps.m_Allowed...)`
  (`.../SpecializedZoningSystem.cs#L295-L300`). A clone can only allow a *subset* of
  what its template allowed; it can never permit a resource the vanilla template did
  not. If you need a superset you must assign a fresh list, not intersect.

- **Null filter set: guarded in one place, NREs in another (asymmetry).** In the zone
  path each intersect is wrapped in `if (combinedFilter.X != null)`
  (`.../SpecializedZoningSystem.cs#L294-L300`), so a null set is skipped. But the
  building-eligibility check calls `...Intersect(combinedFilter.ManufacturedResources)`
  with **no** null guard
  (`.../SpecializedZoningSystem.cs#L331-L333`), so a null filter set there throws
  `ArgumentNullException`. The two code paths disagree - always populate all three sets
  or replicate the null guard.

- **Register before `PrefabSystem` bakes, or the clone misses the bake pass.** SIZ uses
  `UpdateBefore<..., PrefabSystem>` (step 1); Anarchy hooks earlier via a Harmony patch
  and does baked-data work in a second `onGameLoadingComplete` phase. Adding a prefab
  after baking means its ECS `IComponentData` never materialises for that load.

- **Reflection into `m_Prefabs` is fragile across game updates.** Both mods read the
  private field by name and treat a null result as a hard failure ("likely broken by
  game update", `.../SpecializedZoningSystem.cs#L51`). Expect to re-verify this on each
  CS2 patch. `Needs Verification (in-game)` for any specific post-1.5.x version.

- **Idempotency guards are manual.** Anarchy uses static `installed` / `postInstalled`
  booleans to stop its two phases running twice, and also `TryGetPrefab(new
  PrefabID("FencePrefab", ...))` to bail if the clone already exists
  (`../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/ExtendedRoadUpgrades/UpgradesManager.cs#L85-L90,#L146,#L246`,
  @a6311e898d20a775368668b234aaa32f06e3e1eb). Without such guards, re-entry double-adds
  prefabs.

## Variations

- **Manual rebuild instead of `Clone` (Anarchy).** Rather than `PrefabBase.Clone`,
  Anarchy instantiates a fresh typed prefab and copies each component across, giving it
  full control over which references carry over:

  ```csharp
  FencePrefab fencePrefab = ScriptableObject.CreateInstance<FencePrefab>();
  fencePrefab.name = upgradeMode.Id;
  fencePrefab.prefab = fencePrefab;
  foreach (ComponentBase componentBase in grassUpgradePrefab.components)
  {
      componentBase.prefab = fencePrefab;       // reparent to the clone
      fencePrefab.AddComponentFrom(componentBase);
  }
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/ExtendedRoadUpgrades/UpgradesManager.cs#L153-L163` (@a6311e898d20a775368668b234aaa32f06e3e1eb)

- **Swap the icon by replacing `UIObject`, not editing the shared one.** Anarchy removes
  the copied `UIObject` and adds a *fresh* instance so it never mutates the template's
  UI entry, then points it at a mod-shipped SVG:

  ```csharp
  fencePrefab.Remove<UIObject>();
  var upgradePrefabUIObject = ScriptableObject.CreateInstance<UIObject>();
  upgradePrefabUIObject.m_Icon = $"{COUIBaseLocation}{upgradeMode.ObsoleteId}.svg";
  upgradePrefabUIObject.name = grassUpgradePrefabUIObject.name.Replace("Grass", upgradeMode.Id);
  fencePrefab.AddComponentFrom(upgradePrefabUIObject);
  if (!prefabSystem.AddPrefab(fencePrefab)) { /* error, exit */ }
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/ExtendedRoadUpgrades/UpgradesManager.cs#L174-L191` (@a6311e898d20a775368668b234aaa32f06e3e1eb)

- **Clone the template's dependent prefabs too.** A zone is useless without buildings
  that spawn in it. SIZ, after cloning the zone, iterates the spawnable buildings tied
  to the source zone, clones each, repoints `SpawnableBuilding.m_ZoneType` at the new
  zone, and registers them alongside
  (`../../../vice-and-order-research/mods/dossiers/specialized-industrial-zones/repo/src/SpecializedIndustryZones/SpecializedZoningSystem.cs#L314-L385`,
  @8d9153c3fda72e3b79a133bee6c83f9ba7df8b5a). Cloning one prefab often means cloning its
  whole dependency cluster.

## See also
- Related recipes: [prefab-field override](prefab-field-override.md) (families A / J -
  edit an existing prefab's fields, do not create a new one),
  [one-shot prefab multiplier](one-shot-prefab-multiplier.md).
- Reference: [ECS components catalog](../../reference/ecs-components-catalog.md)
  (`ZoneProperties`, `BuildingProperties`, `PlaceableNetData`, `ObsoleteIdentifiers`,
  `UIObject`).
- Case studies demonstrating it: [anarchy](../../case-studies/anarchy.md)
  (Extended Road Upgrades clone path); specialized-industrial-zones case study pending.

## Sources
- Canonical mods (dossier + repo):
  - `specialized-industrial-zones` @8d9153c3fda72e3b79a133bee6c83f9ba7df8b5a - `repo/src/SpecializedIndustryZones/SpecializedZoningSystem.cs`, `repo/src/SpecializedIndustryZones/Mod.cs`
  - `anarchy` @a6311e898d20a775368668b234aaa32f06e3e1eb - `repo/Anarchy/ExtendedRoadUpgrades/UpgradesManager.cs`
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding
