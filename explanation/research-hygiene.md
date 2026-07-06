---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Research Hygiene: Verifying (or Gating) a Mod You Cannot Read"
Summary: The discipline behind this handbook's source-of-truth rule - how to tell when a mod's internals are actually verifiable, what to do when the storefront lies or the payload is locked, and how to archive an honest gap instead of fabricating one.
diataxis: explanation
source_version: "~1.4-1.6.0f1 (research-process observations; demand-modifier@e93ec1c repo cites)"
last_reverified: "2026-07-05"
status: source-verified
canonical_mods:
  - demand-modifier@e93ec1c
technique_applicability: [core]
Created: 2026-07-05
Updated: 2026-07-05
Owners:
  - codex
---

# Research Hygiene: Verifying (or Gating) a Mod You Cannot Read

Every code claim in this handbook must be cited to real source at a pinned commit
(see [AGENTS.md](../AGENTS.md) rule 2). That rule is only trustworthy if you also know
**when a mod's internals are genuinely unverifiable** - and refuse to invent them. This page
is the counterpart discipline: how to read a storefront listing skeptically, what is and is
not fetchable, and how to archive an honest gap with a path back to closing it.

The recurring failure it prevents: writing a confident recipe from a storefront description,
a stale public repo, or a mislabeled tag - producing text that looks source-verified but was
never grounded in the shipping build.

## The tag is not the evidence

A Paradox Mods "Code Mod" tag means nothing about whether there is code you can read. Two
asset/prefab packs in this corpus (bridge-expansion, intersection-mod-series-2) carry the
"Code Mod" tag while shipping only assets. Verify **content**, not the label. Mis-tag tells:

- an empty `changelog` and empty `requirements` block,
- a `longDescription` that only describes what the mod does (no build/config surface),
- prebuilt-layout screenshots rather than in-editor/tooling screenshots,
- and, decisively, **no linked public source repository**.

When those signals line up, treat "Code Mod" as unverified metadata, not a promise of source.

## What is locked, and why an empty `repo/` is not negligence

Several payloads simply cannot be fetched without an authenticated game install:

- **The CDN payload is auth-gated.** Unauthenticated fetches of a mod's `repositoryPath`
  (`modscontent.paradox-interactive.com/.../repo/<Platform>__Any`) return `NoSuchKey` / 404.
  Asset internals require an authenticated launcher/Skyve install, then mirroring from the
  local cache (`%LocalAppData%/Colossal Order/Cities Skylines II/Mods/<guid>/repo/...`).
  Reproduced across four asset dossiers (e.g.
  [`bridge-expansion-pack-b-and-p/source.md`](../../vice-and-order-research/mods/dossiers/bridge-expansion-pack-b-and-p/source.md)).
- **The comments/sentiment API is auth-gated too.** The comments route serves only the React
  SPA shell; the REST endpoint returns `Missing Authentication Token`
  ([`anarchy-24/notes/paradox-comments-api-error.txt`](../../vice-and-order-research/mods/dossiers/anarchy-24/notes/paradox-comments-api-error.txt)).
- **`repositoryPath` is a CDN bucket, not a local clone.** Never cite it as if it were
  on-disk source; dossiers with a locked payload carry `references.repo: []`.

An empty `repo/` with a **named blocker** ("no public repo + auth-gated CDN") is a correct,
honest state - not a research failure. It is the signal to gate, not to guess.

## Storefront-versus-source divergence (the demand-modifier lesson)

The published build can be a different program than the public source. Demand Modifier's
public repository is frozen at **v0.3.2** (a Harmony/dropdown design) while the live mod ships
as an unpublished **ECS rewrite** ("Infinite Resources v0.5.2"); the public tree does not even
compile. Two concrete tells, both source-cited:

- **The won't-compile tree's own README maps to code that was deleted.** It still names the
  excised `DemandSystemPatch.cs` (three Harmony classes) that no longer exist in the tree -
  a README pointing at where a removed hook used to live
  ([`demand-modifier/repo/DemandModifier/README.md#L66-L71`](../../vice-and-order-research/mods/dossiers/demand-modifier/repo/DemandModifier/README.md), @e93ec1c).
- **Version stamps disagree with each other.** README/PublishConfiguration and the storefront
  API cite different game versions, and the csproj targets `net472`
  ([`demand-modifier/repo/DemandModifier/DemandModifier.csproj#L8`](../../vice-and-order-research/mods/dossiers/demand-modifier/repo/DemandModifier/DemandModifier.csproj), @e93ec1c)
  while the README describes a different build.

The correct handbook outcome: fold in **only** what the v0.3.2 source proves (its attribute-only
settings UI, the enum-literals-are-the-value idiom, the phased bootstrap logging), and gate the
marquee live behaviour (the ECS resource control) as `Needs Verification` pending a DLL
decompile. The mod's own dossier records this as a HOLD, not a documented feature.

## Label hypotheses; never launder them into findings

When a listing is sparse, carry the unknown as a hypothesis with its source, and do not let it
harden into an asserted mechanism. Anarchy 24 is archived as a "save-only artefact until the
payload is inspected," and its climate/time conflict is explicitly a hypothesis borrowed from a
sibling mod's notes, not a source finding. The intersection-preset mechanism (blueprint vs
prefab vs runtime builder) is likewise carried as an open question, not a documented design.

## Track drift with dated snapshots

Storefronts update silently. Capture dated API snapshots (`paradox-api-YYYYMMDD.json`) and diff
them: one dossier caught a subscriber count and rating moving between two sweeps, and two packs
whose `requiredVersion` lagged the live game (one stuck at `1.4.*` while the game was `1.5.10`,
another drifting `1.3.* -> 1.5.*`). A single capture is a rumor; two dated captures are a delta
you can cite. `source_version` / `last_reverified` on every page exist for exactly this reason.

## Named blocker + reopen triggers

When internals are structurally unverifiable, archive with a **named blocker** and explicit
**reopen triggers** rather than an apologetic gap:

- reopen when a public repo appears,
- when a new version / `requiredVersion` / `changelog` / tag change lands,
- when an authenticated install (or the source) becomes available,
- or when a dependency/state change is observed.

This keeps the gap actionable instead of permanent, and distinguishes "we could not verify this"
from "this is unknowable."

## Dependency drift and recovery

Paradox can transiently unpublish a dependency, breaking downstream packs until players
resubscribe (a render-prefab pack was briefly delisted, stranding its dependents). Two
consequences for authors and researchers: **mirror dependency payloads locally** so a delisting
does not erase your evidence, and **document a resubscribe recovery step** for players. This is
also why runtime dependency validation (detect-and-degrade) matters - see
[dependency strategy](./dependency-strategy.md).

## Cross-link siblings; budget for gated support channels

Two smaller process habits that repeatedly paid off:

- **Cross-link sibling dossiers to seed hypotheses.** The Time & Weather Anarchy dossier naming
  "Anarchy 24" as a possible climate conflict is what seeded that dossier's compatibility
  hypothesis, even though the target listing was undocumented.
- **Expect support channels to be gated.** Ko-fi returns a Cloudflare challenge, Discord history
  needs auth, and Paradox forum threads sit behind a client challenge; when there is no GitHub,
  these are the only update signal and must be captured with an authenticated browser, on a
  later pass, as a named follow-up - not silently skipped.

## See also

- [AGENTS.md](../AGENTS.md) - the source-of-truth rule this page defends (rule 2) and the
  `status` / `source_version` / `last_reverified` conventions (rule 4).
- [Dependency strategy](./dependency-strategy.md) - declare / reference / validate / degrade,
  and the interop/compatibility notes for multi-writer and delisting hazards.
- [External resources](../reference/external-resources.md) - the official wiki, marketplace,
  and community tools (Skyve, Find It) referenced above.
