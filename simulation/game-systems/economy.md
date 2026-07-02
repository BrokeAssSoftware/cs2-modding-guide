---
FrontmatterVersion: 1
DocumentType: Guide
Title: CS2 Economy Systems - Modding Hook Map
Summary: How Cities Skylines II's economy simulation works and the verified-real systems, components, and APIs a mod hooks - written for vno-economy's parallel illicit-finance layer. Anchored to the Market Based Economy decompilation and the live Systems and Components catalog.
Created: 2026-06-20
Updated: 2026-06-20
Owners:
  - codex
Tags:
  - simulation
  - economy
  - vno-economy
References:
  - Label: Market Based Economy - Modding Insights (decompiled hook points)
    Path: ../../../vice-and-order-research/mods/dossiers/market-based-economy/modding.md
  - Label: Market Based Economy - Source Architecture
    Path: ../../../vice-and-order-research/mods/dossiers/market-based-economy/source.md
  - Label: Systems and Components Catalog (wiki snapshot, refreshed 2026-06-20)
    Path: ../../../vice-and-order-research/wiki/systems_and_components_catalog.md
  - Label: vno-economy Module Guide
    Path: ../../../vice-and-order/vno-economy/README.md
---

# CS2 Economy Systems - Modding Hook Map

## Purpose and scope

This document maps how the Cities Skylines II economy simulation actually works at the ECS level and which systems, components, and APIs a mod can hook. It is written specifically for [vno-economy](../../../vice-and-order/vno-economy/README.md), which is a **parallel Illicit Finance Engine** (dirty/clean/escrow balances, laundering, heat metrics, the Identity Triangle) layered alongside the vanilla economy via `IFinanceService`/`IIdentityService` and `TransactionEvent`/`HeatChangedEvent`. vno-economy **observes and hooks** the vanilla economy - it does not replace it - so the goal here is to know which vanilla signals to read, which surfaces are safe to patch, and which are moving targets.

The verified-real type names below come from the [Market Based Economy decompilation](../../../vice-and-order-research/mods/dossiers/market-based-economy/modding.md) (MBE - the closest existing analog to a vno-economy override) cross-checked against the [Systems and Components catalog](../../../vice-and-order-research/wiki/systems_and_components_catalog.md). Always re-verify names against the catalog snapshot before relying on them in code.

## Currency caveat - the economy is a moving target

CS2's economy/demand has been reworked repeatedly and the work is ongoing under Iceflake Studios (which took over from Colossal Order at the start of 2026). Relevant rebalances since the prior research baseline (1.5.2f1):

- 1.5.6f1 - office-demand rework (phase 1); zone toggling; citizens-stuck-in-crime-scenario fixes.
- 1.5.7f1 - large economy/demand rebalance (office overhaul, mixed housing, education demand); commercial "stuck-robbery" cases reduced.
- 1.5.9f1 - citizen/company electricity-fee calculation fix.

Treat demand calculation and economy parameters as **unstable hook points**: document them, but expect Iceflake to keep changing them. Anchor vno-economy to the most stable surfaces (price reads, tax pre-passes) and isolate the volatile ones behind adapters.

## Verified-real hook points

### 1. Pricing - `EconomyUtils.GetMarketPrice`
The API that price consumers call. The proven least-invasive pattern (MBE `Harmony/HarmonyBridge.cs`) is a **Harmony postfix** on the getter that swaps in a managed multiplier - vanilla callers keep working, which simplifies compatibility.

- **vno-economy use:** read prices for laundering math and illicit-margin calculations **without rewriting them**. A read-only postfix or a plain read avoids fighting other economy mods.

### 2. Economy parameters - `EconomyParameterData`
A global parameter block; MBE's `WageAdjustmentSystem` rewrites it each frame from labor metrics.

- **Stability: UNSTABLE.** This is exactly the surface CS2's demand/economy reworks touch. If vno-economy must influence parameters, do it through a narrow adapter so a parameter-shape change in a future patch is a one-file fix.

### 3. Workforce and companies
How staffing and production are read/modeled (catalog namespace `Game.Companies.*`):

- `WorkProviderSystem.OnUpdate` - Harmony target in MBE; `WorkProvider.m_MaxWorkers` / the `WorkProvider` buffer is where staffing floors are enforced.
- `Employee` buffer and `CountHouseholdDataSystem` - shared buffers other workforce mods also touch (load-order sensitivity; see MBE's documented soft conflict with Realistic JobSearch).
- `CompanyProductionTracker` - production accounting.
- Other relevant components: `Game.Companies.{ResourceBuyer, FreeWorkplaces, TradeCost, ServiceAvailable}`.

- **vno-economy use:** read company profit/production/trade signals to drive illicit-revenue and front-business modeling; avoid writing to shared workforce buffers unless necessary, and document load order if you do.

### 4. Taxation - `TaxPayer` + pre-vanilla pass
MBE's optional `CompanyProfitAdjustmentSystem` iterates tax payers, derives profit-per-day from employee buffers and process data, and rewrites the `TaxPayer` component **before** the vanilla taxation pass - effectively converting turnover-based tax to profit-based.

- **Pattern:** "insert a system before the vanilla economy pass, mutate the component it will read." This is the canonical model for vno-economy's seizure/laundering effects on taxable income.
- MBE gates this behind a setting (`Setting.EnableCompanyTaxAdjustments`) because it is a heavy pass - mirror that opt-in discipline.

### 5. Scheduling and performance - `SimulationUtils.GetUpdateFrame`
MBE staggers production work across **32 slots per day** via `SimulationUtils.GetUpdateFrame` to avoid per-frame spikes while staying deterministic.

- **Required pattern:** any vno-economy system that iterates companies/citizens must stagger with `GetUpdateFrame` (or equivalent phased scheduling) rather than doing full sweeps every tick.

### 6. Config-driven baselines - `RealWorldBaseline.json`
MBE ships `Config/RealWorldBaseline.json` (per-resource `price` / `outputPerWorkerPerDay`, company multipliers, prefab overrides like `Industrial_BioRefinery` / `Commercial_FastFood`, plus a wage ladder). Three initializer systems (resource, company, economy-parameter) load it.

- **Precedent for vno-economy:** validates the YAML/JSON finance-pack approach (`finance_config.yml`, `launder_methods.yml`, etc.) and the "swap presets mid-session via a settings toggle" pattern.

## Reference-mod caveat - the analog is stale

Market Based Economy is the closest existing economy override, but as of 2026-06-20 it is **still pinned to CS2 1.4.\*** (v0.3.1 / modVersion 7, no release since 2025-11-20) while the game is on 1.5.10f1. Its decompiled hook map is still the best available reference, but **re-verify every type/signature against a current 1.5.x decompile or the refreshed catalog before relying on it** - some economy surfaces changed in the 1.5.6/1.5.7 reworks. See the dossier tracker note for the lag flag.

## V&O integration summary

| Vanilla surface | vno-economy interaction | Stability |
| --- | --- | --- |
| `EconomyUtils.GetMarketPrice` | Read prices for laundering/margin math (postfix or read) | Stable |
| `TaxPayer` (pre-vanilla pass) | Adjust taxable income for seizures/laundering; gate behind setting | Moderate |
| `WorkProvider` / `Employee` / `CompanyProductionTracker` | Read staffing/production for front-business modeling | Moderate (shared buffers) |
| `EconomyParameterData` | Influence only via narrow adapter | UNSTABLE |
| `SimulationUtils.GetUpdateFrame` | Required staggering for all economy systems | Stable pattern |
| `RealWorldBaseline.json`-style config | Precedent for vno-economy finance packs | n/a (pattern) |

## Verification

- Cross-check every `Game.*` / `EconomyUtils` / `*System` name against [systems_and_components_catalog.md](../../../vice-and-order-research/wiki/systems_and_components_catalog.md) (refreshed 2026-06-20) before coding.
- When MBE ships a 1.5.x-targeted release, re-run the dossier's Stage 5+ load-order and profiling TODOs and update this doc's hook stability ratings.
- Treat any system named in the "UNSTABLE" rows as requiring a re-check on each CS2 patch.
