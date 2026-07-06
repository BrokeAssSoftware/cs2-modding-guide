---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Import Custom Assets at Runtime (Decals, Surfaces, Net Lanes)"
Summary: How to load custom decals, surfaces, and net-lane decals into Cities: Skylines II at runtime without the asset editor - JSON + textures compiled into prefabs and registered in a custom AssetDatabase, using Extra Assets Importer's importer roster, database, and COUI icon host as the worked example.
diataxis: how-to
source_version: "~1.5.10f1 (extra-assets-importer@ed23afc; date-pinned, static source only)"
last_reverified: "2026-07-04"
canonical_mods:
  - extra-assets-importer@ed23afc5c76442ccc4d1a2cc0817c9bc6f42d256
technique_applicability: [content, tooling]
status: source-verified
Created: 2026-07-03
Updated: 2026-07-03
Owners:
  - codex
References:
  - Label: Manage and audit asset packs
    Path: ./asset-pack-management.md
  - Label: Author a PBR asset (model to import)
    Path: ./pbr-asset-workflow.md
  - Label: "Technique H - COUI icon/host registration"
    Path: ../../technique-index.md
  - Label: "Reference: ExtraLib (forward-ref)"
    Path: ../../reference/shared-libraries/extralib.md
---

# Import Custom Assets at Runtime (Decals, Surfaces, Net Lanes)

Until the official CS2 asset editor covers every content type, you can still ship
high-quality **decals, surfaces, and decal-based net lanes** by loading texture packs at
runtime: a folder of textures plus a JSON material definition is compiled into a real
prefab and registered so it behaves like vanilla content in the build menus.

This how-to uses **Extra Assets Importer (EAI)** as the worked, source-cited example
(Paradox Mods id `80529`, source `github.com/AlphaGaming7780/ExtraAssetsImporter`). Every
mechanism below is cited to EAI at pinned commit
`ed23afc5c76442ccc4d1a2cc0817c9bc6f42d256` (tag `v1.7.4`). EAI is the pattern to copy
whether you are building your own importer or shipping a pack *against* EAI.

> **Dependency note.** EAI does its runtime work through **ExtraLib** (Paradox Mods id
> `75724`): the prefab system, the notification UI, localization ingestion, the COUI icon
> host, and the deferred boot hook all come from ExtraLib. ExtraLib is a hard build +
> runtime dependency - it must be installed and load *before* EAI, or EAI throws at load
> (EAI GitHub issue #28). Keep ExtraLib earlier in the playset for every pack.

---

## The shape of the pipeline

At load, EAI registers a fixed roster of **importers** (one per asset kind) plus
**component importers** (that attach extra data to a prefab), points them at asset
folders, and defers the heavy work until ExtraLib says the game is ready. Each importer
walks `category -> asset` folders, hashes the source, builds a prefab from the JSON +
textures, writes it into a **custom AssetDatabase**, and registers it with the prefab
system so it shows up in the UI.

```csharp
// EAI.OnLoad: register the importer roster + component importers, then the asset root.
AssetsImporterManager.AddImporter<LocalizationImporter>();
AssetsImporterManager.AddImporter<AssetPackImporter>();
AssetsImporterManager.AddImporter<DecalsImporterNew>();
AssetsImporterManager.AddImporter<NetLanesDecalImporterNew>();
AssetsImporterManager.AddImporter<SurfacesImporterNew>();

AssetsImporterManager.AddComponentImporter<UIObjectComponent>();
AssetsImporterManager.AddComponentImporter<ObsoleteIdentifiersComponent>();
AssetsImporterManager.AddComponentImporter<UtilityLaneComponent>();
// ... CurveProperties, RenderedArea, EnclosedArea ...

if (m_Setting.UseNewImporters) AssetsImporterManager.AddAssetFolder(pathModsData);
```
Source: `../../../vice-and-order-research/mods/dossiers/extra-assets-importer/repo/EAI.cs#L131-L145` (@ed23afc5c76442ccc4d1a2cc0817c9bc6f42d256)

The three "New" importers (`DecalsImporterNew`, `SurfacesImporterNew`,
`NetLanesDecalImporterNew`) are the modern JSON pipeline; `LocalizationImporter` and
`AssetPackImporter` are pre-importers that run first. The actual import work is queued via
ExtraLib's boot hook (`EL.AddOnInitialize(Initialize)`) so it runs once the game world is
up, not during `OnLoad`
(`extra-assets-importer/repo/EAI.cs#L160`, @ed23afc5).

---

## Step 1: Lay out the source folders

New-pipeline assets live under the mod's data folder, keyed by each importer's
`ImporterId` (`Decals`, `Surfaces`, `NetLanesDecal`), then a category, then one folder per
asset:

```
%UserData%/ModsData/ExtraAssetsImporter/
  Decals/<Category>/<AssetName>/        # DecalsImporterNew reads here
  Surfaces/<Category>/<AssetName>/      # SurfacesImporterNew
  NetLanesDecal/<Category>/<AssetName>/ # NetLanesDecalImporterNew
  _AssetPacks/<PackName>/               # curated packs (see Step 5)
```
The legacy pipeline instead reads flat `CustomDecals/`, `CustomSurfaces/`, and
`CustomNetLanes/` folders, which EAI creates unconditionally on load and wires into the old
importers only when `UseOldImporters` (and the matching per-feature toggle) is on.
Source: `../../../vice-and-order-research/mods/dossiers/extra-assets-importer/repo/EAI.cs#L127-L156` (@ed23afc5c76442ccc4d1a2cc0817c9bc6f42d256)

## Step 2: Define the material as JSON

Each asset folder pairs its texture files with a flat material JSON. The decal material
document is a `{ "UiPriority", "Float": {...}, "Vector": {...} }` shape; the load-bearing
keys include `colossal_DecalLayerMask`, `_DrawOrder`, and the `_BaseColor` /
`colossal_MeshSize` / `colossal_TextureArea` vectors:

```json
{
  "UiPriority": 0,
  "Float": {
    "_Metallic": 1, "_Smoothness": 1,
    "colossal_DecalLayerMask": 1, "_DrawOrder": 0,
    "_DecalStencilRef": 16, "_DecalStencilWriteMask": 16
  },
  "Vector": {
    "_BaseColor": { "x": 1, "y": 1, "z": 1, "w": 1 }
  }
}
```
Source (schema, abridged): `../../../vice-and-order-research/mods/dossiers/extra-assets-importer/repo/ExempleJSON/decal.json#L1-L30` (@ed23afc5c76442ccc4d1a2cc0817c9bc6f42d256)

Do not author this schema from memory. EAI ships an **`ExportDefaultJson`** settings
button that regenerates a `_DefaultJson/` folder from the *installed* importer/component
set every run, so the template always matches the version you have installed
(`extra-assets-importer/repo/MOD/Setting.cs#L33`, @ed23afc5). Study `ExempleJSON/` and the
exported template before onboarding artists.

## Step 3: Let the importer build and register the prefab

You do not call the build yourself - the importer does. `PrefabImporterBase` hashes the
source folder, builds the prefab (a `StaticObjectPrefab` decal, `SurfacePrefab`, or
`NetLaneGeometryPrefab`), attaches a UI object + category override, applies any JSON
`Components`, writes a `PrefabAsset` into the database, and registers it via ExtraLib's
prefab system so it becomes placeable. It **hashes both the source and the built output**,
so an unchanged asset is skipped on the next load and an edited one is rebuilt
(`extra-assets-importer/repo/MOD/AssetImporter/PrefabImporterBase.cs#L39-L236`, @ed23afc5).

The prefabs land in a **custom AssetDatabase** that EAI registers with the game's global
database registry (named `"EAI"`):

```csharp
// EAIAssetDataBaseDescriptor: a custom asset database the game will read from.
public readonly struct EAIAssetDataBaseDescriptor
    : IAssetDatabaseDescriptor<EAIAssetDataBaseDescriptor>, IEquatable<EAIAssetDataBaseDescriptor>
{
    public static string kRootPath => EAIDataBaseManager.eaiDataBase.ActualDataBasePath;
    public bool canWriteSettings => false;
    public string name => "EAI";
    public IDataSourceProvider dataSourceProvider =>
        new FileSystemDataSource(name, kRootPath, assetFactory);
}
```
Source: `../../../vice-and-order-research/mods/dossiers/extra-assets-importer/repo/MOD/DataBase/EAIAssetDataBaseDescriptor.cs#L7-L20` (@ed23afc5c76442ccc4d1a2cc0817c9bc6f42d256)

Registration goes through `AssetDatabase.global.RegisterDatabase(EAIAssetDataBase)`, after
which it is populated from disk; the index is versioned (`DataBaseVersion = 3`) and reset
on a version mismatch.
Source: `../../../vice-and-order-research/mods/dossiers/extra-assets-importer/repo/MOD/DataBase/EAIDataBaseManager.cs#L22,#L103-L129` (@ed23afc5c76442ccc4d1a2cc0817c9bc6f42d256)

## Step 4: Register a COUI host so icons resolve

Custom assets need icons the UI can address. EAI registers a **COUI host** keyed
`extraassetsimporter`, giving every icon a `coui://extraassetsimporter/...` URL, again via
an ExtraLib helper:

```csharp
internal const string IconsResourceKey = "extraassetsimporter";
internal static readonly string COUIBaseLocation = $"coui://{IconsResourceKey}";
// coui://extraassetsimporter

internal static void LoadIcons(string path)
    => ExtraLib.Helpers.Icons.LoadIconsFolder(IconsResourceKey, path);
```
Source: `../../../vice-and-order-research/mods/dossiers/extra-assets-importer/repo/MOD/Icons.cs#L10-L18` (@ed23afc5c76442ccc4d1a2cc0817c9bc6f42d256)

This is the reusable **COUI icon/host registration** technique (family H in the
[technique index](../../technique-index.md)); Write Everywhere and Unified Icon Library use
the same host-registration idea.

## Step 5: Ship a pack - either build one or hook into EAI

Two distribution shapes, both source-visible in EAI:

- **A curated pack (`_AssetPacks/`).** Drop a pack folder with an `AssetPack.json`; the
  `AssetPackImporter` (a pre-importer keyed on that file) spawns an `AssetPackPrefab` with
  a localized name/icon, and `BuildAllAssetPacks` compiles the pack into a redistributable
  per-pack database under `ImportedData`
  (`extra-assets-importer/repo/MOD/AssetImporter/Importers/AssetPackImporter.cs#L16-L122`,
  @ed23afc5).
- **A thin C# pack mod that hooks EAI.** Ship your own `IMod` that hands your folder to
  EAI's public entry point in `OnLoad`:

  ```csharp
  public static void LoadCustomAssets(string modPath)
  {
      AssetsImporterManager.AddAssetFolder(modPath);
      // also picks up any CustomSurfaces/CustomDecals/CustomNetLanes subfolders
      if (Directory.Exists(Path.Combine(modPath, "CustomSurfaces")))
          SurfacesImporter.AddCustomSurfacesFolder(Path.Combine(modPath, "CustomSurfaces"));
      // ...
  }
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/extra-assets-importer/repo/EAI.cs#L215-L233` (@ed23afc5c76442ccc4d1a2cc0817c9bc6f42d256)

  Call `ExtraAssetsImporter.EAI.LoadCustomAssets(path)` from your mod's `OnLoad`. This is
  exactly what community signage packs (e.g. the Sao Paulo Metro Decals Pack) do.

---

## Pitfalls & gotchas

- **ExtraLib must load first.** EAI's runtime is entirely ExtraLib-backed; if ExtraLib is
  missing or loads after EAI, localization/prefab calls NRE at load (GitHub issue #28).
  Keep ExtraLib installed and earlier in the playset.
- **Never hand-edit the compiled database.** EAI hashes source *and* build folders; edit
  the JSON/textures under `<ImporterId>/...` (new) or `Custom*/` (old) and let the importer
  regenerate. Manual edits to the compiled `.Database` are overwritten once a hash changes
  (`extra-assets-importer/repo/MOD/AssetImporter/PrefabImporterBase.cs#L79-L124`, @ed23afc5).
- **`DeleteDataBaseOnClose` is a destructive, unguarded toggle.** Its setter flips a
  write-once bool with no confirmation dialog; on dispose EAI deletes the whole `.Database`
  (`extra-assets-importer/repo/MOD/Setting.cs#L73`, @ed23afc5). Do not expose it casually.
- **`DeleteNotLoadedAssets` (default on) prunes anything not revalidated this session.**
  Creators who keep experimental packs offline can lose cached builds
  (`extra-assets-importer/repo/MOD/Setting.cs#L69`, @ed23afc5).
- **`UnLoadCustomAssets` is a no-op.** It is `[Obsolete]` and its body is fully commented
  out; do not rely on it to tear anything down
  (`extra-assets-importer/repo/EAI.cs#L235-L241`, @ed23afc5).
- **This path is transitional.** The maintainer intends to deprecate runtime import when
  the official editor ships (a "convert EAI packs to editor-compatible packs" step already
  exists). Warn players that unofficial assets may need republishing, and to keep save
  backups. `Needs Verification (in-game)`: the exact editor-migration output mapping.

## Variations

- **Old vs. new pipeline / compatibility naming.** EAI can make a single asset masquerade
  under ELT2/ELT3/LocalAsset/PreEditor names via the `ObsoleteIdentifiers` component and
  the two compatibility dropdowns - flip the dropdown instead of duplicating packs
  (`extra-assets-importer/repo/MOD/AssetImporter/PrefabImporterBase.cs#L248-L297`, @ed23afc5).
- **Async re-marshalling.** New importers run on background `Task.Run` and re-marshal
  prefab-system mutations onto the main thread (technique AA, MainThreadDispatcher). See
  the [technique index](../../technique-index.md).

## See also

- **Manage/audit the packs you ship:** [Asset pack management](./asset-pack-management.md).
- **Author the source asset first:** [PBR asset workflow](./pbr-asset-workflow.md).
- **The icon-host technique in general:** [Technique Index, family H](../../technique-index.md).
- **The ExtraLib dependency:** [shared libraries reference](../../reference/shared-libraries/extralib.md)
  (forward-ref).

## Sources

- Canonical mod (dossier + repo): `extra-assets-importer`
  @ed23afc5c76442ccc4d1a2cc0817c9bc6f42d256 (tag v1.7.4) -
  `repo/EAI.cs`, `repo/MOD/Setting.cs`, `repo/MOD/Icons.cs`,
  `repo/MOD/AssetImporter/PrefabImporterBase.cs`,
  `repo/MOD/AssetImporter/Importers/AssetPackImporter.cs`,
  `repo/MOD/DataBase/EAIAssetDataBaseDescriptor.cs`,
  `repo/MOD/DataBase/EAIDataBaseManager.cs`, `repo/ExempleJSON/decal.json`.
- Dependency: ExtraLib (Paradox Mods id `75724`) - hard build + runtime dependency.
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding
