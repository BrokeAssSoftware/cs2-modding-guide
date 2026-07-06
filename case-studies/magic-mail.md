---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Case study: Magic Mail"
case_study: magic-mail
mod: "Magic Mail"
dossier: ../../vice-and-order-research/mods/dossiers/magic-mail/
repo_commit: 6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47
source_version: "1.6.0f1 (magic-mail@6fb3d2b; date-pinned, static source only)"
last_reverified: "2026-07-04"
diataxis: explanation
techniques: [L, K, J, A]
technique_applicability: [economy, simulation, core]
status: source-verified
Created: 2026-07-04
Updated: 2026-07-04
Owners:
  - codex
Summary: How Magic Mail combines a slow periodic resource-buffer system with a one-shot, non-stacking prefab-multiplier system to keep postal logistics playable without touching the vanilla mail pipeline.
---

# Magic Mail - case study

> A small economy mod that never patches the vanilla mail systems: it runs one
> slow periodic scan that nudges each post facility's `Resources` buffer, and one
> self-disabling system that rescales postal prefab capacities from immutable
> authoring baselines so the multipliers can never stack.

All code claims below are cited to `repo/<path>#Lxx` at pinned commit
`6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47`, surfaced through the dossier at
`../../vice-and-order-research/mods/dossiers/magic-mail/`. Every type and member
named here was confirmed to exist at that commit via `git show`.

## What it does / why it's instructive

Magic Mail (author River-Mochi, namespace `MagicMail`) makes Cities: Skylines II
postal logistics less fiddly. It periodically "tops up" the mail buffers of post
offices and sorting facilities so they keep working, optionally clamps overflowing
storage back down, and lets the player rescale post-van / truck / facility mail
capacity and sorting speed through Options sliders.

It is instructive because it is a **minimal, clean pairing of two very different
update disciplines** that share the same problem domain (postal prefabs and their
`Resources` buffers) but need opposite scheduling:

- a **slow, always-running periodic** simulation system that mutates live
  building entities, and
- a **one-shot, event-woken** system that mutates prefab *data* components once
  and then switches itself off.

Neither uses Harmony. There are no decompiled clones. It is a compact reference
for how to write economy nudges that coexist with the stock simulation instead of
replacing it - and for the discipline needed to make a "percentage" slider that
does not compound every time it fires.

## Architecture at a glance

Both systems are registered in `Mod.OnLoad` into
`SystemUpdatePhase.GameSimulation` via `UpdateBefore`
(repo/Mod.cs#L110-L114 @ 6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47). There is no
Harmony, no `PatchAll`, and no system disabling; the mod only *adds* two systems.

### MagicMailSystem - the periodic scanner

`MagicMailSystem` is a `GameSystemBase` that overrides `GetUpdateInterval` to run
on a slow cadence. In Release it returns `262144 / UpdatesPerDay` where
`UpdatesPerDay = 32`, i.e. `8192` ticks (~once per 45 in-game minutes); under a
`DEBUG` build it instead returns `256` to match the vanilla
`PostFacilityAISystem` cadence
(repo/Systems/MagicMailSystem.cs#L57-L67 @ 6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47).
`GetUpdateOffset` returns `48` to keep the system in a fixed slot "before the
vanilla system"
(repo/Systems/MagicMailSystem.cs#L71-L75 @ 6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47).

`OnCreate` builds an `EntityQuery` for live post facilities - entities with
`PrefabRef`, `Game.Buildings.PostFacility`, and a read-write `Resources` buffer,
excluding `Destroyed` / `Deleted` / `Temp` - and calls `RequireForUpdate` on it
(repo/Systems/MagicMailSystem.cs#L84-L100 @ 6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47).
Each `OnUpdate` reads `PostFacilityData` off the entity's prefab to learn its
`m_MailCapacity` and `m_SortingRate`, fetches the entity's `DynamicBuffer<Resources>`,
and branches: `m_SortingRate == 0` is treated as a plain post office, otherwise a
sorting facility
(repo/Systems/MagicMailSystem.cs#L136-L198 @ 6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47).

### MailCapacitySystem - the one-shot multiplier

`MailCapacitySystem` is a `sealed` `GameSystemBase` that starts disabled
(`Enabled = false` at the end of `OnCreate`) and only wakes on two triggers: a
real-game load (`OnGameLoadingComplete` sets `Enabled = true` when
`mode == GameMode.Game` and purpose is `NewGame` / `LoadGame`) and a settings
change (see below)
(repo/Systems/MailCapacitySystem.cs#L40-L77 @ 6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47).
`GetUpdateInterval` returns `1`, and `OnUpdate` finishes by setting
`Enabled = false` again, so it runs exactly one pass per wake
(repo/Systems/MailCapacitySystem.cs#L82-L130 @ 6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47).

Its queries target prefab *data* entities (`PostFacilityData`, `PostVanData` with
`PrefabData`), not live buildings - it edits the shared prefab components that the
vanilla AI reads
(repo/Systems/MailCapacitySystem.cs#L47-L59 @ 6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47).

### Data flow / wake path

`Setting.Apply()` re-enables both systems on any slider change: it fetches each
managed system from `World.DefaultGameObjectInjectionWorld` and sets
`Enabled = true`, giving "instant" slider feedback via the one-shot system while
the periodic scanner resumes its slow cadence
(repo/Settings/Settings.cs#L107-L131 @ 6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47).

## Techniques demonstrated

- [Periodic UpdateInterval system](../how-to/recipes/periodic-updateinterval-system.md)
  (family L) - `MagicMailSystem` sets its own cadence by overriding
  `GetUpdateInterval` (`262144 / 32 = 8192` ticks Release, `256` DEBUG) and pins
  its slot with `GetUpdateOffset = 48`, the canonical "run my simulation logic on
  a slow tick" idiom
  (repo/Systems/MagicMailSystem.cs#L57-L75 @ 6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47).
- [DynamicBuffer resources nudge](../how-to/recipes/dynamicbuffer-resources-nudge.md)
  (family K) - each scan fetches `EntityManager.GetBuffer<Resources>(postEntity)`
  and mutates it in place through local `GetResourceAmount` / `AddResourceAmount`
  helpers that walk the buffer by `m_Resource`, top up `LocalMail` / `UnsortedMail`
  when under a threshold, and proportionally clamp overflow back toward a target
  total
  (repo/Systems/MagicMailSystem.cs#L155-L161 @ 6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47;
  repo/Systems/MagicMailSystem.cs#L428-L472 @ 6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47).
- [One-shot prefab multiplier](../how-to/recipes/one-shot-prefab-multiplier.md)
  (family J) - `MailCapacitySystem` runs once per wake and disables itself, and
  crucially scales from *cached vanilla baselines* rather than the current field
  value, so repeated applies never compound
  (repo/Systems/MailCapacitySystem.cs#L62-L63 @ 6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47;
  repo/Systems/MailCapacitySystem.cs#L128-L130 @ 6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47;
  repo/Systems/MailCapacitySystem.cs#L174-L188 @ 6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47).
- [Prefab field override](../how-to/recipes/prefab-field-override.md) (family A) -
  the baseline is read off the authoring `PrefabBase` via
  `PrefabSystem.TryGetPrefab` + `PrefabBase.TryGet<PostVan>` /
  `TryGet<PostFacility>`, then written back into the runtime `PostVanData` /
  `PostFacilityData` components (`m_MailCapacity`, `m_PostVanCapacity`,
  `m_PostTruckCapacity`, `m_SortingRate`)
  (repo/Systems/MailCapacitySystem.cs#L212-L253 @ 6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47;
  repo/Systems/MailCapacitySystem.cs#L132-L209 @ 6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47).

Non-stacking multiplier, cited minimal snippet:

```csharp
// Baseline is the immutable authoring value, read from PrefabBase - never the
// current (possibly already-scaled) component field. So re-applying is idempotent.
if (!TryGetPostFacilityBaseline(prefabEntity, out FacilityBaseline baseline))
    continue;
...
int newPostVanCapacity   = ScalePercentKeepZero(baseline.PostVanCapacity, vanFleetPercent);
int newPostTruckCapacity = ScalePercentKeepZero(baseline.PostTruckCapacity, truckFleetPercent);
```
(repo/Systems/MailCapacitySystem.cs#L167-L178 @ 6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47)

## Key decisions & tradeoffs

- **Two systems, two clocks.** The core design decision is the split between a
  periodic `MagicMailSystem` (mutates live building `Resources` buffers on a slow
  ~45-minute tick) and a one-shot `MailCapacitySystem` (mutates prefab capacity
  *data* on demand and sleeps). Capacity changes must feel instant and must not
  run every tick; mail top-ups must run continuously but cheaply. Encoding those
  as separate systems with different `GetUpdateInterval` semantics keeps each
  simple
  (repo/Systems/MagicMailSystem.cs#L59-L67 @ 6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47;
  repo/Systems/MailCapacitySystem.cs#L82-L85 @ 6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47).
- **Non-stacking via immutable baseline.** Percentage sliders are the classic
  stacking trap - scale a field by 150% twice and you get 225%. Magic Mail avoids
  this by always scaling from a `FacilityBaseline` captured from the authoring
  `PrefabBase`, so the result depends only on the current slider, not on prior
  applies
  (repo/Systems/MailCapacitySystem.cs#L32-L38 @ 6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47;
  repo/Systems/MailCapacitySystem.cs#L230-L253 @ 6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47).
- **Self-disabling instead of polling.** `MailCapacitySystem` does not check "did
  a setting change?" every frame; it stays `Enabled = false` and is externally
  woken by `Setting.Apply()` and by a real-game load, then flips itself off. This
  keeps a capacity updater off the hot path while still being reactive
  (repo/Systems/MailCapacitySystem.cs#L62-L77 @ 6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47;
  repo/Settings/Settings.cs#L127-L131 @ 6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47).
- **Floor at 1, keep zero at zero.** `ScalePercentMin1` floors any positive
  base value at `1` after rounding so a facility never rounds down to a broken
  zero capacity, while both scalers preserve genuine zeros (`baseValue <= 0`
  returns `0`) so non-applicable fields stay off
  (repo/Systems/MailCapacitySystem.cs#L255-L273 @ 6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47).
- **Coexist, don't replace.** No Harmony and no disabled vanilla systems: the mod
  reads `PostFacilityData` / the vanilla `MailAccumulationSystem` and writes the
  same buffers the stock AI uses, so it layers onto the running simulation instead
  of owning it
  (repo/Systems/MagicMailSystem.cs#L212-L221 @ 6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47;
  repo/Mod.cs#L110-L114 @ 6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47).

## Pitfalls / upstream-watch

- **DEBUG-vs-Release interval divergence.** `GetUpdateInterval` returns `256`
  under `#if DEBUG` but `262144 / 32 = 8192` in Release - a 32x cadence gap. A
  behaviour tuned or timed against a debug build will fire far more often than the
  shipped mod; always reason about the Release branch
  (repo/Systems/MagicMailSystem.cs#L59-L67 @ 6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47).
- **Baseline capture depends on `PrefabBase.TryGet`.** The non-stacking guarantee
  hinges on being able to read the authoring `PostVan` / `PostFacility` component
  off `PrefabBase`. If a prefab lacks that authoring component (`TryGet` returns
  false) the entity is skipped and never rescaled - a modded or DLC postal prefab
  could silently be excluded
  (repo/Systems/MailCapacitySystem.cs#L216-L227 @ 6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47;
  repo/Systems/MailCapacitySystem.cs#L234-L242 @ 6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47).
- **Field-name coupling to `Game.dll`.** The mod reads/writes concrete vanilla
  fields - `PostFacilityData.m_MailCapacity` / `m_SortingRate` /
  `m_PostVanCapacity` / `m_PostTruckCapacity`, `PostVanData.m_MailCapacity`,
  `PostFacility.m_MailStorageCapacity` / `m_PostVanCapacity`, `PostVan.m_MailCapacity`.
  If Colossal renames or restructures these in a patch, both systems fail to
  compile or silently no-op after a rebuild
  (repo/Systems/MailCapacitySystem.cs#L226-L249 @ 6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47;
  repo/Systems/MagicMailSystem.cs#L152-L153 @ 6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47).
- **Stats are best-effort.** `MailAccumulationSystem` is resolved lazily via
  `World.GetExistingSystemManaged` inside a try/catch; if it is absent the
  city-wide mail stats simply stay stale rather than erroring
  (repo/Systems/MagicMailSystem.cs#L478-L491 @ 6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47).
- **`ScalePercentMin1` vs `ScalePercentKeepZero` are currently identical.** Both
  helpers floor positive values at `1` and return `0` for non-positive input; the
  two names document intent (van/truck *fleet* counts vs facility mins) but there
  is no behavioural difference at this commit, so a future edit to one must not be
  assumed applied to the other
  (repo/Systems/MailCapacitySystem.cs#L255-L273 @ 6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47).

Needs Verification (in-game): the real-world "feel" of the top-up threshold and
overflow ratio, and whether the ~45-minute Release cadence keeps facilities
visibly stocked under heavy postal demand, require the running game to confirm.

## Source pointers

- Dossier:
  `../../vice-and-order-research/mods/dossiers/magic-mail/`
  (`index.md`, `source.md`, `modding.md`, `guide.md`, `notes/`).
- Repo @ `6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47`, key files:
  - `repo/Mod.cs` - registers both systems into `GameSimulation` (no Harmony).
  - `repo/Systems/MagicMailSystem.cs` - periodic scanner: `GetUpdateInterval` /
    `GetUpdateOffset`, `Resources` buffer top-up + overflow clamp helpers.
  - `repo/Systems/MailCapacitySystem.cs` - one-shot, self-disabling prefab
    multiplier reading immutable `PrefabBase` baselines.
  - `repo/Settings/Settings.cs` - `Apply()` wakes both systems on slider change.
