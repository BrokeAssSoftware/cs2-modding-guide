# Write Everywhere Modules

Reference mods: `CS2-WriteEverywhere`, `CS2-WEModuleTemplate`.

Write Everywhere (WE) lets creators place custom text, images, and meshes anywhere in the city. Modules extend the core mod by shipping additional content. Use this checklist when building Vice & Order overlays or publishing community packs.

## 1. Understand The Core Capabilities

- **Text nodes**: render dynamic strings with custom fonts, colours, and shader options.
- **Image planes**: project atlas images onto meshes, including custom OBJ meshes exported from Blender.
- **Animation**: combine atlases and shader properties to create animated signage or scrolling displays.
- **Layout instancing**: generate arrays of sub-layouts with configurable spacing, alignment, and index variables (`$idx`).

## 2. Structure A Module Project

- Start from the template repository (`CS2-WEModuleTemplate`). Rename the project and solution so the resulting DLL and `mod.json` use your module ID.
- Populate the provided `Resources/` subfolders:
  - `Layouts/` for default layouts and city-importable layouts.
  - `Atlases/` for image atlases (PNG base colour plus optional normal/mask maps).
  - `Fonts/` for TTF font files.
- Update the `.csproj` metadata (display name, description, social links, Paradox mod ID) and provide thumbnails/screenshots under `Properties/` and `Screenshots/`.

## 3. Register Content Programmatically

- During `OnLoad`, call the WE registration API to append layouts, fonts, and atlases shipped with your module.
- Provide metadata so the WE UI can list which modules supplied each asset. This helps players install fonts/layouts from the in-game picker.
- Remember that fonts must always be installed manually in the city via the "City Fonts" tab; modules simply make them available.

## 4. Ship Custom Meshes

- Save custom meshes as Wavefront `.obj` files under a folder such as `objMeshes`.
- Ensure each mesh includes vertices, normals, UVs, and triangles. Only one mesh per file.
- Reference the mesh in your layout or atlas definitions so WE can unwrap the image correctly.
- Mesh support is currently limited to image nodes; plan for follow-up updates when WE exposes more mesh hooks.

## 5. Provide Reset And Maintenance Tools

- Include buttons to reload atlases/meshes during gameplay so creators can iterate without restarting the game.
- Document any prerequisites (e.g., Scene Explorer for debugging, dev-mode flags) in the module README.

## 6. Publish Responsibly

- Once the module works locally, publish via the in-game toolchain and record the mod ID in your `.csproj`.
- Bump the version string before each release and keep a changelog so players know when new layouts or fonts arrive.
