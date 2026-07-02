# Realistic Path Finding

- Replaces the resident AI stack with custom systems and disables `Game.Simulation.ResidentAISystem` before registering replacements via `UpdateAt`/`UpdateAfter`.
- Provides deep slider coverage grouped by transport mode using `[SettingsUIGroupOrder]` and localised dictionaries.
- Logs Harmony patches and detected companion mods (for example `Time2Work`) to simplify troubleshooting.
- Shares interop helpers so other mods can query real-time parameters such as wait-time factors.
- **Takeaway:** follow this pattern when swapping whole system families while keeping options approachable.
