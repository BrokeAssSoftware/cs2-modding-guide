# CS2 Modding Guide - Agents Briefing

Use this file as the entry point for LLM agents, automation scripts, and new contributors who need to navigate `docs/cs2-modding-guide/` quickly.

## Mission Overview
- **Goal:** Provide a single source of truth for Cities: Skylines II modding within the Vice & Order ecosystem.
- **Audience:** Engineers, technical writers, tooling automatisms, and onboarding assistants.
- **Structure:** The directory is organised by theme (setup, architecture, simulation, UI, tooling, content, dependencies, operations).

## High-Priority Reads
1. [Setup Overview](setup/overview.md) - launch flags, toolchain installation, project bootstrap, and publishing routines.
2. [Architecture Overview](architecture/overview.md) - module lifecycle, dependency checks, shared terminology.
3. [Simulation Overview](simulation/overview.md) - DOTS replacement patterns, templates, diagnostics, and testing.
4. [UI Overview](ui/overview.md) plus [React Pipeline](ui/react-pipeline/overview.md) - Options UI, key bindings, runtime panels, and Gameface workflows.
5. [Operations Overview](operations/overview.md) - logging, debugging, release checklist, automation ideas, and incident response.
6. [Content Overview](content/overview.md) - PBR workflow, editor integration, map authoring, and publishing support.
7. **Options <-> UI Integration** – review the “Vice & Order Console” section in [React Pipeline (overview)](ui/react-pipeline/overview.md) for the current split between C# trigger bindings (`ViceAndOrder/Mod`) and the React modal (`vno-ui`).

## Cross-References
- External wiki links and research snapshots live in `docs/research/wiki/` and `docs/research/mods/`.
- Shared dependency docs reference individual module briefings under `docs/vision/` and `docs/modules/`.
- Backlog stories cite terminology and performance targets from [Performance Targets and Terminology](architecture/performance-and-terminology.md).

## Agent Tasks & Conventions
- When answering technical questions, link to the specific path and section headings rather than reproducing entire files.
- Maintain ASCII output in generated docs; non-ASCII characters require explicit justification and existing usage.
- Reference dependencies by both human-readable name and mod ID (see [Shared Icon Library](dependencies/shared-icon-library.md), [I18n Everywhere Integration](localization/i18n-integration.md), and the optional [Shared Library: ExtraLib](dependencies/shared-library-extra.md) guide).
- Note which side of the Vice & Order console pipeline a change touches (C# backend vs React frontend) and reflect updates in `ViceAndOrder/Mod.cs` and `vno-ui` docs accordingly.
- Update this file and [README](README.md) whenever major guidance changes.

## Getting Help
- Reference exact file paths and headings when raising questions or issues.
- When docs conflict, defer to the most recently updated file (tracked via git) and flag inconsistencies for follow-up.

Keep this briefing current so agents and teammates can navigate the guide without context-switching.

