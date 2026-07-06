---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: Prefab-subnet traversal + host->upgrade propagation"
recipe: prefab-subnet-traversal
technique_family: "BG - Prefab-subnet traversal + host->upgrade propagation"
diataxis: how-to
source_version: "1.6.0f1 (smooth-left-hand-traffic@37b2850; date-pinned, static source only)"
last_reverified: "2026-07-05"
status: source-verified
canonical_mods:
  - smooth-left-hand-traffic@37b2850b10ced1cd17457c25778409e57e00a03e
technique_applicability: [infrastructure]
Created: 2026-07-05
Updated: 2026-07-05
Owners:
  - codex
---

# Prefab-subnet traversal + host->upgrade propagation

> Classify a building prefab by whether it embeds a drivable internal road, then
> propagate one prefab edit across the whole upgrade family (host + its extensions)
> with a cycle-safe visited-set recursion.

## Problem
You have a prefab edit you want to apply *selectively* - only to building prefabs
that actually contain a drivable sub-net (an internal road, tram track, or bike
path) - and once you flip a value on a host building you need the *same* value on
every service extension that stacks onto it, without editing the same prefab twice or
looping forever on a cyclic upgrade graph. Named-lookup (family A) does not help here:
you do not know the names up front and you need to *discover* which prefabs qualify.

## Solution
Two reusable pieces, both running at `SystemUpdatePhase.PrefabUpdate`:

1. **Subnet traversal to classify.** Walk each prefab's `ObjectSubNets.m_SubNets` to
   its `NetGeometryPrefab`, down `m_Sections` -> `m_Section.m_Pieces` ->
   `NetPieceLanes.m_Lanes`, and test each lane for an *active* `CarLane`
   (`RoadTypes.Car`/`Bicycle`) or `TrackLane` (`TrackTypes.Tram`). If any lane
   matches, the prefab has a drivable internal road and is eligible.
2. **Host->upgrade propagation.** Build a `hostName -> List<upgradePrefab>` map from
   each extension's `ServiceUpgrade.m_Buildings`, then recurse from the host into its
   upgrades, guarding every step with a `HashSet<string>` of visited names so the
   walk is cycle-safe and each prefab is edited exactly once.

The classifier only *reads* prefabs; the actual field write reuses the
[prefab-field override](prefab-field-override.md) commit path
(`ObjectSubNets.m_InvertWhen` + `PrefabSystem.UpdatePrefab`).

## Steps & Code

### 1. Query building prefabs and hand them to the scanner

Build an `EntityQuery` over `PrefabData` with either building component, materialize
the entities, and scan. The scan returns both the eligible prefabs and the
host->upgrade map in one pass:

```csharp
allAssets = SystemAPI.QueryBuilder()
    .WithAll<PrefabData>()
    .WithAny<BuildingData, BuildingExtensionData>()
    .Build();
using var allAssetEntities = allAssets.ToEntityArray(Allocator.Temp);
var scanResult = prefabScanner.Scan(allAssetEntities);
```
Source: `../../../vice-and-order-research/mods/dossiers/smooth-left-hand-traffic/repo/SmoothLHT/Systems/InvertPrefabLHTSystem.cs#L123-L131` (@37b2850b10ced1cd17457c25778409e57e00a03e)

This system is scheduled at `SystemUpdatePhase.PrefabUpdate`, the phase where prefab
component data is settled:

```csharp
updateSystem.UpdateAt<InvertPrefabLHTSystem>(SystemUpdatePhase.PrefabUpdate);
```
Source: `../../../vice-and-order-research/mods/dossiers/smooth-left-hand-traffic/repo/SmoothLHT/Mod.cs#L26` (@37b2850b10ced1cd17457c25778409e57e00a03e)

### 2. Gate the prefab: right type, has `ObjectSubNets`, has a drivable lane

Each candidate is filtered through ordered guards. Note the two prefab kinds that
carry sub-nets (`BuildingPrefab`, `BuildingExtensionPrefab`) and the managed-component
fetch `prefab.TryGet(out ObjectSubNets ...)`:

```csharp
if (prefab is not BuildingPrefab and not BuildingExtensionPrefab)
{ skipReason = PrefabSkipReason.UnsupportedPrefabType; return false; }

if (!prefab.TryGet(out ObjectSubNets subNets) || subNets is null)
{ skipReason = PrefabSkipReason.MissingObjectSubNets; return false; }

if (!HasSupportedTransportKinds(subNets))
{ skipReason = PrefabSkipReason.UnsupportedTransportKinds; return false; }
```
Source: `../../../vice-and-order-research/mods/dossiers/smooth-left-hand-traffic/repo/SmoothLHT/Services/InvertiblePrefabScanner.cs#L62-L84` (@37b2850b10ced1cd17457c25778409e57e00a03e)

### 3. Walk `m_SubNets` -> `NetGeometryPrefab`

The first hop: each sub-net entry references a net prefab; keep only the
`NetGeometryPrefab` ones and ask what transport kinds their geometry supports:

```csharp
foreach (var subNet in subNets.m_SubNets)
{
    if (subNet.m_NetPrefab is NetGeometryPrefab geometryPrefab)
    {
        var transportKinds = GetSupportedTransportKinds(geometryPrefab);
        if (transportKinds != SupportedTransportKinds.None)
            return true;
    }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/smooth-left-hand-traffic/repo/SmoothLHT/Services/InvertiblePrefabScanner.cs#L109-L119` (@37b2850b10ced1cd17457c25778409e57e00a03e)

### 4. Descend `m_Sections` -> `NetPieceLanes.m_Lanes` and test each lane

The deep hop: geometry -> sections -> the section's pieces -> each piece's
`NetPieceLanes` -> its lanes. A lane qualifies only if it is an **active** `CarLane`
carrying `RoadTypes.Car`/`Bicycle`, or an **active** `TrackLane` of `TrackTypes.Tram`:

```csharp
if (pieceInfo.m_Piece is null ||
    !pieceInfo.m_Piece.TryGet(out NetPieceLanes pieceLanes) || pieceLanes.m_Lanes is null)
    continue;

foreach (var laneInfo in pieceLanes.m_Lanes)
{
    if (laneInfo.m_Lane.TryGet(out CarLane carLane) && carLane.active)
    {
        if ((carLane.m_RoadType & Game.Net.RoadTypes.Car) != 0)
            transportKinds |= SupportedTransportKinds.Car;
        if ((carLane.m_RoadType & Game.Net.RoadTypes.Bicycle) != 0)
            transportKinds |= SupportedTransportKinds.Bicycle;
    }

    if (laneInfo.m_Lane.TryGet(out TrackLane trackLane) && trackLane.active &&
        trackLane.m_TrackType == Game.Net.TrackTypes.Tram)
        transportKinds |= SupportedTransportKinds.Tram;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/smooth-left-hand-traffic/repo/SmoothLHT/Services/InvertiblePrefabScanner.cs#L142-L173` (@37b2850b10ced1cd17457c25778409e57e00a03e)

### 5. Build the host->upgrade map from `ServiceUpgrade.m_Buildings`

For extension prefabs only, read `ServiceUpgrade.m_Buildings` - the host buildings this
extension attaches to - and index the extension under each host's name. This inverts
the relationship so a host lookup yields all its upgrades:

```csharp
if (prefab is not BuildingExtensionPrefab || !prefab.TryGet(out ServiceUpgrade serviceUpgrade))
    return 0;
if (serviceUpgrade.m_Buildings is null) return 0;

foreach (var building in serviceUpgrade.m_Buildings)
{
    if (!buildingUpgrades.TryGetValue(building.name, out var upgrades))
    {
        upgrades = new List<PrefabBase>();
        buildingUpgrades[building.name] = upgrades;
    }
    if (!upgrades.Contains(prefab)) { upgrades.Add(prefab); mappedCount++; }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/smooth-left-hand-traffic/repo/SmoothLHT/Services/InvertiblePrefabScanner.cs#L183-L207` (@37b2850b10ced1cd17457c25778409e57e00a03e)

### 6. Propagate the edit with a cycle-safe visited-set recursion

Start from the host, edit it, record its name in the visited set, then recurse into
each upgrade. The `visitedPrefabNames.Add(prefab.name)` guard returns `false` on a
name already seen, which both prevents double edits and terminates any cycle in the
upgrade graph. The seed set is a fresh `HashSet<string>()` per top-level call:

```csharp
if (prefab is null || !visitedPrefabNames.Add(prefab.name))
    return;

result.UpdatedPrefabCount += UpdatePrefabInvertMode(prefab, invertMode);
preferenceStore.SetInvertMode(prefab.name, invertMode);

if (!buildingUpgrades.TryGetValue(prefab.name, out var upgrades))
    return;

foreach (var upgrade in upgrades)
    ApplyInvertModeRecursively(upgrade, invertMode, buildingUpgrades, preferenceStore, visitedPrefabNames, result);
```
Source: `../../../vice-and-order-research/mods/dossiers/smooth-left-hand-traffic/repo/SmoothLHT/Services/PrefabInvertService.cs#L48-L65` (@37b2850b10ced1cd17457c25778409e57e00a03e)

The actual write is the family-A commit path, idempotent behind a `!= invertMode`
guard so an already-correct prefab is skipped and not re-counted:

```csharp
if (prefab.TryGet(out ObjectSubNets subNets) && subNets.m_InvertWhen != invertMode)
{
    subNets.m_InvertWhen = invertMode;
    prefabSystem.UpdatePrefab(prefab);
    return 1;
}
return 0;
```
Source: `../../../vice-and-order-research/mods/dossiers/smooth-left-hand-traffic/repo/SmoothLHT/Services/PrefabInvertService.cs#L68-L78` (@37b2850b10ced1cd17457c25778409e57e00a03e)

## Pitfalls & gotchas

- **Null at every hop.** The traversal chain is a minefield of nullable managed
  references: `subNets.m_SubNets`, `geometryPrefab.m_Sections`,
  `sectionInfo.m_Section.m_Pieces`, `pieceInfo.m_Piece`, `pieceLanes.m_Lanes`, and
  `laneInfo.m_Lane` are each null-checked before use in the canonical source
  (`InvertiblePrefabScanner.cs#L104-L152`). Skip any one guard and a stock prefab with
  a partially-authored sub-net throws mid-scan. Treat every `.m_*` collection as
  possibly null.

- **`active` is not optional.** A lane that exists but is inactive must not count -
  the source gates on `carLane.active` / `trackLane.active`
  (`InvertiblePrefabScanner.cs#L154-L169`). Dropping the `active` check would classify
  decorative or disabled lanes as drivable and pull in prefabs you did not intend to
  edit.

- **The upgrade map is keyed host-name -> upgrades, not the reverse.** `ServiceUpgrade`
  lives on the **extension** prefab and lists its **host** buildings
  (`InvertiblePrefabScanner.cs#L194`). The scanner inverts that into
  `hostName -> [extensions]` so a single host toggle fans out to its extensions. If
  you index the map the other way you will edit hosts from extensions and miss the
  propagation direction the mod actually uses.

- **Cycle-safety hinges on one shared visited set.** The `HashSet<string>` is created
  once per top-level `InvertPrefabAndUpgrades` call and threaded through every
  recursive frame (`PrefabInvertService.cs#L34, #L64`). If you allocate a new set per
  level, a host<->upgrade cycle (or two extensions sharing a host) loops forever or
  re-edits. Keys are prefab **names** (strings), not entities.

- **Name-keyed dedup can collide.** Because the visited set and the upgrade map key on
  `prefab.name`, two distinct prefabs that somehow share a name would be treated as one.
  Not observed in the source, but worth knowing if you reuse this over a prefab set you
  do not control.

- **In-game effect of the propagated edit is not proven from this code.** That the
  invert flip actually reorders traffic correctly on every upgrade instance is a
  runtime behaviour - `Needs Verification (in-game)`. The source only shows the
  classification and write, not the simulation outcome.

## Variations

- **Classify only, no propagation.** If you just need the yes/no "has a drivable
  internal road" answer, stop after step 4: `HasSupportedTransportKinds` returns as
  soon as the first qualifying lane is found (`InvertiblePrefabScanner.cs#L114-L116`),
  so the scan short-circuits without building the upgrade map.

- **Broaden or narrow the lane predicate.** The `SupportedTransportKinds` flags enum
  (`Car`/`Bicycle`/`Tram`, `InvertiblePrefabScanner.cs#L234-L241`) is the single choke
  point for "what counts as drivable". Add pedestrian, add other `TrackTypes`, or
  require a specific combination by editing only `GetSupportedTransportKinds`.

- **Different commit path.** This recipe commits via `PrefabSystem.UpdatePrefab`
  (managed-component path for `ObjectSubNets`). For a plain ECS struct field, swap the
  write for the `TryGetComponentData` -> `AddComponentData` sequence from
  [prefab-field override](prefab-field-override.md); the traversal and recursion above
  are independent of how you finally write.

## See also
- Related recipes: [prefab-field override](prefab-field-override.md) (family A - the
  read-mutate-commit path this recipe reuses for the actual write),
  [prefab-clone synthesis](prefab-clone-synthesis.md).
- Reference: [ECS components catalog](../../reference/ecs-components-catalog.md)
  (`ObjectSubNets`, `NetGeometryPrefab`, `NetPieceLanes`, `CarLane`, `TrackLane`,
  `ServiceUpgrade`).
- Case study demonstrating it:
  [smooth-left-hand-traffic](../../case-studies/smooth-left-hand-traffic.md).

## Sources
- Canonical mods (dossier + repo):
  - `smooth-left-hand-traffic` @37b2850b10ced1cd17457c25778409e57e00a03e -
    `repo/SmoothLHT/Services/InvertiblePrefabScanner.cs`,
    `repo/SmoothLHT/Services/PrefabInvertService.cs`,
    `repo/SmoothLHT/Systems/InvertPrefabLHTSystem.cs`,
    `repo/SmoothLHT/Mod.cs`
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding
