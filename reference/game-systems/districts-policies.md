---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Reference: CS2 Districts and Policies"
Summary: Lookup of the Cities Skylines II district ECS surfaces a code mod reads - Game.Areas district membership and modifier buffers, the AreaUtils.ApplyModifier read pattern - plus an honest gap on the policy-application system, with source-verified names from real mods.
diataxis: reference
source_version: "~1.4.x (market-based-economy@b83f196; date-pinned, static source only)"
last_reverified: "2026-07-04"
status: source-verified
canonical_mods:
  - market-based-economy@b83f196a36bc74388accebdeb7c81f0f35dbab37
  - realistic-path-finding@50645fa6a078181365e36a42e2e27b96699bf02a
  - time2work-realistic-trips@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39
technique_applicability: [simulation, core]
Created: 2026-07-03
Updated: 2026-07-03
Owners:
  - codex
References:
  - Label: Technique Index
    Path: ../../technique-index.md
  - Label: "Explanation: System Replacement"
    Path: ../../explanation/system-replacement.md
---

# Reference: CS2 Districts and Policies

> **Reference - patch-sensitive: verify against the current game version; last checked CS2 1.6.x.**
> Zoning and policy surfaces shift across patches. Re-verify every name on each game patch, and
> prefer the official CS2 wiki's Systems and Components catalog and Policies catalogue for the
> authoritative live lists.

Dry lookup of the vanilla district and policy surfaces a code mod reads or applies. Claims tagged
**Verified** cite a real mod reading that surface at its pinned commit; claims a dossier could not
confirm are tagged **Needs Verification**. Districts are well-witnessed by our dossiers; the
**policy-application system is not** - that section is an honest gap, not fact.

Verified surfaces are cited across three mods (re-verify with
`git -C ../vice-and-order-research/mods/dossiers/<slug>/repo show <pin>:<path>`):

- **Market Based Economy (MBE)** at pin `b83f196a36bc74388accebdeb7c81f0f35dbab37` (ModId 122923).
- **Realistic Path Finding (RPF)** at pin `50645fa6a078181365e36a42e2e27b96699bf02a` (ModId 121226).
- **Time2Work / Realistic Trips (T2W)** at pin `d42921ffb1f6bbbf2c2b086bb17727bf0f637c39` (ModId 77171).

## Districts (`Game.Areas.*`) (Verified)

Three independent mods read the same district surfaces, so these are well-confirmed. A building or
property resolves to its district through `CurrentDistrict`, and per-district numeric effects live in
a `DistrictModifier` buffer on the district entity.

| Surface | Kind | What it is | Cite |
| --- | --- | --- | --- |
| `Game.Areas.CurrentDistrict` | component | on a building/property; field `m_District` (an `Entity`) points to the district | MBE `repo/Economy/CompanyProfitAdjustmentSystem.cs#L324-L325`; RPF `repo/RealisticPathFinding/Systems/RPFTripNeededSystem.cs#L430,L1497`; T2W `repo/NightShift/Systems/Time2WorkCitizenBehaviorSystem.cs#L447,L553` |
| `Game.Areas.DistrictModifier` | DynamicBuffer | per-district effect entries, held on the district entity | MBE `.../CompanyProfitAdjustmentSystem.cs#L29,L288`; T2W `.../Time2WorkCitizenBehaviorSystem.cs#L446,L551` |
| `Game.Areas.DistrictModifierType` | enum | which effect a modifier entry targets (e.g. `CarReserveProbability`, `BikeProbability`) | T2W `.../Time2WorkCitizenBehaviorSystem.cs#L554`; `.../Time2WorkLeisureSystem.cs#L1648` |
| `Game.Areas.Node` | DynamicBuffer | district/area boundary geometry | RPF `repo/RealisticPathFinding/Systems/RPFResidentAISystem.cs#L261` |

### Reading a district effect - `AreaUtils.ApplyModifier` (Verified)

The canonical way to apply a district's modifier to a value is `AreaUtils.ApplyModifier`, which
folds the relevant `DistrictModifier` entries of a type into a running value:

```
AreaUtils.ApplyModifier(ref num, bufferData, DistrictModifierType.CarReserveProbability);
```

Source: T2W `repo/NightShift/Systems/Time2WorkCitizenBehaviorSystem.cs#L554` (and
`.../Time2WorkLeisureSystem.cs#L1648` for `BikeProbability`). The full lookup chain a mod uses is:
resolve the entity's `CurrentDistrict` -> take `m_District` -> `TryGetBuffer<DistrictModifier>` on
that district entity -> `AreaUtils.ApplyModifier(...)` (T2W `.../Time2WorkCitizenBehaviorSystem.cs#L553-L554`).
RPF wraps the same resolve step in a helper `FindDistrict(building)` that reads `CurrentDistrict.m_District`
(`repo/RealisticPathFinding/Systems/RPFTripNeededSystem.cs#L1493-L1497`).

### District-aware taxation (Verified)

Tax rates are district-aware: MBE calls the vanilla
`TaxSystem.GetModifiedCommercialTaxRate(resource, taxRates, district, districtModifiers)`, passing the
`CurrentDistrict.m_District` entity and the `DistrictModifier` buffer
(`repo/Economy/CompanyProfitAdjustmentSystem.cs#L326`). See [economy.md](economy.md#taxation---gamesimulationtaxpayer--pre-vanilla-pass).

## Policies (NOT source-verified here - honest gap)

**None of our current dossiers read or apply the policy-application system.** A repo-wide search
across MBE, RPF, and T2W at their pinned commits finds no `PolicySystem`, `Game.Policies.*`,
`ActivePolicy`, or policy-application code. Do not guess these names.

What is known:
- Per-district *numeric effects* apply through the verified `DistrictModifier` buffer +
  `AreaUtils.ApplyModifier` path above. District-level policies are widely understood to feed those
  modifier entries, but the specific system that reads active policies and writes the modifier buffer
  is **Needs Verification** - no dossier confirms it.
- The authoritative live list of policies (city / district / building level, their milestones and
  effects) is the official CS2 wiki Policies catalogue - link out to the wiki rather than
  duplicating it here.

## Needs Verification (not source-confirmed vanilla behavior)

- The district entity's own components (a `District` / `DistrictData` component). Only the pointer
  from a building/property (`CurrentDistrict.m_District`) and the `DistrictModifier` buffer are
  confirmed - no dossier reads a `District` component directly.
- The policy-application system (`*PolicySystem`), the component holding a district's active policy
  set, and how a policy maps to `DistrictModifier` entries.
- How area services (police / health / etc.) compute per-district coverage.
- The full policy list and any per-patch policy/zoning changes (e.g. Urban Cycling effect tuning,
  road-side zone toggling) - these are patch notes / wiki data, not source-verified surfaces.

Derive the unverified names from a current decompile or a catalog refresh, record findings, and
promote them from this list only when a mod is confirmed reading them.

## Related

- [technique-index.md](../../technique-index.md) - family M (disable/replace vanilla system),
  family J (one-shot non-stacking prefab multiplier).
- [economy.md](economy.md) - district-aware tax rates; [citizens-households.md](citizens-households.md) -
  citizens resolve to a district via `CurrentDistrict`.
- [explanation/system-replacement.md](../../explanation/system-replacement.md) - the pattern for
  layering on vanilla simulation without owning it.
