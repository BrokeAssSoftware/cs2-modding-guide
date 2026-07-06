---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: Register a system into the update loop"
recipe: system-registration
technique_family: "Foundational - system registration & scheduling (underlies families B, L, M)"
diataxis: how-to
source_version: "~1.5.2f1 (road-speed-adjuster@e0c0c0b; date-pinned, static source only)"
last_reverified: "2026-07-04"
canonical_mods:
  - road-speed-adjuster@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed
  - magic-mail@6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47
technique_applicability: [core, simulation]
status: source-verified
Created: 2026-07-03
Updated: 2026-07-03
Owners:
  - codex
---

# Register a system into the update loop

> Attach your DOTS systems to the CS2 simulation loop from `OnLoad` using
> `updateSystem.UpdateAt / UpdateBefore / UpdateAfter<T>(SystemUpdatePhase.X)`.

## Problem

You have written a `GameSystemBase` (or `SystemBase`) but it never runs. In Cities:
Skylines II a system does nothing until it is *placed* into the update loop. Merely
defining the class, or even creating it with `World.GetOrCreateSystemManaged<T>()`,
does not schedule its `OnUpdate`. You need to register it against a
`SystemUpdatePhase` - and often against a specific neighbouring system - during your
mod's `OnLoad`.

This recipe is the task-level "how do I wire a system in." For the *why* behind the
phases, cadence tuning, and the ordering model, read the concept page
[System Scheduling](../../explanation/system-scheduling.md); this page does not
re-explain it.

## Solution

The game passes an `UpdateSystem` scheduler into `IMod.OnLoad(UpdateSystem updateSystem)`.
Three verbs place a system, each taking a `SystemUpdatePhase`:

- `updateSystem.UpdateAt<TSystem>(phase)` - run `TSystem` during `phase`.
- `updateSystem.UpdateBefore<TSystem, TOther>(phase)` - run `TSystem` before `TOther`
  within `phase`. There is also a single-generic `UpdateBefore<TSystem>(phase)` that
  orders relative to the *front* of the phase rather than a named neighbour.
- `updateSystem.UpdateAfter<TSystem, TOther>(phase)` - run `TSystem` after `TOther`
  within `phase` (and a single-generic `UpdateAfter<TSystem>(phase)`).

Pick the phase for *when in the frame* the work is valid, then use the two-generic
forms only when you must sit before/after a specific system that reads or writes the
same data. Call these once, in `OnLoad`.

## Steps & Code

### 1. Register each system in its phase (Road Speed Adjuster)

Road Speed Adjuster registers six distinct systems across five phases from `OnLoad`,
one line each, mirroring a feature's data lifecycle: load -> interact -> commit ->
surface -> draw. This is the plain `UpdateAt<T>(phase)` form:

```csharp
public void OnLoad(UpdateSystem updateSystem)
{
    // ... settings / localization setup ...

    // Register save/load system (must be early)
    updateSystem.UpdateAt<RoadSpeedSaveDataSystem>(SystemUpdatePhase.Deserialize);

    // Register the road speed tool system
    updateSystem.UpdateAt<RoadSpeedToolSystem>(SystemUpdatePhase.ToolUpdate);

    // Apply speeds at ModificationEnd
    updateSystem.UpdateAt<RoadSpeedApplySystem>(SystemUpdatePhase.ModificationEnd);
    updateSystem.UpdateAt<ClearCustomSpeedsSystem>(SystemUpdatePhase.ModificationEnd);

    // InfoSection systems need the UIUpdate phase
    updateSystem.UpdateAt<RoadSpeedToolUISystem>(SystemUpdatePhase.UIUpdate);

    // Direct mesh rendering, no overlay buffer
    updateSystem.UpdateAt<SpeedLimitRenderSystem>(SystemUpdatePhase.Rendering);
}
```
Source: `../../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/Mod.cs#L36-L51` (@e0c0c0b)

Two systems (`RoadSpeedApplySystem`, `ClearCustomSpeedsSystem`) share the same phase
(`ModificationEnd`) - registering multiple systems into one phase is normal; when the
relative order between two of *your own* systems matters, use `UpdateAfter`/`UpdateBefore`
(step 2) instead of relying on registration order.

### 2. Order relative to a phase boundary (Magic Mail)

When you do not need to name a specific neighbour but *do* want to sit at the front of
a phase, use the single-generic `UpdateBefore<TSystem>(phase)`. Magic Mail places both
its simulation systems near the start of `GameSimulation`:

```csharp
// GameSimulation systems:
// - MailCapacitySystem: one-shot capacity update when sliders change
// - MagicMailSystem: slow, periodic magic top-ups + overflow cleanup
updateSystem.UpdateBefore<MagicMailSystem>(SystemUpdatePhase.GameSimulation);
updateSystem.UpdateBefore<MailCapacitySystem>(SystemUpdatePhase.GameSimulation);
```
Source: `../../../vice-and-order-research/mods/dossiers/magic-mail/repo/Mod.cs#L113-L114` (@6fb3d2b)

### 3. Order against a named vanilla or mod system

To guarantee your system sees the world *before* (or *after*) a specific other system
touched it, use the two-generic overloads with the neighbour as the second type
argument. This is the mechanism behind whole-system takeovers - see
[ECS system replacement via ordering](ecs-system-replacement-ordering.md) for the
disable-and-replace variant, which chains its own replacements with
`UpdateAfter<RPFResidentActionsSystem, RPFResidentAISystem>(SystemUpdatePhase.GameSimulation)`.
The shape is:

```csharp
// Run MySystem before a specific vanilla system, within the same phase
updateSystem.UpdateBefore<MyMod.Core.MySystem, Game.Simulation.SomeVanillaSystem>(SystemUpdatePhase.GameSimulation);
```
`Needs Verification`: substitute the real vanilla system type for your case - a wrong
type name fails silently at scheduling time, not at compile time, if the type does not
exist in the loaded `Game.dll`.

### 4. Register one system into several phases (multi-phase)

The same `T` can be registered more than once. That is a distinct pattern with its own
concept page - when one object owns state that spans load/save/gameplay, register it
into `Deserialize`, `Serialize`, and a gameplay phase. See
[Multi-Phase Scheduling](../../explanation/multi-phase-scheduling.md) for when to reach
for it and when to prefer separate systems (as Road Speed Adjuster does above).

## Pitfalls & gotchas

- **Registration is required - creation is not enough.** A class that exists, or a
  system fetched via `GetOrCreateSystemManaged<T>`, still never ticks until you call an
  `UpdateAt/Before/After` for it. Registration is what schedules `OnUpdate`.
- **`UpdateBefore`/`UpdateAfter` only order *within one phase*.** They do not move a
  system across phases. Picking the wrong `SystemUpdatePhase` silently runs your work in
  the wrong part of the frame (edits read a frame stale, UI feeds run before data is
  ready) with no exception. Choose the phase first; order second.
- **Only some phases are source-confirmed.** The phases used across the verified corpus
  are `Deserialize`, `Serialize`, `ToolUpdate`, `Modification4B` / `ModificationEnd`,
  `GameSimulation`, `UIUpdate`, `Rendering` (and `UITooltip`). The full
  `SystemUpdatePhase` enum has more members; treat any you have not seen used as
  `Needs Verification` before relying on it. See
  [System Scheduling](../../explanation/system-scheduling.md) for the cited phase table.
- **Ordering against a system that does not exist yet.** If you plan to disable or order
  against a vanilla system, resolve it with `GetOrCreateSystemManaged<T>()` first so it
  is guaranteed instantiated before you reference it.
- **Registration order is not a contract.** Two systems in the same phase are not
  guaranteed to run in the order you registered them; if the order matters, state it
  explicitly with `UpdateAfter`/`UpdateBefore`.
- **Do not re-register per frame.** These calls belong in `OnLoad` (once), not in
  `OnUpdate`.

## Variations

- **`UpdateAt<T>(phase)`** - simplest; use when nothing else in the phase depends on your
  order (Road Speed Adjuster, step 1).
- **`UpdateBefore<T>(phase)` single-generic** - sit at the front of a phase without naming
  a neighbour (Magic Mail, step 2).
- **`UpdateBefore/After<T, TOther>(phase)` two-generic** - hard ordering against a named
  vanilla or mod system (step 3); the basis of
  [ECS system replacement via ordering](ecs-system-replacement-ordering.md).
- **Same `T`, multiple phases** - multi-phase registration for one stateful object
  ([Multi-Phase Scheduling](../../explanation/multi-phase-scheduling.md)).

## See also

- Concept (read for the "why"): [System Scheduling](../../explanation/system-scheduling.md),
  [Multi-Phase Scheduling](../../explanation/multi-phase-scheduling.md).
- Related recipes: [System skeleton (GameSystemBase / SystemBase)](system-template.md)
  (what you register), [ECS system replacement via ordering](ecs-system-replacement-ordering.md)
  (ordering to take over vanilla behaviour), `burst-ijobchunk` (queued - the parallel
  job body a registered system schedules).
- Reference: [Technique Index](../../technique-index.md).

## Sources

- Canonical mods (dossier + repo, pinned commits):
  - `road-speed-adjuster` @e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed
  - `magic-mail` @6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47
- Official/community references (link out, do not duplicate):
  - Unity DOTS / ECS systems: https://docs.unity3d.com/Packages/com.unity.entities@latest
  - CS2 modding wiki (system update phases): https://cs2.paradoxwikis.com/Modding
