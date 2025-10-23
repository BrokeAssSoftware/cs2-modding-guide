# Dependency Strategy

Vice & Order relies on shared libraries such as ExtraLib, Unified Icon Library, I18n Everywhere, and Write Everywhere. Coordinate these dependencies so modules degrade gracefully when they are missing.

## Declaring Dependencies
- For code mods, add `<Dependency Id="..." DisplayName="..." />` entries to `PublishConfiguration.xml` so the in-game publisher installs prerequisites automatically.
- For UI mods, list dependencies in `mod.json` under `dependencies`.
- Mention required mods in READMEs and release notes to reduce support churn.

## Runtime Guards
- Detect dependency assemblies during `OnLoad` and set boolean flags (`HasExtraLib`, `HasUIL`, `HasI18n`) so systems can downgrade gracefully.
- Downgrade to text labels or default behaviour when a dependency is absent and log a single warning.

## Shared Wrappers
- Place common helpers (icon hosts, prefab loaders, notification utilities) in shared libraries such as ExtraLib instead of duplicating code across modules.
- Document new patterns in module-specific `Agents.md` files so other teams can follow them.

## Inter-Mod Hooks
- When integrating with third-party mods (for example Time2Work), create wrapper services that guard against missing assemblies and provide typed access to their APIs.

## Dependency Hygiene
- Keep third-party DLLs in `dependencies/<Vendor>/<Version>/` and never commit them under `bin/` or `obj/`.
- Version dependencies explicitly and avoid floating to "latest" without coordinated testing across modules.
- Use CI checks to ensure `PublishConfiguration.xml`, `mod.json`, and documentation stay aligned with actual dependencies.

