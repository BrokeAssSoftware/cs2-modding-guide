# Asset and Content

Vice & Order ships custom models, maps, data packs, and detailing content. This guide explains how to build assets with the Cities: Skylines II toolchain, manage unofficial packs responsibly, and prepare releases for Paradox Mods.

## Authoring PBR Assets
1. **Modeling basics**
   - Model at 1:1 scale and keep pivots aligned with placement expectations (0,0,0 at ground contact).
   - Maintain consistent texel density; aim for 256 px per metre for close-up assets and higher for hero pieces.
2. **Export settings**
   - Export mesh files as FBX 2018 with Y-up, centimetres, and tangents.
   - Separate material slots by suffix (`_Win`, `_Gls`, `_Gra`) to follow vanilla conventions.
3. **Texture workflow**
   - Use the official Substance templates or a custom pipeline that outputs the CS2 channel packing:  
     - `_BaseColor` – RGB albedo  
     - `_MaskMap` – RGBA packed mask  
     - `_ControlMask` – optional atlas control mask  
     - `_Normal` – OpenGL normal map  
     - `_Emissive` – emissive intensity  
   - Stick to PNG or TGA files with power-of-two resolutions (512, 1024, 2048). Atlas only when multiple assets share a single material.
4. **LOD strategy**
   - Author LOD meshes with 60 percent fewer triangles and simplified materials.
   - Preview LOD swaps inside the editor and profile using the in-game render stats window.

## Bringing Assets into the Unity Editor
1. Launch the CS2 editor from the game launcher with developer mode enabled.
2. Import meshes, materials, and textures into the project’s `Assets/` folder, keeping per-asset subfolders.
3. Create prefabs using the vanilla templates (building, prop, decal, surface) and assign exported materials.
4. Validate lighting and normal orientation using the editor’s preview scenes.
5. Populate metadata (cost, maintenance, service radius) before exporting.

## Map and Scenario Authoring
1. **Terrain setup**
   - Lock elevation ranges early; altering them later disrupts water tables and spline meshes.
   - Sculpt river beds before importing road or rail splines.
2. **Resource painting**
   - Paint resources with broad, feathered brushes to avoid sharp simulation transitions.
3. **Climate and weather**
   - Adjust climate curves (temperature, precipitation, aurora) to fit the story scenario and test with accelerated time.
4. **Checklist before export**
   - Set start tile, inbound/outbound connections, water availability, and localized display names.
   - Run the simulation preview for at least five in-game days to ensure services function.

## Managing Unofficial Surface and Decal Packs
- Stage texture packs under `ModsData/<Module>/AssetPacks/<PackName>` and document expected resolutions and authors.
- During `OnLoad`, iterate packs, validate manifest files, and register surfaces/decals with the asset database.
- Provide an in-game browser that lists categories, previews each texture, and allows quick placement with accompanying tools (transform gizmo, snap toggles).
- Communicate risks clearly: unofficial packs may require republishing when the official editor updates; encourage players to back up saves before heavy experimentation.

## Policy and Data Packs
- Base YAML or JSON policy definitions on the official `Policies` catalogue. Include unlock milestones, prerequisites, and effect summaries.
- Store module manifests under `docs/modules/` and keep raw research artefacts in `docs/research/`.
- When data-only updates roll out, bump dependency versions so downstream modules know to pull fresh data.
- Plan validation scripts (CI or command-line tools) to lint manifests and catch missing fields.

## Marketplace Readiness
1. Prepare high-resolution thumbnails (16:9) and animated GIFs or short videos showing the asset in action.
2. Write release notes that include new content, dependency changes, and compatibility notes.
3. Run through the in-game publishing checklist: dependency declarations, tags, and authentication.
4. After publishing, verify the Paradox Mods page lists required dependencies (ExtraLib, UIL, etc.) and that downloads include the latest assets.

## Support and Maintenance
- Keep a regression library of maps or saves that feature new assets so QA can reproduce issues quickly.
- Document known issues, performance costs (triangle counts, texture memory), and recommended LOD distances in the module README.
- Encourage translators and asset creators to contribute via shared platforms (Crowdin, Discord) and credit them in changelogs.

Following these steps ensures Vice & Order asset work remains high quality, performant, and easy for players to install and maintain.
