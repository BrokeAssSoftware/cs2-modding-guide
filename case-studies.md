# Case Studies

Field notes from community mods illustrate patterns Vice & Order can adopt.

## Realistic Path Finding

- Replaces the resident AI stack with custom systems; disables `Game.Simulation.ResidentAISystem` and registers new schedulers via `UpdateAt`/`UpdateAfter`.
- Provides deep slider coverage grouped by transport mode; uses `[SettingsUIGroupOrder]` and localized dictionaries per group.
- Logs Harmony patches and detected companion mods (`Time2Work`) to simplify troubleshooting.
- Shares interop helpers so other mods can query real-time parameters (wait-time factors).
- Takeaway: follow this pattern when swapping whole system families while keeping options approachable.

## Achievement Fixer

- Adds a short-lived `GameSystemBase` that runs for ~300 frames after load to flip `achievementsEnabled` back on.
- Hooks localization changes to reapply option overrides and custom banner strings.
- Keeps settings and static state lightweight; the system idles when not needed.
- Takeaway: implement diagnostic or safeguard systems with tight execution windows and full localization coverage.

## Time2Work (Realistic Trips)

- Disables numerous vanilla simulation and UI systems, installing replacements across `GameSimulation`, `UIUpdate`, `EditorSimulation`, and `Deserialize`.
- Splits configuration across `ModsSettings` (user options) and `ModsData` (runtime state) for clarity.
- Registers multiple locale sources and exposes utility methods (time-scaling factor) to partner mods.
- Unpatches Harmony hooks on dispose to support hot reload.
- Takeaway: use this as the template for Vice & Order's multi-phase, full-stack modules that touch both simulation and UI.
