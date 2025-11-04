# Runtime UI Panels

- Use `[SettingsUIMultilineText]` to present read-only summaries (status, telemetry) within the Options UI.
- Build Gameface React dashboards for dynamic content by reading ECS buffers or service APIs.
- Use `[SettingsUIDirPicker]` and `[SettingsUIFilePicker]` for file system interactions; validate paths before saving.
- Keep React components lightweight and align them with the patterns described in the [React Pipeline](react-pipeline/overview.md).
- Verify layouts against the [Gameface Runtime Basics](gameface-basics.md) checklist and clear any parser warnings from `UI.log` before shipping UI updates.
