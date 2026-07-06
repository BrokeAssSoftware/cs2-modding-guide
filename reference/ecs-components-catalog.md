---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Reference: vanilla ECS components & systems mods touch"
Summary: "Lookup table of the vanilla Game.* ECS components, buffers, prefab-data, and systems that this handbook's source corpus actually reads, writes, disables, or reorders - each row cited to the exact mod file+line that touches it at a pinned commit."
diataxis: reference
source_version: "1.6.0f1 (time2work-realistic-trips@d42921f; date-pinned, static source only)"
last_reverified: "2026-07-04"
status: source-verified
canonical_mods:
  - time2work-realistic-trips@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39
  - market-based-economy@b83f196a36bc74388accebdeb7c81f0f35dbab37
  - traffic-tool-essentials@10973595ac9ed37f47ca24eb788d4421dc295fa0
  - outside-traffic-adjuster@42afd29638267c9dc116b040515ceaf41eb9a9e1
  - magic-mail@6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47
  - advanced-road-naming@559e72cdb3180e3e71869643367e094a22eed988
  - realistic-path-finding@50645fa6a078181365e36a42e2e27b96699bf02a
  - realistic-jobsearch@7a096b2ab974bb03cc4cf0937f250bf1d7671f31
  - magic-garbage-truck@1b6a478753e1ef4e43ac9b90d567f3d7183c7be2
  - advanced-simulation-speed@d224017f15b6be7384ffcc34528553f86b22280d
  - abandoned-building-remover-deviance-fix@a515bfe588965cbc988cba6a974242f66e5386b7
  - road-speed-adjuster@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed
technique_applicability: [core, simulation, economy]
Created: 2026-07-04
Updated: 2026-07-05
Owners:
  - codex
References:
  - Label: "Reference: CS2 Economy Systems"
    Path: ./game-systems/economy.md
  - Label: "Reference: Citizens & Households"
    Path: ./game-systems/citizens-households.md
  - Label: "Explanation: ECS Fundamentals"
    Path: ../explanation/ecs-fundamentals.md
  - Label: Official CS2 modding wiki (authoritative type list)
    Path: https://cs2.paradoxwikis.com/Modding
  - Label: Technique Index
    Path: ../technique-index.md
---

# Reference: vanilla ECS components & systems mods touch

> **Reference - patch-sensitive.** Current live game: **1.6.0f1 "Summer Solstice"** (2026-06-22).
> Every row below is cited from mod source at a **1.5.x-era pin** (see `canonical_mods`), so the
> vanilla type names, field names, and system names are `source_version` ~1.5.x and were **NOT**
> re-verified against 1.6.0f1 (`last_reverified: 2026-07-04`, static source only). CS2's
> `Game.Simulation` / `Game.Economy` / `Game.Citizens` / `Game.Net` type layouts change across
> patches. Treat every type and field as a moving target and re-verify on each game release.
> For the authoritative live type list, prefer the
> [official CS2 modding wiki](https://cs2.paradoxwikis.com/Modding).

Dry lookup of the **vanilla** ECS surfaces (components, dynamic buffers, prefab-data, and whole
systems) that this handbook's source corpus reads, writes, disables, or reorders. Every row is
**Verified**: it cites a real mod file+line that touches that surface at its pinned commit.
Mod-added components (e.g. `CitizenSchedule`, `CustomTrafficLights`) are **not** vanilla and are
listed separately at the bottom for contrast. Anything not grounded in a cited line is labelled
**Needs Verification**.

For the "why" of components vs. systems vs. buffers, see
[explanation/ecs-fundamentals.md](../explanation/ecs-fundamentals.md). For the deeper economy hook
map (price getters, wages, tax) see [game-systems/economy.md](./game-systems/economy.md); for the
citizen/household data model see [game-systems/citizens-households.md](./game-systems/citizens-households.md).

Re-verify any cite with (Git-Bash):

```
git -C ../../vice-and-order-research/mods/dossiers/<slug>/repo show <pin>:<path>
```

## How to read the columns

| Column | Meaning |
| --- | --- |
| Surface | The vanilla type or system name (as written in source). |
| Kind | `component` / `buffer` (DynamicBuffer) / `prefab-data` / `enum` / `system`. |
| R/W | `read`, `write`, `read/write`, or `disable/reorder` (for whole systems). |
| Vanilla namespace | The `Game.*` namespace the type lives in (from `using` + fully-qualified refs at the pin). |
| Touched by (cite at pin) | `slug/repo/<path>#Lxx`. |

Pins used below (short form in cites; full 40-char commits in `canonical_mods`):
`time2work-realistic-trips@d42921f`, `magic-mail@6fb3d2b`, `outside-traffic-adjuster@42afd29`,
`traffic-tool-essentials@1097359`, `advanced-road-naming@559e72c`, `realistic-path-finding@50645fa`,
`market-based-economy@b83f196`, `realistic-jobsearch@7a096b2`, `magic-garbage-truck@1b6a478`,
`advanced-simulation-speed@d224017`, `abandoned-building-remover-deviance-fix@a515bfe`,
`road-speed-adjuster@e0c0c0b`. The `abandoned-building-remover-deviance-fix` dossier slug is the
cloned repo for the Abandoned Building Remover mod; cite paths use that slug.

## Citizens & scheduling - `Game.Citizens.*`, `Game.City.*`

Read by time2work's `CitizenScheduleSystem` (which builds a per-citizen shift schedule into its
**mod-added** `CitizenSchedule` component) and by realistic-path-finding's replacement trip systems.

| Surface | Kind | R/W | Vanilla namespace | Touched by (cite at pin) |
| --- | --- | --- | --- | --- |
| `Citizen` | component | read | `Game.Citizens` | `time2work-realistic-trips/repo/NightShift/Systems/CitizenScheduleSystem.cs#L71` (query `ReadOnly`) |
| `Worker` | component | read | `Game.Citizens` | `time2work-realistic-trips/repo/NightShift/Systems/CitizenScheduleSystem.cs#L76` |
| `Student` | component | read | `Game.Citizens` | `time2work-realistic-trips/repo/NightShift/Systems/CitizenScheduleSystem.cs#L77` (`Game.Citizens.Student`) |
| `CommuterHousehold` | component | read | `Game.Citizens` | `time2work-realistic-trips/repo/NightShift/Systems/CitizenScheduleSystem.cs#L291` (`ComponentLookup`) |
| `Population` | component | read | `Game.City` | `time2work-realistic-trips/repo/NightShift/Systems/CitizenScheduleSystem.cs#L105`; also `realistic-path-finding/repo/RealisticPathFinding/Systems/RPFResourceBuyerSystem.cs#L120` |
| `TripNeeded` | buffer | read/write | `Game.Citizens` | `realistic-path-finding/repo/RealisticPathFinding/Systems/RPFTripNeededSystem.cs#L133,L137` (`ReadWrite`) |
| `ResourceBought` | component | read | `Game.Citizens` | `realistic-path-finding/repo/RealisticPathFinding/Systems/RPFResourceBuyerSystem.cs#L107` |
| `MailSender` | component | read | `Game.Citizens` | `realistic-path-finding/repo/RealisticPathFinding/Systems/RPFTripNeededSystem.cs#L419` (type handle) |
| `HouseholdMember` | component | read | `Game.Citizens` | `realistic-path-finding/repo/RealisticPathFinding/Systems/RPFTripNeededSystem.cs#L133,L418` |
| `CarKeeper` | component | read | `Game.Citizens` | `realistic-path-finding/repo/RealisticPathFinding/Systems/RPFResourceBuyerSystem.cs#L269` (`ComponentLookup`) |
| `BicycleOwner` | component | read | `Game.Citizens` | `realistic-path-finding/repo/RealisticPathFinding/Systems/RPFResourceBuyerSystem.cs#L270` (`ComponentLookup`) |

See [game-systems/citizens-households.md](./game-systems/citizens-households.md) for the household
data model these hang off.

## Buildings & properties - `Game.Buildings.*`

| Surface | Kind | R/W | Vanilla namespace | Touched by (cite at pin) |
| --- | --- | --- | --- | --- |
| `PropertyRenter` | component | read | `Game.Buildings` | `time2work-realistic-trips/repo/NightShift/Systems/CitizenScheduleSystem.cs#L283,L541` (`ComponentLookup`); `realistic-path-finding/repo/RealisticPathFinding/Systems/RPFResourceBuyerSystem.cs#L268` |
| `Student` (building buffer) | buffer | read | `Game.Buildings` | `time2work-realistic-trips/repo/NightShift/Systems/CitizenScheduleSystem.cs#L287` (`BufferLookup<Game.Buildings.Student>`) |
| `PostFacility` | component | read | `Game.Buildings` | `magic-mail/repo/Systems/MagicMailSystem.cs#L89` (query `ReadOnly<Game.Buildings.PostFacility>`) |

## Economy & companies - `Game.Economy.*`, `Game.Companies.*`

`Game.Economy.Resources` is the per-entity resource ledger (a `DynamicBuffer`). Magic Mail rewrites
the mail resources on post facilities; MBE and realistic-path-finding read/write it for production
and buying. See [game-systems/economy.md](./game-systems/economy.md) for the full economy hook map.

| Surface | Kind | R/W | Vanilla namespace | Touched by (cite at pin) |
| --- | --- | --- | --- | --- |
| `Resources` | buffer | read/write | `Game.Economy` | `magic-mail/repo/Systems/MagicMailSystem.cs#L90` (query `ReadWrite`), `#L161` (`GetBuffer`), `#L465` (`resources.Add`); `market-based-economy/repo/Economy/MarketProductSystem.cs#L45,L128` (via `EconomyUtils.AddResources`); `realistic-path-finding/repo/RealisticPathFinding/Systems/RPFTripNeededSystem.cs#L137` (`ReadWrite<Game.Economy.Resources>`) |
| `Resource` (enum) | enum | read | `Game.Economy` | `magic-mail/repo/Systems/MagicMailSystem.cs#L238-L240` (`Resource.LocalMail`, `Resource.OutgoingMail`, `Resource.UnsortedMail`); `market-based-economy/repo/Economy/MarketProductSystem.cs#L160` (`Resource.Money`) |
| `EconomyUtils.AddResources` / `GetResources` | static util | read/write | `Game.Economy` | `market-based-economy/repo/Economy/MarketProductSystem.cs#L128,L131,L159` |
| `EconomyParameterData` | component | read | `Game.Prefabs` | `time2work-realistic-trips/repo/NightShift/Systems/CitizenScheduleSystem.cs#L66`; `realistic-path-finding/repo/RealisticPathFinding/Systems/RPFResourceBuyerSystem.cs#L118`; `market-based-economy/repo/Economy/CompanyProfitAdjustmentSystem.cs#L44,L64,L104` (singleton read/write via `EconomyParameterAccess` reflected getters/setters, `repo/Economy/EconomyParameterAccess.cs#L26-L61`) |
| `ResourceBuyer` | component | read/write | `Game.Companies` | `realistic-path-finding/repo/RealisticPathFinding/Systems/RPFResourceBuyerSystem.cs#L94` (`ReadWrite`); `#L266` (`ServiceAvailable` lookup adjacent) |
| `ServiceAvailable` | component | read | `Game.Companies` | `realistic-path-finding/repo/RealisticPathFinding/Systems/RPFResourceBuyerSystem.cs#L266` (`ComponentLookup`) |
| `Employee` | buffer | read | `Game.Companies` | `market-based-economy/repo/Economy/MarketProductSystem.cs#L32,L39` (`BufferLookup<Employee>`) |
| `WorkProvider` (`m_MaxWorkers`) | component | read/write | `Game.Companies` | see [game-systems/economy.md](./game-systems/economy.md) - `market-based-economy/repo/Economy/WorkforceUtilizationManager.cs#L47` |
| `TaxPayer` | component | read/write | `Game.Simulation` | see [game-systems/economy.md](./game-systems/economy.md) - `market-based-economy/repo/Economy/CompanyProfitAdjustmentSystem.cs#L59` |

## Jobs & job-seeking - `Game.Agents.*`, `Game.Companies.*`, `Game.Pathfind.*`

Realistic Job Search reweights which workplace a job seeker accepts. It runs its own systems
`UpdateBefore` vanilla `FindJobSystem`, reads the free-slot and path-result state, and both filters
the pathfind setup queue (via a Harmony prefix on `PathfindSetupSystem.CompleteSetup`) and strips
`PathInformation` off seekers it wants vanilla to defer.

| Surface | Kind | R/W | Vanilla namespace | Touched by (cite at pin) |
| --- | --- | --- | --- | --- |
| `JobSeeker` | component | read | `Game.Agents` | `realistic-jobsearch/repo/RealisticJobSearch/Systems/GravityAcceptanceGateSystem.cs#L48` (query `ReadOnly<JobSeeker>`, "same shape as vanilla `StartWorkingJob`'s query") |
| `FreeWorkplaces` | component | read | `Game.Companies` | `realistic-jobsearch/repo/RealisticJobSearch/Systems/GravityAcceptanceGateSystem.cs#L65,L110-L111` (`ComponentLookup`, `.Count`); `realistic-jobsearch/repo/RealisticJobSearch/Systems/GravityPreFilterSystem.cs#L27,L41,L101`; `realistic-jobsearch/repo/RealisticJobSearch/Patches/Patch_CompleteSetup_FilterJobSeekerTargets.cs#L46,L60,L112` |
| `PathInformation` | component | read/write | `Game.Pathfind` | `realistic-jobsearch/repo/RealisticJobSearch/Systems/GravityAcceptanceGateSystem.cs#L50,L64,L80` (read fields `m_Destination`/`m_State`), `#L121` (`ECB.RemoveComponent<PathInformation>` to defer a seeker) |
| `PathTarget` | struct (setup item) | read/write | `Game.Pathfind` | `realistic-jobsearch/repo/RealisticJobSearch/Patches/Patch_CompleteSetup_FilterJobSeekerTargets.cs#L94` (`(PathTarget t, float U)` scoring), `#L181,L184,L190,L197` (mutate the setup target `UnsafeList<PathTarget>`) |
| `PathfindSetupSystem.SetupListItem` | nested struct | read/write | `Game.Pathfind` | `realistic-jobsearch/repo/RealisticJobSearch/Patches/Patch_CompleteSetup_FilterJobSeekerTargets.cs#L27-L28` (`AccessTools.FieldRef` into private `m_SetupList` `NativeList<PathfindSetupSystem.SetupListItem>`), `#L86` (`dst.m_Target.m_Type == SetupTargetType.JobSeekerTo`) |
| `SetupTargetType` (enum) | enum | read | `Game.Pathfind` | `realistic-jobsearch/repo/RealisticJobSearch/Systems/GravityPreFilterSystem.cs#L172` (`SetupTargetType.JobSeekerTo`); `realistic-jobsearch/repo/RealisticJobSearch/Patches/Patch_CompleteSetup_FilterJobSeekerTargets.cs#L86` |

## Garbage & waste collection - `Game.Buildings.*`, `Game.Vehicles.*`, `Game.Simulation.*`, `Game.Notifications.*`

Magic Garbage Truck rescales garbage accumulation/collection: it reads producer garbage levels,
mutates the `GarbageParameterData` singleton thresholds, inspects truck state, and clears the vanilla
"garbage piling up" map icon through the vanilla `IconCommandSystem` command buffer.

| Surface | Kind | R/W | Vanilla namespace | Touched by (cite at pin) |
| --- | --- | --- | --- | --- |
| `GarbageProducer` | component | read/write | `Game.Buildings` | `magic-garbage-truck/repo/Systems/TotalMagicSystem.cs#L56` (`ReadWrite<GarbageProducer>`), `#L99,L128,L137,L142` (chunk array, `m_Flags`); read at `magic-garbage-truck/repo/Systems/GarbageStatusSystem.cs#L250,L435,L522-L524` (`m_Garbage`); `magic-garbage-truck/repo/Systems/GarbagePriorityAssistSystem.cs#L187-L188` |
| `GarbageProducerFlags` (enum) | enum | read/write | `Game.Buildings` | `magic-garbage-truck/repo/Systems/TotalMagicSystem.cs#L146` (`& GarbageProducerFlags.GarbagePilingUpWarning`), `#L156` (`&= ~...` clears the flag) |
| `GarbageParameterData` | component (singleton) | read/write | `Game.Prefabs` | `magic-garbage-truck/repo/Systems/GarbageThresholdSystem.cs#L40,L68,L73` (`TryGetSingletonRW`, `ValueRW`); `magic-garbage-truck/repo/Systems/GarbagePriorityAssistSystem.cs#L79,L134,L143`; read `magic-garbage-truck/repo/Systems/TotalMagicSystem.cs#L93-L94,L149` (`m_GarbageNotificationPrefab`) |
| `GarbageTransferRequest` (`m_Flags`) | component | read | `Game.Simulation` | `magic-garbage-truck/repo/Systems/GarbageTransferProbe.cs#L119-L120` (`ComponentLookup<Game.Simulation.GarbageTransferRequest>`), `#L163,L168,L183,L188` (read `m_Flags & GarbageTransferRequestFlags.RequireTransport`) |
| `GarbageTruck` (`m_State`) | component | read | `Game.Vehicles` | `magic-garbage-truck/repo/Systems/GarbageStatusSystem.cs#L280,L540-L541` (query `Game.Vehicles.GarbageTruck`), `#L551` (`truckRef.ValueRO.m_State`), `#L614` (`HasComponent`) |
| `GarbageTruckFlags` (enum) | enum | read | `Game.Vehicles` | `magic-garbage-truck/repo/Systems/GarbageStatusSystem.cs#L551,L553,L558` (`state & GarbageTruckFlags.Returning`/`.Disabled`) |
| `Resource.Garbage` (enum member) | enum | read | `Game.Economy` | `magic-garbage-truck/repo/Systems/GarbageTransferProbe.cs#L147` (`Game.Economy.Resource.Garbage`) |

## Simulation control & shared settings - `Game.Simulation.*`, `Game.Settings.*`

Advanced Simulation Speed reads and drives the vanilla `SimulationSystem` speed fields and mirrors
the game's legacy-UI toggle, exposing them to its own UI bindings.

| Surface | Kind | R/W | Vanilla namespace | Touched by (cite at pin) |
| --- | --- | --- | --- | --- |
| `SimulationSystem.selectedSpeed` | system field | read/write | `Game.Simulation` | `advanced-simulation-speed/repo/Systems/AdvancedSimSpeedUISystem.cs#L26-L27` (get/set property wrapper), `#L31,L54` (binding getter) |
| `SimulationSystem.smoothSpeed` | system field | read | `Game.Simulation` | `advanced-simulation-speed/repo/Systems/AdvancedSimSpeedUISystem.cs#L30` (`GetActualSpeed`) |
| `SharedSettings.instance.userInterface` (`InterfaceSettings`) | settings object | read | `Game.Settings` | `advanced-simulation-speed/repo/Systems/AdvancedSimSpeedUISystem.cs#L36` (`SharedSettings.instance.userInterface`) |
| `InterfaceSettings.useLegacyInterface` | settings field | read | `Game.Settings` | `advanced-simulation-speed/repo/Systems/AdvancedSimSpeedUISystem.cs#L60,L95` (mirrors legacy-UI flag into a binding) |

## Prefab-data (`Game.Prefabs.*`) written to reconfigure vanilla content

These are prefab-data components that mods mutate on **prefab entities** at load to rescale vanilla
behaviour (spawn rates, mail/van capacities) without touching runtime instances.

| Surface | Kind | R/W | Vanilla namespace | Touched by (cite at pin) |
| --- | --- | --- | --- | --- |
| `TrafficSpawnerData` (`m_SpawnRate`) | prefab-data | read/write | `Game.Prefabs` | `outside-traffic-adjuster/repo/SpawnRateEditorSystem.cs#L110-L114` (`TryGetComponentData` -> set `m_SpawnRate` -> `AddComponentData`) |
| `PostFacilityData` (`m_PostVanCapacity`, `m_PostTruckCapacity`, `m_SortingRate`) | prefab-data | read/write | `Game.Prefabs` | `magic-mail/repo/Systems/MailCapacitySystem.cs#L160-L192` (`RefRW<PostFacilityData>`, write `m_PostVanCapacity` `#L192`); read `magic-mail/repo/Systems/MagicMailSystem.cs#L146,L197` |
| `PostVanData` (`m_MailCapacity`) | prefab-data | read/write | `Game.Prefabs` | `magic-mail/repo/Systems/MailCapacitySystem.cs#L134-L149` (`RefRW<PostVanData>`, write `m_MailCapacity` `#L149`) |
| `PostVan` / `PostFacility` (prefab component baselines) | prefab-data | read | `Game.Prefabs` | `magic-mail/repo/Systems/MailCapacitySystem.cs#L221` (`Game.Prefabs.PostVan`), `#L239` (`Game.Prefabs.PostFacility`) |
| `PrefabRef` | component | read | `Game.Prefabs` | `time2work-realistic-trips/repo/NightShift/Systems/CitizenScheduleSystem.cs#L285`; `realistic-path-finding/repo/RealisticPathFinding/Systems/CarCongestionEwmaSystem.cs#L48,L124` |
| `RoadData` | prefab-data | read | `Game.Prefabs` | `realistic-path-finding/repo/RealisticPathFinding/Systems/CarCongestionEwmaSystem.cs#L49,L125` |
| `TimeSettingsData` | component (singleton) | read | `Game.Prefabs` | `time2work-realistic-trips/repo/NightShift/Systems/Time2WorkTimeSystem.cs#L37` (`ReadOnly<TimeSettingsData>`), `#L55,L60,L66,L73` (passed to `GetTicks`/`GetTimeOfDay`); `.../Time2WorkDeathCheckSystem.cs#L74,L119,L230` (`GetSingleton`); read as vanilla `TimeSystem.GetYear`/`GetTimeOfYear` argument in Harmony patches `.../Patches/Time2WorkPatches.cs#L59,L71,L133,L151,L161` |

## Net & roads - `Game.Net.*`, `Game.UI.NameSystem`

Advanced Road Naming reads the road/aggregate graph and drives the vanilla `NameSystem` to store
custom names; traffic-tool-essentials replaces the vanilla traffic-light pass and reads/writes the
lane-signal graph.

| Surface | Kind | R/W | Vanilla namespace | Touched by (cite at pin) |
| --- | --- | --- | --- | --- |
| `Edge` | component | read | `Game.Net` | `advanced-road-naming/repo/Systems/SegmentMetadataSystem.cs#L71` (query `ReadOnly<Edge>`) |
| `Road` | component | read | `Game.Net` | `advanced-road-naming/repo/Systems/SegmentMetadataSystem.cs#L71` |
| `Aggregated` | component | read | `Game.Net` | `advanced-road-naming/repo/Systems/SegmentMetadataSystem.cs#L71` |
| `AggregateElement` | buffer | read | `Game.Net` | `advanced-road-naming/repo/Systems/SegmentMetadataSystem.cs#L72` |
| `TrafficLights` | component | read/write | `Game.Net` | `traffic-tool-essentials/repo/TrafficToolEssentials/Systems/TrafficLightSystems/Simulation/PatchedTrafficLightSystem.cs#L49` (`ComponentTypeHandle`), `#L114,L129` (read+mutate) |
| `LaneSignal` | component | read/write | `Game.Net` | `traffic-tool-essentials/repo/TrafficToolEssentials/Systems/TrafficLightSystems/Simulation/PatchedTrafficLightSystem.cs#L97` (`ComponentLookup`), written via `UpdateLaneSignals` `#L159` |
| `TrafficLight` | component | read | `Game.Net` | `traffic-tool-essentials/repo/TrafficToolEssentials/Systems/TrafficLightSystems/Simulation/PatchedTrafficLightSystem.cs#L100` (`ComponentLookup`) |
| `SubLane` | buffer | read | `Game.Net` | `traffic-tool-essentials/repo/TrafficToolEssentials/Systems/TrafficLightSystems/Simulation/PatchedTrafficLightSystem.cs#L41,L115` (`BufferTypeHandle<Game.Net.SubLane>`) |
| `Curve` | component | read | `Game.Net` | `realistic-path-finding/repo/RealisticPathFinding/Systems/CarCongestionEwmaSystem.cs#L47,L123` |
| `CarCurrentLane` | component | read | `Game.Vehicles` | `realistic-path-finding/repo/RealisticPathFinding/Systems/CarCongestionEwmaSystem.cs#L46,L122,L71` |
| `PathInformation` | component | read | `Game.Pathfind` | `realistic-path-finding/repo/RealisticPathFinding/Systems/RPFResourceBuyerSystem.cs#L267` (`ComponentLookup`) |
| `CarLane` (`m_DefaultSpeedLimit`, `m_SpeedLimit`) | component | read/write | `Game.Net` | `road-speed-adjuster/repo/Systems/RoadSpeedApplySystem.cs#L114,L120-L121,L126` (`GetComponentData` -> set both speed fields -> `SetComponentData`, `m_Flags` preserved `#L117,L124`); read `road-speed-adjuster/repo/Systems/RoadSpeedToolSystem.cs#L397-L403` (`m_SpeedLimit`); restore `road-speed-adjuster/repo/Systems/ClearCustomSpeedsSystem.cs#L105-L108` |
| `TrackLane` (`m_SpeedLimit`) | component | read/write | `Game.Net` | `road-speed-adjuster/repo/Systems/RoadSpeedApplySystem.cs#L129,L131,L133,L135` (`GetComponentData` -> set `m_SpeedLimit` -> `SetComponentData`); read `road-speed-adjuster/repo/Systems/RoadSpeedToolSystem.cs#L407-L410`; restore `.../ClearCustomSpeedsSystem.cs#L112-L115` |

## Areas, sub-nets & cascade deletion - `Game.Areas.*`, `Game.Net.*`, `Game.Common.*`

Abandoned Building Remover queries abandoned buildings and cascades a `Deleted` tag onto their
owned sub-objects before deleting the building itself - the vanilla pattern for removing an entity
plus everything it owns. It reads the `SubArea` / `SubNet` / `SubLane` buffers **writably** (to walk
them) and stamps `Game.Common.Deleted` on each referenced entity.

| Surface | Kind | R/W | Vanilla namespace | Touched by (cite at pin) |
| --- | --- | --- | --- | --- |
| `Abandoned` | component | read | `Game.Buildings` | `abandoned-building-remover-deviance-fix/repo/AbandonedBuildingRemoverSystem.cs#L32` (query `ReadOnly<Abandoned>`) |
| `Building` | component | read | `Game.Buildings` | `abandoned-building-remover-deviance-fix/repo/AbandonedBuildingRemoverSystem.cs#L33` (query `ReadOnly<Building>`) |
| `SubArea` (`m_Area`) | buffer | read | `Game.Areas` | `abandoned-building-remover-deviance-fix/repo/AbandonedBuildingRemoverSystem.cs#L61,L73` (`GetBufferLookup<SubArea>(false)`, `TryGetBuffer<SubArea>`), `#L116,L133` (job lookup, `entry.m_Area`) |
| `SubNet` (`m_SubNet`) | buffer | read | `Game.Net` | `abandoned-building-remover-deviance-fix/repo/AbandonedBuildingRemoverSystem.cs#L60,L82` (`GetBufferLookup<SubNet>(false)`, `TryGetBuffer<SubNet>`), `#L87,L117,L149` (`net.m_SubNet`, `entry.m_SubNet`) |
| `SubLane` (`m_SubLane`) | buffer | read | `Game.Net` | `abandoned-building-remover-deviance-fix/repo/AbandonedBuildingRemoverSystem.cs#L96,L141` (`lane.m_SubLane`, `entry.m_SubLane`) |
| `Deleted` | component | write | `Game.Common` | `abandoned-building-remover-deviance-fix/repo/AbandonedBuildingRemoverSystem.cs#L78,L87,L96,L100` (`EntityManager.AddComponent<Deleted>` on sub-areas/sub-nets/sub-lanes then the building), `#L133,L141,L149,L153` (ECB cascade in job); also excluded from the query `#L37` |

## Whole vanilla systems disabled or reordered

Some mods do not just read/write components - they take over a whole vanilla `SystemBase` by
disabling it (`system.Enabled = false`) and/or scheduling a replacement before it. See
[explanation/ecs-fundamentals.md](../explanation/ecs-fundamentals.md) and
[game-systems/economy.md](./game-systems/economy.md) for the ordering pattern.

| Vanilla system | Vanilla namespace | Interaction | Touched by (cite at pin) |
| --- | --- | --- | --- |
| `ResidentAISystem` | `Game.Simulation` | disabled (`Enabled = false`), replaced by `RPFResidentAISystem` | `realistic-path-finding/repo/RealisticPathFinding/Mod.cs#L54` |
| `ResidentAISystem.Actions` | `Game.Simulation` | disabled | `realistic-path-finding/repo/RealisticPathFinding/Mod.cs#L55` |
| `TripNeededSystem` | `Game.Simulation` | disabled, replaced by `RPFTripNeededSystem` | `realistic-path-finding/repo/RealisticPathFinding/Mod.cs#L56` |
| `ResourceBuyerSystem` | `Game.Simulation` | disabled, replaced by `RPFResourceBuyerSystem` | `realistic-path-finding/repo/RealisticPathFinding/Mod.cs#L58` |
| `LanesModifiedSystem` | `Game.Pathfind` | ordering target (`UpdateBefore`) | `realistic-path-finding/repo/RealisticPathFinding/Mod.cs#L70` |
| `TrafficLightInitializationSystem` | `Game.Net` | disabled + query-narrowed + `UpdateBefore` replacement | `traffic-tool-essentials/repo/TrafficToolEssentials/Mod.cs#L122,L212,L218,L249` |
| `TrafficLightSystem` | `Game.Simulation` | disabled + query-narrowed + `UpdateBefore` replacement | `traffic-tool-essentials/repo/TrafficToolEssentials/Mod.cs#L123,L213,L219,L250` |
| `NameSystem` | `Game.UI` | driven (`GetOrCreateSystemManaged`, then `SetCustomName` / `TryGetCustomName` / `GetRenderedLabelName`) | `advanced-road-naming/repo/Systems/SegmentMetadataSystem.cs#L70,L852,L855,L1658`; `advanced-road-naming/repo/Systems/RoadSelectionInfoSectionSystem.cs#L25,L192-L195` |
| `NetToolSystem` | `Game.Tools` | read (toolID) for tool integration | `traffic-tool-essentials/repo/TrafficToolEssentials/Mod.cs#L153,L207` |
| `WorkProviderSystem.OnUpdate` | `Game.Simulation` | Harmony postfix target (see economy.md) | `market-based-economy/repo/Harmony/HarmonyBridge.cs#L139-L157` |
| `FindJobSystem` | `Game.Simulation` | `UpdateBefore` ordering target for three replacement systems | `realistic-jobsearch/repo/RealisticJobSearch/Mod.cs#L34,L36,L38,L43` |
| `PathfindSetupSystem` | `Game.Pathfind` | Harmony prefix on `CompleteSetup` (filters the private `m_SetupList` job-seeker targets); also `GetOrCreateSystemManaged` + `GetQueue` to push setup items | `realistic-jobsearch/repo/RealisticJobSearch/Patches/Patch_CompleteSetup_FilterJobSeekerTargets.cs#L15-L16,L49`; `realistic-jobsearch/repo/RealisticJobSearch/Systems/GravityPreFilterSystem.cs#L141-L142` |
| `IconCommandSystem` | `Game.Notifications` | driven (`GetOrCreateSystemManaged`, `CreateCommandBuffer`, `AddCommandBufferWriter`) to remove the vanilla garbage map-icon | `magic-garbage-truck/repo/Systems/TotalMagicSystem.cs#L37,L50,L91,L104` |
| `SimulationSystem` | `Game.Simulation` | driven (`GetOrCreateSystemManaged`, read/write `selectedSpeed`, read `smoothSpeed`) | `advanced-simulation-speed/repo/Systems/AdvancedSimSpeedUISystem.cs#L47,L26-L27,L30` |
| `PayWageSystem` / `TaxSystem` / `ResourceExporterSystem` | `Game.Simulation` | `UpdateBefore` ordering targets (see economy.md) | `market-based-economy/repo/Mod.cs#L54-L56` |

## Mod-added components (NOT vanilla - listed for contrast)

These are `IComponentData` structs the mods define themselves and attach to vanilla entities; do
not treat them as vanilla surfaces.

| Component | Defining mod (cite at pin) | Attached to |
| --- | --- | --- |
| `CitizenSchedule` | `time2work-realistic-trips/repo/NightShift/Components/CitizenSchedule.cs`; written `ReadWrite` at `.../Systems/CitizenScheduleSystem.cs#L72` | vanilla citizen entities |
| `CustomTrafficLights` | `traffic-tool-essentials/repo/TrafficToolEssentials/Components/CustomTrafficLights.cs#L6` (`struct CustomTrafficLights : IComponentData, IQueryTypeParameter, ISerializable`) | vanilla node entities that carry `Game.Net.TrafficLights` |
| `AdvancedRoadNamingManagedAggregate` | `advanced-road-naming/repo/Systems/SegmentMetadataSystem.cs#L72` (query) | vanilla road-aggregate entities |
| `LaneSampleState` / `CarEdgeTiming` | `realistic-path-finding/repo/RealisticPathFinding/Systems/CarCongestionEwmaSystem.cs#L36` (`struct LaneSampleState`), `#L29` (`struct CarEdgeTiming`) | `LaneSampleState` added to vanilla car/vehicle entities (`#L143`), `CarEdgeTiming` read off lane entities (`#L191`) |

## Needs Verification

- **Exact field layouts** of every prefab-data and component above (e.g. the full field set of
  `PostFacilityData`, `TrafficSpawnerData`, `CustomTrafficLights`'s vanilla counterparts) - only the
  fields cited by line are confirmed; the rest are patch-sensitive.
- **1.6.0f1 type/namespace stability** - all cites are 1.5.x-era pins; type moves/renames between
  1.5.x and 1.6.0f1 are not verified here. Cross-check against the
  [official wiki](https://cs2.paradoxwikis.com/Modding).
- **`ResidentAISystem.Actions`** as a nested vanilla type name - confirmed only as the string used in
  `Mod.cs#L55`; its internal shape is not read here.
- Any vanilla surface **not** in the corpus (rendering, zoning, water/electricity networks, etc.) is
  simply out of scope, not "does not exist".
- **Reflection-resolved names** are string lookups, not compiler-checked references, so their vanilla
  existence is `Needs Verification (in-game)`: `PathfindSetupSystem`'s private `m_SetupList` /
  `m_SetupDependencies` fields (`realistic-jobsearch/.../Patch_CompleteSetup_FilterJobSeekerTargets.cs#L27-L31`,
  via `AccessTools.FieldRefAccess`), and the `EconomyParameterData` field getters/setters that
  `market-based-economy/repo/Economy/EconomyParameterAccess.cs` resolves by logical name (e.g.
  `s_ResidentialMinimumEarnings`, `s_Pension`). The mod itself logs a warning when a name is absent
  (`EconomyParameterAccess.cs#L86`), i.e. these are known-fragile across patches.
- **`market-based-economy@b83f196` is the oldest pin (~1.4.x)** - highest drift risk. Its
  `WorkProvider` / `Employee` / `TaxPayer` / `EconomyParameterData` cites were re-confirmed at the
  pin (static source), but the 1.4->1.6 type layout gap is the largest here; re-verify first.
- **Enum members and nested types** cited by name (`GarbageTruckFlags.Returning`/`.Disabled`,
  `GarbageProducerFlags.GarbagePilingUpWarning`, `Resource.Garbage`, `SetupTargetType.JobSeekerTo`,
  `PathfindSetupSystem.SetupListItem`) are confirmed present at the mod's pin but are especially
  patch-sensitive (enum reorderings/renames are common); re-verify against the running game.

## Related

- [game-systems/economy.md](./game-systems/economy.md) - deep economy hook map (price getters,
  wages, tax, `Employee`/`WorkProvider`/`TaxPayer`).
- [game-systems/citizens-households.md](./game-systems/citizens-households.md) - citizen/household
  data model behind `Citizen` / `HouseholdMember` / `Worker`.
- [explanation/ecs-fundamentals.md](../explanation/ecs-fundamentals.md) - components vs. systems vs.
  buffers; why prefab-data differs from instance components.
- [technique-index.md](../technique-index.md) - the technique families (system replacement,
  prefab-data mutation, Harmony postfix) that these surfaces are hooked through.
- Official [CS2 modding wiki](https://cs2.paradoxwikis.com/Modding) - authoritative live type list.
