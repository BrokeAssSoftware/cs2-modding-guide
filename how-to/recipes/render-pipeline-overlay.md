---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: Render-pipeline overlay"
recipe: render-pipeline-overlay
technique_family: "I - Render-pipeline overlay (beginContextRendering / DrawMesh)"
diataxis: how-to
source_version: "~1.5.x (write-everywhere@13c70eb; date-pinned, static source only)"
last_reverified: "2026-07-04"
status: source-verified
canonical_mods:
  - write-everywhere@13c70eb04e6bed152257c516a982455148f591a5
  - road-speed-adjuster@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed
  - advanced-road-naming@559e72cdb3180e3e71869643367e094a22eed988
technique_applicability: [ui]
Created: 2026-07-04
Updated: 2026-07-05
Owners:
  - codex
---

# Render-pipeline overlay

> Draw your own meshes or gizmos into the 3D world each frame - either by hooking
> the Scriptable Render Pipeline (SRP) callback and calling `Graphics.DrawMesh`
> yourself, or by borrowing the game's built-in `OverlayRenderSystem` gizmo buffer.

## Problem
You want to paint something into the game world that CS2 does not draw for you:
text labels floating over roads, a speed number above an edge, a highlighted route,
a selection halo. This is not UI panel work (that is HTML/React in the coherent UI
layer) - it is geometry rendered in world space, camera-aware, every frame. You need
a hook that fires during rendering, a camera to project against, and a mesh +
material (or a gizmo primitive) to emit.

## Solution
There are **two distinct techniques** in this family, and picking the wrong one
wastes effort:

1. **SRP `DrawMesh` overlay** - subscribe to
   `RenderPipelineManager.beginContextRendering`, and inside that callback call
   `Graphics.DrawMesh(mesh, matrix, material, ...)` for each thing you want drawn.
   You own the mesh, the material, and the transform matrix (position/rotation/scale,
   typically camera-facing). This is maximum control - arbitrary meshes, custom
   shaders, per-camera billboarding - at the cost of managing render state yourself.
   Write Everywhere and Road Speed Adjuster both do this (Road Speed Adjuster
   explicitly models its render system on Write Everywhere's approach).

2. **Vanilla `OverlayRenderSystem` gizmo buffer** - get the game's shared overlay
   buffer with `OverlayRenderSystem.GetBuffer(...)`, `Complete()` its job handle, and
   call `buffer.DrawCurve(...)` / `buffer.DrawCircle(...)`. No SRP hook, no mesh, no
   material - the game renders the primitives for you. Far less code, but you are
   limited to the primitive gizmos the buffer exposes (lines, curves, circles).
   Advanced Road Naming draws all its route highlights this way.

Reach for technique 1 when you need custom meshes or text; reach for technique 2 when
lines and circles are enough.

## Steps & Code

### Technique 1 - SRP `DrawMesh` overlay

#### 1. Hook `beginContextRendering` when the system is created

Subscribe your render callback to the static `RenderPipelineManager.beginContextRendering`
event. Write Everywhere does this in its create hook, after caching the sibling systems
it needs:

```csharp
protected unsafe override void OnCreateWithBarrier()
{
    m_pickerController = World.GetExistingSystemManaged<WEWorldPickerController>();
    m_pickerTool = World.GetExistingSystemManaged<WEWorldPickerTool>();
    m_wePreCullSys = World.GetExistingSystemManaged<WEPreCullingSystem>();
    m_atlasesLibrary = World.GetExistingSystemManaged<WEAtlasesLibrary>();
    RenderPipelineManager.beginContextRendering += Render;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/write-everywhere/repo/BelzontWE/Systems/WERendererSystem.cs#L37-L44` (@13c70eb04e6bed152257c516a982455148f591a5)

#### 2. Unhook on destroy (non-negotiable)

The event is static, so a dangling subscription leaks and fires against a dead system.
Always remove the handler in `OnDestroy`:

```csharp
protected override void OnDestroy()
{
    base.OnDestroy();
    RenderPipelineManager.beginContextRendering -= Render;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/write-everywhere/repo/BelzontWE/Systems/WERendererSystem.cs#L48-L52` (@13c70eb04e6bed152257c516a982455148f591a5)

#### 3. The callback signature and the `DrawMesh` call

The handler receives the render context and the frame's cameras. Inside, per item, you
compute a transform matrix and emit a mesh. Write Everywhere's actual draw call passes
the full `Graphics.DrawMesh` overload (layer, camera, submesh, property block, shadow
mode, light-probe usage):

```csharp
private void Render(ScriptableRenderContext context, List<Camera> cameras)
{
    // ... resolve mesh (geomMesh), material (ownMaterial[i]) and effectiveMatrix per item ...
    Graphics.DrawMesh(geomMesh, effectiveMatrix, ownMaterial[i], 0, null, 0, null,
        ShadowCastingMode.TwoSided, true, null, LightProbeUsage.BlendProbes);
}
```
Source: `../../../vice-and-order-research/mods/dossiers/write-everywhere/repo/BelzontWE/Systems/WERendererSystem.cs#L56`, `#L179` (@13c70eb04e6bed152257c516a982455148f591a5)

#### 4. Build a camera-facing matrix (billboarding + distance scale)

`DrawMesh` needs a `Matrix4x4`. To make world-space text face the camera and stay
legible when zoomed out, Road Speed Adjuster rotates toward the camera and scales by
distance, then draws once **per camera** (passing the specific `camera` to `DrawMesh`):

```csharp
foreach (Camera camera in cameras)
{
    if (camera.cameraType == CameraType.Game || camera.cameraType == CameraType.SceneView)
    {
        Quaternion rotation = Quaternion.LookRotation(camera.transform.forward, camera.transform.up);
        float distance = Vector3.Distance(camera.transform.position, position);
        float dynamicScale = Mathf.Clamp(0.6f * Mathf.Sqrt(distance / 100f), 0.3f, 2.5f);
        Vector3 scale = new Vector3(dynamicScale, dynamicScale, dynamicScale);
        Matrix4x4 matrix = Matrix4x4.TRS(position, rotation, scale);
        Graphics.DrawMesh(meshInfo.Mesh, matrix, meshInfo.Material, 0, camera, 0, null,
            castShadows: false, receiveShadows: false);
    }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/Systems/SpeedLimitRenderSystem.cs#L207-L238` (@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed)

#### 5. Source the mesh + material (text via TextMeshPro)

For text you need a real mesh. Road Speed Adjuster borrows the SDF font setup from the
vanilla `OverlayRenderSystem.GetTextMesh()`, bakes the `TMP_MeshInfo` into a plain
Unity `Mesh`, clones the TMP material, and caches the pair per speed value:

```csharp
TextMeshPro textMesh = m_OverlayRenderSystem.GetTextMesh();  // borrow the game's SDF text component
// ... configure font size, alignment, build speedText ...
TMP_MeshInfo tmpMeshInfo = textMesh.GetTextInfo(speedText).meshInfo[0];
Mesh mesh = new Mesh();
mesh.vertices  = tmpMeshInfo.vertices;
mesh.triangles = tmpMeshInfo.triangles;
mesh.uv        = tmpMeshInfo.uvs0;
mesh.colors32  = tmpMeshInfo.colors32;
mesh.RecalculateBounds();
Material material = new Material(tmpMeshInfo.material);
m_OverlayRenderSystem.CopyFontAtlasParameters(tmpMeshInfo.material, material);  // keep SDF setup correct
```
Source: `../../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/Systems/SpeedLimitRenderSystem.cs#L260`, `#L301-L334` (@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed)

The cached `Mesh`/`Material` pair is destroyed explicitly in `OnDestroy` (see Pitfalls);
you own that lifetime.

### Technique 2 - vanilla `OverlayRenderSystem` gizmo buffer

#### 6. Resolve `OverlayRenderSystem` in `OnCreate`

No SRP hook. Just cache the game's overlay system:

```csharp
protected override void OnCreate()
{
    base.OnCreate();
    _toolSystem = World.GetOrCreateSystemManaged<RoadRouteToolSystem>();
    _geometrySystem = World.GetOrCreateSystemManaged<RoadRouteOverlayGeometrySystem>();
    _overlayRenderSystem = World.GetOrCreateSystemManaged<OverlayRenderSystem>();
}
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Systems/RoadRouteHighlightSystem.cs#L44-L50` (@559e72cdb3180e3e71869643367e094a22eed988)

#### 7. Get the buffer, complete its job handle, draw primitives

In `OnUpdate` (not an SRP callback), grab the buffer, `Complete()` the returned job
handle before writing to it, then emit primitives. Advanced Road Naming draws route
curves and waypoint circles for saved/active/preview/hover layers:

```csharp
protected override void OnUpdate()
{
    if (_toolSystem == null || !_toolSystem.IsRunning || _overlayRenderSystem == null || _geometrySystem == null)
        return;

    var buffer = _overlayRenderSystem.GetBuffer(out var renderBufferJobHandle);
    renderBufferJobHandle.Complete();

    DrawGeometry(buffer, _geometrySystem.ActiveCurves, RouteColor, RouteWidth);
    DrawNodes(buffer, _geometrySystem.ActiveNodes, WaypointColor, WaypointRadius);
    // ... saved / preview / hover layers ...
}
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Systems/RoadRouteHighlightSystem.cs#L52-L71` (@559e72cdb3180e3e71869643367e094a22eed988)

#### 8. The primitive calls

The buffer exposes the gizmos directly - no mesh, no material, no per-camera loop:

```csharp
for (var i = 0; i < curves.Count; i++)
    buffer.DrawCurve(lineColor, curves[i], lineWidth, RoundedLine);   // RoundedLine = new float2(0f, 1f)
// ...
for (var i = 0; i < nodes.Count; i++)
    buffer.DrawCircle(color, nodes[i], radius);
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Systems/RoadRouteHighlightSystem.cs#L95-L110` (@559e72cdb3180e3e71869643367e094a22eed988)

#### 9. Build the curves in a pure geometry helper (turn-adaptive cubic joins)

The `ActiveCurves` list the highlight system draws (step 7) is not raw segment geometry.
Advanced Road Naming runs the route's edges through a static
`RouteOverlayGeometryBuilder` that trims each segment back from a junction and stitches a
cubic Bezier connector across the gap, so a multi-segment route reads as one continuous
ribbon instead of overlapping stubs. The trim distance is **turn-adaptive** - it measures
how sharply the two segments meet (`turnAmount`, from the dot product of the endpoint
tangents) and lerps the trim between 4 m and 20 m, so gentle joins barely trim and hard
turns trim more:

```csharp
var alignment  = math.clamp(math.dot(routeIn, routeOut), -1f, 1f);
var turnAmount = math.saturate((1f - alignment) * 0.5f);
var desiredTrimDistance = math.lerp(MinJoinTrimDistance, MaxJoinTrimDistance, turnAmount); // 4f..20f
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Services/RouteOverlayGeometryBuilder.cs#L164-L167` (@559e72cdb3180e3e71869643367e094a22eed988)

It then builds the connector as a `Bezier4x3`, picking straight vs curved control handles
by the same alignment threshold and forcing the control points level (`.y = start/end.y`)
so the ribbon does not roll around its center:

```csharp
if (connection.Alignment > StraightJoinThreshold)   // 0.996f - near-collinear join
{
    var straightDirection = math.normalizesafe(directionIntoJoin + directionOutOfJoin, directionIntoJoin);
    control1 = start + straightDirection * straightHandle;
    control2 = end   - straightDirection * straightHandle;
}
else
{
    control1 = start + directionIntoJoin  * handleDistance;
    control2 = end   - directionOutOfJoin * handleDistance;
}
control1.y = start.y;  control2.y = end.y;           // keep the connector flat
joinCurve = new Bezier4x3(start, control1, control2, end);
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Services/RouteOverlayGeometryBuilder.cs#L212-L229` (@559e72cdb3180e3e71869643367e094a22eed988)

`RoadRouteOverlayGeometrySystem` calls this builder to fill `ActiveCurves`;
`RoadRouteHighlightSystem` (step 7) only reads that list and emits `buffer.DrawCurve`.
That is the full two-stage split - geometry math in one system, per-frame gizmo emit in
another - and it is the same shape whichever technique you pick.

## Pitfalls & gotchas

- **Leaked static subscription.** `RenderPipelineManager.beginContextRendering` is a
  static event. If you `+=` in create but forget to `-=` in `OnDestroy`, the handler
  survives the system and fires against disposed state. Both SRP mods pair the hook and
  unhook exactly (Write Everywhere `WERendererSystem.cs#L43` / `#L51`; Road Speed
  Adjuster `SpeedLimitRenderSystem.cs#L91` / `#L97`). Always unhook.

- **You own the Mesh/Material lifetime (technique 1).** Meshes and materials you `new`
  are Unity objects and leak if not destroyed. Road Speed Adjuster caches per-value
  meshes and destroys both mesh and material in `OnDestroy`, and also clears the cache
  when the map theme or unit setting changes so stale text is not drawn
  (`SpeedLimitRenderSystem.cs#L108-L118`, `#L120-L162`). If you cache generated meshes,
  invalidate them when their inputs change.

- **The render callback runs every frame - gate it hard.** `beginContextRendering`
  fires constantly. Do the minimum: bail early when your tool/overlay is inactive.
  Road Speed Adjuster returns immediately unless its tool is active and overlays are not
  hidden, and wraps the whole body in try/catch so a render exception cannot spam or
  crash the frame loop (`SpeedLimitRenderSystem.cs#L166-L175`, `#L243-L246`).

- **Draw once per camera, and filter camera types (technique 1).** The callback hands
  you a `List<Camera>`; the game may include cameras you should not draw to. Road Speed
  Adjuster only renders for `CameraType.Game` and `CameraType.SceneView`
  (`SpeedLimitRenderSystem.cs#L207-L209`). Passing the specific `camera` to `DrawMesh`
  (argument 5) scopes the draw to that camera.

- **Complete the buffer's job handle before writing (technique 2).**
  `OverlayRenderSystem.GetBuffer` returns a `JobHandle` out-param; Advanced Road Naming
  calls `renderBufferJobHandle.Complete()` before any `DrawCurve`/`DrawCircle`
  (`RoadRouteHighlightSystem.cs#L57-L58`). Skipping the `Complete()` races the game's
  overlay jobs.

- **Gizmo buffer cannot draw text or arbitrary meshes.** Technique 2 is lines/curves/
  circles only. Advanced Road Naming at this pin (@559e72c) draws **no floating labels
  at all** - only route geometry and waypoint gizmos via the buffer. If you need text,
  you are back to technique 1 (bake a TMP mesh and `DrawMesh` it).

- **Pin drift for Advanced Road Naming.** Cite at `559e72c`. The working tree there has
  advanced past this (a later "1.7 Update" HEAD), so read the pinned snapshot with
  `git show 559e72c:...`, not the checked-out file, or the line numbers will not match.

- **Exact render ordering / z-fighting vs vanilla overlays** is
  `Needs Verification (in-game)` - the draw calls are visible in source but their layering
  against other overlays is a runtime-visual property not provable from code.

- **Give the whole render path one global kill switch (technique 1).** Write Everywhere
  gates its entire pipeline behind a single settings flag: both the SRP `Render` callback
  and the `WEPreCullingSystem` update `return` immediately when
  `WriteEverywhereCS2Mod.WeData.TempDisableRendering` is set (`WERendererSystem.cs#L58`,
  `WEPreCullingSystem.cs#L62`; flag declared in `WEModData.cs#L112`). A one-line
  top-of-callback bail is the cheapest way to shut an overlay off in the field without
  tearing down systems.

- **Cap recursion when you draw nested layouts (technique 1).** Write Everywhere's layout
  tree can nest sub-layouts, and it draws them by recursing `DrawTree(..., nthCall + 1)`.
  The first line of `DrawTree` is a hard depth cap - `if (nthCall >= 16) return;` - so a
  self-referential or pathological layout cannot blow the stack or lock the frame
  (`WEPreCullingSystem.cs#L427-L429`). Any recursive per-frame draw walk needs a guard
  like this.

- **A one-frame "dump" flag is a cheap render diagnostic.** To debug why an item is or is
  not drawn, Write Everywhere exposes a one-shot static `dumpNextFrame` flag: a controller
  action sets it true (`WEWorldPickerController.cs#L132`), the render loop logs every item
  it touches for that single frame, then clears the flag at the end of `Render`
  (`WERendererSystem.cs#L26`, `#L129`, `#L182-L188`, `#L197`). One frame of targeted
  logging beats a breakpoint inside a callback that fires every frame.

## Variations

- **Custom shader / decal projection instead of billboarded text.** Write Everywhere's
  draw path selects meshes and materials by shader type and issues a second `DrawMesh`
  with a semi-transparent material to visualize decal projection when editing
  (`WERendererSystem.cs#L180-L184`). The same `beginContextRendering` + `DrawMesh` hook
  supports arbitrary materials, not just SDF text.

- **Precompute geometry in a separate system, draw in another.** Advanced Road Naming
  splits work: `RoadRouteOverlayGeometrySystem` builds the curve/node lists, and
  `RoadRouteHighlightSystem` only reads those lists and emits gizmos
  (`RoadRouteHighlightSystem.cs#L60-L70`). Keeping the per-frame draw system thin (no
  geometry math) is the cleaner shape for either technique.

- **Mesh caching keyed by content.** For repeated identical draws (same speed number on
  many roads), build the mesh once and reuse it - Road Speed Adjuster keys a
  `Dictionary<int, TextMeshInfo>` by speed value (`SpeedLimitRenderSystem.cs#L45`,
  `#L192-L196`) so N roads at 50 km/h share one mesh.

- **Precompute per-entity geometry in a Burst job, cache it as a buffer.** Mesh caching
  keys the *output* by content; for overlays anchored to network nodes you can also cache
  the *input geometry* on the entity. Write Everywhere does not recompute node geometry in
  the render loop - a `WENodeExtraDataUpdater` system runs a `[BurstCompile] IJobChunk`
  (`NodeCacheCalculation`) over nodes that still lack the cache and writes a
  `WENetNodeInformation` dynamic buffer (per-edge azimuth, center point, ref point,
  version hash) onto each node entity; the query's `None` filter means each node is
  computed once and skipped thereafter
  (`WENodeExtraDataUpdater.cs#L39-L51`, `#L57-L73`, `#L82-L91`). The render/precull side
  then reads the cached buffer instead of doing curve math every frame.

## See also
- Explanation: [React UI in CS2](../../explanation/react-ui.md) (the *other* UI layer -
  panels/HTML, not world-space geometry; use that when you are not drawing in 3D).
- Related recipes: [custom names via NameSystem](namesystem-custom-names.md) (naming
  data that overlays like Advanced Road Naming's visualize);
  [HDRP light from ECS](hdrp-light-from-ecs.md) (the sibling "render something per entity
  each frame" recipe on the lighting side - real HDRP lights per entity rather than
  meshes/gizmos).
- Case studies demonstrating it:
  [Write Everywhere ecosystem](../../case-studies/write-everywhere-ecosystem.md),
  [Advanced Road Naming](../../case-studies/advanced-road-naming.md).

## Sources
- Canonical mods (dossier + repo):
  - `write-everywhere` @13c70eb04e6bed152257c516a982455148f591a5 - `repo/BelzontWE/Systems/WERendererSystem.cs`, `repo/BelzontWE/Systems/WEPreCullingSystem.cs`, `repo/BelzontWE/Systems/WENodeExtraDataUpdater.cs`, `repo/BelzontWE/Controllers/WEWorldPickerController.cs`, `repo/BelzontWE/WEModData.cs`
  - `road-speed-adjuster` @e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed - `repo/Systems/SpeedLimitRenderSystem.cs`
  - `advanced-road-naming` @559e72cdb3180e3e71869643367e094a22eed988 - `repo/Systems/RoadRouteHighlightSystem.cs`, `repo/Services/RouteOverlayGeometryBuilder.cs`
- Official/community references (link out, do not duplicate): https://cs2.paradoxwikis.com/Modding
