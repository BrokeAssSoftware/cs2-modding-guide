# Simulation Systems

Cities: Skylines II exposes its simulation through Unity DOTS. Vice & Order modules replace and extend systems using the patterns outlined here.

## Update Phases

- `GameSimulation`: primary runtime simulation tick; use for AI, economy, policing logic.
- `EditorSimulation`: runs when the map/editor is active; replicate behavior that must function in authoring tools.
- `UIUpdate`: execute UI-facing systems that push data to Gameface components.
- `PrefabUpdate`: initialize prefab data during asset processing.
- `Deserialize`: rebake caches immediately after save-game loads.

Register systems explicitly with `updateSystem.UpdateAt<T>(phase)` and chain dependencies via `UpdateAfter<TAnchor, TDependency>(phase)` to avoid implicit ordering.

## Replacing Vanilla Systems

- Acquire the vanilla system with `World.DefaultGameObjectInjectionWorld.GetOrCreateSystemManaged<T>()` and set `Enabled = false`.
- Install custom systems whose `OnCreate` injects required singletons/components and verifies compatibility.
- Keep replacement scopes tight; for example, `RealisticPathFinding` disables `Game.Simulation.ResidentAISystem` plus its nested `Actions` system before inserting a customized stack.
- When overriding UI controllers, disable the vanilla `UISystem` counterpart (see `Time2WorkTimeUISystem`) to prevent duplicate rendering.

## Conditional Scheduling

- Wrap `UpdateAt` registrations in configuration gates to keep optional features cheap (for example, skip `PedestrianWalkCostFactorSystem` when `disable_ped_cost` is true).
- Use component lookups and `EnabledRefRW` handles inside systems to toggle behaviors without forcing world rebuilds.
- Implement short-lived diagnostic systems that self-disable after a fixed frame count (`AchievementFixerSystem` runs for ~300 frames post-load).

## Multi-Phase Participation

- Systems can register against multiple phases when the behavior spans contexts. `Time2WorkTimeSystem` participates in `GameSimulation`, `EditorSimulation`, and `Deserialize` to synchronize clocks everywhere.
- Guard per-phase logic inside `OnUpdate` with `UnityEngine.Application.isPlaying` or custom flags when side effects must avoid the editor.

## Harmony + Systems

- Patch high-level API calls (path queries, expenses) while DOTS systems handle per-entity logic; keep patches focused and log when they execute.
- Unpatch in `OnDispose` to avoid duplicate patches after reloads or when disabling modules mid-session.

## Testing Hooks

- Expose internal toggles or commands (with `DeveloperMode` active) to adjust multipliers or dump component state.
- For profiling, wrap heavy computations in `using (ProfilerMarker.Auto())` blocks and run targeted traces in developer builds.

## Failure Recovery

- Validate system prerequisites (component types, shared data) during `OnCreate` and log actionable errors if dependencies are missing.
- Use `TryGetSingleton` patterns when reading optional data from other modules (e.g., Realistic Path Finding sampling `Time2Work` factors).
