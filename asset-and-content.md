# Asset And Content

Vice & Order leans on custom models, maps, and data packs. Use these guidelines from the official asset pipeline and community notes.

## PBR Asset Workflow

- Model at 1:1 scale; keep texel density consistent and avoid deleting downward faces that contribute to GI.
- Export meshes as 2018 `.fbx` files with clean pivot alignment; split submeshes with suffixes like `_Win`, `_Gls`, `_Gra` to match vanilla conventions.
- Supply texture sets as PNGs with case-sensitive suffixes (`_BaseColor`, `_MaskMap`, `_ControlMask`, `_Normal`, `_Emissive`).
- Use Substance 3D Painter templates (`substance_3d_painter_setup.md`) to export the correct channel packing.
- Target square textures at least 512x512; atlas variants only when multiple assets share materials.

## Unity Editor Integration

- Install the CS2 editor via the game launcher and open it in developer mode to access debug panels.
- Workspace focus areas: Map (terrain, water, resources), Climate (lighting curves, weather patterns), Environment (props, vegetation, surfaces), Objects (placeable prefabs and gameplay entities).
- Use the publishing checklist: start tile, inbound/outbound road, water in the starting area, English map name, and external connections when applicable.
- For environment assets, profile draw calls and LODs inside the editor before exporting.

## Map Creation Tips

- Define elevation bounds early; adjusting later breaks water tables.
- Paint resources with smooth gradients; avoid hard edges that generate simulation artifacts.
- Verify climate curves (temperature, precipitation) align with scenario goals; extreme values affect citizen needs.
- Use the simulation preview to test spawn points and service coverage prior to shipping.

## Detailing & Placement Overrides

- Study Anarchy for techniques to relax placement validation, add relative elevation controls, and expose per-tool toggles without breaking vanilla systems.
- Better Bulldozer demonstrates filtered demolition flows (surfaces, invisible markers, sub-elements) and reset buttons that preserve save integrity.
- ExtraDetailingTools bundles a transform gizmo, snap-to-surface toggle, and curated menus (surfaces, decals, net lanes); replicate the pattern when exposing rich asset banks.
- ExtraAssetsImporter highlights how to stage unofficial surface and decal packs with clear risk messaging ahead of official editor support.
- Keep shared helpers (e.g., ExtraLib) in a dedicated dependency module so content packs and tooling stay lightweight.

## Data And Policy Packs

- Base new policy YAMLs on the vanilla catalogue (`Policies Catalogue` wiki snapshot); document unlock milestones and expected effects.
- Store module manifests under `docs/modules/` once the schema stabilizes; keep research artifacts under `docs/research/`.
- When shipping data-only updates, flag dependency versions clearly so downstream modules know when a content patch is required.

## Marketplace Readiness

- Review top-rated mods on [Paradox Mods](https://mods.paradoxplaza.com/games/cities_skylines_2?orderBy=desc&sortBy=best&time=month) for packaging baseline (icons, descriptions, dependency declarations).
- Include high-resolution thumbnails, changelog entries, and support links in every release.
- Bundle LOD screenshots or GIFs demonstrating the content in action to build player trust.
