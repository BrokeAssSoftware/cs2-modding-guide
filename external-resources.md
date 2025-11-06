# External Resources

Authoritative and community references that inform the Vice & Order modding stack. Mirror these links in `Agents.md` when major updates land.

## Official Wiki

- [Modding Overview](https://cs2.paradoxwikis.com/Modding) - high-level entry point and policy reminders.
- [Modding Toolchain](https://cs2.paradoxwikis.com/Modding_Toolchain) - dependency installation, Unity/Burst setup, publishing walkthrough.
- [Creating UI And Code Mods](https://cs2.paradoxwikis.com/Creating_UI_And_Code_Mods) - combined C# + React pipeline.
- [UI Modding](https://cs2.paradoxwikis.com/UI_Modding) - Gameface modules, `cs2/*` packages, styling notes.
- [Options UI](https://cs2.paradoxwikis.com/Options_UI) - attribute catalogue for settings classes.
- [Mod Key Binding](https://cs2.paradoxwikis.com/Mod_Key_Binding) - rebindable action pipeline and localization.
- [Asset Creation Guide](https://cs2.paradoxwikis.com/Asset_Creation_Guide) - mesh/texture specs and PBR workflow.
- [Policies Catalogue](https://cs2.paradoxwikis.com/Policies) - vanilla policy definitions and milestone unlocks.
- [Developer Mode](https://cs2.paradoxwikis.com/Developer_mode) - console flags, diagnostics, and risk guidance.
- [Editor](https://cs2.paradoxwikis.com/Editor) & [Map Creation](https://cs2.paradoxwikis.com/Map_Creation) - environment authoring and publishing checklists.
- [Community-made Guides](https://cs2.paradoxwikis.com/Community-made_guides) - consolidated best practices, ECS notes, localization tips.

## Marketplace Recon

- [Paradox Mods - Cities: Skylines II](https://mods.paradoxplaza.com/games/cities_skylines_2?orderBy=desc&sortBy=best&time=month) - benchmark packaging, metadata, and update cadence.

## Reference Mods

- [Realistic Path Finding](https://github.com/ruzbeh0/RealisticPathFinding) - DOTS system replacement patterns, settings UX, interop hooks.
- [Time2Work (Realistic Trips)](https://github.com/ruzbeh0/Time2Work) - multi-phase scheduling, UI replacement, data separation.
- [Achievement Fixer](https://github.com/River-Mochi/AchievementFixer) - lightweight systems, localization overrides, idle-by-default design.
- [Anarchy](https://github.com/yenyang/Anarchy) - disables placement error checks, adds elevation lock, and exposes net/prop overrides.
- [Better Bulldozer](https://github.com/yenyang/BetterBulldozer) - bulldozer filters for hidden markers, sub-elements, and moving objects.
- [ExtraDetailingTools](https://github.com/AlphaGaming7780/ExtraDetailingTools) - transform gizmo, net lane kit, surfaces, decals, snap-to-surface toggle.
- [ExtraAssetsImporter](https://github.com/AlphaGaming7780/ExtraAssetsImporter) - imports unofficial surfaces and decals with asset pack recommendations.
- [ExtraLib](https://github.com/AlphaGaming7780/ExtraLib) - shared dependency library for the Extra toolchain.
- [Unified Icon Library](https://github.com/algernon-A/UnifiedIconLibrary) - shared SVG icon bundles injected via `coui://uil/...`.
- [I18n Everywhere](https://github.com/baka-gourd/I18NEveryWhere) - localization pipeline with embedded locale support and language packs.
- [Write Everywhere](https://github.com/klyte45/CS2-WriteEverywhere) - customizable text, image, and mesh overlays with layout instancing.
- [WE Module Template](https://github.com/klyte45/CS2-WEModuleTemplate) - starter project for distributing Write Everywhere atlases, layouts, and fonts.
- [Asset Packs Manager](https://github.com/CitiesSkylinesModding/CS2-AssetPacksManager) - playset aware asset catalog with reports and localization helpers.

_Dependency stance_: Vice & Order ships with Unified Icon Library and I18n Everywhere, and builds overlays on top of Write Everywhere. ExtraLib remains an optional integration. Tooling techniques from Anarchy/Better Bulldozer/ExtraDetailingTools inform our internal implementations rather than direct dependencies.

## Local Snapshots

- `docs/research/wiki/` - archived wiki content for offline reference (`modding_toolchain_reference.md`, `ui_modding_reference.md`, etc.).
- `docs/research/mods/` - source excerpts and field notes from dissected mods (Realistic Path Finding, Time2Work, Achievement Fixer).

Keep this list curated: remove dead links, add patch notes, SDK changelogs, and new community guides as they appear.
