---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: Enumerate installed mods / the active playset"
recipe: modsdata-playset-autodiscovery
technique_family: "BF - Enumerate installed mods / the active playset"
diataxis: how-to
source_version: "~1.6.x (i18n-everywhere@d9285c2; date-pinned, static source only)"
last_reverified: "2026-07-05"
status: source-verified
canonical_mods:
  - i18n-everywhere@d9285c2490079c6d67303da536207b47b106ce64
technique_applicability: [platform]
Created: 2026-07-05
Updated: 2026-07-05
Owners:
  - codex
---

# Enumerate installed mods / the active playset

> Discover which *other* mods are installed and which are enabled in the current
> playset - by combining on-disk directory scans with the PDX-SDK playset query, and
> falling back to the game's own `ModManager` when the SDK path is unavailable.

## Problem
Your mod needs to know what *else* is loaded: to find contributor mods that ship
optional data for you (extra locales, asset packs, patch files), to detect a
dependency, or to enumerate the user's active playset. There is no single "list all
mods" call that covers every case - subscribed workshop mods, local development mods,
and mod *data* directories all live in different places, and the tidy PDX-SDK query
can be disabled or throw. You want one routine that scans all the sources and degrades
gracefully.

## Solution
Scan three sources in order and merge the results, then fall back to the game's
`ModManager` if anything fails. i18n-everywhere's `CacheMods` does exactly this: (1)
walk `ModsData/` for mod *data* directories, (2) walk the local `Mods/` directory for
development mods, (3) ask the PDX SDK for the enabled mods in the active playset via
`PlatformManager.instance.GetPSI<PdxSdkPlatform>("PdxSdk").GetModsInActivePlayset()`.
If the whole block throws (or the user disabled the new path in settings), it calls
`LegacyCacheMods`, which iterates `GameManager.instance.modManager` directly. The
PDX-SDK playset query plus the `ModManager` fallback is the reusable core; the
directory scans are how you find data that ships *alongside* mods rather than as code.

## Steps & Code

### 1. Query the active playset through the PDX SDK

`PlatformManager.instance.GetPSI<PdxSdkPlatform>("PdxSdk")` returns the Paradox
platform interface; `GetModsInActivePlayset()` is async and returns the enabled mods.
i18n-everywhere blocks on it synchronously and treats a failure as an empty set:

```csharp
private static HashSet<Mod> TrickyGetActiveMods()
{
    PdxSdkPlatform manager = PlatformManager.instance.GetPSI<PdxSdkPlatform>("PdxSdk");

    // PdxSdk now handle errors, they will return an empty HashSet when error.
    HashSet<Mod> playsetResult = manager.GetModsInActivePlayset()
        .ConfigureAwait(false)
        .GetAwaiter()
        .GetResult();
    return playsetResult;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/i18n-everywhere/repo/I18NEverywhere/Mod.cs#L465-L475` (@d9285c2490079c6d67303da536207b47b106ce64)

`Mod` here is `Colossal.PSI.Common.Mod`; the fields consumed downstream are
`mod.path` (install directory) and `mod.displayName`
(`../../../vice-and-order-research/mods/dossiers/i18n-everywhere/repo/I18NEverywhere/Mod.cs#L564-L582`, @d9285c2490079c6d67303da536207b47b106ce64).

### 2. Scan `ModsData/` for mod *data* directories

Code mods live under `Mods/`, but per-mod *data* (settings, shipped content) lives
under `<UserDataPath>/ModsData/<ModName>`. Enumerate it with `EnvPath.kUserDataPath`:

```csharp
if (Directory.Exists(Path.Combine(EnvPath.kUserDataPath, "ModsData")))
{
    DirectoryInfo[] modsData =
        new DirectoryInfo(Path.Combine(EnvPath.kUserDataPath, "ModsData")).GetDirectories("*",
            SearchOption.TopDirectoryOnly);
    foreach (DirectoryInfo info in modsData)
    {
        if (!info.Name.EndsWith("Localization", StringComparison.InvariantCulture))
            continue;                                    // this mod's own filter
        if (File.Exists(Path.Combine(info.FullName, "i18n.json")))
            CachedLanguagePacks.Add(new ModInfo { Name = info.Name, /* ... */ });
    }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/i18n-everywhere/repo/I18NEverywhere/Mod.cs#L505-L528` (@d9285c2490079c6d67303da536207b47b106ce64)

The `EndsWith("Localization")` / `i18n.json` checks are this mod's own filter for
language packs - swap in whatever marker file identifies data meant for *you*.

### 3. Scan the local `Mods/` directory (development mods)

Locally installed / development mods live under `<UserDataPath>/Mods/`. Skip hidden
and disabled directories (names starting with `.` or `~`):

```csharp
DirectoryInfo[] localMods = new DirectoryInfo(Path.Combine(EnvPath.kUserDataPath, "Mods")).GetDirectories();
foreach (DirectoryInfo localMod in localMods)
{
    if (localMod.Name.StartsWith('.') || localMod.Name.StartsWith('~'))
        continue;                                        // hidden / disabled
    // ... classify by a marker file, then add to your list ...
    CachedMods.Add(new ModInfo { Name = localMod.Name, Path = Path.Combine(localMod.FullName, "lang") });
}
```
Source: `../../../vice-and-order-research/mods/dossiers/i18n-everywhere/repo/I18NEverywhere/Mod.cs#L531-L557` (@d9285c2490079c6d67303da536207b47b106ce64)

### 4. Merge SDK results, guarded by a settings toggle and a catch-all fallback

The whole three-source scan runs only when the `UseNewModDetectMethod` setting is on;
any exception drops to `LegacyCacheMods`. This is the graceful-degradation core:

```csharp
private static void CacheMods()
{
    if (Setting.UseNewModDetectMethod)
    {
        try
        {
            // Source 1: ModsData/ ...  Source 2: Mods/ ...
            var mods = TrickyGetActiveMods();            // Source 3: PDX SDK playset
            foreach (Mod mod in mods)
            {
                string absolutePath = Path.GetFullPath(mod.path);
                CachedMods.Add(new ModInfo { Name = mod.displayName ?? absolutePath, /* ... */ });
            }
        }
        catch (Exception trickyException)
        {
            Logger.Error(trickyException, trickyException.Message);
            LegacyCacheMods();                           // SDK path failed -> fall back
        }
    }
    else { LegacyCacheMods(); }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/i18n-everywhere/repo/I18NEverywhere/Mod.cs#L498-L594` (@d9285c2490079c6d67303da536207b47b106ce64)

`UseNewModDetectMethod` defaults to `true`
(`../../../vice-and-order-research/mods/dossiers/i18n-everywhere/repo/I18NEverywhere/Setting.cs#L38`, @d9285c2490079c6d67303da536207b47b106ce64),
so the SDK path is the norm and the legacy walk is the safety net.

### 5. Fallback: iterate `GameManager.instance.modManager`

When the SDK path is unavailable, enumerate the game's own mod manager. Each entry is
a `ModManager.ModInfo` exposing `.asset.name` and `.asset.path`; dedup via a
`HashSet`:

```csharp
private static void LegacyCacheMods()
{
    HashSet<ModInfo> set = new();
    foreach (ModManager.ModInfo modInfo in GameManager.instance.modManager)
    {
        if (modInfo.asset.name == "0Harmony") continue;  // skip the Harmony shim
        string modDir = Path.GetDirectoryName(modInfo.asset.path);
        if (string.IsNullOrEmpty(modDir)) continue;
        set.Add(new ModInfo { Name = modInfo.asset.name, Path = Path.Combine(modDir, "lang") });
    }
    CachedMods = [.. set];
}
```
Source: `../../../vice-and-order-research/mods/dossiers/i18n-everywhere/repo/I18NEverywhere/Mod.cs#L606-L631` (@d9285c2490079c6d67303da536207b47b106ce64)

`GameManager.instance.modManager` is directly `foreach`-able and needs no PDX SDK, so
it works even when the platform layer is absent - but note it yields *loaded* mods,
not the *playset*, and (in this mod) carries no marker to distinguish data types.

## Pitfalls & gotchas

- **The playset query is async and can throw - always guard it.** `CacheMods` blocks
  on `GetModsInActivePlayset()` with `.GetAwaiter().GetResult()` inside a `try`, and
  the source comment notes the SDK "will return an empty HashSet when error"
  (`Mod.cs#L465-L475`). Never call the SDK path unguarded: on failure you want the
  `ModManager` fallback, not an unhandled exception during startup.

- **Playset != loaded mods.** The SDK path (`GetModsInActivePlayset`) returns the
  *enabled playset*; the legacy path (`GameManager.instance.modManager`) returns what
  the game actually loaded. They can differ. Pick the one that matches your question,
  and know that the fallback silently changes the semantics of the answer.

- **Main-menu stall risk on large mod lists.** This scan does synchronous directory
  enumeration over `ModsData/` and `Mods/` plus a blocking SDK await, and it runs
  during load. On users with hundreds of subscribed mods this can add up. **Profile
  it** before running it on a hot path, and cache the result (i18n-everywhere caches
  into `CachedMods` / `CachedLanguagePacks` static lists rather than re-scanning). The
  actual per-mod cost on a large playset is `Needs Verification (in-game)`.

- **Dedup by path, not name.** `ModInfo.Equals`/`GetHashCode` key on `Path`, not
  `Name` (`../../../vice-and-order-research/mods/dossiers/i18n-everywhere/repo/I18NEverywhere/Models/ModInfo.cs#L11-L29`,
  @d9285c2490079c6d67303da536207b47b106ce64). Two mods can share a display name;
  their install paths won't. If you build your own set, key on the path.

- **`mod.displayName` can be null.** The SDK loop falls back to the absolute path
  (`mod.displayName ?? absolutePath`, `Mod.cs#L564-L582`). Do not assume a name is
  present.

- **The legacy path skips `0Harmony` and can miss data types.** The fallback filters
  the Harmony shim asset by literal name and only records a `lang` path - it has no
  marker-file classification, so any "is this a data pack for me" logic you rely on in
  the SDK path is absent in the fallback (`Mod.cs#L606-L631`).

## Variations

- **Marker-file classification.** i18n-everywhere splits results into regular mods vs
  language packs by probing for `i18n.json` in each candidate directory
  (`Mod.cs#L519-L523`, `#L545-L556`, `#L568-L576`). Substitute your own marker file
  (e.g. an asset-pack manifest) to auto-discover mods that ship content *for* your
  mod.

- **SDK-only or ModManager-only.** If you only need the enabled playset, call
  `TrickyGetActiveMods()` alone. If you need every loaded mod regardless of platform
  (e.g. dependency detection that must work offline / on non-PDX installs), iterate
  `GameManager.instance.modManager` directly as in step 5.

- **Expose a user toggle.** Gating the SDK path behind a setting
  (`UseNewModDetectMethod`) lets users force the legacy walk when the platform layer
  misbehaves - a cheap escape hatch for a fragile external API.

## See also
- Explanation: [dependency strategy](../../explanation/dependency-strategy.md) (when
  to detect vs. hard-depend on another mod).
- How-to: [dependency handling](../ui/dependency-handling.md) (acting on what you
  discovered).
- How-to: [asset pack management](../content/asset-pack-management.md) (discovering
  and loading content shipped by other mods).
- Case study: [localization-and-ui](../../case-studies/localization-and-ui.md) (this
  routine in its original localization context).

## Sources
- Canonical mods (dossier + repo):
  - `i18n-everywhere` @d9285c2490079c6d67303da536207b47b106ce64 - `repo/I18NEverywhere/Mod.cs`, `repo/I18NEverywhere/Models/ModInfo.cs`, `repo/I18NEverywhere/Setting.cs`
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding
