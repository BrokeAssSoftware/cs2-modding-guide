---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Case study: Traffic Tool Essentials"
case_study: traffic-tool-essentials
mod: "Traffic Tool Essentials"
dossier: ../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/
repo_commit: 10973595ac9ed37f47ca24eb788d4421dc295fa0
source_version: "~1.5.x (traffic-tool-essentials@1097359; date-pinned, static source only)"
last_reverified: "2026-07-05"
diataxis: explanation
techniques: [T, M, S, E, P, BA, BB]
technique_applicability: [simulation, infrastructure, ui, core]
status: source-verified
Created: 2026-07-05
Updated: 2026-07-05
Owners:
  - codex
Summary: A no-Harmony traffic overhaul that narrows (not disables) the vanilla traffic-light systems by rewriting their EntityQueries, then layers three independent ECS subsystems - Green Wave gate-only signal sync, custom polygon Depot Zones, and vehicle-stats telemetry - each with its own persistence schema and cross-SystemGroup ordering hazard.
---

# Traffic Tool Essentials - case study

> A traffic-light overhaul that takes over the vanilla signal systems without a
> single Harmony patch - by *narrowing their EntityQueries* instead of disabling
> them - and then stacks three unrelated ECS subsystems (green-wave sync, polygon
> depot zones, live telemetry) on top, each carrying its own save schema and its
> own ordering trap. A tour of how far pure ECS can go, and what it costs.

All code claims below are cited to `repo/<path>#Lxx` at pinned commit
`10973595ac9ed37f47ca24eb788d4421dc295fa0` (Release v3.0.7 "Foxglove",
mod id `C2VM.TrafficToolEssentials`, displayName "Traffic Tool Essentials",
`repo/TLEFrontend/mod.json`), surfaced through the dossier at
`../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/`. This pin
is date-pinned static source only; `source_version` is approximate. There is **no
Harmony dependency anywhere in the mod** - every takeover below is done through
ECS query rewriting and update-phase ordering.

## What it does / why it's instructive

Traffic Tool Essentials (TTE, the successor to C2VM's Traffic Lights Enhancement)
gives players per-intersection control over traffic-light phasing, a "Green Wave"
that synchronises a corridor of lights, a polygon tool that fences transport
depots into service zones, and a dashboard of live vehicle/transit statistics.

It is instructive because it is really **four teaching artifacts in one process**:

1. A no-Harmony replacement of two vanilla systems
   (`Game.Net.TrafficLightInitializationSystem` and
   `Game.Simulation.TrafficLightSystem`) done by *editing their private
   `EntityQuery` fields* so they stop matching modded intersections, then running
   patched clones `UpdateBefore` them.
2. A gate-only coordination state machine (`SyncGroupSystem`) that never seizes a
   phase - it only ever *prevents* one - so it composes with the still-running
   vanilla state machine instead of fighting it.
3. A custom polygon-zone tool (`DepotZoneSystem`) with point-in-polygon
   hit-testing, self-intersection rejection, and save-stable overlap resolution.
4. Ring-buffer telemetry components persisted through `ISerializable` with
   independent, per-component schema versioning and corruption clamps.

Because the four are only loosely coupled, TTE also shows the *seams* between
subsystems - most sharply the cross-`SystemGroup` write-after-write bug that
forced the green-wave sync to be re-homed into the traffic-light system's own
`OnUpdate` (below).

## Architecture at a glance

### Load-time: create systems, narrow queries, order phases

`Mod.OnLoad` calls `GetOrCreateSystemManaged` for the two vanilla systems and the
two patched clones (repo/TrafficToolEssentials/Mod.cs#L122-L125), creates the
green-wave, transit, and depot-zone systems
(repo/TrafficToolEssentials/Mod.cs#L128-L147), then delegates to `SystemSetup`.

`SystemSetup` is the heart of the takeover. Instead of disabling the vanilla
systems, it **removes modded intersections from their queries** by appending a
`WithNone<CustomTrafficLights>()` clause to each system's private query field
(repo/TrafficToolEssentials/Mod.cs#L209-L216):

```csharp
var noneList = new NativeList<ComponentType>(1, Allocator.Temp);
noneList.Add(ComponentType.ReadOnly<Components.CustomTrafficLights>());

Utils.EntityQueryUtils.UpdateEntityQuery(m_TrafficLightInitializationSystem, "m_TrafficLightsQuery", noneList);
Utils.EntityQueryUtils.UpdateEntityQuery(m_TrafficLightSystem, "m_TrafficLightQuery", noneList);
noneList.Dispose();
```

It then schedules the two clones to run `UpdateBefore` their vanilla counterparts,
in the matching phases (`Modification4B`, `GameSimulation`)
(repo/TrafficToolEssentials/Mod.cs#L218-L219), and registers the UI, tool,
transit, and depot systems into their phases
(repo/TrafficToolEssentials/Mod.cs#L220-L242). A vanilla intersection keeps the
`TrafficLights` component but not `CustomTrafficLights`, so it stays with the
stock system; the moment TTE adds `CustomTrafficLights` to an intersection it
drops out of the vanilla query and is picked up by the clone. No system is
disabled in the default path - the split is entirely query-driven.

### The compatibility toggle

`SetCompatibilityMode` is the escape hatch for that split. When enabled it flips
the vanilla systems back on (`Enabled = enable`) and tells both clones to switch
their queries to the co-existence form
(repo/TrafficToolEssentials/Mod.cs#L247-L256). In the patched simulation clone
the toggle rebuilds `m_TrafficLightQuery`: the compat form additionally requires
`CustomTrafficLights`, the default form does not
(repo/TrafficToolEssentials/Systems/TrafficLightSystems/Simulation/PatchedTrafficLightSystem.cs#L941-L963).
The setting is wired straight through: the settings property setter calls
`Mod.SetCompatibilityMode(value)` on change
(repo/TrafficToolEssentials/Settings.cs#L90-L99).

### Subsystem 1 - Green Wave sync (gate-only)

`SyncGroupSystem` is a `GameSystemBase` driving a six-state machine
(`BARRIER, UNSYNCED, WAITING_FOR_OTHERS, RUSHING_TO_P1, READY_FOR_SYNC, SYNCED`)
(repo/TrafficToolEssentials/Systems/TrafficLightSystems/Simulation/SyncGroupSystem.cs#L14-L33,
repo/TrafficToolEssentials/Systems/TrafficLightSystems/Simulation/SyncGroupSystem.cs#L101).
It never writes a phase directly. Its only lever is the vanilla-honoured
`m_AvoidSignalGroup` byte on `CustomTrafficLights`: setting it to a phase index
tells the *vanilla* state machine "do not advance into this phase yet," which is
how it holds a light at its last-before-P1 phase until the whole group is aligned
(repo/TrafficToolEssentials/Systems/TrafficLightSystems/Simulation/SyncGroupSystem.cs#L2179-L2188).
Because it only ever prevents a phase, it layers on top of the running signal job
rather than replacing it.

### Subsystem 2 - Depot Zones (polygon tool)

`DepotZoneSystem` (`GameSystemBase`) owns user-drawn polygons that assign
transport depots to service zones
(repo/TrafficToolEssentials/Systems/DepotZone/DepotZoneSystem.cs#L57). It provides
static geometry primitives - a ray-cross `IsPointInPolygon`
(repo/TrafficToolEssentials/Systems/DepotZone/DepotZoneSystem.cs#L1399-L1424) and
a Cramer's-rule `DoLineSegmentsIntersect`
(repo/TrafficToolEssentials/Systems/DepotZone/DepotZoneSystem.cs#L1428-L1462) -
and builds a persistent `NativeHashMap<int, NativeList<Entity>>` cache mapping
each zone id to the depots inside it
(repo/TrafficToolEssentials/Systems/DepotZone/DepotZoneSystem.cs#L108,
repo/TrafficToolEssentials/Systems/DepotZone/DepotZoneSystem.cs#L461-L479). A
companion `DepotZoneDispatchSystem` consumes that mapping during dispatch
(scheduled in `GameSimulation`, repo/TrafficToolEssentials/Mod.cs#L237-L239).

### Subsystem 3 - Telemetry + UI bridge

`VehicleStatsSystem` / `TransitOperationsSystem` sample per-tick counts into
ring-buffer components and the `UISystem` (a `partial class` split across several
`UISystem.*Bindings.cs` files) exposes everything to a webpack-built React
frontend (`repo/TLEFrontend/package.json#L8`, `"build": "webpack"`) via Colossal
UI `GetterValueBinding`/`ValueBinding`/`CallBinding` objects under the `C2VM.TLE`
group (repo/TrafficToolEssentials/Systems/UI/UISystem.UIBindings.cs#L18,
repo/TrafficToolEssentials/Systems/UI/UISystem.UIBindings.cs#L103-L129).

### Data-flow / phase summary

- `Modification4B`: `PatchedTrafficLightInitializationSystem` runs before vanilla
  init (repo/TrafficToolEssentials/Mod.cs#L218).
- `GameSimulation`: `PatchedTrafficLightSystem` runs `UpdateBefore` vanilla
  `TrafficLightSystem`; it drives green-wave sync itself (below), then schedules
  its Burst signal job. Transit, depot-dispatch, and `SimulationUpdateSystem` also
  live here (repo/TrafficToolEssentials/Mod.cs#L219-L242).
- `ToolUpdate` / `UIUpdate` / `UITooltip`: the depot tool, UI, and tooltip systems
  (repo/TrafficToolEssentials/Mod.cs#L220-L235).

## Techniques demonstrated

- [ECS query rewriting](../how-to/recipes/ecs-query-rewriting.md) (family T) -
  `EntityQueryUtils.UpdateEntityQuery` reflects the private query field off a
  vanilla system, rebuilds it through an `EntityQueryBuilder` that copies every
  existing `Any/None/All/Disabled/Absent/Present` clause and appends the new
  `None`, then writes it back
  (repo/TrafficToolEssentials/Utils/EntityQueryUtils.cs#L21-L26,
  repo/TrafficToolEssentials/Utils/EntityQueryUtils.cs#L37-L86). This is the
  narrow-not-disable core of the whole mod.
- [Disable / replace a vanilla system](../how-to/recipes/disable-replace-vanilla-system.md)
  (family M) - the same two vanilla systems are *narrowed* by default and only
  hard-toggled (`Enabled = enable`) when compatibility mode is on, with the clones
  swapping query shape to match
  (repo/TrafficToolEssentials/Mod.cs#L247-L256;
  repo/TrafficToolEssentials/Systems/TrafficLightSystems/Simulation/PatchedTrafficLightSystem.cs#L941-L963).
- [ECS system replacement & ordering](../how-to/recipes/ecs-system-replacement-ordering.md)
  (family B) - the clones are placed with `UpdateBefore<Clone, Vanilla>` in the
  matching phases so they consume/produce before the vanilla pass
  (repo/TrafficToolEssentials/Mod.cs#L218-L219).
- [Gate-only cross-entity coordination](../how-to/recipes/gate-only-coordination.md)
  (family BA) - `SyncGroupSystem`'s six-state machine coordinates a corridor of
  intersections using only the `m_AvoidSignalGroup` *veto* byte plus a
  force-release deadlock breaker, never seizing a phase
  (repo/TrafficToolEssentials/Systems/TrafficLightSystems/Simulation/SyncGroupSystem.cs#L14-L33,
  repo/TrafficToolEssentials/Systems/TrafficLightSystems/Simulation/SyncGroupSystem.cs#L2123-L2145,
  repo/TrafficToolEssentials/Systems/TrafficLightSystems/Simulation/SyncGroupSystem.cs#L2179-L2188).
- [Custom polygon-zone tool](../how-to/recipes/polygon-zone-tool.md) (family BB) -
  `DepotZoneSystem` does point-in-polygon hit-testing, rejects self-intersecting
  and overlapping polygons at create time, and resolves double-covered depots
  deterministically (below)
  (repo/TrafficToolEssentials/Systems/DepotZone/DepotZoneSystem.cs#L1399-L1462,
  repo/TrafficToolEssentials/Systems/DepotZone/DepotZoneSystem.cs#L525-L530).
- [ECS ISerializable save data](../how-to/recipes/ecs-serializable-savedata.md)
  (family E) - ring-buffer telemetry and every zone/light component implement
  `ISerializable` with independent per-component schema versions and
  corruption-clamp reads (below).
- [Burst IJobChunk jobs](../how-to/recipes/burst-ijobchunk.md) (family S) - the
  patched signal pass is a `[BurstCompile] struct UpdateTrafficLightsJob :
  IJobChunk` scheduled with `ScheduleParallel`, allocating its per-chunk scratch
  with `Allocator.TempJob`
  (repo/TrafficToolEssentials/Systems/TrafficLightSystems/Simulation/PatchedTrafficLightSystem.cs#L31-L35,
  repo/TrafficToolEssentials/Systems/TrafficLightSystems/Simulation/PatchedTrafficLightSystem.cs#L118-L120).
- [UISystemBase / React binding](../how-to/recipes/uisystembase-react-binding.md)
  (family P) - a `partial UISystem : UISystemBase` split across bindings files
  registers Colossal UI value/call bindings for the webpack React frontend
  (repo/TrafficToolEssentials/Systems/UI/UISystem.cs#L21,
  repo/TrafficToolEssentials/Systems/UI/UISystem.UIBindings.cs#L103-L129).

## Key decisions & tradeoffs

- **Narrow, don't disable (and keep an escape hatch).** The default takeover only
  edits query membership, so vanilla and modded intersections coexist in the same
  world and an uninstall leaves the vanilla systems intact. Compatibility mode
  (`SetCompatibilityMode`) is the pressure valve for other traffic-light mods,
  flipping the vanilla systems back on and re-shaping the clone queries so both
  run (repo/TrafficToolEssentials/Mod.cs#L247-L256).
- **Gate, don't drive, for coordination.** Green Wave only ever writes the
  `m_AvoidSignalGroup` veto, so the vanilla state machine remains the single
  authority that commits lane signals. A first-party comment block records that an
  earlier "direct phase copy" approach - main-thread writes of the reference
  light's `m_CurrentSignalGroup`/`m_Timer` onto followers - caused
  Ongoing<->Ending oscillation and permanently-red-but-shown-green lanes, because
  external `TrafficLights` writes bypass the state machine's commit discipline;
  the gate-only path was restored as the only safe design
  (repo/TrafficToolEssentials/Systems/TrafficLightSystems/Simulation/SyncGroupSystem.cs#L2145-L2178).
- **Fix the cross-SystemGroup write hazard by relocating the writer.**
  `SyncGroupSystem.OnUpdate` is deliberately empty; sync is invoked as
  `m_SyncGroupSystem.UpdateSync()` from `PatchedTrafficLightSystem.OnUpdate`
  *before* the signal job is scheduled
  (repo/TrafficToolEssentials/Systems/TrafficLightSystems/Simulation/SyncGroupSystem.cs#L729-L736;
  repo/TrafficToolEssentials/Systems/TrafficLightSystems/Simulation/PatchedTrafficLightSystem.cs#L884-L897).
  The recorded reason: running sync from `UISystem` (a different `SystemGroup`
  that executes after the simulation group) let the signal job read the old
  `m_AvoidSignalGroup`, process, and write the *entire* struct back - clobbering
  the sync write. Ordering the write immediately before the read closes the
  window.
- **Force lookups fresh, don't trust `Update(this)`.** `UpdateSync` re-acquires
  its component lookups via `GetComponentLookup<...>()` every call rather than
  relying on `Update(this)`, with a first-party note that stale chunk data was
  observed when the dependency chain crosses SystemGroups
  (repo/TrafficToolEssentials/Systems/TrafficLightSystems/Simulation/SyncGroupSystem.cs#L513-L519).
- **Deterministic overlap resolution keyed on a stable int.** When two depot zones
  overlap, a depot could fall inside both. The cache rebuild sorts candidate zones
  ascending by `m_ZoneId` (a stable int, not chunk order which is not stable across
  save/load) so the lowest/oldest zone id wins reproducibly; a depot already
  claimed is skipped
  (repo/TrafficToolEssentials/Systems/DepotZone/DepotZoneSystem.cs#L419-L479).
- **Reject bad polygons up front.** `CreateZone` rejects a polygon that overlaps
  any existing zone (vertex-in-polygon both directions plus edge-edge intersection
  via `DoPolygonEdgesIntersect`) and clamps point count to 20; vertex edits are
  refused if they would self-intersect (`WouldPolygonSelfIntersect`, O(n^2)
  non-adjacent edge test)
  (repo/TrafficToolEssentials/Systems/DepotZone/DepotZoneSystem.cs#L518-L530,
  repo/TrafficToolEssentials/Systems/DepotZone/DepotZoneSystem.cs#L1294-L1319,
  repo/TrafficToolEssentials/Systems/DepotZone/DepotZoneSystem.cs#L1496-L1560).
- **Independent, defensive save schemas per component.** Each serialised component
  carries and gates on its own version, and each read clamps against corruption:
  - `CustomTrafficLights` is at schema 9, with `Deserialize` applying every
    incremental `>= N` migration plus a one-shot "stale sequential phases" repair
    for schema <= 8 saves
    (repo/TrafficToolEssentials/Components/CustomTrafficLights.cs#L116-L120,
    repo/TrafficToolEssentials/Components/CustomTrafficLights.cs#L177-L263).
  - `DepotZoneData` is at schema 3 and clamps write index, count, normalized time,
    day counter, and every activity bucket (`MAX_REASONABLE_VEHICLES = 1000`) on
    read (repo/TrafficToolEssentials/Components/DepotZoneData.cs#L241-L307).
  - `VehicleStatsHistoryComponent` stores five `FixedList128Bytes<ushort>` rings,
    clamps every sample to `ushort` range on write, and clamps `WriteIndex`,
    `Count`, and `LastSampleTime` on read
    (repo/TrafficToolEssentials/Components/VehicleStatsHistoryComponent.cs#L27-L31,
    repo/TrafficToolEssentials/Components/VehicleStatsHistoryComponent.cs#L67-L71,
    repo/TrafficToolEssentials/Components/VehicleStatsHistoryComponent.cs#L182-L186).
- **Corrupt-settings recovery.** The `Settings` constructor calls `SetDefaults()`
  first, then wraps `LoadSettings` in try/catch: on a corrupt/incompatible file it
  keeps the defaults and `ApplyAndSave()`s a fresh file rather than throwing
  (repo/TrafficToolEssentials/Settings.cs#L199-L224).

## Pitfalls / upstream-watch

- **Reflected private query-field names are hardcoded.** `SystemSetup` looks up
  `"m_TrafficLightsQuery"` and `"m_TrafficLightQuery"` by string on the vanilla
  systems (repo/TrafficToolEssentials/Mod.cs#L212-L213). If Colossal renames
  either field the reflection returns null and the takeover silently fails - a
  no-throw, no-effect breakage to watch on every CS2 patch.
- **`Allocator.Temp` in the parallel signal job leaks.** The job's per-chunk
  `laneSignals` list must be `Allocator.TempJob`; a first-party comment records
  that `Allocator.Temp` caused a `JobTempAlloc` memory leak in the parallel path,
  which is also why `[BurstCompile]` had to stay on
  (repo/TrafficToolEssentials/Systems/TrafficLightSystems/Simulation/PatchedTrafficLightSystem.cs#L31-L35,
  repo/TrafficToolEssentials/Systems/TrafficLightSystems/Simulation/PatchedTrafficLightSystem.cs#L118-L120).
- **Antivirus false positives.** The mod carries first-party notes that a version
  check via `Type.GetType` was removed because it triggered AV false positives,
  and that Burst codegen is flagged by some scanners though reported clean by
  major ones (repo/TrafficToolEssentials/Mod.cs#L112-L113;
  repo/TrafficToolEssentials/Systems/TrafficLightSystems/Simulation/PatchedTrafficLightSystem.cs#L31-L33).
- **Dispatch ordering comment vs attribute drift.** `Mod.cs` comments the depot
  dispatch system as running "BEFORE `TransportVehicleDispatchSystem`"
  (repo/TrafficToolEssentials/Mod.cs#L237-L238), but the system's own attribute is
  `[UpdateAfter(typeof(TransportVehicleDispatchSystem))]`
  (repo/TrafficToolEssentials/Systems/DepotZone/DepotZoneDispatchSystem.cs#L85).
  The attribute wins; the stale comment is a maintenance hazard, not a live bug.
- **Green-wave deadlocks are papered over, not proven absent.** The state machine
  carries an explicit force-release safety net (release the veto after
  `maxCycleDuration * 3` ticks stuck on one phase, with a grace period)
  precisely because prior deadlocks (`A-1..A-4`, `B-2/3/5`) were found and fixed at
  source; the net exists for the *next* unknown one
  (repo/TrafficToolEssentials/Systems/TrafficLightSystems/Simulation/SyncGroupSystem.cs#L2099-L2145).
  Treat corridor sync as robust-but-not-formally-verified.
- **TCZ subsystem is compiled out.** The Traffic-Control-Zone systems are gated
  behind `#if TCZ_ENABLED`, which is not defined at this pin
  (repo/TrafficToolEssentials/Mod.cs#L130-L138); depot zones are the live
  polygon-tool subsystem, TCZ is frozen source.

Needs Verification (in-game): the actual routing/throughput effect of the green
wave on a live corridor, whether the force-release net ever visibly trips in
normal play, and whether narrow-not-disable fully co-exists with other
traffic-light mods without enabling compatibility mode. None can be confirmed from
static source.

## Source pointers

- Dossier:
  `../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/`
  (`index.md`, `source.md`, `modding.md`, `guide.md`, `notes/`).
- Repo @ `10973595ac9ed37f47ca24eb788d4421dc295fa0` (Release v3.0.7 "Foxglove"),
  key files:
  - `repo/TrafficToolEssentials/Mod.cs` - create + narrow queries + phase order +
    compatibility toggle.
  - `repo/TrafficToolEssentials/Utils/EntityQueryUtils.cs` - reflective
    query-field rewrite.
  - `repo/TrafficToolEssentials/Systems/TrafficLightSystems/Simulation/PatchedTrafficLightSystem.cs`
    - Burst `IJobChunk` signal pass; drives `UpdateSync`; compat query swap.
  - `repo/TrafficToolEssentials/Systems/TrafficLightSystems/Simulation/SyncGroupSystem.cs`
    - six-state gate-only green-wave machine + force-release.
  - `repo/TrafficToolEssentials/Systems/DepotZone/DepotZoneSystem.cs` - polygon
    tool, point-in-polygon, overlap/self-intersection rejection, deterministic
    resolution.
  - `repo/TrafficToolEssentials/Components/{CustomTrafficLights,DepotZoneData,VehicleStatsHistoryComponent}.cs`
    - per-component `ISerializable` schemas + corruption clamps.
  - `repo/TrafficToolEssentials/Systems/UI/UISystem*.cs`,
    `repo/TLEFrontend/` - partial UISystem bindings bridging the webpack React
    frontend.
