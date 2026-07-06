---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Build a Write Everywhere Module (Image, Font, and Mesh Atlases)"
Summary: How to extend Write Everywhere with a content module that ships image atlases, TTF fonts, and OBJ meshes - registered through WE's reverse-patch Bridge API so your module never hard-references the WE assembly.
diataxis: how-to
source_version: "~1.5.x (write-everywhere@13c70eb; date-pinned, static source only)"
last_reverified: "2026-07-04"
canonical_mods:
  - write-everywhere@13c70eb04e6bed152257c516a982455148f591a5
technique_applicability: [ui, content, tooling]
status: source-verified
Created: 2026-07-03
Updated: 2026-07-03
Owners:
  - codex
References:
  - Label: Import custom assets at runtime (sibling content path)
    Path: ./unofficial-asset-import.md
  - Label: "Technique D - Harmony redirector / reverse-patch bridge"
    Path: ../../technique-index.md
  - Label: "Reference: Klyte.Commons / BasicIMod (forward-ref)"
    Path: ../../reference/shared-libraries/README.md
---

# Build a Write Everywhere Module (Image, Font, and Mesh Atlases)

**Write Everywhere (WE)** lets players place custom text, images, and meshes anywhere in
the city. A **module** is a separate mod that ships additional content - image atlases,
TTF fonts, OBJ meshes, and layouts - and registers it with WE so it appears in WE's
pickers. This how-to shows the registration path that matters: WE deliberately makes you
integrate through a **reverse-patch Bridge API** rather than by referencing its DLL.

The worked example is WE itself, cited at pinned commit
`13c70eb04e6bed152257c516a982455148f591a5` (tag `v2.0.0r12`, Paradox Mods id `92908`,
source `github.com/klyte45/CS2-WriteEverywhere`). The official module scaffold is
`github.com/klyte45/CS2-WEModuleTemplate`.

> **Dependencies.** WE is built on Klyte's `BasicIMod` framework (the `Klyte.Commons`
> library, vendored into WE as the `BelzontWE/Commons` git submodule) and declares
> **Unified Icon Library** (Paradox Mods id `74417`) as a hard dependency
> (`write-everywhere/repo/BelzontWE/WriteEverywhereCS2Mod.cs` / `BelzontWE.csproj#L36-L39`,
> @13c70eb0). WE also uses Harmony 2.2.2 (`BelzontWE.csproj#L191`, @13c70eb0).

---

## The integration rule: reverse-patch, do not hard-reference

WE exposes its module-facing API as **seven static Bridge classes** under
`BelzontWE/Bridge/`: `FontManagementBridge`, `ImageManagementBridge`, `MeshManagementBridge`,
`TemplatesManagementBridge`, `ModuleOptionsBridge`, `LocalizationBridge`, and
`RoadFnBridge`. Every one is marked `[Obsolete(..., error: true)]`, so a direct compile-time
reference to it **fails to build on purpose**:

```csharp
[Obsolete("Don't reference methods on this class directly. Always use reverse patch to " +
          "access them, and don't use this mod DLL as hard dependency of your own mod.", true)]
public static class ImageManagementBridge
{
    public static void RegisterImageAtlas(Assembly mainAssembly, string atlasName,
        string[] imagePaths, Action<string> onCompleteLoading = null)
    { /* defers to a coroutine -> WEAtlasesLibrary.LoadImagesToAtlas(...) */ }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/write-everywhere/repo/BelzontWE/Bridge/ImageManagementBridge.cs#L18-L28` (@13c70eb04e6bed152257c516a982455148f591a5)

Consumers reach these methods with `Harmony.ReversePatch`: you declare a stub method with
the same signature in your module and reverse-patch WE's real method onto it, so your
module keeps working (and simply no-ops) even when WE is absent. The reverse-patch
machinery lives in the Commons submodule:

```csharp
Harmony.ReversePatch(srcMethod, new HarmonyMethod(method));
```
Source: `../../../vice-and-order-research/mods/dossiers/write-everywhere/repo/BelzontWE/Commons/Utils/BridgeUtils.cs#L166` (submodule `CS2-BelzontCommons` @3698b64795eec8fd7effe75cf08cb4d1cbe3c674 - the `BelzontWE/Commons` gitlink recorded at WE commit 13c70eb0)

This is the reusable **Harmony redirector / reverse-patch bridge** technique (family D in
the [technique index](../../technique-index.md)) - a versioned static API the host mod owns
and consumers access without compile-time coupling.

## Step 1: Scaffold the module from the template

Start from `CS2-WEModuleTemplate`, rename the project/solution so your DLL and `mod.json`
carry your module ID, and update the `.csproj` metadata (display name, description, Paradox
mod ID) and thumbnails/screenshots.

> **Needs Verification.** The template's internal `Resources/` folder layout (e.g.
> `Layouts/`, `Atlases/`, `Fonts/` subfolders) comes from `CS2-WEModuleTemplate`, which is
> a separate repository not part of the cited WE source snapshot. Confirm the current
> subfolder names against the template repo before wiring your build to them. The verified
> facts below are the *WE-side* registration APIs and the runtime folders WE reads.

## Step 2: Register content through the bridges (in `OnLoad`)

Use one bridge call per content kind. Pass your module's own `Assembly` so WE can attribute
the content to your module and key its caches per-mod:

- **Image atlas** - `ImageManagementBridge.RegisterImageAtlas(mainAssembly, atlasName, imagePaths, onComplete)`
  (async; resolves to `WEAtlasesLibrary.LoadImagesToAtlas`)
  (`.../BelzontWE/Bridge/ImageManagementBridge.cs#L26`, @13c70eb0).
- **Fonts** - `FontManagementBridge.RegisterModFonts(mainAssembly, rootFolder)`
  (`.../BelzontWE/Bridge/FontManagementBridge.cs#L12`, @13c70eb0).
- **Meshes** - `MeshManagementBridge.RegisterMesh(mainAssembly, meshName, meshObjFilePath)`,
  or `RegisterMeshFromMemory(mainAssembly, meshName, vertices, normals, uv, triangles)`
  (`.../BelzontWE/Bridge/MeshManagementBridge.cs#L12-L15`, @13c70eb0).

Atlas registration is intentionally asynchronous - `RegisterImageAtlas` starts a coroutine
and reports progress/errors through WE's localized notification helper, so a large atlas
does not stall the load:

```csharp
public static void RegisterImageAtlas(Assembly mainAssembly, string atlasName,
    string[] imagePaths, Action<string> onCompleteLoading = null)
{
    new CoroutineWithData<string>(GameManager.instance,
        RegisterImageAtlas_Internal(mainAssembly, atlasName, imagePaths), onCompleteLoading);
}
```
Source: `../../../vice-and-order-research/mods/dossiers/write-everywhere/repo/BelzontWE/Bridge/ImageManagementBridge.cs#L26-L28` (@13c70eb04e6bed152257c516a982455148f591a5)

## Step 3: Expose module options (optional)

If your module needs its own settings inside WE's UI, register them through
`WEModulesSystem.RegisterOptions`, which takes your assembly and a keyed option map:

```csharp
public void RegisterOptions(Assembly mainAssembly, Dictionary<string, (int, object)> options)
```
Source: `../../../vice-and-order-research/mods/dossiers/write-everywhere/repo/BelzontWE/Controllers/WEModulesSystem.cs#L20` (@13c70eb04e6bed152257c516a982455148f591a5)

(The corresponding reverse-patch surface is `ModuleOptionsBridge` in `BelzontWE/Bridge/`.)

## Step 4: Know where WE reads runtime assets

WE roots all of its file-system content under `BasicIMod.ModSettingsRootFolder`, declared
by the `[FileLocation("K45_WE_settings")]` attribute on `WEModData`:

```csharp
[FileLocation("K45_WE_settings")]
```
Source: `../../../vice-and-order-research/mods/dossiers/write-everywhere/repo/BelzontWE/WEModData.cs#L17` (@13c70eb04e6bed152257c516a982455148f591a5)

Under that root, the asset libraries resolve fixed subfolders (all verified at @13c70eb0):

| Content | Folder | Constant (source) |
|---------|--------|-------------------|
| Image atlases | `imageAtlases/` | `WEAtlasesLibrary.IMAGES_FOLDER` (`Font/Sprites/WEAtlasesLibrary.cs#L35`) |
| Fonts (TTF) | `fonts/` | `FontServer.FONTS_FILES_FOLDER` (`Font/FontServer.cs#L30`) |
| Meshes (OBJ) | `objMeshes/` | `WECustomMeshLibrary.MESHES_FOLDER` (`Mesh/WECustomMeshLibrary.cs#L25`) |
| Layouts | `layouts/` | `WETemplateManager.SAVED_PREFABS_FOLDER` (`Templates/WETemplateManager.cs#L29`) |

WE 2.0 also caches virtual-texture atlas data on disk under `.cache/vtAtlases`,
`.cache/vtTiles`, and `.cache/vtTilesCity`, created on library startup
(`.../BelzontWE/Font/Sprites/WEAtlasesLibrary.cs#L37-L59`, @13c70eb0).

## Step 5: Meshes are OBJ, image-node only

Save custom meshes as Wavefront `.obj` (one mesh per file, with vertices, normals, UVs, and
triangles) and register them via `MeshManagementBridge`. Reference the mesh from your image
node so WE can project the atlas image onto it.

> **Needs Verification (in-game).** The legacy guidance that "mesh support is limited to
> image nodes" and "fonts must be installed manually via the City Fonts tab" reflects older
> WE behaviour and is not established from the cited 2.0 source. Confirm against the current
> WE UI. What *is* source-verified is 2.0's new `WEGamePropSpawnSystem`, which spawns real
> in-game props from a "GameProp" layout node with X/Y/Z array instancing
> (`.../BelzontWE/Systems/WEGamePropSpawnSystem.cs`, @13c70eb0).

---

## Pitfalls & gotchas

- **Do not add WE as a hard dependency.** The Bridge classes are `[Obsolete(..., true)]`;
  referencing them directly will not compile. Reverse-patch them so your module degrades
  gracefully when WE is absent
  (`.../BelzontWE/Bridge/ImageManagementBridge.cs#L18`, @13c70eb0).
- **Pass your own `Assembly` to every bridge call.** WE keys per-mod caches and
  attribution off the calling assembly; passing the wrong one misattributes or collides
  atlases.
- **Atlas loads are async.** `RegisterImageAtlas` returns immediately and completes later
  via its callback - do not assume the atlas exists synchronously after the call.
- **WE reverse-patches the game's `PrefabSystem`.** WE's `PrefabSystemOverrides` (a Commons
  `Redirector`) patches `PrefabSystem.UpdatePrefabs` to mark WE templates dirty when the
  game reloads prefabs; a module that also patches prefab reloads should expect to coexist
  with this (`.../BelzontWE/Overrides/PrefabSystemOverrides.cs#L12-L39`, @13c70eb0).
- **Version-pin against a WE release.** The Bridge signatures are the compatibility contract;
  they are the surface you reverse-patch, so re-verify them when WE bumps major versions
  (this page is checked at `v2.0.0r12`).

## See also

- **Sibling runtime-content path (decals/surfaces):**
  [Unofficial asset import](./unofficial-asset-import.md).
- **The reverse-patch bridge technique in general:**
  [Technique Index, family D](../../technique-index.md).
- **The Klyte.Commons / BasicIMod framework WE builds on:**
  [shared libraries reference](../../reference/shared-libraries/README.md) (forward-ref).

## Sources

- Canonical mod (dossier + repo): `write-everywhere`
  @13c70eb04e6bed152257c516a982455148f591a5 (tag v2.0.0r12) -
  `repo/BelzontWE/Bridge/*` (Image/Font/Mesh/ModuleOptions bridges),
  `repo/BelzontWE/Controllers/WEModulesSystem.cs`, `repo/BelzontWE/WEModData.cs`,
  `repo/BelzontWE/Font/FontServer.cs`, `repo/BelzontWE/Font/Sprites/WEAtlasesLibrary.cs`,
  `repo/BelzontWE/Mesh/WECustomMeshLibrary.cs`, `repo/BelzontWE/Templates/WETemplateManager.cs`,
  `repo/BelzontWE/Overrides/PrefabSystemOverrides.cs`, `repo/BelzontWE.csproj`.
  Reverse-patch machinery: `repo/BelzontWE/Commons/Utils/BridgeUtils.cs` in the
  `CS2-BelzontCommons` submodule @3698b64795eec8fd7effe75cf08cb4d1cbe3c674.
- Dependencies: Unified Icon Library (Paradox Mods id `74417`); Klyte.Commons / BasicIMod
  (vendored submodule).
- Module scaffold (not part of the cited snapshot): `github.com/klyte45/CS2-WEModuleTemplate`.
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding
