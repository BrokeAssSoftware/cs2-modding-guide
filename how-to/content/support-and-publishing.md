---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Publish and Maintain Content and Asset Packs"
Summary: How to package, publish, and maintain CS2 content - surface/decal packs, asset collections, and policy/data packs - on PDX Mods, including marketplace readiness and ongoing support.
diataxis: how-to
source_version: "n/a - not source-verified (PDX Mods publishing workflow, not run this pass)"
last_reverified: "2026-07-04"
status: needs-verification
Created: 2026-07-03
Updated: 2026-07-03
Owners:
  - codex
References:
  - Label: Build, Test, and Publish (full PDX Mods publish walkthrough)
    Path: ../../tutorials/build-and-publish.md
  - Label: PBR Asset Workflow (author the assets)
    Path: ./pbr-asset-workflow.md
  - Label: Editor Integration (export assets from the editor)
    Path: ./editor-integration.md
  - Label: Map Authoring (author maps)
    Path: ./map-authoring.md
  - Label: Technique Index
    Path: ../../technique-index.md
---

# Publish and Maintain Content and Asset Packs

This how-to covers shipping CS2 *content* - surface and decal packs, asset collections,
and policy/data packs - to **PDX Mods**, and keeping it healthy afterward. For the
mechanics of the publish step itself (Release build, `PublishConfiguration.xml`, and the
IDE publish commands), follow **[Build, Test, and Publish](../../tutorials/build-and-publish.md)**;
this page is the content-specific layer on top of it.

Assume you already have finished assets (see [PBR Asset Workflow](./pbr-asset-workflow.md)
and [Editor Integration](./editor-integration.md)) or a finished map
([Map Authoring](./map-authoring.md)).

> **Version note.** Where an asset pack is staged on disk, how policies are defined, and
> the exact publish commands are toolchain- and version-specific. Items marked
> **Needs Verification** should be confirmed against the current toolchain and the
> official modding wiki.

---

## Package surface and decal packs

Distribute texture-based content (surfaces, decals) as self-contained packs so players can
mix and match.

- **Group content into themed packs** (for example beaches, road markings, graffiti) and
  distribute a pack separately from any loader/importer mod, so players install only what
  they want.
- **Stage packs under a predictable folder** and document the expected resolutions and the
  original authors alongside each pack. A convention such as
  `ModsData/<ModName>/AssetPacks/<PackName>` keeps packs discoverable by a loader.
  - **Needs Verification:** the exact `ModsData`/user-content path the current game and
    toolchain use for side-loaded packs.
- **Ship metadata with every pack** - a README, a screenshot/thumbnail, and creator
  credit - so players know what they installed and who made it.
- **Register content at load.** If a loader mod owns the packs, iterate the staged packs
  during load, validate each pack's manifest, and register its surfaces/decals with the
  asset database. Guard each pack in a try/catch so one malformed pack logs a helpful error
  and the rest still load.
- **Offer an in-game browser** where useful: categorized tabs, texture previews, and quick
  placement (transform gizmos, snap toggles). Cache thumbnails so browsing stays
  responsive.

## Author policy and data packs

For data-only content (policies, presets, tuning), keep the source declarative and
validated.

- **Base definitions on the game's own catalogue.** Model YAML/JSON policy definitions on
  the official policy catalogue, with unlock milestones and expected effects, so they slot
  into vanilla systems rather than fighting them.
  - **Needs Verification:** the exact vanilla policy catalogue/identifiers to key against
    in the current version.
- **Keep manifests together and versioned** in your mod repo (a single, documented
  manifests folder) so the data set is easy to review and diff.
- **Bump the dependency version whenever the data changes**, even for a data-only update,
  so dependent mods know to refresh against the new data.
- **Add validation.** A CI or command-line linter that checks manifests for missing fields
  and bad references catches data errors before players do.

## Get marketplace-ready

Before you publish, prepare the listing so the content presents well and installs cleanly.

1. **Prepare media.** High-resolution thumbnails (16:9) and screenshots/video showing the
   content in use.
2. **Write release notes.** Cover new content, dependency changes, and compatibility notes
   so players can decide whether to install/update.
3. **Declare dependencies.** In `PublishConfiguration.xml`, list every hard dependency by
   **both name and mod ID** (a shared icon library, a loader/importer your pack requires).
   A missing dependency loads for you and breaks for everyone else. See
   [Build, Test, and Publish](../../tutorials/build-and-publish.md).
4. **Publish**, then **verify the live listing.** After publishing, confirm the PDX Mods
   page lists the required dependencies and that the uploaded archive actually contains the
   latest assets - a publish that silently omits content is a common regression.

## Communicate risk to players

Content packs are more exposed to editor/game changes than code mods, so set expectations.

- **Warn that unofficial packs may need republishing** when the official editor or game
  updates, and that locally installed content may not receive updates automatically.
- **Tell players to back up saves** before experimenting with new packs, and to remove
  experimental packs before reporting a bug so reports are reproducible.
- **Point players to support channels** (the relevant modding Discord/forum) in the pack
  README.

## Maintain after launch

- **Keep regression saves** that feature your published content so you can reproduce
  reported issues quickly across game updates.
- **Document performance costs.** Record triangle counts, texture-memory footprint, and
  recommended LOD distances in the pack README so players can judge the load.
- **Credit contributors** - translators and asset creators - in changelogs, and keep
  translations current on whatever localization platform you use.
- **Re-verify after each CS2 release.** Content is patch-sensitive; re-test packs and maps
  against new game versions and republish if the editor format changed.

---

## Publish checklist

- [ ] Packs staged in the expected folder with documented resolutions and credits.
- [ ] Every pack ships a README, thumbnail, and creator credit.
- [ ] Data/policy manifests validated by a linter; dependency version bumped.
- [ ] 16:9 thumbnails and demo media prepared.
- [ ] Release notes written.
- [ ] Every hard dependency declared by name and mod ID in `PublishConfiguration.xml`.
- [ ] Published, then verified the live listing and archive contents.
- [ ] Player-facing risk/back-up guidance in the pack README.
- [ ] Regression saves and performance notes kept for maintenance.

---

## Where to go next

- **Full publish mechanics (Release build, `PublishConfiguration.xml`, IDE commands):**
  [Build, Test, and Publish](../../tutorials/build-and-publish.md).
- **Author the content you are shipping:**
  [PBR Asset Workflow](./pbr-asset-workflow.md),
  [Editor Integration](./editor-integration.md),
  [Map Authoring](./map-authoring.md).
- **Pick a related technique:** [Technique Index](../../technique-index.md).
