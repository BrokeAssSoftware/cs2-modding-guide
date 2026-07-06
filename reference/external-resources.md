---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Reference: External Resources"
Summary: Curated link-out index for Cities Skylines II modding - the official Paradox wiki pages, the Paradox Mods storefront, community canon (Cities2Modding, community guides, the modding Discord), and the key libraries and SDKs a code mod builds on (Colossal Modding SDK, Harmony, Unity Entities). A pointer list, not a duplicate - each entry links to the authoritative source.
diataxis: reference
source_version: "n/a - concept/process page, no pinned source"
last_reverified: "2026-07-04"
status: needs-verification
Created: 2026-07-03
Updated: 2026-07-03
Owners:
  - codex
References:
  - Label: "Reference: Unified Icon Library (shared library)"
    Path: ./shared-libraries/unified-icon-library.md
  - Label: "Reference: ExtraLib (shared library)"
    Path: ./shared-libraries/extralib.md
  - Label: "Explanation: Dependency Strategy"
    Path: ../explanation/dependency-strategy.md
  - Label: "How-to: I18n Everywhere localization backbone"
    Path: ../how-to/localization/i18n-integration.md
  - Label: Technique Index
    Path: ../technique-index.md
---

# Reference: External Resources

A curated pointer list to the authoritative Cities: Skylines II modding sources. This page is
deliberately **link-out only** - it does not duplicate the content it references. When a topic
is covered on the official wiki, link the wiki page rather than copying it here; when the
handbook has a source-verified page on the same topic, that page links out to the matching entry
below. Keep this list curated: remove dead links, and add SDK changelogs and new community
guides as they appear.

## Official wiki (authoritative)

The Paradox community wiki is the primary reference for the modding toolchain, UI/code
pipeline, and asset workflow. Prefer it over any duplicate.

