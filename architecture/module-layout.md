# Module Layout

Keep module structure predictable so shared tooling and automation can operate across the repository without special cases.

## Naming
- Use the same identifier for the assembly, namespace root, and published mod ID (for example `vno-core`).
- Mirror the published ID in `mod.json`, `PublishConfiguration.xml`, and release notes.

## Folder Structure
```
vno-module/
  Mod.cs
  Setting.cs
  Systems/
  Services/
  UI/
  Localization/
  Properties/PublishConfiguration.xml
  Icons/ (optional, shared icon host assets)
  lang/ (embedded locale files for I18n Everywhere)
```

## Shared MSBuild Configuration
- Reference shared dependencies (ExtraLib, UIL, Harmony) via `Directory.Build.props` so every module inherits the same configuration.
- Keep Unity references outside project folders; point to the toolchain install path instead of copying DLLs locally.

## Environment Paths
- Resolve paths with `EnvPath` or toolchain helpers. Avoid hard-coded absolute paths so modules run on Steam, Xbox / PC Game Pass, and developer installs without changes.
