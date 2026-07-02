---
FrontmatterVersion: 1
DocumentType: Guide
Title: CS2 Citizens and Households - Modding Hook Map
Summary: How Cities Skylines II models citizens, households, needs, and behavior at the ECS level, and the verified-real components a mod reads - written for vno-vice (behavior signals) and vno-health (mortality/health stress). Anchored to the live Systems and Components catalog.
Created: 2026-06-20
Updated: 2026-06-20
Owners:
  - codex
Tags:
  - simulation
  - citizens
  - households
  - vno-vice
  - vno-health
References:
  - Label: Systems and Components Catalog (wiki snapshot, refreshed 2026-06-20)
    Path: ../../../vice-and-order-research/wiki/systems_and_components_catalog.md
  - Label: vno-vice Module Guide
    Path: ../../../vice-and-order/vno-vice/README.md
  - Label: vno-health Module Guide
    Path: ../../../vice-and-order/vno-health/README.md
---

# CS2 Citizens and Households - Modding Hook Map

## Purpose and scope

How CS2 models citizens, households, needs, and behavior, and which components a mod reads. Written for [vno-vice](../../../vice-and-order/vno-vice/README.md) (which reads behavior/need signals to model recruitment, territory presence, and illicit participation) and [vno-health](../../../vice-and-order/vno-health/README.md) (which models mortality and health stress). Type names are from the [Systems and Components catalog](../../../vice-and-order-research/wiki/systems_and_components_catalog.md) (refreshed 2026-06-20).

## Core system

`Game.Simulation.CitizenBehaviorSystem` drives per-citizen behavior and schedules `CitizenAITickJob`, which iterates citizens carrying `Game.Citizens.Citizen` + `Game.Citizens.CurrentBuilding` + `Game.Citizens.HouseholdMember` (and excludes those with `TravelPurpose`, `Game.Companies.ResourceBuyer`, `Game.Common.Deleted`, `Game.Tools.Temp`). It decides movement, leisure, shopping, sleep, mailing, and move-away behavior.

## Key components (catalog namespace `Game.Citizens.*`)

- `Citizen` - the core citizen; carries flags (e.g. `CitizenFlags.Commuter`) and age (`CitizenAge.Child/Adult/Elderly`).
- `Household`, `HouseholdMember`, `HouseholdNeed`, `CommuterHousehold` - household grouping and demand/need signals.
- `Worker`, `Student` - employment/education roles (removed when a citizen is moving away).
- `Leisure`, `Purpose` / `TravelPurpose`, `TripNeeded` - what a citizen is currently doing and where they intend to go (`GoToOutsideConnection`, `MovingAway`, etc.).
- `HealthProblem` - health/illness state (the key vno-health hook; see death model below).
- `ResourceBought`, `CarKeeper` - consumption and vehicle ownership signals.
- Related building components: `Game.Buildings.{Building, CitizenPresence, PropertyRenter}`.

(Note: the catalog preserves upstream source typos such as `Puspose` for `Purpose` - reference exact spellings from the snapshot when binding.)

## Behavior change since the research baseline

- **1.5.4f1 death-calc overhaul** - death is now evaluated **16 times per citizen per day** (up from 4), factors time-of-day, and Easy Mode now permits old-age deaths (previously ~80% never died of old age). This directly reshapes vno-health mortality/health-stress modeling: any assumptions about death cadence or `HealthProblem` -> death conversion from the prior baseline are stale.
- Bicycle trips reduced ~80% (1.5.4f1) and Urban Cycling policy boosted 20% -> 50% (1.5.7f1) - affects `TravelPurpose`/`TripNeeded` mode distributions vno-vice may read for movement modeling.
- 1.5.6f1 fixed citizens stuck in home-seeking, city-exit, and **crime scenarios** - relevant to how vno-order/vno-vice observe crime-related citizen states.

## V&O application

- **vno-vice:** read `HouseholdNeed`, `Leisure`, `Purpose`/`TravelPurpose`, and employment (`Worker`/`Student`) as behavior signals for recruitment likelihood and territory presence. Do not mutate citizen AI; observe and layer.
- **vno-health:** treat `HealthProblem` + the post-1.5.4 death model as the mortality hook. Re-derive any rate assumptions against the current model rather than the prior baseline.
- Use phased scheduling (see the economy doc's `SimulationUtils.GetUpdateFrame` note) for any per-citizen sweep.

## Verification

- Re-verify every component name and the `CitizenBehaviorSystem`/`CitizenAITickJob` shape against [systems_and_components_catalog.md](../../../vice-and-order-research/wiki/systems_and_components_catalog.md) before coding.
- Re-validate mortality assumptions against the live 1.5.x death model (16x/day) - flagged as patch-sensitive.
