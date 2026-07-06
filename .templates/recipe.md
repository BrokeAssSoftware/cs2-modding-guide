---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: <Technique name>"
recipe: <slug-kebab-case>
technique_family: "<A-AB letter> - <family name from technique-index.md>"
diataxis: how-to
source_version: "1.5.x (<dossier>@<commit>)"  # game version the cited PIN targeted (source pins are 1.5.x-era; current live game is 1.6.0f1)
last_reverified: "YYYY-MM-DD"                 # date checked against current game (static source only unless a runtime QA line says otherwise)
canonical_mods:                               # source-of-truth: real cloned mods, pinned commits
  - <dossier-slug>@<commit>
technique_applicability: [core]      # domains: core / simulation / economy / ui / content / tooling
status: draft                         # source-verified (has repo cites at a pin) | needs-verification | draft
Created: YYYY-MM-DD
Updated: YYYY-MM-DD
Owners:
  - codex
---

# <Technique name>

> One-sentence statement of what this recipe lets you do.

## Problem
When and why you reach for this (the situation that makes this the right tool).

## Solution
The approach in 2-4 sentences - the "why before how" before any code.

## Steps & Code
Numbered steps. Each code excerpt is MINIMAL and cited to real source:

```csharp
// <what this shows>
```
Source: `../../vice-and-order-research/mods/dossiers/<slug>/repo/<path>#Lxx` (@<commit>)

Keep snippets short (10-25 lines); cite, don't paste whole files.

## Pitfalls & gotchas
- Concrete failure modes observed in source (e.g. non-stacking `PrefabBase` baseline; silent prefab-miss with no log; persistence asymmetry network-baked vs transient marker; `Entity.Index` key fragility across sessions).
- Mark anything not proven from source as `Needs Verification (in-game)`.

## Variations
Alternative approaches / when to deviate.

## See also
- Related recipes: [...]
- Reference: [...]
- Case studies demonstrating it: [...]

## Sources
- Canonical mods (dossier + repo): `<slug>` @<commit>
- Official/community references (link out, do not duplicate): <url>
