# Localization & UI Infrastructure

- Unified Icon Library preloads SVG icon sets by style (`coui://uil/<Style>/<Icon>.svg`) so dependent mods reuse assets without bundling copies.
- I18n Everywhere loads embedded JSON locale files, supports community packs, and enables optional language packs via `i18n.json`.
- **Takeaway:** standardise UI assets and translations by delegating to shared dependency mods, simplifying module UIs and cross-mod coordination.
