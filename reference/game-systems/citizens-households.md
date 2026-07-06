---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Reference: CS2 Citizens and Households"
Summary: Lookup table of the Cities Skylines II citizen/household ECS model - the Game.Citizens components, the vanilla simulation systems that drive behavior and death, and the query patterns a code mod reads - with source-verified names from real mods.
diataxis: reference
source_version: "1.6.0f1 (realistic-path-finding@50645fa; date-pinned, static source only)"
last_reverified: "2026-07-04"
status: source-verified
canonical_mods:
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
  - Label: "Case study: Realistic Path Finding"
    Path: ../../case-studies/realistic-path-finding.md
  - Label: "Explanation: System Replacement"
    Path: ../../explanation/system-replacement.md
---

# Reference: CS2 Citizens and Households

> **Reference - patch-sensitive: verify against the current game version; last checked CS2 1.6.x.**
> Citizen behavior, trip demand, and the death model have all changed across recent patches.
> Re-verify every component name and system shape on each game patch; prefer the official CS2
> wiki's Systems and Components catalog for the authoritative live type list.

Dry lookup of the vanilla citizen/household ECS model a code mod reads. Every claim tagged
**Verified** cites a real mod reading or writing that surface at its pinned commit. Claims a
dossier could not confirm are tagged **Needs Verification**.

Verified surfaces are cited to two mods by ruzbeh0, both of which clone vanilla citizen systems
(so they name the real vanilla types they read):

- **Realistic Path Finding (RPF)** at pin `50645fa6a078181365e36a42e2e27b96699bf02a` (ModId 121226).
- **Time2Work / Realistic Trips (T2W)** at pin `d42921ffb1f6bbbf2c2b086bb17727bf0f637c39` (ModId 77171).

Re-verify a cite with:

```
git -C ../vice-and-order-research/mods/dossiers/<slug>/repo show <pin>:<path>
```

## Vanilla simulation systems (Verified - confirmed by mods disabling them)

A mod that replaces a vanilla behavior loop first disables the vanilla system by name. These calls
confirm the systems exist under `Game.Simulation.*` at the pinned commits.

| Vanilla system | Role | Confirmed by (disables it) |
| --- | --- | --- |
| `CitizenBehaviorSystem` | per-citizen behavior loop (movement, leisure, shopping, sleep) | T2W `repo/NightShift/Mod.cs#L94` |
| `CitizenTravelPurposeSystem` | assigns `TravelPurpose` | T2W `repo/NightShift/Mod.cs#L95` |
| `WorkerSystem` | worker role/employment updates | T2W `repo/NightShift/Mod.cs#L96` |
| `LeisureSystem` | leisure demand | T2W `repo/NightShift/Mod.cs#L97` |
| `StudentSystem` | student role/education | T2W `repo/NightShift/Mod.cs#L98` |
| `TourismSystem`, `TouristSpawnSystem`, `AttractionSystem` | tourism/attraction | T2W `repo/NightShift/Mod.cs#L100-L102` |
| `ResidentAISystem` (+ nested `.Actions`) | resident movement AI decision loop | RPF `repo/RealisticPathFinding/Mod.cs#L54-L55` |
| `TripNeededSystem` | queues `TripNeeded` trips | RPF `repo/RealisticPathFinding/Mod.cs#L56` |
| `ResourceBuyerSystem` | shopping / resource purchase trips | RPF `repo/RealisticPathFinding/Mod.cs#L57` |
| `DeathCheckSystem` | evaluates citizen death (see below) | T2W `repo/NightShift/Mod.cs#L109` |

## Key components (`Game.Citizens.*`) (Verified)

Confirmed by RPF's `RPFResidentAISystem` / `RPFTripNeededSystem` component lookups and queries,
and T2W's `Time2WorkCitizenBehaviorSystem`. Access mode shown is how these mods bind them.

| Component | Kind | What it is | Cite |
| --- | --- | --- | --- |
| `Citizen` | component | core citizen; carries `CitizenAge` (`Child`/`Teen`/`Adult`/`Elderly`) | RPF `repo/RealisticPathFinding/Systems/RPFResidentAISystem.cs#L211`; age enum `.../Systems/BicycleOwnerLimiterSystem.cs#L114-L120` |
| `HouseholdMember` | component | links a citizen to its household | RPF `.../RPFResidentAISystem.cs#L212` |
| `Household` | component | household record | RPF `.../RPFResidentAISystem.cs#L213` |
| `HouseholdCitizen` | DynamicBuffer | citizens in a household | RPF `.../RPFResidentAISystem.cs#L254` |
| `HouseholdNeed` | component | household demand/need signal | RPF `.../RPFResidentAISystem.cs#L223` |
| `HouseholdAnimal` | DynamicBuffer | household pets | RPF `.../RPFResidentAISystem.cs#L253` |
| `HomelessHousehold`, `TouristHousehold` | components | household-kind tags | RPF `.../RPFResidentAISystem.cs#L221-L222` |
| `CurrentBuilding` | component | building the citizen is in | RPF `.../RPFResidentAISystem.cs#L214` |
| `CurrentTransport` | component | vehicle/transport the citizen is using | RPF `.../RPFResidentAISystem.cs#L215` |
| `Worker` | component | employment role and workplace | RPF `.../RPFResidentAISystem.cs#L216` |
| `Student` (`Game.Citizens.Student`) | component | education role | RPF `.../RPFTripNeededSystem.cs#L441` |
| `CarKeeper` | component | car ownership | RPF `.../RPFResidentAISystem.cs#L217` |
| `BicycleOwner` | component | bicycle ownership | RPF `.../RPFResidentAISystem.cs#L218` |
| `HealthProblem` | component | illness/injury/death state (key health hook) | RPF `.../RPFResidentAISystem.cs#L219` |
| `TravelPurpose` | component | what the citizen is currently doing/going to | RPF `.../RPFResidentAISystem.cs#L220` |
| `TripNeeded` | DynamicBuffer | pending trips (ReadWrite by trip systems) | RPF `.../RPFTripNeededSystem.cs#L133,L422` |
| `Leisure` | component | current leisure activity | T2W `repo/NightShift/Systems/Time2WorkCitizenBehaviorSystem.cs` (Game.Citizens lookup) |
| `AttendingMeeting`, `CoordinatedMeeting`, `CoordinatedMeetingAttendee` | components / buffer | scheduled meetups | RPF `.../RPFResidentAISystem.cs#L224-L225`; `.../RPFTripNeededSystem.cs#L450` |
| `MailSender` | component | outgoing-mail behavior | RPF `.../RPFTripNeededSystem.cs#L419` |
| `Criminal` | component | criminal state | RPF `.../RPFTripNeededSystem.cs#L456` |

Related building-side component: `Game.Buildings.CitizenPresence` (read by T2W's behavior clone,
`repo/NightShift/Systems/Time2WorkCitizenBehaviorSystem.cs`).

## Canonical citizen query pattern (Verified)

RPF's trip system shows the standard "iterate active citizens" query - useful as a template for a
per-citizen sweep:

```
GetEntityQuery(
    ComponentType.ReadOnly<Citizen>(),
    ComponentType.ReadOnly<HouseholdMember>(),
    ComponentType.ReadWrite<TripNeeded>(),
    ComponentType.Exclude<TravelPurpose>(),
    ComponentType.ReadOnly<CurrentBuilding>(),
    ComponentType.Exclude<ResourceBuyer>(),
    ComponentType.Exclude<Deleted>(),
    ComponentType.Exclude<Temp>());
```

Source: RPF `repo/RealisticPathFinding/Systems/RPFTripNeededSystem.cs#L133`. Excluding
`Game.Companies.ResourceBuyer`, `Game.Common.Deleted`, and `Game.Tools.Temp` is the common way to
skip shopping-agents and transient/deleted entities. Do not mutate citizen AI; read and layer.

## Death model - `Game.Simulation.DeathCheckSystem` (Verified cadence)

T2W ships `Time2WorkDeathCheckSystem`, a clone of the vanilla `DeathCheckSystem` (which it disables,
`repo/NightShift/Mod.cs#L109`). The clone confirms the current death cadence and hooks:

- `const int kUpdatesPerDay = 16` - death is evaluated **16 times per citizen per day**
  (`repo/NightShift/Systems/Time2WorkDeathCheckSystem.cs#L31`).
- Staggered via `SimulationUtils.GetUpdateFrame(frameIndex, kUpdatesPerDay, 16)`
  (`.../Time2WorkDeathCheckSystem.cs#L88`).
- The death job binds `HealthProblem` **ReadWrite** - `HealthProblem` is the component death is
  written to (`.../Time2WorkDeathCheckSystem.cs#L98`).
- Easy Mode death behavior is gated on a save `FormatTags.EasyModeDeathRateFix` format tag
  (`.../Time2WorkDeathCheckSystem.cs#L170`).

`HealthProblem` + this 16x/day cadence is the mortality hook. Re-derive any rate assumptions
against the current model rather than an older baseline.

## Needs Verification (not source-confirmed vanilla behavior)

- `CitizenAITickJob` (the internal job scheduled by `CitizenBehaviorSystem`) - the *system* is
  confirmed vanilla, but no dossier confirms the job's name or exact iterated set.
- `CitizenFlags.Commuter` and the `Citizen` flags layout - not exercised by these mods.
- `CommuterHousehold`, `ResourceBought` - named in the legacy draft, not found in these snapshots.
- The `Purpose` vs `Puspose` upstream typo - not confirmed here; check the exact spelling in a live
  decompile before binding.
- Exact vanilla death-rate curve, time-of-day weighting, and old-age probabilities - T2W clones the
  system but its numeric curve is a reconstruction, not proof of the vanilla math.
- Patch-note behavior deltas (bicycle-trip reductions, Urban Cycling policy boosts) - patch notes,
  not source-verified surfaces.

## Related

- [technique-index.md](../../technique-index.md) - family M (disable/replace vanilla system),
  family B (ECS system replacement via ordering), family S (Burst IJobChunk jobs).
- [case-studies/realistic-path-finding.md](../../case-studies/realistic-path-finding.md) - a full
  tour of cloning the resident AI loop.
- [explanation/system-replacement.md](../../explanation/system-replacement.md) - disabling a vanilla
  system and inserting a clone.
- [how-to/recipes/ecs-system-replacement-ordering.md](../../how-to/recipes/ecs-system-replacement-ordering.md) -
  the ordering recipe.
- [economy.md](economy.md) (`Worker`/`Employee` employment side),
  [districts-policies.md](districts-policies.md) (district a citizen resolves to).
