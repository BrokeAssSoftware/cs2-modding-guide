---
FrontmatterVersion: 1
DocumentType: Guide
Title: CS2 Modding Handbook - Operating Manual
Summary: Operating manual for the CS2 Modding Handbook - the rules for reading, citing, and contributing, for AI agents and human contributors.
Created: 2025-11-20
Updated: 2026-07-04
Owners:
  - codex
References:
  - Label: README (human index)
    Path: ./README.md
  - Label: LLM entry index
    Path: ./llms.txt
  - Label: Technique Index
    Path: ./technique-index.md
  - Label: Research source-of-truth rules
    Path: ../vice-and-order-research/mods/Agents.md
---

# CS2 Modding Handbook - Operating Manual

The entry point for AI agents and contributors editing this handbook. Read this before adding or changing anything.

## Mission
- **Goal:** A single, source-verified source of truth for Cities: Skylines II modding techniques, patterns, and worked examples.
- **Audience:** AI coding agents writing CS2 mods (primary) and human modders (secondary).
- **Structure:** Organised by **Diataxis mode** (tutorials / how-to / reference / explanation) + case studies; every page declares its `diataxis:` mode. Cross-cutting aids: `technique-index.md` (which technique, when) and `llms.txt` (agent entry index).

## Operating rules (NON-NEGOTIABLE)

This is a living, compounding knowledge base built from source-verified mod research. Follow these so it grows coherently.

### 1. Diataxis discipline (one mode per page)
Every content page declares a `diataxis:` mode and serves ONE mode only - never mix:
- **tutorial** (learning) - `tutorials/`: hand-hold a beginner to a first success.
- **how-to** (task) - `how-to/` (incl. `how-to/recipes/`): "how do I accomplish X"; assume competence, be goal-directed.
- **reference** (information) - `reference/`: dry, accurate lookups. **Link out to the official wiki instead of duplicating it.**
- **explanation** (understanding) - `explanation/`, `case-studies/`: the "why" and how pieces fit.
If a page does two jobs, split it. See https://diataxis.fr/.

### 2. Source-of-truth rule (inherited from research `../vice-and-order-research/mods/Agents.md`)
Every code/mechanism claim MUST cite real source: `repo/<path>#Lxx` from a cloned mod at a **pinned commit**, via that mod's dossier (`../vice-and-order-research/mods/dossiers/{slug}/`). **Never document from inference, memory, or a storefront description.** Anything unverified is labelled `Needs Verification`, not stated as fact.
- **Cite at the pinned commit, not the working tree.** A dossier's `repo/` tree can drift ahead of its `repo_snapshot.commit`. Read the pinned snapshot (`git -C <dossier>/repo show <pinned>:<path>`) and cite there. Verify a type/method actually exists at that commit before naming it (a fabricated type name is the most common review failure).

### 3. Write for AI + humans (page conventions)
- **Self-contained**: a page should be usable in isolation (an agent may load only it). Restate essential context; link, don't assume prior pages.
- **Concise + chunkable**: clear H2/H3 headers, short sections, markdown-native. "Good for humans is not always good for agents" - be explicit.
- **Real cited code**: minimal snippets (10-25 lines) from a real mod with the `repo/...#L` citation; no placeholder examples like `"string"`/`123`.
- **Clean ASCII.** Reference dependencies by both name and mod ID.

### 4. Templates + required frontmatter
New pages use `.templates/recipe.md` or `.templates/case-study.md`. Recipes/case-studies carry `diataxis:`, `canonical_mods:` (slug@commit), `technique_applicability:` (domains: core / simulation / economy / ui / content / tooling), plus the currency + status fields below.

**`status` vocabulary (honest + machine-parseable):**
- `source-verified` - every code/mechanism claim on the page carries a `repo/<path>#Lxx` citation resolved at a pinned commit. **Requires `canonical_mods`.** Never stamp `source-verified` without repo cites.
- `needs-verification` - the page has no pinned source, or its key claims are hedged/unconfirmed (e.g. toolchain commands not run, behaviour that needs the running game). Toolchain/tutorial/workflow pages live here until confirmed.
- `draft` - incomplete or in-progress skeleton.

**Currency fields (these replace the older `cs2_version_verified`):**
- `source_version` - the game version the cited source targeted at its pin (e.g. `1.5.9`); `n/a` when the page has no pinned source.
- `last_reverified` - ISO date the page was last checked against the live game.

Current live game: **1.6.0f1 "Summer Solstice"** (2026-06-22), SDK 2.1.2. `cs2_version_verified` is **deprecated**: where it still appears it denotes the handbook's *target* (current) version, NOT proof of verification, and is migrated to `source_version`/`last_reverified` as each page is touched.

### 5. The Stage-8 -> Handbook Fold-In loop (how new knowledge enters)
When a mod dossier reaches Stage 8 in the research project, it is NOT done until:
1. Each technique it teaches -> create or augment the matching `how-to/recipes/{slug}.md` (add the mod as a canonical example + any new pitfall/variation), source-cited.
2. Its case study (`case-studies/{slug}.md`) is created/refreshed from the dossier.
3. The dossier's `guide.md` `recommendations`/`guide_placements` are resolved into real links here (no dangling refs).
4. `technique-index.md` and `llms.txt` are updated (move families `queued` -> `source-verified`; add new pages to the index).

### 6. Version-tagging & staleness
Pages carry the currency fields from rule 4 (`source_version` + `last_reverified`); the current live game is **1.6.0f1 "Summer Solstice"**. Source pins span **~1.4 to 1.6.0f1** (dated by commit): several mods are pinned at the current 1.6.0f1 (e.g. realistic-path-finding, magic-mail, time2work, smooth-left-hand-traffic), most are ~1.5.x, and market-based-economy is the oldest at ~1.4.x. So each page's `source_version` reflects *its* primary pin's era - a source-cited page is not automatically 1.6.0f1-verified. Keep `source_version` honest per page. Stale/unverified content is annotated `Needs Verification`, never silently trusted. `reference/game-systems/*` are patch-sensitive - re-verify each CS2 release. Prefer linking the official wiki over copying it.

### 7. Improvement mandate (deepen the live tree)
The initial rebuild is complete and the archived `.legacy/` draft has been retired. New knowledge enters ONLY through the Stage-8 fold-in loop (rule 5), written against the **live tree**: when a mod dossier is studied, its techniques are folded into the correct Diataxis-mode page here - source-verified at a pinned commit, generalised (no project-internal names), and deepened over any prior stub. Improve pages in place; never regress a `source-verified` page back to inference.

### 8. Coverage & novelty
`technique-index.md` is the ledger of technique families, their canonical mods, applicability, and the uncovered "wanted" list. Use it to judge whether a candidate mod would teach something NEW before investing in it.

## Navigating the handbook
- **Choose an approach:** [Technique Index](technique-index.md).
- **Do a task:** [Recipes](how-to/recipes/README.md) and the rest of `how-to/`.
- **Look something up:** `reference/` (+ the official CS2 wiki, linked from `reference/external-resources`).
- **Understand a concept:** `explanation/`.
- **See it combined in a real mod:** `case-studies/`.
- **Agent entry index:** [llms.txt](llms.txt). **Maintenance workflow:** [GUIDE.md](GUIDE.md).

When structure changes, update this file, `README.md`, and `llms.txt` together so the three stay in sync (README's index and `llms.txt`'s page set must match).
