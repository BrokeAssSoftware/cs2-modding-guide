---
FrontmatterVersion: 1
DocumentType: Guide
Title: "How-to: Transform tweaks, locks, and gizmo tools"
diataxis: how-to
source_version: "~1.5.9 (anarchy@a6311e8; date-pinned, static source only)"
last_reverified: "2026-07-04"
canonical_mods:
  - anarchy@a6311e898d20a775368668b234aaa32f06e3e1eb
  - extra-detailing-tools@41df3c2b8b444197f7c3e2de4496c01cfd8bfc23
technique_applicability: [tooling, content]
status: source-verified
Created: 2026-07-03
Updated: 2026-07-03
Owners:
  - codex
---

# Transform tweaks, locks, and gizmo tools

> Let players fine-tune where a placed object sits - lock a transform so the game never
> re-drops it, prevent the vanilla override that culls overlapping props, or drive an
> interactive move/rotate gizmo - by attaching small ECS marker components and
> committing transform edits through a tool barrier.

## Problem
When you place an object off its "legal" spot (floating, overlapping, on a slope) the
game tends to fight you: it re-projects the object to the ground next load, or culls it
because another object overrides its footprint. You want two capabilities: (1) **pin**
an object's exact position/rotation so it stays put across save/load, and (2) offer a
UI to nudge or freely gizmo-drag the transform with precise, reversible edits.

## Solution
Both capabilities are ECS-component work applied through a custom `ToolBaseSystem`:

- A **transform lock** is a serializable component that records the intended
  position/rotation (`TransformRecord`). A background check system compares the live
  `Game.Objects.Transform` against the record and snaps it back if the game moved it.
- An **override-prevention lock** is an empty tag component (`PreventOverride`) that a
  query uses to exclude the entity from the vanilla overlap/cull pass.
- A **gizmo/nudge tool** is a `ToolBaseSystem` that raycasts for a static object, then
  writes an updated `Game.Objects.Transform` (plus `Updated` to re-render) through a
  `ToolOutputBarrier` command buffer.

The two lock components are applied and removed by a radius/single-select tool, so the
player paints them on rather than editing data by hand.

## Steps & Code

### 1. Define the lock components

`TransformRecord` is a serializable struct holding the position and rotation to hold the
object at, with a sanity clamp on deserialize so a corrupt save cannot fling an object
to infinity:

```csharp
public struct TransformRecord : IComponentData, IQueryTypeParameter, ISerializable
{
    public float3 m_Position;
    public quaternion m_Rotation;

    public Game.Objects.Transform Transform => new (m_Position, m_Rotation);
}
```
Source: `../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/Components/TransformRecord.cs#L15-L32` (@a6311e898d20a775368668b234aaa32f06e3e1eb)

`PreventOverride` is an *empty* tag - it carries no data, it just marks membership so
queries can exclude the entity from the vanilla override sweep:

```csharp
public struct PreventOverride : IComponentData, IQueryTypeParameter, IEmptySerializable
{
}
```
Source: `../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/Components/PreventOverride.cs#L13` (@a6311e898d20a775368668b234aaa32f06e3e1eb)

### 2. Model the tool's modes as a flags enum

The tool decides which component(s) to paint from a UI-bound flags enum:

```csharp
public enum AnarchyComponentType
{
    None = 0,
    PreventOverride = 1,
    TransformRecord = 2,
}
```
Source: `../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/Systems/AnarchyComponentsTool/AnarchyComponentsToolUISystem.cs#L37-L47` (@a6311e898d20a775368668b234aaa32f06e3e1eb)

### 3. Build the tool as a `ToolBaseSystem` with a barrier

```csharp
public partial class AnarchyComponentsToolSystem : ToolBaseSystem
{
    public override string toolID => "AnarchyComponentsTool";
    private ToolOutputBarrier m_Barrier;   // set in OnCreate via GetOrCreateSystemManaged
```
Source: `../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/Systems/AnarchyComponentsTool/AnarchyComponentsToolSystem.cs#L38-L62, #L41, #L138` (@a6311e898d20a775368668b234aaa32f06e3e1eb)

Its `InitializeRaycast` switches between terrain (radius mode) and static objects
(single-select) - see [raycast filters](raycast-filters.md):

```csharp
public override void InitializeRaycast()
{
    base.InitializeRaycast();
    if (m_UISystem.CurrentSelectionMode == SelectionMode.Radius)
        m_ToolRaycastSystem.typeMask = TypeMask.Terrain;
    else
    {
        m_ToolRaycastSystem.typeMask = TypeMask.StaticObjects;
        m_ToolRaycastSystem.raycastFlags |= RaycastFlags.Markers | RaycastFlags.Placeholders;
    }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/Systems/AnarchyComponentsTool/AnarchyComponentsToolSystem.cs#L94-L116` (@a6311e898d20a775368668b234aaa32f06e3e1eb)

### 4. Paint / erase the lock via a radius job on the barrier

On apply, the tool schedules an `AddOrRemoveComponentWithinRadiusJob` that adds (or,
with `m_Add = false`, removes) the chosen component to every matching entity in the
brush radius, writing through a barrier command buffer:

```csharp
AddOrRemoveComponentWithinRadiusJob addPreventOverrideJob = new()
{
    m_Position      = hit.m_HitPosition,
    m_Radius        = radius,
    buffer          = m_Barrier.CreateCommandBuffer(),
    m_Add           = true,
    m_ComponentType = ComponentType.ReadOnly<PreventOverride>(),
    // ... lookups ...
};
inputDeps = JobChunkExtensions.Schedule(addPreventOverrideJob, m_NotPreventOverrideQuery, inputDeps);
```
Source: `../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/Systems/AnarchyComponentsTool/AnarchyComponentsToolSystem.cs#L404-L420` (@a6311e898d20a775368668b234aaa32f06e3e1eb)

