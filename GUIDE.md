---
FrontmatterVersion: 1
DocumentType: Guide
Title: CS2 Modding Guide Operations
Summary: Operational guide for maintaining the CS2 modding guide, folding in existing README and Agents expectations.
Created: 2025-11-20
Updated: 2026-07-04
Owners:
  - codex
References:
  - Label: Workspace Guide
    Path: ../../WORKSPACE_GUIDE.md
  - Label: Workspace AGENTS
    Path: ../../AGENTS.md
  - Label: CS2 Modding Guide README
    Path: ./README.md
  - Label: CS2 Modding Handbook Operating Manual
    Path: ./AGENTS.md
---

# CS2 Modding Guide Operations

Use this guide to keep the modding playbook current and aligned with the broader CS2 modding ecosystem.

## Working Rules
- Ingest `README.md` and `AGENTS.md` before edits so navigation and high-priority reads stay in sync. `AGENTS.md` is the canonical operating manual (Diataxis discipline, source-of-truth rule, `status`/currency frontmatter conventions, the fold-in loop).
- Preserve document-relative links; use `.templates/recipe.md` / `.templates/case-study.md` and carry the required frontmatter (`diataxis`, `status`, `source_version`, `last_reverified`, `canonical_mods`, `technique_applicability`).
- Reflect upstream changes: when Colossal docs or community guides change, update the relevant section, refresh `last_reverified`, and note deltas in commit messages.

## Update Workflow
1. Identify the section cluster (tutorials, how-to (incl. recipes), reference, explanation, case-studies).
2. Edit or add Markdown with frontmatter; keep paths relative and ASCII where possible.
3. Update `AGENTS.md` if high-priority reads or pipelines change; update `README.md` index to surface new material.
4. Run link/frontmatter validation (`deno task prompts:verify-frontmatter`, `deno task link:verify`) from workspace root.

## Cross-Project Alignment
- When guidance affects the mod stack, cross-link to `../vice-and-order/` docs and relevant backlog entries in `../vice-and-order-planning/plan/`.
- Pull research references from `../vice-and-order-research/wiki/` or `mods/` and cite capture paths when summarizing.

## Handoff
- Log what changed and which sources were used in References; capture commands run if automation was involved.
- Keep `README.md` and `AGENTS.md` updated to avoid divergence for onboarding and agents.
