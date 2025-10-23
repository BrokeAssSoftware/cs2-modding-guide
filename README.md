# Vice & Order CS2 Modding Guide

Welcome to the working playbook for building and maintaining the Vice & Order mod ecosystem. Everything in `docs/cs2-modding-guide/` is curated from official Cities: Skylines II documentation, community research, and lessons learned from dissected mods such as Realistic Path Finding, Achievement Fixer, and Time2Work.

## Start Here
- [Agents](Agents.md) - orientation file for automation assistants and newcomers.
- [Setup and Toolchain](setup-and-toolchain.md) - workstation prep, launch parameters, code/UI build integration, publishing workflow.
- [Project Architecture](project-architecture.md) - module lifecycle, dependency checks, shared terminology and API contracts.
- [Simulation Systems](simulation-systems.md) - DOTS system replacement patterns, multi-phase scheduling, testing checklist.
- [Quality and Operations](quality-and-operations.md) - logging, debugging, memory hygiene, release checklist, automation ideas.

## UI and Interaction
- [UI and Options](ui-and-options.md) - Options UI patterns, localization workflow, key bindings, runtime UI guidance.
- [UI React Pipeline](ui-react-pipeline.md) - Gameface React setup, hot reload loop, module registry usage, communication with C# systems.
- [Shared Icon Library](ui/shared-icon-library.md) - integrating Unified Icon Library and shipping custom icon hosts.
- [I18n Integration](localization/i18n-integration.md) - I18n Everywhere workflow for embedded locales, packs, and fallbacks.

## Content and Assets
- [Asset and Content](asset-and-content.md) - PBR workflow, map authoring checklist, unofficial pack management, publishing readiness.
- [Unofficial Asset Import](content/unofficial-asset-import.md) - structure and load custom surfaces/decals safely.
- [Write Everywhere Modules](content/write-everywhere-modules.md) - build content packs for Write Everywhere.
- [Asset Pack Management](content/asset-pack-management.md) - playset-aware asset cataloguing and diagnostics.

## Tooling Patterns
- [Raycast Filters](tooling/raycast-filters.md) - targeted bulldozer/object filtering.
- [Sub-Element Removal](tooling/sub-element-removal.md) - remove props/decals/net elements without destroying hosts.
- [Validation Overrides](tooling/validation-overrides.md) - safe anarchy-style placement overrides.
- [Transform Gizmos](tooling/transform-gizmos.md) - transform panels, snap controls, curated asset menus.

## Shared Dependencies
- [Shared Library: ExtraLib](dependencies/shared-library-extra.md) - ExtraLib integration and shared library hygiene.
- [Shared Icon Library](ui/shared-icon-library.md) - Unified Icon Library usage (icons, COUI hosts, QA tips).
- [I18n Everywhere Integration](localization/i18n-integration.md) - I18n Everywhere integration.

## Reference Material
- [Case Studies](case-studies.md) - distilled patterns from community mods with Vice & Order takeaways.
- [Code Patterns](code-patterns.md) - bootstrap snippets (`Mod`, `Setting`, localization, system scheduling).
- [Options Attributes](options-attributes.md) - attribute quick reference for Options UI.
- [External Resources](external-resources.md) - curated official wiki pages, community guides, marketplace reconnaissance.

## Maintenance Notes
- Keep all files ASCII unless existing content requires otherwise.
- When adding major guidance, update [Agents](Agents.md) and this index.
- Reference mod IDs when documenting dependencies so automation can enforce publish requirements.

Use this directory as the authoritative source for CS2-related workflows inside Vice & Order. Contributions should prioritise actionable steps, code examples, and references to upstream documentation.
