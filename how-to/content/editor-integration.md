---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Import Meshes and Build Prefabs in the CS2 Editor"
Summary: The editor workflow for a content mod - launch the CS2 asset editor, import meshes/materials/textures, build prefabs from vanilla templates, validate lighting and metadata, and export a mod asset.
diataxis: how-to
source_version: "n/a - editor how-to, wiki-sourced (Asset_Creation_Guide verified 1.5.2f1; Patch_1.6.X)"
last_reverified: "2026-07-04"
status: needs-verification
Created: 2026-07-03
Updated: 2026-07-03
Owners:
  - codex
References:
  - Label: PBR Asset Workflow (produces the mesh + textures you import here)
    Path: ./pbr-asset-workflow.md
  - Label: Support and Publishing (ship the exported asset)
    Path: ./support-and-publishing.md
  - Label: Build, Test, and Publish (publish walkthrough)
    Path: ../../tutorials/build-and-publish.md
  - Label: Technique Index
    Path: ../../technique-index.md
---

# Import Meshes and Build Prefabs in the CS2 Editor

This how-to takes an authored asset - a mesh with LODs and packed PBR textures, produced
in [PBR Asset Workflow](./pbr-asset-workflow.md) - into the Cities: Skylines II editor,
turns it into a placeable prefab, validates it, and exports it as a mod asset ready to
publish.

Assume you already have a valid FBX and its texture set. This page is about the editor
side of the pipeline, not modeling or texturing.

---

## The official Asset Mods editor (beta)

CS2 ships an official in-game **Editor** mode for creating and importing custom assets -
buildings and props - and publishing them to **Paradox Mods**. That is the pipeline the
stages below use. It is still **beta**.

- **When it shipped (timeline correction).** The Asset Mods editor beta was introduced in
  **patch 1.5.2f1 (2025-12-04)**, *not* in 1.6.0. Paradox's
  [Asset Mods Patch](https://www.paradoxinteractive.com/games/cities-skylines-ii/news/asset-mods-patch-notes)
  news (1.5.2f1, 2025-12-04) states the patch "contains a beta update to the Editor that
  supports asset mods" and "Added the ability to create and share assets with the Editor."
  The [Asset Creation Guide](https://cs2.paradoxwikis.com/Asset_Creation_Guide) wiki page
  is correspondingly stamped "Verified ... for version 1.5.2f1."
- **What 1.6.0 changed.** The 1.6.0 "Summer Solstice" release (2026-06-22) only
  **refined** the pre-existing, still-beta editor (editor gizmo/widget behavior, materials
  tooltips, map saving with custom climate/terrain, Paradox Mods fixes). It did **not**
  introduce Asset Mods. See [Patch 1.6.X](https://cs2.paradoxwikis.com/Patch_1.6.X).
- **The pipeline.** Open the **Editor from the Main Menu** - the same entry you would use
  to create a custom map - then create or import **buildings and props** (**FBX 2018**
  meshes, **PNG** textures, per the Asset Creation Guide) and **publish to Paradox Mods**.
- **What it supersedes.** Before 1.5.2f1, importing custom assets relied on **community
  mods** such as the
  [Extra assets importer](https://cs2.paradoxwikis.com/Extra_assets_importer). The
  official in-editor asset pipeline supersedes that community route, though it remains beta.

---

> **Version note.** The asset editor's menu names, panel labels, folder layout, prefab
> template names, and metadata field names change between releases. Every editor-specific
> label below is marked **Needs Verification** - confirm the exact wording against the
> current editor and the official modding wiki before scripting or documenting a fixed
> path.

---

## Stage 1: Launch the editor in developer mode

1. Open the CS2 **Editor** from the **Main Menu** - the same entry you would use to create
   a custom map (per Paradox's "Adding Custom Assets" news), separate from the normal
   "Play" flow - with developer mode enabled so importer and validation tooling is
   available.
2. Confirm the editor loads and that the asset/import tooling is present before you begin.

> **Needs Verification:** the Main-Menu Editor entry is officially documented, but the
> exact developer-mode flag or setting that exposes the importer, and whether any extra
> command-line switch is required, are not - confirm against the current client and the
> [Asset Creation Guide](https://cs2.paradoxwikis.com/Asset_Creation_Guide) wiki.

## Stage 2: Import meshes, materials, and textures

1. Place your import files into the editor project's asset folder.
2. Group the files per asset - keep each asset's mesh, LODs, and textures together so the
   importer can resolve materials by naming, and so the project stays navigable as it
   grows.
3. Import the mesh; confirm the LOD meshes are recognized and associated with the base
   mesh.
4. Import the textures and confirm each packed map is read into the channel the shader
   expects.

> **Needs Verification:** the exact asset-folder path/name the editor watches (an
> `Assets/`-style folder is typical), how LODs must be named or nested to be picked up
> automatically, and the supported import formats. Verify against the current importer.

## Stage 3: Create a prefab from a vanilla template

1. Start from the vanilla prefab template that matches your asset type - for example a
   building, prop, decal, or surface template - so the prefab inherits the correct
   components and defaults.
2. Assign your imported materials to the prefab's mesh/material slots.
3. Set up any type-specific structure (for a building: sub-meshes, attachment points; for
   a surface/decal: projection and tiling).

> **Needs Verification:** the exact set and names of vanilla prefab templates, the menu
> path to create a prefab from one, and how material slots are bound in the current
> editor. Verify against the current editor UI.

## Stage 4: Validate

1. **Check lighting and normals.** Preview the asset under the editor's lighting/preview
   scenes and confirm surfaces are lit correctly and normals point outward (no inverted
   or "inside-out" faces, no flipped normal-map channel). If it looks wrong here, fix the
   texture/mesh before proceeding - see
   [PBR Asset Workflow](./pbr-asset-workflow.md).
2. **Check scale and pivot** against a known-size vanilla reference in the same scene.
3. **Check the LOD swaps** by moving the camera through the LOD transition distances.

> **Needs Verification:** the exact names of the editor's preview/validation scenes and
> any built-in asset validator or checklist the editor provides. Verify against the
> current editor.

## Stage 5: Populate metadata

Before export, fill in the gameplay metadata the asset type requires so it behaves
correctly once placed - for example construction cost, upkeep/maintenance, and (for
service buildings) service radius or coverage. Missing or default metadata is a common
reason an asset imports fine but "does nothing" in game.

> **Needs Verification:** the exact metadata fields per asset type and their editor
> labels. Verify against the current editor's asset property panel.

## Stage 6: Export the mod asset

1. Export the validated prefab as a mod asset through the editor's export/save-as-mod
   flow.
2. Confirm the export bundles the mesh, LODs, textures, and metadata together.
3. Load the exported asset in a normal game session and place it to confirm it behaves as
   authored before you publish.

> **Needs Verification:** the exact export menu/command name and the on-disk output layout
> the editor produces. Verify against the current editor, then publish through the
> toolchain (see below).

---

## Editor validation checklist

- [ ] Mesh and LODs imported and associated.
- [ ] Packed textures read into the correct shader channels.
- [ ] Prefab built from the correct vanilla template.
- [ ] Materials assigned to all slots.
- [ ] Lighting and normals validated in a preview scene.
- [ ] Scale and pivot checked against a vanilla reference.
- [ ] LOD transitions previewed and clean.
- [ ] Gameplay metadata (cost, upkeep, radius, ...) populated.
- [ ] Exported and placed in a live session before publishing.

---

## Where to go next

- **Fix a mesh/texture problem found during validation:**
  [PBR Asset Workflow](./pbr-asset-workflow.md).
- **Author a full map rather than a single asset:**
  [Map Authoring](./map-authoring.md).
- **Publish the exported asset:** [Support and Publishing](./support-and-publishing.md)
  and [Build, Test, and Publish](../../tutorials/build-and-publish.md).
- **Pick a related technique:** [Technique Index](../../technique-index.md).
