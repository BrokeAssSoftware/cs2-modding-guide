---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: External JSON side-car persistence"
recipe: json-sidecar-persistence
technique_family: "F - External JSON side-car persistence"
diataxis: how-to
source_version: "~1.5.2f1 (road-speed-adjuster@e0c0c0b; date-pinned, static source only)"
last_reverified: "2026-07-04"
status: source-verified
canonical_mods:
  - road-speed-adjuster@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed
  - specialized-industrial-zones@8d9153c3fda72e3b79a133bee6c83f9ba7df8b5a
  - smooth-left-hand-traffic@37b2850b10ced1cd17457c25778409e57e00a03e
  - time2work-realistic-trips@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39
  - anarchy@a6311e898d20a775368668b234aaa32f06e3e1eb
  - time-weather-anarchy@71128d3958c31b03163a2cba981909655a38bc83
  - traffic-tool-essentials@10973595ac9ed37f47ca24eb788d4421dc295fa0
technique_applicability: [core]
Created: 2026-07-04
Updated: 2026-07-05
Owners:
  - codex
---

# External JSON side-car persistence

> Persist mod state in a plain JSON file under the game's `ModsData` folder -
> written on save/unload and read back on load - instead of pushing it through the
> ECS save-stream.

## Problem
You have mod state that must survive a session - per-road speed overrides, a set of
opted-out prefabs, a user-editable zone config - but it does **not** belong in the
`ModSetting` (that is global app config, not per-city) and you do not want to author
an ECS `IJsonWritable`/`IJsonReadable` component into the save-stream (versioning,
migration, and load-order pain; the save file is not user-editable). You want a file
you control: readable, hand-editable, and decoupled from the save's binary format.

## Solution
Serialize a small plain-C# data object to a JSON file in
`.../LocalLow/Colossal Order/Cities Skylines II/ModsData/<YourMod>/`, write it when the
game saves or the system tears down, and read it back when a city loads. Two design
axes vary between mods and you must choose deliberately:

1. **Which JSON library** - `Newtonsoft.Json` (`JsonConvert`) or Colossal's built-in
   `Colossal.Json` (`JSON.Load/Dump/MakeInto`). Both are available to a mod; they are
   not interchangeable APIs.
2. **File scope + key** - one file **per save** (keyed by sanitized city name) or one
   **global** file shared by all cities. And inside the file, what you key entries by:
   an `Entity.Index` (fragile across sessions) or a stable string (prefab/zone name).

The three canonical mods below split cleanly across these axes; read the comparison
table before you copy one.

## Steps & Code

The primary walkthrough is **road-speed-adjuster** (Newtonsoft, per-save file); the
Variations section shows the Colossal.Json / global-file alternatives.

### 1. Resolve the `ModsData/<YourMod>` directory and create it

Road Speed Adjuster builds the LocalLow path by hand (`LocalApplicationData` + the
literal `"Low"`) and lazily creates the folder:

```csharp
if (string.IsNullOrEmpty(_baseDirectory))
{
    // Use the game's LocalLow directory
    string localLowPath = Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData) + "Low";
    _baseDirectory = Path.Combine(localLowPath, "Colossal Order", "Cities Skylines II", "ModsData", "RoadSpeedAdjuster");

    if (!Directory.Exists(_baseDirectory))
    {
        Directory.CreateDirectory(_baseDirectory);
    }
}
return _baseDirectory;
```
Source: `../../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/Data/PersistentSpeedStorage.cs#L26-L40` (@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed)

Prefer `Colossal.PSI.Environment.EnvPath.kUserDataPath` over the hand-built string (see
Variations); it resolves the same root without the fragile `+ "Low"` concatenation.

### 2. Pick the per-save filename and load-or-create on init

The file name is `<sanitizedCityName>_<saveId>.json`. If it exists, deserialize it;
otherwise start a fresh in-memory object:

```csharp
string sanitizedName = SanitizeFileName(saveGameName);
string fileName = $"{sanitizedName}_{saveGameId}.json";
_currentFilePath = Path.Combine(BaseDirectory, fileName);

if (File.Exists(_currentFilePath))
{
    LoadFromFile();
}
else
{
    _currentMapData = new MapSpeedData
    {
        MapName = saveGameName, SaveGameId = saveGameId, LastSaved = DateTime.Now
    };
}
```
Source: `../../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/Data/PersistentSpeedStorage.cs#L52-L72` (@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed)

### 3. Define the serializable data object

A plain `[Serializable]` class with public get/set properties. Note the payload is a
`Dictionary<int, RoadSpeedEntry>` keyed by the road's `Entity.Index` - the fragile
choice called out in Pitfalls:

```csharp
[Serializable]
public class MapSpeedData
{
    public string MapName { get; set; }
    public string SaveGameId { get; set; }
    public DateTime LastSaved { get; set; }
    public Dictionary<int, RoadSpeedEntry> Roads { get; set; } = new Dictionary<int, RoadSpeedEntry>();
    public int Version { get; set; } = 1;   // format version for future migrations
}
```
Source: `../../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/Data/RoadSpeedData.cs#L36-L62` (@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed)

### 4. Serialize / deserialize with `JsonConvert`

The write and read are one call each. Both are wrapped so an I/O or parse failure logs
and falls back to a fresh object rather than throwing into the game loop:

```csharp
// Save
string json = JsonConvert.SerializeObject(_currentMapData, Formatting.Indented);
File.WriteAllText(_currentFilePath, json);

// Load
string json = File.ReadAllText(_currentFilePath);
_currentMapData = JsonConvert.DeserializeObject<MapSpeedData>(json);
if (_currentMapData == null)
    _currentMapData = new MapSpeedData();   // null-guard: empty/corrupt file
```
Source: `../../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/Data/PersistentSpeedStorage.cs#L177-L201` (@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed)

### 5. Sanitize the save name before using it as a filename

A city name is user text and can contain path-illegal characters. Strip them, then cap
the length so you never hand the filesystem a bad or over-long name:

```csharp
char[] invalids = Path.GetInvalidFileNameChars();
string sanitized = string.Join("_", fileName.Split(invalids, StringSplitOptions.RemoveEmptyEntries)).TrimEnd('.');

if (sanitized.Length > 50)
    sanitized = sanitized.Substring(0, 50);

return string.IsNullOrEmpty(sanitized) ? "unnamed" : sanitized;
```
Source: `../../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/Data/PersistentSpeedStorage.cs#L236-L243` (@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed)

### 6. Wire the load hook (city ready) and the save hook (teardown)

Initialization is gated in `OnUpdate` on `GameManager.State.WorldReady` + a Game/Editor
mode - not `OnCreate`, because the city name is not available until the world loads:

```csharp
if (!_initialized && _gameManager != null &&
    _gameManager.state >= GameManager.State.WorldReady &&
    (_gameManager.gameMode == GameMode.Game || _gameManager.gameMode == GameMode.Editor))
{
    InitializePersistentStorage();   // -> PersistentSpeedStorage.Initialize(cityName, cityName)
    _initialized = true;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/Systems/RoadSpeedSaveDataSystem.cs#L36-L42` (@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed)

The mirror-image save happens on system teardown (this mod also auto-saves after every
edit, so the file is rarely stale):

```csharp
protected override void OnDestroy()
{
    if (_initialized)
    {
        PersistentSpeedStorage.Save();
    }
    base.OnDestroy();
}
```
Source: `../../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/Systems/RoadSpeedSaveDataSystem.cs#L153-L162` (@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed)

## The two axes, tabulated

| Mod | JSON library | File scope | Filesystem anchor | Entry key |
| --- | --- | --- | --- | --- |
| road-speed-adjuster | `Newtonsoft.Json` (`JsonConvert`) | **per-save** (`<city>_<id>.json`) | hand-built LocalLow + `ModsData/RoadSpeedAdjuster` | `Entity.Index` (int) - **fragile** |
| smooth-left-hand-traffic | **`Colossal.Json`** (`JSON.Load/Dump/MakeInto`) | **global** (one file) | `EnvPath.kUserDataPath` + `ModsData/SmoothLHT` | prefab name (string) - stable |
| specialized-industrial-zones | `Newtonsoft.Json` + `StringEnumConverter` | **global** (one file) | `EnvPath.kUserDataPath` + `ModsData/SpecializedZones` | zone `Name` (string) - stable |

Key takeaway: the library choice and the scope choice are **independent**. Per-save vs
global is dictated by whether the data is about *this city's* entities (per-save) or
*prefab/config* preferences that apply to every city (global).

## Pitfalls & gotchas

- **`Entity.Index` keys do not survive a session.** Road Speed Adjuster keys entries by
  the road's `Entity.Index` (an `int`), e.g. it calls
  `PersistentSpeedStorage.StoreRoadSpeed(targetEdge.Index, ...)`
  (`../../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/Systems/RoadSpeedToolSystem.cs#L429`, @e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed).
  `Entity.Index` is an ECS slot index that is **reassigned on reload** - the entity that
  was index 4200 last session may be a different road (or nothing) next session. A file
  keyed this way can silently re-apply the wrong values. The other two mods sidestep
  this by keying on a **stable string** (prefab name / zone name). If you must persist
  per-instance data, key by a durable identity (a name, a GUID you store, or a
  reconstructable spatial key), not `Entity.Index`. Whether the specific reload
  behaviour corrupts data in the live 1.6.0f1 game is `Needs Verification (in-game)`.

- **Always sanitize the save name.** A city name is arbitrary user text; feeding it to
  `Path.Combine` unsanitized invites path traversal and illegal-character exceptions.
  Strip `Path.GetInvalidFileNameChars()` and cap the length (Step 5). Note the two mods
  sanitize with *different* rules (`PersistentSpeedStorage.SanitizeFileName` caps at 50;
  the system's own `SanitizeFileName` at
  `RoadSpeedSaveDataSystem.cs#L125-L150` caps at 100 and also strips spaces/dots) - keep
  one canonical sanitizer to avoid the file you write and the file you read diverging.

- **Never hard-code an absolute path.** Anarchy's debug locale export writes to a
  literal developer path,
  `File.WriteAllText("C:\\Users\\TJ\\source\\repos\\Anarchy\\Anarchy\\UI\\src\\lang\\en-US.json", str)`
  (`../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/AnarchyMod.cs#L104`, @a6311e898d20a775368668b234aaa32f06e3e1eb).
  That is inside `#if DEBUG` and is a dev-only export (not save persistence), but it is
  the canonical example of the bug: on any other machine the write throws. Always derive
  the root from `EnvPath.kUserDataPath` (or the LocalLow lookup in Step 1).

- **Guard every read against a corrupt/empty file.** Deserialization can return `null`
  (empty or truncated file) or throw (malformed JSON). Both canonical readers catch and
  fall back to defaults rather than letting the exception escape - Road Speed Adjuster
  null-checks the result (Step 4) and Smooth LHT catches `IOException` and generic
  `Exception` separately, reverting to its default set
  (`../../../vice-and-order-research/mods/dossiers/smooth-left-hand-traffic/repo/SmoothLHT/Services/InvertPreferenceStore.cs#L41-L49`, @37b2850b10ced1cd17457c25778409e57e00a03e).

- **Recover a corrupt file by re-writing defaults, not just null-guarding.** Beyond
  falling back to defaults in memory (previous bullet), the durable fix is to persist
  those defaults immediately so the next run reads clean data instead of re-hitting the
  bad file. Traffic Tool Essentials does this for its `ModSetting` store (the pattern
  applies verbatim to a side-car): its ctor calls `SetDefaults()`, wraps
  `AssetDatabase.global.LoadSettings(...)` in `try/catch`, and on the catch path runs
  `ApplyAndSave()` to write a valid file
  (`../../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Settings.cs#L199-L225`, @10973595ac9ed37f47ca24eb788d4421dc295fa0).

- **Side-car files are NOT deleted with the save.** Deleting a city in-game does not
  remove your `ModsData` file. A per-save file keyed by city name will be re-read (or
  collide) if a new city reuses that name. None of the canonical mods garbage-collect
  orphaned files - factor this in if file count matters. The related hazard is a
  **renamed or reformatted mod inheriting an old file it cannot read.** Traffic Tool
  Essentials (renamed from TrafficLightsEnhancement) does not migrate the old file - it
  **deletes** it on load and starts fresh, because the old format was incompatible and
  caused hotkey issues: `DeleteLegacyCocFile()` locates the prior
  `C2VM-TrafficLightsEnhancement.coc` under LocalLow and calls `File.Delete` inside a
  swallow-all `try/catch`
  (`../../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Mod.cs#L84-L103`, @10973595ac9ed37f47ca24eb788d4421dc295fa0).
  When a clean break is safer than a migration path, delete the legacy file deliberately -
  do not silently deserialize it.

- **`Colossal.Json` and `Newtonsoft.Json` are not the same API.** Do not mix
  `JsonConvert.SerializeObject` with `JSON.Load` on the same file expecting identical
  output; enum handling, formatting, and type coercion differ (Specialized Zones must
  add a `StringEnumConverter` to make Newtonsoft write enums as strings - see
  Variations). Pick one library per file.

## Variations

- **Colossal's built-in `Colossal.Json`, global file, stable string key
  (smooth-left-hand-traffic).** No Newtonsoft dependency. The store anchors on
  `EnvPath.kUserDataPath` and a single fixed filename, and holds a `HashSet<string>` of
  prefab names:

  ```csharp
  storageFolder = Path.Combine(EnvPath.kUserDataPath, "ModsData", "SmoothLHT");
  Directory.CreateDirectory(storageFolder);
  // ...
  NonInvertedAssets = JSON.MakeInto<HashSet<string>>(JSON.Load(File.ReadAllText(path)))
                      ?? new HashSet<string>(defaultNonInvertedAssets);   // Load
  File.WriteAllText(path, JSON.Dump(NonInvertedAssets));                  // Save
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/smooth-left-hand-traffic/repo/SmoothLHT/Services/InvertPreferenceStore.cs#L19-L58` (@37b2850b10ced1cd17457c25778409e57e00a03e)

  `JSON.Load` -> `Variant`, `JSON.MakeInto<T>` -> typed object, `JSON.Dump` -> string.
  Because the key is a **prefab name**, this file is portable across cities and
  sessions.

- **Newtonsoft with a global config file + custom `JsonSerializerSettings`
  (specialized-industrial-zones).** When you serialize enums or risk reference loops,
  configure Newtonsoft explicitly and reuse the settings object for both read and write:

  ```csharp
  private static readonly JsonSerializerSettings JsonSettings = new()
  {
      ReferenceLoopHandling = ReferenceLoopHandling.Ignore,
      Formatting = Formatting.Indented,
      Converters = [new StringEnumConverter()]   // enums as readable strings, not ints
  };
  private static readonly string ZoneFilePath =
      Path.Combine(EnvPath.kUserDataPath, "ModsData", "SpecializedZones", "SpecializedZones.json");
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/specialized-industrial-zones/repo/src/SpecializedIndustryZones/SpecializedZoningSystem.cs#L31-L36,#L187` (@8d9153c3fda72e3b79a133bee6c83f9ba7df8b5a)

  Its `SaveZoneFile` also creates the directory defensively before writing
  (`SpecializedZoningSystem.cs#L208-L221`), so a missing `ModsData` subfolder is not a
  first-run crash.

- **User-editable config with hot-reload.** Because Specialized Zones' global file is a
  human-authored config (default-generated on first run, then editable), the system
  watches the file's last-modified timestamp (`_lastFileModifiedTimestamp`,
  `SpecializedZoningSystem.cs#L38-L39`) and re-loads when it changes - a pattern worth
  adopting when the JSON is meant to be edited by hand rather than only written by the
  mod. Whether the watch fires reliably at runtime is `Needs Verification (in-game)`.

- **Versioned schema with migrate-on-load + re-save (specialized-industrial-zones).**
  Step 3's `Version` field is only useful if something acts on it. Specialized Zones does:
  its file object carries a `Version` string and a `static readonly CurrentVersion`, and
  an `UpgradeIfNeeded()` method mutates old data forward in place and returns `true` when
  it changed anything:

  ```csharp
  public static readonly string CurrentVersion = "v1alpha2";
  public string Version { get; set; } = CurrentVersion;

  public bool UpgradeIfNeeded()
  {
      if (Version == CurrentVersion) return false;
      if (Version == "v1alpha1")
          AddFishToAgricultureZones();   // v1alpha1 -> v1alpha2 data fix-up
      Version = CurrentVersion;
      return true;
  }
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/specialized-industrial-zones/repo/src/SpecializedIndustryZones/SpecializedZoneSpecFile.cs#L9-L26` (@8d9153c3fda72e3b79a133bee6c83f9ba7df8b5a)

  The load path calls it immediately after deserialize and, crucially, **re-saves when the
  upgrade fired** so the migration runs once and the on-disk file is left current:

  ```csharp
  var specs = LoadZoneFile();
  if (specs == null) { /* warn + bail */ }
  if (specs.UpgradeIfNeeded())
  {
      SaveZoneFile(specs);
      _log.Info($"Upgraded specialized zone specs to version {specs.Version}.");
  }
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/specialized-industrial-zones/repo/src/SpecializedIndustryZones/SpecializedZoningSystem.cs#L100-L111` (@8d9153c3fda72e3b79a133bee6c83f9ba7df8b5a)

  This is the side-car analogue of ECS component version-gating: bump `CurrentVersion`,
  add an `if (Version == "old")` branch, and let the next load rewrite the file.

- **Config keyed by the save's session GUID + auto-apply on matching load
  (time-weather-anarchy).** When you want a preference to re-attach itself to *the exact
  save it was set on* (not the city name, which can collide), key the file by the save's
  session GUID. Time Weather Anarchy stores a
  `Dictionary<string, SaveLinkData>` in `save_links.json` under
  `EnvPath.kUserDataPath/ModsSettings/TimeWeatherAnarchy/` (note: `ModsSettings`, not
  `ModsData`), via `Colossal.Json`, and swallows any read error back to an empty map:

  ```csharp
  private static readonly string LinksPath = Path.Combine(
      EnvPath.kUserDataPath, "ModsSettings", nameof(TimeWeatherAnarchy), "save_links.json");

  private static Dictionary<string, SaveLinkData> LoadFile(string path)
  {
      if (!File.Exists(path)) return new Dictionary<string, SaveLinkData>();
      try { return JSON.MakeInto<Dictionary<string, SaveLinkData>>(JSON.Load(File.ReadAllText(path)))
                   ?? new Dictionary<string, SaveLinkData>(); }
      catch { return new Dictionary<string, SaveLinkData>(); }
  }
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/time-weather-anarchy/repo/TimeWeatherAnarchy/TimeWeatherAnarchy/Code/Settings/SaveLinkUtils.cs#L18-L44` (@71128d3958c31b03163a2cba981909655a38bc83)

  Entries are written keyed by the GUID (`AttachSave` does
  `SaveGameLinks[guid] = new SaveLinkData {...}` then re-dumps the file,
  `Setting.cs#L96-L97`). On load, the control system resolves the running save's
  `sessionGuid` and auto-applies the linked profile - the "read back on load" hook, but
  keyed to identity rather than city name:

  ```csharp
  _currentSaveGuid = saveMetadata.target.sessionGuid.ToString();
  Mod.m_Setting.UpdateSaveDisplayName(_currentSaveGuid, _currentSaveName);
  ApplyLinkedProfile(_currentSaveGuid);   // -> GetLinkedProfile(guid), then set SelectedProfile
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/time-weather-anarchy/repo/TimeWeatherAnarchy/TimeWeatherAnarchy/Code/System/TimeAndWeatherControlSystem.cs#L76-L93` (@71128d3958c31b03163a2cba981909655a38bc83)

  A stable GUID is exactly the "durable identity" the Pitfalls section prescribes instead
  of `Entity.Index`. Whether `sessionGuid` stays constant across the live game's save /
  save-as flow is `Needs Verification (in-game)`.

- **When to NOT use a side-car.** If the data is intrinsically tied to specific entities
  and must migrate with the save's entity remapping, an ECS serializable component
  (save-stream) is the correct tool despite the extra ceremony - see
  [ECS serializable save-data](ecs-serializable-savedata.md).

## See also
- Related recipes: [ECS serializable save-data](ecs-serializable-savedata.md) (the
  save-stream alternative and when to prefer it).
- Reference: [Settings, ModSetting vs ECS vs side-car JSON](../../explanation/settings-and-data.md),
  [Serialization](../../explanation/serialization.md).
- Case studies demonstrating it: [anarchy](../../case-studies/anarchy.md).

## Sources
- Canonical mods (dossier + repo):
  - `road-speed-adjuster` @e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed - `repo/Data/PersistentSpeedStorage.cs`, `repo/Data/RoadSpeedData.cs`, `repo/Systems/RoadSpeedSaveDataSystem.cs`, `repo/Systems/RoadSpeedToolSystem.cs`
  - `smooth-left-hand-traffic` @37b2850b10ced1cd17457c25778409e57e00a03e - `repo/SmoothLHT/Services/InvertPreferenceStore.cs`
  - `specialized-industrial-zones` @8d9153c3fda72e3b79a133bee6c83f9ba7df8b5a - `repo/src/SpecializedIndustryZones/SpecializedZoningSystem.cs`, `repo/src/SpecializedIndustryZones/SpecializedZoneSpecFile.cs` (schema versioning + migrate-on-load)
  - `time-weather-anarchy` @71128d3958c31b03163a2cba981909655a38bc83 - nested `repo/TimeWeatherAnarchy/TimeWeatherAnarchy/Code/Settings/SaveLinkUtils.cs`, `.../Setting.cs`, `.../Code/System/TimeAndWeatherControlSystem.cs` (save-GUID-keyed config + auto-apply on load)
  - `traffic-tool-essentials` @10973595ac9ed37f47ca24eb788d4421dc295fa0 - `repo/TrafficToolEssentials/Settings.cs` (corrupt-file recovery), `repo/TrafficToolEssentials/Mod.cs` (legacy `.coc` deletion instead of migration)
  - `anarchy` @a6311e898d20a775368668b234aaa32f06e3e1eb - `repo/Anarchy/AnarchyMod.cs` (cautionary: hard-coded path)
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding
