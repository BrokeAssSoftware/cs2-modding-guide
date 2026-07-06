---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: Custom AssetDatabase registration + component-importer plugin registry"
recipe: custom-assetdatabase-registration
technique_family: "BC - Custom AssetDatabase registration + component-importer plugin registry"
diataxis: how-to
source_version: "~1.6.0f1 (extra-assets-importer@ed23afc; date-pinned, static source only)"
last_reverified: "2026-07-05"
status: source-verified
canonical_mods:
  - extra-assets-importer@ed23afc5c76442ccc4d1a2cc0817c9bc6f42d256
technique_applicability: [content]
Created: 2026-07-05
Updated: 2026-07-05
Owners:
  - codex
---

# Custom AssetDatabase registration + component-importer plugin registry

> Register your own relocatable, versioned `AssetDatabase` backed by a folder on disk,
> and dispatch data-driven ECS-component injection to a pluggable registry of importers
> keyed by `ComponentType.FullName`.

## Problem
You are loading content the game did not ship - decals, surfaces, net lanes, whole asset
packs - from files a user drops in a mod-data folder, and you need the game to treat that
folder as a first-class asset source (indexable, prunable, relocatable). You also want the
authored JSON beside each asset to drive **which ECS components** get attached to each
prefab, without hard-coding a giant switch. In other words you need two reusable pieces:
(1) a registered `AssetDatabase` whose backing store is a directory you control, and (2) a
plugin registry that turns `"Game.Prefabs.CurveProperties": { ... }` in a JSON file into a
real `CurveProperties` component on the prefab. This is the database-registration and
plugin-dispatch layer beneath [unofficial asset import](../content/unofficial-asset-import.md).

## Solution
Describe your database once with an `IAssetDatabaseDescriptor` - it names the database, its
asset factory, and (crucially) a `FileSystemDataSource` rooted at a path you compute at
runtime. Register the descriptor's singleton instance with `AssetDatabase.global.RegisterDatabase(...)`,
then `PopulateFromDataSource(...)` to index what is on disk. Stamp the on-disk index with a
`DataBaseVersion` integer so a schema change wipes and rebuilds cleanly, and keep the root
path in a saved field so you can `Directory.Move` the whole store to a new location. For the
component layer, keep a `Dictionary<Type, ComponentImporter>` of small plugins; each plugin
declares the `ComponentType` it handles and the `PrefabType` it is valid on. When you import
a prefab, look up each JSON component fragment by `ComponentType.FullName`, check prefab-type
compatibility, and hand the fragment to the matching plugin's `Process`.

## Steps & Code

### 1. Describe the database with `IAssetDatabaseDescriptor`

The descriptor is a small `readonly struct`. The `dataSourceProvider` is what makes the
database file-backed: a `FileSystemDataSource` rooted at a runtime-computed path. Note
`kRootPath` reads a *mutable* field (`ActualDataBasePath`) so the root can move (step 6).

```csharp
public readonly struct EAIAssetDataBaseDescriptor
    : IAssetDatabaseDescriptor<EAIAssetDataBaseDescriptor>, IEquatable<EAIAssetDataBaseDescriptor>
{
    public static string kRootPath => EAIDataBaseManager.eaiDataBase.ActualDataBasePath;
    public bool canWriteSettings => false;
    public string name => "EAI";
    public IAssetFactory assetFactory => DefaultAssetFactory.instance;
    public IDataSourceProvider dataSourceProvider => new FileSystemDataSource(name, kRootPath, assetFactory);
    public DlcId dlcId => DlcId.Virtual;
    // Equals/GetHashCode omitted - the descriptor is a value identity (all instances equal)
}
```
Source: `../../../vice-and-order-research/mods/dossiers/extra-assets-importer/repo/MOD/DataBase/EAIAssetDataBaseDescriptor.cs#L7-L20` (@ed23afc5c76442ccc4d1a2cc0817c9bc6f42d256)

### 2. Resolve the database singleton and version its on-disk index

The concrete database is `AssetDatabase<TDescriptor>.instance`. Alongside it, keep a plain
JSON index (`EAIDatabase`) whose `DataBaseVersion` you bump when the index schema changes -
a mismatch clears and rebuilds the store instead of loading stale entries.

```csharp
public const int DataBaseVersion = 3;
public static EAIDatabase eaiDataBase;
public static AssetDatabase<EAIAssetDataBaseDescriptor> EAIAssetDataBase
    => AssetDatabase<EAIAssetDataBaseDescriptor>.instance;
// ...in LoadDataBase():
if (eaiDataBase.DataBaseVersion != DataBaseVersion)
{
    EAI.Logger.Warn($"The database version is not the good one, expected {DataBaseVersion}, got {eaiDataBase.DataBaseVersion}. The database will be reseted.");
    eaiDataBase.ClearDatabase();
}
```
Source: `../../../vice-and-order-research/mods/dossiers/extra-assets-importer/repo/MOD/DataBase/EAIDataBaseManager.cs#L22-L52` (@ed23afc5c76442ccc4d1a2cc0817c9bc6f42d256)

### 3. Register the database globally, then populate from the data source

`AssetDatabase.global.RegisterDatabase(...)` returns a `Task`; only after it succeeds do you
`PopulateFromDataSource(...)` to walk the `FileSystemDataSource` and index the files. Both
are async - chain them and surface faults, do not fire-and-forget.

```csharp
Task task = AssetDatabase.global.RegisterDatabase(EAIAssetDataBase).ContinueWith((t) =>
{
    if (t.IsFaulted)
    {
        EAI.Logger.Error($"Failed to register the Extra Assets Importer Asset Database : {t.Exception}");
        // ...surface a failed notification...
    }
    else
    {
        Task task2 = EAIAssetDataBase.PopulateFromDataSource(false, cancellationToken, taskProgress);
        task2.Wait();
        // ...surface complete / faulted notification...
    }
});
```
Source: `../../../vice-and-order-research/mods/dossiers/extra-assets-importer/repo/MOD/DataBase/EAIDataBaseManager.cs#L116-L151` (@ed23afc5c76442ccc4d1a2cc0817c9bc6f42d256)

### 4. Register component-importer plugins into a keyed registry

Each plugin is a `ComponentImporter` subclass; the registry is a `Dictionary<Type, ComponentImporter>`.
`AddComponentImporter<T>` is idempotent (returns `false` on a duplicate type).

```csharp
private static readonly Dictionary<Type, ComponentImporter> s_ComponentImporters = new();

public static bool AddComponentImporter<T>() where T : ComponentImporter, new()
{
    if (s_ComponentImporters.ContainsKey(typeof(T))) return false;
    T importer = new T();
    s_ComponentImporters.Add(typeof(T), importer);
    return true;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/extra-assets-importer/repo/MOD/AssetImporter/AssetsImporterManager.cs#L32-L69` (@ed23afc5c76442ccc4d1a2cc0817c9bc6f42d256)

Register the whole roster once at mod init:

```csharp
AssetsImporterManager.AddComponentImporter<UIObjectComponent>();
AssetsImporterManager.AddComponentImporter<ObsoleteIdentifiersComponent>();
AssetsImporterManager.AddComponentImporter<UtilityLaneComponent>();
AssetsImporterManager.AddComponentImporter<CurvePropertiesComponent>();
AssetsImporterManager.AddComponentImporter<RenderedAreaComponent>();
AssetsImporterManager.AddComponentImporter<EnclosedAreaComponent>();
```
Source: `../../../vice-and-order-research/mods/dossiers/extra-assets-importer/repo/EAI.cs#L138-L143` (@ed23afc5c76442ccc4d1a2cc0817c9bc6f42d256)

### 5. Define a plugin: declare its component + prefab types and a `Process`

A plugin binds one `ComponentType` to one `PrefabType` and deserializes its JSON fragment
into a real component. `AddOrGetComponent` is idempotent, so re-processing does not duplicate.

```csharp
public class CurvePropertiesComponent : ComponentImporter
{
    public override Type ComponentType => typeof(CurveProperties);
    public override Type PrefabType => typeof(NetLanePrefab);
    public override ComponentJson GetDefaultJson() => new CurvePropertiesJson();

    public override void Process(PrefabImportData data, Variant componentJson, PrefabBase prefab)
    {
        CurvePropertiesJson curvePropertiesJson = componentJson.Make<CurvePropertiesJson>();
        if (curvePropertiesJson is null) { /* ...log + return... */ return; }
        CurveProperties curveProperties = prefab.AddOrGetComponent<CurveProperties>();
        curveProperties.m_TilingCount = curvePropertiesJson.TilingCount;
        // ...copy remaining fields from JSON onto the component...
    }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/extra-assets-importer/repo/MOD/AssetImporter/Components/CurvePropertiesComponent.cs#L9-L37` (@ed23afc5c76442ccc4d1a2cc0817c9bc6f42d256)

The base contract is four abstract members - `ComponentType`, `PrefabType`, `Process`,
`GetDefaultJson` (`ComponentImporter.cs#L8-L14`).

### 6. Dispatch each JSON component fragment to its matching plugin

`ProcessComponentImporters` pulls the `Components` object out of the prefab JSON, then for
each registered plugin looks for a fragment keyed by that plugin's `ComponentType.FullName`.
Before calling `Process` it verifies the plugin's `PrefabType` matches the actual prefab type
(or a derived type) and **warns + skips** on a mismatch rather than attaching a bad component.

```csharp
Variant componentsVariant = prefabJson.TryGet(nameof(PrefabJson.Components));
if (componentsVariant == null) { /* ...log + return... */ return; }

foreach (ComponentImporter importer in s_ComponentImporters.Values)
{
    if (componentsVariant.TryGetValue(importer.ComponentType.FullName, out Variant componentJson))
    {
        if (importer.PrefabType != prefabBase.GetType()
            && !FindAllDerivedTypes(importer.PrefabType).Contains(prefabBase.GetType()))
        {
            EAI.Logger.Warn($"The component importer {importer.ComponentType.FullName} is not compatible with the prefab type {prefabBase.GetType().FullName} for the asset {prefabBase.name}.");
            continue;
        }
        importer.Process(data, componentJson, prefabBase);
    }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/extra-assets-importer/repo/MOD/AssetImporter/AssetsImporterManager.cs#L114-L134` (@ed23afc5c76442ccc4d1a2cc0817c9bc6f42d256)

### 7. Relocate the store, and prune unvalidated prefabs

Because the descriptor's root is `ActualDataBasePath`, moving the store is: unregister,
dispose, `Directory.Move`, re-point the field, save. Relocation only fires when the saved
target differs from the current root.

```csharp
AssetDatabase.global.UnregisterDatabase(EAIAssetDataBase).Wait();
EAIAssetDataBase.Dispose();
try
{
    Directory.Move(eaiDataBase.ActualDataBasePath, newDirectory);
    eaiDataBase.ActualDataBasePath = newDirectory;
    SaveDataBase();
}
catch (Exception ex) { /* ...log, return false... */ }
```
Source: `../../../vice-and-order-research/mods/dossiers/extra-assets-importer/repo/MOD/DataBase/EAIDataBaseManager.cs#L244-L262` (@ed23afc5c76442ccc4d1a2cc0817c9bc6f42d256)

After an import pass, assets that were not re-validated this run are optionally pruned - the
prefab is removed from `PrefabSystem`, the `PrefabAsset` unloaded and deleted, and its folder
removed - gated behind the `DeleteNotLoadedAssets` setting so a partial import does not nuke
everything:

```csharp
if (!EAI.m_Setting.DeleteNotLoadedAssets)
{
    _ValidateAssetsDataBase.AddRange(AssetsDataBase); // keep everything
    AssetsDataBase.Clear();
}
else
{
    ClearNotLoadedAssetsFromFiles(importerSettings); // prune what wasn't re-validated
}
```
Source: `../../../vice-and-order-research/mods/dossiers/extra-assets-importer/repo/MOD/DataBase/EAIDataBaseManager.cs#L286-L296` (@ed23afc5c76442ccc4d1a2cc0817c9bc6f42d256)

## Pitfalls & gotchas

- **The descriptor root is read lazily - the field must be set before registration.**
  `kRootPath` dereferences `EAIDataBaseManager.eaiDataBase.ActualDataBasePath`
  (`EAIAssetDataBaseDescriptor.cs#L9`). If `eaiDataBase` is null or its path is unset when
  `RegisterDatabase` builds the `FileSystemDataSource`, the source points nowhere. Load/construct
  the index (`LoadDataBase`) before `RegisterEaiAssetDatabase` runs.

- **Register **before** populate, and honor the fault.** `PopulateFromDataSource` is only
  reached inside the `else` branch of the `RegisterDatabase` continuation
  (`EAIDataBaseManager.cs#L127-L129`). Populating an unregistered database, or ignoring a
  faulted register task, silently yields an empty database with no assets and no error at the
  call site.

- **Version bump is destructive by design.** A `DataBaseVersion` mismatch calls
  `ClearDatabase()`, which deletes the entire `ActualDataBasePath` directory
  (`EAIDataBaseManager.cs#L48-L52`, `#L432-L438`). That is the intended reset, but it means a
  down-grade or a hand-edited index wipes all imported assets on next load. Only bump when you
  truly want a rebuild.

- **Relocation refuses to overwrite and unregisters first.** `RelocateAssetDataBase` bails if
  the target directory already exists, and it `UnregisterDatabase(...).Wait()` +
  `Dispose()` before the `Directory.Move` (`EAIDataBaseManager.cs#L230-L245`). If you relocate
  while an import is mid-flight the database is torn down under it. Relocate only between passes.

- **Component dispatch is keyed by fully-qualified type name, matched to the live game type.**
  The JSON key must be the exact `ComponentType.FullName` (e.g. `Game.Prefabs.CurveProperties`),
  and the plugin's `ComponentType` must resolve to a real game type at runtime
  (`AssetsImporterManager.cs#L124`). A renamed or moved vanilla component across a game patch
  breaks the key silently - the fragment simply never matches a plugin and is skipped with no error.

- **Prefab-type mismatch warns and skips, it does not throw.** If a JSON fragment targets a
  component whose plugin `PrefabType` is incompatible with the actual prefab, the component is
  *not* attached; you get a `Warn` log and a `continue` (`AssetsImporterManager.cs#L126-L129`).
  Watch the log - a missing component on an imported asset is often this, not a data error.

- **Pruning removes prefabs from `PrefabSystem` and deletes folders on disk.** With
  `DeleteNotLoadedAssets` enabled, `ClearNotLoadedAssetsFromFiles` calls `RemovePrefab`,
  `Unload`, `DeleteAsset`, and `Directory.Delete(path, true)`
  (`EAIDataBaseManager.cs#L367-L387`). A run that failed to re-validate an asset (e.g. an
  importer threw) will delete it. Keep the setting off unless a full clean import is guaranteed.

- The exact in-game load ordering guarantees (that `RegisterDatabase` completes before the
  game enumerates global databases for a given screen) are **Needs Verification (in-game)** -
  the source shows the async chain but not the surrounding game lifecycle timing.

## Variations

- **Second database instance for asset packs.** The same manager builds an additional
  `EAIDatabase` from an `AssetPackDatabase.json` and targets `AssetDatabase.user` (rooted at
  `AssetDatabase.user.rootPath/ImportedData`) instead of the global custom store, reusing the
  identical load/validate machinery (`AssetsImporterManager.cs#L194-L233`,
  `EAIDataBaseManager.cs#L59-L90`). Use this pattern when you want a separate, user-scoped store.

- **Prefab-first plugin selection.** Instead of iterating all plugins per prefab,
  `GetComponentImportersForPrefab(Type)` returns only the plugins whose `PrefabType` matches
  (or is a base of) a given prefab type, via the same `FindAllDerivedTypes` compatibility check
  (`AssetsImporterManager.cs#L93-L104`). Handy when you drive the loop from the prefab side.

- **Wildcard plugins that apply to any prefab.** A plugin can set `PrefabType => typeof(PrefabBase)`
  so it matches every prefab type (e.g. the `UIObject` and `ObsoleteIdentifiers` importers,
  `AssetsImporterManager` roster in step 4). Use the most-derived `PrefabType` that is still
  correct - broad types weaken the compatibility guard.

## See also
- Related how-to: [unofficial asset import](../content/unofficial-asset-import.md) (the importer
  roster and asset-pipeline layer this database sits under).
- Related recipes: [main-thread dispatcher for async work](mainthread-dispatcher-async.md)
  (the register/populate chain in this recipe is async and touches `PrefabSystem`, which must
  happen on the main thread).
- Reference: [ECS components catalog](../../reference/ecs-components-catalog.md) (the component
  types a plugin's `ComponentType` can target).
- Case study demonstrating it: [extra detailing suite](../../case-studies/extra-detailing-suite.md).

## Sources
- Canonical mods (dossier + repo):
  - `extra-assets-importer` @ed23afc5c76442ccc4d1a2cc0817c9bc6f42d256 -
    `repo/MOD/DataBase/EAIAssetDataBaseDescriptor.cs`, `repo/MOD/DataBase/EAIDataBaseManager.cs`,
    `repo/MOD/AssetImporter/AssetsImporterManager.cs`,
    `repo/MOD/AssetImporter/Components/ComponentImporter.cs`,
    `repo/MOD/AssetImporter/Components/CurvePropertiesComponent.cs`, `repo/EAI.cs`
- Official/community references (link out, do not duplicate): https://cs2.paradoxwikis.com/Modding
