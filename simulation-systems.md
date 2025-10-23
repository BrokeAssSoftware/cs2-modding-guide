# Simulation Systems

Cities: Skylines II exposes most gameplay through Unity DOTS systems. Vice & Order modules replace or extend these systems to introduce new mechanics while keeping performance predictable. This guide documents the end-to-end workflow: disabling vanilla systems, wiring custom systems into the update loop, coordinating with Harmony patches, and testing safely.

## Update Phases at a Glance
| Phase | When it Runs | Typical Use |
| --- | --- | --- |
| `GameSimulation` | Every frame during live gameplay | AI, economics, policing, vice loops |
| `EditorSimulation` | While the map/editor is open | Authoring tools, preview logic |
| `UIUpdate` | Each UI frame | Push data into Gameface bindings |
| `Deserialize` | Immediately after loading a save | Rehydrate caches or rebuild derived data |
| `PrefabUpdate` | During asset compilation | Bake prefab metadata, register custom prefabs |

Always schedule systems explicitly with `updateSystem.UpdateAt` and use `UpdateAfter` / `UpdateBefore` to make dependencies obvious. Avoid relying on implicit DOTS ordering.

## Replacing Vanilla Systems
1. Grab the existing system and disable it:
   ```csharp
   var world = World.DefaultGameObjectInjectionWorld;
   world.GetOrCreateSystemManaged<Game.Simulation.ResidentAISystem>().Enabled = false;
   ```
2. Register your replacement:
   ```csharp
   updateSystem.UpdateAt<VnoResidentAISystem>(SystemUpdatePhase.GameSimulation);
   updateSystem.UpdateAfter<VnoResidentAISystem, Game.Simulation.StatisticSystem>(SystemUpdatePhase.GameSimulation);
   ```
3. Keep scope tight. For example, Realistic Path Finding only disables the resident AI stack, leaving unrelated transport systems untouched. Document each vanilla dependency you disable so future maintainers understand the impact.

## Writing a SystemBase Safely
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
**Key practices**
- Call `RequireForUpdate<T>()` to prevent the system from running when required components are absent.
- Cache queries and lookups in `OnCreate` and refresh them inside `OnUpdate` with `.Update(this)`.
- Use `ScheduleParallel()` when you are not making structural changes; switch to `Schedule()` or `Run()` only when necessary.
- Guard heavy side effects with settings flags so optional features cost nothing when disabled.

## Multi-Phase Participation
Some systems need to run in multiple phases (for example, to rebuild caches immediately after loading a save and again during normal gameplay):
```csharp
updateSystem.UpdateAt<VnoHeatCacheSystem>(SystemUpdatePhase.Deserialize);
updateSystem.UpdateAt<VnoHeatCacheSystem>(SystemUpdatePhase.GameSimulation);
```
Inside `OnUpdate`, branch depending on the current phase:
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
This keeps deserialization work isolated from regular per-frame logic.

## Conditional Scheduling
- Gate optional systems behind configuration checks before calling `UpdateAt`. This avoids allocating world handles you never intend to use.
- If the feature can be toggled at runtime, register the system but disable its logic internally (for example by checking a setting each frame and returning early).
- Use lightweight components (`EnabledRefRW`) or shared singleton flags to toggle specific behaviours without rebuilding worlds.

## Harmony and Systems
Harmony patches complement DOTS systems when you need to intercept high-level game API calls:
- Patch entry points such as `CitizenDestinationSystem.FindDestination` to adjust inputs or parameters before your DOTS system consumes them.
- Keep patches minimal and log when they execute so conflicts are easier to spot.
- Always unpatch in `OnDispose` (and on hot reload) using the same Harmony ID you used during load.

## Diagnostics and Testing Hooks
- Expose developer commands (just-in-time toggles, dumps) behind `--developerMode`. Example: `heat.dump` to write current heat indices to `ModsDataTemp`.
- Wrap hot code paths in `ProfilerMarker` scopes and capture traces on regression saves.
- Provide short-lived diagnostic systems that disable themselves after use. Achievement Fixer runs for ~300 frames to flip achievement flags and then idles.

## Failure Recovery Patterns
- Validate prerequisites in `OnCreate` and log actionable errors (missing components, absent dependencies). Fail fast so QA knows which module to inspect.
- Use `TryGetSingleton` when depending on data provided by other VNO modules. If the dependency is missing, log once and degrade gracefully.
- For expensive operations triggered by save load, run them inside the `Deserialize` phase so players do not experience hitches later in gameplay.

## Test Checklist Before Shipping a System
1. **Unit smoke test** – run `dotnet test` or targeted integration tests if available.
2. **Regression saves** – open the high-crime, budget-collapse, and vice-escalation saves to profile CPU time and watch for errors.
3. **Hot reload** – reload the mod (disable/enable) to ensure systems unpatch cleanly and settings survive.
4. **Dependency failure** – launch without ExtraLib / I18n Everywhere and confirm your systems log a warning but continue running with fallbacks.
5. **Editor session** – open the map editor to confirm systems that should opt out of editor contexts remain idle.

With these patterns, every Vice & Order module can extend the simulation confidently while remaining performant and resilient to future game updates.
