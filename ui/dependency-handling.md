# UI Dependency Handling

- Detect missing Unified Icon Library or I18n Everywhere during `OnLoad` and set feature flags so the UI can downgrade gracefully.
- Display warnings or guidance inside the Options UI when a dependency is unavailable and link to installation instructions where possible.
- Default to text labels or neutral icons if shared assets are missing so players can continue using core features.
