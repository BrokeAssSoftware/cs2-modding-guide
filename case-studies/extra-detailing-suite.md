---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Case study: Extra Detailing Tools"
case_study: extra-detailing-suite
mod: "Extra Detailing Tools (80528)"
dossier: ../../vice-and-order-research/mods/dossiers/extra-detailing-tools/
repo_commit: 41df3c2b8b444197f7c3e2de4496c01cfd8bfc23
source_version: "n/a - no pinned source"
last_reverified: "2026-07-04"
diataxis: explanation
techniques: [V, A]
technique_applicability: [tooling, ui, content]
status: source-verified
Created: 2026-07-03
Updated: 2026-07-03
Owners:
  - codex
---

# Extra Detailing Tools - case study

> Extra Detailing Tools ships an in-scene transform gizmo as a real `ToolBaseSystem`
> (drag to move/rotate via camera-ray plane projection), a generic reflective snap
> framework that postfix-patches any tool's snap/raycast methods, and a load-time
> asset-menu builder that reclassifies vanilla surfaces, decals, and net lanes into new
> UI categories by adding one component to each prefab. It is a tour of building editor
> tooling on top of vanilla systems - and of where reflection buys power at the cost of
> fragility.

All code claims below are cited to `repo/<path>#Lxx` at pinned commit
`41df3c2b8b444197f7c3e2de4496c01cfd8bfc23` (branch `main`, commit dated 2026-05-14,
publish config ModVersion 1.2.5.2; the live Paradox build 1.6.* is newer than this repo
tip), surfaced through the dossier at
`../../vice-and-order-research/mods/dossiers/extra-detailing-tools/`. The architecture
changed substantially since the 2025-11-07 dossier: the old `MoveHandle` MonoBehaviour
gizmo was replaced by the interactive `TransformGizmoTool`, and snapping was rewritten as
the generic `ExtraSnapBase<TTool, TSnap>` framework with a new `ObjectSide` edge-snap
mode. Deep snap-math internals are labelled `Needs Verification` below where they cannot
be cleanly cited.

## What it does / why it's instructive

Extra Detailing Tools (EDT, author AlphaGaming7780/Triton Supreme, Paradox ModId 80528)
gives detailers editor-grade controls in the live game: a Selected-Info transform panel
with numeric position/rotation/scale and copy/paste; an interactive drag gizmo launched
from that panel; surface- and edge-snapping for props; asset menus that surface hidden
vanilla surfaces, decals, and fence/utility lanes; and a marker-visibility toggle. It
hard-depends on ExtraLib (modId 75724) for shared UI/gizmo helpers, and on Anarchy and
Better Bulldozer.

It is instructive because it shows how far a detailing toolset can go on top of vanilla
systems:

1. **A gizmo tool built as ECS, not a MonoBehaviour.** `TransformGizmoTool` is a
   `ToolBaseSystem` that raycasts against gizmo entities and drags by projecting the
   camera ray onto an axis-aligned plane - the standard way to add interactive
   manipulation without a persistent GameObject.
2. **A generic, reflective snap framework.** Rather than one hand-written snap prefix,
   `ExtraSnapBase<TTool, TSnap>` validates a `[Flags]` enum and reflectively postfixes
   any tool's `SnapControlPoint`/`InitializeRaycast`, decoupling new snap modes from the
   patched method bodies - powerful, and the mod's biggest maintenance liability.
3. **Asset menus by prefab-component override.** The catalog is generated at load by
   querying vanilla prefabs and adding a `UIObject` component to slot them into new UI
   categories - a clean family-A example with no hand-authored asset lists.

## Architecture at a glance

`EDT.OnLoad` loads icons and localization, builds the asset menus, registers the
tool/gizmo/UI systems into explicit phases, adds the transform panel to selected-info,
registers the snap framework, and finally applies Harmony patches
(`repo/EDT.cs#L51-L124`):

```csharp
EditEntities.SetupEditEntities();
updateSystem.UpdateAt<GizmosRenderSystem>(SystemUpdatePhase.Rendering);
updateSystem.UpdateAt<GizmosRaycastSystem>(SystemUpdatePhase.Raycast);
updateSystem.UpdateAt<TransformGizmoTool>(SystemUpdatePhase.ToolUpdate);
updateSystem.UpdateAt<TransformGizmoToolUI>(SystemUpdatePhase.UIUpdate);
...
selectedInfoUISystem.AddMiddleSection(World.GetOrCreateSystemManaged<TransformSection>());
ExtraSnapBase.RegisterInstance<ObjectToolSystemExtraSnap>();
harmony = new($"{nameof(ExtraDetailingTools)}.{nameof(EDT)}");
harmony.PatchAll(typeof(EDT).Assembly);
```
(`repo/EDT.cs#L68-L112`)

Gizmos are data (`GizmosData` entities) drawn by `GizmosRenderSystem` (Rendering) and
hit-tested by a bespoke parallel `GizmosRaycastSystem` (Raycast), both via ExtraLib's
`GizmoBatcher`. Grass systems and a `GameGrassPrefab` are only wired under the optional
`#if Extra4` build (`repo/EDT.cs#L78-L85`). `OnDispose` unpatches only EDT's own Harmony
id (`repo/EDT.cs#L126-L130`).

### The transform gizmo is a ToolBaseSystem

`TransformGizmoTool` is `internal partial class TransformGizmoTool : ToolBaseSystem`
with `toolID => "TransformGizmoTool"` and Default/Move/Rotate/Scale modes
(`repo/MOD/Systems/Tools/TransformGizmoTool.cs#L42`, `#L770`, `#L784`). Dragging projects
the camera ray onto a plane built from the selected axis and applies the offset; the
gizmo defaults to local-axis and sub-building movement
(`repo/MOD/Systems/Tools/TransformGizmoTool.cs#L815-L816`,
`repo/MOD/Systems/Tools/TransformGizmoTool.cs#L1041`):

```csharp
Plane dragPlane = CreateDragPlane(axisDir, m_DragStartGizmoPos);
Ray ray = Camera.main.ScreenPointToRay(Input.mousePosition);
if (dragPlane.Raycast(ray, out float enter))
{
    float3 hitPos = ray.origin + ray.direction * enter;
    float3 projectedDelta = math.dot(hitPos - m_DragStartMouseHitPos, axisDir) * axisDir;
    // newPos = projectedDelta + m_DragStartGizmoPos
}
```

`CreateDragPlane` uses the axis as the plane normal for Rotate and `cross(axis, camRight)`
for Move to avoid the axis-parallel-to-camera degenerate case
(`repo/MOD/Systems/Tools/TransformGizmoTool.cs#L1477`). The tool is launched from the
info panel's `selectTransformGizmosTool` trigger, which sets
`toolSystem.activeTool = _transformGizmoTool`
(`repo/MOD/Systems/UI/TransformSection.cs#L121`).

### The transform panel binds ECS to React

`TransformSection` (an `InfoSectionBase`, `group => "Transform Tool"`) registers
`GetterValueBinding`/`TriggerBinding` pairs under the `edt` namespace for position,
rotation, scale, their increments (`new double3(1, 45, 1)`), copy/paste, local-axis, and
move-sub-buildings, then writes changes back as `Updated` components
(`repo/MOD/Systems/UI/TransformSection.cs#L65`, `#L87-L121`). Scale is stored in a custom
`TransformObject` component that is removed when reset to unit scale (sparse storage),
and `TransformObject` is `ISerializable`, so scale persists across save/reload.

### Asset menus are built by adding a component to vanilla prefabs

`EditEntities.SetupEditEntities` registers three `EL.AddOnEditEnities` callbacks over
`EntityQueryDesc`s for surfaces (`SurfaceData`), decals (`StaticObjectData` +
spawnable/placeable), and net lanes (`NetLaneData` + secondary/utility)
(`repo/MOD/EditEntities.cs#L20-L60`). Each callback resolves the prefab and, if it lacks
one, adds a `UIObject` component with an icon and priority - slotting the vanilla asset
into a new UI category (`repo/MOD/EditEntities.cs#L79-L99`):

```csharp
if (EL.m_PrefabSystem.TryGetPrefab(entity, out SurfacePrefab prefab))
{
    var prefabUI = prefab.GetComponent<UIObject>();
    if (prefabUI == null)
    {
        prefabUI = prefab.AddComponent<UIObject>();
        prefabUI.m_Icon = Icons.GetIcon(prefab);
        prefabUI.m_Priority = 1;
    }
}
```

Surface categories key off `RenderedArea.m_RendererPriority`, so a new built-in surface
lands in a predictable bucket with no per-prefab maintenance; when Asset Icon Library is
installed EDT leaves icon fields blank for the companion to fill
(`repo/MOD/EditEntities.cs#L83-L99`).

### The snap framework is generic and reflective

`ExtraSnapBase<TTool, TSnap>` requires `TTool : ToolBaseSystem` and `TSnap : Enum`. Its
static constructor rejects a `TSnap` that is not a `[Flags]` `uint` enum; the instance
constructor builds a private `Harmony` and reflectively patches the tool's
`SnapControlPoint` and `InitializeRaycast`
(`repo/MOD/ExtraSnap/ExtraSnap.cs#L50-L113`):

```csharp
var snapMethod = typeof(TTool).GetMethod("SnapControlPoint", BindingFlags.Instance | BindingFlags.Public | BindingFlags.NonPublic);
var raycastMethod = typeof(TTool).GetMethod("InitializeRaycast", ...);
_harmony.Patch(snapMethod, postfix: new HarmonyMethod(... nameof(PostfixSnapControlPoint) ...));
```

`ObjectToolSystemExtraSnap : ExtraSnapBase<ObjectToolSystem, ObjectToolExtraSnap>` defines
`[Flags] enum ObjectToolExtraSnap : uint { ObjectSurface, ObjectSide }` and schedules a
Burst `SnapJob`, reading the tool's private state through Harmony `Traverse`
(`m_Prefab`, `m_ControlPoints`, `m_MovingObject`, `m_LastRaycastPoint`, and the
`GetActualSnap` method) (`repo/MOD/ExtraSnap/ObjectTool.cs#L27-L87`). Declarative
postfixes on `ObjectToolSystem.GetAvailableSnapMask`/`GetAllowRotation` add the new snap
options and re-enable rotation for free-standing props (`repo/MOD/Patches/ObjectToolSystem.cs#L28`).

## Techniques demonstrated

- [Custom tool with in-scene selection/drag](../how-to/recipes/tool-drag-select.md)
  (family V) - `TransformGizmoTool` is a `ToolBaseSystem` scheduled in `ToolUpdate` that
  raycasts gizmo entities and drags via camera-ray plane projection
  (`repo/MOD/Systems/Tools/TransformGizmoTool.cs#L42`, `#L1041`, `#L1477`), launched from
  the Selected-Info panel's `selectTransformGizmosTool` trigger
  (`repo/MOD/Systems/UI/TransformSection.cs#L121`). The panel's `GetterValueBinding`/
  `TriggerBinding` surface is the React bridge that drives it
  (`repo/MOD/Systems/UI/TransformSection.cs#L87-L121`).
- [Prefab-field override](../how-to/recipes/prefab-field-override.md) (family A) -
  `EditEntities` performs the canonical `TryGetPrefab -> read/AddComponent -> set fields`
  shape on vanilla prefabs, adding a `UIObject` component (icon + priority) to slot each
  surface/decal/net-lane prefab into a new UI category
  (`repo/MOD/EditEntities.cs#L87-L99`). The gizmo's scale edit is the same pattern on a
  custom `TransformObject` component, added only when scale is non-unit and removed when
  reset (`repo/MOD/Systems/UI/TransformSection.cs#L65`).

Supporting techniques on display but not a headline family here: the generic reflective
snap framework `ExtraSnapBase<TTool, TSnap>` (`repo/MOD/ExtraSnap/ExtraSnap.cs#L50-L113`),
which postfix-patches tool snap/raycast methods and reads private fields via `Traverse`;
and a data-driven ECS gizmo render/raycast subsystem
(`repo/MOD/Gizmos/GizmosRenderSystem.cs`, `.../GizmosRaycastSystem.cs`).

