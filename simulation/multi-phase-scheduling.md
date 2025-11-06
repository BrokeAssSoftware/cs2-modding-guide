# Multi-Phase Scheduling

Register systems against multiple phases when behaviour spans load, gameplay, or UI contexts.

```csharp
updateSystem.UpdateAt<VnoHeatCacheSystem>(SystemUpdatePhase.Deserialize);
updateSystem.UpdateAt<VnoHeatCacheSystem>(SystemUpdatePhase.GameSimulation);
updateSystem.UpdateAt<VnoHeatOverlay>(SystemUpdatePhase.UIUpdate);
```

Inside `OnUpdate`, branch based on the current context:
```csharp
protected override void OnUpdate()
{
    if (SystemAPI.TryGetSingleton(out DeserializeInProgress tag) && tag.value)
    {
        RebuildCaches();
        return;
    }

    if (!Application.isPlaying) return;
    TickSimulation(Time.DeltaTime);
}
```

Use `UpdateAfter` / `UpdateBefore` to order systems relative to other modules when sequence matters.
