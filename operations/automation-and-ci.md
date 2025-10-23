# Automation and CI

- Run `dotnet build`, `npm run build`, and linting tasks on every pull request.
- Validate YAML/JSON policy packs and module manifests against a schema to catch missing fields early.
- Sync official wiki snapshots periodically (`docs/research/wiki/`) and highlight game updates that affect the guide.
- Compare dependency declarations (e.g., `PublishConfiguration.xml`, `mod.json`) against actual project references.
- Execute telemetry smoke tests that load scripted cities and assert log error thresholds.
