# Vice & Order CS2 Modding Guide

Welcome to the working playbook for building and maintaining the Vice & Order mod ecosystem. Everything in `docs/cs2-modding-guide/` is curated from official Cities: Skylines II documentation, community research, and lessons learned from dissected mods such as Realistic Path Finding, Achievement Fixer, and Time2Work.

## Start Here
- `Agents.md` – orientation file for automation assistants and newcomers.
- `setup-and-toolchain.md` – workstation prep, launch parameters, code/UI build integration, publishing workflow.
- `project-architecture.md` – module lifecycle, dependency checks, shared terminology and API contracts.
- `simulation-systems.md` – DOTS system replacement patterns, multi-phase scheduling, testing checklist.
- `quality-and-operations.md` – logging, debugging, memory hygiene, release checklist, automation ideas.

## UI and Interaction
- `ui-and-options.md` – Options UI patterns, localization workflow, key bindings, runtime UI guidance.
- `ui-react-pipeline.md` – Gameface React setup, hot reload loop, module registry usage, communication with C# systems.
- `ui/shared-icon-library.md` – integrating Unified Icon Library and shipping custom icon hosts.
- `localization/i18n-integration.md` – I18n Everywhere workflow for embedded locales, packs, and fallbacks.

## Content and Assets
- `asset-and-content.md` – PBR workflow, map authoring checklist, unofficial pack management, publishing readiness.
- `content/unofficial-asset-import.md` – structure and load custom surfaces/decals safely.
- `content/write-everywhere-modules.md` – build content packs for Write Everywhere.
- `content/asset-pack-management.md` – playset-aware asset cataloguing and diagnostics.

## Tooling Patterns
- `tooling/raycast-filters.md` – targeted bulldozer/object filtering.
- `tooling/sub-element-removal.md` – remove props/decals/net elements without destroying hosts.
- `tooling/validation-overrides.md` – safe anarchy-style placement overrides.
- `tooling/transform-gizmos.md` – transform panels, snap controls, curated asset menus.

## Shared Dependencies
- `dependencies/shared-library-extra.md` – ExtraLib integration and shared library hygiene.
- `ui/shared-icon-library.md` – Unified Icon Library usage (icons, COUI hosts, QA tips).
- `localization/i18n-integration.md` – I18n Everywhere integration.

## Reference Material
- `case-studies.md` – distilled patterns from community mods with Vice & Order takeaways.
- `code-patterns.md` – bootstrap snippets (`Mod`, `Setting`, localization, system scheduling).
- `options-attributes.md` – attribute quick reference for Options UI.
- `external-resources.md` – curated official wiki pages, community guides, marketplace reconnaissance.

## Maintenance Notes
- Keep all files ASCII unless existing content requires otherwise.
- When adding major guidance, update `Agents.md` and this index.
- Reference mod IDs when documenting dependencies so automation can enforce publish requirements.

Use this directory as the authoritative source for CS2-related workflows inside Vice & Order. Contributions should prioritise actionable steps, code examples, and references to upstream documentation.