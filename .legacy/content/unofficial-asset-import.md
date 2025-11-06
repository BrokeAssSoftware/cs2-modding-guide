# Unofficial Surface And Decal Import

Reference mod: `ExtraAssetsImporter`.

Until the official asset editor ships, you can still provide high-quality surfaces and decals by loading texture packs at runtime. Follow these guidelines to keep the workflow stable and future-proof.

## 1. Package Texture Packs

- Group textures into themed packs (beaches, road markings, graffiti, etc.) and distribute them separately from the importer so players can mix and match.
- Provide multiple resolutions (512px, 1k, 4k) and document the file sizes so players understand the trade-offs.
- Ship metadata (README, screenshot) alongside each pack and credit the creators.

## 2. Load Assets At Startup

- Scan a well-known directory (for example `ModsData/<Mod>/AssetPacks`) for pack definitions.
- For each pack, register surfaces and decals with the game's asset database so they appear in your custom menus.
- Guard the process with try/catch blocks - if a pack is malformed, log a helpful error and continue loading the remaining packs.

## 3. Offer An In-Game Browser

- Present categorized tabs (Ground, Grass, Tiles, Road Markings, etc.) so users can find assets quickly.
- Display pack metadata and thumbnails so players know which creator produced each item.
- Keep the UI responsive by caching thumbnails and deferring heavy loads until a category is opened.

## 4. Coordinate With Detailing Tools

- If you ship a transform or placement tool (e.g., ExtraDetailingTools), integrate the asset browser directly so the same UI can spawn custom surfaces and decals.
- Provide snap options (ground, wall) to place decals correctly.

## 5. Communicate Risks

- Make it clear that unofficial assets may need republishing when the official editor releases.
- Encourage players to keep backups of their saves and to remove experimental packs before reporting bugs to Colossal.
- Use a localisation platform (e.g., Crowdin) to keep warnings and feature descriptions translated.


