# System Registration Pattern

```csharp
updateSystem.UpdateAt<MySimulationSystem>(SystemUpdatePhase.GameSimulation);
updateSystem.UpdateAt<MySimulationSystem>(SystemUpdatePhase.Deserialize);
updateSystem.UpdateAt<MyUISystem>(SystemUpdatePhase.UIUpdate);
```

**Highlights**
- Registering the same system in multiple phases keeps data consistent during load and gameplay.
- Ensure `OnUpdate` checks the current phase or context so side effects run only when appropriate.
- Use `UpdateAfter` / `UpdateBefore` when ordering relative to other systems is critical.
