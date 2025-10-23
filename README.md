# Vice & Order CS2 Modding Guide

Welcome to the working playbook for building and maintaining the Vice & Order mod ecosystem. Everything in `docs/cs2-modding-guide/` is curated from official Cities: Skylines II documentation, community research, and lessons learned from dissected mods such as Realistic Path Finding, Achievement Fixer, and Time2Work.

## Start Here
- [Agents](Agents.md) - orientation for automation assistants and newcomers.
- [Setup Overview](setup/overview.md) - launch profiles, toolchain installation, project bootstrap, and publishing routines.
- [Architecture Overview](architecture/overview.md) - module lifecycle, dependency checks, shared terminology, and API contracts.
- [Simulation Overview](simulation/overview.md) - patterns for replacing vanilla systems, templates, diagnostics, and testing.
- [Operations Overview](operations/overview.md) - logging, debugging, release management, automation, and incident response.
- [UI Overview](ui/overview.md) - Options UI patterns, key bindings, runtime panels, and React guidance.
- [Patterns Overview](patterns/overview.md) - reusable code snippets for common module constructs.

## Setup
- [Launch Profiles and Developer Flags](setup/launch-profiles.md)
- [Toolchain Installation](setup/toolchain-installation.md)
- [Bootstrap a Code Mod](setup/code-mod-bootstrap.md)
- [Bootstrap a UI Project](setup/ui-project-bootstrap.md)
- [Build and Publish Workflow](setup/build-and-publish.md)
- [Environment Health Checks](setup/environment-health-checks.md)

## Architecture
- [Module Layout](architecture/module-layout.md)
- [Lifecycle and Initialization](architecture/lifecycle-and-initialization.md)
- [Settings and Data Management](architecture/settings-and-data.md)
- [System Scheduling](architecture/system-scheduling.md)
- [Dependency Strategy](architecture/dependency-strategy.md)
- [Performance Targets and Terminology](architecture/performance-and-terminology.md)

## Simulation
- [Overview](simulation/overview.md)
- [Replacing Vanilla Systems](simulation/replacing-vanilla-systems.md)
- [System Template](simulation/system-template.md)
- [Multi-Phase Scheduling](simulation/multi-phase-scheduling.md)
- [Conditional Execution and Diagnostics](simulation/conditional-and-diagnostics.md)
- [Testing and Recovery](simulation/testing-and-recovery.md)

## UI and Interaction
- [Options UI Patterns](ui/options.md)
- [Key Binding Pipeline](ui/key-bindings.md)
- [Runtime UI Panels](ui/runtime-ui.md)
- [Dependency Handling](ui/dependency-handling.md)
- [UI Testing Checklist](ui/testing.md)
- [React Pipeline Overview](ui/react-pipeline/overview.md) and supporting pages
- [Options Attribute Reference](ui/reference/options-attributes.md)
- [Shared Icon Library](dependencies/shared-icon-library.md)
- [I18n Everywhere Integration](localization/i18n-integration.md)

## Content and Assets
- [Content Overview](content/overview.md)
- [Unofficial Asset Import](content/unofficial-asset-import.md)
- [Write Everywhere Modules](content/write-everywhere-modules.md)
- [Asset Pack Management](content/asset-pack-management.md)

## Tooling Patterns
- [Raycast Filters](tooling/raycast-filters.md)
- [Sub-Element Removal](tooling/sub-element-removal.md)
- [Validation Overrides](tooling/validation-overrides.md)
- [Transform Gizmos](tooling/transform-gizmos.md)

## Shared Dependencies
- [Shared Library: ExtraLib](dependencies/shared-library-extra.md)
- [Shared Icon Library](dependencies/shared-icon-library.md)

## Development Patterns
- [Patterns Overview](patterns/overview.md)
- [Mod Bootstrap](patterns/mod-bootstrap.md)
- [Settings Patterns](patterns/settings.md)
- [Localisation Helper](patterns/localization.md)
- [System Registration](patterns/system-registration.md)

## Operations
- [Operations Overview](operations/overview.md)
- [Logging and Debugging](operations/logging-and-debugging.md)
- [Memory and Performance](operations/memory-and-performance.md)
- [Security and Stability](operations/security-and-stability.md)
- [Release Checklist](operations/release-checklist.md)
- [Automation and CI](operations/automation-and-ci.md)
- [Incident Response](operations/incident-response.md)

## Case Studies
- [Overview](case-studies/overview.md)
- [Realistic Path Finding](case-studies/realistic-path-finding.md)
- [Achievement Fixer](case-studies/achievement-fixer.md)
- [Time2Work (Realistic Trips)](case-studies/time2work-realistic-trips.md)
- [Anarchy](case-studies/anarchy.md)
- [Better Bulldozer](case-studies/better-bulldozer.md)
- [Extra Detailing Suite](case-studies/extra-detailing-suite.md)
- [Localization & UI Infrastructure](case-studies/localization-and-ui.md)
- [Write Everywhere Ecosystem](case-studies/write-everywhere-ecosystem.md)
- [Asset Packs Manager](case-studies/asset-packs-manager.md)

## Reference Material
- [External Resources](external-resources.md)

## Maintenance Notes
- Keep all files ASCII unless existing content requires otherwise.
- Update [Agents](Agents.md) and this index when major guidance changes.
- Reference mod IDs when documenting dependencies so automation can enforce publish requirements.

Use this directory as the authoritative source for CS2-related workflows inside Vice & Order. Contributions should prioritise actionable steps, code examples, and references to upstream documentation.

