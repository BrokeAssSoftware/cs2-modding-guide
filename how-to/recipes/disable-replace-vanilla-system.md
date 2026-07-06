---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: Disable / replace a vanilla system"
recipe: disable-replace-vanilla-system
technique_family: "M - Disable / replace a vanilla system"
diataxis: how-to
source_version: "1.6.0f1 (realistic-path-finding@50645fa; date-pinned, static source only)"
last_reverified: "2026-07-04"
status: source-verified
canonical_mods:
  - realistic-path-finding@50645fa6a078181365e36a42e2e27b96699bf02a
  - traffic-tool-essentials@10973595ac9ed37f47ca24eb788d4421dc295fa0
technique_applicability: [simulation]
Created: 2026-07-04
Updated: 2026-07-04
Owners:
  - codex
---

# Disable / replace a vanilla system

> Take a vanilla ECS simulation system out of the update loop and run your own
> re-implementation in its place - by flipping `Enabled = false` on the resolved
> vanilla system and scheduling your replacement into the same update phase.

## Problem
You want to change *behaviour* that lives inside a vanilla simulation system, not a
prefab field or a component value - for example how residents choose trips, how
agents buy resources, or how pathfinding weights are applied. The logic is baked into
a compiled `GameSystemBase` you cannot edit. Rewriting a few methods with Harmony is
brittle when the system is large; the cleaner move is to stop the vanilla system from
running at all and provide your own system that produces the equivalent (or improved)
output in the same slot of the frame.

## Solution
Resolve the target vanilla system through the managed world
(`GetOrCreateSystemManaged<T>()`), set `Enabled = false` so it stays instantiated but
never ticks, then register your replacement with `UpdateAt<T>(phase)` in the **same**
`SystemUpdatePhase` the vanilla system ran in. `Enabled = false` is the key: it does
not destroy the system (other systems and queries that reference it still resolve), it
just skips its `OnUpdate`. Your replacement inherits the responsibility for whatever
downstream state the vanilla system used to write. This is an all-or-nothing swap - the
whole vanilla system goes dark - so see Variations for the softer "narrow its query"
alternative when you need runtime compatibility toggles.

This recipe is the how-to companion to the concept page
[system-replacement](../../explanation/system-replacement.md); read that for *why* the
approach works and its compatibility model. This page is the mechanics.

## Steps & Code

### 1. Disable the vanilla systems you are replacing

Resolve each target by type and flip `Enabled` off. Realistic Path Finding takes out
four resident/economy systems unconditionally in `OnLoad`:

```csharp
// Disable original systems
World.DefaultGameObjectInjectionWorld.GetOrCreateSystemManaged<Game.Simulation.ResidentAISystem>().Enabled = false;
World.DefaultGameObjectInjectionWorld.GetOrCreateSystemManaged<Game.Simulation.ResidentAISystem.Actions>().Enabled = false;
World.DefaultGameObjectInjectionWorld.GetOrCreateSystemManaged<Game.Simulation.TripNeededSystem>().Enabled = false;
World.DefaultGameObjectInjectionWorld.GetOrCreateSystemManaged<Game.Simulation.ResourceBuyerSystem>().Enabled = false;
```
Source: `../../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Mod.cs#L53-L58` (@50645fa6a078181365e36a42e2e27b96699bf02a)

`GetOrCreateSystemManaged<T>()` resolves the system if it already exists and creates it
otherwise, so this is safe even if the vanilla system has not been touched yet.
`Enabled = false` leaves the instance in the world - it is skipped, not removed.

### 2. Write your replacement as a `GameSystemBase`

Each replacement is an ordinary system. It must reproduce the ECS writes the vanilla
system was responsible for, since nothing else does now:

```csharp
public partial class RPFResidentAISystem : GameSystemBase
```
Source: `../../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Systems/RPFResidentAISystem.cs#L54` (@50645fa6a078181365e36a42e2e27b96699bf02a)

The mod ships one replacement per disabled system: `RPFResidentAISystem`,
`RPFResidentActionsSystem`, `RPFTripNeededSystem`, `RPFResourceBuyerSystem`.

### 3. Schedule the replacement into the same phase the vanilla system ran in

Register with `UpdateAt<T>(phase)` so your system ticks where the vanilla one used to.
The four disabled systems all ran in `GameSimulation`, so the replacements go there:

```csharp
updateSystem.UpdateAt<RealisticPathFinding.Systems.RPFResidentAISystem>(SystemUpdatePhase.GameSimulation);
updateSystem.UpdateAfter<RealisticPathFinding.Systems.RPFResidentActionsSystem, RealisticPathFinding.Systems.RPFResidentAISystem>(SystemUpdatePhase.GameSimulation);
// ...
updateSystem.UpdateAt<RealisticPathFinding.Systems.RPFTripNeededSystem>(SystemUpdatePhase.GameSimulation);
updateSystem.UpdateAt<RealisticPathFinding.Systems.RPFResourceBuyerSystem>(SystemUpdatePhase.GameSimulation);
```
Source: `../../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Mod.cs#L62-L78` (@50645fa6a078181365e36a42e2e27b96699bf02a)

Note `UpdateAfter<Replacement, Anchor>` is used where the original had an ordering
relationship (the actions system runs after the AI system). Matching the vanilla
phase and relative order is what keeps downstream systems seeing the same data
lifecycle. For the ordering rules in depth see
[ecs-system-replacement-ordering](ecs-system-replacement-ordering.md).

## Pitfalls & gotchas

- **This is all-or-nothing - the whole vanilla system goes dark.** Once
  `Enabled = false`, *every* effect of that system stops, including edge cases and
  interactions you did not re-implement. Your replacement now owns 100% of that
  system's output contract. If two mods both disable-and-replace the same vanilla
  system, they collide: the second replacement is simply another enabled system in the
  phase, and both write competing state. There is no partial hand-off with this
  technique - if you need a runtime on/off or coexistence with the vanilla behaviour,
  use the query-narrowing variant below instead.

- **Decompiled-clone drift.** A replacement is typically a decompiled-and-modified copy
  of the vanilla system, frozen at the game version it was written against. When the
  base game patches the original `ResidentAISystem` / `ResourceBuyerSystem` /
  `TripNeededSystem`, your clone does not get those fixes - it keeps running the old
  logic while the rest of the simulation moves on. Re-verify replacements every game
  version; this is the maintenance cost the technique trades for control.

- **Order and phase must match the original, or downstream breaks.** Scheduling your
  replacement in the wrong `SystemUpdatePhase` (or before/after the wrong anchor) means
  systems that consumed the vanilla output now read stale or missing data for a frame.
  RPF deliberately mirrors the vanilla phase (`GameSimulation`) and re-creates the
  AI -> Actions ordering.

- **`GetOrCreateSystemManaged` creates if absent - so disabling always "works" but may
  no-op silently.** If you name a system that no longer exists (renamed in a patch), you
  do not get an error at this call - you get a freshly created, empty, disabled system
  and the real one keeps running. Verify the vanilla type name exists at your target
  game version before trusting the disable.

- **Runtime toggling of a replacement is not shown in the canonical source.** RPF
  disables unconditionally (a commented-out `if (!realisticTripsMod)` guard exists in
  source but is inactive). Whether flipping `Enabled` back and forth mid-session cleanly
  restores vanilla behaviour is **Needs Verification (in-game)**.

## Variations

- **Narrow the vanilla query instead of disabling the whole system (soft, toggleable).**
  Traffic Tool Essentials does *not* hard-disable the vanilla traffic-light systems. It
  resolves them, then rewrites their `EntityQuery` to exclude entities the mod manages,
  so vanilla keeps running for everything else:

  ```csharp
  m_TrafficLightInitializationSystem = m_World.GetOrCreateSystemManaged<Game.Net.TrafficLightInitializationSystem>();
  m_TrafficLightSystem = m_World.GetOrCreateSystemManaged<Game.Simulation.TrafficLightSystem>();
  // ...
  var noneList = new NativeList<ComponentType>(1, Allocator.Temp);
  noneList.Add(ComponentType.ReadOnly<Components.CustomTrafficLights>());
  Utils.EntityQueryUtils.UpdateEntityQuery(m_TrafficLightInitializationSystem, "m_TrafficLightsQuery", noneList);
  Utils.EntityQueryUtils.UpdateEntityQuery(m_TrafficLightSystem, "m_TrafficLightQuery", noneList);
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Mod.cs#L122-L213` (@10973595ac9ed37f47ca24eb788d4421dc295fa0)

  The helper reads the vanilla system's private query field by reflection, rebuilds it
  with an added `WithNone(...)`, and writes it back:

  ```csharp
  public static void UpdateEntityQuery(SystemBase systemBase, string fieldName, NativeList<ComponentType> none)
  {
      EntityQuery query = GetEntityQuery(systemBase, fieldName);
      EntityQuery newQuery = GetEntityQueryBuilder(query, none).Build(systemBase);
      SetEntityQuery(systemBase, fieldName, newQuery);
  }
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Utils/EntityQueryUtils.cs#L21-L26` (@10973595ac9ed37f47ca24eb788d4421dc295fa0)

  This is a fundamentally softer swap: the vanilla system still processes every entity
  *without* the mod's marker component (`CustomTrafficLights`), so unmanaged
  intersections stay fully vanilla and the mod can hand an intersection back simply by
  removing its marker - a runtime compatibility toggle the disable technique cannot
  offer. The full mechanics (query descs, marker-driven partitioning) are in
  [ecs-query-rewriting](ecs-query-rewriting.md).

- **Conditional disable based on another mod.** RPF's source contains a
  `realisticTripsMod` detection and a commented `if (!realisticTripsMod)` around the
  `ResourceBuyerSystem` disable + replacement - a pattern for skipping a swap when a
  co-operating mod already owns that behaviour. In the pinned source it is inactive
  (the disable runs unconditionally), so treat conditional disable as a shape to copy,
  not a proven runtime path here.

## See also
- Concept: [system-replacement](../../explanation/system-replacement.md) (why disabling
  vs. narrowing, and the compatibility model - do not duplicate here).
- Related recipes: [ecs-query-rewriting](ecs-query-rewriting.md) (the query-narrowing
  alternative), [ecs-system-replacement-ordering](ecs-system-replacement-ordering.md)
  (getting the phase/order right).
- Case study: [realistic-path-finding](../../case-studies/realistic-path-finding.md).

## Sources
- Canonical mods (dossier + repo):
  - `realistic-path-finding` @50645fa6a078181365e36a42e2e27b96699bf02a - `repo/RealisticPathFinding/Mod.cs`, `repo/RealisticPathFinding/Systems/RPFResidentAISystem.cs`
  - `traffic-tool-essentials` @10973595ac9ed37f47ca24eb788d4421dc295fa0 - `repo/TrafficToolEssentials/Mod.cs`, `repo/TrafficToolEssentials/Utils/EntityQueryUtils.cs`
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding
