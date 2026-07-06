---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Reference: SystemUpdatePhase members"
Summary: "Lookup table of the Game.SystemUpdatePhase enum members that real CS2 mods schedule ECS systems into via UpdateAt/UpdateBefore/UpdateAfter, each with its frame role and a source-verified example mod citation at a pinned commit."
diataxis: reference
source_version: "1.6.0f1 (time2work-realistic-trips@d42921f; date-pinned, static source only)"
last_reverified: "2026-07-04"
status: source-verified
canonical_mods:
  - time2work-realistic-trips@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39
  - advanced-road-naming@559e72cdb3180e3e71869643367e094a22eed988
  - write-everywhere@13c70eb04e6bed152257c516a982455148f591a5
  - traffic-tool-essentials@10973595ac9ed37f47ca24eb788d4421dc295fa0
  - realistic-path-finding@50645fa6a078181365e36a42e2e27b96699bf02a
  - better-bulldozer@4408466f226db811159d92859479ae1e1c28ba06
technique_applicability: [core, simulation]
Created: 2026-07-04
Updated: 2026-07-05
Owners:
  - codex
References:
  - Label: "Explanation: System scheduling"
    Path: ../explanation/system-scheduling.md
  - Label: "Explanation: Multi-phase scheduling"
    Path: ../explanation/multi-phase-scheduling.md
  - Label: "Recipe: Event-driven ModificationEnd"
    Path: ../how-to/recipes/event-driven-modificationend.md
  - Label: Official CS2 modding wiki (authoritative enum)
    Path: https://cs2.paradoxwikis.com/Modding
---

# Reference: SystemUpdatePhase members

> **Reference - patch-sensitive.** Current live game: **1.6.0f1 "Summer Solstice"** (2026-06-22).
> Members below are cited from mods at **1.5.x-era pins**; `last_reverified: 2026-07-04` is a
> static-source check only, not a run against 1.6.0f1. The full enum and its authoritative
> frame ordering live in the game DLL - link out to the
> [official CS2 modding wiki](https://cs2.paradoxwikis.com/Modding), do not trust this list as exhaustive.

`Game.SystemUpdatePhase` is the enum a mod passes to `UpdateSystem.UpdateAt<T>(phase)`,
`UpdateBefore<T,U>(phase)`, and `UpdateAfter<T,U>(phase)` (called from `IMod.OnLoad`) to insert
an ECS `SystemBase`/`GameSystemBase` into a specific slot of the per-frame update loop. This page
tables **only the members that appear in this handbook's source corpus** - each with the phase's
frame role and one real mod that schedules into it, verified at a pinned commit. Members of the
enum not exercised by any corpus mod are **not** listed (see the wiki).

The "frame role" column is a role summary derived from the member name and cited usage; the
exact tick position and inter-phase ordering are authoritative on the wiki, not here.

## Corpus and pins

| Slug | Pinned commit | Scheduling site (at pin) |
| --- | --- | --- |
| time2work-realistic-trips | `d42921ffb1f6bbbf2c2b086bb17727bf0f637c39` | `NightShift/Mod.cs` |
| advanced-road-naming | `559e72cdb3180e3e71869643367e094a22eed988` | `Mod.cs` |
| write-everywhere | `13c70eb04e6bed152257c516a982455148f591a5` | `BelzontWE/WriteEverywhereCS2Mod.cs` |
| traffic-tool-essentials | `10973595ac9ed37f47ca24eb788d4421dc295fa0` | `TrafficToolEssentials/Mod.cs` |
| realistic-path-finding | `50645fa6a078181365e36a42e2e27b96699bf02a` | `RealisticPathFinding/Mod.cs` |
| better-bulldozer | `4408466f226db811159d92859479ae1e1c28ba06` | `BetterBulldozer/BetterBulldozerMod.cs` |

**time2work-realistic-trips** is the widest single exhibit: its `NightShift/Mod.cs` `OnLoad`
makes 37 `UpdateAt`/`UpdateBefore`/`UpdateAfter` calls across six distinct phases
(`repo/NightShift/Mod.cs#L117-L159`).

## Members used in the corpus

| Phase member | Frame role (summary) | Example mod + cite (at pin) |
| --- | --- | --- |
| `GameSimulation` | Main gameplay simulation pass; systems that advance city state each simulation tick run here. This is the default home for simulation logic. | time2work: `../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Mod.cs#L117` (`Time2WorkTimeSystem`); realistic-path-finding: `../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Mod.cs#L62` (`RPFResidentAISystem`) |
| `EditorSimulation` | Simulation pass used while in the map/asset **editor** (mirror of `GameSimulation` for editor context). | time2work: `../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Mod.cs#L118` (`Time2WorkTimeSystem` scheduled into both `GameSimulation` and `EditorSimulation`) |
| `Modification1` | First named sub-step of the ordered **Modification** pipeline (earliest `ModificationBarrier`); systems that must run at the very start of world/network modification slot here. | better-bulldozer: `../../vice-and-order-research/mods/dossiers/better-bulldozer/repo/BetterBulldozer/BetterBulldozerMod.cs#L122` (`UpdateBefore<AutomaticallyRemoveManicuredGrassSurfaceSystem>(SystemUpdatePhase.Modification1)`) |
| `Modification2B` | A named sub-step of the ordered **Modification** pipeline, aligned with the game's `SubObjectSystem` pass (`ModificationBarrier2B`). write-everywhere self-registers systems here via a reflected `UpdateSystem.UpdateAt<T>` call keyed off an `AllowedPhase` mirror of `SystemUpdatePhase`, not a literal `UpdateAt` line in `OnLoad`. | write-everywhere: `../../vice-and-order-research/mods/dossiers/write-everywhere/repo/BelzontWE/Systems/WENodeExtraDataUpdater2B.cs#L18` (`UpdatePhase => AllowedPhase.Modification2B`), registered via `../../vice-and-order-research/mods/dossiers/write-everywhere/repo/BelzontWE/Commons/IBelzontBasicSystem.cs#L55-L71` (`UpdateAt.Invoke(updateSystem, [(SystemUpdatePhase)UpdatePhase])`; Commons submodule @`3698b64`) |
| `ModificationEnd` | Tail of the world/network **Modification** pipeline, after geometry/aggregate edits are applied; common hook for reacting to placement/edit changes. | advanced-road-naming: `../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Mod.cs#L56` (`UpdateAfter<SegmentMetadataSystem, AggregateSystem>`); write-everywhere: `../../vice-and-order-research/mods/dossiers/write-everywhere/repo/BelzontWE/WriteEverywhereCS2Mod.cs#L69` (`WEWorldPickerController`); realistic-path-finding: `../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Mod.cs#L70` (`UpdateBefore<CarPrefabTurnCostFactorSystem, Game.Pathfind.LanesModifiedSystem>`) |
| `Modification4B` | A named sub-step of the ordered **Modification** pipeline (used to slot a system between vanilla modification passes). | traffic-tool-essentials: `../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Mod.cs#L218` (`UpdateBefore<PatchedTrafficLightInitializationSystem, Game.Net.TrafficLightInitializationSystem>`) |
| `Modification5` | A late named sub-step of the ordered **Modification** pipeline (`ModificationBarrier5`), after the earlier `Modification*` passes; used to defer work to the tail of the modification block. | better-bulldozer: `../../vice-and-order-research/mods/dossiers/better-bulldozer/repo/BetterBulldozer/BetterBulldozerMod.cs#L128` (`UpdateAt<HandleUpdateNextFrameSystem>(SystemUpdatePhase.Modification5)`) |
| `ToolUpdate` | Per-frame tool phase; active editing/placement tools update their input and preview here. | advanced-road-naming: `../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Mod.cs#L58` (`RoadRouteToolSystem`); traffic-tool-essentials: `../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Mod.cs#L222` (`ToolSystem`) |
| `Rendering` | Render-data preparation pass; systems that build overlay/mesh/font render data run here. | advanced-road-naming: `../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Mod.cs#L62` (`RoadRouteOverlayGeometrySystem`); write-everywhere: `../../vice-and-order-research/mods/dossiers/write-everywhere/repo/BelzontWE/WriteEverywhereCS2Mod.cs#L74` (`FontServer`) |
| `PreCulling` | Runs ahead of the visibility/culling pass (prepare per-entity visibility data before culling). | write-everywhere: `../../vice-and-order-research/mods/dossiers/write-everywhere/repo/BelzontWE/WriteEverywhereCS2Mod.cs#L83` (`WEPreCullingSystem`) |
| `UIUpdate` | UI binding/update pass; systems that push data to the Gameface UI or read UI state run here. | time2work: `../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Mod.cs#L150-L153` (four `*UISystem`s); write-everywhere: `../../vice-and-order-research/mods/dossiers/write-everywhere/repo/BelzontWE/WriteEverywhereCS2Mod.cs#L70` (`WEUISystem`); traffic-tool-essentials: `../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Mod.cs#L221` (`UISystem`) |
| `UITooltip` | Tooltip-building sub-pass within the UI phase. | advanced-road-naming: `../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Mod.cs#L59` (`RoadRouteToolTooltipSystem`); write-everywhere: `../../vice-and-order-research/mods/dossiers/write-everywhere/repo/BelzontWE/WriteEverywhereCS2Mod.cs#L67` (`WEWorldPickerTooltip`); traffic-tool-essentials: `../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Mod.cs#L220` (`TooltipSystem`) |
| `PrefabUpdate` | Prefab (re)build pass; where prefab-derived data is (re)applied. Mods that scale prefab-driven parameters slot in around here. | time2work: `../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Mod.cs#L154` (`UpdateAfter<TimeSettingsMultiplierSystem>`) |
| `PrefabReferences` | Prefab-reference resolution pass (runs during prefab initialization, ahead of `PrefabUpdate`). | time2work: `../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Mod.cs#L155` (`UpdateBefore<TimeSettingsMultiplierSystem>`) |
| `Cleanup` | End-of-frame cleanup/disposal pass. | write-everywhere: `../../vice-and-order-research/mods/dossiers/write-everywhere/repo/BelzontWE/WriteEverywhereCS2Mod.cs#L81` (`WETemplateDisposalSystem`) |
| `Deserialize` | Save-load: runs when a save game is **read** (state loaded). Systems that rebuild mod state from a save schedule here. | time2work: `../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Mod.cs#L119` (`UpdateAfter<Time2WorkTimeSystem>`); advanced-road-naming: `../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Mod.cs#L54` (`SegmentMetadataSystem`) |
| `Serialize` | Save-load: runs when a save game is **written** (state persisted). | advanced-road-naming: `../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Mod.cs#L55` (`SegmentMetadataSystem` scheduled into both `Deserialize` and `Serialize`) |

## Scheduling API (as used above)

All three overloads take a `SystemUpdatePhase` and are called on the `UpdateSystem` passed to
`IMod.OnLoad`:

| Call | Effect |
| --- | --- |
| `UpdateAt<T>(phase)` | Registers system `T` to run **in** `phase`. |
| `UpdateBefore<T,U>(phase)` | Runs `T` **before** system `U`, within `phase`. |
| `UpdateAfter<T,U>(phase)` | Runs `T` **after** system `U`, within `phase`. |

Witnessed together in one exhibit at advanced-road-naming
`../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Mod.cs#L54-L63` (all three
forms) and time2work `../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Mod.cs#L117-L159`.
A single system may be scheduled into **multiple** phases (e.g. time2work's `Time2WorkTimeSystem`
into `GameSimulation`, `EditorSimulation`, and after-`Deserialize`;
`../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Mod.cs#L117-L119`).

## Needs Verification (in-game)

- The **exact frame ordering** between phases (e.g. whether `PrefabReferences` precedes
  `PrefabUpdate`, or where `Modification4B` sits among the `Modification*` sub-steps) is not
  cited from game source here - confirm on the wiki / against a decompile per patch.
- Enum members **not** exercised by this corpus (other `Modification*` sub-phases, apply/clear
  modification phases, pre-simulation, culling, etc.) are intentionally omitted rather than
  guessed. Treat the wiki's enum listing as the authoritative superset.
- `EditorSimulation` behavior is inferred from the member name plus time2work's paired
  `GameSimulation`/`EditorSimulation` registration; its exact editor-context semantics are
  `Needs Verification (in-game)`.

## See also

- [explanation/system-scheduling.md](../explanation/system-scheduling.md) - the "why" of inserting
  systems into phases and ordering relative to vanilla systems.
- [explanation/multi-phase-scheduling.md](../explanation/multi-phase-scheduling.md) - registering one
  concern across several phases.
- [how-to/recipes/event-driven-modificationend.md](../how-to/recipes/event-driven-modificationend.md) -
  reacting to edits via `ModificationEnd`.
- [Official CS2 modding wiki](https://cs2.paradoxwikis.com/Modding) - authoritative, exhaustive
  `SystemUpdatePhase` enum and frame ordering; link out rather than duplicating.
