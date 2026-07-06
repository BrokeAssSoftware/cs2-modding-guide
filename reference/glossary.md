---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Reference: DOTS / CS2 modding glossary"
Summary: "An A-Z lookup of the DOTS/ECS/CS2 modding vocabulary used across this handbook, with each code-level term anchored to a real occurrence in a source-verified mod at a pinned commit."
diataxis: reference
source_version: "1.6.0f1 (realistic-path-finding@50645fa; date-pinned, static source only)"
last_reverified: "2026-07-04"
status: source-verified
canonical_mods:
  - realistic-path-finding@50645fa6a078181365e36a42e2e27b96699bf02a
technique_applicability: [core]
Created: 2026-07-04
Updated: 2026-07-04
Owners:
  - codex
References:
  - Label: Performance and Terminology (the layered vocabulary + tick constant)
    Path: ./performance-and-terminology.md
  - Label: ECS Fundamentals (the "why" behind these terms)
    Path: ../explanation/ecs-fundamentals.md
  - Label: Technique Index
    Path: ../technique-index.md
---

# Reference: DOTS / CS2 modding glossary

A dry, alphabetical lookup of the DOTS/ECS and CS2-modding terms this handbook uses.
Each entry is 1-3 sentences. Where a term names a real type, method, or attribute, the
**Seen in** column points at a confirmed occurrence in a source-verified mod. For the
grouped-by-layer view (runtime / scheduling / performance) and the simulation tick
constant, see [Performance and Terminology](./performance-and-terminology.md); for the
concepts behind the terms, see [ECS Fundamentals](../explanation/ecs-fundamentals.md).
Link out to the engine docs rather than trusting a duplicate:

- Unity DOTS / Entities: https://docs.unity3d.com/Packages/com.unity.entities@latest
- Official CS2 modding wiki: https://cs2.paradoxwikis.com/Modding

## How to read the citations

All `Seen in` cites resolve against ONE pinned corpus:

- **realistic-path-finding** @ `50645fa6a078181365e36a42e2e27b96699bf02a` (short `50645fa`).
- File shorthand below (e.g. `Mod.cs#L24`) is relative to
  `../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/`.
  Full path for `Mod.cs` is
  [`repo/.../Mod.cs`](../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Mod.cs);
  read the pinned content with
  `git -C <dossier>/repo show 50645fa6a078181365e36a42e2e27b96699bf02a:RealisticPathFinding/<file>`.

Entries with **no** mod cite are definitional vocabulary; they link to the handbook page
that demonstrates the term against verified source. Entries marked `Needs Verification
(in-game)` are not exercised in this corpus.

## A - E

| Term | Meaning | Seen in (`50645fa`) |
| --- | --- | --- |
| **Burst / `[BurstCompile]`** | Unity's ahead-of-time compiler that turns unmanaged jobs into native code; Burst code cannot touch managed objects. The attribute marks a job struct for compilation. | `Systems/RPFResidentAISystem.cs#L564` (on the job struct) |
| **Chunk** | A fixed-size block of memory holding entities with identical component layout; the granularity a Burst job iterates. | see [performance-and-terminology.md](./performance-and-terminology.md) |
| **Component** | A struct of plain data attached to an entity; the unit of simulation state. Selected/filtered via `ComponentType.ReadOnly<T>()` / `ReadWrite<T>()` / `Exclude<T>()`. | `Patches/RPFPatches.cs#L60`; `Systems/RPFResidentAISystem.cs#L88` |
| **`ComponentLookup<T>`** | A cached random-access handle for reading/writing one component type by entity; built once (via `GetComponentLookup<T>`) and passed into a job. | `Systems/RPFResidentAISystem.cs#L185` |
| **`ComponentTypeHandle<T>`** | A per-chunk accessor for a component type inside an `IJobChunk`; obtained via `GetComponentTypeHandle<T>` and refreshed each schedule. | `Systems/RPFResidentAISystem.cs#L172` |
| **COUI (`coui://`)** | The virtual host scheme the Gameface browser uses to load a mod's bundled HTML/JS/CSS UI assets. | see [react-ui.md](../explanation/react-ui.md) |
| **DOTS** (Data-Oriented Technology Stack) | Unity's umbrella for the ECS runtime plus the Job System and Burst compiler that CS2 is built on. | see [ecs-fundamentals.md](../explanation/ecs-fundamentals.md) |
| **`DynamicBuffer<T>` / `BufferLookup<T>`** | A variable-length (list-like) component on an entity. `BufferLookup<T>` is the cached random-access handle used inside jobs; `EntityManager.GetBuffer<T>` returns the buffer on the main thread. | `Systems/RPFResidentAISystem.cs#L727` (`BufferLookup`); `Patches/RPFPatches.cs#L72` (`GetBuffer`) |
| **ECS** (Entity Component System) | The data-oriented architecture of CS2's simulation: behaviour in *systems*, state in *components*, on lightweight *entities*. | see [ecs-fundamentals.md](../explanation/ecs-fundamentals.md) |
| **Entity** | A lightweight id that components attach to (a citizen, road segment, building). `Entity.Null` is the sentinel empty value. | `Patches/RPFPatches.cs#L76` |
| **`EntityCommandBuffer` / `.ParallelWriter`** | A deferred structural-change queue; changes recorded during a job are played back later. `.ParallelWriter` is the thread-safe form passed into a parallel job. | `Systems/RPFResidentAISystem.cs#L794` |
| **`EntityManager`** | The main-thread API for creating queries and reading/writing components/buffers directly (`CreateEntityQuery`, `HasComponent`, `GetComponentData`, `HasBuffer`, `GetBuffer`). | `Patches/RPFPatches.cs#L56` |
| **`EntityQuery`** | A filter selecting the entities a system operates on, built from `ComponentType.ReadWrite`/`ReadOnly`/`Exclude`. Created via `GetEntityQuery` (in a system) or `EntityManager.CreateEntityQuery`. | `Systems/RPFResidentAISystem.cs#L63`, `#L88`; `Patches/RPFPatches.cs#L59` |

## F - O

| Term | Meaning | Seen in (`50645fa`) |
| --- | --- | --- |
| **`[FileLocation]`** | Class attribute on a `ModSetting` naming the on-disk settings file (under the user's `ModsSettings` tree). | `Setting.cs#L16` |
| **`GameSystemBase`** | Colossal's base class CS2 mod systems derive from (a wrapper over Unity's `SystemBase`); provides `OnCreate` / `OnUpdate` and query/handle helpers. | `Systems/CarCongestionEwmaSystem.cs#L41` |
| **`GetOrCreateSystemManaged<T>()`** | Resolves (or creates) a managed system instance on a `World`, so you can order against it, read it, or toggle its `.Enabled`. The standard entry point for system replacement. | `Mod.cs#L54` |
| **Harmony** | The runtime patching library (`HarmonyLib`) used to inject into vanilla methods. A mod builds a `new Harmony(id)` and calls `PatchAll(assembly)` in `OnLoad`. | `Mod.cs#L18`, `#L82`, `#L84` |
| **Harmony prefix / postfix** | `[HarmonyPrefix]` runs before the target method (can skip it); `[HarmonyPostfix]` runs after (can read/rewrite the return). `[HarmonyPatch(type, name)]` names the target. | `Patches/RPFPatches.cs#L34`, `#L51`, `#L52`; `Patches/BusLanePatches.cs#L42`, `#L52` |
| **Harmony reverse patch / Transpiler / `Traverse`** | Reverse patch = copy a stock method body to call it directly; Transpiler = rewrite target IL (`CodeInstruction`); `Traverse` = reflection helper for private members. Not exercised in this corpus. | `Needs Verification (in-game)` |
| **`__instance` / `__result`** | Harmony's injected parameters inside a patch: `__instance` is the patched object, `__result` is the (by-ref) return value a postfix can rewrite. | `Patches/RPFPatches.cs#L53`; `Patches/BusLanePatches.cs#L52` |
| **`IJobChunk`** | A job that processes matched entities a *chunk* at a time - the common parallel pattern for heavy simulation work; its `Execute` receives one chunk. | `Systems/RPFResidentAISystem.cs#L565` (struct), `#L811` (`Execute`) |
| **`IJobEntity`** | A source-generated job that iterates matched entities one at a time (an ergonomic alternative to `IJobChunk`). Not used in this corpus, which schedules `IJobChunk` directly. | not seen in corpus; see Unity DOTS docs |
| **`ISerializable` / `SaveVersion`** | The Colossal serialization contract (`Serialize`/`Deserialize`) a system or component implements to persist state into the save; a version constant guards format changes across mod releases. | see [serialization.md](../explanation/serialization.md) |
| **Job System** | Unity's worker-thread scheduler; DOTS jobs run work in parallel across chunks. | see [performance-and-terminology.md](./performance-and-terminology.md) |
| **Managed vs unmanaged system** | Managed systems (resolved via `GetOrCreateSystemManaged<T>`) can hold references and be enabled/disabled at runtime; unmanaged systems are struct-based and Burst-friendly. | `Mod.cs#L54` (managed) |
| **`ModSetting`** | Colossal base class for a mod's Options/settings object; public properties become Options-page controls, and it is registered via `RegisterInOptionsUI()`. | `Setting.cs#L19` |
| **`moduleRegistry` / module registration** | The UI-side registration step (`registerModule` / bundled `index.js`) that mounts a mod's React components into the game's Gameface UI. | see [react-ui.md](../explanation/react-ui.md) |
| **ModsData side-car** | A separate on-disk file a mod writes (outside the save) for state that should not, or cannot, ride inside the save stream. | see [settings-and-data.md](../explanation/settings-and-data.md) |
| **`OnCreate` / `OnUpdate`** | System lifecycle hooks: `OnCreate` builds queries/lookups and gates once; `OnUpdate` runs each time the system's phase fires. | `Systems/CarCongestionEwmaSystem.cs#L65`, `#L117` |
| **`OnLoad(UpdateSystem)`** | The `IMod` entry point CS2 calls at mod load; where you read settings, register systems on the scheduler, and apply Harmony patches. `OnDispose` is the teardown counterpart. | `Mod.cs#L24`; `#L94` (`OnDispose`) |

## P - Z

| Term | Meaning | Seen in (`50645fa`) |
| --- | --- | --- |
| **`PrefabRef` / PrefabBase / PrefabSystem** | `PrefabRef` is the component linking an instance entity to its prefab entity; `PrefabBase` is the authored prefab asset and `PrefabSystem` registers/converts prefabs into entities. | `Patches/RPFPatches.cs#L79` (`PrefabRef`); PrefabBase/PrefabSystem see [ecs-fundamentals.md](../explanation/ecs-fundamentals.md) |
| **`ProfilerMarker`** | A named scope attributing frame time to a code path in the Unity profiler. | `Needs Verification (in-game)` - not in corpus |
| **`RegisterInOptionsUI()` / localization source** | `ModSetting.RegisterInOptionsUI()` publishes the Options page; a mod also adds an `IDictionarySource` (e.g. `LocaleEN`) via `localizationManager.AddSource` for labels. | `Mod.cs#L38`, `#L39` |
| **`RequireForUpdate<T>()` / `RequireForUpdate(query)`** | A gate declared in `OnCreate`: the system's `OnUpdate` is skipped entirely unless the requirement (a singleton type or a non-empty query) is met. | `Systems/CarCongestionEwmaSystem.cs#L100` |
| **`ScheduleParallel<TJob>(query, deps)`** | Schedules a job to run across the query's chunks on worker threads, returning a `JobHandle` chained through the system `Dependency`. | `Systems/RPFResidentAISystem.cs#L306`, `#L308` |
| **System** | A unit of behaviour that runs when its phase fires; CS2 mod systems derive from `GameSystemBase`. | `Systems/CarCongestionEwmaSystem.cs#L41` |
| **System replacement (`.Enabled = false`)** | Disabling a stock system (resolved via `GetOrCreateSystemManaged<T>`) and scheduling a mod system in its place - the core "override vanilla behaviour" pattern. | `Mod.cs#L54`-`L58` |
| **`SystemUpdatePhase`** | The ordered per-frame slot a system attaches to (`GameSimulation`, `ModificationEnd`, `Deserialize`, `UIUpdate`, ...); ordering does not cross phase boundaries. | `Mod.cs#L62` (`GameSimulation`), `#L70` (`ModificationEnd`) |
| **`TriggerBinding`** | A UI->C# one-shot binding: the React UI invokes a named trigger and the mod's C# handler runs (contrast `ValueBinding`, which is C#->UI state). | see [ui-cs-communication.md](../explanation/ui-cs-communication.md) |
| **`UpdateAt<T>` / `UpdateBefore<T,U>` / `UpdateAfter<T,U>`** | Scheduler registration on `UpdateSystem`: run `T` in a phase, or order it before/after another system *within the same phase*. | `Mod.cs#L62`, `#L63`, `#L70` |
| **`UpdateSystem`** | The scheduler object CS2 passes to `OnLoad`; you register your systems and their ordering on it. | `Mod.cs#L24` (parameter), `#L62` (usage) |
| **`ValueBinding<T>` / `GetterValueBinding`** | A C#->UI binding that publishes a value to the React UI and re-pushes it when it changes. | see [ui-cs-communication.md](../explanation/ui-cs-communication.md) |
| **`World` / `World.DefaultGameObjectInjectionWorld`** | The container holding all entities, components, and systems; mods reach the running world via `DefaultGameObjectInjectionWorld`. | `Mod.cs#L54` |

## See also

- Reference: [Performance and Terminology](./performance-and-terminology.md) - the same
  vocabulary grouped by layer, plus the `262144` ticks-per-day constant and budget guidance.
- Explanation: [ECS Fundamentals](../explanation/ecs-fundamentals.md) - the "why" behind
  entities, components, systems, queries, and jobs.
- Explanation: [Harmony Patching](../explanation/harmony-patching.md),
  [System Replacement](../explanation/system-replacement.md),
  [System Scheduling](../explanation/system-scheduling.md),
  [Serialization](../explanation/serialization.md),
  [UI <-> C# Communication](../explanation/ui-cs-communication.md).
- Reference: [Technique Index](../technique-index.md) - which technique family a term belongs to.
- Official [CS2 modding wiki](https://cs2.paradoxwikis.com/Modding) and
  [Unity DOTS docs](https://docs.unity3d.com/Packages/com.unity.entities@latest) - authoritative; link out, do not duplicate.
