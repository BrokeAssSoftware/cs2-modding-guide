# Vice & Order CS2 Modding Guide

This guide curates everything we learned from official Cities: Skylines II docs, community wikis, and the codebases we studied (`RealisticPathFinding`, `AchievementFixer`, `Time2Work`, and the vanilla templates). Use it as the working playbook when building and maintaining the Vice & Order module stack.

## Core References
- `setup-and-toolchain.md` - workstation prep, launch flags, build and publish flow.
- `project-architecture.md` - base mod structure, folders, logging, dependency patterns.
- `simulation-systems.md` - ECS update phases, system replacement, harmony considerations.
- `ui-and-options.md` - Options UI attributes, localization, input, React module wiring.
- `ui-react-pipeline.md` - JS/TS build setup, hot reload, bundling UI with code mods.
- `asset-and-content.md` - asset creation, map/editor workflow, publishing checklists.
- `quality-and-operations.md` - debugging, memory, security, validation automation.
- `case-studies.md` - distilled lessons from researched mods with Vice & Order hooks.
- `code-patterns.md` - bootstrap snippets for `Mod`, `Setting`, localization, and system scheduling.
- `options-attributes.md` - attribute quick reference for building Options UI classes.
- `external-resources.md` - curated list of official wiki pages, community guides, and sample mods.

## Tooling Patterns
- `tooling/raycast-filters.md` - limit bulldozer or object raycasts to specific entity families.
- `tooling/sub-element-removal.md` - remove props, decals, and upgrades without deleting the host asset.
- `tooling/validation-overrides.md` - implement anarchy-style error suppression with safeguards.
- `tooling/transform-gizmos.md` - add transform panels, snap toggles, and curated asset menus.

## Content & Asset Pipelines
- `content/unofficial-asset-import.md` - load custom surfaces and decals before the official editor.
- `content/write-everywhere-modules.md` - build content packs for Write Everywhere overlays.
- `content/asset-pack-management.md` - manage large asset collections with playset awareness.

## Shared Dependencies
- `dependencies/shared-library-extra.md` - structure and version shared helper libraries.
- `ui/shared-icon-library.md` - consume SVG icon packs via Unified Icon Library.
- `localization/i18n-integration.md` - integrate I18n Everywhere for translation support.

Reference links use the official wiki slugs documented in `Agents.md`. Update this index when new sections land.
