---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Case study: Realistic Path Finding"
case_study: realistic-path-finding
mod: "Realistic Path Finding (121226)"
dossier: ../../vice-and-order-research/mods/dossiers/realistic-path-finding/
repo_commit: 50645fa6a078181365e36a42e2e27b96699bf02a
source_version: "n/a - no pinned source"
last_reverified: "2026-07-04"
diataxis: explanation
techniques: [B, M, G, S]
technique_applicability: [simulation, core]
status: source-verified
Created: 2026-07-01
Updated: 2026-07-01
Owners:
  - codex
---

# Realistic Path Finding - case study

> A pathfinding overhaul that disables four vanilla simulation systems, replaces
> them with decompiled clones threaded with slider-driven cost heuristics, and
> layers Burst congestion sampling plus reflection-based mod interop on top - a
> full tour of how far a code mod can go without owning the whole path graph.

All code claims below are cited to `repo/<path>#Lxx` at pinned commit
`50645fa6a078181365e36a42e2e27b96699bf02a` (branch `master`, userModVersion
0.9.4 / modVersion 25, "Updated for game version 1.6.0f1"), surfaced through the
dossier at `../../vice-and-order-research/mods/dossiers/realistic-path-finding/`.
This dossier corrected earlier fabrications (a non-existent
`PathfindQueueDiagnosticsSystem`, a "Diagnostics" settings group, and all CSV
telemetry) and a removed `PedestrianDensityPenaltySystem`; none of those exist at
this commit and none are reintroduced here.

## What it does / why it's instructive

Realistic Path Finding (RPF, author ruzbeh0, Paradox ModId 121226) recalibrates
how citizens choose routes in Cities: Skylines II. Instead of nudging a few
weights, it takes over the resident AI decision loop and threads a large surface
of tunable cost factors - transit waits, transfer penalties, stop crowding,
per-mode in-vehicle comfort, walking and biking attractiveness, congestion
feedback, turn/hierarchy bias, and taxi surge pricing - into the perceived-time
cost model the game uses to pick paths. Every factor is exposed as an in-game
slider, so the same machinery serves both a "transit realism" and a "throughput"
playstyle.

It is instructive because it is a maximal example of the **system-replacement**
pattern under real maintenance pressure. Rather than Harmony-patching the vanilla
resident AI, RPF disables it outright and schedules its own decompiled copy - and
then demonstrates the full spectrum of supporting techniques a serious code mod
needs around such a swap: disabling and re-registering vanilla systems in a
specific update order, Burst-compiled jobs that feed live telemetry back into the
path graph, reflection bridges to an optional companion mod, and a small,
surgical set of Harmony postfixes reserved only for the seams that cannot be
reached cleanly through system registration. It is a good teaching artifact
precisely because it shows the tradeoff cost of going this far: four decompiled
`Game.dll` clones (~5.6k lines in the resident AI alone) that must be re-diffed
after every CS2 patch.

## Architecture at a glance

### The load-time swap

`Mod.OnLoad` runs the whole setup in one place. It first probes the mod manager
for a companion asset named `Time2Work` and logs the detected time factor
(repo/RealisticPathFinding/Mod.cs#L43-L51). It then **disables four vanilla
systems** by setting `Enabled = false` on the managed instances:
`ResidentAISystem`, `ResidentAISystem.Actions`, `TripNeededSystem`, and
`ResourceBuyerSystem` (repo/RealisticPathFinding/Mod.cs#L54-L58). Note the
`if (!realisticTripsMod)` guard around the `ResourceBuyerSystem` disable is
commented out, so RPF now takes over resource buying unconditionally even when
Realistic Trips is loaded (repo/RealisticPathFinding/Mod.cs#L57-L58).

It then schedules the replacement and auxiliary systems into
`SystemUpdatePhase.GameSimulation` using `UpdateAt` / `UpdateAfter`
(repo/RealisticPathFinding/Mod.cs#L62-L78):

- `RPFResidentAISystem` (`UpdateAt`), with `RPFResidentActionsSystem` scheduled
  `UpdateAfter` it, and `Game.Simulation.UpdateGroupSystem` after that
  (repo/RealisticPathFinding/Mod.cs#L62-L65).
- Cost systems: `WalkSpeedUpdaterSystem`, `CarTurnAndHierarchyBiasSystem`,
  `PedestrianWalkCostFactorSystem`, `PedestrianCrosswalkCostFactorSystem`,
  `TaxiStandCrowdingSystem`, `CarCongestionEwmaSystem`,
  `BicycleOwnerLimiterSystem` (repo/RealisticPathFinding/Mod.cs#L68-L75).
- The other two vanilla clones: `RPFTripNeededSystem` and
  `RPFResourceBuyerSystem` (repo/RealisticPathFinding/Mod.cs#L76-L78).
- One system is deliberately in a different phase:
  `CarPrefabTurnCostFactorSystem` is scheduled with
  `UpdateBefore<..., Game.Pathfind.LanesModifiedSystem>` in
  `SystemUpdatePhase.ModificationEnd`, so prefab turn costs land before the graph
  rebuild consumes them (repo/RealisticPathFinding/Mod.cs#L70).

Finally it applies every Harmony patch in the assembly with
`harmony.PatchAll(...)` and logs each patched method
(repo/RealisticPathFinding/Mod.cs#L82-L90). Harmony (Lib.Harmony 2.2.2) is a
real, load-bearing dependency here, not incidental
(repo/RealisticPathFinding/RealisticPathFinding.csproj#L84).

### The replacement systems

`RPFResidentAISystem` is a decompiled copy of Colossal's resident AI with
targeted edits: it injects slider values (`crowdness_factor`,
`scheduled_wt_factor`, `transfer_penalty`, `feeder_trunk_transfer_penalty`,
`crowdness_stop_threashold`, walk/bike comfort and ramp params, `choice_tau_sec`,
`waiting_time_factor`) plus the cached Time2Work factor into the resident tick job
(repo/RealisticPathFinding/Systems/RPFResidentAISystem.cs#L284-L303).
`RPFResidentActionsSystem` mirrors the vanilla action consumer so the custom AI
queue still drains through the same service-fee, statistics, and transport-usage
writers, keeping the swap compatible with the rest of the stock pipeline
(repo/RealisticPathFinding/Systems/RPFResidentActionSystem.cs#L32). (Filename is
singular `RPFResidentActionSystem.cs`; the class is the plural
`RPFResidentActionsSystem`.) `RPFTripNeededSystem` and `RPFResourceBuyerSystem`
are likewise full decompiled clones of their `Game.dll` counterparts.

### Burst congestion and turn-bias jobs

`CarCongestionEwmaSystem` runs on a light cadence -
`GetUpdateInterval` returns `262144 / 64` (~7.5 in-game minutes)
(repo/RealisticPathFinding/Systems/CarCongestionEwmaSystem.cs#L111-L115). It
pre-allocates persistent buffers - a `NativeQueue<Sample>` and two 4096-entry
`NativeParallelHashMap`s - in `OnCreate`
(repo/RealisticPathFinding/Systems/CarCongestionEwmaSystem.cs#L81-L83). A
Burst-compiled job (`[BurstCompile] partial struct SampleJob : IJobEntity`,
repo/RealisticPathFinding/Systems/CarCongestionEwmaSystem.cs#L257-L258)
accumulates per-vehicle time on the current lane and emits a per-lane
travel-time sample on lane change. The main thread then aggregates samples,
derives freeflow from lane length and speed limit, and updates an **asymmetric
EWMA** (`alpha` up, `alpha * 0.7` down for slower recovery), converts the
`ewma / freeflow` ratio plus a global `car_mode_weight - 1` bias into a clamped
density add, and writes it as a step-limited delta (`maxStep 0.08`) into the path
graph's lane density
(repo/RealisticPathFinding/Systems/CarCongestionEwmaSystem.cs#L203-L249). Config
is re-read from `Mod.m_Setting` each update
(repo/RealisticPathFinding/Systems/CarCongestionEwmaSystem.cs#L130-L135).

`CarTurnAndHierarchyBiasSystem` is event-driven and holds two more Burst jobs:
`TurnScanJob` derives a turn angle from Bezier tangents and maps it to seconds
(capped at `base_turn_penalty`), and `LaneScanJob` adds hierarchy density to
non-highway lanes by speed-limit bucket
(repo/RealisticPathFinding/Systems/CarTurnAndHierarchyBiasSystem.cs#L302-L303,
repo/RealisticPathFinding/Systems/CarTurnAndHierarchyBiasSystem.cs#L344-L345).
Both commit results as **non-stacking deltas** against cached prior adjustments
(repo/RealisticPathFinding/Systems/CarTurnAndHierarchyBiasSystem.cs#L233-L262).

> Needs Verification: the task classified these as "Burst IJobChunk" (technique
> family S). At this commit the actual job types are `[BurstCompile] IJobEntity`
> (`SampleJob`, `TurnScanJob`, `LaneScanJob`), not `IJobChunk`. The Burst
> IJobChunk recipe still applies as the technique-family reference, but the
> concrete implementation here uses the source-generated `IJobEntity` form.

### Time2Work reflection interop

`Time2WorkInterop.GetFactor` reflects over `AppDomain.CurrentDomain` assemblies
for the type `Time2Work.Time2WorkTimeSystem`, reads the public static
`timeReductionFactor` field (or falls back to deriving from `kTicksPerDay` vs
`TimeSystem.kTicksPerDay`), and caches the result; when the mod is absent the
factor stays `1.0` (repo/RealisticPathFinding/Time2WorkInterop.cs#L13-L50). This
lets RPF align perceived travel time with a faster/slower day-night pacing mod
with no compile-time dependency and no hard requirement.

### Data flow / phases summary

- `ModificationEnd`: `CarPrefabTurnCostFactorSystem` writes prefab-side turn/U-turn
  cost components before `LanesModifiedSystem` rebuilds the graph
  (repo/RealisticPathFinding/Mod.cs#L70).
- `GameSimulation`: the resident AI clone plus all live cost systems run; several
  are event-driven (re-enabled on `onSettingsApplied`, run one pass, disable) so
  slider edits apply mid-session without a restart.
- Path-query time: Harmony postfixes on `PathUtils` add the bus-lane second
  penalty per query; a `TransportLineSystem.OnUpdate` postfix scales route
  durations by per-mode comfort.

## Techniques demonstrated

- [ECS system replacement & ordering](../how-to/recipes/ecs-system-replacement-ordering.md)
  (family B) - `Mod.OnLoad` schedules the RPF systems into explicit phases with
  `UpdateAt` / `UpdateAfter` / `UpdateBefore`, including the deliberate
  `ModificationEnd` placement of the prefab turn-cost system before
  `LanesModifiedSystem` (repo/RealisticPathFinding/Mod.cs#L62-L78).
- [Disable / replace a vanilla system](../how-to/recipes/disable-replace-vanilla-system.md)
  (family M) - four vanilla simulation systems are hard-disabled via
  `Enabled = false` and superseded by decompiled RPF clones that reuse the
  vanilla action queue to stay compatible downstream
  (repo/RealisticPathFinding/Mod.cs#L54-L58;
  repo/RealisticPathFinding/Systems/RPFResidentActionSystem.cs#L32).
- [Reflection mod bridges](../how-to/recipes/reflection-mod-bridges.md) (family G) -
  `Time2WorkInterop.GetFactor` detects and reads an optional companion mod purely
  through reflection, with a cached fallback of `1.0` when absent
  (repo/RealisticPathFinding/Time2WorkInterop.cs#L13-L50;
  repo/RealisticPathFinding/Mod.cs#L43-L51).
- [Burst IJobChunk jobs](../how-to/recipes/burst-ijobchunk.md) (family S) -
  `[BurstCompile]` jobs sample per-agent telemetry off the main thread and feed
  step-limited deltas back into the path graph
  (repo/RealisticPathFinding/Systems/CarCongestionEwmaSystem.cs#L257-L258,
  repo/RealisticPathFinding/Systems/CarCongestionEwmaSystem.cs#L203-L249;
  repo/RealisticPathFinding/Systems/CarTurnAndHierarchyBiasSystem.cs#L302-L345).
  See the Needs-Verification note above: the concrete job type is `IJobEntity`,
  not `IJobChunk`.

Supporting technique also on display (not one of the four headline families):
selective Harmony postfixes on `PathUtils.GetCarDriveSpecification` (two
overloads) and `PathUtils.GetTaxiDriveSpecification`, adding
`nonbus_buslane_penalty_sec` only on `CarLaneFlags.PublicOnly` lanes and applying
the **same** penalty to taxis (taxis are not exempt)
(repo/RealisticPathFinding/Patches/BusLanePatches.cs#L22-L97).

## Key decisions & tradeoffs

- **Clone, don't patch, the resident AI.** RPF disables the vanilla resident
  systems and ships full decompiled copies rather than Harmony-patching the tick
  job. This buys complete control over the trip heuristics but creates a large
  per-patch diff surface: `RPFResidentAISystem` is ~5.6k lines, and
  `RPFTripNeededSystem` / `RPFResourceBuyerSystem` are also full clones, so every
  CS2 update demands re-diffing against fresh `Game.dll` sources
  (repo/RealisticPathFinding/Systems/RPFResidentAISystem.cs#L284-L303).
- **Reserve Harmony for the seams that ECS cannot reach.** Turn/U-turn costs are
  applied on the *prefab* side via `CarPrefabTurnCostFactorSystem` specifically to
  avoid Harmony prefixes on the Burst-compiled `PathUtils.GetCarDriveSpecification`
  rebuild path, while the bus-lane penalty is a *path-side* Harmony postfix on the
  same `PathUtils` family (repo/RealisticPathFinding/Mod.cs#L70;
  repo/RealisticPathFinding/Patches/BusLanePatches.cs#L42-L97). The rule the mod
  encodes: mutate prefab cost components for graph-rebuild-time costs, reserve
  Harmony for per-query path costs.
- **Delta-safe graph writes.** Congestion, hierarchy, and pedestrian cost systems
  cache their prior adjustment and apply only the delta, so repeated updates and
  live slider edits never stack raw values onto the path graph
  (repo/RealisticPathFinding/Systems/CarCongestionEwmaSystem.cs#L239-L247;
  repo/RealisticPathFinding/Systems/CarTurnAndHierarchyBiasSystem.cs#L233-L262).
- **Event-driven live application.** Most cost systems subscribe to
  `onSettingsApplied`, set `Enabled = true`, run one pass, then disable - the
  mod's standard idiom for applying slider changes on the next tick without a
  save restart.
- **Optional interop over hard dependency.** Time2Work integration is
  reflection-based and degrades to a `1.0` factor when the mod is missing, so RPF
  ships with zero declared dependencies while still coordinating pacing when the
  companion is present (repo/RealisticPathFinding/Time2WorkInterop.cs#L13-L50).
- **Asymmetric EWMA smoothing.** Congestion recovers more slowly than it builds
  (`alphaDown = alpha * 0.7`), a deliberate choice to keep congested corridors
  "sticky" rather than snapping back to freeflow
  (repo/RealisticPathFinding/Systems/CarCongestionEwmaSystem.cs#L205-L208).

## Pitfalls / upstream-watch

- **Decompiled-clone drift.** The four vanilla clones will desync from service
  fees / stat writers if the game changes those systems and the clones are not
  re-diffed; schedule a diff audit after every CS2 patch
  (repo/RealisticPathFinding/Systems/RPFResidentAISystem.cs#L284-L303).
- **Harmony signature fragility.** The bus-lane patches bind by explicit
  parameter-type arrays on three `PathUtils` overloads; if Colossal changes a
  signature the patch silently fails to bind and the penalty stops applying
  (repo/RealisticPathFinding/Patches/BusLanePatches.cs#L42-L97).
- **Lane-flag dependency.** The bus-lane penalty requires
  `CarLaneFlags.PublicOnly`; any mod that rewrites lane flags can neutralize it
  (repo/RealisticPathFinding/Patches/BusLanePatches.cs#L27).
- **Resident-AI stack collisions.** Because RPF hard-disables the vanilla
  resident systems, a second resident-AI mod will very likely conflict
  (repo/RealisticPathFinding/Mod.cs#L54-L56).
- **Pedestrian multi-writer conflict (Lazy Pedestrians).** RPF restores
  pedestrian costs from its own cached view, so another mod editing the same
  prefab cost or live-graph edge after RPF's capture can be overwritten on a
  later RPF reapply. The repo's own `docs/pedestrian-cost-systems.md` documents
  this limitation first-party; `disable_ped_cost = true` restores baseline and is
  the documented escape hatch but is not a guaranteed multi-writer fix
  (repo/docs/pedestrian-cost-systems.md#L47-L56).
- **Memory pressure.** The congestion sampler pre-allocates persistent 4096-entry
  hash maps and a queue; profile allocations on population-heavy saves
  (repo/RealisticPathFinding/Systems/CarCongestionEwmaSystem.cs#L81-L83).
- **Dead seam - `ScaleWaitingTimesSystem`.** It is fully implemented but its
  `Mod.OnLoad` registration is commented out, so it never runs; live wait scaling
  actually happens inside `RPFRouteUtils.StripTransportSegments`. Treat it as a
  teaching example, not a live capability
  (repo/RealisticPathFinding/Systems/ScaleWaitingTimeSystem.cs#L12-L37;
  repo/RealisticPathFinding/Mod.cs#L61).
- **Unconditional resource-buyer takeover.** The `if (!realisticTripsMod)` guard
  is commented out, so RPF disables vanilla `ResourceBuyerSystem` and runs its own
  clone even alongside Realistic Trips - watch for interaction effects
  (repo/RealisticPathFinding/Mod.cs#L57-L58).

Needs Verification (requires the running game, cannot be confirmed from source):
the exact magnitude of routing change per slider, and whether
`disable_ped_cost = true` fully isolates the Lazy Pedestrians conflict.

## Source pointers

- Dossier:
  `../../vice-and-order-research/mods/dossiers/realistic-path-finding/`
  (`index.md`, `source.md`, `modding.md`, `guide.md`, `notes/`).
- Repo @ `50645fa6a078181365e36a42e2e27b96699bf02a` (branch `master`), key files:
  - `repo/RealisticPathFinding/Mod.cs` - disable + schedule + Harmony PatchAll.
  - `repo/RealisticPathFinding/Systems/RPFResidentAISystem.cs`,
    `.../RPFResidentActionSystem.cs` - decompiled resident AI clone + action drain.
  - `repo/RealisticPathFinding/Systems/CarCongestionEwmaSystem.cs`,
    `.../CarTurnAndHierarchyBiasSystem.cs` - Burst jobs + delta graph writes.
  - `repo/RealisticPathFinding/Time2WorkInterop.cs` - reflection interop.
  - `repo/RealisticPathFinding/Patches/BusLanePatches.cs`,
    `.../RPFPatches.cs` - Harmony postfixes.
  - `repo/docs/pedestrian-cost-systems.md` - first-party multi-writer conflict note.
