---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Case study: Asset Packs Manager (storefront-described, unverified)"
case_study: asset-packs-manager
mod: "Asset Packs Manager (78903)"
dossier: ../../vice-and-order-research/mods/dossiers/asset-packs-manager/
repo_commit: "n/a - dossier repo/ is empty (no source clone; storefront/API-described)"
source_version: "n/a - no pinned source"
last_reverified: "2026-07-04"
diataxis: explanation
techniques: []   # maps to currently-uncovered "wanted" families (asset/playset management); no source to attribute a letter
technique_applicability: [content, tooling]
status: needs-verification
Created: 2026-07-03
Updated: 2026-07-03
Owners:
  - codex
---

# Asset Packs Manager - case study

> Asset Packs Manager is the reference tool for a category the handbook does not yet cover
> from source: playset-aware asset-pack management (reconcile, backfill, diagnose). This
> case study describes what it is *documented* to do - not confirmed mechanism - because
> its dossier has **no source clone**, so nothing here can be cited to a pinned commit.

> **Source status - read this first.** Unlike the other case studies in this handbook, the
> claims below are **not source-verified**. The Asset Packs Manager (APM) dossier was
> researched from the Paradox Mods storefront listing and API only; the dossier's `repo/`
> clone is **empty**, so there is no pinned commit and **no `repo/<path>#Lxx` citation is
> possible**. Every "how APM does X" statement is therefore labelled **Needs Verification**
> and describes *documented behaviour*, not confirmed source. Before relying on any internal
> detail (an API name, a folder path, a data format), read the current APM source at
> `github.com/CitiesSkylinesModding/CS2-AssetPacksManager` and confirm it. Contrast this with
> the fully source-cited [Write Everywhere](./write-everywhere-ecosystem.md) and
> [localization & UI](./localization-and-ui.md) case studies.

## What it does / why it's instructive

Asset Packs Manager (APM; Mimonsi, code by Konsi; Paradox Mods id `78903`) is a support
layer for Cities: Skylines II **asset packs** - large collections of buildings, props, and
surfaces. Cities: Skylines II now downloads Region Pack assets natively from Paradox Mods,
but big legacy/community collections still benefit from a management layer. APM is
documented to provide (storefront/API, all **Needs Verification**):

- **Playset integration** and an overview of loaded packs.
- **Thumbnails and localization for legacy packs** that predate current metadata conventions.
- **Reports that identify broken or missing assets**, with optional Skyve warnings.

It is instructive precisely because it sits in a **gap the handbook has not yet covered from
source**. The [Technique Index](../technique-index.md) lists "Asset bundle loading (external)"
and "UI info-panel/tab extensions" among its *wanted* (uncovered) families - APM is a live
example of both. Treat this page as a **specification of the problem space** and a pointer
to the working implementation, not as a verified teardown. The task-oriented checklist for
this space is [how-to: Manage and audit CS2 asset packs](../how-to/content/asset-pack-management.md),
which shares the same source caveat.

## Architecture at a glance (documented, unverified)

No source was available to confirm the system structure, so the following is the
storefront-described behaviour only, decomposed into the three jobs a management layer of
this kind performs. **Each bullet is Needs Verification.**

- **Reconcile packs against the active playset.** APM is documented as "playset-aware" and
  shows an overview of loaded packs, which implies enumerating the active playset and
  cross-checking it against what is present on disk (surfacing both "enabled in playset but
  missing locally" and "present locally but inert"). **Needs Verification.** The exact API
  it uses to read the active playset is not confirmed. A *source-verified* example of
  querying the playset through the PDX SDK exists elsewhere in the handbook - I18n Everywhere
  calls `PlatformManager.instance.GetPSI<PdxSdkPlatform>("PdxSdk").GetModsInActivePlayset()`
  (see the [localization & UI case study](./localization-and-ui.md) and
  [I18n Everywhere integration](../how-to/localization/i18n-integration.md)) - but whether
  APM uses the same entry point is unverified.
- **Backfill metadata for legacy packs.** Documented to generate thumbnails and supply
  localized names/descriptions for packs that ship without them, presumably with a cache so
  browsing a large collection stays responsive. **Needs Verification.** The
  generation/caching strategy and on-disk cache location are not confirmed.
- **Diagnose broken and missing assets.** Documented to produce reports that identify broken
  or missing assets and to optionally raise Skyve warnings, exportable to disk for support.
  **Needs Verification.** The scan's exact checks and the report format are not confirmed.
  (APM's own docs note that startup errors from its "ApmLogger" logging tool are safe to
  ignore - verify against the current build before treating any specific log line as benign.)

## Techniques demonstrated

No technique-family letter is attributed here because there is no source to ground one.
Conceptually, APM occupies uncovered entries on the [Technique Index](../technique-index.md)
"wanted" list:

- **Playset/asset-pack management** - reconciling an asset catalogue against the active
  playset. The only source-verified fragment adjacent to this is the PDX SDK playset query
  documented for I18n Everywhere (cited in the
  [localization & UI case study](./localization-and-ui.md)); APM's own use is unverified.
- **UI info-panel / diagnostics surface** - an in-game overview and exportable report.
  Uncovered from source.

If APM is later cloned and folded in per the Stage-8 loop, these would become real recipe
links with `repo/<path>#Lxx` citations; until then they remain pointers, not lessons.

## Key decisions & tradeoffs (as described)

- **Support layer, not a substitute for the toolchain.** APM's stated stance is that it is
  *not* intended to fully support manually installed assets; it encourages creators to
  publish through the Paradox Modding Toolchain to Paradox Mods while still helping advanced
  users who accept the local-install trade-off. This is a deliberate scoping decision worth
  emulating: a management tool that reconciles and diagnoses, rather than a parallel
  installer. **Needs Verification** for any implementation specifics.
- **Two failure directions are different bugs.** "Enabled in the playset but missing on disk"
  and "present on disk but not in the playset" have different fixes; collapsing them in one
  UI state leads players to chase the wrong problem. (Design guidance, not an APM source
  claim.)
- **Metadata generation must be cached.** Thumbnail/localization backfill for a large legacy
  collection is not free; without a cache, browsing can stall the UI. Whether and how APM
  caches is **Needs Verification**.

## Pitfalls / upstream-watch

- **The whole page is storefront-described.** The headline risk is treating documented
  behaviour as confirmed mechanism. If you fork or extend APM, re-derive the real API surface
  from its source before depending on it. There are deliberately **no `repo/<path>#Lxx`
  citations** on this page because the dossier `repo/` is empty.
- **Version/target drift.** The dossier records APM version `1.7.3` (2025-10-29) with a
  storefront `requiredVersion` of **CS2 1.3.6+** and no declared dependencies. This case
  study carries `status: needs-verification` (no pinned source); the current live game is
  1.6.0f1, which is **not** a confirmed compatibility statement for APM - re-check the current
  listing. **Needs Verification.**
- **"ApmLogger" startup errors.** APM's docs say these are safe to ignore; confirm against
  the current build before treating any specific log line as benign. **Needs Verification.**
- **Local assets and updates do not mix.** A player on a hand-installed pack will not receive
  fixes; APM is documented to warn about this, but make the trade-off explicit rather than
  silent. (Design guidance.)

## Source pointers

- Dossier: `../../vice-and-order-research/mods/dossiers/asset-packs-manager/`
  (index + notes only; **`repo/` is empty - no source clone, no pinned commit**).
- Storefront/API (the only researched surface): Paradox Mods id `78903`, source
  `github.com/CitiesSkylinesModding/CS2-AssetPacksManager`, author Mimonsi (code by Konsi),
  version `1.7.3` (2025-10-29), requiredVersion CS2 `1.3.6+`, no declared dependencies.
- Task checklist (same source caveat):
  [how-to: Manage and audit CS2 asset packs](../how-to/content/asset-pack-management.md).
- Related (source-verified) handbook pages: the runtime asset-import path
  [Unofficial asset import](../how-to/content/unofficial-asset-import.md); the PDX SDK
  playset query in the [localization & UI case study](./localization-and-ui.md); the coverage
  ledger [Technique Index](../technique-index.md).
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding
