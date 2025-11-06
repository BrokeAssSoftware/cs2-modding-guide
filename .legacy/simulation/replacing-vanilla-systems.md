# Replacing Vanilla Systems

1. Fetch the vanilla system and set `Enabled = false` before registering your replacement.
2. Keep a reference if you might re-enable the vanilla system (for debugging or fallback behaviour).
3. Document each system you disable so future contributors understand the scope of the override.

Example:
```csharp
var world = World.DefaultGameObjectInjectionWorld;
world.GetOrCreateSystemManaged<Game.Simulation.ResidentAISystem>().Enabled = false;

updateSystem.UpdateAt<VnoResidentAISystem>(SystemUpdatePhase.GameSimulation);
updateSystem.UpdateAfter<VnoResidentAISystem, Game.Simulation.StatisticSystem>(SystemUpdatePhase.GameSimulation);
```

This pattern mirrors Realistic Path Finding and other full-stack replacements where a custom system takes ownership of an entire simulation slice.
