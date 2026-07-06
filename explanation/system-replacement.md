---
FrontmatterVersion: 1
DocumentType: Guide
Title: System Replacement
Summary: Why and when a CS2 mod disables (or narrows) a vanilla DOTS system and registers its own in the same phase - the two takeover shapes, the risks, and how to decide - explained against real mod source.
diataxis: explanation
source_version: "~1.5.10f1 (traffic-tool-essentials@1097359; date-pinned, static source only)"
last_reverified: "2026-07-04"
canonical_mods:
  - traffic-tool-essentials@10973595ac9ed37f47ca24eb788d4421dc295fa0
  - realistic-path-finding@50645fa6a078181365e36a42e2e27b96699bf02a
status: source-verified
Created: 2026-07-03
Updated: 2026-07-03
Owners:
  - codex
References:
  - Label: System Scheduling (companion concept)
    Path: ./system-scheduling.md
  - Label: Lifecycle and Initialization
    Path: ./mod-lifecycle.md
  - Label: Recipe - ECS system replacement ordering
    Path: ../how-to/recipes/ecs-system-replacement-ordering.md
  - Label: Technique Index
    Path: ../technique-index.md
---

# System Replacement

Cities: Skylines II runs its simulation as DOTS/ECS systems that fire every frame.
When you want to *change* what a built-in behaviour does - resident pathfinding,
traffic-light phasing, mail accumulation - one option is to take the vanilla system
out of the loop and put your own in its place. This page explains the *concept*: why
you would replace a vanilla system rather than patch it, the two shapes replacement
takes, and the risks that come with owning a slice of the simulation.

It is a concept page. The step-by-step mechanics live in the how-to companion
[`recipes/ecs-system-replacement-ordering.md`](../how-to/recipes/ecs-system-replacement-ordering.md);
the phase/ordering model it builds on lives in
[System Scheduling](./system-scheduling.md). This page does not repeat those steps.

## Why replace a system at all

CS2 gives you three broad ways to change stock behaviour:

- **Harmony method patching** - splice a prefix/postfix into a single method. Precise,
  but brittle for whole-behaviour changes: you patch internal methods Colossal can
  rename or Burst-compile out of reach, and two mods patching the same method collide.
- **Prefab / component data edits** - change the inputs the vanilla system reads. Works
  when the behaviour is fully data-driven, but many behaviours are logic, not data.
- **System replacement** - disable (or narrow) the vanilla system and schedule your own
  in its phase. Reach for this when the behaviour is owned by an identifiable
  `GameSystemBase` and you can express your version as its own system.

Replacement is the heavy option. You take ownership of a whole simulation slice, and
with it the responsibility to keep that slice correct across CS2 patches. Choose it
when the behaviour is too tangled for a method patch and too much logic to express as
data.

## The two shapes of replacement

Both shapes resolve the vanilla target with `GetOrCreateSystemManaged<T>()` first, so
the system is guaranteed to exist before you disable or order against it (see
[System Scheduling](./system-scheduling.md) for why ordering only works within one
`SystemUpdatePhase`).

### 1. Full disable + register a replacement

You turn the vanilla system off entirely and schedule your clone into the same phase.
This says: *"I own this behaviour for every entity it used to service."*

Realistic Path Finding replaces resident AI wholesale. In `OnLoad` it disables four
vanilla systems, then schedules its own systems into `SystemUpdatePhase.GameSimulation`
([`realistic-path-finding` `repo/RealisticPathFinding/Mod.cs#L54-L58`](../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Mod.cs), commit `50645fa`):

```csharp
// Disable original systems
World.DefaultGameObjectInjectionWorld.GetOrCreateSystemManaged<Game.Simulation.ResidentAISystem>().Enabled = false;
World.DefaultGameObjectInjectionWorld.GetOrCreateSystemManaged<Game.Simulation.ResidentAISystem.Actions>().Enabled = false;
World.DefaultGameObjectInjectionWorld.GetOrCreateSystemManaged<Game.Simulation.TripNeededSystem>().Enabled = false;
World.DefaultGameObjectInjectionWorld.GetOrCreateSystemManaged<Game.Simulation.ResourceBuyerSystem>().Enabled = false;
```

Setting `Enabled = false` on a managed system stops it from updating without destroying
it, so the type still exists to be ordered against or re-enabled later. The replacements
(`RPFResidentAISystem` and siblings) are decompiled copies of the Game.dll systems, which
is what makes full disable powerful *and* expensive - see the risks below.

### 2. Narrow the vanilla query instead of disabling

You leave the vanilla system running but shrink the set of entities it acts on, so it
skips the ones you tag, and run your system alongside it. This says: *"vanilla keeps its
untouched entities; I only own the ones I marked."*

Traffic Tool Essentials does not disable the vanilla traffic-light systems by default.
It rebuilds their `EntityQuery` to exclude entities carrying its own
`CustomTrafficLights` marker, then schedules its patched systems *before* the vanilla
ones ([`traffic-tool-essentials` `repo/TrafficToolEssentials/Mod.cs#L209-L213`, `#L218-L219`](../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Mod.cs), commit `1097359`):

```csharp
var noneList = new NativeList<ComponentType>(1, Allocator.Temp);
noneList.Add(ComponentType.ReadOnly<Components.CustomTrafficLights>());

Utils.EntityQueryUtils.UpdateEntityQuery(m_TrafficLightInitializationSystem, "m_TrafficLightsQuery", noneList);
Utils.EntityQueryUtils.UpdateEntityQuery(m_TrafficLightSystem, "m_TrafficLightQuery", noneList);
noneList.Dispose();
```

The `noneList` becomes the query's "none" filter - the same idea as `Exclude<CustomTrafficLights>`.
Because vanilla never gets hard-disabled, TTE can offer a **runtime compatibility toggle**
that flips both the vanilla and patched systems on and off without a reload
([`repo/TrafficToolEssentials/Mod.cs#L247-L256`](../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Mod.cs), commit `1097359`):

```csharp
public static void SetCompatibilityMode(bool enable)
{
    m_TrafficLightInitializationSystem.Enabled = enable;
    m_TrafficLightSystem.Enabled = enable;

    m_PatchedTrafficLightInitializationSystem.SetCompatibilityMode(enable);
    m_PatchedTrafficLightSystem.SetCompatibilityMode(enable);
    // ...
}
```

That live toggle is only possible *because* nothing was permanently disabled at load.

## How to choose

| Question | Full disable | Query-narrowing |
| --- | --- | --- |
| Do you own the behaviour for *every* entity? | Yes, all-or-nothing | No, only tagged entities |
| Can a user revert at runtime? | No (needs reload) | Yes (flip marker / compatibility mode) |
| How does a second mod on the same system fare? | Hard conflict | Coexists per-entity |
| Bisection when something breaks | Harder (behaviour fully replaced) | Easier (remove your marker) |
| Maintenance cost | High (clone drifts from vanilla) | Lower (vanilla still runs its own logic) |

Full disable is the right call when the behaviour genuinely has to be replaced for all
entities and there is no clean per-entity boundary. Query-narrowing is preferable
whenever you *can* express "the ones I own" as a marker component, because it keeps
vanilla in charge of everything else and gives users a safe revert.

## The risks of owning a slice

- **All-or-nothing conflicts.** Two mods that both disable the same vanilla system
  cannot coexist - the second one to load wins, or both misbehave. Full-disable takeovers
  are the least compatible option. RPF's own dossier flags exactly this hazard for
  `ResidentAISystem`.
- **Decompiled-clone drift.** A full replacement is usually a copy of the vanilla system's
  logic. Every CS2 patch that changes that vanilla system silently diverges from your
  clone; you must re-diff and re-port it. This is the standing cost of full disable.
- **No loud failure.** Harmony throws when a patched signature changes. Pure ECS
  replacement has no patch site to guard, so a CS2 layout change can break your behaviour
  with no exception - the city just runs wrong. `Needs Verification (in-game)`: the exact
  symptoms of a broken replacement vary per system and only show up at runtime.
- **Live toggles need a re-sync.** Flipping systems on and off at runtime (TTE
  `SetCompatibilityMode`) does not retroactively fix entities the disabled system skipped
  while it was off; expect to force a re-evaluation when ownership changes live.
- **Ordering hazards.** `UpdateBefore`/`UpdateAfter` only order systems *within the same
  phase*. Put your replacement in the wrong `SystemUpdatePhase` and it runs in the wrong
  part of the frame - reading stale data or clobbering shared state - with no compile
  error. See [System Scheduling](./system-scheduling.md).

## See also

- How-to: [ECS system replacement via ordering](../how-to/recipes/ecs-system-replacement-ordering.md)
  - the step-by-step mechanics, including a third "bracket + re-assert" shape.
- Concept: [System Scheduling](./system-scheduling.md) - phases, ordering verbs, and why
  `GetOrCreateSystemManaged<T>` must resolve the target first.
- Concept: [Lifecycle and Initialization](./mod-lifecycle.md) - where the disable step
  sits inside `OnLoad`.
- Reference: [Technique Index](../technique-index.md) - families B (replacement via
  ordering), M (disable/replace), and T (query rewriting).
