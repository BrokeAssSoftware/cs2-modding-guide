# Support and Publishing

## Managing Unofficial Surface and Decal Packs
- Stage texture packs under `ModsData/<Module>/AssetPacks/<PackName>` and document expected resolutions and authors.
- During `OnLoad`, iterate packs, validate manifest files, and register surfaces/decals with the asset database.
- Provide an in-game browser that lists categories, previews textures, and allows quick placement with tools such as transform gizmos and snap toggles.
- Communicate risks clearly: unofficial packs may require republishing when the official editor updates; encourage players to back up saves before experimentation.

## Policy and Data Packs
- Base YAML or JSON policy definitions on the official `Policies` catalogue with unlock milestones and expected effects.
- Store module manifests under `docs/modules/` and keep research artefacts under `docs/research/`.
- Bump dependency versions alongside data-only updates so downstream modules know to refresh.
- Add validation scripts (CI or command-line) to lint manifests and catch missing fields.

## Marketplace Readiness
1. Prepare high-resolution thumbnails (16:9) and media showing assets in action.
2. Write release notes covering new content, dependency changes, and compatibility notes.
3. Run through the in-game publishing checklist: dependency declarations, tags, authentication.
4. After publishing, verify the Paradox Mods page lists required dependencies and that downloads include the latest assets.

## Support and Maintenance
- Maintain regression saves that feature new assets so QA can reproduce issues quickly.
- Document known issues, performance costs (triangle counts, texture memory), and recommended LOD distances in module READMEs.
- Credit translators and asset creators in changelogs and shared localisation platforms.
