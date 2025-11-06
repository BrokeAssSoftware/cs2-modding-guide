# System Scheduling

Disable vanilla systems carefully and schedule custom systems with explicit update phases so behaviour stays deterministic.

## Replacing Vanilla Systems
1. Fetch the existing system and set `Enabled = false` before you register replacements.
2. Keep the disabled system reference if you might re-enable it later (for example in debug builds).
3. Document each override so future maintainers understand the scope of the replacement.

## Register Custom Systems
- Use `updateSystem.UpdateAt<T>(phase)` with explicit `SystemUpdatePhase` values (`GameSimulation`, `UIUpdate`, `EditorSimulation`, `Deserialize`, `PrefabUpdate`).
- Chain dependencies with `updateSystem.UpdateAfter<TDependency, TSystem>()` or `UpdateBefore` to make ordering explicit.
- For multi-phase behaviour, register the same system in multiple phases and branch inside `OnUpdate` based on context (deserialisation, editor, runtime).

## Caching and Lookups
- Cache `EntityQuery`, `ComponentLookup`, and `SystemHandle` instances in `OnCreate` and refresh them inside `OnUpdate` to avoid repeated lookups.
- Use `RequireForUpdate<TComponent>()` so systems do not execute when prerequisites are missing.

## Diagnostics
- Wrap heavy code paths in `ProfilerMarker` scopes.
- Expose developer-only commands behind `-developerMode` to toggle diagnostics or dump ECS state.

These practices keep system execution predictable and make multi-module scheduling easier to reason about.
