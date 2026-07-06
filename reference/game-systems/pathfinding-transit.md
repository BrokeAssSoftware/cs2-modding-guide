---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Reference: CS2 Pathfinding & Transit"
Summary: Lookup table of the Cities Skylines II pathfinding and transit surfaces a code mod reads or hooks - the transit perceived-cost model, PathUtils drive-spec getters, PathfindCarData turn costs, CarLaneFlags, PathInformation, and the 2x-m/s speed-unit convention - with source-verified hook points from realistic-path-finding and road-speed-adjuster at pinned commits.
diataxis: reference
source_version: "1.6.0f1 (realistic-path-finding@50645fa; date-pinned, static source only)"
last_reverified: "2026-07-05"
status: source-verified
canonical_mods:
  - realistic-path-finding@50645fa6a078181365e36a42e2e27b96699bf02a
  - road-speed-adjuster@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed
technique_applicability: [simulation, infrastructure]
Created: 2026-07-05
Updated: 2026-07-05
Owners:
  - codex
References:
  - Label: "Reference: vanilla ECS components & systems mods touch"
    Path: ../ecs-components-catalog.md
  - Label: "Explanation: ECS Fundamentals"
    Path: ../../explanation/ecs-fundamentals.md
  - Label: "Case study: Realistic Path Finding"
    Path: ../../case-studies/realistic-path-finding.md
  - Label: Official CS2 modding wiki (authoritative type list)
    Path: https://cs2.paradoxwikis.com/Modding
  - Label: Technique Index
    Path: ../../technique-index.md
---

# Reference: CS2 Pathfinding & Transit

> **Reference - patch-sensitive.** Current live game: **1.6.0f1 "Summer Solstice"** (2026-06-22).
> Every surface below is cited from mod source at a **date-pinned commit**
> (`realistic-path-finding@50645fa`, dated 2026-06-22, the 1.6.0f1 release day; and
> `road-speed-adjuster@e0c0c0b3`, dated 2025-12-15). These are **static source reads only** -
> the type names, field names, and `PathUtils` method signatures were confirmed present at the pin
> but were **NOT** exercised against the running game (`last_reverified: 2026-07-05`). CS2's
> `Game.Pathfind` / `Game.Net` / `Game.Prefabs` pathfinding types and the internal
> `Game.Pathfind.PathUtils` helper change across patches, and the drive-spec overload argument
> lists are exactly the kind of thing a patch reshuffles. Treat every type, field, and method
> signature as a moving target and re-verify on each game release. For the authoritative live type
> list, prefer the [official CS2 modding wiki](https://cs2.paradoxwikis.com/Modding).

Dry lookup of the **vanilla** pathfinding and transit surfaces a code mod reads, writes, or Harmony-
patches. Every row is **Verified**: it cites a real mod file+line that touches that surface at its
pinned commit. Anything not grounded in a cited line is labelled **Needs Verification** (see the
section at the bottom). For the "why" of systems vs. components see
[explanation/ecs-fundamentals.md](../../explanation/ecs-fundamentals.md); for the full narrative of
how realistic-path-finding assembles these into a replacement pathfinding layer see the
[case study](../../case-studies/realistic-path-finding.md).

Re-verify any cite with (Git-Bash), from this page's directory (`reference/game-systems/`, **three**
`../` up to the research tree):

```
git -C ../../../vice-and-order-research/mods/dossiers/<slug>/repo show <pin>:<path>
```

Pins used below (short form in cites; full 40-char commits in `canonical_mods`):
`realistic-path-finding@50645fa`, `road-speed-adjuster@e0c0c0b3`.

## How to read the columns

| Column | Meaning |
| --- | --- |
| Surface | The vanilla type, field, method, or enum member (as written in source). |
| Kind | `component` / `field` / `enum` / `method` (vanilla `PathUtils` static) / `Harmony patch`. |
| Namespace | The `Game.*` namespace it lives in (from `using` + fully-qualified refs at the pin). |
| Cite at pin | `slug/repo/<path>#Lxx`. Line hints are approximate - confirm with the git command above. |

---

## 1. The transit perceived-cost model (7 stages)

`realistic-path-finding` rewrites how a citizen "feels" the cost of using public transit inside a
generic helper, `RPFRouteUtils.StripTransportSegments<T>(...)`. As it walks the transit segments of
a computed path, at each boarding waypoint it accumulates a **perceived wait in seconds** and feeds
it to a transport-estimate buffer. The seven stages below are applied in order to a single `seconds`
accumulator. All tunables (`kCrowd`, `schedule_factor`, `transfer_penalty`,
`feeder_trunk_transfer_penalty`, `t2w_timefactor`, `waiting_weight`, `crowdness_stop_threashold`)
arrive as method parameters from the mod settings.

Method signature (all vanilla lookups it consumes are visible here):
`realistic-path-finding/repo/RealisticPathFinding/Utils/RPFRouteUtils.cs#L67`.

The vehicle capacity used by the crowding stage is read from vanilla
`Game.Prefabs.PublicTransportVehicleData.m_PassengerCapacity`
(`.../Utils/RPFRouteUtils.cs#L94`; accumulator initialised `#L77`).

| # | Stage | What it does (source) | Cite at pin (`.../Utils/RPFRouteUtils.cs`) |
| --- | --- | --- | --- |
| 1 | Boarding time | `seconds = MathUtils.RoundToIntRandom(ref random, componentData.m_BoardingTime)` - base boarding cost from vanilla `Game.Prefabs.TransportStopData.m_BoardingTime`. | `#L150` |
| 2 | Add real wait (t2w) | `seconds += (int)(wp.m_AverageWaitingTime / t2w_timefactor)` - adds the stop's average wait from `Game.Routes.WaitingPassengers.m_AverageWaitingTime`, divided by the time2work time factor. | `#L154` |
| 3 | Crowding gate | If `wp.m_Count > capacity * crowdness_stop_threashold`, compute `signal = saturate(wp.m_Count / capacity)` and `crowdingFactor = 1 + kCrowd * signal` (only above the threshold; else factor stays `1`). Uses `WaitingPassengers.m_Count`. | `#L160`, `#L163`, `#L165` |
| 4 | Waiting weight | `seconds = ceil(seconds * crowdingFactor * waiting_weight)` - applies both the crowding factor and the global waiting-weight multiplier at once. | `#L169` |
| 5 | Scheduled halving | For `TransportType.Train`, `Ship`, or `Airplane`, `seconds = (int)(seconds * schedule_factor)` - scheduled/known-timetable modes get a reduced perceived wait. | `#L180` |
| 6 | Transfer penalty | On a detected transfer (`currentRoute != lastBoardedRoute`), multiply by `feeder_trunk_transfer_penalty` when feeder->trunk, else by `transfer_penalty`. | `#L190`, `#L193` |
| 7 | Subtract counted wait | `seconds -= (int)(wp.m_AverageWaitingTime / t2w_timefactor)` - removes the wait the game already counted in `PathUtils`, to avoid double-charging; if the result goes negative the estimate is skipped. | `#L203` |
| - | Commit | `transportEstimateBuffer.AddWaitEstimate(pathElement.m_Target, seconds)` - writes the final perceived wait for that waypoint. | `#L211` |

Vanilla surfaces this model reads:

| Surface | Kind | Namespace | Cite at pin |
| --- | --- | --- | --- |
| `TransportStopData.m_BoardingTime` | field | `Game.Prefabs` | `.../Utils/RPFRouteUtils.cs#L150` |
| `TransportStopData.m_TransportType` | field | `Game.Prefabs` | `.../Utils/RPFRouteUtils.cs#L177` (branch on it), also `#L188`, `#L200` |
| `PublicTransportVehicleData.m_PassengerCapacity` | field | `Game.Prefabs` | `.../Utils/RPFRouteUtils.cs#L94` |
| `WaitingPassengers.m_AverageWaitingTime` | field | `Game.Routes` | `.../Utils/RPFRouteUtils.cs#L154`, `#L203` |
| `WaitingPassengers.m_Count` | field | `Game.Routes` | `.../Utils/RPFRouteUtils.cs#L160`, `#L163` |
| `TransportType` (`Train`/`Ship`/`Airplane`) | enum | `Game.Prefabs` | `.../Utils/RPFRouteUtils.cs#L177` (compared at); also `Bus`/`Tram`/`Subway`/`Ferry` at `#L27`, `#L30-L31` |
| `CurrentRoute` | component | `Game.Routes` | `.../Utils/RPFRouteUtils.cs#L67` (lookup param) |

---

## 2. `PathUtils` drive-spec getters (Harmony postfix targets)

The internal vanilla helper `Game.Pathfind.PathUtils` builds a `PathSpecification` (with a
`PathfindCosts m_Costs` field) for each drivable lane. `realistic-path-finding` postfix-patches its
drive-spec getters to add a per-lane penalty. Because `GetCarDriveSpecification` is **overloaded**,
each Harmony patch disambiguates the target with an explicit argument-type array
(`new[] { typeof(...), ... }`) passed to `[HarmonyPatch(typeof(PathUtils), nameof(...), <types>)]`.

| Vanilla method | Overload discriminator (arg types) | Patch class + hook | Cite at pin (`.../Patches/BusLanePatches.cs`) |
| --- | --- | --- | --- |
| `PathUtils.GetCarDriveSpecification` | `NetCurve, NetCarLane, NetMasterLane, PrefabCarLaneData, PrefabPathfindCarData, RuleFlags, float` | `Patch_GetCarDriveSpecification_Road` (Postfix) | `#L42` (attr), `#L50-L53` (class + postfix) |
| `PathUtils.GetCarDriveSpecification` (2nd overload) | `NetCurve, NetCarLane, NetMasterLane, NetTrackLane, PrefabCarLaneData, PrefabPathfindCarData, RuleFlags, float` | `Patch_GetCarDriveSpecification_RoadWithTrack` (Postfix) | `#L64` (attr), `#L73-L76` (class + postfix) |
| `PathUtils.GetTaxiDriveSpecification` | `NetCurve, NetCarLane, PrefabPathfindCarData, PrefabPathfindTransportData, RuleFlags, float` | `Patch_GetTaxiDriveSpecification` (Postfix) | `#L86` (attr), `#L93-L96` (class + postfix) |
| `PathUtils.TryAddCosts(ref m_Costs, PathfindCosts)` | (helper the postfix calls to fold in the penalty) | called from `BusLanePenaltyConfig.AddPenalty` | `#L29` |

Each postfix takes `ref PathSpecification __result` and `ref NetCarLane carLane` and, when the lane
is public-transit-only, adds a fixed-seconds cost to `__result.m_Costs`
(`.../Patches/BusLanePatches.cs#L27-L31`). The penalty seconds come from a mod-settings value,
`Mod.m_Setting.nonbus_buslane_penalty_sec` (`.../Patches/BusLanePatches.cs#L19-L20`).

> Note: `PathUtils` is an **internal** engine helper, not part of the documented modding surface;
> these overload argument lists are the most patch-fragile thing on this page. Re-verify the exact
> `typeof(...)` sequence on every game update before trusting a `[HarmonyPatch]` against it.

---

## 3. `CarLaneFlags.PublicOnly` (public-transit-only lane gate)

The bus-lane penalty above is gated on the vanilla lane flag `Game.Net.CarLaneFlags.PublicOnly`:
only lanes flagged public-transit-only receive the extra perceived cost for private cars/taxis.

| Surface | Kind | Namespace | Cite at pin |
| --- | --- | --- | --- |
| `CarLaneFlags.PublicOnly` | enum | `Game.Net` | `realistic-path-finding/repo/RealisticPathFinding/Patches/BusLanePatches.cs#L27` (`(carLane.m_Flags & CarLaneFlags.PublicOnly) != 0`) |
| `NetCarLane.m_Flags` | field | `Game.Net` | `.../Patches/BusLanePatches.cs#L27` |
| `PathfindCosts.m_Value` (`float4`) | field | `Game.Pathfind` | `.../Patches/BusLanePatches.cs#L30` (penalty written into `.x`) |

---

## 4. `PathfindCarData` turn-cost fields

`Game.Prefabs.PathfindCarData` is prefab-data on each car prefab holding the pathfinder's per-maneuver
cost weights. `realistic-path-finding/repo/.../Systems/CarPrefabTurnCostFactorSystem.cs` multiplies
these by mod-settings factors (caching the untouched originals in a mod-added `CarTurnCostOrig`
component so the multiplier is reversible). Query: `All = ReadWrite<PathfindCarData>`
(`.../Systems/CarPrefabTurnCostFactorSystem.cs#L41`); read via `GetComponentData<PathfindCarData>`
(`#L93`), written back via `SetComponentData` (`#L136`).

| Field | Kind | Namespace | Cite at pin (`.../Systems/CarPrefabTurnCostFactorSystem.cs`) |
| --- | --- | --- | --- |
| `PathfindCarData.m_TurningCost` | field | `Game.Prefabs` | read `#L108`, write `#L130` |
| `PathfindCarData.m_UnsafeTurningCost` | field | `Game.Prefabs` | read `#L109`, write `#L131` |
| `PathfindCarData.m_CurveAngleCost` | field | `Game.Prefabs` | read `#L110`, write `#L132` |
| `PathfindCarData.m_UTurnCost` | field | `Game.Prefabs` | read `#L111`, write `#L133` |
| `PathfindCarData.m_UnsafeUTurnCost` | field | `Game.Prefabs` | read `#L112`, write `#L134` |

Each cost is a `PathfindCosts` (the same vanilla type as the drive-spec penalty in section 2), whose
`m_Value` is a `float4`; the multiplier scales it directly (e.g. `turning.m_Value *= turnMultiplier`,
`.../CarPrefabTurnCostFactorSystem.cs#L117`). The mod-added cache struct
`CarTurnCostOrig : IComponentData` (`.../CarPrefabTurnCostFactorSystem.cs#L13`) declares its five
fields as `PathfindCosts` (`#L15-L19`) and snapshots the vanilla values into them - not vanilla.

---

## 5. `PathInformation` (result of a pathfind request)

`Game.Pathfind.PathInformation` is the component the pathfinder writes back onto a requester once a
route is computed; it pairs with a `PathElement` dynamic buffer. `realistic-path-finding` requests a
path by adding both (`ComponentTypeSet(ReadWrite<PathInformation>(), ReadWrite<PathElement>())`,
`.../Systems/RPFResourceBuyerSystem.cs#L124`) with `m_State = PathFlags.Pending`
(`#L1181`), then reads the result once `Pending` clears.

| Surface | Kind | Namespace | Cite at pin (`.../Systems/RPFResourceBuyerSystem.cs`) |
| --- | --- | --- | --- |
| `PathInformation` | component | `Game.Pathfind` | `ComponentLookup<PathInformation>` `#L267`, `#L779`; type set `#L124` |
| `PathInformation.m_State` (`PathFlags`) | field | `Game.Pathfind` | read `#L951` (`(m_State & PathFlags.Pending) == 0`), write `#L1181` |
| `PathInformation.m_Destination` | field | `Game.Pathfind` | `#L953` (`Entity destination = pathInformation.m_Destination`) |
| `PathFlags.Pending` | enum | `Game.Pathfind` | `#L951`, `#L1181` |
| `PathElement` | buffer | `Game.Pathfind` | type set `#L124` |

See [reference/ecs-components-catalog.md](../ecs-components-catalog.md) for the wider list of
`Game.Pathfind` surfaces the corpus touches.

---

## 6. The CS2 speed-unit gotcha (game speed = 2x m/s)

CS2 lane speed limits (`Game.Net.CarLane.m_SpeedLimit` / `m_DefaultSpeedLimit`,
`Game.Net.TrackLane.m_SpeedLimit`) are stored **not** in plain m/s: the game's internal unit is
**2x m/s**. So the conversion from km/h to game units is `km/h / 1.8` (i.e. `/3.6` to m/s, then
`x2`), **not** the usual `/3.6`. Getting this wrong makes every custom speed off by a factor of 2.

`road-speed-adjuster` documents and applies this directly:

| Fact | Cite at pin |
| --- | --- |
| Comment: "Game uses 2x m/s, so divide by 1.8 instead of 3.6" | `road-speed-adjuster/repo/Systems/RoadSpeedApplySystem.cs#L92-L93` |
| `float speedGameUnits = speedKmh / 1.8f;` | `.../Systems/RoadSpeedApplySystem.cs#L94` |
| Write `carLane.m_DefaultSpeedLimit` and `carLane.m_SpeedLimit` in game units | `.../Systems/RoadSpeedApplySystem.cs#L120-L121` |
| Write `trackLane.m_SpeedLimit` in game units | `.../Systems/RoadSpeedApplySystem.cs#L133` |
| Reverse direction (game units -> km/h): `carLane.m_SpeedLimit * 1.8f` | `road-speed-adjuster/repo/Systems/RoadSpeedToolSystem.cs#L403`, `#L410` |
| Same `/1.8` conversion on restore/clear | `road-speed-adjuster/repo/Systems/ClearCustomSpeedsSystem.cs#L94` |

The same 2x-m/s convention shows up in `realistic-path-finding`, which reads a prefab speed limit as
a free-flow proxy and hard-codes m/s thresholds against it:

| Fact | Cite at pin |
| --- | --- |
| `float mps = math.max(cfg.MinFreeflowMps, rd.m_SpeedLimit)` (prefab `RoadData.m_SpeedLimit` used as speed proxy) | `realistic-path-finding/repo/RealisticPathFinding/Systems/CarCongestionEwmaSystem.cs#L198` |
| Speed thresholds in m/s (`>= 16.7f` == ~60 km/h) with the comment noting the unit | `realistic-path-finding/repo/RealisticPathFinding/Systems/CarTurnAndHierarchyBiasSystem.cs#L367-L368` |

> Note: `road-speed-adjuster` works in km/h at the CarLane/TrackLane level and calls the unit "game
> units (2x m/s)". `RoadData.m_SpeedLimit` (prefab-data) is treated by realistic-path-finding as a
> plain m/s proxy for free-flow. Confirm which field you are touching and its unit before scaling -
> the `/1.8` factor is specifically for the lane-instance `m_SpeedLimit` fields.

---

## Needs Verification

- **`PathUtils` overload argument lists** - the exact `typeof(...)` sequences in section 2 are
  confirmed present at `50645fa` but `PathUtils` is an internal engine helper; the overload set is
  the single most likely thing to drift between patches. Re-verify before shipping any `[HarmonyPatch]`
  against it. Confirmed only that the mod compiled against these at the pin, not that they match the
  live 1.6.0f1 assembly.
- **Full field layouts** of `PathfindCarData`, `PathInformation`, `TransportStopData`,
  `PublicTransportVehicleData`, `WaitingPassengers`, `PathSpecification`, `PathfindCosts` - only the
  fields cited by line are confirmed; the remaining fields are patch-sensitive and unread here.
- **`PathfindCosts.m_Value` layout** - both the `PathfindCarData` turn-cost fields (section 4) and the
  drive-spec penalty (section 2) use the vanilla type `PathfindCosts`, whose `m_Value` is a `float4`
  (`.../Patches/BusLanePatches.cs#L30`). Only the `m_Value` member is exercised at the pin; the rest of
  the `PathfindCosts` layout is patch-sensitive and unverified here. Note: only `PathfindCosts`
  (plural) appears in source at `50645fa` - there is no separate singular `PathfindCost` type in this
  corpus.
- **Runtime behaviour** of the 7-stage perceived-cost model (does the accumulated `seconds` actually
  reroute citizens as intended) is `Needs Verification (in-game)` - only the static formula is
  source-confirmed.
- Any pathfinding/transit surface **not** in this two-mod corpus (cargo/aircraft/ship routing
  internals, the pathfind job scheduler itself, `NavigationLane`, etc.) is out of scope, not
  "does not exist".

## See also

- [reference/ecs-components-catalog.md](../ecs-components-catalog.md) - the wider catalog of vanilla
  `Game.Pathfind` / `Game.Net` / `Game.Prefabs` ECS surfaces the corpus reads or writes.
- [explanation/ecs-fundamentals.md](../../explanation/ecs-fundamentals.md) - components vs. systems
  vs. buffers; why prefab-data (`PathfindCarData`, `RoadData`) differs from instance components
  (`CarLane`).
- [case-studies/realistic-path-finding.md](../../case-studies/realistic-path-finding.md) - how these
  hooks combine into a full replacement pathfinding/transit layer.
- [Technique Index](../../technique-index.md) - the technique families (Harmony postfix,
  prefab-data mutation, system replacement) these hooks belong to.
- Official [CS2 modding wiki](https://cs2.paradoxwikis.com/Modding) - authoritative live type list.
