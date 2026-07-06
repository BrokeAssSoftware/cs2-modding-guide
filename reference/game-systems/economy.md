---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Reference: CS2 Economy Systems"
Summary: Lookup table of the Cities Skylines II economy systems, components, and APIs a code mod reads or hooks - price getters, workforce/company staffing, taxation, and scheduling - with source-verified hook points from real mods.
diataxis: reference
source_version: "~1.4.x (market-based-economy@b83f196; 2025-10-30, oldest pin in corpus)"
last_reverified: "2026-07-04"
status: source-verified
canonical_mods:
  - market-based-economy@b83f196a36bc74388accebdeb7c81f0f35dbab37
technique_applicability: [economy, simulation]
Created: 2026-07-03
Updated: 2026-07-03
Owners:
  - codex
References:
  - Label: Technique Index
    Path: ../../technique-index.md
  - Label: "Explanation: System Replacement"
    Path: ../../explanation/system-replacement.md
  - Label: "Case study: Realistic Path Finding"
    Path: ../../case-studies/realistic-path-finding.md
---

# Reference: CS2 Economy Systems

> **Reference - patch-sensitive.** Current live game: **1.6.0f1 "Summer Solstice"** (2026-06-22).
> Source below is cited from market-based-economy at pin `b83f196` (2025-10-30) - the **oldest
> pin in the corpus, ~1.4.x-era** - so these hook points are the most likely to have drifted and
> are NOT re-verified against 1.6.0f1 (`last_reverified: 2026-07-04`, static source only).
> CS2's economy and demand model have been reworked repeatedly and remain under active
> change. Treat every type, method signature, and parameter shape below as a moving target
> and re-verify on each game patch. Prefer the official CS2 wiki's Systems and Components
> catalog for the authoritative live type list.

Dry lookup of the vanilla economy surfaces a code mod reads or hooks. Every claim tagged
**Verified** cites a real mod reading or writing that surface at its pinned commit. Claims a
dossier could not confirm are tagged **Needs Verification** and must not be treated as fact.

Verified hook points are cited to **Market Based Economy (MBE)** at pinned commit
`b83f196a36bc74388accebdeb7c81f0f35dbab37` (git tag `0.3.1`, author Adwozo, Paradox ModId
122923). MBE is a Harmony + ECS economy override - the closest existing analog to a from-scratch
economy layer - so its hook map is the best available witness to which vanilla surfaces exist and
are patchable. Re-verify any cite with:

```
git -C ../vice-and-order-research/mods/dossiers/market-based-economy/repo \
    show b83f196:<path>
```

**Staleness caveat:** MBE's published build (userModVersion 0.3.1 / modVersion 7) targets an
older game version than current 1.6.x. Its hook map is source-real but some price/wage/demand
surfaces changed in the 1.5.6/1.5.7 economy reworks - re-verify signatures against a current
decompile before relying on them in code.

## Pricing - `Game.Economy.EconomyUtils` price getters (Verified)

Price consumers across the simulation call static getters on `EconomyUtils`. These are the
most stable and least invasive economy hook: a Harmony **postfix** on the getter can rescale
the returned price while every vanilla caller keeps working. MBE applies exactly five postfixes
(four price getters + one workforce system) manually in `HarmonyBridge.ApplyAll`.

| Vanilla method | Signature MBE patches | Cite (at b83f196) |
| --- | --- | --- |
| `EconomyUtils.GetMarketPrice` | `(Resource, ResourcePrefabs, ref ComponentLookup<ResourceData>)` | `repo/Harmony/HarmonyBridge.cs#L44-L60` (target L47) |
| `EconomyUtils.GetMarketPrice` | `(Resource, ResourcePrefabs, EntityManager)` | `repo/Harmony/HarmonyBridge.cs#L69-L92` (target L74) |
| `EconomyUtils.GetIndustrialPrice` | `(Resource, ResourcePrefabs, ref ComponentLookup<ResourceData>)` | `repo/Harmony/HarmonyBridge.cs#L96-L137` (target L101) |
| `EconomyUtils.GetServicePrice` | `(Resource, ResourcePrefabs, ref ComponentLookup<ResourceData>)` | `repo/Harmony/HarmonyBridge.cs#L96-L137` (target L106) |

The postfix body just rewrites `ref float __result` from a managed multiplier, e.g.
`MarketPricePostfix` at `repo/Harmony/HarmonyBridge.cs#L161-L170`. Key resource types in these
signatures - `Game.Economy.Resource`, `Game.Prefabs.ResourcePrefabs`, `Game.Prefabs.ResourceData` -
are confirmed by these AccessTools lookups resolving at that commit.

- **Pattern:** read-only postfix (or a plain `EconomyUtils.GetMarketPrice` call) to observe or
  rescale prices without rewriting them - avoids fighting other economy mods.
- **Needs Verification:** a `ResourceSeller` component (paired with `ResourceBuyer`) - not found
  in the MBE snapshot; do not assume it exists under that name.

## Workforce and companies (`Game.Companies.*`) (Verified)

How staffing and production are read and modeled. MBE's fifth Harmony postfix targets a workforce
system; the rest is plain ECS reads/writes.

| Surface | Kind | What it holds | Cite (at b83f196) |
| --- | --- | --- | --- |
| `Game.Simulation.WorkProviderSystem.OnUpdate` | Harmony postfix target | staffing recompute pass | `repo/Harmony/HarmonyBridge.cs#L139-L157` (target L142), body L222 |
| `Game.Companies.WorkProvider` (`m_MaxWorkers`) | component (ReadWrite) | per-workplace staffing cap; MBE raises it to a prefab-derived floor | `repo/Economy/WorkforceUtilizationManager.cs#L47,L100-L107` |
| `Game.Companies.Employee` | DynamicBuffer (ReadOnly) | staffed workers, iterated for utilization/profit | `repo/Economy/WorkforceUtilizationManager.cs#L100`; `repo/Economy/CompanyProfitAdjustmentSystem.cs#L58` |
| `Game.Companies.ServiceAvailable` | component (used as query `Exclude`) | marks service companies (MBE's product query excludes them) | `repo/Economy/MarketProductSystem.cs#L46` |
| `Game.Simulation.CompanyProductionTracker` | singleton/manager | production accounting read each slot | `repo/Economy/MarketProductSystem.cs#L67` |
| `Workplaces` via `GetFreeWorkplaces()` | struct returned by a workplace system | free jobs per education level | `repo/Economy/LaborMarketManager.cs#L128,L253,L316` |

- **Compatibility:** `WorkProvider`, `Employee`, and `CountHouseholdDataSystem` are shared buffers
  other workforce mods also touch (MBE documents a soft conflict with Realistic JobSearch). Writing
  to them is load-order sensitive - document ordering if you do.
- **Needs Verification:** `Game.Companies.FreeWorkplaces` as a *component* - MBE reads free
  workplaces through a system method returning a `Workplaces` struct, not a component of that name.
  Also `TradeCost` - named in the legacy draft but not present in the MBE snapshot.

## Wages - `Game.Economy.EconomyParameterData` (Verified surface, UNSTABLE)

A global parameter singleton. MBE's `WageAdjustmentSystem` queries it `ReadWrite` and rewrites
wage-related fields each update from labor metrics, scheduled **before** the vanilla wage pass.

- Query + write: `repo/Economy/WageAdjustmentSystem.cs#L22,L32-L45`
  (`GetEntityQuery(ComponentType.ReadWrite<EconomyParameterData>())`).
- Ordering: `updateSystem.UpdateBefore<WageAdjustmentSystem, PayWageSystem>` at `repo/Mod.cs#L54`
  (vanilla `Game.Simulation.PayWageSystem` confirmed as the ordering target).
- **Stability: UNSTABLE.** This is exactly the surface CS2's demand/economy reworks touch. Isolate
  it behind a narrow adapter so a parameter-shape change is a one-file fix. The individual field
  layout of `EconomyParameterData` is **Needs Verification** per patch.

## Taxation - `Game.Simulation.TaxPayer` + pre-vanilla pass (Verified)

MBE's optional `CompanyProfitAdjustmentSystem` iterates entities with `Employee` +
`TaxPayer`, derives profit-per-day from employee buffers and process data, then rewrites
`TaxPayer` **before** the vanilla taxation pass - converting turnover-based tax to profit-based.

| Surface | Kind | Cite (at b83f196) |
| --- | --- | --- |
| `Game.Simulation.TaxPayer` | component (ReadWrite) | `repo/Economy/CompanyProfitAdjustmentSystem.cs#L59` |
| `Game.Companies.Employee` | buffer (ReadOnly) | `repo/Economy/CompanyProfitAdjustmentSystem.cs#L58` |
| `Game.Prefabs.IndustrialProcessData` | lookup (ReadOnly) | `repo/Economy/CompanyProfitAdjustmentSystem.cs#L34` |
| `Game.Simulation.TaxSystem` | vanilla system (ordering target) | `repo/Economy/CompanyProfitAdjustmentSystem.cs#L41,L80` |
| `TaxSystem.GetModifiedCommercialTaxRate(resource, taxRates, district, districtModifiers)` | static, applies per-district modifiers to tax | `repo/Economy/CompanyProfitAdjustmentSystem.cs#L326` |

- **Pattern:** "insert a system before the vanilla economy pass, mutate the component it will
  read." Ordering: `UpdateBefore<CompanyProfitAdjustmentSystem, TaxSystem>` at `repo/Mod.cs#L56`.
- MBE gates this behind a setting (`Setting.EnableCompanyTaxAdjustments`,
  `repo/Setting.cs#L64-L69`) because it is a heavy per-company pass - mirror that opt-in discipline.
- Tax rates are district-aware: see [districts-policies.md](districts-policies.md) for
  `CurrentDistrict` / `DistrictModifier`.

## Scheduling - `Game.Simulation.SimulationUtils.GetUpdateFrame` (Verified)

Any economy system that sweeps all companies must stagger, not sweep every tick. MBE's
`MarketProductSystem` spreads production work across **32 slots per day**.

- `const int kUpdatesPerDay = 32` and
  `SimulationUtils.GetUpdateFrame(m_SimulationSystem.frameIndex, kUpdatesPerDay, 16)` -
  `repo/Economy/MarketProductSystem.cs#L24,L60`. Companies carry an `UpdateFrame` component that
  the query filters on (`repo/Economy/MarketProductSystem.cs#L44,L82`).
- Ordering: `UpdateBefore<MarketProductSystem, ResourceExporterSystem>` at `repo/Mod.cs#L55`
  (vanilla `Game.Simulation.ResourceExporterSystem`).
- See [explanation/multi-phase-scheduling.md](../../explanation/multi-phase-scheduling.md) for the
  general phased-scheduling pattern.

## Config-driven baselines (pattern, Verified in MBE)

MBE ships `Config/RealWorldBaseline.json` (per-resource `price` / `outputPerWorkerPerDay`, company
multipliers, prefab overrides, a wage ladder) loaded by three initializer systems (resource,
company, economy-parameter) via `repo/Economy/RealWorldBaselineFeature.cs#L56-L165`. This is a
precedent for shipping swappable economy presets, not a vanilla surface.

## Stability summary

| Vanilla surface | Typical interaction | Stability |
| --- | --- | --- |
| `EconomyUtils.GetMarketPrice` / `GetIndustrialPrice` / `GetServicePrice` | Read/rescale via postfix or plain read | Stable |
| `TaxPayer` (pre-vanilla pass) | Rewrite before `TaxSystem`; gate behind a setting | Moderate |
| `WorkProvider` / `Employee` / `CompanyProductionTracker` | Read staffing/production; write is load-order sensitive | Moderate (shared buffers) |
| `EconomyParameterData` | Influence only via a narrow adapter | UNSTABLE |
| `SimulationUtils.GetUpdateFrame` | Required staggering for all economy sweeps | Stable pattern |

## Needs Verification (not source-confirmed vanilla behavior)

- `ResourceSeller`, `Game.Companies.FreeWorkplaces` (as a component), `TradeCost` - named in the
  legacy draft, not found in the MBE snapshot.
- The internal field layout of `EconomyParameterData` and the demand/RCI calculation parameters -
  reworked repeatedly; re-derive per patch.
- Exact 1.5.x/1.6.x rebalance behavior (office demand, mixed housing, electricity-fee calc) - patch
  notes, not source-verified surfaces.
- Whether the four price getters retain these exact signatures on the current game build.

## Related

- [technique-index.md](../../technique-index.md) - family C (Harmony postfix on economy/price
  getters), family M (disable/replace vanilla system), family B (ECS system replacement via ordering).
- [explanation/system-replacement.md](../../explanation/system-replacement.md) and
  [explanation/system-scheduling.md](../../explanation/system-scheduling.md) - the "why" of insert-before-vanilla passes.
- [how-to/recipes/ecs-system-replacement-ordering.md](../../how-to/recipes/ecs-system-replacement-ordering.md) -
  the ordering recipe.
- [how-to/recipes/settings-patterns.md](../../how-to/recipes/settings-patterns.md) - gating heavy
  passes behind a setting.
- [citizens-households.md](citizens-households.md), [districts-policies.md](districts-policies.md) -
  adjacent game systems.
