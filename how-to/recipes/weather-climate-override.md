---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: Weather / climate system override"
recipe: weather-climate-override
technique_family: "AB - Weather / climate system override"
diataxis: how-to
source_version: "1.6.0f1 (time2work-realistic-trips@d42921f; date-pinned, static source only)"
last_reverified: "2026-07-04"
status: source-verified
canonical_mods:
  - time2work-realistic-trips@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39
  - time-weather-anarchy@71128d3958c31b03163a2cba981909655a38bc83
technique_applicability: [simulation]
Created: 2026-07-04
Updated: 2026-07-04
Owners:
  - codex
---

# Weather / climate system override

> Take control of the game's climate sampling - temperature, precipitation,
> cloudiness, aurora, fog, and season - either by replacing `ClimateSystem`'s
> sampling method wholesale (Harmony) or by driving the system's built-in
> `overrideValue`/`overrideState` fields it already exposes.

## Problem
You want the weather or season to be something other than what the vanilla
`ClimateSystem` computes from the map's `ClimatePrefab` curves plus time-of-year:
force permanent summer, pin temperature to a fixed value, mute precipitation, or
recompute the whole climate sample from your own logic. The climate is produced
inside a running simulation system, so you cannot reach it by editing a prefab
field once at load - you have to intercept the value as it is sampled, or push a
live override onto the system every time your settings change.

## Solution
There are **two distinct mechanisms** under this family, and choosing between them
is the whole point of this recipe:

1. **Hard override via Harmony prefix (time2work).** Patch
   `ClimateSystem.SampleClimate(ClimatePrefab, float)` with a Harmony **Prefix** that
   computes the entire `ClimateSample` itself and returns `false` to skip the
   original. This is a total takeover of how climate is sampled - maximal control,
   but you own the whole computation and you are coupled to the method signature.

2. **Cooperative override via the system's own fields (time-weather-anarchy).** No
   Harmony. Resolve `ClimateSystem` as a managed system and set the
   `overrideValue`/`overrideState` pair on each channel (`temperature`, `aurora`,
   `cloudiness`, `precipitation`, `fog`) plus `currentDate` for the season. The game
   already honours these override fields; you are using a supported seam, not
   replacing code.

Reach for mechanism 1 when you must change the *shape* of the sampling (e.g. drive
climate from a different time source). Reach for mechanism 2 when a per-channel
clamp/pin is enough - it is far less brittle.

## Steps & Code

### 1. (Mechanism 1) Register a Harmony patch class at load

time2work bootstraps Harmony once in its mod entry point and patches every
annotated method in the assembly:

```csharp
var harmony = new Harmony(harmonyID);
//Harmony.DEBUG = true;
harmony.PatchAll(typeof(Mod).Assembly);
```
Source: `../../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Mod.cs#L164-L166` (@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39)

### 2. (Mechanism 1) Prefix `SampleClimate`, disambiguating the overload

`ClimateSystem.SampleClimate` is overloaded, so the target is pinned with an explicit
`Type[]` - `new Type[] { typeof(ClimatePrefab), typeof(float) }` - or Harmony cannot
tell which overload you mean. The prefix rebuilds the whole `ClimateSample` from the
prefab's curves and returns `false` to skip the original method entirely:

```csharp
[HarmonyPatch(typeof(ClimateSystem), "SampleClimate", new Type[] { typeof(ClimatePrefab), typeof(float)})]
[HarmonyPrefix]
public static bool ClimateSystemPatches_SampleClimate_Prefix(ClimatePrefab prefab, float t, ref ClimateSample __result, ClimateSystem __instance)
{
    float time = t * 12;
    float num1 = prefab.m_Temperature.Evaluate(time);
    float num2 = prefab.m_Precipitation.Evaluate(time);
    float num3 = prefab.m_Cloudiness.Evaluate(time);
    float num4 = prefab.m_Aurora.Evaluate(time);
    float num5 = prefab.m_Aurora.Evaluate(time);
    __result = new ClimateSystem.ClimateSample()
    {
        temperature = num1, precipitation = num2, cloudiness = num3,
        aurora = num4, fog = num5
    };
    return false;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Patches/Time2WorkPatches.cs#L171-L191` (@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39)

Two things to note in the real code: the result is written through `ref ClimateSample
__result`, and returning `false` is what makes this a **hard override** - the vanilla
body never runs. (See Pitfalls for the `m_Aurora`-into-`fog` line, which is a real
quirk of this source, not a typo introduced here.)

### 3. (Mechanism 2) Resolve `ClimateSystem` as a managed system

time-weather-anarchy takes the cooperative path. It grabs `ClimateSystem` (and the
related `PlanetarySystem`) in `OnCreate` and keeps the reference:

```csharp
protected override void OnCreate()
{
    base.OnCreate();
    _climateSystem = World.GetOrCreateSystemManaged<ClimateSystem>();
    _planetarySystem = World.GetOrCreateSystemManaged<PlanetarySystem>();
    _simulationSystem = World.GetOrCreateSystemManaged<SimulationSystem>();
}
```
Source: `../../../vice-and-order-research/mods/dossiers/time-weather-anarchy/repo/TimeWeatherAnarchy/TimeWeatherAnarchy/Code/System/TimeAndWeatherControlSystem.cs#L38-L44` (@71128d3958c31b03163a2cba981909655a38bc83)

### 4. (Mechanism 2) Drive each channel through its `overrideValue` / `overrideState`

`ClimateSystem` exposes a paired override per channel: `overrideValue` (what to force)
and `overrideState` (whether to force it). Setting both is the entire technique - no
patching. Each channel is gated behind an `IsProfileActive()` check so the override
can be conditionally disabled:

```csharp
public void UpdateTemperature()
{
    var active = IsProfileActive();
    _climateSystem.temperature.overrideValue = active ? Mod.m_Setting.Profile.Temperature : 0;
    _climateSystem.temperature.overrideState = active && Mod.m_Setting.Profile.EnableCustomTemperature;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/time-weather-anarchy/repo/TimeWeatherAnarchy/TimeWeatherAnarchy/Code/System/TimeAndWeatherControlSystem.cs#L160-L165` (@71128d3958c31b03163a2cba981909655a38bc83)

The mod repeats this exact shape for `aurora` (`#L153-L158`), `cloudiness`
(`#L167-L172`), `precipitation` (`#L175-L180`), and `fog` (`#L182-L187`) - same
`active ? value : 0` / `active && EnableCustom...` pattern per channel.

### 5. (Mechanism 2) Force the season via `currentDate`

The season is not a separate channel - it is the point in the year, so it is overridden
through `ClimateSystem.currentDate`. The mod maps a season enum to a normalized
year-fraction and flips `currentDate.overrideState` on:

```csharp
case ((int)WeatherOptions.Spring):
    _climateSystem.currentDate.overrideValue = 0.250f;
    break;
case ((int)WeatherOptions.Summer):
    _climateSystem.currentDate.overrideValue = 0.500f;
    break;
case ((int)WeatherOptions.Fall):
    _climateSystem.currentDate.overrideValue = 0.750f;
    break;
case ((int)WeatherOptions.Winter):
    _climateSystem.currentDate.overrideValue = 1f;
    break;
```
Source: `../../../vice-and-order-research/mods/dossiers/time-weather-anarchy/repo/TimeWeatherAnarchy/TimeWeatherAnarchy/Code/System/TimeAndWeatherControlSystem.cs#L252-L263` (@71128d3958c31b03163a2cba981909655a38bc83)

`overrideState` for the date is set from whether a non-default season is selected
(`_climateSystem.currentDate.overrideState = Mod.m_Setting.Profile.WeatherOption != (int) WeatherOptions.Default;`,
`TimeAndWeatherControlSystem.cs#L194`), and cleared back to `false` when the profile is
inactive (`#L198`).

### 6. (Mechanism 2) Re-apply on load and when settings change

Because these are live fields on a running system, they must be (re)written at the right
moments - not once. The control system is registered on the main loop and re-applies on
game-loaded:

```csharp
updateSystem.UpdateAt<TimeAndWeatherControlSystem>(SystemUpdatePhase.MainLoop);
```
Source: `../../../vice-and-order-research/mods/dossiers/time-weather-anarchy/repo/TimeWeatherAnarchy/TimeWeatherAnarchy/Mod.cs#L37` (@71128d3958c31b03163a2cba981909655a38bc83)

`OnGameLoadingComplete` sets `_isEditor`, bails in the editor, then calls
`UpdateTimeAndWeather()` which fans out to all the `Update*` methods above
(`TimeAndWeatherControlSystem.cs#L54-L65`, `#L97-L122`).

## Pitfalls & gotchas

- **Overloaded target must be disambiguated (mechanism 1).** `SampleClimate` has more
  than one overload; without the explicit `new Type[] { typeof(ClimatePrefab),
  typeof(float) }`, Harmony throws (ambiguous match) or patches the wrong method. This
  is the single most important detail of the Harmony path
  (`Time2WorkPatches.cs#L171`).

- **`return false` is a total takeover.** The prefix returns `false`, so vanilla
  `SampleClimate` never runs and your `__result` is the only source of the climate
  sample. Any vanilla logic in the original body (smoothing, additional channels, future
  patch changes) is discarded. If another mod also prefixes the same method, only one
  "skip original" wins and the interaction is order-dependent - **Needs Verification
  (in-game)**.

- **The source has `fog = m_Aurora` (mechanism 1).** In the canonical code both `num4`
  and `num5` are `prefab.m_Aurora.Evaluate(time)`, and `num5` is assigned to `fog`
  (`Time2WorkPatches.cs#L179-L187`). Whether that is intentional or a bug in the mod,
  do not "correct" it silently when citing - reproduce what the source actually does,
  and derive `fog` from `m_Fog` if you want correct fog.

- **Two override fields, not one (mechanism 2).** Setting `overrideValue` alone does
  nothing - `overrideState` must also be `true` or the game ignores it. time-weather-
  anarchy always writes the pair together (`temperature.overrideValue` +
  `temperature.overrideState`, `#L163-L164`). Forgetting `overrideState` is the classic
  silent no-op here.

- **Overrides are sticky - you must clear them.** These fields live on the persistent
  `ClimateSystem`, so an override stays in force until you set `overrideState = false`.
  time-weather-anarchy explicitly writes `false` when a profile is inactive
  (`UpdateSeason` -> `currentDate.overrideState = false`, `#L198`; each channel writes
  `active && ...`). If you never clear, the override bleeds across save loads / into the
  editor.

- **Editor guard.** time-weather-anarchy checks `_isEditor` and returns early in
  `UpdateWeather`/`UpdateTime` (`#L112`, `#L297`) and skips override work in the map
  editor. Applying climate overrides in the editor is usually wrong; gate on the game
  mode.

- **Whether these overrides visibly change gameplay/rendering is runtime behaviour.**
  The code sets the fields; the resulting on-screen weather and any simulation effects
  are **Needs Verification (in-game)**.

## Variations

- **Season inversion pass (mechanism 2).** time-weather-anarchy also carries an
  *inverted* season map (`SetInvertedSeason`, `TimeAndWeatherControlSystem.cs#L270-L293`)
  and only applies it after a delay once it detects the current season name does not
  match the requested one (`CheckIfInvertSeason` compares
  `_climateSystem.currentSeasonName` against `"SeasonSummer"` etc.,
  `#L205-L242`; applied from `OnUpdate` after `_timeDelay`, `#L315-L339`). Use this when
  a single `currentDate` write lands on the wrong half of the year.

- **Override time-of-day too (mechanism 2).** The same system pins the clock through the
  sibling `PlanetarySystem`: `_planetarySystem.overrideTime`, `.time`, `.dayOfYear`,
  `.latitude`, `.longitude` (`UpdateTime`, `#L295-L313`; `UpdateLatitude`/`Longitude`
  `#L124-L132`). Same "override field" idea, different system - pair it with the climate
  overrides for full time+weather control.

- **Prefix a different time method instead of `SampleClimate` (mechanism 1).** time2work
  prefixes many `TimeSystem` getters with the same `return false` skip-original shape
  (e.g. `GetTimeOfYear`, `Time2WorkPatches.cs#L151-L159`), which is how it feeds a custom
  time source into climate sampling indirectly. If you only need to shift *when* the year
  is sampled, patch the time source rather than climate itself.

## See also
- Explanation: [Harmony patching](../../explanation/harmony-patching.md) (prefix/postfix,
  `return false` skip-original, overload disambiguation).
- Related recipe: [Harmony price-getter postfix](harmony-price-getter-postfix.md)
  (the postfix-adjust-`__result` counterpart to this recipe's prefix-replace).
- Case study demonstrating it: [time2work-realistic-trips](../../case-studies/time2work-realistic-trips.md).

## Sources
- Canonical mods (dossier + repo):
  - `time2work-realistic-trips` @d42921ffb1f6bbbf2c2b086bb17727bf0f637c39 - `repo/NightShift/Patches/Time2WorkPatches.cs`, `repo/NightShift/Mod.cs`
  - `time-weather-anarchy` @71128d3958c31b03163a2cba981909655a38bc83 - `repo/TimeWeatherAnarchy/TimeWeatherAnarchy/Code/System/TimeAndWeatherControlSystem.cs`, `repo/TimeWeatherAnarchy/TimeWeatherAnarchy/Mod.cs`
- Official/community references (link out, do not duplicate): https://cs2.paradoxwikis.com/Modding