- [Modding overview](https://cs2.paradoxwikis.com/Modding) - entry point and policy reminders.
- [Modding Toolchain](https://cs2.paradoxwikis.com/Modding_Toolchain) - SDK/tooling install, Unity/Burst setup, publishing walkthrough.
- [Creating UI and Code Mods](https://cs2.paradoxwikis.com/Creating_UI_And_Code_Mods) - the combined C# + React pipeline.
- [UI Modding](https://cs2.paradoxwikis.com/UI_Modding) - Gameface modules, `cs2/*` packages, `coui://` hosts, styling.
- [Options UI](https://cs2.paradoxwikis.com/Options_UI) - the settings-attribute catalogue.
- [Mod Key Binding](https://cs2.paradoxwikis.com/Mod_Key_Binding) - rebindable-action pipeline and localization.
- [Asset Creation Guide](https://cs2.paradoxwikis.com/Asset_Creation_Guide) - mesh/texture specs and PBR workflow.
- [Policies](https://cs2.paradoxwikis.com/Policies) - vanilla policy definitions and milestone unlocks.
- [Developer Mode](https://cs2.paradoxwikis.com/Developer_mode) - console flags, diagnostics, risk guidance.
- [Editor](https://cs2.paradoxwikis.com/Editor) and [Map Creation](https://cs2.paradoxwikis.com/Map_Creation) - environment authoring and publishing checklists.
- [Community-made guides](https://cs2.paradoxwikis.com/Community-made_guides) - the aggregated index of community best-practice guides, ECS notes, and localization tips (see Community canon below).

## Marketplace

- [Paradox Mods - Cities: Skylines II](https://mods.paradoxplaza.com/games/cities_skylines_2) - the official storefront; a benchmark for packaging, metadata, dependency declaration, and update cadence.

## Community canon

- [Cities2Modding (optimus-code)](https://github.com/optimus-code/Cities2Modding) - long-running community starter/toolkit repo: project templates, examples, and setup notes.
- **ps1ke's community modding guide** - a widely-cited community walkthrough; find the current link via the wiki's [Community-made guides](https://cs2.paradoxwikis.com/Community-made_guides) index (the canonical, maintained pointer).
- **Cities: Skylines Modding Discord** - the primary real-time channel for modding help, icon/library requests, and load-order advice. Invite codes rotate; get the current invite from the [Community-made guides](https://cs2.paradoxwikis.com/Community-made_guides) page or a maintained library's storefront/README rather than a hard-coded link here.

## Key libraries and SDKs

The libraries a CS2 code mod builds on. For depending on a *shared mod* library (icons,
localization), see the shared-library pages
([UIL](./shared-libraries/unified-icon-library.md), [ExtraLib](./shared-libraries/extralib.md))
and [Dependency Strategy](../explanation/dependency-strategy.md).

- **Colossal Modding SDK** - the official mod toolchain (referenced by the `CSII_TOOLPATH` environment variable and the `Mod.props`/`Mod.targets` build imports); distributed with the game's modding tools. Install and setup are documented on the wiki [Modding Toolchain](https://cs2.paradoxwikis.com/Modding_Toolchain).
- [Harmony (pardeike/Harmony)](https://github.com/pardeike/Harmony) - the runtime patching library CS2 mods use for prefix/postfix/transpiler patches; see its [wiki/docs](https://harmony.pardeike.net/).
- [Unity Entities (DOTS/ECS)](https://docs.unity3d.com/Packages/com.unity.entities@latest) - the Entity Component System the CS2 simulation is built on (`EntityManager`, `SystemBase`, `IJobChunk`, Burst). The [Burst](https://docs.unity3d.com/Packages/com.unity.burst@latest) and [Collections](https://docs.unity3d.com/Packages/com.unity.collections@latest) packages accompany it.

## Shared-library dependencies (handbook pages)

Handbook reference pages for the two most commonly depended-on shared-library mods, each
source-verified against a pinned commit:

- [Unified Icon Library (UIL, id 74417)](./shared-libraries/unified-icon-library.md) - shared SVG icon catalogue injected under `coui://uil/...`.
- [ExtraLib (id 75724)](./shared-libraries/extralib.md) - shared services library (icon host, notification UI, localization loader, entity-edit queue, ExtraPanels).

## Reference mod repositories

Open-source mods worth reading for specific techniques. These are *reading references* - study
the technique, do not assume any of them is a dependency. Where the handbook has a
source-verified page, follow the cross-link from the [Technique Index](../technique-index.md).

- [Unified Icon Library](https://github.com/algernon-A/UnifiedIconLibrary) - shared SVG icon host (`coui://uil/...`).
- [ExtraLib](https://github.com/AlphaGaming7780/ExtraLib) - shared dependency library for the Extra toolchain.
- [ExtraAssetsImporter](https://github.com/AlphaGaming7780/ExtraAssetsImporter) - imports unofficial surfaces/decals; a real ExtraLib consumer.
- [ExtraDetailingTools](https://github.com/AlphaGaming7780/ExtraDetailingTools) - transform gizmo, net-lane kit, surfaces, decals, snap-to-surface.
- [I18n Everywhere](https://github.com/baka-gourd/I18NEverywhere) - localization pipeline with embedded locales and language packs.
- [Anarchy](https://github.com/yenyang/Anarchy) - disables placement checks; elevation lock; net/prop overrides; a strong settings/keybinding reference.
- [Better Bulldozer](https://github.com/yenyang/BetterBulldozer) - bulldozer filters for hidden markers, sub-elements, moving objects.
- [Write Everywhere](https://github.com/klyte45/CS2-WriteEverywhere) - customizable text/image/mesh overlays with layout instancing (declares UIL as a hard dependency).
- [WE Module Template](https://github.com/klyte45/CS2-WEModuleTemplate) - starter project for distributing Write Everywhere atlases, layouts, and fonts.
- [Realistic Path Finding](https://github.com/ruzbeh0/RealisticPathFinding) - DOTS system replacement, settings UX, and a cited runtime mod-interop detection pattern.
- [Time2Work (Realistic Trips)](https://github.com/ruzbeh0/Time2Work) - multi-phase scheduling, UI replacement, data separation.
- [Achievement Fixer](https://github.com/River-Mochi/AchievementFixer) - lightweight systems, localization overrides, idle-by-default design.
- [Asset Packs Manager](https://github.com/CitiesSkylinesModding/CS2-AssetPacksManager) - playset-aware asset catalog with reports and localization helpers.

## See also

- [Dependency Strategy](../explanation/dependency-strategy.md) - how to declare, reference,
  validate, and degrade around any of the dependencies listed above.
- Shared libraries in depth: [Unified Icon Library](./shared-libraries/unified-icon-library.md)
  and [ExtraLib](./shared-libraries/extralib.md).
- [Technique Index](../technique-index.md) - which of these mods teaches which technique.
