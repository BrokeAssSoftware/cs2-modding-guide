# Vice & Order CS2 Modding Guide

Welcome to the working playbook for building and maintaining the Vice & Order mod ecosystem. Everything in `docs/cs2-modding-guide/` is curated from official Cities: Skylines II documentation, community research, and lessons learned from dissected mods such as Realistic Path Finding, Achievement Fixer, and Time2Work.

## Start Here
- [Agents](Agents.md) - orientation file for automation assistants and newcomers.
- [Setup Overview](setup/overview.md) - links to launch profiles, toolchain installation, project bootstrap, and publishing routines.
- [Architecture Overview](architecture/overview.md) - module lifecycle, dependency checks, shared terminology and API contracts.
- [Simulation Systems](simulation-systems.md) - DOTS system replacement patterns, multi-phase scheduling, testing checklist.
- [Quality and Operations](quality-and-operations.md) - logging, debugging, memory hygiene, release checklist, automation ideas.

## Setup
- [Launch Profiles and Developer Flags](setup/launch-profiles.md) - configure Steam, Xbox / PC Game Pass, and desktop shortcuts.
- [Toolchain Installation](setup/toolchain-installation.md) - install Unity, the modding project template, .NET, and Node.
- [Bootstrap a Code Mod](setup/code-mod-bootstrap.md) - generate the base C# project and add it to the repository.
- [Bootstrap a UI Project](setup/ui-project-bootstrap.md) - scaffold the Gameface React bundle and align IDs.
- [Build and Publish Workflow](setup/build-and-publish.md) - wire MSBuild, run daily build loops, and publish releases.
- [Environment Health Checks](setup/environment-health-checks.md) - verify folders, certificates, and regression saves.

## Architecture
- [Module Layout](architecture/module-layout.md) - naming conventions, folder structure, shared props.
- [Lifecycle and Initialization](architecture/lifecycle-and-initialization.md) - standard load sequence and sample Mod implementation.
- [Settings and Data Management](architecture/settings-and-data.md) - options, localisation, logging, data boundaries.
- [System Scheduling](architecture/system-scheduling.md) - replacing vanilla systems and managing update phases.
- [Dependency Strategy](architecture/dependency-strategy.md) - coordinate shared libraries and fallbacks.
- [Performance Targets and Terminology](architecture/performance-and-terminology.md) - hardware budgets, vocabulary, draft contracts.

## UI and Interaction
- [UI and Options](ui-and-options.md) - Options UI patterns, localization workflow, key bindings, runtime UI guidance.
- [UI React Pipeline](ui-react-pipeline.md) - Gameface React setup, hot reload loop, module registry usage, communication with C# systems.
- [Shared Icon Library](dependencies/shared-icon-library.md) - integrating Unified Icon Library and shipping custom icon hosts.
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
- [Shared Icon Library](dependencies/shared-icon-library.md) - Unified Icon Library usage (icons, COUI hosts, QA tips).
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




