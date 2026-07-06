---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Case study: <Mod name>"
case_study: <slug-kebab-case>
mod: "<Mod name> (<modId>)"
dossier: ../../vice-and-order-research/mods/dossiers/<slug>/
repo_commit: <commit>
source_version: "1.5.x (<slug>@<commit>)"  # game version the cited PIN targeted (current live game is 1.6.0f1)
last_reverified: "YYYY-MM-DD"              # date checked against current game (static source only)
diataxis: explanation
techniques: [A, B]                    # technique-family letters from technique-index.md
technique_applicability: [core]      # domains: core / simulation / economy / ui / content / tooling
status: draft                         # source-verified (has repo cites at a pin) | needs-verification | draft
Created: YYYY-MM-DD
Updated: YYYY-MM-DD
Owners:
  - codex
---

# <Mod name> - case study

> One-sentence framing: what makes this mod instructive.

## What it does / why it's instructive
Player-facing purpose + why it's a good teaching example.

## Architecture at a glance
The systems, their `SystemUpdatePhase` scheduling, and data flow. Cite `repo/...#Lxx`.

## Techniques demonstrated
Bullet each technique this mod shows, linking to its recipe:
- [<Technique>](../how-to/recipes/<slug>.md) - how it's used here (`repo/...#Lxx`).

## Key decisions & tradeoffs
Notable design choices (e.g. no-Harmony pure-ECS vs Harmony patching; persistence model; dependency strategy).

## Pitfalls / upstream-watch
Fragilities and maintenance risks surfaced from source (fabrication-corrected, source-grounded).

## Source pointers
- Dossier: `../../vice-and-order-research/mods/dossiers/<slug>/` (index/source/modding/guide + notes)
- Repo @<commit>
