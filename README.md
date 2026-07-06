---
FrontmatterVersion: 1
DocumentType: Guide
Title: CS2 Modding Handbook README
Summary: Orientation and index for the CS2 Modding Handbook - a source-verified, technique-organized guide for writing Cities Skylines II code and UI mods, built for AI agents and human modders.
Created: 2025-11-20
Updated: 2026-07-04
Owners:
  - codex
References:
  - Label: Agents (operating manual)
    Path: ./AGENTS.md
  - Label: LLM entry index
    Path: ./llms.txt
  - Label: Technique Index
    Path: ./technique-index.md
  - Label: Maintenance guide
    Path: ./GUIDE.md
---

# CS2 Modding Handbook

A working playbook for building **Cities: Skylines II** code and UI mods. Everything here is curated from official Cities: Skylines II documentation, community research, and **source-verified study of real, public mods** - every code claim is cited to a specific mod at a pinned commit.

It is written to serve two audiences: **AI coding agents** writing mods (see [llms.txt](llms.txt) and [AGENTS.md](AGENTS.md)) and **human modders**. If you are contributing, read [AGENTS.md](AGENTS.md) first.

## How this guide is organised (Diataxis)

Every page serves ONE mode (declared in its `diataxis:` frontmatter); modes are the primary navigation:

- **Tutorials** (learning) - `tutorials/`: get from zero to a running mod.
- **How-to** (task) - `how-to/`: accomplish a specific task. Includes the technique **recipes** cookbook.
- **Reference** (information) - `reference/`: look things up. We link out to the official wiki rather than duplicate it.
- **Explanation** (understanding) - `explanation/`: the concepts and "why".
- **Case studies** - `case-studies/`: how techniques combine in real mods.

Two cross-cutting aids: the [Technique Index](technique-index.md) ("which technique, when" - the catalog of technique families + coverage) and [llms.txt](llms.txt) (the agent entry index).

## Index

This index mirrors the agent entry index in [llms.txt](llms.txt); keep the two in sync. Each page
declares a `status` (`source-verified` = code claims cited to a mod at a pinned commit;
`needs-verification` = no pinned source or hedged) - see [AGENTS.md](AGENTS.md).

### Start here
- [AGENTS.md](AGENTS.md) - operating manual (conventions, source-of-truth rule, templates, the fold-in loop).
- [llms.txt](llms.txt) - entry index for AI agents.
- [Technique Index](technique-index.md) - choose an approach (family catalog + coverage status).

### Tutorials (learning)
- [Getting started](tutorials/getting-started.md), [Your first code mod](tutorials/first-code-mod.md), [Your first UI mod](tutorials/first-ui-mod.md), [Build and publish](tutorials/build-and-publish.md)

### How-to (task)
- **Recipes:** [index](how-to/recipes/README.md) - **all 59 technique families (A-BG) are source-verified** (A-AB original; AC-BG from the 2026-07-05 deep re-mine); the [recipes index](how-to/recipes/README.md) has the full grouped list, and the [Technique Index](technique-index.md) maps each family letter to its canonical mods and coverage.
- **Tooling:** [raycast filters](how-to/tooling/raycast-filters.md), [sub-element removal](how-to/tooling/sub-element-removal.md), [transform gizmos](how-to/tooling/transform-gizmos.md), [validation overrides](how-to/tooling/validation-overrides.md).
- **UI:** [options UI](how-to/ui/options-ui.md), [key bindings](how-to/ui/key-bindings.md), [runtime panels](how-to/ui/runtime-ui.md), [dependency handling](how-to/ui/dependency-handling.md), [React dev loop](how-to/ui/react-development.md), [module registry](how-to/ui/module-registry.md), [React testing](how-to/ui/react-testing.md).
- **Content:** [PBR asset workflow](how-to/content/pbr-asset-workflow.md), [editor integration](how-to/content/editor-integration.md), [map authoring](how-to/content/map-authoring.md), [asset-pack management](how-to/content/asset-pack-management.md), [unofficial asset import](how-to/content/unofficial-asset-import.md), [Write Everywhere modules](how-to/content/write-everywhere-modules.md), [support & publishing](how-to/content/support-and-publishing.md).
- **Localization:** [I18n Everywhere integration](how-to/localization/i18n-integration.md).
- **Operations:** [logging & debugging](how-to/operations/logging-and-debugging.md), [MSBuild build/packaging automation](how-to/operations/msbuild-packaging.md), [release checklist](how-to/operations/release-checklist.md), [security & stability](how-to/operations/security-and-stability.md), [UI testing checklist](how-to/operations/ui-testing-checklist.md), [memory & performance](how-to/operations/memory-and-performance.md), [incident response](how-to/operations/incident-response.md), [CI automation](how-to/operations/ci-automation.md), [testing & recovery](how-to/operations/testing-and-recovery.md).

### Explanation (understanding)
- **Fundamentals (read before coding):** [ECS/DOTS fundamentals](explanation/ecs-fundamentals.md), [Harmony patching](explanation/harmony-patching.md), [React UI consume side](explanation/react-ui.md).
- [Mod lifecycle](explanation/mod-lifecycle.md), [system scheduling](explanation/system-scheduling.md), [multi-phase scheduling](explanation/multi-phase-scheduling.md), [module layout](explanation/module-layout.md), [dependency strategy](explanation/dependency-strategy.md), [serialization](explanation/serialization.md), [system replacement](explanation/system-replacement.md), [conditional execution](explanation/conditional-execution.md), [settings vs data](explanation/settings-and-data.md), [Gameface runtime](explanation/gameface-runtime.md), [UI <-> C# communication](explanation/ui-cs-communication.md), [research hygiene](explanation/research-hygiene.md).

### Reference (lookup)
- [System update phases](reference/system-update-phases.md), [SDK & toolchain](reference/sdk-and-toolchain.md), [ECS components catalog](reference/ecs-components-catalog.md), [glossary](reference/glossary.md).
- Game systems: [economy](reference/game-systems/economy.md), [citizens & households](reference/game-systems/citizens-households.md), [districts & policies](reference/game-systems/districts-policies.md), [pathfinding & transit](reference/game-systems/pathfinding-transit.md) (patch-sensitive).
- [Performance & terminology](reference/performance-and-terminology.md), [options UI attributes](reference/options-attributes.md), [external resources](reference/external-resources.md).
- Shared libraries: [overview](reference/shared-libraries/README.md), [Unified Icon Library](reference/shared-libraries/unified-icon-library.md), [ExtraLib](reference/shared-libraries/extralib.md).

### Case studies (worked examples)
- **Original:** [Realistic Path Finding](case-studies/realistic-path-finding.md), [Anarchy](case-studies/anarchy.md), [Achievement Fixer](case-studies/achievement-fixer.md), [Better Bulldozer](case-studies/better-bulldozer.md), [Extra Detailing Suite](case-studies/extra-detailing-suite.md), [Time2Work (Realistic Trips)](case-studies/time2work-realistic-trips.md), [Write Everywhere ecosystem](case-studies/write-everywhere-ecosystem.md), [Localization & UI stack](case-studies/localization-and-ui.md), [Asset Packs Manager](case-studies/asset-packs-manager.md), [Advanced Road Naming](case-studies/advanced-road-naming.md), [Outside Traffic Adjuster](case-studies/outside-traffic-adjuster.md), [Magic Mail](case-studies/magic-mail.md).
- **2026-07-05 deep re-mine:** [Traffic Tool Essentials](case-studies/traffic-tool-essentials.md), [Market Based Economy](case-studies/market-based-economy.md), [Road Speed Adjuster](case-studies/road-speed-adjuster.md), [Smooth Left-Hand Traffic](case-studies/smooth-left-hand-traffic.md), [Realistic Job Search](case-studies/realistic-jobsearch.md), [Magic Garbage Truck](case-studies/magic-garbage-truck.md), [Advanced Simulation Speed](case-studies/advanced-simulation-speed.md), [Abandoned Building Remover](case-studies/abandoned-building-remover.md), [Elections (RT module)](case-studies/elections-rt-module.md), [Custom Chirps](case-studies/custom-chirps.md), [Demand Modifier](case-studies/demand-modifier.md).

## Conventions
- Keep all files clean ASCII.
- Every code/mechanism claim cites real source (`repo/...#L` at a pinned commit) via a mod's research dossier; never document from inference. See [AGENTS.md](AGENTS.md).
- Reference dependencies by both name and mod ID.
- Update [AGENTS.md](AGENTS.md), [llms.txt](llms.txt), and this index when structure changes.

## Provenance
Maintained under the Vice & Order project; the real-mod examples come from that project's source-verified research dossiers. The guide itself is general-purpose CS2 modding knowledge, usable by anyone.
