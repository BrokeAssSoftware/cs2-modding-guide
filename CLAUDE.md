# CLAUDE.md - CS2 Modding Guide

This handbook has a single canonical operating manual: **[AGENTS.md](AGENTS.md)**.

Before adding to or editing anything here, read `AGENTS.md` in full. It defines the
non-negotiables that keep this a coherent, compounding knowledge base:

- **Diataxis discipline** - every page is one mode (tutorial / how-to / reference / explanation); never mix.
- **Source-of-truth rule** - recipe/case-study claims cite real `repo/<path>#Lxx` from a cloned mod at a pinned commit (via its dossier); never document from inference or a storefront; label unverified as `Needs Verification`.
- **Templates + frontmatter** - use `.templates/recipe.md` / `.templates/case-study.md`; carry `diataxis`, `source_version`, `last_reverified`, `canonical_mods`, `technique_applicability`, `status` (see AGENTS.md rule 4 for the status vocabulary + currency fields; `cs2_version_verified` is deprecated).
- **Stage-8 Fold-In loop** - the mandatory way new mod research enters the handbook (recipe + case study + resolved cross-links + `technique-index.md` update).
- **Version-tagging, staleness, improvement mandate, coverage/novelty** - see `AGENTS.md`.

Human orientation and the chapter index live in **[README.md](README.md)**; maintenance
workflow (validation commands, handoff) lives in **[GUIDE.md](GUIDE.md)**.
