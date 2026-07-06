---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Manage and Audit CS2 Asset Packs (Playsets, Metadata, Diagnostics)"
Summary: How to keep a Cities: Skylines II asset-pack collection healthy - reconcile packs against the active playset, backfill missing thumbnails/localization for legacy packs, and run diagnostics that surface broken or missing assets.
diataxis: how-to
source_version: "n/a - no pinned source"
last_reverified: "2026-07-04"
canonical_mods:
  - asset-packs-manager  # dossier has NO source clone (repo/ is empty); every mechanism claim here is Needs Verification
status: needs-verification
Created: 2026-07-03
Updated: 2026-07-03
Owners:
  - codex
References:
  - Label: Import custom decals/props/surfaces at runtime
    Path: ./unofficial-asset-import.md
  - Label: Author a PBR asset (model to import)
    Path: ./pbr-asset-workflow.md
  - Label: Technique Index
    Path: ../../technique-index.md
  - Label: "Reference: shared libraries (forward-ref)"
    Path: ../../reference/shared-libraries/README.md
---

# Manage and Audit CS2 Asset Packs (Playsets, Metadata, Diagnostics)

This how-to is for creators and power users who distribute or maintain **large asset-pack
collections** (buildings, props, surfaces) and need players to understand *what is
installed, what is missing, and what is broken*. Cities: Skylines II now downloads Region
Pack assets natively from Paradox Mods, but big legacy/community collections still benefit
from a management layer that reconciles packs against the active **playset**, backfills
missing metadata, and reports broken assets.

The reference tool in this space is **Asset Packs Manager (APM)** (Paradox Mods id
`78903`, source `github.com/CitiesSkylinesModding/CS2-AssetPacksManager`). Use this page as
the task checklist; use APM as the working implementation.

> **Source status - read this first.** Unlike the other content how-tos in this handbook,
> the claims below are **not source-verified**. The Asset Packs Manager dossier was
> researched from the Paradox storefront listing and API only; the dossier's `repo/` clone
> is **empty**, so there is no pinned commit to cite a mechanism against. Every "how APM
> does X" statement here is therefore labelled **Needs Verification** and describes the
> *documented behaviour*, not confirmed source. Before you rely on an internal detail
> (an API name, a folder path, a data format), read the current APM source at
> `github.com/CitiesSkylinesModding/CS2-AssetPacksManager` and confirm it. Contrast this
> with the runtime-import path in
> [Import custom decals/props/surfaces](./unofficial-asset-import.md), which *is* fully
> source-cited.

---

## When to reach for pack-management tooling

- You ship (or curate) dozens-to-hundreds of assets and players report "some assets are
  missing" without a way to tell *which*.
- Your collection includes **legacy packs** authored before the current metadata
  conventions, so items show blank thumbnails or raw locale-ID names.
- Players run multiple **playsets** and need to know which packs are active where, and
  which are enabled-in-playset but absent-on-disk (or vice-versa).

If you are instead *importing* custom surfaces/decals/props at runtime, that is a
different task - see [Import custom decals/props/surfaces](./unofficial-asset-import.md).

---

## Step 1: Reconcile packs against the active playset

Enumerate the player's active playset and cross-check it against what is actually present
on disk, so discrepancies surface immediately instead of as silent missing content.

- List every installed pack and mark its **origin**: subscribed from Paradox Mods vs. a
  local folder.
- Flag the two failure directions explicitly:
  - **enabled in the playset but missing locally** (the player expects it, it will not
    load), and
  - **present locally but not in the playset** (installed but inert).
- Present the reconciliation as a single overview the player can scan, not buried per-item.

> **Needs Verification.** APM is documented as "playset-aware" and shows an overview of
> loaded packs. The exact API it uses to read the active playset is not confirmed from
> source here. A source-verified example of querying the active playset through the PDX SDK
> exists elsewhere in the handbook - I18n Everywhere calls
> `PlatformManager.instance.GetPSI<PdxSdkPlatform>("PdxSdk").GetModsInActivePlayset()`
> (see [I18n Everywhere integration](../localization/i18n-integration.md)). Confirm whether
> APM uses the same entry point before copying it.

## Step 2: Backfill metadata for legacy packs

Many older packs lack thumbnails or localized names/descriptions, which makes a large
collection hard to browse.

- Generate a thumbnail on the fly when a pack has none, or fall back to a generic
  placeholder image so no item renders blank.
- Allow **community-supplied translations** for pack names/descriptions so non-English
  players get readable labels. (For the general localization mechanism, see the
  [localization helper recipe](../recipes/localization-helper.md) and
  [I18n Everywhere integration](../localization/i18n-integration.md).)
- **Cache** generated thumbnails/metadata so browsing a large collection stays responsive
  and you do not regenerate on every open.

> **Needs Verification.** APM is documented to provide "thumbnails and localization for
> legacy packs." The generation/caching strategy and on-disk cache location are not
> confirmed from source here.

## Step 3: Diagnose broken and missing assets

Give players a way to find problems *before* they turn into a confusing bug report.

- Scan packs for **missing dependencies, outdated references, or corrupt files**.
- Surface issues **inline** (for example a badge next to an affected pack) so a problem is
  visible without opening a full report.
- Let the player **export the report to disk** so they can attach it when asking for
  support.

> **Needs Verification.** APM is documented to produce reports that "identify broken or
> missing assets" and to optionally raise Skyve warnings. The scan's exact checks and the
> report format are not confirmed from source here. (Note: APM's own docs mention that
> startup errors logged by its "ApmLogger" logging tool are safe to ignore - verify
> against the current build before treating any specific log line as benign.)

## Step 4: Keep local-only assets an informed choice

Locally installed assets bypass the Paradox toolchain, so they carry caveats the player
should see.

- Warn that **locally installed assets may not receive updates** and can bypass
  Skyve-style validation.
- Encourage creators to **publish through the Paradox Modding Toolchain** whenever
  possible, while still supporting local packs for advanced users who accept the
  trade-off.
- APM's own stance (per its storefront description) is that it is *not* intended to fully
  support manually installed assets - it is a support layer, not a substitute for the
  official pipeline.

## Step 5: Wire in community support

- Point players at the relevant support channels (the mod's Discord / Paradox forum
  thread) directly from the tool or its README.
- Keep a concise README that covers: how to install a new pack, how to refresh metadata,
  and how to file a bug report (with the exported diagnostic attached).

---

## Pitfalls & gotchas

- **Do not present documented behaviour as confirmed mechanism.** Everything on this page
  is storefront-described. If you fork or extend APM, re-derive the real API surface from
  its source before depending on it.
- **Missing-locally vs. inert-locally are different bugs** with different fixes; collapse
  them in the UI and players will chase the wrong one.
- **Thumbnail/metadata generation is not free.** Without a cache, browsing a large
  collection can stall the UI - budget for caching from the start.
- **Local assets and updates do not mix well.** A player on a hand-installed pack will not
  get fixes; make that trade-off explicit rather than silent.

## Where to go next

- **Import custom decals/props/surfaces at runtime:**
  [Unofficial asset import](./unofficial-asset-import.md) (fully source-cited).
- **Author a new asset end-to-end:** [PBR asset workflow](./pbr-asset-workflow.md).
- **Localize pack names/descriptions:**
  [Localization helper](../recipes/localization-helper.md) and
  [I18n Everywhere integration](../localization/i18n-integration.md).
- **Pick a related technique:** [Technique Index](../../technique-index.md).

## Sources

- Canonical mod (dossier, storefront/API research only - **no source clone**):
  `asset-packs-manager` - Paradox Mods id `78903`,
  `github.com/CitiesSkylinesModding/CS2-AssetPacksManager`. The dossier `repo/` is empty;
  there is no pinned commit, so no `repo/<path>#Lxx` citations are possible and all
  mechanism claims above are **Needs Verification**.
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding
