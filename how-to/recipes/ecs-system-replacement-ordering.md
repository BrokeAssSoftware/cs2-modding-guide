---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: ECS system replacement via ordering"
recipe: ecs-system-replacement-ordering
technique_family: "B - ECS system replacement via ordering"
diataxis: how-to
source_version: "~1.5.10f1 (traffic-tool-essentials@1097359; date-pinned, static source only)"
last_reverified: "2026-07-04"
canonical_mods:
  - traffic-tool-essentials@10973595ac9ed37f47ca24eb788d4421dc295fa0
  - realistic-path-finding@50645fa6a078181365e36a42e2e27b96699bf02a
  - advanced-road-naming@559e72cdb3180e3e71869643367e094a22eed988
technique_applicability: [core, simulation]
status: source-verified
Created: 2026-07-01
Updated: 2026-07-01
Owners:
  - codex
---

# ECS system replacement via ordering

> Take over a vanilla simulation behavior by disabling (or narrowing) the stock
> ECS system and scheduling your own system before/after it - no Harmony.

## Problem

You want to change what a built-in Cities: Skylines II simulation system does -
traffic-light phasing, resident pathfinding, road naming - but the behavior lives
inside a Colossal `GameSystemBase` that runs every frame. Harmony method patching
(technique family C, [harmony-price-getter-postfix](harmony-price-getter-postfix.md))
can splice into a single method, but for a whole-system takeover it is brittle:
you patch internal methods that Colossal can rename or Burst-compile out of reach,
and two mods patching the same method collide hard.

Reach for ECS ordering instead when: (a) the behavior is owned by an identifiable
system you can disable or whose `EntityQuery` you can narrow, and (b) you can
express your replacement as its own system scheduled at a known `SystemUpdatePhase`.

## Solution

The game runs its systems through an `UpdateSystem` scheduler keyed by
`SystemUpdatePhase`. Two levers give you a Harmony-free takeover:

1. **Turn the vanilla system off** - either fully
   (`World.GetOrCreateSystemManaged<T>().Enabled = false`) or *partially* by
   narrowing its `EntityQuery` so it skips the entities you own
   (`Exclude<YourMarker>`).
2. **Schedule your system relative to vanilla** with
   `updateSystem.UpdateBefore/UpdateAfter<Custom, Vanilla>(SystemUpdatePhase.X)`
   (or `UpdateAt` in the same phase), so your writes land in the right order.

Full disable = "I own this behavior entirely." Query-narrowing = "vanilla keeps
its untouched entities; I only own the ones I tagged." Writing through vanilla
APIs (e.g. `NameSystem.SetCustomName`) instead of poking components directly keeps
you compatible with the rest of the engine.

## Steps & Code

### 1. Full disable + ordered replacement (Realistic Path Finding)

RPF replaces resident AI wholesale. In `OnLoad` it disables four vanilla systems,
then schedules its clones into `SystemUpdatePhase.GameSimulation`:

```csharp
// Disable original systems
World.DefaultGameObjectInjectionWorld.GetOrCreateSystemManaged<Game.Simulation.ResidentAISystem>().Enabled = false;
World.DefaultGameObjectInjectionWorld.GetOrCreateSystemManaged<Game.Simulation.ResidentAISystem.Actions>().Enabled = false;
World.DefaultGameObjectInjectionWorld.GetOrCreateSystemManaged<Game.Simulation.TripNeededSystem>().Enabled = false;
//if (!realisticTripsMod)
World.DefaultGameObjectInjectionWorld.GetOrCreateSystemManaged<Game.Simulation.ResourceBuyerSystem>().Enabled = false;

updateSystem.UpdateAt<RealisticPathFinding.Systems.RPFResidentAISystem>(SystemUpdatePhase.GameSimulation);
updateSystem.UpdateAfter<RealisticPathFinding.Systems.RPFResidentActionsSystem, RealisticPathFinding.Systems.RPFResidentAISystem>(SystemUpdatePhase.GameSimulation);
```
Source: `../../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Mod.cs#L53-L63` (@50645fa)

Note the ordering intent: `RPFResidentActionsSystem` runs `UpdateAfter` the AI
system so the custom AI's queue is drained by the action consumer in the same
tick. RPF also uses `UpdateBefore` against a vanilla anchor for a prefab-side pass:

```csharp
updateSystem.UpdateBefore<RealisticPathFinding.Systems.CarPrefabTurnCostFactorSystem, Game.Pathfind.LanesModifiedSystem>(SystemUpdatePhase.ModificationEnd);
```
Source: `../../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Mod.cs#L70` (@50645fa)

### 2. Partial takeover via query-narrowing (Traffic Tool Essentials)

TTE does *not* disable the vanilla traffic-light systems by default. Instead it
narrows their `EntityQuery` to exclude entities carrying its own
`CustomTrafficLights` marker, then runs its Burst systems `UpdateBefore` the
vanilla ones. Vanilla keeps handling untagged intersections; TTE owns the rest:

```csharp
var noneList = new NativeList<ComponentType>(1, Allocator.Temp);
noneList.Add(ComponentType.ReadOnly<Components.CustomTrafficLights>());

Utils.EntityQueryUtils.UpdateEntityQuery(m_TrafficLightInitializationSystem, "m_TrafficLightsQuery", noneList);
Utils.EntityQueryUtils.UpdateEntityQuery(m_TrafficLightSystem, "m_TrafficLightQuery", noneList);
noneList.Dispose();

updateSystem.UpdateBefore<...PatchedTrafficLightInitializationSystem, Game.Net.TrafficLightInitializationSystem>(SystemUpdatePhase.Modification4B);
updateSystem.UpdateBefore<...PatchedTrafficLightSystem, Game.Simulation.TrafficLightSystem>(SystemUpdatePhase.GameSimulation);
```
Source: `../../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Mod.cs#L209-L219` (@1097359) (system-type args elided; see file)

Because it never hard-disables vanilla, TTE can offer a runtime **compatibility
mode** that flips both the vanilla and patched systems on/off without a reload:

```csharp
public static void SetCompatibilityMode(bool enable)
{
    m_TrafficLightInitializationSystem.Enabled = enable;
    m_TrafficLightSystem.Enabled = enable;

    m_PatchedTrafficLightInitializationSystem.SetCompatibilityMode(enable);
    m_PatchedTrafficLightSystem.SetCompatibilityMode(enable);
    ...
}
```
Source: `../../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Mod.cs#L247-L256` (@1097359)

### 3. Bracket the vanilla producer + write via a vanilla API (Advanced Road Naming)

ARN never disables anything. CS2 recomputes connected-road names in
`AggregateSystem`; ARN **brackets** that producer at `ModificationEnd` with two of
its own systems: `RoadAggregateProtectionSystem` runs `UpdateBefore` the producer
(defending mod-managed aggregates from being torn down) and `SegmentMetadataSystem`
runs `UpdateAfter` it (re-asserting the custom names), writing them through the
vanilla `NameSystem` rather than mutating name components directly:

```csharp
updateSystem.UpdateAt<SegmentMetadataSystem>(SystemUpdatePhase.Deserialize);
updateSystem.UpdateAt<SegmentMetadataSystem>(SystemUpdatePhase.Serialize);
updateSystem.UpdateAfter<SegmentMetadataSystem, AggregateSystem>(SystemUpdatePhase.ModificationEnd);
updateSystem.UpdateBefore<RoadAggregateProtectionSystem, AggregateSystem>(SystemUpdatePhase.ModificationEnd);
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Mod.cs#L54-L57` (@559e72c)

The name is applied through the vanilla API on the real owner entity:

```csharp
var safeName = string.IsNullOrWhiteSpace(finalName) ? $"Road Segment {nameEntity.Index}" : finalName.Trim();
_nameSystem.SetCustomName(nameEntity, safeName);
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Systems/SegmentMetadataSystem.cs#L1828-L1829` (@559e72c)
(`_nameSystem = World.GetOrCreateSystemManaged<NameSystem>();` at `SegmentMetadataSystem.cs#L90`; clear via `SetCustomName(edge, null)` at `#L1859`.)

Because the vanilla producer runs every frame and can merge/recompute aggregates
back, ARN pairs the ordering with a read-back verification + re-apply loop (the
"self-healing" pattern) rather than assuming one write sticks.

## Pitfalls & gotchas

- **Query-narrowing (`Exclude<Marker>`) vs full disable.** Narrowing
  (TTE, `Mod.cs#L209-L219`) leaves vanilla running for everything you did not tag,
  so untouched entities keep stock behavior and you get a clean bisection path
  (flip your marker / compatibility mode). Full disable (RPF, `Mod.cs#L53-L58`)
  is all-or-nothing: any entity that system used to service is now yours, and a
  second mod that also disables/patches that same system will conflict
  (RPF's dossier flags exactly this for `ResidentAISystem`).
- **Ordering hazards.** `UpdateBefore`/`UpdateAfter` only orders systems *within
  the same phase*; picking the wrong `SystemUpdatePhase` silently runs your system
  in the wrong part of the frame. TTE moved its Green Wave sync call out of the UI
  group into the traffic-light system's own update because the later UI phase let
  the traffic-light job clobber shared state (write-after-write hazard) - a
  documented behavioral bug, not a compile error. `Needs Verification (in-game)`:
  the exact frame-timing symptoms of a mis-ordered phase.
- **Decompiled-clone drift.** RPF's replacements (`RPFResidentAISystem`,
  `RPFTripNeededSystem`, `RPFResourceBuyerSystem`) are full decompiled copies of
  Game.dll systems (~5.6k lines for the AI); every CS2 patch requires re-diffing
  them against vanilla. Full-disable takeover buys control at the cost of a large
  per-patch maintenance surface.
- **No patch site to guard game changes.** With Harmony you at least fail loudly
  when a signature changes; with pure ordering, a CS2 layout/aggregate change can
  break behavior with no exception. ARN's dossier calls this out - its mitigation
  is the verification + post-load re-apply loops.
- **Save/load & recompute.** Writing through vanilla APIs (`NameSystem.SetCustomName`)
  keeps engine-owned state consistent, but engine recompute (AggregateSystem) can
  overwrite you after a load; ARN re-asserts names via a deferred post-load pass.
  Custom marker/state components must be serializable if the takeover has to
  survive a save (TTE persists `CustomTrafficLights`; ARN persists its metadata).
- **Compatibility toggle needs a re-sync.** Flipping systems at runtime (TTE
  `SetCompatibilityMode`) does not retroactively fix entities the disabled system
  skipped; TTE additionally forces a node update so ECS re-converges. Expect to
  trigger a re-evaluation when you flip ownership live.

## Variations

- **Full disable** (RPF) - you own the behavior for all entities; simplest to
  reason about, heaviest to maintain, least compatible.
- **Query-narrowing with a marker component** (TTE, `Exclude<CustomTrafficLights>`)
  - co-exist with vanilla per-entity; enables a runtime revert. See the sibling
  recipe [ecs-query-rewriting](ecs-query-rewriting.md) (family T) for the query
  mechanics.
- **Bracket + re-assert through a vanilla API** (ARN) - don't disable at all;
  order around the producer and repair after it via `UpdateAfter` + a
  stability/re-apply loop. Best when the vanilla output is a recomputed value you
  cannot own outright.
- **Mixed with selective Harmony** - RPF uses ECS ordering for the big swap but
  keeps three small Harmony postfixes (bus-lane / taxi cost) for method-level
  seams that have no clean system to replace. Ordering and Harmony are not
  mutually exclusive.

## See also

- Related recipes:
  [harmony-price-getter-postfix](harmony-price-getter-postfix.md) (family C - the
  method-patching alternative and contrast),
  [ecs-query-rewriting](ecs-query-rewriting.md) (family T - the `Exclude`/`WithNone`
  narrowing used in step 2),
  [disable-replace-vanilla-system](disable-replace-vanilla-system.md) (family M),
  [namesystem-custom-names](namesystem-custom-names.md) (family Q - the
  `SetCustomName` write used in step 3),
  [ecs-serializable-savedata](ecs-serializable-savedata.md) (family E - persisting
  the takeover's marker/state).
- Reference: [technique-index](../../technique-index.md) (family B).
- Case studies demonstrating it:
  [realistic-path-finding](../../case-studies/realistic-path-finding.md),
  [time2work-realistic-trips](../../case-studies/time2work-realistic-trips.md).

## Sources

- Canonical mods (dossier + repo, pinned commits):
  - `traffic-tool-essentials` @10973595ac9ed37f47ca24eb788d4421dc295fa0
  - `realistic-path-finding` @50645fa6a078181365e36a42e2e27b96699bf02a
  - `advanced-road-naming` @559e72cdb3180e3e71869643367e094a22eed988
- Official/community references (link out, do not duplicate):
  - Unity DOTS / ECS system scheduling: https://docs.unity3d.com/Packages/com.unity.entities@latest
  - CS2 modding wiki (system update phases): https://cs2.paradoxwikis.com/Modding
