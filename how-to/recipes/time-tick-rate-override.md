---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: Time/calendar tick-rate override (visual time dilation)"
recipe: time-tick-rate-override
technique_family: "AL - Time/calendar tick-rate override (visual time dilation)"
diataxis: how-to
source_version: "1.6.0f1 (time2work-realistic-trips@d42921f; date-pinned, static source only)"
last_reverified: "2026-07-05"
status: source-verified
canonical_mods:
  - time2work-realistic-trips@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39
technique_applicability: [simulation]
Created: 2026-07-05
Updated: 2026-07-05
Owners:
  - codex
---

# Time/calendar tick-rate override (visual time dilation)

> Make in-game clock and calendar time pass faster or slower by redefining how many
> simulation ticks make up a day/year - WITHOUT touching the simulation-speed
> multiplier, so pathfinding, aging, and economy keep running at their normal rate.

## Problem
You want a day (or a year) to *feel* longer or shorter - citizens should reach work
at plausible clock times, a "month" should span more than a couple of minutes - but
you must NOT slow the actual simulation. Slowing `SimulationSystem` speed dilates
*everything* (physics, jobs, spawning) and is notoriously brittle; you only want the
mapping from `frameIndex` to "what the clock reads" to change. This is a fundamentally
different axis from simulation speed (see [simulation speed control](simulation-speed-control.md))
and from weather/climate cadence (see [weather/climate override](weather-climate-override.md)).

## Solution
CS2's vanilla `TimeSystem` derives the clock from a single constant,
`TimeSystem.kTicksPerDay`: time-of-day is `ticks % kTicksPerDay / kTicksPerDay`, and
the year adds `m_DaysPerYear` on top. time2work reimplements those getters in its own
`Time2WorkTimeSystem` against a *rescaled* `kTicksPerDay` (a factor times the vanilla
constant), rescales `TimeSettingsData.m_DaysPerYear` for the calendar, then uses
Harmony to make vanilla's time getters return the mod's values and to write the mod's
`m_Time`/`m_Date`/`m_Year` back into the real `TimeSystem` each frame. The simulation
frame counter is never touched - only the *interpretation* of it. That is why the
mod's own lesson holds: visual time scaling is far less brittle than simulation-speed
multipliers.

## Steps & Code

### 1. Rescale the ticks-per-day constant from the vanilla baseline

Compute your own `kTicksPerDay` as `floor(factor * TimeSystem.kTicksPerDay)`. The
vanilla `TimeSystem.kTicksPerDay` is the immutable baseline, so re-applying never
stacks. When the factor is `1f`, fall through to the vanilla value unchanged:

```csharp
public static int kTicksPerDay;
public static float timeReductionFactor;
// ...
if (Mod.m_Setting.slow_time_factor != 1f)
{
    timeReductionFactor = Mod.m_Setting.slow_time_factor;
    kTicksPerDay = (int)Math.Floor(timeReductionFactor * TimeSystem.kTicksPerDay);
}
else
{
    kTicksPerDay = TimeSystem.kTicksPerDay;
    timeReductionFactor = 1f;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Systems/Time2WorkTimeSystem.cs#L23-L50` (@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39)

### 2. Reimplement the time getters against your rescaled constant

Every clock read divides by *your* `kTicksPerDay`, not the vanilla one. This is the
whole dilation: the frame index is untouched, but a larger `kTicksPerDay` means each
frame advances a smaller fraction of the day:

```csharp
protected float GetTimeOfDay(TimeSettingsData settings, TimeData data)
{
    return (float)(this.GetTicks(settings, data) % kTicksPerDay) / kTicksPerDay;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Systems/Time2WorkTimeSystem.cs#L79-L82` (@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39)

### 3. Rescale the calendar (`m_DaysPerYear`) from a cached base value

The year length lives on the `TimeSettingsData` prefab component. A dedicated system
caches the ORIGINAL value per entity, then writes `base * factor` back - caching the
base is what keeps re-application non-stacking:

```csharp
if (!_baseTimeSettingsData.TryGetValue(tsd, out var baseData))
{
    baseData = EntityManager.GetComponentData<TimeSettingsData>(tsd);
    _baseTimeSettingsData[tsd] = baseData;
}
var updatedData = baseData;
int factor = Math.Max(Mod.m_Setting.daysPerMonth, 1);
updatedData.m_DaysPerYear = Math.Max(baseData.m_DaysPerYear * factor, 1);
EntityManager.SetComponentData(tsd, updatedData);
```
Source: `../../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Systems/TimeSettingsMultiplierSystem.cs#L43-L56` (@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39)

### 4. Make vanilla's time getters return your values (Harmony Prefix -> false)

Roughly eleven `TimeSystem`/`TimeUISystem` getters are prefixed to delegate to the
mod's system and `return false` so the original body never runs. One representative
(the pattern repeats for `GetYear` x2, `GetDay`, `GetTicks`, `GetCurrentDateTime`,
`GetStartingDate`, `GetElapsedYears`, `GetTimeOfYear`, `GetTimeOfDay`):

```csharp
[HarmonyPatch(typeof(TimeSystem), "get_normalizedDate")]
[HarmonyPrefix]
static bool TimeSystemPatches_normalizedDate(ref float __result)
{
    Time2WorkTimeSystem t2wTimeSystem = World.DefaultGameObjectInjectionWorld
        .GetOrCreateSystemManaged<Time2WorkTimeSystem>();
    __result = t2wTimeSystem.normalizedDate;
    return false; // Skip original getter
}
```
Source: `../../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Patches/Time2WorkPatches.cs#L59-L169` (@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39)

### 5. Write your time back into the real `TimeSystem` each frame (Postfix + Traverse)

Some consumers read `TimeSystem`'s private backing fields directly rather than the
getters, so a Postfix on `TimeSystem.OnUpdate` overwrites `m_Time`, `m_Date`,
`m_Year` with the mod's values via Harmony `Traverse`:

```csharp
[HarmonyPatch(typeof(TimeSystem), "OnUpdate")]
[HarmonyPostfix]
public static void TimeSystemPatches_OnUpdate_Postfix(TimeSystem __instance)
{
    Time2WorkTimeSystem t2wTimeSystem = World.DefaultGameObjectInjectionWorld
        .GetOrCreateSystemManaged<Time2WorkTimeSystem>();
    Traverse.Create(__instance).Field("m_Time").SetValue(t2wTimeSystem.normalizedTime);
    Traverse.Create(__instance).Field("m_Date").SetValue(t2wTimeSystem.normalizedDate);
    Traverse.Create(__instance).Field("m_Year").SetValue(t2wTimeSystem.year);
}
```
Source: `../../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Patches/Time2WorkPatches.cs#L34-L42` (@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39)

### 6. Re-enable the calendar system on settings-apply (the resync)

The calendar rescaler self-disables after one pass (see the gotcha below). To pick up
a changed `daysPerMonth`/`slow_time_factor` you must re-enable it - the mod does this
from `Setting.Apply()`:

```csharp
var timeSettingsMultiplierSystem = world
    .GetExistingSystemManaged<Time2Work.Systems.TimeSettingsMultiplierSystem>();
if (timeSettingsMultiplierSystem != null)
    timeSettingsMultiplierSystem.Enabled = true;
```
Source: `../../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Setting.cs#L297-L299` (@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39)

## Pitfalls & gotchas

- **The calendar rescaler runs ONCE per enable, then disables itself.** After writing
  `m_DaysPerYear`, `TimeSettingsMultiplierSystem.OnUpdate` sets `Enabled = false`
  (`.../TimeSettingsMultiplierSystem.cs#L59-L60`). A changed `daysPerMonth` (or
  `slow_time_factor`) will NOT take effect until the system is re-enabled - the mod
  re-enables it from `Setting.Apply()` (Step 6). If you build the same pattern and
  forget the re-enable, your first value sticks and later changes silently do nothing
  until a reload/apply resync.

- **`kTicksPerDay` is captured in `OnCreate` from `slow_time_factor`.** The tick
  constant is computed once at system creation
  (`.../Time2WorkTimeSystem.cs#L42-L50`); it is a static field, not recomputed per
  frame. Treat a `slow_time_factor` change as needing a fresh
  `Time2WorkTimeSystem`/session, not just an in-place settings poke -
  `Needs Verification (in-game)` whether an apply mid-session reruns this `OnCreate`.

- **You are overriding getters app-wide - miss one and it desyncs.** The technique
  only works because ~11 getters plus the `OnUpdate` field write-back are ALL patched
  (`.../Time2WorkPatches.cs#L34-L169`). Any vanilla or third-party consumer that reads
  a time surface you did NOT patch will see the unscaled clock and disagree with the
  patched UI. When CS2 patches add/rename a `TimeSystem` getter, this override needs
  re-verification.

- **Derive from the immutable baseline, never the live value.** Both halves cache a
  base and multiply it: ticks use `TimeSystem.kTicksPerDay` (the vanilla constant),
  the calendar caches `baseData.m_DaysPerYear` per entity before scaling. Reading the
  *current* `m_DaysPerYear` and multiplying would compound on every apply.

- **This is NOT simulation-speed control.** Changing `kTicksPerDay` changes only how
  `frameIndex` maps to the clock; the simulation still advances one frame per frame.
  Do not conflate it with `SimulationSystem` speed - keep the two mechanisms separate
  (see [simulation speed control](simulation-speed-control.md)).

- **Distinct from weather/climate override (family AB).** Retiming the day does not by
  itself retime climate; time2work additionally patches `ClimateSystem.SampleClimate`
  to keep weather aligned to its clock (`.../Time2WorkPatches.cs#L171-L191`). If you
  only override the tick rate, verify whether climate/lighting still tracks it -
  `Needs Verification (in-game)`.

## Variations

- **Slow OR speed the clock with one factor.** The factor is just a float multiplier
  on `TimeSystem.kTicksPerDay`; a factor above 1 lengthens the day, below 1 shortens
  it, and exactly `1f` short-circuits to vanilla
  (`.../Time2WorkTimeSystem.cs#L42-L50`). The exact perceived direction/feel is
  `Needs Verification (in-game)`.

- **Retime the day only, or the calendar only.** The two mechanisms are independent:
  override `kTicksPerDay` (Steps 1-2, 4-5) to change day length without touching year
  length, or rescale `m_DaysPerYear` (Step 3) to change month/year length without
  changing the day. time2work applies both, but either stands alone.

- **Field write-back vs. pure getter override.** If a consumer reads `TimeSystem`'s
  getters you can stop at Step 4 (Prefix -> `return false`); the `OnUpdate` Postfix
  (Step 5) is only needed because some consumers read the private `m_Time`/`m_Date`/
  `m_Year` fields directly. Use `Traverse.Create(__instance).Field(name).SetValue(...)`
  to reach those private fields.

## See also
- Related recipes: [simulation speed control](simulation-speed-control.md) (family AV -
  the other, more brittle time axis), [weather/climate override](weather-climate-override.md)
  (family AB - retiming climate cadence).
- Reference: [Harmony patching](../../explanation/harmony-patching.md) (Prefix ->
  `return false`, Postfix, and `Traverse` for private fields).
- Case study demonstrating it: [time2work-realistic-trips](../../case-studies/time2work-realistic-trips.md).

## Sources
- Canonical mods (dossier + repo):
  - `time2work-realistic-trips` @d42921ffb1f6bbbf2c2b086bb17727bf0f637c39 -
    `repo/NightShift/Systems/Time2WorkTimeSystem.cs`,
    `repo/NightShift/Systems/TimeSettingsMultiplierSystem.cs`,
    `repo/NightShift/Patches/Time2WorkPatches.cs`,
    `repo/NightShift/Setting.cs`
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding
