---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Case study: Realistic Trips (Time2Work)"
case_study: time2work-realistic-trips
mod: "Realistic Trips (Time2Work) (77171)"
dossier: ../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/
repo_commit: d42921ffb1f6bbbf2c2b086bb17727bf0f637c39
source_version: "n/a - no pinned source"
last_reverified: "2026-07-04"
diataxis: explanation
techniques: [E, G, C, S, AB]
technique_applicability: [simulation, economy, ui]
status: source-verified
Created: 2026-07-03
Updated: 2026-07-03
Owners:
  - codex
---

# Realistic Trips (Time2Work) - case study

> Realistic Trips disables eleven vanilla simulation and UI systems and replaces them
> with a weekly-calendar pipeline: per-citizen `ISerializable` schedules, three
> reflection/static bridges to peer mods, a wall of Harmony full-getter replacements that
> drive "visual time dilation," a Burst `IJobChunk` that biases pathfinding toward active
> events, and a full `ClimateSystem` override. It is the maximal example of a full-stack
> simulation mod and every interop technique one needs around such a swap.

All code claims below are cited to `repo/<path>#Lxx` at pinned commit
`d42921ffb1f6bbbf2c2b086bb17727bf0f637c39` (branch `master`, snapshot 2026-06-25,
PublishConfiguration ModVersion 2.9.1 / GameVersion 1.6.*), surfaced through the dossier
at `../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/`. The source
tree uses the `NightShift/` project folder. That dossier corrected a prior fabrication -
a `Time2WorkDiagnosticsSystem` writing `ModsData/Time2Work/diagnostics/schedule-
telemetry.csv`, plus an `enable_diagnostics` toggle - **none of which exist** at this
commit. There is no CSV telemetry exporter; the only real diagnostics are opt-in log
toggles. That fabrication is not reintroduced here.

## What it does / why it's instructive

Realistic Trips (author ruzbeh0, Paradox ModId 77171; the code namespace and assembly are
`Time2Work`) rebuilds how citizens spend a day. It gives each cim a work/lunch/day-off
schedule drawn from country-preset probabilities, lengthens the visible day without
speeding the simulation, adds shift work and remote work, schedules special events, and
reworks tourism - so every trip decision flows through one weekly calendar. It ships 20
presets (Balanced default, Performance, and 18 country bundles) and declares I18n
Everywhere (75426) and Custom Chirps (121812) as dependencies.

It is instructive as the maximal **system-replacement** case, and as a catalog of the
interop techniques a serious simulation mod needs around such a swap:

1. **Disable vanilla, schedule bespoke replacements.** Eleven vanilla systems are hard-
   disabled and ~30 Time2Work systems are scheduled in their place across several phases.
2. **Serialize per-agent state.** A versioned `ISerializable CitizenSchedule` component
   persists each cim's plan across save/reload with a legacy-tolerant fallback.
3. **Bridge to peer mods without hard references.** Three bridges - two outbound
   reflection bridges (Custom Chirps, Social Trips) and one inbound static surface
   (Elections) - integrate optional partners that may be absent.
4. **Reserve Harmony for getters ECS cannot own.** A wall of `TimeSystem` getter
   replacements is the real mechanism behind visual time dilation; an economy-cost
   postfix applies a night discount; a `ClimateSystem` override scales the seasons.
5. **Bias pathfinding with a Burst job.** A prefix replaces only the leisure branch of
   `PathfindSetupSystem.FindTargets` with a Burst `IJobChunk` that discounts active-event
   venues.

## Architecture at a glance

`Mod.OnLoad` scans loaded mods (setting `realLifePresent` / `realisticPathFindingPresent`),
registers en-US/pt-BR localization, loads settings, then disables the vanilla systems
(`repo/NightShift/Mod.cs#L67-L110`):

```csharp
World...GetOrCreateSystemManaged<Game.Simulation.CitizenBehaviorSystem>().Enabled = false;
World...GetOrCreateSystemManaged<Game.Simulation.CitizenTravelPurposeSystem>().Enabled = false;
World...GetOrCreateSystemManaged<Game.Simulation.WorkerSystem>().Enabled = false;
World...GetOrCreateSystemManaged<Game.Simulation.LeisureSystem>().Enabled = false;
// ... StudentSystem, TourismSystem, TouristSpawnSystem, AttractionSystem,
//     BuyingCompanySystem, TimeUISystem, StatisticsUISystem
if (!realLifePresent)
    World...GetOrCreateSystemManaged<Game.Simulation.DeathCheckSystem>().Enabled = false;
```
(`repo/NightShift/Mod.cs#L94-L110`)

It then schedules ~30 systems, mostly into `GameSimulation`, with the timekeeper also in
`EditorSimulation` and `Deserialize`, the UI systems in `UIUpdate`, and three parameter
updaters around the prefab phases (`repo/NightShift/Mod.cs#L117-L159`). Finally it applies
Harmony (`new Harmony(harmonyID); harmony.PatchAll(typeof(Mod).Assembly)`) and unpatches
in `OnDispose` to support hot reload (`repo/NightShift/Mod.cs#L164-L166`, `#L182-L183`).

### Per-citizen schedules are serialized

`CitizenSchedule` is `IComponentData, IQueryTypeParameter, ISerializable`, holding the
day, day-off flag, work/lunch times, remote flag, and work type. `Serialize` writes a
version int first; `Deserialize` reads inside a try/catch that falls back to a
legacy-compatible default so an older save does not crash the reader
(`repo/NightShift/Components/CitizenSchedule.cs#L12-L78`):

```csharp
public void Serialize<TWriter>(TWriter writer) where TWriter : IWriter
{
    writer.Write(version);
    writer.Write(day); writer.Write(dayoff);
    writer.Write(go_to_work); writer.Write(start_work); writer.Write(end_work);
    // ... start_lunch, end_lunch, work_from_home, work_type
}
```

`CitizenScheduleSystem` refreshes these daily via `CitizenScheduleHelper.
CalculateScheduleForCitizen`, which folds prefab ownership, economy parameters, commute
percentiles, and slider probabilities into a deterministic-but-varied plan every ECS
system then reads (`repo/NightShift/Utils/CitizenScheduleHelper.cs#L18-L102`).

### Three bridges to peer mods

Two bridges reach **outward** by reflection. `CustomChirpsBridge` mirrors the Custom
Chirps `DepartmentAccount` enum locally, resolves `CustomChirpApiSystem.PostChirp`
lazily (by FQN, then by scanning loaded assemblies), and gates every call on
`IsAvailable` so the mod no-ops cleanly when Custom Chirps is absent
(`repo/NightShift/Bridge/CustomChirpsBridge.cs#L40-L119`):

```csharp
_apiType = Type.GetType("CustomChirps.Systems.CustomChirpApiSystem, CustomChirps")
           ?? FindType("CustomChirps.Systems.CustomChirpApiSystem");
if (_apiType != null)
    _postChirp = _apiType.GetMethod("PostChirp", BindingFlags.Public | BindingFlags.Static);
```

One bridge is an **inbound** static surface. `ElectionsBridge` is a `public static class`
(`ApiVersion = 3`) the optional Elections [RT Module] mod pushes state into - a mayor
resource-consumption multiplier (clamped 0.75-1.25), an election-day Sunday override, and
special-event suppression - which Time2Work's own systems read
(`repo/NightShift/Bridge/ElectionsBridge.cs#L5-L47`):

```csharp
public static void SetMayorResourceConsumptionMultiplier(int effectId, float multiplier)
{ s_EffectId = effectId; s_ResourceConsumptionMultiplier = Clamp(multiplier, 0.75f, 1.25f); }

public static float GetEffectiveResourceConsumption(float baseValue)
{ return Math.Max(1f, baseValue * s_ResourceConsumptionMultiplier); }
```

`EconomyParameterUpdaterSystem` consumes `ElectionsBridge.GetEffectiveResourceConsumption`;
absent the Elections mod, the bridge returns its neutral defaults and behavior is
unchanged (`repo/NightShift/Systems/EconomyParameterUpdaterSystem.cs#L43`).

### Harmony full-getter replacement drives visual time dilation

`Time2WorkPatches` Prefix-replaces the entire `TimeSystem` read API - both `GetYear`
overloads, `get_normalizedDate`, `GetDay`, `GetCurrentDateTime`, `GetStartingDate`,
`GetElapsedYears`, `GetTimeOfYear`, `GetTimeOfDay`, plus `TimeUISystem.GetDay`/`GetTicks`
- each returning `false` to skip the original and substitute the mod's timekeeper value
(`repo/NightShift/Patches/Time2WorkPatches.cs#L58-L169`):

```csharp
[HarmonyPatch(typeof(TimeSystem), "GetCurrentDateTime")]
[HarmonyPrefix]
static bool TimeSystemPatches_GetCurrentDateTime(ref DateTime __result, TimeSystem __instance)
{
    var t2w = World...GetOrCreateSystemManaged<Time2WorkTimeSystem>();
    __result = t2w.GetCurrentDateTime();
    return false;                       // full replacement
}
```

A `TimeSystem.OnUpdate` Postfix writes `m_Time`/`m_Date`/`m_Year` back through Harmony
`Traverse` to keep the vanilla fields consistent
(`repo/NightShift/Patches/Time2WorkPatches.cs#L34-L42`). This is why day length changes
without touching simulation speed.

### An economy postfix and a climate override

The `service_expenses_night_reduction` setting is enforced by a **Harmony Postfix on the
economy cost getter** `CityServiceUpkeepSystem.CalculateUpkeep`, scaling `__result` only
between 23:00 and 06:00 (`repo/NightShift/Patches/Time2WorkPatches.cs#L193-L224`):

```csharp
int hour = timeSys.GetCurrentDateTime().Hour;
if (!(hour >= 23 || hour <= 6)) return;
float factor = (100f - pct) / 100f;
__result = (int)math.round(__result * factor);
```

A companion Postfix on `CityServiceBudgetSystem.OnUpdate` reflects into the private
`m_Expenses`/`m_ExpensesTemp` arrays and scales the `ServiceUpkeep` slot the same way
(`repo/NightShift/Patches/Time2WorkPatches.cs#L226-L269`). Separately, a Prefix fully
replaces `ClimateSystem.SampleClimate` with a 12-month-scaled recompute (`time = t * 12`)
so temperature/precipitation/cloud/aurora track the elongated calendar
(`repo/NightShift/Patches/Time2WorkPatches.cs#L171-L190`).

### A Burst IJobChunk biases pathfinding toward events

`PathfindSetupSystem_LeisureEventBiasPatch` prefixes the private
`PathfindSetupSystem.FindTargets`, running vanilla for every target type except Leisure.
For leisure it reads the protected `SystemBase.Dependency` by reflection, schedules a
Burst `IJobChunk` that adds `EVENT_COST_BONUS = -500000f` to venues carrying
`SpecialEventData`, returns the `JobHandle` in `__result`, and returns `false` to skip
the vanilla branch - preserving the TempJob queue lifetime
(`repo/NightShift/Patches/PathfindSetupSystem_LeisureEventBiasPatch.cs#L37-L136`):

```csharp
[BurstCompile]
private struct SetupLeisureTargetJob_Biased : IJobChunk { ... }
```

## Techniques demonstrated

- [ECS ISerializable save data](../how-to/recipes/ecs-serializable-savedata.md)
  (family E) - `CitizenSchedule` is a versioned `ISerializable IComponentData` whose
  `Deserialize` wraps reads in a try/catch that falls back to a legacy default, so per-
  citizen plans persist across save/reload without breaking older saves
  (`repo/NightShift/Components/CitizenSchedule.cs#L12-L78`).
- [Reflection bridges between mods](../how-to/recipes/reflection-mod-bridges.md)
  (family G) - three integration surfaces: `CustomChirpsBridge` resolves and invokes the
  Custom Chirps API purely by reflection, gated on `IsAvailable`
  (`repo/NightShift/Bridge/CustomChirpsBridge.cs#L40-L119`); `ElectionsBridge` is a static
  surface the Elections mod pushes into and Time2Work reads
  (`repo/NightShift/Bridge/ElectionsBridge.cs#L5-L47`); `SocialTripsBridge` reflects into
  the optional Social Trips mod. All degrade to neutral no-ops when the partner is absent.
- [Harmony postfix on economy/price getters](../how-to/recipes/harmony-price-getter-postfix.md)
  (family C) - the night discount is a Postfix on the cost getter
  `CityServiceUpkeepSystem.CalculateUpkeep` that scales `__result` only at night
  (`repo/NightShift/Patches/Time2WorkPatches.cs#L193-L224`). The same file shows the
  broader Harmony-on-getters surface: a wall of `TimeSystem` getter *Prefix*
  replacements (`return false`) that fully substitute the mod's time values
  (`repo/NightShift/Patches/Time2WorkPatches.cs#L58-L169`).
- [Burst IJobChunk jobs](../how-to/recipes/burst-ijobchunk.md) (family S) - the leisure
  event-bias patch schedules a `[BurstCompile] struct SetupLeisureTargetJob_Biased :
  IJobChunk` over leisure providers, adding a large negative cost to active-event venues
  and returning the `JobHandle` so vanilla queue lifetime is preserved
  (`repo/NightShift/Patches/PathfindSetupSystem_LeisureEventBiasPatch.cs#L107-L136`). This
  is a genuine `IJobChunk` (not the source-generated `IJobEntity` form).
- [Weather/climate system override](../how-to/recipes/weather-climate-override.md)
  (family AB) - a Prefix fully replaces `ClimateSystem.SampleClimate` with a 12-month-
  scaled recompute so seasons track the elongated calendar
  (`repo/NightShift/Patches/Time2WorkPatches.cs#L171-L190`).

Supporting techniques on display: ECS system replacement by disable + reschedule
(`repo/NightShift/Mod.cs#L94-L159`); visual time dilation via
`Time2WorkTimeSystem.kTicksPerDay = floor(slow_time_factor * TimeSystem.kTicksPerDay)`
(`repo/NightShift/Systems/Time2WorkTimeSystem.cs#L23-L75`); and preset/country profile
arrays fanned out by `SetParameters(index)`.

See the [technique index](../technique-index.md) for the family ledger. Related
explanation pages: [system replacement](../explanation/system-replacement.md),
[multi-phase scheduling](../explanation/multi-phase-scheduling.md),
[serialization](../explanation/serialization.md),
[dependency strategy](../explanation/dependency-strategy.md), and
[settings and data](../explanation/settings-and-data.md).

## Key decisions & tradeoffs

- **Disable + reschedule vs. Harmony-patch the sim.** Hard-disabling eleven vanilla
  systems and scheduling ~30 replacements gives Time2Work full ownership of trip
  behavior but guarantees conflict with any mod that expects those vanilla systems, and
  demands re-verification of the disable list every patch
  (`repo/NightShift/Mod.cs#L94-L159`).
- **Prefab-side + getter Harmony vs. deep sim patches.** Time dilation is implemented by
  replacing `TimeSystem` getters (per-query values) rather than mutating the tick loop
  (`repo/NightShift/Patches/Time2WorkPatches.cs#L58-L169`), while the night discount is a
  cost-getter postfix - Harmony reserved for the seams ECS scheduling cannot reach.
- **Reflection/static bridges vs. hard dependencies.** Custom Chirps and Social Trips are
  reached by reflection and Elections by a static surface, so the declared dependency
  list stays short and every integration no-ops gracefully when its partner is missing
  (`repo/NightShift/Bridge/CustomChirpsBridge.cs#L48-L51`,
  `repo/NightShift/Bridge/ElectionsBridge.cs#L44-L47`).
- **Versioned schedule serialization vs. recompute-every-frame.** Storing each cim's plan
  in a serialized component lets many systems share one answer for "today" and survive
  reload; the try/catch fallback tolerates a schema change
  (`repo/NightShift/Components/CitizenSchedule.cs#L57-L78`).
- **Unpatch on dispose for hot reload.** `OnDispose` rebuilds the Harmony handle and calls
  `UnpatchAll(harmonyID)` so a mid-session mod reload removes the getter/climate/economy
  patches cleanly (`repo/NightShift/Mod.cs#L182-L183`).

## Pitfalls / upstream-watch

- **Resident-AI and time-mod collisions.** Because it disables the vanilla citizen,
  worker, student, tourism, and UI systems, another AI-replacement mod (for example
  Realistic Path Finding) or another time mod (Time & Weather Anarchy) will very likely
  conflict; coordinate load order and test combined stacks
  (`repo/NightShift/Mod.cs#L94-L110`).
- **Harmony getter fragility.** The time-dilation and economy patches bind vanilla getter
  signatures on `TimeSystem`/`TimeUISystem`/`CityServiceUpkeepSystem`/`ClimateSystem`; a
  base-game signature change silently unbinds the patch
  (`repo/NightShift/Patches/Time2WorkPatches.cs#L58-L169`).
- **Private-field reflection in the budget postfix.** `CityServiceBudgetSystem_OnUpdate`
  reflects into the private `m_Expenses`/`m_ExpensesTemp` arrays by field name via
  `AccessTools.Field`; a rename breaks the night discount's budget path
  (`repo/NightShift/Patches/Time2WorkPatches.cs#L226-L269`).
- **Slow-time is a save-level option.** `TimeSettingsMultiplierSystem` runs once per
  session, so `slow_time_factor` / `daysPerMonth` changes need a reload to fully apply
  (`repo/NightShift/Mod.cs#L154-L155`).
- **PostChirp does not wrap its invoke.** `CustomChirpsBridge.PostChirp` catches
  reflection *resolution* failures but not a runtime signature mismatch on the invoke, so
  an API drift would surface as an unhandled exception rather than a silent skip; callers
  gate on `IsAvailable` first (`repo/NightShift/Bridge/CustomChirpsBridge.cs#L62-L70`).
- **No CSV telemetry exporter.** The only diagnostics are opt-in log toggles
  (`shopping_log_enabled`, `personal_car_diagnostics_enabled`) that write to `Mod.log`;
  there is no `Time2WorkDiagnosticsSystem` and no `schedule-telemetry.csv` (a prior
  fabrication) - do not document one. `CitizenScheduleDebugSystem` exists in-tree but is
  not registered and never runs (`repo/NightShift/Mod.cs#L124`).
- **Dormant reference systems.** `TruckScheduleSystem` (removed as a feature in
  userModVersion 2.0.2), `LedgerTrendSystem`, and two commented-out systems exist in the
  tree but are not scheduled in `Mod.OnLoad`; treat them as teaching/reference code, not
  capabilities (`repo/NightShift/Mod.cs#L138`, `#L148`).
- `Needs Verification (in-game)`: the exact routing/economy change per slider, and the
  graceful-degradation behavior of I18n Everywhere-routed locales when that dependency is
  missing - neither is confirmable from source alone.

## Source pointers

- Dossier: `../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/`
  (index / source / modding / guide + notes, incl. `notes/depth-challenge-20260625.md`).
- Repo @ `d42921ffb1f6bbbf2c2b086bb17727bf0f637c39` (branch `master`), key files:
  - `repo/NightShift/Mod.cs` - disable vanilla + schedule ~30 systems + Harmony `PatchAll`.
  - `repo/NightShift/Components/CitizenSchedule.cs`,
    `.../Utils/CitizenScheduleHelper.cs` - serialized per-citizen schedule + synthesis.
  - `repo/NightShift/Bridge/CustomChirpsBridge.cs`, `.../ElectionsBridge.cs`,
    `.../SocialTripsBridge.cs` - the three peer-mod bridges.
  - `repo/NightShift/Patches/Time2WorkPatches.cs` - getter replacements, economy postfix,
    climate override.
  - `repo/NightShift/Patches/PathfindSetupSystem_LeisureEventBiasPatch.cs` - Burst
    `IJobChunk` leisure event bias.
  - `repo/NightShift/Systems/Time2WorkTimeSystem.cs` - `kTicksPerDay` visual time dilation.
- Dependencies: I18n Everywhere (75426), Custom Chirps (121812, reflection bridge);
  Social Trips and Elections [RT Module] are optional bridge partners with no declared
  dependency. Lib.Harmony 2.2.2. Storefront modId 77171, PublishConfiguration 2.9.1 /
  GameVersion 1.6.*.
