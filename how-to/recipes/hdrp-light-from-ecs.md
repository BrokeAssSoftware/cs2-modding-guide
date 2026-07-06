---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: HDRP light-per-entity from ECS render data"
recipe: hdrp-light-from-ecs
technique_family: "BE - HDRP light-per-entity from ECS render data"
diataxis: how-to
source_version: "~1.6.0f1 (write-everywhere@13c70eb; date-pinned, static source only)"
last_reverified: "2026-07-05"
status: source-verified
canonical_mods:
  - write-everywhere@13c70eb04e6bed152257c516a982455148f591a5
technique_applicability: [ui]
Created: 2026-07-05
Updated: 2026-07-05
Owners:
  - codex
---

# HDRP light-per-entity from ECS render data

> Make an overlay actually illuminate the scene: spawn one managed HDRP light
> (`HDAdditionalLightData`) per emissive ECS entity, driven every frame from the
> same pre-culled render data your mesh overlay already uses.

## Problem
You render custom geometry into the world - text, decals, glowing signs - via a
render-pipeline overlay ([render-pipeline-overlay](render-pipeline-overlay.md)),
and now you want that geometry to cast **real** light onto nearby surfaces, not
just glow as an emissive material. CS2 places its own dynamic lights through the
internal `HDRPDotsInputs` DOTS buffer, but that buffer is not reachable from mod
code without reflection. You need a way to add HDRP lights that follow your ECS
entities frame-by-frame and clean themselves up when the entities disappear.

## Solution
Bridge ECS to managed HDRP by keeping **one `GameObject` + `HDAdditionalLightData`
per light-emitting entity**, tracked in a `Dictionary<Entity, LightEntry>`. Each
frame, walk the same pre-culled draw list your overlay uses; for every entity whose
material actually emits (`Shader == Default`, `EmissiveIntensityEffective > 0`,
`UseGlobalLight`), create the light on first sight, then push position, rotation,
color, and intensity onto its `HDAdditionalLightData`. Choose the HDRP light shape
from the entity's mesh type (cube-like -> `BoxSpot`, flat/text -> `RectangleArea`)
and remember the two shapes take **different intensity units** (Lux vs Lumen).
Deactivate lights whose entity left the visible set, and `Destroy` the GameObject
when the entity no longer exists. This mirrors CS2's own `LightCullingSystem`
defaults but uses managed light objects because the DOTS light buffer is inaccessible.

## Steps & Code

### 1. Run in the EndFrame phase and hold the per-entity light map

The system runs after simulation, keyed by `Entity`, alongside a per-frame "seen"
set and an orphan scratch list. `LumensPerUnit` is the single tuning constant that
converts one material emissive unit into HDRP light output:

```csharp
protected override AllowedPhase UpdatePhase => AllowedPhase.EndFrame;
private const float LumensPerUnit = 25f;

private struct LightEntry
{
    public GameObject go;
    public Light light;
    public HDAdditionalLightData hdLight;
    public WESimulationTextType textType;
}

private readonly Dictionary<Entity, LightEntry> m_lights = new();
private readonly HashSet<Entity> m_seenThisFrame = new();
private readonly List<Entity> m_orphaned = new();
```
Source: `../../../vice-and-order-research/mods/dossiers/write-everywhere/repo/BelzontWE/Systems/WEEmissiveLightSystem.cs#L20-L37` (@13c70eb04e6bed152257c516a982455148f591a5)

### 2. Gate on "actually emits" and hide the light when it does not

Iterate the pre-culled draw list, read the material and mesh components, and only
keep a light live when all three emissive conditions hold. When they fail, the
existing GameObject is deactivated (not destroyed) so it can be cheaply reused:

```csharp
if (materialData.Shader != WEShader.Default
    || materialData.EmissiveIntensityEffective <= 0f
    || !materialData.UseGlobalLight)
{
    if (m_lights.TryGetValue(item.textDataEntity, out var existing))
        existing.go.SetActive(false);
    continue;
}
m_seenThisFrame.Add(item.textDataEntity);
```
Source: `../../../vice-and-order-research/mods/dossiers/write-everywhere/repo/BelzontWE/Systems/WEEmissiveLightSystem.cs#L71-L79` (@13c70eb04e6bed152257c516a982455148f591a5)

### 3. Create the HDRP light, picking shape and unit from the mesh type

`AddHDLight` (HDRP's `GameObject` extension) attaches both the Unity `Light` and its
`HDAdditionalLightData`. Cube-like meshes get a `BoxSpot`; everything flat/text gets
a `RectangleArea`. The chosen shape dictates the intensity unit - **`BoxSpot` only
accepts Lux, `RectangleArea` accepts Lumen** - and the defaults mirror CS2's own
`LightCullingSystem`:

```csharp
var lightShape = textType == WESimulationTextType.WhiteCube
    ? HDLightTypeAndShape.BoxSpot
    : HDLightTypeAndShape.RectangleArea;
var hdLight = go.AddHDLight(lightShape);

hdLight.lightlayersMask = LightLayerEnum.Everything;
hdLight.includeForRayTracing = false;
hdLight.affectDiffuse = true;
hdLight.affectSpecular = true;
hdLight.applyRangeAttenuation = true;

// BoxSpot only accepts Lux; RectangleArea accepts Lumen.
hdLight.lightUnit = lightShape == HDLightTypeAndShape.BoxSpot ? LightUnit.Lux : LightUnit.Lumen;
```
Source: `../../../vice-and-order-research/mods/dossiers/write-everywhere/repo/BelzontWE/Systems/WEEmissiveLightSystem.cs#L162-L175` (@13c70eb04e6bed152257c516a982455148f591a5)

### 4. Drive transform, color, and intensity each frame

Position and rotation come straight out of the pre-culled transform matrix (the same
matrix the overlay mesh uses). Color tracks the emissive tint, and intensity is the
effective emissive value scaled by the tuning constant:

```csharp
var col = baseMatrix.GetColumn(3);
entry.go.transform.SetPositionAndRotation(
    new Vector3(col.x, col.y, col.z), baseMatrix.rotation);

var emissiveColor = materialData.EmissiveColorEffective;
entry.light.color = new Color(emissiveColor.r, emissiveColor.g, emissiveColor.b, 1f);

// Sync intensity (Lumen = total luminous flux, most intuitive for a glowing surface).
float lumens = materialData.EmissiveIntensityEffective * LumensPerUnit;
entry.hdLight.intensity = lumens;
```
Source: `../../../vice-and-order-research/mods/dossiers/write-everywhere/repo/BelzontWE/Systems/WEEmissiveLightSystem.cs#L99-L124` (@13c70eb04e6bed152257c516a982455148f591a5)

### 5. Recreate on shape change, deactivate off-screen, destroy on death

Because a `BoxSpot` and a `RectangleArea` are different HDRP lights, a change of mesh
type must rebuild the entry. Lights not seen this frame are deactivated; lights whose
entity no longer exists are destroyed and dropped from the map:

```csharp
// Deactivate lights whose entities are no longer in the visible set.
foreach (var kv in m_lights)
    if (!m_seenThisFrame.Contains(kv.Key))
        kv.Value.go.SetActive(false);

// Destroy light GameObjects for entities that no longer exist in the world.
m_orphaned.Clear();
foreach (var kv in m_lights)
    if (!EntityManager.Exists(kv.Key))
    {
        GameObject.Destroy(kv.Value.go);
        m_orphaned.Add(kv.Key);
    }
foreach (var entity in m_orphaned)
    m_lights.Remove(entity);
```
Source: `../../../vice-and-order-research/mods/dossiers/write-everywhere/repo/BelzontWE/Systems/WEEmissiveLightSystem.cs#L135-L153` (@13c70eb04e6bed152257c516a982455148f591a5)

The rebuild-on-type-change guard (`entry.textType != mesh.TextType` -> `Destroy` +
`CreateLightEntry`) is at
`../../../vice-and-order-research/mods/dossiers/write-everywhere/repo/BelzontWE/Systems/WEEmissiveLightSystem.cs#L82-L87`.

### 6. Destroy every light on system teardown

The dictionary owns unmanaged `GameObject`s, so `OnDestroy` must dispose them or they
leak across world reloads:

```csharp
protected override void OnDestroy()
{
    foreach (var entry in m_lights.Values)
        GameObject.Destroy(entry.go);
    m_lights.Clear();
    base.OnDestroy();
}
```
Source: `../../../vice-and-order-research/mods/dossiers/write-everywhere/repo/BelzontWE/Systems/WEEmissiveLightSystem.cs#L44-L50` (@13c70eb04e6bed152257c516a982455148f591a5)

## Pitfalls & gotchas

- **The DOTS light buffer is not yours to write.** CS2 feeds its dynamic lights
  through the internal `HDRPDotsInputs` DOTS buffer, which the source calls out as
  "inaccessible from mod code without reflection"
  (`../../../vice-and-order-research/mods/dossiers/write-everywhere/repo/BelzontWE/Systems/WEEmissiveLightSystem.cs#L13-L17`).
  That is the whole reason for the managed-`GameObject`-per-entity bridge - do not
  expect to just append to CS2's light list.

- **Intensity units differ by shape.** `BoxSpot` takes **Lux**, `RectangleArea` takes
  **Lumen** (`...#L175`). If you swap shapes without swapping `lightUnit`, the same
  numeric intensity produces wildly wrong brightness. Keep the unit tied to the shape.

- **Managed GameObjects need explicit lifecycle management.** Nothing garbage-collects
  these lights. You are responsible for three transitions: deactivate when the entity
  leaves the visible set (`...#L135-L140`), destroy when the entity ceases to exist
  (`...#L142-L153`), and destroy all of them in `OnDestroy` (`...#L44-L50`). Miss any
  and you leak lights or leave stale ones glowing.

- **Deactivate, do not destroy, for transient off-screen entities.** The source keeps
  the GameObject and toggles `SetActive(false/true)` (`...#L74-L75`, `...#L132`) so a
  briefly-culled entity does not pay a full recreate cost every frame. Only a genuine
  type change or entity death triggers `Destroy`.

- **A light shape change is a full rebuild.** Because `BoxSpot` and `RectangleArea`
  are distinct HDRP light types, mutating an existing entry's shape in place is not
  the pattern here; the code destroys and recreates the entry (`...#L82-L87`).

- **Run this off the hot path.** The system uses `AllowedPhase.EndFrame` (`...#L20`),
  reading an already-pre-culled draw list rather than querying every entity itself.
  Doing HDRP GameObject work per visible entity per frame is not free; the visible-set
  gate is what keeps it bounded.

- **`Needs Verification (in-game)`:** the actual on-screen brightness of the
  `LumensPerUnit = 25f` constant, and whether `BoxSpot` vs `RectangleArea` looks
  correct for a given mesh, are runtime-visual judgements not provable from source -
  the code even labels the constant "Tune this constant if the emitted light appears
  too dim or too bright in-game" (`...#L21-L23`).

## Variations

- **Draw-only overlay (no light).** If you just want geometry to appear, drop the
  light entirely and use a plain render-pipeline overlay:
  [render-pipeline-overlay](render-pipeline-overlay.md). This recipe is the "and also
  illuminate the scene" superset of that one.

- **Range/size heuristics per shape.** The source derives `BoxSpot` range from the
  mesh Z extent but `RectangleArea` range from a brightness heuristic
  (`sqrt(width^2+height^2) / EmissiveExposureWeight`), and sets `shapeWidth`/
  `shapeHeight` from world-space mesh bounds (`...#L104-L130`). If your entities are
  uniform you can hard-code these instead of computing them per frame.

- **Interpolated vs raw transform.** When the geometry entity carries an
  `InterpolatedTransform`, the source composes the interpolated matrix with the base
  matrix for smoother light motion (`...#L89-L95`); a static overlay can skip that and
  use the raw matrix directly.

## See also
- Related recipes: [render-pipeline-overlay](render-pipeline-overlay.md) (family I -
  the draw-only overlay this technique extends).
- Explanation: [React UI in CS2](../../explanation/react-ui.md).
- Operations: [memory & performance](../operations/memory-and-performance.md)
  (managed-object lifecycle and per-frame cost).
- Case study: [write-everywhere ecosystem](../../case-studies/write-everywhere-ecosystem.md).

## Sources
- Canonical mods (dossier + repo):
  - `write-everywhere` @13c70eb04e6bed152257c516a982455148f591a5 - `repo/BelzontWE/Systems/WEEmissiveLightSystem.cs`
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding
