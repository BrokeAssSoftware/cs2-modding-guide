---
FrontmatterVersion: 1
DocumentType: Guide
Title: System Scheduling
Summary: How CS2 mods slot DOTS systems into the simulation loop with UpdateAt/UpdateBefore/UpdateAfter across SystemUpdatePhase, and tune cadence with GetUpdateInterval/GetUpdateOffset, explained against real mod source.
diataxis: explanation
source_version: "~1.5.10f1 (traffic-tool-essentials@1097359; date-pinned, static source only)"
last_reverified: "2026-07-04"
canonical_mods:
  - traffic-tool-essentials@10973595ac9ed37f47ca24eb788d4421dc295fa0
  - magic-mail@6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47
  - road-speed-adjuster@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed
  - realistic-path-finding@50645fa6a078181365e36a42e2e27b96699bf02a
  - outside-traffic-adjuster@42afd29638267c9dc116b040515ceaf41eb9a9e1
status: source-verified
Created: 2026-07-01
Updated: 2026-07-05
Owners:
  - codex
References:
  - Label: Lifecycle and Initialization (companion concept)
    Path: ./mod-lifecycle.md
  - Label: Recipe - ECS system replacement ordering
    Path: ../how-to/recipes/ecs-system-replacement-ordering.md
  - Label: Recipe - Periodic UpdateInterval system
    Path: ../how-to/recipes/periodic-updateinterval-system.md
  - Label: Technique Index
    Path: ../technique-index.md
---

# System Scheduling

Cities: Skylines II runs on DOTS/ECS. Your mod's behaviour lives in *systems*, and a
system does nothing until it is placed into the game's update loop. That placement happens
through the `UpdateSystem` scheduler the game hands you in `OnLoad` (see
[Lifecycle and Initialization](./mod-lifecycle.md)).

This page explains the scheduling model: the phases you can attach to, the three attachment
verbs, how to order against vanilla systems, and how to tune how *often* a system runs. It
is a concept page - the "why the loop is shaped this way" - not a step-by-step recipe.

## The scheduler API

Three methods place a system, all taking a `SystemUpdatePhase`:

- `updateSystem.UpdateAt<TSystem>(phase)` - run `TSystem` during `phase`.
- `updateSystem.UpdateBefore<TSystem, TOther>(phase)` - run `TSystem` before `TOther`
  within `phase`.
- `updateSystem.UpdateAfter<TSystem, TOther>(phase)` - run `TSystem` after `TOther` within
  `phase`.

There is also a single-generic overload, `UpdateBefore<TSystem>(phase)` /
`UpdateAfter<TSystem>(phase)`, which orders relative to the phase boundary rather than a
named system. MagicMail uses that form to sit at the front of `GameSimulation`
([`magic-mail` `repo/Mod.cs#L113-L114`](../../vice-and-order-research/mods/dossiers/magic-mail/repo/Mod.cs), commit `6fb3d2b`):

```csharp
updateSystem.UpdateBefore<MagicMailSystem>(SystemUpdatePhase.GameSimulation);
updateSystem.UpdateBefore<MailCapacitySystem>(SystemUpdatePhase.GameSimulation);
```

The two-generic overloads are how you express a hard "my system must see the world *before*
/ *after* this specific vanilla or mod system touched it" constraint.

## The phases in our corpus

`SystemUpdatePhase` is an ordered set of slots that fire each frame. The phases actually
used across our source-verified mods, roughly in loop order, are:

| Phase | When it runs / what it is for | Cited real usage |
| --- | --- | --- |
| `Deserialize` | During save/load, to read your data back into ECS. | Road Speed Adjuster save-data system ([`repo/Mod.cs#L36`](../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/Mod.cs)) |
| `ToolUpdate` | Tool interaction (raycasts, apply-on-click) each frame. | Road Speed Adjuster tool ([`repo/Mod.cs#L39`](../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/Mod.cs)); TTE tool ([`repo/Mod.cs#L222`](../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Mod.cs)) |
| `Modification4B` / `ModificationEnd` | The "modification" band where net/prefab edits are committed; `ModificationEnd` is the tail of it. | Outside Traffic Adjuster spawn-rate editor at `ModificationEnd` ([`repo/Mod.cs#L29`](../../vice-and-order-research/mods/dossiers/outside-traffic-adjuster/repo/Mod.cs)); Road Speed Adjuster apply + clear at `ModificationEnd` ([`repo/Mod.cs#L42-L45`](../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/Mod.cs)); RPF turn-cost `UpdateBefore ... LanesModifiedSystem` in `ModificationEnd` ([`repo/Mod.cs#L70`](../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Mod.cs)); TTE init `UpdateBefore ... TrafficLightInitializationSystem` in `Modification4B` ([`repo/Mod.cs#L218`](../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Mod.cs)) |
| `GameSimulation` | The main city simulation tick - most gameplay logic lives here. | MagicMail two systems ([`repo/Mod.cs#L113-L114`](../../vice-and-order-research/mods/dossiers/magic-mail/repo/Mod.cs)); RPF's many AI/cost systems ([`repo/Mod.cs#L62-L78`](../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Mod.cs)); TTE patched light system ([`repo/Mod.cs#L219`](../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Mod.cs)) |
| `UIUpdate` | Feeding data to the UI layer (info panels, overlays' data side). | Road Speed Adjuster info-section UI ([`repo/Mod.cs#L48`](../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/Mod.cs)); TTE UI system ([`repo/Mod.cs#L221`](../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Mod.cs)) |
| `Rendering` | Frame rendering - direct mesh / overlay draw. | Road Speed Adjuster speed-limit render system ([`repo/Mod.cs#L51`](../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/Mod.cs)) |

TTE also uses `UITooltip` for its tooltip system
([`repo/Mod.cs#L220`](../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Mod.cs), commit `1097359`).
The full `SystemUpdatePhase` enum has more members than our corpus exercises; treat any
phase you have not seen used in real source as `Needs Verification` before relying on it.

Road Speed Adjuster is the clearest single illustration because one mod spreads six systems
across five phases in a way that mirrors a system's data lifecycle - load (`Deserialize`),
interact (`ToolUpdate`), commit (`ModificationEnd`), surface (`UIUpdate`), draw (`Rendering`)
([`road-speed-adjuster` `repo/Mod.cs#L36-L51`](../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/Mod.cs), commit `e0c0c0b`).

## Choosing a phase

The phase encodes *when in the frame* your data is valid and safe to touch:

- Put **read-back / restore** logic in `Deserialize` so ECS is populated before simulation
  runs.
- Put **player tool interaction** in `ToolUpdate`.
- Put **edits to net/prefab/lane data** in the `Modification*` band, and order *before* the
  vanilla system that consumes those edits (see below).
- Put **ongoing gameplay logic** in `GameSimulation`.
- Put **UI data feeds** in `UIUpdate` and **draw calls** in `Rendering`.

## Ordering: disable-and-replace vs. run-alongside

Two distinct patterns show up, and they answer different needs.

**Disable then replace.** When you want your system to *supersede* a vanilla one, disable
the vanilla system in `OnLoad` (`GetOrCreateSystemManaged<T>().Enabled = false`) and
schedule your replacement into the same phase. Realistic Path Finding disables
`ResidentAISystem`, `ResidentAISystem.Actions`, `TripNeededSystem`, and `ResourceBuyerSystem`,
then schedules `RPFResidentAISystem`, `RPFResidentActionsSystem`, `RPFTripNeededSystem`, and
`RPFResourceBuyerSystem` in `GameSimulation`
([`repo/Mod.cs#L54-L78`](../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Mod.cs), commit `50645fa`).
It even chains its own replacements against each other and against a vanilla group:

```csharp
updateSystem.UpdateAt<RPFResidentAISystem>(SystemUpdatePhase.GameSimulation);
updateSystem.UpdateAfter<RPFResidentActionsSystem, RPFResidentAISystem>(SystemUpdatePhase.GameSimulation);
updateSystem.UpdateAfter<Game.Simulation.UpdateGroupSystem, RPFResidentActionsSystem>(SystemUpdatePhase.GameSimulation);
```

**Run alongside, ordered against vanilla.** Sometimes you do not disable the vanilla system
- you insert yours immediately before it so your edits are in place when it runs. Traffic
Tool Essentials schedules its patched systems *before* the vanilla ones rather than
disabling them outright, and toggles the vanilla pair on/off at runtime via a compatibility
switch
([`repo/Mod.cs#L218-L219`, `#L247-L256`](../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Mod.cs), commit `1097359`):

```csharp
updateSystem.UpdateBefore<PatchedTrafficLightInitializationSystem, Game.Net.TrafficLightInitializationSystem>(SystemUpdatePhase.Modification4B);
updateSystem.UpdateBefore<PatchedTrafficLightSystem, Game.Simulation.TrafficLightSystem>(SystemUpdatePhase.GameSimulation);
```

Realistic Path Finding shows the same "order-before-a-named-vanilla-system" idea in the
modification band, placing its turn-cost system before `Game.Pathfind.LanesModifiedSystem`
([`repo/Mod.cs#L70`](../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Mod.cs)).

Both patterns rely on `GetOrCreateSystemManaged<T>` so the vanilla/target system is
guaranteed to exist before you order against or disable it. The decision and its trade-offs
are the subject of [`recipes/ecs-system-replacement-ordering.md`](../how-to/recipes/ecs-system-replacement-ordering.md).

### Attribute-based ordering against a vanilla producer

The `updateSystem` calls in `OnLoad` are not the only way to order. A system can also carry
`[UpdateInGroup]` plus `[UpdateAfter]`/`[UpdateBefore]` attributes on the class itself to
slot into a group after a *named vanilla producer*. TTE's `DepotZoneDispatchSystem` runs in
`SimulationSystemGroup` and marks itself to run after the vanilla
`TransportVehicleDispatchSystem`, so it observes dispatch/pathfinding output for the same tick
([`traffic-tool-essentials` `repo/TrafficToolEssentials/Systems/DepotZone/DepotZoneDispatchSystem.cs#L84-L86`](../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Systems/DepotZone/DepotZoneDispatchSystem.cs), commit `1097359`):

```csharp
[UpdateInGroup(typeof(SimulationSystemGroup))]
[UpdateAfter(typeof(TransportVehicleDispatchSystem))]  // run AFTER pathfinding + dispatch
public partial class DepotZoneDispatchSystem : GameSystemBase
```

This is the read-side mirror of the `UpdateBefore` pattern above: rather than inserting
*before* a consumer to pre-stage edits, it inserts *after* a producer to intercept freshly
produced data. Prefer the `UpdateSystem` calls when you want scheduling centralised in
`OnLoad`; class attributes are handy when a system's ordering is intrinsic to it and should
travel with the type.

### Ordering across SystemGroups: the write-after-write hazard

`UpdateBefore`/`UpdateAfter` only order systems *within the same phase / SystemGroup*. They
say nothing about a system whose work is actually driven from a *different* group - and that
gap is a real trap. Traffic Tool Essentials hit it: its `SyncGroupSystem` originally ran its
`UpdateSync()` work from `UISystem` (the UI SystemGroup, which executes *after*
`SimulationSystemGroup`). The vanilla-patched traffic-light job in `SimulationSystemGroup`
read the old `m_AvoidSignalGroup`, processed it, and wrote the *entire* struct back -
clobbering the value `UpdateSync` had written a group earlier. Classic write-after-write.

The fix is to drive the write from the *consuming* system's `OnUpdate` so it lands before the
job reads. `SyncGroupSystem.OnUpdate` is left intentionally empty
([`repo/TrafficToolEssentials/Systems/TrafficLightSystems/Simulation/SyncGroupSystem.cs#L729-L736`](../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Systems/TrafficLightSystems/Simulation/SyncGroupSystem.cs)),
and `PatchedTrafficLightSystem.OnUpdate` calls `UpdateSync()` first
([`repo/TrafficToolEssentials/Systems/TrafficLightSystems/Simulation/PatchedTrafficLightSystem.cs#L884-L897`](../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Systems/TrafficLightSystems/Simulation/PatchedTrafficLightSystem.cs), commit `1097359`):

```csharp
// PatchedTrafficLightSystem.OnUpdate - run GW sync BEFORE scheduling the light job
if (m_SyncGroupSystem != null)
{
    m_SyncGroupSystem.UpdateSync();   // writes m_AvoidSignalGroup with a fresh lookup
}
// ...then schedule the traffic-light job, which now reads the value just written.
```

The lesson: phase-relative ordering is a *within-group* guarantee. When a producer's write
and a consumer's read live in different SystemGroups, order them by calling the producer
directly from the consumer, not by scheduler verbs - `UpdateBefore` across groups will not
save you.

## Tuning cadence: GetUpdateInterval and GetUpdateOffset

Placing a system in `GameSimulation` runs it every simulation tick. That is wasteful for
work that only needs to happen periodically. A system controls its own cadence by overriding
two methods:

- `GetUpdateInterval(SystemUpdatePhase phase)` - ticks between runs.
- `GetUpdateOffset(SystemUpdatePhase phase)` - phase offset, used to stagger a system into a
  distinct slot relative to others.

CS2's simulation uses a day quantized into **262144** ticks. So `262144 / N` means "run N
times per in-game day." MagicMail's periodic system runs 32 times per day
([`magic-mail` `repo/Systems/MagicMailSystem.cs#L57-L67`](../../vice-and-order-research/mods/dossiers/magic-mail/repo/Systems/MagicMailSystem.cs), commit `6fb3d2b`):

```csharp
private const int UpdatesPerDay = 32;   // once per ~45 in-game minutes

public override int GetUpdateInterval(SystemUpdatePhase phase)
{
#if DEBUG
    return 256;                 // vanilla PostFacilityAISystem interval, for debugging
#else
    return 262144 / UpdatesPerDay;   // 8192 ticks
#endif
}

public override int GetUpdateOffset(SystemUpdatePhase phase)
{
    return 48;   // safe slot before the vanilla system
}
```

The offset of `48` deliberately places the run in a slot ahead of the vanilla facility
system it cooperates with (`#L71-L75`). The `#if DEBUG` branch dropping to `256` (matching
vanilla `PostFacilityAISystem`) shows a practical tactic: run frequently while debugging,
throttle in release.

Contrast this with a **one-shot** system. MagicMail's `MailCapacitySystem` returns
`GetUpdateInterval => 1` and starts with `Enabled = false`, only enabling itself when
settings change or a city loads, then disabling itself again after it runs once
([`magic-mail` `repo/Systems/MailCapacitySystem.cs#L61-L85`](../../vice-and-order-research/mods/dossiers/magic-mail/repo/Systems/MailCapacitySystem.cs), commit `6fb3d2b`).
This "enable -> run once -> disable" idiom is how apply-on-change work avoids polling every
tick. The full pattern is written up in
[`recipes/periodic-updateinterval-system.md`](../how-to/recipes/periodic-updateinterval-system.md).

## Caching, gating, and diagnostics

These are properties of well-behaved systems that make scheduling pay off:

- **Cache lookups in `OnCreate`.** Build `EntityQuery`, `ComponentLookup`, and
  `SystemHandle` handles once in `OnCreate` rather than per tick. MagicMail builds its
  facility query and resolves the vanilla `MailAccumulationSystem` in `OnCreate`
  ([`repo/Systems/MagicMailSystem.cs#L79-L106`](../../vice-and-order-research/mods/dossiers/magic-mail/repo/Systems/MagicMailSystem.cs)).
- **Gate with `RequireForUpdate<T>()`.** A system with an unmet requirement is skipped
  entirely, which is cheaper than early-returning inside `OnUpdate`. MagicMail requires its
  post-facilities query (`#L100`); `MailCapacitySystem` requires both its prefab queries
  ([`repo/Systems/MailCapacitySystem.cs#L58-L59`](../../vice-and-order-research/mods/dossiers/magic-mail/repo/Systems/MailCapacitySystem.cs)).
- **Diagnostics behind a flag.** Prefer a debug-logging toggle over always-on verbose logs.
  TTE gates verbose logging behind a settings flag with `LogDebug` versus `LogInfo`
  ([`repo/Mod.cs#L27-L46`](../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Mod.cs), commit `1097359`).

## Common pitfalls

- **Scheduling in the wrong phase.** Editing lane/net data outside the `Modification` band,
  or ordering *after* the vanilla consumer, means your edits are read stale (or a frame late).
- **Polling every tick.** Apply-on-change work belongs in an enable/run-once/disable
  one-shot, not a `GetUpdateInterval => 1` system left permanently enabled.
- **Ordering against a system that does not exist yet.** Always resolve targets with
  `GetOrCreateSystemManaged<T>` before disabling or ordering against them.
- **Trusting `UpdateBefore`/`UpdateAfter` across SystemGroups.** Those verbs order only
  within a phase/group. If your producer's write is driven from a different group than the
  consumer's read (e.g. UI-group work feeding a simulation-group job), you can still get a
  write-after-write clobber; call the producer from the consumer's `OnUpdate` instead.
- **Assuming an unverified phase behaves a certain way.** Only the phases enumerated above
  are source-confirmed in our corpus; label others `Needs Verification`.

## See also

- [Lifecycle and Initialization](./mod-lifecycle.md) - where scheduling sits
  inside `OnLoad`, and the disable-vanilla-systems step.
- Recipes: [ECS system replacement ordering](../how-to/recipes/ecs-system-replacement-ordering.md),
  [Periodic UpdateInterval system](../how-to/recipes/periodic-updateinterval-system.md).
- [Technique Index](../technique-index.md) - coverage ledger for these techniques.
