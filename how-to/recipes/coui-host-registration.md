---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: COUI icon / host registration (AddHostLocation)"
recipe: coui-host-registration
technique_family: "H - COUI icon / host registration (AddHostLocation)"
diataxis: how-to
source_version: "~1.5.x (unified-icon-library@b200d89; date-pinned, static source only)"
last_reverified: "2026-07-04"
status: source-verified
canonical_mods:
  - unified-icon-library@b200d8901346457f97c032e86e1d8437a12c0cc8
  - extra-lib@4879487b7df62e83676030a87f92d4b102955eee
  - extra-assets-importer@ed23afc5c76442ccc4d1a2cc0817c9bc6f42d256
  - write-everywhere@13c70eb04e6bed152257c516a982455148f591a5
technique_applicability: [ui, media]
Created: 2026-07-04
Updated: 2026-07-04
Owners:
  - codex
---

# COUI icon / host registration (AddHostLocation)

> Mount your mod's on-disk icon/asset folder as a `coui://` host so the React UI
> can load your SVGs/PNGs by URL, with one `AddHostLocation` call.

## Problem
Your C# side has icons (SVG/PNG) sitting in a folder next to your mod DLL, and your
React UI needs to reference them - as `<img src="coui://mymod/Standard/Plus.svg">`
or an `m_Icon` string on a prefab. The game's UI runs in a sandboxed browser
(CEF/Coherent) that cannot read arbitrary disk paths; it only resolves `coui://`
URLs against **registered host locations**. Until you register one, every icon URL
your UI points at your mod resolves to nothing (broken image). You need to bind a
short host key (e.g. `uil`) to a real directory once, at load.

## Solution
Call `UIManager.defaultUISystem.AddHostLocation(key, path)` in your `IMod.OnLoad`.
It maps the host key to a filesystem directory; from then on `coui://<key>/<rel>`
resolves to `<path>/<rel>` for the whole UI (yours and other mods'). The path is
almost always derived at runtime from your executing assembly's directory, because
the install path is not known at build time. Two shapes exist: the minimal 2-arg
form for a **static** catalog you mount once and never touch, and the 3-arg form
(`+ shouldWatch`) plus `RemoveHostLocation` for **dynamic** hosts whose backing
folders come and go (or that you want unmounted on dispose). Pick by lifecycle:
static -> mount and forget; dynamic -> track and remove.

## Steps & Code

### 1. Resolve your mod's on-disk directory at load

The host path must be a real directory, discovered at runtime. Unified Icon Library
finds its own executable asset through the `AssetDatabase` and caches the containing
folder as `AssemblyPath`:

```csharp
string assemblyName = Assembly.GetExecutingAssembly().FullName;
ExecutableAsset modAsset = AssetDatabase.global.GetAsset(
    SearchFilter<ExecutableAsset>.ByCondition(x => x.definition?.FullName == assemblyName));
if (modAsset is null) { Log.Error("mod executable asset not found"); return null; }
s_assemblyPath = Path.GetDirectoryName(modAsset.GetMeta().path);   // cached
```
Source: `../../../vice-and-order-research/mods/dossiers/unified-icon-library/repo/Code/Mod.cs#L46-L55` (@b200d8901346457f97c032e86e1d8437a12c0cc8)

### 2. Mount the host in `OnLoad` (minimal 2-arg form)

One call binds the host key `uil` to `<AssemblyPath>/Icons/`. After this, the UI can
load `coui://uil/<anything under Icons>`:

```csharp
public void OnLoad(UpdateSystem updateSystem)
{
    Instance = this;
    Log = LogManager.GetLogger(ModName);
    // Add mod UI resource directory to UI resource handler.
    UIManager.defaultUISystem.AddHostLocation("uil",  AssemblyPath + "/Icons/");
}
```
Source: `../../../vice-and-order-research/mods/dossiers/unified-icon-library/repo/Code/Mod.cs#L72-L87` (@b200d8901346457f97c032e86e1d8437a12c0cc8)

### 3. Reference the icon from React by URL

Once the host is mounted, the URL is `coui://<key>/<path-relative-to-host-dir>`. Here
write-everywhere (a *consumer*, not the registrant) loads icons out of UIL's `uil`
host straight into its React frontend - `coui://uil/Standard/Plus.svg` resolves to
`<UIL AssemblyPath>/Icons/Standard/Plus.svg`:

```tsx
const i_addItem = "coui://uil/Standard/Plus.svg";
const i_bookmarkMods = "coui://uil/Standard/Puzzle.svg";
```
Source: `../../../vice-and-order-research/mods/dossiers/write-everywhere/repo/_Frontends/UI/k45-we-vuio/src/mainUI/CityLayoutsTab.tsx#L20-L21` (@13c70eb04e6bed152257c516a982455148f591a5)

This is the payoff and the contract: the host key is a **global namespace** shared
across all mods. UIL registers `uil`; any mod's UI (even another author's) can then
reference `coui://uil/...`. Keep your key unique and stable - it is effectively
public API.