The `m_NotPreventOverrideQuery` / `m_PreventOverrideQuery` pair (built in `OnCreate`,
`#L156-L183`) ensures add only touches entities that lack the tag and remove only
touches entities that have it - so the pass is idempotent.

### 5. Enforce the lock from a background system

`TransformRecord` alone does nothing; a check system compares the recorded transform to
the live one and re-applies it if the game moved the object. Anarchy ships
`CheckTransformSystem` / `ResetTransformSystem` for this
(`Anarchy/Systems/ObjectElevation/CheckTransformSystem.cs`,
`ResetTransformSystem.cs`). The record's `Equals(Transform)` helper
(`TransformRecord.cs#L40-L47`) is the comparison used to detect drift.

## Pitfalls & gotchas

- **A recorded transform is inert without an enforcing system.** `TransformRecord` is
  just data. If you attach it but never run a check-and-reset system, the game will
  still re-drop the object. The record + check-system pair is the whole mechanism.
- **`PreventOverride` is a query filter, not magic.** It only works because the vanilla
  override/cull logic (or your own systems) is written to exclude entities carrying the
  tag. Adding the empty component to an entity that no system queries against does
  nothing.
- **Commit transform edits through a barrier, in the right phase.** Writing
  `Game.Objects.Transform` directly during a tool update races the render/simulation.
  Route it through a `ToolOutputBarrier` command buffer and add `Updated` so the change
  re-renders. See [multi-phase scheduling](../../explanation/multi-phase-scheduling.md).
- **Clamp on deserialize.** `TransformRecord.Deserialize` resets to origin/identity when
  the position is outside +/-100000 or the rotation is non-finite
  (`TransformRecord.cs#L69-L74`). A saved lock is attacker/corruption-adjacent data;
  guard it before you shove it back into a `Transform`.
- **Idempotent add/remove.** Split your queries into "has tag" and "lacks tag" so the
  add job never re-adds and the remove job never errors on a missing component
  (Anarchy's `m_NotPreventOverrideQuery` vs `m_PreventOverrideQuery`).

## Variations

- **Interactive move/rotate gizmo instead of a paint brush.** Extra Detailing Tools
  ships a full gizmo tool - a `ToolBaseSystem` (`toolID "TransformGizmoTool"`) that
  raycasts for static/moving objects and writes an updated `Game.Objects.Transform`
  (position + rotation) through a `ToolOutputBarrier`, cascading the offset to
  sub-objects, sub-lanes, sub-areas, and installed upgrades:

  ```csharp
  internal partial class TransformGizmoTool : ToolBaseSystem
  {
      public override string toolID => "TransformGizmoTool";
      private ToolOutputBarrier m_ToolOutputBarrier;  // set in OnCreate
  }
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/extra-detailing-tools/repo/MOD/Systems/Tools/TransformGizmoTool.cs#L42-L42, #L784, #L789, #L839` (@41df3c2b8b444197f7c3e2de4496c01cfd8bfc23)

  Its rotation math projects cursor movement onto a plane and applies
  `quaternion.AxisAngle` (`TransformGizmoTool.cs#L1035`); the raycast filter is set in
  `InitializeRaycast` (`#L881-L905`).

- **Surface / edge snapping.** Extra Detailing Tools also implements a generic snap
  framework (`ExtraSnap`) with an object-side (edge) snap mode. The exact snap-toggle
  field wiring (e.g. mapping a UI toggle to a `snapToSurface`-style boolean on the
  active tool) and the decal-to-wall raycast-normal behavior described in older
  community notes are `Needs Verification` against this pin - the snap system was
  rewritten as `ExtraSnapBase<TTool,TSnap>` and the field names differ from earlier
  versions.

- **Elevation nudge.** A lighter-weight variant exposes only up/down keybinds that edit
  the placement command's elevation field before it is submitted, rather than a full
  gizmo. Treat specific vanilla method/field names for this as `Needs Verification
  (in-game)` unless cited.

## See also
- Sibling tooling how-tos: [raycast filters](raycast-filters.md) (how the gizmo tool
  finds its target), [sub-element removal](sub-element-removal.md),
  [validation overrides](validation-overrides.md) (Anarchy's other placement freedoms).
- Reference: [technique index](../../technique-index.md) - family Y (event-driven
  ModificationEnd / ToolOutputBarrier) and family V (tool drag-select + Highlighted).
- Explanation: [serialization](../../explanation/serialization.md) (persisting
  `TransformRecord` across save/load),
  [multi-phase scheduling](../../explanation/multi-phase-scheduling.md).

## Sources
- Canonical mods (dossier + repo):
  - `anarchy` @a6311e898d20a775368668b234aaa32f06e3e1eb -
    `repo/Anarchy/Components/TransformRecord.cs`,
    `repo/Anarchy/Components/PreventOverride.cs`,
    `repo/Anarchy/Systems/AnarchyComponentsTool/AnarchyComponentsToolSystem.cs`,
    `repo/Anarchy/Systems/AnarchyComponentsTool/AnarchyComponentsToolUISystem.cs`
  - `extra-detailing-tools` @41df3c2b8b444197f7c3e2de4496c01cfd8bfc23 -
    `repo/MOD/Systems/Tools/TransformGizmoTool.cs` (structure verified; snap-field
    specifics Needs Verification)
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding
