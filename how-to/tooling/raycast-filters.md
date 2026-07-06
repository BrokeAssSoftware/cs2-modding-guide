---
FrontmatterVersion: 1
DocumentType: Guide
Title: "How-to: Raycast filters for custom tools"
diataxis: how-to
source_version: "~1.5.2f1 (road-speed-adjuster@e0c0c0b; date-pinned, static source only)"
last_reverified: "2026-07-04"
canonical_mods:
  - road-speed-adjuster@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed
  - better-bulldozer@4408466f226db811159d92859479ae1e1c28ba06
  - anarchy@a6311e898d20a775368668b234aaa32f06e3e1eb
technique_applicability: [tooling, ui]
status: source-verified
Created: 2026-07-03
Updated: 2026-07-03
Owners:
  - codex
---

# Raycast filters for custom tools

> Make a tool (bulldozer, object, net, or a bespoke `ToolBaseSystem`) hover and
> apply to only the entities you want - a specific network layer, invisible markers,
> a utility type, sub-elements - by writing the mask fields on `ToolRaycastSystem`
> inside `InitializeRaycast`.

## Problem
Every tool in CS2 finds what the cursor is over through a shared `ToolRaycastSystem`.
By default a custom tool inherits whatever mask the base class set, so it either hits
everything or hits nothing useful. You want the raycaster to report **only** roads and
tracks, or **only** invisible markers, or **only** a building's sub-props - and to
ignore the rest so `GetRaycastResult` returns the entity family your tool actually
operates on.

## Solution
Override `InitializeRaycast()` on your `ToolBaseSystem` and set the filter fields on
`m_ToolRaycastSystem`. The engine calls `InitializeRaycast` every frame the tool is
active, so whatever you write there is the live filter. The knobs are:

- `typeMask` (`TypeMask.Net`, `TypeMask.StaticObjects`, `TypeMask.Areas`,
  `TypeMask.MovingObjects`, `TypeMask.Terrain`, `TypeMask.Lanes`, ...) - the broad
  entity categories.
- `netLayerMask` (`Layer.Road`, `Layer.TrainTrack`, `Layer.MarkerPathway`,
  `Layer.PowerlineLow`, ...) - which network/marker layers count when `TypeMask.Net`
  is set.
- `utilityTypeMask` (`UtilityTypes.SewagePipe`, `UtilityTypes.HighVoltageLine`, ...) -
  narrows buried utilities.
- `areaTypeMask` (`AreaTypeMask.Lots`, `AreaTypeMask.Spaces`, ...) - narrows areas.
- `raycastFlags` (`RaycastFlags.Markers`, `RaycastFlags.SubElements`,
  `RaycastFlags.Decals`, `RaycastFlags.NoMainElements`, ...) - special cases.
- `collisionMask` (`CollisionMask.OnGround | CollisionMask.Underground | Overground`) -
  vertical band the ray considers.

Always call `base.InitializeRaycast()` first, then overwrite the fields. If you do not
own the vanilla tool you are extending, you can achieve the same thing with a Harmony
`Postfix` on that tool's `InitializeRaycast` and mutate the same fields.

## Steps & Code

### 1. Override `InitializeRaycast` on your own tool

The simplest case: a custom `ToolBaseSystem` that only wants to hit road-like
networks. Set `typeMask`, then OR the layers you care about into `netLayerMask`:

```csharp
public override void InitializeRaycast()
{
    base.InitializeRaycast();

    // Detect roads, train/tram/subway tracks, and waterways - nothing else.
    m_ToolRaycastSystem.typeMask = TypeMask.Net;
    m_ToolRaycastSystem.netLayerMask =
        Layer.Road | Layer.TrainTrack | Layer.TramTrack | Layer.SubwayTrack | Layer.Waterway;
    m_ToolRaycastSystem.raycastFlags = RaycastFlags.SubElements | RaycastFlags.Markers;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/Systems/RoadSpeedToolSystem.cs#L145-L152` (@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed)

A tool that carries no prefab of its own returns null/false from the prefab hooks so
the toolbar does not try to bind one:

