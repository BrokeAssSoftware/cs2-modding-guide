# Testing and Recovery

- Validate prerequisites in `OnCreate` and log actionable errors when dependencies are missing.
- Use `TryGetSingleton` when reading data produced by other modules and fall back gracefully when absent.
- Offload expensive cache rebuilds to the `Deserialize` phase to avoid mid-session stutters.
- Maintain regression saves (high-crime, budget-collapse, vice escalation) and profile them before shipping features.
- Before release:
  1. Run unit or integration tests (`dotnet test`) if available.
  2. Load regression saves and watch CPU time and logs.
  3. Toggle the mod off/on to confirm systems unpatch cleanly.
  4. Launch without shared dependencies (ExtraLib, I18n Everywhere) and confirm graceful degradation.
  5. Open the map editor to ensure editor-only contexts do not execute runtime logic.
