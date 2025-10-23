# Simulation Systems

Cities: Skylines II exposes most gameplay through Unity DOTS systems. Vice & Order modules replace or extend these systems to introduce new mechanics while keeping performance predictable. This guide documents the workflow for disabling vanilla systems, wiring custom systems into the update loop, coordinating with Harmony patches, and testing safely.

## Update Phases
| Phase | When it Runs | Typical Use |
| --- | --- | --- |
| `GameSimulation` | Every frame during live gameplay | AI, economics, policing, vice loops |
| `EditorSimulation` | While the map/editor is open | Authoring tools, preview logic |
| `UIUpdate` | Each UI frame | Push data into Gameface bindings |
| `Deserialize` | Immediately after loading a save | Rehydrate caches or rebuild derived data |
| `PrefabUpdate` | During asset compilation | Bake prefab metadata, register custom prefabs |

Always schedule systems explicitly with `updateSystem.UpdateAt` and use `UpdateAfter` / `UpdateBefore` to make dependencies obvious. Avoid relying on implicit DOTS ordering.

## Replacing Vanilla Systems
1. Disable the existing system:
   ```csharp
   var world = World.DefaultGameObjectInjectionWorld;
   world.GetOrCreateSystemManaged<Game.Simulation.ResidentAISystem>().Enabled = false;
   ```
2. Register your replacement:
   ```csharp
   updateSystem.UpdateAt<VnoResidentAISystem>(SystemUpdatePhase.GameSimulation);
   updateSystem.UpdateAfter<VnoResidentAISystem, Game.Simulation.StatisticSystem>(SystemUpdatePhase.GameSimulation);
   ```
3. Keep the scope tight. Disable only the vanilla systems you replace and document the changes so future maintainers understand the impact.

## SystemBase Template
```csharp
public partial class VnoResidentAISystem : SystemBase
{
    private EntityQuery _citizenQuery;
    private ComponentLookup<VnoAttributes> _attributesLookup;

    protected override void OnCreate()
    {
        RequireForUpdate<VnoAttributes>();
        _citizenQuery = GetEntityQuery(ComponentType.ReadOnly<Citizen>(), ComponentType.ReadWrite<VnoAttributes>());
        _attributesLookup = GetComponentLookup<VnoAttributes>();
    }

    protected override void OnUpdate()
    {
        var deltaTime = Time.DeltaTime;
        _attributesLookup.Update(this);

        Entities
            .WithName("VnoResidentAISystem")
            .WithStoreEntityQueryInField(ref _citizenQuery)
            .ForEach((ref VnoAttributes attributes, in Citizen citizen) =>
            {
                attributes.Heat = math.clamp(attributes.Heat + deltaTime * attributes.GainRate, 0f, 100f);
            })
            .ScheduleParallel();
    }
}
```
**Highlights**
- `RequireForUpdate` prevents the system from running when prerequisites are missing.
- Cache queries and lookups in `OnCreate` and refresh them inside `OnUpdate`.
- Prefer `ScheduleParallel()` when you are not making structural changes.
- Guard optional behaviour with settings flags so optional features are free when disabled.

## Multi-Phase Participation
Register the same system for multiple phases when you need work during load and gameplay:
```csharp
updateSystem.UpdateAt<VnoHeatCacheSystem>(SystemUpdatePhase.Deserialize);
updateSystem.UpdateAt<VnoHeatCacheSystem>(SystemUpdatePhase.GameSimulation);
```
Inside `OnUpdate`, branch on the current context:
```csharp
protected override void OnUpdate()
{
    if (SystemAPI.TryGetSingleton(out DeserializeInProgress deserialize) && deserialize.value)
    {
        RebuildCaches();
        return;
    }

    if (!Application.isPlaying) return;
    TickSimulation(Time.DeltaTime);
}
```

## Conditional Scheduling
- Register systems only when configuration enables the feature. For runtime toggles, keep the system registered but return early when disabled.
- Use lightweight components (`EnabledRefRW`, shared singletons) to flip behaviours without rebuilding worlds.
- Defer heavy work to short-lived diagnostic systems that disable themselves after completion.

## Harmony and Systems
- Use Harmony patches for high-level API hooks (path queries, expenses) while DOTS handles the per-entity logic.
- Keep patches minimal, log when they run, and unpatch in `OnDispose` so hot reload works.

## Diagnostics and Testing
- Expose developer-only commands for dumps and toggles. Protect them behind `-developerMode`.
- Wrap hot code paths in `ProfilerMarker` scopes and profile regression saves before releases.
- Maintain automated smoke tests that load common scenarios and assert absence of errors in the log.

## Failure Recovery Checklist
- Validate prerequisites in `OnCreate` and log actionable errors when dependencies are missing.
- Use `TryGetSingleton` when reading data from other modules and fall back gracefully.
- Run expensive cache rebuilds inside the `Deserialize` phase to avoid mid-session stutters.

## Pre-Ship Test Plan
1. Run unit or integration tests if available (`dotnet test`).
2. Load high-crime, budget-collapse, and vice-escalation saves; monitor CPU time and logs.
3. Toggle the mod off/on in the mod manager to confirm systems unpatch cleanly.
4. Launch without shared dependencies (ExtraLib, I18n Everywhere) and confirm graceful degradation.
5. Open the map editor to ensure editor-only contexts do not execute simulation logic.

Follow these patterns to extend the simulation confidently while keeping performance predictable and the codebase easy to maintain.
