# Project Architecture

Vice & Order modules follow the standard CS2 code mod layout, expanded with shared patterns from the researched mods.

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
