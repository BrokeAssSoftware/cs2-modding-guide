---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Author a PBR Asset: Model, Export, Texture, LOD, Import"
Summary: The end-to-end asset-authoring workflow for a CS2 content mod - model to scale, export FBX, pack PBR textures into the CS2 channel layout, build LODs, and import the result into the editor.
diataxis: how-to
source_version: "n/a - not source-verified (DCC/asset workflow, not run this pass)"
last_reverified: "2026-07-04"
status: needs-verification
Created: 2026-07-03
Updated: 2026-07-03
Owners:
  - codex
References:
  - Label: Editor Integration (where imported meshes become prefabs)
    Path: ./editor-integration.md
  - Label: Support and Publishing (ship the finished asset)
    Path: ./support-and-publishing.md
  - Label: Build, Test, and Publish (publish walkthrough)
    Path: ../../tutorials/build-and-publish.md
  - Label: Technique Index
    Path: ../../technique-index.md
---

# Author a PBR Asset: Model, Export, Texture, LOD, Import

This how-to walks the full pipeline for a physically-based (PBR) content asset - a
building, prop, or vehicle - from a modeling package (Blender, 3ds Max, Maya) into
Cities: Skylines II. The goal is an asset that lands at the right scale, reads correctly
under the game's lighting, packs its textures into the layout the renderer expects, and
degrades cleanly at distance through LODs.

Assume you are comfortable in your modeling package and a texture tool (Substance
Painter/Designer or an equivalent). This page is goal-directed; it does not teach
modeling or PBR theory.

> **Version note.** Exact texel densities, FBX export options, channel-packing names, and
> LOD counts are editor- and version-specific and are marked **Needs Verification**
> below. Treat the numbers as a starting point and confirm them against the current CS2
> asset editor and the official modding wiki before you commit a pipeline to them.

---

## Stage 1: Model to scale

1. **Model at 1:1 real-world scale.** A four-story building should be four stories tall in
   world units. Assets that are modeled "by eye" fight the game's snapping, zoning, and
   camera framing.
2. **Place the pivot at ground contact.** Put the origin (0,0,0) at the point where the
   asset meets the ground, centered on the footprint. The game places and rotates the
   asset about this pivot, so a mis-placed origin makes the asset float or sink.
3. **Keep a consistent texel density.** Aim for a uniform pixels-per-meter budget across
   the asset so no surface looks noticeably softer than its neighbors, and reserve higher
   density only for hero surfaces the player sees up close.
   - **Needs Verification:** a target of ~256 px/m for close-up assets (higher for hero
     pieces) is a reasonable working figure but is not a published constant - confirm the
     density the current asset editor and vanilla assets actually use.
4. **Watch your triangle budget.** Model the high-detail mesh with the final in-game
   silhouette in mind; you will build reduced LODs from it in Stage 4.

## Stage 2: Export the mesh (FBX)

Export the mesh to FBX for import into the CS2 editor.

- Export as **FBX** with a **Y-up** axis, **centimeter** units, and **tangents** included
  (tangents are required for correct normal-map shading).
- Split materials into separate slots and name them with clear suffixes so the editor can
  map each slot to its material. Match vanilla naming conventions where they exist (for
  example glass, window, and ground/graffiti slots) so shared shaders resolve correctly.
- Triangulate on export (or before) so the in-game mesh matches what you previewed.
- Apply transforms/scale before exporting so the FBX carries no residual object-level
  scale.

> **Needs Verification:** the exact FBX version (e.g. FBX 2018), the required up-axis and
> unit settings, whether tangents must be exported vs. recomputed on import, and the exact
> vanilla material-slot suffix scheme. These are set by the current asset editor's
> importer - confirm against the official importer docs before standardizing your export
> preset.

## Stage 3: Pack PBR textures into the CS2 channel layout

CS2 uses a packed PBR layout rather than one texture per property. Author your maps in
Substance (using the official CS2 templates, if available) or an equivalent pipeline that
outputs the game's channel packing.

Typical maps and their roles:

| Map | Content | Notes |
|-----|---------|-------|
| Base color / albedo | RGB surface color, no baked lighting | Keep it lighting-neutral; the renderer lights it. |
| Normal | Surface detail as a tangent-space normal map | Author/export in the orientation the game expects (see below). |
| Mask (packed) | Several grayscale properties packed into one RGBA texture | Each channel carries a different property (e.g. metallic, smoothness, occlusion, emissive/other). |
| Control mask (optional) | Atlas/variation control | Only when the shader/material needs it. |
| Emissive | Emission color/intensity | Only for self-lit surfaces. |

Rules of thumb:

- **Use power-of-two resolutions** (512, 1024, 2048) and only go as large as the surface
  warrants. Reserve the largest maps for hero assets.
- **Atlas materials only when multiple assets genuinely share them** - atlasing a
  one-off asset just wastes texture memory.
- **Prefer lossless formats** (PNG or TGA) for authored source maps so you are not baking
  compression artifacts into normals and masks.

> **Needs Verification:** the exact map/suffix names (for example `_BaseColor`,
> `_Normal`, `_MaskMap`, `_ControlMask`, `_Emissive`), the precise property-to-channel
> assignment inside the packed mask (which of R/G/B/A is metallic vs. smoothness vs.
> occlusion), and the normal-map convention (OpenGL vs. DirectX / +Y vs. -Y green). These
> are dictated by the current CS2 shaders - verify against the official Substance
> templates or shader docs, because a wrong channel or a flipped green channel is the most
> common "why does my asset look wrong" failure.

## Stage 4: Build LODs

Level-of-detail meshes keep large scenes affordable by swapping in cheaper geometry at
distance.

1. **Author reduced LOD meshes** from the high-detail mesh, dropping triangle count
   substantially at each level and simplifying (or dropping) materials that do not read at
   distance.
2. **Keep the silhouette.** Remove interior and small-scale detail first; preserve the
   outline the player recognizes from far away.
3. **Preview the swaps in the editor** and confirm the transitions do not "pop"
   distractingly.
4. **Profile the result** with the in-game render statistics so you know the asset's real
   cost, not an estimate.

> **Needs Verification:** how many LOD levels the current asset editor expects, the target
> triangle reduction per level (a ~60% drop is a common starting figure), whether LOD
> distances are author-set or automatic, and the exact name/location of the in-game render
> statistics overlay. Confirm against the current editor before committing to an
> LOD budget.

## Stage 5: Import into the editor

Hand the finished mesh, LODs, and packed textures to the CS2 editor and build the prefab
there. That step (import folders, prefab templates, material assignment, validation, and
mod export) is covered in **[Editor Integration](./editor-integration.md)** - continue
there.

---

## Pre-import checklist

- [ ] Modeled at 1:1 scale, pivot at ground contact (0,0,0).
- [ ] Texel density consistent across the asset.
- [ ] FBX exported with the editor's required axis/units/tangents (**verify settings**).
- [ ] Material slots named to match vanilla conventions.
- [ ] Base color is lighting-neutral (no baked shadows/AO in albedo).
- [ ] Mask channels packed in the order the CS2 shader expects (**verify order**).
- [ ] Normal map exported in the game's expected orientation (**verify convention**).
- [ ] Textures are power-of-two and no larger than needed.
- [ ] LOD meshes authored and preview clean, silhouette preserved.
- [ ] Asset profiled with the render statistics overlay.

---

## Where to go next

- **Turn the mesh into a placeable prefab:** [Editor Integration](./editor-integration.md).
- **Author a whole map instead of a single asset:** [Map Authoring](./map-authoring.md).
- **Ship the finished content:** [Support and Publishing](./support-and-publishing.md)
  and [Build, Test, and Publish](../../tutorials/build-and-publish.md).
- **Pick a related technique:** [Technique Index](../../technique-index.md).
