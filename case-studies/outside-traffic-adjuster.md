---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Case study: Outside Traffic Adjuster"
case_study: outside-traffic-adjuster
mod: "Outside Traffic Adjuster"
dossier: ../../vice-and-order-research/mods/dossiers/outside-traffic-adjuster/
repo_commit: 42afd29638267c9dc116b040515ceaf41eb9a9e1
source_version: "~1.5.2f1 (outside-traffic-adjuster@42afd29; date-pinned, static source only)"
last_reverified: "2026-07-04"
diataxis: explanation
techniques: [A, N]
technique_applicability: [simulation, core]
status: source-verified
Created: 2026-07-04
Updated: 2026-07-04
Owners:
  - codex
---

# Outside Traffic Adjuster - case study

> The smallest complete code mod in the handbook: three source files, one
> ECS system, four settings sliders, and a single component field it rewrites.
> Read it to see the prefab-field-override and settings-patterns techniques
> stripped down to their skeleton, with nothing else in the way.

All code claims below are cited to `repo/<path>#Lxx` at pinned commit
`42afd29638267c9dc116b040515ceaf41eb9a9e1` (the head of the clone at pin time,
publish config v1.0.1), surfaced through the dossier at
`../../vice-and-order-research/mods/dossiers/outside-traffic-adjuster/`. The mod
is exactly three C# files: `Mod.cs`, `Setting.cs`, and
`SpawnRateEditorSystem.cs`.

## What it does / why it's instructive

Outside Traffic Adjuster lets a player scale how much traffic enters the map
from outside connections. The game seeds outside-to-outside traffic by attaching
a `TrafficSpawnerData` component (with an `m_SpawnRate` field) to a handful of
marker-object prefabs - one pair each for road and train, one for ship, one for
plane. The mod exposes four sliders (road / train / ship / plane), reads the
vanilla spawn rate baselines, multiplies them, and writes the result back onto
those same prefab components (repo/SpawnRateEditorSystem.cs#L108-L152). Setting a
slider to `0` disables that mode's outside traffic entirely.

It is instructive precisely because it does *nothing else*. There is no Harmony,
no decompiled clone, no Burst job, no custom component, no persistence beyond the
stock settings file. It is the minimal viable shape of two very common
techniques - **prefab field override** (family A) and **settings sliders**
(family N) - wired together by the standard `onSettingsApplied` event idiom. If
you want to learn what the irreducible core of a "tweak a vanilla value from a
slider" mod looks like, this is it. Contrast it with a mod like magic-mail that
*reads* the live prefab value before scaling; Outside Traffic Adjuster instead
hard-codes the vanilla baselines, which is simpler but drift-prone (see Key
decisions).

## Architecture at a glance

One system, one phase. `Mod.OnLoad` builds the settings object, registers it in
the options UI, loads any saved values, and schedules the single system with
`updateSystem.UpdateAt<SpawnRateEditorSystem>(SystemUpdatePhase.ModificationEnd)`
(repo/Mod.cs#L24-L29). `ModificationEnd` is the phase where prefab-side edits
are expected to land, matching the prefab-mutation pattern.

`SpawnRateEditorSystem` is a plain `GameSystemBase` that is **normally disabled**.
In `OnCreate` it grabs the `PrefabSystem`, seeds its multiplier fields to the
vanilla baseline (`1.0`), subscribes to `onSettingsApplied`, and immediately sets
`this.Enabled = false` (repo/SpawnRateEditorSystem.cs#L17-L30). `OnUpdate` does
nothing but disable itself again (repo/SpawnRateEditorSystem.cs#L32-L35), so the
system never spins on the simulation loop - it is a passive host for two event
hooks:

- **Game load** - `OnGameLoadingComplete` applies the player's saved slider
  values when entering an actual game (`GameMode.Game`), and otherwise resets the
  multipliers to `1.0` and reapplies - so leaving a save restores vanilla spawn
  rates rather than leaking the last game's edits into the main menu / editor
  (repo/SpawnRateEditorSystem.cs#L37-L50).
- **Settings applied** - the `onSettingsApplied` handler filters to the mod's own
  `Setting` type and calls `UpdateAndApplyMultipliers`, which reapplies only the
  modes whose slider actually changed (repo/SpawnRateEditorSystem.cs#L22-L28,
  repo/SpawnRateEditorSystem.cs#L68-L90).

The write path is uniform. `TryGetPrefab` looks up a prefab by
`new PrefabID(prefabType, prefabName)` and resolves both the `PrefabBase` and its
entity (repo/SpawnRateEditorSystem.cs#L100-L106). Each `Apply*SpawnRates` method
then guards on both the lookup **and** `TryGetComponentData<TrafficSpawnerData>`,
mutates `m_SpawnRate`, and re-adds the component
(repo/SpawnRateEditorSystem.cs#L108-L120):

```csharp
private void ApplyRoadSpawnRates()
{
    if (TryGetPrefab(nameof(MarkerObjectPrefab), "Road Outside Connection - Oneway", out PrefabBase prefabBase, out Entity entity) && m_PrefabSystem.TryGetComponentData<TrafficSpawnerData>(prefabBase, out TrafficSpawnerData comp))
    {
        comp.m_SpawnRate = 0.3f * roadMultiplier;
        m_PrefabSystem.AddComponentData(prefabBase, comp);
    }
    // ... "Road Outside Connection - Twoway" is handled identically
}
```

`SpawnRateEditorSystem` is the **sole writer** of `TrafficSpawnerData` in the
mod - there is exactly one read-modify-write path, replicated per mode with
different prefab names and baselines.

### The hard-coded baselines

The vanilla spawn rates are literal constants in the multiply expressions, one
per mode (repo/SpawnRateEditorSystem.cs#L108-L152):

- Road (one-way + two-way markers): `0.3f * roadMultiplier`
  (repo/SpawnRateEditorSystem.cs#L112, repo/SpawnRateEditorSystem.cs#L117).
- Train (one-way + two-way markers): `0.003f * trainMultiplier`
  (repo/SpawnRateEditorSystem.cs#L126, repo/SpawnRateEditorSystem.cs#L131).
- Ship (two-way marker only): `0.002f * shipMultiplier`
  (repo/SpawnRateEditorSystem.cs#L140).
- Plane (single marker): `0.005f * planeMultiplier`
  (repo/SpawnRateEditorSystem.cs#L149).

A multiplier of `1.0` therefore restores the exact vanilla rate, and the sliders
run `0..2` in `0.1` steps (repo/Setting.cs#L17-L27).

## Techniques demonstrated

- [Prefab field override](../how-to/recipes/prefab-field-override.md) (family A) -
  the canonical `TryGetPrefab` -> `TryGetComponentData<T>` -> mutate field ->
  `AddComponentData` cycle, run against `TrafficSpawnerData.m_SpawnRate` on the
  outside-connection marker prefabs
  (repo/SpawnRateEditorSystem.cs#L100-L120,
  repo/SpawnRateEditorSystem.cs#L136-L152). Re-adding the mutated struct via
  `AddComponentData` is how the edit is committed back to the prefab entity.
- [Settings sliders / options UI](../how-to/recipes/settings-patterns.md)
  (family N) - a `ModSetting` subclass with a `[FileLocation]` attribute and
  exactly four `[SettingsUISlider(min = 0, max = 2, step = 0.1f)]` float
  properties, defaulted to `1` in `SetDefaults`, plus a `LocaleEN`
  `IDictionarySource` supplying the labels/descriptions
  (repo/Setting.cs#L9-L35, repo/Setting.cs#L17-L27). `Mod.OnLoad` wires it up with
  `RegisterInOptionsUI` + `AddSource` + `AssetDatabase.global.LoadSettings`
  (repo/Mod.cs#L24-L27).

The glue between the two families is the `onSettingsApplied` event: the system
subscribes in `OnCreate` and reapplies changed multipliers when the player edits
a slider (repo/SpawnRateEditorSystem.cs#L22-L28,
repo/SpawnRateEditorSystem.cs#L68-L90), so edits take effect live without a save
restart.

## Key decisions & tradeoffs

- **Hard-coded vanilla baselines vs reading the prefab.** The mod embeds
  `0.3 / 0.003 / 0.002 / 0.005` directly in the multiply expressions
  (repo/SpawnRateEditorSystem.cs#L112-L149). This is maximally simple - no need to
  capture an "original" value or reason about apply-order - but it is **drift
  prone**: if Colossal retunes any outside-connection spawn rate in a future
  patch, a slider of `1.0` will silently reset it to the mod's stale constant
  instead of the new vanilla value. A mod that instead *reads* the current
  `m_SpawnRate` before scaling (the magic-mail approach) would track vanilla
  changes automatically at the cost of more bookkeeping.
- **Passive, self-disabling system.** Rather than run every tick, the system
  disables itself in `OnCreate` and `OnUpdate` and only does work from two event
  callbacks (`OnGameLoadingComplete`, `onSettingsApplied`)
  (repo/SpawnRateEditorSystem.cs#L29-L50). This is the cheapest possible way to
  "apply on change" without polling the simulation loop.
- **Reset-on-leave.** `OnGameLoadingComplete` resets multipliers to `1.0`
  whenever the mode is not `GameMode.Game`
  (repo/SpawnRateEditorSystem.cs#L45-L49), so the edit does not persist into the
  editor or a subsequent save that the player has not opted into. Because the mod
  mutates shared prefab entities (not per-save data), this reset is what keeps the
  change scoped to the intended game session.
- **Per-mode granular reapply.** `UpdateAndApplyMultipliers` compares each slider
  against the cached value and only reapplies the modes that changed
  (repo/SpawnRateEditorSystem.cs#L68-L90) - a micro-optimization that also avoids
  redundant `AddComponentData` calls on unrelated prefabs.
- **No dependencies.** No Harmony, no companion-mod interop, no custom
  components; the entire behaviour rides stock `PrefabSystem` APIs and the stock
  settings pipeline.

## Pitfalls / upstream-watch

- **Silent prefab miss.** Every write is guarded by
  `TryGetPrefab(...) && TryGetComponentData<TrafficSpawnerData>(...)`
  (repo/SpawnRateEditorSystem.cs#L110, repo/SpawnRateEditorSystem.cs#L124). If
  Colossal renames a marker prefab (the names are string literals like
  `"Road Outside Connection - Oneway"`) or removes `TrafficSpawnerData` from it,
  the guard fails, the branch is skipped, and the slider silently stops working
  for that mode with no error surfaced - `SetShowsErrorsInUI(false)` is set on the
  logger (repo/Mod.cs#L12).
- **Baseline drift across patches.** As above, the hard-coded
  `0.3 / 0.003 / 0.002 / 0.005` constants are the maintenance liability: they must
  be re-checked against `Game.dll` after each CS2 update, or a "neutral" slider of
  `1.0` will diverge from actual vanilla (repo/SpawnRateEditorSystem.cs#L112-L149).
- **Shared-prefab mutation.** Because the edit lands on shared prefab entities
  rather than save data, another mod that also writes `TrafficSpawnerData` on the
  same markers would collide; the last writer wins. Not observed here (no other
  writer exists in this mod), but relevant when stacking traffic mods.

Needs Verification (in-game): the real-world magnitude of the traffic change per
slider step, and whether `ModificationEnd` scheduling is early enough that the
first `OnGameLoadingComplete` apply beats any vanilla system that consumes
`m_SpawnRate` on the same load - both require the running game to confirm.

## Source pointers

- Dossier: `../../vice-and-order-research/mods/dossiers/outside-traffic-adjuster/`
  (index/source/modding/guide + notes).
- Repo @ `42afd29638267c9dc116b040515ceaf41eb9a9e1`, all three files:
  - `repo/Mod.cs` - `OnLoad` settings wiring + single-system schedule into
    `ModificationEnd` (repo/Mod.cs#L24-L29).
  - `repo/Setting.cs` - four `[SettingsUISlider]` floats + `LocaleEN`
    (repo/Setting.cs#L17-L27).
  - `repo/SpawnRateEditorSystem.cs` - the sole `TrafficSpawnerData` writer:
    `TryGetPrefab` -> `TryGetComponentData` -> mutate `m_SpawnRate` ->
    `AddComponentData`, with hard-coded baselines and event-driven apply
    (repo/SpawnRateEditorSystem.cs#L100-L152).
