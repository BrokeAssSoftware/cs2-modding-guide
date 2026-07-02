---
FrontmatterVersion: 1
DocumentType: Guide
Title: CS2 Districts and Policies - Modding Hook Map
Summary: How Cities Skylines II models districts, area services, and policies, and the verified-real surfaces a mod reads/applies - written for vno-order and vno-governance. Anchored to the live Policies catalogue and Systems and Components catalog snapshots.
Created: 2026-06-20
Updated: 2026-06-20
Owners:
  - codex
Tags:
  - simulation
  - districts
  - policies
  - vno-order
  - vno-governance
References:
  - Label: Policies Catalogue (wiki snapshot, refreshed 2026-06-20)
    Path: ../../../vice-and-order-research/wiki/policies_catalogue.md
  - Label: Systems and Components Catalog (wiki snapshot, refreshed 2026-06-20)
    Path: ../../../vice-and-order-research/wiki/systems_and_components_catalog.md
  - Label: vno-order Module Guide
    Path: ../../../vice-and-order/vno-order/README.md
  - Label: vno-governance Module Guide
    Path: ../../../vice-and-order/vno-governance/README.md
---

# CS2 Districts and Policies - Modding Hook Map

## Purpose and scope

How CS2 models districts, area services, and policies, and which surfaces a mod reads or applies. Written for [vno-order](../../../vice-and-order/vno-order/README.md) (policing/justice service coverage) and [vno-governance](../../../vice-and-order/vno-governance/README.md) (policy lifecycle, voting, corruption hooks).

## Status of source coverage (honest gap)

The current [Systems and Components catalog](../../../vice-and-order-research/wiki/systems_and_components_catalog.md) snapshot is **thin on the precise district-entity and policy-application system names** - it surfaces area/zoning signals like `Game.Companies.ServiceAvailable` and `Game.Buildings.PropertyRenter`, but not a complete district/policy system map. **Do not guess these names.** The authoritative live list of policies is the [Policies catalogue](../../../vice-and-order-research/wiki/policies_catalogue.md); the district-entity and `PolicySystem`-style application names should be derived from a fresh decompile or a future catalog refresh before coding. This doc records what is verified and flags what must be confirmed.

## Policies (verified via the catalogue)

The Policies catalogue lists the live policy set: city-level, district-level, and building-level policies with their milestones and effects. As of the 2026-06-20 refresh the **on-wiki policy list is unchanged** from the prior capture (same 6 city / 7 district / 1 building policies), but two game-level changes are NOT yet reflected on the wiki page:

- 1.5.7f1 - "Urban Cycling Initiative" policy effect boosted 20% -> 50% bicycle usage.
- 1.5.6f1 - new zone-toggling feature (road-side zoning control).

## Area-service and zoning signals (verified)

- `Game.Companies.ServiceAvailable` - service availability signal for area-service coverage.
- `Game.Buildings.PropertyRenter` / `Game.Buildings.Building` - occupancy/zoning context.

## To confirm before coding (do not guess)

- District entity component(s) and the district membership/lookup system.
- The policy-application system (the `*PolicySystem` that reads active policies and applies effects) and the component holding a district's active policy set.
- How area services (police/health/etc.) compute district coverage.

Derive these from a current decompile or the next catalog refresh; record findings back into the Systems and Components catalog snapshot and update this doc.

## V&O application

- **vno-order:** model policing/justice **service coverage** per district by reading area-service availability; layer enforcement pressure rather than replacing service systems.
- **vno-governance:** drive the policy lifecycle (proposal -> vote -> apply) against the verified policy set; corruption/bribery hooks should influence policy application through the confirmed `*PolicySystem` (once its name is verified), not by mutating policy data blindly.
- Watch the Urban Cycling effect change and zone-toggling feature as examples of how policy effects and zoning shift across patches.

## Verification

- Treat the policy list as accurate per the catalogue snapshot; treat the district/policy **system names** as unverified until confirmed against a live decompile.
- Re-check after any CS2 patch that touches zoning or policies.
