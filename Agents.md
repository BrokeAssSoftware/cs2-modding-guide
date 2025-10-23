# CS2 Modding Guide - Agents Briefing

Use this file as the entry point for LLM agents, automation scripts, and new contributors who need to navigate `docs/cs2-modding-guide/` quickly.

## Mission Overview
- **Goal:** Provide a single source of truth for Cities: Skylines II modding within the Vice & Order ecosystem.
- **Audience:** Engineers, technical writers, tooling automatisms, and onboarding assistants.
- **Structure:** The directory is organised by theme (setup, architecture, systems, UI, tooling, content, dependencies).

## High-Priority Reads
1. [Setup Overview](setup/overview.md) - launch flags, toolchain installation, project bootstrap, and publishing routines.
2. [Architecture Overview](architecture/overview.md) - module lifecycle, dependency checks, shared terminology.
3. [Simulation Systems](simulation-systems.md) - DOTS patterns, replacement workflow, testing checklist.
4. [UI and Options](ui-and-options.md) plus [UI React Pipeline](ui-react-pipeline.md) - UI patterns, settings, and Gameface integration.
5. [Quality and Operations](quality-and-operations.md) - logging, debugging, release checklist, automation ideas.

## Cross-References
- External wiki links and research snapshots live in `docs/research/wiki/` and `docs/research/mods/`.
- Shared dependency docs reference individual module briefings under `docs/vision/` and `docs/modules/`.
- Backlog stories cite terminology and performance targets from [Architecture Overview](architecture/overview.md).

## Agent Tasks & Conventions
- When answering technical questions, point requesters to the specific path and section headings rather than reproducing entire files.
- Maintain ASCII output in generated docs; non-ASCII characters require explicit justification and existing usage.
- Reference dependencies by both human-readable name and mod ID (see [Shared Library: ExtraLib](dependencies/shared-library-extra.md), [Shared Icon Library](dependencies/shared-icon-library.md), [I18n Everywhere Integration](localization/i18n-integration.md)).
- If new guidance is added, update both this file and [README](README.md) so automation stays aware of available material.

## Getting Help
- Open questions should reference the relevant doc and line number to keep discussions anchored.
- When docs conflict, defer to the most recently updated file (timestamps tracked via Git commits) and flag inconsistencies for follow-up.

Keep this briefing in sync with major updates so agents and teammates can navigate the guide without context-switching.