See the [technique index](../technique-index.md) for the family ledger. Related
explanation pages: [UI <-> C# communication](../explanation/ui-cs-communication.md),
[dependency strategy](../explanation/dependency-strategy.md),
[serialization](../explanation/serialization.md), and
[module layout](../explanation/module-layout.md).

## Key decisions & tradeoffs

- **ECS gizmo tool vs. a MonoBehaviour handle.** Modeling the gizmo as data entities
  drawn/hit-tested by ECS systems (replacing the old `MoveHandle` MonoBehaviour) keeps
  it inside the DOTS world and Burst-friendly, at the cost of a bespoke parallel raycast
  engine to intersect the screen ray against gizmo primitives
  (`repo/MOD/Systems/Tools/TransformGizmoTool.cs#L42`, `repo/EDT.cs#L73-L75`).
- **Generic reflective snap framework vs. per-tool prefixes.** `ExtraSnapBase<TTool,
  TSnap>` decouples new snap modes from the patched method bodies and is reusable across
  tools, but it reflects method and field names (`SnapControlPoint`, `InitializeRaycast`,
  `m_ControlPoints`, `m_Prefab`, ...), so a Colossal refactor of `ObjectToolSystem` can
  throw `MissingMethodException` at load
  (`repo/MOD/ExtraSnap/ExtraSnap.cs#L100-L113`, `repo/MOD/ExtraSnap/ObjectTool.cs#L64-L74`).
- **Asset menus by component override vs. hand-authored lists.** Generating the catalog
  from vanilla prefab queries plus a `UIObject` add means new built-in assets appear
  automatically by renderer-priority/name heuristics, with no per-asset maintenance
  (`repo/MOD/EditEntities.cs#L79-L99`).
- **Hard dependency on the Extra suite.** EDT declares Anarchy, Better Bulldozer, and
  ExtraLib as required (its UI/icon/gizmo helpers live in ExtraLib), trading a heavier
  dependency footprint for not reimplementing shared infrastructure
  (`repo/Properties/PublishConfiguration.xml`).
- **Optional Extra4 build.** Grass systems, transform-object scaling, and a
  `ToolUISystem.AllowBrush` patch only compile under the `Extra4` symbol; the packaged
  release ships without them, so those APIs are gated behind a build flag
  (`repo/EDT.cs#L78-L96`, `repo/MOD/Systems/UI/TransformSection.cs#L302`).

## Pitfalls / upstream-watch

- **Reflection fragility (snap framework).** The snap path reflects tool method and field
  names via Harmony `Traverse`; any base-game refactor of `ObjectToolSystem` can break
  snapping or throw at load - regression-test snapping after each game patch
  (`repo/MOD/ExtraSnap/ObjectTool.cs#L64-L74`, `repo/MOD/ExtraSnap/ExtraSnap.cs#L100-L108`).
  `Needs Verification (in-game)`: the `ObjectSurface`/`ObjectSide` snap accuracy and the
  OBB-vs-edge snap math cannot be confirmed from source alone.
- **Shared `selectedSnap` state leaks across tools.** The snap-mask postfixes toggle bits
  on the shared vanilla `selectedSnap`, so a snap left enabled while detailing can affect
  later road/object placement; GitHub issue #24 tracks snap-to-surface defaulting on and
  latching roads onto trees. Mitigation: toggle snap off before switching tools
  (`repo/MOD/Patches/ObjectToolSystem.cs#L48-L65`).
- **ExtraSnap Harmony not disposed.** `EDT.OnDispose` unpatches only EDT's own Harmony id;
  the ExtraSnap framework's separate `Harmony` instance has a `Dispose()` that is never
  called, so its postfixes may leak or double-apply on a mid-session reload
  (`repo/EDT.cs#L126-L130`, `repo/MOD/ExtraSnap/ExtraSnap.cs#L95`). `Needs Verification
  (in-game)`.
- **`Camera.main` dependency on the main thread.** The gizmo drag and raycast read
  `Camera.main` every frame; a null/disabled main camera would throw
  (`repo/MOD/Systems/Tools/TransformGizmoTool.cs#L1477`).
- **Forced `deletable = true`.** An `ActionsSection.OnProcess` postfix sets
  `deletable = true` unconditionally for the selected entity, potentially exposing
  deletes vanilla protects - verify no orphaning of service-owned objects
  (`repo/MOD/Patches/ActionsSection.cs#L33`). `Needs Verification (in-game)`.
- **Repo tip predates the live 1.6.* release.** This commit's publish config still lists
  GameVersion 1.5.* / ModVersion 1.2.5.2 while the live Paradox build is 1.6.*; cite this
  commit for source, the storefront for the current shipping version
  (`repo/Properties/PublishConfiguration.xml`).

## Source pointers

- Dossier: `../../vice-and-order-research/mods/dossiers/extra-detailing-tools/`
  (index / source / modding / guide + notes, incl. `notes/depth-challenge-20260625.md`).
- Repo @ `41df3c2b8b444197f7c3e2de4496c01cfd8bfc23` (branch `main`), key files:
  - `repo/EDT.cs` - `OnLoad` registration, `harmony.PatchAll`, snap-framework registration.
  - `repo/MOD/Systems/Tools/TransformGizmoTool.cs` - interactive gizmo `ToolBaseSystem`.
  - `repo/MOD/Systems/UI/TransformSection.cs` - Selected-Info transform panel bindings.
  - `repo/MOD/ExtraSnap/ExtraSnap.cs`, `.../ObjectTool.cs` - generic reflective snap
    framework + `ObjectSurface`/`ObjectSide` Burst `SnapJob`.
  - `repo/MOD/EditEntities.cs` - asset-menu catalog via `UIObject` component override.
  - `repo/MOD/Gizmos/GizmosRenderSystem.cs`, `.../GizmosRaycastSystem.cs` - data-driven
    gizmo render/raycast subsystem.
- Dependencies: ExtraLib (75724), Anarchy (74604), Better Bulldozer (75250);
  Lib.Harmony 2.2.2. Storefront modId 80528.