```csharp
public override PrefabBase GetPrefab()          => null;
public override bool TrySetPrefab(PrefabBase p)  => false;
```
Source: `../../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/Systems/RoadSpeedToolSystem.cs#L154-L162` (@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed)

### 2. Switch the filter by UI state

Real tools expose several targets. Read your UI state at the top of
`InitializeRaycast` and build a different mask per mode. Better Bulldozer's
sub-element tool branches on a "reset vs markers vs default" selection mode and sets a
completely different mask in each branch:

```csharp
public override void InitializeRaycast()
{
    base.InitializeRaycast();
    m_ToolRaycastSystem.collisionMask = CollisionMask.OnGround | CollisionMask.Overground;

    if (m_RenderingSystem.markersVisible &&
        m_BetterBulldozerUISystem.SelectedRaycastTarget == RaycastTarget.Markers)
    {
        m_ToolRaycastSystem.typeMask = m_BetterBulldozerUISystem.MarkersFilter;
        m_ToolRaycastSystem.netLayerMask =
            Layer.MarkerPathway | Layer.MarkerTaxiway | Layer.PowerlineLow |
            Layer.PowerlineHigh | Layer.WaterPipe | Layer.SewagePipe;
        m_ToolRaycastSystem.raycastFlags = RaycastFlags.Markers;
        m_ToolRaycastSystem.utilityTypeMask =
            UtilityTypes.LowVoltageLine | UtilityTypes.HighVoltageLine | UtilityTypes.SewagePipe;
        m_ToolRaycastSystem.collisionMask =
            CollisionMask.OnGround | CollisionMask.Underground | CollisionMask.Overground;
    }
    // ... other branches ...
    m_ToolRaycastSystem.raycastFlags |= RaycastFlags.SubElements | RaycastFlags.NoMainElements;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/better-bulldozer/repo/BetterBulldozer/Tools/SubElementBulldozerTool.cs#L98-L152` (@4408466f226db811159d92859479ae1e1c28ba06)

Note the two things that unlock invisible geometry: the `m_RenderingSystem.markersVisible`
gate (markers must be shown for the raycast to see them) and the terminal
`raycastFlags |= RaycastFlags.SubElements | RaycastFlags.NoMainElements` (hit child
elements, skip the parent).

### 3. If you don't own the tool, Harmony-patch its `InitializeRaycast`

To extend a **vanilla** tool without replacing it, postfix its `InitializeRaycast` and
mutate the shared raycast system. Better Bulldozer patches `BulldozeToolSystem` and
rewrites the mask from its own UI state:

```csharp
[HarmonyPatch(typeof(BulldozeToolSystem), "InitializeRaycast")]
public class BulldozeToolSystemInitializeRaycastPatch
{
    public static void Postfix()
    {
        ToolSystem toolSystem = World.DefaultGameObjectInjectionWorld
            .GetOrCreateSystemManaged<ToolSystem>();
        if (!toolSystem.actionMode.IsGame()) return;

        ToolRaycastSystem toolRaycastSystem = World.DefaultGameObjectInjectionWorld
            .GetOrCreateSystemManaged<ToolRaycastSystem>();
        // ... set typeMask / netLayerMask / areaTypeMask / raycastFlags by UI target ...
    }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/better-bulldozer/repo/BetterBulldozer/Patches/BulldozeToolSystemInitializeRaycastPatch.cs#L20-L71` (@4408466f226db811159d92859479ae1e1c28ba06)

Anarchy uses the same postfix technique on `NetToolSystem` to add extra upgradeable
layers, reaching the private field through Harmony `Traverse` and OR-ing new layers
only in `Replace` mode:

```csharp
[HarmonyPatch(typeof(NetToolSystem), "InitializeRaycast")]
internal class NetToolSystem_InitializeRaycast
{
    static void Postfix(NetToolSystem __instance)
    {
        var toolRaycastSystem = Traverse.Create(__instance)
            .Field<ToolRaycastSystem>("m_ToolRaycastSystem").Value;
        if (toolRaycastSystem == null) return;

        if (__instance.actualMode == NetToolSystem.Mode.Replace)
            toolRaycastSystem.netLayerMask |=
                Layer.Pathway | Layer.TrainTrack | Layer.PublicTransportRoad |
                Layer.TramTrack | Layer.SubwayTrack;
    }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/Patches/NetToolSystem_InitializeRaycast.cs#L31-L48` (@a6311e898d20a775368668b234aaa32f06e3e1eb)

## Pitfalls & gotchas

- **`InitializeRaycast` runs every frame - set the whole mask, don't assume state.**
  The engine re-initializes the raycast each frame the tool is active. Anything you do
  not set falls back to the base class value, so build the complete filter each call
  rather than mutating incrementally between frames.
- **Markers are invisible to the raycast unless markers are shown.** Filtering to
  `Layer.MarkerPathway` etc. does nothing while `RenderingSystem.markersVisible` is
  false. Better Bulldozer gates its marker branch on `m_RenderingSystem.markersVisible`
  (`SubElementBulldozerTool.cs#L116`). If your tool must target markers, force markers
  visible while it is active and restore the previous value on exit.
- **`OR (|=)` vs assign (`=`).** A Harmony postfix that does `netLayerMask |= ...`
  *adds* to whatever the vanilla tool already set (Anarchy). A branch that does
  `netLayerMask = ...` *replaces* it (Better Bulldozer's own tool). Pick deliberately:
  OR to extend vanilla behavior, assign to take full control.
- **`collisionMask` is a separate axis.** Buried utilities and underground markers need
  `CollisionMask.Underground` added, or the ray never reaches them even with the right
  `netLayerMask` (`SubElementBulldozerTool.cs#L124`).
- **Harmony postfix must guard game mode.** Better Bulldozer bails when
  `!toolSystem.actionMode.IsGame()` so the patch does not rewrite the raycast in the
  editor/main menu. Skipping this leaks your filter into contexts where it makes no
  sense.
- Whether a specific `Layer`/`TypeMask` combination actually surfaces the entity you
  expect in-game is `Needs Verification (in-game)` for any layer not shown in the cited
  source - the enums are broad and some combinations no-op.

## Variations

- **Combine with vanilla filters instead of replacing them.** Read the vanilla tool's
  existing bitmask and OR your custom layers in (the Anarchy `|=` postfix pattern), so
  your filter stacks with the base game's category toggles rather than clobbering them.
- **Filter areas or moving objects, not just networks.** The same override takes
  `typeMask = TypeMask.Areas` + `areaTypeMask`, or `typeMask = TypeMask.MovingObjects`
  (vehicles/cims/animals). Better Bulldozer's vanilla-target patch switches between all
  of these by `RaycastTarget` (`BulldozeToolSystemInitializeRaycastPatch.cs#L52-L67`).
- **Terrain-only radius brushes.** A radius/brush tool can set
  `typeMask = TypeMask.Terrain` when in radius mode and switch to `TypeMask.StaticObjects`
  in single-select mode (see [transform gizmos](transform-gizmos.md), whose
  `AnarchyComponentsTool` does exactly this).

## See also
- Sibling tooling how-tos: [sub-element removal](sub-element-removal.md),
  [transform gizmos](transform-gizmos.md), [validation overrides](validation-overrides.md).
- Reference: [technique index](../../technique-index.md) - families V (tool
  drag-select + Highlighted) and Y (event-driven ModificationEnd / ToolOutputBarrier),
  which most raycast-driven tools build on.
- Explanation: [conditional execution](../../explanation/conditional-execution.md)
  (guarding tool work by game mode / active tool).

## Sources
- Canonical mods (dossier + repo):
  - `road-speed-adjuster` @e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed - `repo/Systems/RoadSpeedToolSystem.cs`
  - `better-bulldozer` @4408466f226db811159d92859479ae1e1c28ba06 - `repo/BetterBulldozer/Tools/SubElementBulldozerTool.cs`, `repo/BetterBulldozer/Patches/BulldozeToolSystemInitializeRaycastPatch.cs`
  - `anarchy` @a6311e898d20a775368668b234aaa32f06e3e1eb - `repo/Anarchy/Patches/NetToolSystem_InitializeRaycast.cs`
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding
