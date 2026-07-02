# Time2Work (Realistic Trips)

- Disables numerous vanilla simulation and UI systems, installing replacements across `GameSimulation`, `UIUpdate`, `EditorSimulation`, and `Deserialize`.
- Splits configuration across `ModsSettings` (user options) and `ModsData` (runtime state) for clarity.
- Registers multiple locale sources and exposes utility methods (time-scaling factors) to partner mods.
- Unpatches Harmony hooks on dispose to support hot reload.
- **Takeaway:** use this as the template for multi-phase, full-stack modules that touch both simulation and UI.
