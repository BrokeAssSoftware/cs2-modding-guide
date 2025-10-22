# Case Studies

Field notes from community mods illustrate patterns Vice & Order can adopt.

## Realistic Path Finding

- Replaces the resident AI stack with custom systems; disables `Game.Simulation.ResidentAISystem` and registers new schedulers via `UpdateAt`/`UpdateAfter`.
- Provides deep slider coverage grouped by transport mode; uses `[SettingsUIGroupOrder]` and localized dictionaries per group.
- Logs Harmony patches and detected companion mods (`Time2Work`) to simplify troubleshooting.
- Shares interop helpers so other mods can query real-time parameters (wait-time factors).
- Takeaway: follow this pattern when swapping whole system families while keeping options approachable.

## Achievement Fixer

- Adds a short-lived `GameSystemBase` that runs for ~300 frames after load to flip `achievementsEnabled` back on.
- Hooks localization changes to reapply option overrides and custom banner strings.
- Keeps settings and static state lightweight; the system idles when not needed.
- Takeaway: implement diagnostic or safeguard systems with tight execution windows and full localization coverage.

## Time2Work (Realistic Trips)

- Disables numerous vanilla simulation and UI systems, installing replacements across `GameSimulation`, `UIUpdate`, `EditorSimulation`, and `Deserialize`.
- Splits configuration across `ModsSettings` (user options) and `ModsData` (runtime state) for clarity.
- Registers multiple locale sources and exposes utility methods (time-scaling factor) to partner mods.
- Unpatches Harmony hooks on dispose to support hot reload.
- Takeaway: use this as the template for Vice & Order's multi-phase, full-stack modules that touch both simulation and UI.

## Anarchy

- Suppresses placement error checks (overlap, tight curve, shoreline, lot limits) on demand and surfaces a toggleable tool icon plus shortcut.
- Adds elevation lock and relative elevation controls for props, trees, and nets; supports per-tool auto toggles and optional unique-building duplication.
- Provides compatibility switches such as auto-disabling while brushing, auto refreshing culled props, and minimum clearance for networks.
- Takeaway: adopt this approach for authoring opt-in sandbox affordances with granular safety rails and per-tool behavior.

## Better Bulldozer

- Layers bulldozer filters to target invisible paths, surfaces, net lanes, and static markers without touching other assets.
- Introduces moving-object and sub-element removal modes (single item, exact match, category) with safeguards and reset flow.
- Automates developer marker visibility, optional cleanup (manicured grass, branding), and integrates with Enhanced Detailing assets.
- Takeaway: build diagnostic tools with scoped filters, undo-friendly reset buttons, and toggles that manage related dev UI options.

## Extra Detailing Suite

- ExtraDetailingTools adds a transform gizmo, snap-to-surface toggle, and an extra assets menu exposing vanilla surfaces, decals, and fence-style net lanes.
- ExtraAssetsImporter loads unofficial surface and decal packs prior to the official asset pipeline, bundling best-practice recommendations and risk messaging.
- ExtraLib centralizes shared systems, UI, and translation support for the Extra toolchain.
- Takeaway: when shipping a family of detailing mods, isolate shared libraries, pair UX affordances (transform, snapping) with curated content, and document editor parity.

## Localization & UI Infrastructure

- Unified Icon Library preloads SVG icon sets by style (`coui://uil/<Style>/<Icon>.svg`) so dependent mods can reference assets without bundling copies; styles include standard, dark, and colored themes with layer recolor support.
- I18n Everywhere streamlines localization via embedded JSON files (`lang/<locale>.json`), centralized community packs, and optional language packs defined by `i18n.json`.
- Takeaway: standardize UI assets and translations by delegating to shared dependency mods, simplifying Vice & Order module UIs and cross-mod coordination.

## Write Everywhere Ecosystem

- Write Everywhere enables customizable text, image, and mesh overlays; supports Wavefront OBJ meshes, image atlases with normal/spec maps, fonts, shaders, and instanced sublayout arrays.
- WEModuleTemplate scaffolds module packs (atlases, layouts, fonts) with publishing guidance (`Resources/` folders, project metadata, mod ID tagging).
- Takeaway: mirror this modular architecture for Vice & Order overlays - ship the core system separately, then distribute content packs with shared templates.

## Asset Packs Manager

- Provides a playset-aware asset catalog with thumbnail/localization fallbacks for legacy packs and reporting for missing assets.
- Encourages distribution through Paradox Mods while still exposing local asset diagnostics and startup warning mitigation.
- Takeaway: leverage similar tooling to audit Vice & Order asset dependencies, detect drift between modules, and keep localization metadata consistent.