## Pitfalls & gotchas

- **UIL never removes its host (and that is fine here).** Unified Icon Library's
  `OnDispose` only nulls its instance and logs; there is **no** `RemoveHostLocation`
  call. For a static, always-present icon catalog that lives as long as the game
  session, leaving the host mounted is intentional and harmless:

  ```csharp
  public void OnDispose()
  {
      Log.Info("disposing");
      Instance = null;
  }
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/unified-icon-library/repo/Code/Mod.cs#L92-L96` (@b200d8901346457f97c032e86e1d8437a12c0cc8)

  Do NOT copy this blindly for a **dynamic** host (one whose folder is temporary, or
  a host you re-register). A stale mapping to a deleted directory is a broken-icon
  source. If the backing folder's lifetime is shorter than the session, remove it
  (see Variations).

- **Host keys are a shared global namespace - collisions clobber.** `AddHostLocation`
  keys are not scoped to your mod. Two mods registering the same key both point the
  same `coui://key/...` namespace at their own folders; whoever the UI system
  resolves against wins. Use a distinctive key (`uil`, `extralib`,
  `extraassetsimporter` in the canonical mods) and treat it as public API other mods
  may depend on.

- **The path must exist and be your real install dir - don't hard-code it.** All
  three registrant mods derive the directory at runtime from the executing assembly
  (UIL via `AssetDatabase`, ExtraLib via `modManager.TryGetExecutableAsset`). A
  build-time or absolute path will not survive a different install location.

- **Exact overload signatures / `shouldWatch` semantics beyond the call shapes shown
  here are `Needs Verification (in-game)`.** The source proves the 2-arg
  `AddHostLocation(key, path)`, the 3-arg `AddHostLocation(uri, path, shouldWatch)`,
  and `RemoveHostLocation(uri)` / `RemoveHostLocation(uri, path)` forms compile and
  are called; what the game's UI system does with `shouldWatch=true` (live file
  reload) and how multiple paths under one key are merged at resolve time are runtime
  behaviours not visible in this source.

## Variations

- **Dynamic host with explicit unmount (3-arg form + `RemoveHostLocation`).**
  ExtraLib wraps the 3-arg `AddHostLocation(uri, path, shouldWatch)`, tracks every
  `(uri, path)` it has mounted, and exposes matching unload calls. Note the two
  `RemoveHostLocation` shapes: remove the whole key, or remove just one path under
  it:

  ```csharp
  public static void LoadIconsFolder(string uri, string path, bool shouldWatch = false)
  {
      if (pathToIconLoaded.ContainsKey(uri)) { if (pathToIconLoaded[uri].Contains(path)) return; pathToIconLoaded[uri].Add(path); }
      else                                   { pathToIconLoaded.Add(uri, new List<string>() { path }); }
      UIManager.defaultUISystem.AddHostLocation(uri, path, shouldWatch);
  }
  public static void UnLoadIconsFolder(string uri, string path = null)
  {
      if (!pathToIconLoaded.ContainsKey(uri)) return;
      if (path == null) { pathToIconLoaded.Remove(uri); UIManager.defaultUISystem.RemoveHostLocation(uri); }
      else              { pathToIconLoaded[uri].Remove(path); UIManager.defaultUISystem.RemoveHostLocation(uri, path); }
  }
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/extra-lib/repo/MOD/Helpers/Icons.cs#L23-L56` (@4879487b7df62e83676030a87f92d4b102955eee)

  It mounts on load and, unlike UIL, tears down on dispose:

  ```csharp
  Icons.LoadIconsFolder(Icons.IconsResourceKey, fileInfo.Directory.FullName);   // OnLoad
  // ...
  Icons.UnLoadAllIconsFolder();                                                  // OnDispose
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/extra-lib/repo/EL.cs#L57-L88` (@4879487b7df62e83676030a87f92d4b102955eee)

  The `coui://` base string is built once from the same key it registers, so URLs and
  the host mapping cannot drift apart:

  ```csharp
  internal static readonly string IconsResourceKey = "extralib";
  internal static readonly string COUIBaseLocation = $"coui://{IconsResourceKey}";
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/extra-lib/repo/MOD/Helpers/Icons.cs#L11-L12` (@4879487b7df62e83676030a87f92d4b102955eee)

- **Many folders under ONE key, added as content arrives.** Extra Assets Importer
  registers a whole tree of imported-asset folders under the single key
  `extraassetsimporter`, calling `LoadIcons` again for each newly discovered import
  folder (and `UnLoadIcons` to drop one path). This is the multi-path case the 3-arg
  form and the 2-arg `RemoveHostLocation(uri, path)` exist for:

  ```csharp
  internal const string IconsResourceKey = "extraassetsimporter";
  internal static void LoadIcons(string path)   => ExtraLib.Helpers.Icons.LoadIconsFolder(IconsResourceKey, path);
  internal static void UnLoadIcons(string path) => ExtraLib.Helpers.Icons.UnLoadIconsFolder(IconsResourceKey, path);
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/extra-assets-importer/repo/MOD/Icons.cs#L10-L24` (@ed23afc5c76442ccc4d1a2cc0817c9bc6f42d256)

  ```csharp
  if (this is FolderImporter)   Icons.LoadIcons(new DirectoryInfo(path).Parent.FullName);
  else if (this is FileImporter) Icons.LoadIcons(new FileInfo(path).Directory.FullName);
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/extra-assets-importer/repo/MOD/AssetImporter/ImporterBase.cs#L59-L62` (@ed23afc5c76442ccc4d1a2cc0817c9bc6f42d256)

## See also
- Reference: [shared libraries](../../reference/shared-libraries/README.md),
  [unified-icon-library](../../reference/shared-libraries/unified-icon-library.md).
- Explanation: [React UI](../../explanation/react-ui.md).
- Case studies demonstrating it: [localization and UI](../../case-studies/localization-and-ui.md).

## Sources
- Canonical mods (dossier + repo):
  - `unified-icon-library` @b200d8901346457f97c032e86e1d8437a12c0cc8 - `repo/Code/Mod.cs`
  - `extra-lib` @4879487b7df62e83676030a87f92d4b102955eee - `repo/MOD/Helpers/Icons.cs`, `repo/EL.cs`
  - `extra-assets-importer` @ed23afc5c76442ccc4d1a2cc0817c9bc6f42d256 - `repo/MOD/Icons.cs`, `repo/MOD/AssetImporter/ImporterBase.cs`
  - `write-everywhere` @13c70eb04e6bed152257c516a982455148f591a5 - `repo/_Frontends/UI/k45-we-vuio/src/mainUI/CityLayoutsTab.tsx` (consumer)
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding
