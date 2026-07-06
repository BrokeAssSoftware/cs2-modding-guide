---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Case study: Elections - Realistic Trips Module"
case_study: elections-rt-module
mod: "Elections (Realistic Trips Module)"
dossier: ../../vice-and-order-research/mods/dossiers/elections-rt-module/
repo_commit: 36c25afcda67c80ce75ba68d423fe8238499435a
source_version: "~1.5.x (elections-rt-module@36c25af; date-pinned, static source only)"
last_reverified: "2026-07-05"
diataxis: explanation
techniques: [AP, G, E, Q]
technique_applicability: [platform, simulation, ui]
status: source-verified
Created: 2026-07-05
Updated: 2026-07-05
Owners:
  - codex
Summary: How the Elections mod extends a host mod's private simulation entirely through reflection - a reflection bridge to a foreign mod's private API, an integer trip-type protocol over a shared RequestSocialTrip entry point, physical mayor relocation via a hand-built vanilla Event archetype, a versioned hand-written ISerializable fat singleton, and a NameSystem reflection fallback chain - as the canonical inter-mod-dependency case study.
---

# Elections (Realistic Trips Module) - case study

> A "module mod" that has no build reference to the host it extends: it reaches
> into Realistic Trips / Time2Work's *private* simulation purely through
> reflection, negotiates capability at runtime, and degrades to a logged no-op
> when the host is absent - the canonical worked example of a hard inter-mod
> dependency expressed without a compile-time link.

All code claims below are cited to `repo/<path>#Lxx` at pinned commit
`36c25afcda67c80ce75ba68d423fe8238499435a` ("version 3.1"), surfaced through the
dossier at `../../vice-and-order-research/mods/dossiers/elections-rt-module/`.
This is a **static, date-pinned source-only** read: line numbers are confirmed at
this commit; runtime behaviour that requires the host mods loaded in-game is
labelled `Needs Verification (in-game)` rather than asserted.

## What it does / why it's instructive

Elections adds a mayoral election cycle to Cities: Skylines II - candidates,
parties, polling places, voting trips, a seated mayor with policy effects, and a
Chirper feed of results. None of that would matter for the handbook except for
*how* it is built: Elections is a **module of another author's mod**. It does not
own a trip system, a time system, or a chirp feed. It borrows all three from
Realistic Trips / Time2Work and from Custom Chirps - and it borrows them the hard
way, through reflection into types the host never exposed as public API.

It is instructive because it is a maximal example of the **inter-mod dependency**
pattern under real constraints. Rather than fork the host or ship a shared
contract assembly, Elections resolves the host's private bridge types by name at
load, caches typed delegates, gates every call on an `IsAvailable` probe, and logs
exactly once when a capability is missing. Because both mods are the same author,
this is a deliberate coupling: the two must move together, and the case study is a
good teaching artifact precisely because it shows the cost of that decision - a
private-API contract with no compiler to catch drift.

## Architecture at a glance

`Mod.OnLoad` is a plain schedule: after loading settings it registers six systems
across three phases (repo/Elections/Mod.cs#L43-L48):

- `GameSimulation`: `ElectionLifecycleSystem`, `ElectionVotingSystem`,
  `MayorEffectSystem`, `MayorWorkplaceSystem` (all `UpdateAt`).
- `UIUpdate`: `ElectionUISystem`.
- `Rendering`: `ElectionVotingLocationOverlaySystem`.

There is no Harmony, no disabled vanilla system, and no `csproj` reference to the
host. Every cross-mod interaction flows through three reflection **bridge** static
classes in `Elections/Bridge/`, each of which resolves a foreign type by string
name and hands back typed delegates. The simulation systems call the bridges; the
bridges own all knowledge of the host.

Data flow per election tick:

- `ElectionVotingSystem` (interval `kUpdateInterval`,
  repo/Elections/Systems/ElectionVotingSystem.cs#L51-L54) reads the current date
  through the bridge, selects eligible citizens, and asks Realistic Trips to send
  each voter to a polling place, tagging the citizen with an `ElectionVoteTrip`
  component (repo/Elections/Systems/ElectionVotingSystem.cs#L299-L318).
- `MayorWorkplaceSystem` (interval `512`,
  repo/Elections/Systems/MayorWorkplaceSystem.cs#L34-L37) physically seats the
  winning citizen in city hall and in a home, evicting/firing occupants if needed.
- All persistent election state lives in a single `ElectionState` singleton entity
  with a hand-written `ISerializable` layout.

## Techniques demonstrated

- [Reflection bridges between mods](../how-to/recipes/reflection-mod-bridges.md)
  (family G) - `RealisticTripsBridge.EnsureResolve` resolves each host type with a
  `Type.GetType("Time2Work.Bridge.SocialTripsBridge, Time2Work") ?? FindType(...)`
  pair, where `FindType` scans `AppDomain.CurrentDomain.GetAssemblies()` as a
  fallback (repo/Elections/Bridge/RealisticTripsBridge.cs#L429-L473,
  repo/Elections/Bridge/RealisticTripsBridge.cs#L636-L653). Each host method is
  bound once into a cached `Delegate.CreateDelegate` (the fast path - no
  per-call `MethodInfo.Invoke`) (repo/Elections/Bridge/RealisticTripsBridge.cs#L618-L625).
  Availability is a single gated probe: `IsAvailable` runs `EnsureResolve` then
  returns `s_RequestSocialTrip != null`
  (repo/Elections/Bridge/RealisticTripsBridge.cs#L54-L61). When the host is
  missing the bridge degrades and logs exactly once via a `s_LoggedMissingTime`
  latch (repo/Elections/Bridge/RealisticTripsBridge.cs#L326-L331).
- [Cross-mod runtime service protocol](../how-to/recipes/cross-mod-service-protocol.md)
  (family AP) - the two mods agree on an **integer trip-type protocol** carried
  over one shared `RequestSocialTrip(citizen, target, host, int tripType, ...)`
  entry point: voting is `1001`, victory party is `1002`, bribe meeting is `1003`,
  each wrapped in a named helper
  (repo/Elections/Bridge/RealisticTripsBridge.cs#L154-L203). Capability is
  negotiated with an **overload cascade**: `CustomChirpsBridge` binds one-, two-,
  and three-target chirp overloads and each richer call falls back to the next
  poorer one when its `MethodInfo` did not resolve
  (repo/Elections/Bridge/CustomChirpsBridge.cs#L78-L126), with explicit
  `SupportsChirpWith2Targets`/`3Targets` predicates
  (repo/Elections/Bridge/CustomChirpsBridge.cs#L97-L126). `SocialTripsBridge`
  does the same for its richer *with-candidate* chirp overload
  (repo/Elections/Bridge/SocialTripsBridge.cs#L18-L44).
- [ECS ISerializable save data](../how-to/recipes/ecs-serializable-savedata.md)
  (family E) - two persistence surfaces. The small per-cim `ElectionVoteTrip`
  writes/reads five fields in fixed order
  (repo/Elections/Components/ElectionVoteTrip.cs#L14-L30). The large `ElectionState`
  fat singleton uses a **decoupled version scheme**: it *writes* `CurrentVersion =
  24` but its true field layout is `CurrentSerializedLayoutVersion = 32`, and
  `GetSerializedLayoutVersion` maps the stored marker to the layout to read
  (repo/Elections/Components/ElectionState.cs#L18-L24,
  repo/Elections/Components/ElectionState.cs#L2873-L2887). The reader is one long
  `if (layoutVersion >= N)` staircase, including a **legacy-alias migration** that
  folds two removed per-candidate cash-assistance fields into one surviving field
  (repo/Elections/Components/ElectionState.cs#L2460-L2469).
- [NameSystem custom names](../how-to/recipes/namesystem-custom-names.md)
  (family Q) - the read side is a defensive fallback chain. `ElectionNameUtility`
  tries `NameSystem.TryGetCustomName`, then a *reflected private*
  `NameSystem.GetCitizenName` (bound `Instance | NonPublic`) whose nested `Name`
  struct fields it reads by reflection, then `GetRenderedLabelName`, and finally a
  supplied fallback string
  (repo/Elections/Systems/ElectionNameUtility.cs#L16-L63).

Supporting technique on display: a deterministic data-URI portrait catalog.
`CandidatePortraitCatalog` warms a filename to base64 `data:image/jpeg` cache and
picks a stable portrait index from a hash of the candidate entity's index/version
(repo/Elections/Systems/CandidatePortraitCatalog.cs#L11-L16,
repo/Elections/Systems/CandidatePortraitCatalog.cs#L50-L54).

## Key decisions & tradeoffs

- **Depend on a host's *internals*, on purpose.** The bridges target
  `Time2Work.Bridge.SocialTripsBridge`, a `Time2Work.Bridge.ElectionsBridge`
  policy surface, and `Time2Work.Components.CitizenSchedule` - none of which are a
  stable public contract (repo/Elections/Bridge/RealisticTripsBridge.cs#L429-L473,
  repo/Elections/Bridge/RealisticTripsBridge.cs#L539-L585). Because both mods share
  an author, Elections accepts that it must be re-released in lockstep with the
  host. The reflection layer buys *soft failure* (no hard assembly-load error when
  the host is absent), not decoupling from the host's evolution.
- **Cache delegates, not just types.** Every hot call path resolves to a typed
  delegate once (`CreateDelegate`), so the per-tick voting loop never pays
  reflection cost; only cold `MethodInfo.Invoke` remains for rarely-called chirp
  paths (repo/Elections/Bridge/RealisticTripsBridge.cs#L618-L625;
  repo/Elections/Bridge/CustomChirpsBridge.cs#L236-L308).
- **Gate every entry, degrade loudly-once.** Simulation code checks
  `RealisticTripsBridge.IsAvailable` and `TryGetCurrentDateTime` before doing work,
  and posts a single explanatory chirp when Realistic Trips is missing rather than
  spamming the log each tick (repo/Elections/Systems/ElectionVotingSystem.cs#L114,
  repo/Elections/Systems/ElectionVotingSystem.cs#L162-L174).
- **Physical relocation over a data flag.** A won election does not just set a
  "mayor" field: `MayorWorkplaceSystem` seats the winner as a `Worker` in city
  hall and as a `Renter` in a home, and when a target is full it **evicts** a
  non-mayor household (`FindEvictableHousehold` -> `RemoveRenter(..., makeHomeless:
  true)`) or **fires** a non-mayor employee
  (repo/Elections/Systems/MayorWorkplaceSystem.cs#L617-L641,
  repo/Elections/Systems/MayorWorkplaceSystem.cs#L783-L795,
  repo/Elections/Systems/MayorWorkplaceSystem.cs#L603-L613). To make the game react
  to the renter change it fires a **vanilla event** the manual way: it builds an
  archetype of `Game.Common.Event + RentersUpdated` in `OnCreate`
  (repo/Elections/Systems/MayorWorkplaceSystem.cs#L95-L97), then creates an entity
  from it and sets the `RentersUpdated` payload whenever occupancy changes
  (repo/Elections/Systems/MayorWorkplaceSystem.cs#L762-L768). This is the general
  pattern for triggering a stock reaction system you do not control.
- **Version marker decoupled from field layout.** Writing `24` while the layout is
  `32` is a deliberate hedge (the source comment: keep the marker "lower than
  unpublished layout churn while still mapping it to the current field layout") so
  that in-development layout bumps do not burn published version numbers
  (repo/Elections/Components/ElectionState.cs#L18-L24). The `GetSerializedLayoutVersion`
  table encodes the known published markers (`1 -> 16`, `23 -> 31`, `24 -> 32`)
  (repo/Elections/Components/ElectionState.cs#L2873-L2887).

## Pitfalls / upstream-watch

- **Private-API contract coupling.** Every bound name -
  `SocialTripsBridge.RequestSocialTrip` with its six-parameter signature, the
  `1001/1002/1003` trip-type integers, `ElectionsBridge.SetMayorResourceConsumptionMultiplier`,
  `CitizenSchedule.dayoff/go_to_work/end_work` fields - is a string, not a compiled
  reference (repo/Elections/Bridge/RealisticTripsBridge.cs#L455-L457,
  repo/Elections/Bridge/RealisticTripsBridge.cs#L559-L576,
  repo/Elections/Bridge/RealisticTripsBridge.cs#L506-L508). If the host renames a
  method, changes a parameter type, or renumbers a trip type, `CreateDelegate`
  returns null, `IsAvailable` goes false, and Elections silently stops sending
  trips - no compiler error, no crash. Re-verify the bound signatures on every host
  release.
- **NameSystem private-member reflection.** `ElectionNameUtility` binds
  `GetCitizenName` and the nested `Name` struct's `m_NameType/m_NameID/m_NameArgs`
  as `NonPublic` (repo/Elections/Systems/ElectionNameUtility.cs#L52-L63). A CS2
  patch that renames these drops the middle rung of the fallback chain; the public
  `TryGetCustomName`/`GetRenderedLabelName` rungs still cover it, but rendered
  first/last-name quality degrades.
- **Manual event archetype must match the reaction system's expectation.** The
  hand-built `Game.Common.Event + RentersUpdated` entity is only meaningful if the
  vanilla renter-update pipeline still consumes exactly that pairing
  (repo/Elections/Systems/MayorWorkplaceSystem.cs#L95-L97,
  repo/Elections/Systems/MayorWorkplaceSystem.cs#L762-L768). If the vanilla event
  contract changes, the eviction will mutate the buffer without notifying the game.
- **Eviction/firing side effects.** Seating the mayor can make another household
  `HomelessHousehold` or strip a citizen's `Worker` component
  (repo/Elections/Systems/MayorWorkplaceSystem.cs#L715-L727,
  repo/Elections/Systems/MayorWorkplaceSystem.cs#L791-L795) - a real gameplay
  mutation, not a cosmetic tag; watch for interaction with other
  household/employment mods.

Needs Verification (in-game): whether Realistic Trips actually honours the
`1001/1002/1003` trip types and returns the trip at the requested duration; the
real-world behaviour when the host mod is *absent* (the source path degrades to a
one-shot chirp + skipped trips, but the end-to-end player experience is unverified
from source); and whether the manual `RentersUpdated` event is picked up by the
current vanilla renter pipeline. None of these can be confirmed without the running
game and the host mods loaded.

## Source pointers

- Dossier:
  `../../vice-and-order-research/mods/dossiers/elections-rt-module/`
  (index/source/modding/guide + notes).
- Repo @ `36c25afcda67c80ce75ba68d423fe8238499435a` ("version 3.1"), key files:
  - `repo/Elections/Bridge/RealisticTripsBridge.cs` - primary reflection bridge:
    type resolution, delegate cache, trip-type protocol, policy sub-bridge.
  - `repo/Elections/Bridge/CustomChirpsBridge.cs`,
    `repo/Elections/Bridge/SocialTripsBridge.cs` - overload-cascade capability
    negotiation for the chirp feeds.
  - `repo/Elections/Systems/ElectionVotingSystem.cs` - gated bridge consumption +
    `ElectionVoteTrip` tagging.
  - `repo/Elections/Systems/MayorWorkplaceSystem.cs` - physical mayor relocation,
    eviction/firing, hand-built vanilla `Event` archetype.
  - `repo/Elections/Components/ElectionState.cs`,
    `repo/Elections/Components/ElectionVoteTrip.cs` - versioned `ISerializable`
    persistence.
  - `repo/Elections/Systems/ElectionNameUtility.cs` - NameSystem reflection
    fallback chain.
  - `repo/Elections/Systems/CandidatePortraitCatalog.cs` - deterministic data-URI
    portrait catalog.
  - `repo/Elections/Mod.cs` - system scheduling.
