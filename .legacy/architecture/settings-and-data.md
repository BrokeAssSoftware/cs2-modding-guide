# Settings and Data Management

Define clear boundaries between configuration, localisation, logging, and runtime data so modules stay maintainable.

## Settings
- Derive from `ModSetting`, store user preferences only, and persist changes with `AssetDatabase.global.SaveSettings`.
- Keep runtime caches in dedicated services or ECS systems rather than the settings file.
- Provide reset actions in the Options UI and confirm destructive changes when appropriate.

## Localisation
- Register `IDictionarySource` implementations per language before `RegisterInOptionsUI` so labels resolve immediately.
- Supply an English fallback before other locales and let I18n Everywhere override keys via JSON when available.
- Keep localisation keys consistent with your `Setting` class using `GetOptionLabelLocaleID` and related helpers.

## Logging
- Create a single static logger per module using `LogManager.GetLogger` and disable UI error popups by default.
- Log executable asset paths, dependency versions, and important feature flags during load to aid support and automated checks.

## Data Storage
- Use `ModsData/<Module>` for long-lived caches and analytics that persist across sessions.
- Use `ModsDataTemp/<Module>` for transient data. Clear the directory on unload so temporary files do not linger.
- Document JSON or YAML schema under `docs/modules/` whenever you ship data packs so CI can validate them.
