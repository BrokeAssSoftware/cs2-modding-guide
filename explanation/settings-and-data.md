---
FrontmatterVersion: 1
DocumentType: Guide
Title: Settings and Data - Where State Lives
Summary: The three places a CS2 mod can keep state - a ModSetting settings file, ECS data serialized inside the save via ISerializable, or a side-car JSON file next to the save - and how to decide which one a given piece of state belongs in, explained against real mod source.
diataxis: explanation
source_version: "~1.5.2f1 (road-speed-adjuster@e0c0c0b; date-pinned, static source only)"
last_reverified: "2026-07-04"
canonical_mods:
  - road-speed-adjuster@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed
  - magic-mail@6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47
status: source-verified
Created: 2026-07-03
Updated: 2026-07-03
Owners:
  - codex
References:
  - Label: Recipe - Settings UI patterns
    Path: ../how-to/recipes/settings-patterns.md
  - Label: Save/Load Serialization (the ECS save-stream mechanism)
    Path: ./serialization.md
  - Label: SettingsUI attribute reference
    Path: ../reference/options-attributes.md
  - Label: Technique Index
    Path: ../technique-index.md
---

# Settings and Data - Where State Lives

Every stateful mod eventually has to answer: *where does this value live between sessions?*
Cities: Skylines II gives a mod three distinct stores, and choosing the wrong one is a common
source of bugs - settings that should have been per-save, or per-save data dumped into a global
settings file. This page draws the boundary between the three and gives a rule for picking.

It is a concept page - the "which store, and why" - not a step-by-step recipe. For building the
settings UI, see [Settings UI patterns](../how-to/recipes/settings-patterns.md). For the ECS
save-stream mechanism, see [Save/Load Serialization](./serialization.md).

## The three stores at a glance

| Store | Mechanism | Scope | Lifetime / keying | Use it for |
| --- | --- | --- | --- | --- |
| **Settings** | `ModSetting` subclass + `[FileLocation]` | Global (one file for the mod) | Persists across all saves; not tied to any city | User *preferences* and mod configuration |
| **ECS save data** | `ISerializable` / `IDefaultSerializable` in the save stream | Per-save | Travels *inside* the `.cok` save file; loads with that city | Authoritative *world state* your mod adds |
| **Side-car file** | Your own JSON on disk (e.g. `ModsData/<Mod>`) | Per-save or global (you choose the key) | A separate file you read/write yourself | Per-save data you cannot or prefer not to put in the save stream |

The one-line decision:

- **Is it a user preference?** -> Settings.
- **Is it authoritative world state that must reload exactly with a specific city?** -> ECS save
  data (serialize it into the save stream).
- **Is it per-save data you cannot fit in the save stream** (large, external, or you want it
  outside the save file)? -> a side-car file keyed by the save.
- **Is it a derived cache you can recompute?** -> none of the above; rebuild it at load, persist
  nothing.

## Store 1: Settings (user preferences)

Settings are for choices *the player* makes about how the mod behaves - a multiplier, a toggle,
a display unit. They belong in one global file that is the same regardless of which city is
loaded. You get this by subclassing `Game.Settings.ModSetting` and tagging it with
`[FileLocation(...)]`, which names the settings file under the user's `ModsSettings` tree. Magic
Mail's settings class is the shape
([`magic-mail` `repo/Settings/Settings.cs#L26,#L43`](../../vice-and-order-research/mods/dossiers/magic-mail/repo/Settings/Settings.cs), commit `6fb3d2b`):

```csharp
[FileLocation("ModsSettings/MagicMail/MagicMail")]
public sealed class Setting : ModSetting
{
    // properties decorated with [SettingsUI*] become Options-panel controls
}
```

The game serializes this file for you and restores it at load (`AssetDatabase.global.LoadSettings`),
and `ApplyAndSave()` writes changes back. You never open the file yourself. Because it is global,
**do not** put per-city state here: a value that should differ between two saves cannot live in
a store that has exactly one copy for the whole mod. For the full authoring model - controls,
defaults, presets, conditional visibility, persistence, `Apply()` - see
[Settings UI patterns](../how-to/recipes/settings-patterns.md) and the attribute lookup in
[SettingsUI attribute reference](../reference/options-attributes.md).

## Store 2: ECS save data (authoritative world state)

State your mod *adds to the world* - a named road segment, a per-citizen schedule, a custom
per-entity payload - belongs *with the city it describes*, so it must reload exactly when that
city loads and never leak into a different save. That is what the save stream is for. You opt a
type into it by implementing a serialization interface, and the game runs your `Serialize` /
`Deserialize` during the dedicated `Serialize` / `Deserialize` scheduler phases:

- **`ISerializable` on an ECS component** (`IComponentData`) - per-entity data serialized with
  its entity. Time2Work's `CitizenSchedule` is a component that persists its per-citizen plan
  this way.
- **`IDefaultSerializable` on a system** (`GameSystemBase`) - one blob for the whole mod, for a
  parallel data structure that is not naturally one-component-per-entity. Advanced Road Naming
  persists its entire segment-name database from a single system this way.

Both write a leading **version int** and read fields back in the exact write order, guarding on
the version so old saves still load after a schema change. The full mechanism, with cited code
and the versioning rationale, is its own page:
[Save/Load Serialization](./serialization.md). The rule here is just *which* store: if losing
the data would corrupt or de-sync a specific city, it is world state, and it goes in the save
stream - not in settings, and not in a loose file that can go missing.

## Store 3: Side-car JSON (per-save data outside the save file)

Sometimes per-save data should live *next to* the save rather than inside it - because it is
large, because it is easier to inspect/share as plain JSON, or because you want it to survive
independently of the save's own serialization. The pattern is a file you manage yourself, keyed
so it maps back to the right save. Road Speed Adjuster stores per-map custom speed limits as a
JSON file per save game, under the game's `ModsData` directory
([`road-speed-adjuster` `repo/Data/PersistentSpeedStorage.cs#L29-L30`](../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/Data/PersistentSpeedStorage.cs), commit `e0c0c0b3`):

```csharp
string localLowPath = Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData) + "Low";
_baseDirectory = Path.Combine(localLowPath, "Colossal Order", "Cities Skylines II", "ModsData", "RoadSpeedAdjuster");
```

It keys the filename by save-game id so each city gets its own file, creating it on first use and
loading it if present ([`#L52-L58`](../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/Data/PersistentSpeedStorage.cs)):

```csharp
string fileName = $"{SanitizeFileName(saveGameName)}_{saveGameId}.json";
_currentFilePath = Path.Combine(BaseDirectory, fileName);
if (File.Exists(_currentFilePath)) LoadFromFile();
```

Read and write are plain serialize/deserialize to that path (here via Newtonsoft
`JsonConvert`), with the whole thing wrapped in try/catch so a bad file degrades gracefully
([`#L177-L178,#L195-L196`](../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/Data/PersistentSpeedStorage.cs)):

```csharp
string json = JsonConvert.SerializeObject(_currentMapData, Formatting.Indented);
File.WriteAllText(_currentFilePath, json);
// load:
_currentMapData = JsonConvert.DeserializeObject<MapSpeedData>(File.ReadAllText(_currentFilePath));
```

The trade-off versus Store 2: a side-car file is easy to author and inspect but is **not**
atomic with the save. If the player copies or shares only the `.cok` file, the side-car is left
behind; if the two get out of sync, your mod must cope. Keying by save-game id (as above)
mitigates mismatch but does not eliminate the "save moved without its side-car" failure. Choose
the side-car when the data is genuinely external or too large for the stream; choose the save
stream when the data *is* the city.

## Derived data: persist nothing

The fourth answer is often "none of these". Caches, indexes, and analytics you can recompute
from authoritative state should be **rebuilt at load**, not persisted. Serializing derived data
bloats whichever store you put it in and creates a second source of truth that can disagree with
the first. Persist the authoritative minimum; regenerate the rest. (This is why the serialization
page's guidance is "never serialise what you can rebuild" - the same principle applies to
settings and side-car files.)

## Adjacent concerns

Two related pieces of mod state are handled by their own subsystems rather than any of the three
stores above:

- **Localization** (control labels, enum names, tooltips) comes from an `IDictionarySource`
  registered per language, *not* from the settings file. A settings control with no matching
  dictionary entry renders its raw locale-ID key. Register the settings object before the locale
  sources so labels resolve. See [Settings UI patterns](../how-to/recipes/settings-patterns.md).
- **Logging** is diagnostics, not persisted state. Use a single logger per mod and keep verbose
  output behind a debug flag rather than writing logs into any of these data stores.

## Common pitfalls

- **Per-save data in the global settings file.** Settings has exactly one copy for the mod; a
  value that must differ between two cities cannot live there. Use the save stream or a keyed
  side-car.
- **World state only in a side-car.** If the data *is* the city, prefer the save stream so it
  travels atomically with the save; a side-car can be separated from its save.
- **Serializing derived caches.** Persist authoritative state only; rebuild caches at load.
- **Unkeyed side-car files.** A side-car not keyed to a specific save will bleed one city's data
  into another. Key by save-game id (Road Speed Adjuster does).
- **Assuming a side-car is atomic.** It is a separate file; handle the missing/out-of-date case.

## See also

- Recipe: [Settings UI patterns](../how-to/recipes/settings-patterns.md) - building the settings
  store's UI (Store 1).
- [Save/Load Serialization](./serialization.md) - the ECS save-stream mechanism and versioning
  (Store 2).
- Reference: [SettingsUI attribute reference](../reference/options-attributes.md) - the attribute
  lookup for the settings store.
- [Technique Index](../technique-index.md) - families E (ECS ISerializable save data), F
  (external JSON side-car persistence), N (SettingsUI).
