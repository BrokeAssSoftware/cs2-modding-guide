---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: Custom snap-mode framework + in-scene raycast engine"
recipe: custom-snap-raycast-framework
technique_family: "AZ - Custom snap-mode framework + in-scene raycast engine"
diataxis: how-to
source_version: "~1.5.x (extra-detailing-tools@41df3c2; date-pinned, static source only)"
last_reverified: "2026-07-05"
status: source-verified
canonical_mods:
  - extra-detailing-tools@41df3c2b8b444197f7c3e2de4496c01cfd8bfc23
technique_applicability: [tooling]
Created: 2026-07-05
Updated: 2026-07-05
Owners:
  - codex
---

# Custom snap-mode framework + in-scene raycast engine

> Add new tool snap behaviours (surface-project, edge-snap) through a reusable
> reflective base class that Harmony-postfixes any `ToolBaseSystem`, and stand up a
> bespoke parallel raycast engine that hit-tests in-scene `GizmosData` entities so
> your custom handles are clickable - both decoupled from the vanilla tool internals.

## Problem
You are building an editor-style tool (a transform gizmo, a placement helper) and hit
two walls the stock tool API does not solve. First, you want a **new snap mode** - snap
a placed object onto another object's surface, or flush against its side - but the game's
`SnapControlPoint`/`InitializeRaycast` are private tool internals with a fixed set of
`Snap` flags. Second, the game's `ToolRaycastSystem` only raycasts *world* geometry
(terrain, nets, objects); it cannot tell you the mouse is over the **arrow of your own
gizmo**, because that handle is not a real world entity with collision. You need to
hit-test your own drawn handles.

## Solution
Two independent, reusable pieces from Extra Detailing Tools:

1. **A reflective snap-mode base class** - `ExtraSnapBase<TTool, TSnap>` owns its own
   Harmony instance, validates `TSnap` at type-load, reflectively finds the target
   tool's `SnapControlPoint`/`InitializeRaycast`, and postfixes them. Concrete
   subclasses supply Burst `IJob` snap projections. New snap logic never touches the
   patched method bodies - it is appended after them.
2. **An in-scene raycast engine** - `GizmosRaycastSystem` runs a Burst `IJobChunk` over
   all `GizmosData` entities, intersecting a camera ray (capsule/sphere/AABB/arrow/
   wire-arc, with a pixel-scale tolerance) and returning hits through a **context-keyed**
   `AddInput`/`GetResult` API. Any system can push a ray tagged with `this` and read back
   only its own results.

## Steps & Code

### 1. Define the generic snap base with a type-load contract on `TSnap`

The static constructor is the guardrail: it refuses any `TSnap` that is not a
`[Flags]`-marked, `uint`-backed enum, so a malformed snap enum fails loudly at class
load rather than mis-masking at runtime.

```csharp
public abstract class ExtraSnapBase<TTool, TSnap> : ExtraSnapBase, IDisposable
    where TTool : ToolBaseSystem
    where TSnap : Enum
{
    static ExtraSnapBase()
    {
        var type = typeof(TSnap);
        if (!type.IsDefined(typeof(FlagsAttribute), inherit: false))
            throw new InvalidOperationException($"{type.Name} must be marked with [Flags]");
        if (Enum.GetUnderlyingType(type) != typeof(uint))
            throw new InvalidOperationException($"{type.Name} must have uint as underlying type");
    }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/extra-detailing-tools/repo/MOD/ExtraSnap/ExtraSnap.cs#L50-L77` (@41df3c2b8b444197f7c3e2de4496c01cfd8bfc23)

### 2. Own a private Harmony instance and reflectively grab the tool methods

The instance constructor resolves the tool + support systems, then `Patch()` reflectively
locates `SnapControlPoint` and `InitializeRaycast` on `TTool` (public **or** non-public)
and throws `MissingMethodException` if the game renamed them - so a broken CS2 refactor
surfaces immediately.

```csharp
_harmony = new Harmony(GetType().FullName);   // one Harmony id per snap type
Patch();

private void Patch()
{
    var snapMethod = typeof(TTool).GetMethod("SnapControlPoint",
        BindingFlags.Instance | BindingFlags.Public | BindingFlags.NonPublic);
    if (snapMethod == null)
        throw new MissingMethodException(typeof(TTool).Name, "SnapControlPoint");
    var raycastMethod = typeof(TTool).GetMethod("InitializeRaycast",
        BindingFlags.Instance | BindingFlags.Public | BindingFlags.NonPublic);
    if (raycastMethod == null)
        throw new MissingMethodException(typeof(TTool).Name, "InitializeRaycast");
    // ... _harmony.Patch(snapMethod, postfix: PostfixSnapControlPoint) etc.
}
```
Source: `../../../vice-and-order-research/mods/dossiers/extra-detailing-tools/repo/MOD/ExtraSnap/ExtraSnap.cs#L95-L119` (@41df3c2b8b444197f7c3e2de4496c01cfd8bfc23)

### 3. Postfix chains your snap job after the vanilla result

The postfix reads the static `_instance`, and rewrites the tool's `JobHandle __result`
by scheduling the subclass's snap job *after* the game's own snapping - so vanilla snap
runs first and your mode refines the final control point.

```csharp
private static void PostfixSnapControlPoint(ref JobHandle __result)
{
    var instance = _instance;
    if (instance == null) return;
    __result = instance.SnapControlPoint(__result);   // append custom snap to the chain
}
```
Source: `../../../vice-and-order-research/mods/dossiers/extra-detailing-tools/repo/MOD/ExtraSnap/ExtraSnap.cs#L126-L133` (@41df3c2b8b444197f7c3e2de4496c01cfd8bfc23)

### 4. Subclass it: declare the snap enum and schedule a Burst snap job

`ObjectToolSystemExtraSnap` binds the framework to `ObjectToolSystem` with a two-value
`[Flags] uint` enum, and reads the tool's private state (`m_ControlPoints`, `m_Prefab`,
`m_MovingObject`) via HarmonyLib `Traverse` to feed a Burst `SnapJob`.

```csharp
public class ObjectToolSystemExtraSnap : ExtraSnapBase<ObjectToolSystem, ObjectToolExtraSnap>
{
    [Flags]
    public enum ObjectToolExtraSnap : uint { ObjectSurface, ObjectSide }

    protected override JobHandle SnapControlPoint(JobHandle inputDeps)
    {
        NativeList<ControlPoint> controlPoints =
            traverse.Field("m_ControlPoints").GetValue<NativeList<ControlPoint>>();
        return IJobExtensions.Schedule(new SnapJob
        {
            m_Snap = traverse.Method("GetActualSnap").GetValue<Snap>(),
            m_ControlPoints = controlPoints,
            m_ObjectGeometryData = _Tool.GetComponentLookup<ObjectGeometryData>(true),
            // ... prefab, selected, rotation, owner/transform lookups
        }, inputDeps);
    }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/extra-detailing-tools/repo/MOD/ExtraSnap/ObjectTool.cs#L27-L87` (@41df3c2b8b444197f7c3e2de4496c01cfd8bfc23)

### 5. Implement the snap projections inside the job

`ObjectSurface` walks the hit entity's `Owner` chain to the root transform and projects
onto that surface; `ObjectSide` builds oriented bounding boxes for the placed and target
prefabs and snaps to the nearest base-quad edge. Both are gated on the tool's live `Snap`
mask so they only fire when their mode is active.

```csharp
if ((m_Snap & Snap.ObjectSurface) != Snap.None
    && m_TransformData.HasComponent(controlPoint.m_OriginalEntity))
{
    int parentMesh = controlPoint.m_ElementIndex.x;
    Entity entity2 = controlPoint.m_OriginalEntity;
    while (m_OwnerData.HasComponent(entity2))          // climb owner chain to root
    {
        if (m_LocalTransformCacheData.HasComponent(entity2)
            && !m_ServiceUpgradeData.HasComponent(entity2))
            parentMesh = m_LocalTransformCacheData[entity2].m_ParentMesh + /* select */ 0;
        entity2 = m_OwnerData[entity2].m_Owner;
    }
    if (m_TransformData.HasComponent(entity2))
        SnapSurface(controlPoint, ref bestSnapPosition, entity2, parentMesh);
}
```
Source: `../../../vice-and-order-research/mods/dossiers/extra-detailing-tools/repo/MOD/ExtraSnap/ObjectTool.cs#L141-L158` (@41df3c2b8b444197f7c3e2de4496c01cfd8bfc23)

`InitializeRaycast` is the other half - it widens the tool's raycast mask so the surface
under the cursor is actually returned to snap against:

```csharp
protected override void InitializeRaycast()
{
    Snap snap = GetActualToolSnap();
    if ((snap & Snap.ObjectSide) != Snap.None)
    {
        _ToolRaycastSystem.typeMask |= TypeMask.StaticObjects;
        if (IsEditor) _ToolRaycastSystem.raycastFlags |= RaycastFlags.Placeholders;
    }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/extra-detailing-tools/repo/MOD/ExtraSnap/ObjectTool.cs#L43-L60` (@41df3c2b8b444197f7c3e2de4496c01cfd8bfc23)

### 6. Register the snap at mod init

One line in `OnLoad` constructs the subclass (running the type-load check + Harmony
patch) and stores it in the static registry keyed by type.

```csharp
ExtraSnapBase.RegisterInstance<ObjectToolSystemExtraSnap>();
```
Source: `../../../vice-and-order-research/mods/dossiers/extra-detailing-tools/repo/EDT.cs#L110` (@41df3c2b8b444197f7c3e2de4496c01cfd8bfc23)

### 7. Model your in-scene handles as `GizmosData` entities

The raycast engine hit-tests ECS entities carrying a single `GizmosData` component - a
tagged union of primitive shapes (line, sphere, cube, arrow, wire-arc) with generic
point/param slots. These are the same entities the render pass draws, so what you see is
exactly what you can click.

```csharp
public enum GizmoType { Line, Bezier, ArrowHead, Arrow, Sphere, Cube, WireArc, Cylinder, Cone, Capsule, Frustum }

public struct GizmosData : IComponentData
{
    public GizmoType Type;
    public float3 A, B, C, D;      // generic points
    public float4 Params0, Params1; // radius / size / angles
    public float4x4 TRS;
    public Color Color;
    public int Segments, Flags;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/extra-detailing-tools/repo/MOD/Gizmos/GizmosData.cs#L12-L49` (@41df3c2b8b444197f7c3e2de4496c01cfd8bfc23)

### 8. Push a ray with a context key, read back your own hits

An input is a camera-ray segment plus a `[Flags] uint` type mask and a tolerance.
`AddInput(context, input)` tags the ray with an arbitrary object (`this`); `GetResult(context)`
returns the sub-array of hits for that same key - so multiple tools share one engine
without cross-reading.

```csharp
[Flags] public enum GizmosRaycastType : uint
{ Line = 1<<0, Arrow = 1<<2, Sphere = 1<<4, Cube = 1<<5, WireArc = 1<<6, /* ... */ ALL = uint.MaxValue }

public struct GizmosRaycastInput
{
    public GizmosRaycastType m_Type;
    public Line3.Segment m_Line;
    public float m_Tolerance;
    public bool m_Debug;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/extra-detailing-tools/repo/MOD/Gizmos/GizmosRaycastInput.cs#L10-L37` (@41df3c2b8b444197f7c3e2de4496c01cfd8bfc23)

The caller side (from the transform-gizmo tool) - build the ray from the camera and tag it
with `this`:

```csharp
GizmosRaycastInput input = new GizmosRaycastInput()
{
    m_Type = GizmosRaycastType.ALL,
    m_Line = ToolRaycastSystem.CalculateRaycastLine(Camera.main),
    m_Tolerance = m_Mode == Mode.Rotate ? 0.1f : 0,
    m_Debug = false
};
m_GimzosRaycastSystem.AddInput(this, input);
```
Source: `../../../vice-and-order-research/mods/dossiers/extra-detailing-tools/repo/MOD/Systems/Tools/TransformGizmoTool.cs#L915-L923` (@41df3c2b8b444197f7c3e2de4496c01cfd8bfc23)

### 9. The engine: Burst chunk job over `GizmosData`, results into a `NativeAccumulator`

`GizmoRaycastJob` is an `IJobChunk` that, for every gizmo and every queued input, type-masks,
intersects the correct primitive, and accumulates the hit into a
`NativeAccumulator<RaycastResult>.ParallelWriter` keyed by input index.

```csharp
if (!MatchType(gizmo.Type, input.m_Type)) continue;
if (Intersect(gizmo, input.m_Line, input.m_Tolerance, out RaycastHit hit))
{
    hit.m_HitEntity = entity;
    Accumulator.Accumulate(inputIndex, new RaycastResult { m_Hit = hit, m_Owner = entity });
}
// Intersect() dispatches: Line->capsule, Sphere->sphere, Cube->AABB, Arrow->arrow, WireArc->arc
```
Source: `../../../vice-and-order-research/mods/dossiers/extra-detailing-tools/repo/MOD/Gizmos/GizmosRaycastSystem.cs#L77-L124` (@41df3c2b8b444197f7c3e2de4496c01cfd8bfc23)

`OnUpdate` computes a `pixelScale` from the camera FOV and pixel height (so the tolerance
is screen-space constant regardless of zoom), schedules the parallel raycast into a
fresh `NativeAccumulator`, then copies results out:

```csharp
float tanFOV = math.tan(math.radians(cam.fieldOfView) * 0.5f);
float pixelScale = 2f * tanFOV / cam.pixelHeight;   // screen-space tolerance scale
```
Source: `../../../vice-and-order-research/mods/dossiers/extra-detailing-tools/repo/MOD/Gizmos/GizmosRaycastSystem.cs#L460-L462` (@41df3c2b8b444197f7c3e2de4496c01cfd8bfc23)

### 10. Consume results by the same context key

Back in the tool, read only your ray's hits and act on the first one:

```csharp
NativeArray<RaycastResult> raycastResults = m_GimzosRaycastSystem.GetResult(this);
if (raycastResults.Length <= 0) return inputDeps;
RaycastResult raycastResult = raycastResults[0];
if (raycastResult.m_Owner == Entity.Null) return inputDeps;
StartDragging(raycastResult.m_Hit);
```
Source: `../../../vice-and-order-research/mods/dossiers/extra-detailing-tools/repo/MOD/Systems/Tools/TransformGizmoTool.cs#L1122-L1130` (@41df3c2b8b444197f7c3e2de4496c01cfd8bfc23)

The `GetResult` lookup scans the context list linearly and returns a `GetSubArray` slice;
a missing key logs a warning and returns `default` (see Pitfalls).
Source: `../../../vice-and-order-research/mods/dossiers/extra-detailing-tools/repo/MOD/Gizmos/GizmosRaycastSystem.cs#L512-L536` (@41df3c2b8b444197f7c3e2de4496c01cfd8bfc23)

## Pitfalls & gotchas

- **This closes a `Needs-Verification` gap.** The transform-gizmo reference flags the
  snap wiring as unverified; this recipe resolves it from source: the snap modes are
  wired by the reflective `ExtraSnapBase.Patch()` postfixing `SnapControlPoint` /
  `InitializeRaycast`
  (`MOD/ExtraSnap/ExtraSnap.cs#L100-L119`), not by any vanilla snap-registration API.

- **Heavy reflection = fragile across CS2 tool refactors.** The framework reflectively
  binds `"SnapControlPoint"` / `"InitializeRaycast"` by string, and the concrete subclass
  `Traverse`s private fields by string (`"m_ControlPoints"`, `"m_Prefab"`,
  `"m_MovingObject"`, `"GetActualSnap"`). A rename in a game patch throws
  `MissingMethodException` at patch time (loud, good) or - worse - a `Traverse` field
  miss surfaces only when the tool runs. Decoupling is the payoff; the maintenance cost is
  re-verifying every reflected name each CS2 release.
  Source: `../../../vice-and-order-research/mods/dossiers/extra-detailing-tools/repo/MOD/ExtraSnap/ObjectTool.cs#L62-L74` (@41df3c2b8b444197f7c3e2de4496c01cfd8bfc23)

- **The type-load contract fires before you can catch it.** The `[Flags]` + `uint` check
  is in the **static constructor** of `ExtraSnapBase<TTool,TSnap>`, so a malformed `TSnap`
  throws a `TypeInitializationException` the first time the closed generic is touched -
  not at `RegisterInstance`. Keep the enum `[Flags] : uint` from the start.

- **Raycast targets are ENTITIES, not MonoBehaviours.** The engine only sees entities in
  the `GizmosData` query (`GetEntityQuery(ComponentType.ReadOnly<GizmosData>())`). A handle
  drawn any other way (immediate-mode, a `GameObject`) is invisible to it. Model every
  clickable handle as a `GizmosData` entity - which is also what the render pass consumes,
  keeping draw and hit-test in lockstep.
  Source: `../../../vice-and-order-research/mods/dossiers/extra-detailing-tools/repo/MOD/Gizmos/GizmosRaycastSystem.cs#L419` (query definition), `../../../vice-and-order-research/mods/dossiers/extra-detailing-tools/repo/MOD/Gizmos/GizmosRaycastSystem.cs#L484` (job scheduled over that query) (@41df3c2b8b444197f7c3e2de4496c01cfd8bfc23)

- **Context key is reference-equality on `object`.** `GetResult` compares stored contexts
  with `==` and warns + returns `default(NativeArray<RaycastResult>)` when the key is not
  found. Pass a stable reference (the tool passes `this`); a boxed value or a fresh object
  each frame will silently never match.
  Source: `../../../vice-and-order-research/mods/dossiers/extra-detailing-tools/repo/MOD/Gizmos/GizmosRaycastSystem.cs#L512-L527` (@41df3c2b8b444197f7c3e2de4496c01cfd8bfc23)

- **One-frame input lifecycle.** `CompleteRaycast()` clears the input + context lists after
  completing the job, so inputs are consumed per update - you must `AddInput` every frame
  you want a hit, not once.
  Source: `../../../vice-and-order-research/mods/dossiers/extra-detailing-tools/repo/MOD/Gizmos/GizmosRaycastSystem.cs#L502-L510` (@41df3c2b8b444197f7c3e2de4496c01cfd8bfc23)

- **`[BurstCompile]` is gated on `#if RELEASE` for the snap job.** `ObjectTool.cs`'s
  `SnapJob` is only Burst-compiled in RELEASE builds; the gizmo raycast job's inner
  `RaycastResultJob` is unconditionally `[BurstCompile]`. Do not assume uniform Burst
  behaviour when debugging a debug build.
  Source: `../../../vice-and-order-research/mods/dossiers/extra-detailing-tools/repo/MOD/ExtraSnap/ObjectTool.cs#L89-L92` (@41df3c2b8b444197f7c3e2de4496c01cfd8bfc23)

- **In-game snap correctness is `Needs Verification (in-game)`.** The geometry of
  `ObjectSide` OBB edge-snapping and the surface owner-chain projection is present in
  source, but whether the final snapped placement matches player expectation across
  compound/sub-object prefabs is not provable from code - **Needs Verification (in-game)**.

## Variations

- **Bind a different tool.** The base is generic over `TTool : ToolBaseSystem`, so the same
  framework can postfix `NetToolSystem`, a custom tool, etc. - as long as that tool exposes
  `SnapControlPoint` and `InitializeRaycast`. Provide a new subclass + snap enum and call
  `ExtraSnapBase.RegisterInstance<T>()`.

- **Add a snap mode to an existing subclass.** Extend the `[Flags]` enum and add another
  gated block in the job's `Execute` (the mod stacks `ObjectSurface` and `ObjectSide` as
  mask-gated blocks in one job body, and a vanilla `AutoParent` block further down the same
  `Execute`).
  Source: `../../../vice-and-order-research/mods/dossiers/extra-detailing-tools/repo/MOD/ExtraSnap/ObjectTool.cs#L141-L160` (ObjectSurface + ObjectSide), `../../../vice-and-order-research/mods/dossiers/extra-detailing-tools/repo/MOD/ExtraSnap/ObjectTool.cs#L267` (AutoParent) (@41df3c2b8b444197f7c3e2de4496c01cfd8bfc23)

- **Reuse the raycast engine for any handle UI, not just gizmos.** Because the engine keys
  results by an arbitrary `object` context and the render side draws the same `GizmosData`
  entities, you can drive scene widgets (rotate rings, scale cubes, snap markers) off one
  shared `GizmosRaycastSystem` - each caller tags with its own `this`.
  Source: `../../../vice-and-order-research/mods/dossiers/extra-detailing-tools/repo/MOD/Gizmos/GizmosRenderSystem.cs#L27-L43` (@41df3c2b8b444197f7c3e2de4496c01cfd8bfc23)

- **Debug visualisation path.** Set `m_Debug = true` on an input; `NeedDebug()` then routes
  the raycast job through the game's `GizmosBatcher` so intersections are drawn - useful
  while calibrating tolerance.
  Source: `../../../vice-and-order-research/mods/dossiers/extra-detailing-tools/repo/MOD/Gizmos/GizmosRaycastSystem.cs#L456-L491` (@41df3c2b8b444197f7c3e2de4496c01cfd8bfc23)

## See also
- Tooling: [transform gizmos](../tooling/transform-gizmos.md) (the tool this framework
  serves; this recipe closes its snap-wiring `Needs-Verification` note),
  [raycast filters](../tooling/raycast-filters.md) (masking the *game's* `ToolRaycastSystem`).
- Explanation: [Harmony patching](../../explanation/harmony-patching.md) (postfix chaining,
  per-instance Harmony ids, reflective method binding).
- Case study: [extra-detailing-suite](../../case-studies/extra-detailing-suite.md).

## Sources
- Canonical mod (dossier + repo):
  - `extra-detailing-tools` @41df3c2b8b444197f7c3e2de4496c01cfd8bfc23 -
    `repo/MOD/ExtraSnap/ExtraSnap.cs`, `repo/MOD/ExtraSnap/ObjectTool.cs`,
    `repo/MOD/Gizmos/GizmosData.cs`, `repo/MOD/Gizmos/GizmosRaycastInput.cs`,
    `repo/MOD/Gizmos/GizmosRaycastSystem.cs`, `repo/MOD/Gizmos/GizmosRenderSystem.cs`,
    `repo/MOD/Systems/Tools/TransformGizmoTool.cs`, `repo/EDT.cs`
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding
