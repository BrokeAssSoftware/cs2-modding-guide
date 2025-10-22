# Project Architecture

Vice & Order modules follow the standard CS2 code mod layout, expanded with shared patterns from the researched mods.  
This guide now also tracks shared performance budgets, terminology definitions, and cross-module API contracts referenced by the backlog.

## Core Classes

- `Mod` implements `IMod`; use `OnLoad(UpdateSystem updateSystem)` to register systems, settings, and localization, and `OnDispose()` to unregister.
- Instantiate settings early: `m_Settings = new Setting(this); m_Settings.RegisterInOptionsUI();`.
- Log the active executable asset via `modManager.TryGetExecutableAsset` to help support track deployment paths.
- Disable vanilla systems or UI controllers before adding replacements to avoid double execution.

## Settings Pattern

- Derive from `ModSetting`, annotate with `[FileLocation("ModsSettings/<Mod>/<Mod>")]` and call `AssetDatabase.global.LoadSettings` to hydrate values.
- Use `[SettingsUIGroupOrder]`, `[SettingsUIShowGroupName]`, and `[SettingsUISection]` to organize complex options (see `RealisticPathFinding`).
- Provide sane defaults in `SetDefaults()` and expose a `[SettingsUIButton]` reset action.
- Keep runtime state out of the settings file; write simulation caches to `ModsData/<Mod>` or `ModsDataTemp/<Mod>` instead.

## Localization Flow

- Register locale sources (`GameManager.instance.localizationManager.AddSource`) before the settings UI loads so labels resolve on first run.
- Supply `GetOptionLabelLocaleID`, `GetOptionDescLocaleID`, and `GetOptionGroupLocaleID` mappings in dedicated locale classes.
- Listen for `localizationManager.onActiveDictionaryChanged` when override strings must refresh after a language switch (pattern from `AchievementFixer`).

## Logging & Diagnostics

- Create module loggers via `LogManager.GetLogger("Namespace.Mod")` and call `SetShowsErrorsInUI(false)` to keep the notification feed clean.
- Emit Harmony patch inventories after `PatchAll` to expose conflicts (`GetPatchedMethods()` listing).
- Gate verbose logs behind conditional compilation (`#if DEBUG`) to avoid runtime overhead in release builds.
- Record detected companion mods or incompatible versions at load time; expose helper methods (`Time2WorkInterop.GetFactor`) for cross-mod coordination.

## Folder Conventions

- Resolve paths through `EnvPath` (`EnvPath.kUserDataPath`) to stay cross-platform.
- Lazily create `ModsSettings/<Mod>` and `ModsData/<Mod>` folders if they are missing; prefer `Directory.CreateDirectory` during `OnLoad`.
- Store per-session output (diagnostic dumps) under `ModsDataTemp` and purge on unload.

## Dependency Management

- Keep references to Harmony, DOTS packages, and shared utilities in a `Directory.Packages.props` or `Directory.Build.props` shared across modules.
- When disabling vanilla systems, store the resulting `SystemHandle` or access via `World.DefaultGameObjectInjectionWorld.GetOrCreateSystemManaged<>()` and set `Enabled = false`.
- Use `UpdateSystem.UpdateAt` / `UpdateAfter` / `UpdateBefore` with explicit `SystemUpdatePhase` enums to guarantee deterministic ordering.
- Unpatch Harmony hooks during `OnDispose` or module shutdown to support hot reloads.
- Treat external mod libraries (ExtraLib, Unified Icon Library, I18n Everywhere) as first-class dependencies: declare them in `mod.json`, check presence during `OnLoad`, and provide fallback behaviour when missing.

## Shared Performance Targets

Backlog acceptance criteria reference the following baseline hardware and budgets unless otherwise noted:

- **Reference hardware:** Intel i7-11700K (or equivalent Ryzen 7 5800X), NVIDIA RTX 3070, 32 GB RAM, 1440p, Medium graphics preset.
- **Frame budget targets:** Simulation updates ≤ 5 ms per frame; UI updates ≤ 2 ms per frame; background analytics ≤ 1 ms per frame.
- **Stability definition:** “No frame hitches” means no dropped frames > 16 ms over a one-minute simulated period.
- **Benchmark saves:** Use the shared regression seeds listed in `plan/ep-core/feature-deterministic-simulation-loop.md` for deterministic profiling.

Stories should reference this section (`See Docs → Project Architecture → Shared Performance Targets`) when validating performance criteria.

## Terminology & Schema Glossary

To keep cross-module language consistent, backlog stories should link here when introducing the following concepts:

- **Identity Triangle:** `(loyalty, fear, opportunity)` scalar values in the range `0.0 – 1.0`, defaults `0.5`. Serialized in `vno_economy.identity` payloads.
- **Ideology Vector:** `(reformist, developer, lawAndOrder, unionist, populist, viceAligned)` array of floats `0.0 – 1.0`, normalized to sum ≤ 1.0.
- **Influence Metrics:** Accumulated action points per faction, stored as `float current`, `float decayRate`, `float maxCapacity`.
- **Heat Index:** Normalized scalar `0.0 – 100.0`, updated per laundering cycle; thresholds at `25`, `50`, `75` trigger escalation tiers.
- **Legitimacy & Trust:** Values `0.0 – 100.0` persisted in `vno_order.legitimacy` / `vno_governance.trust`.

Add new terms here when they first appear in design discussions to avoid ambiguity in future stories.

## Event & API Reference Stubs

Stories that introduce or require specific contracts should link to this table as definitions evolve:

| Contract | Summary | Status |
| --- | --- | --- |
| `HeatChangedEvent` | `{ factionId: Guid, previous: float, current: float, delta: float, timestamp: long }` | Draft |
| `GangEvent` | `{ type: enum, territoryId: Guid, actors: Guid[], loyaltyDelta: float, liquidityDelta: float }` | Draft |
| `IFinanceService` | Methods: `GetBalances(factionId)`, `InitiateLaunder(job)`, `SetAutoPolicy(policyId, enabled)` | Draft |
| `IVoteService` | Methods: `ScheduleSession(config)`, `GetForecast(sessionId)`, `SubmitInfluence(action)` | Draft |
| `IHealthService` | Methods: `GetStress(districtId)`, `RegisterProgram(programConfig)`, `ReportOutcome(outcome)` | Draft |

As implementation matures, promote draft entries into `docs/API_REFERENCE.md` and update references accordingly.
