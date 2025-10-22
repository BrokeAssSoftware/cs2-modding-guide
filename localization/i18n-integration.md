# I18n Everywhere Integration

Reference mod: `I18NEveryWhere`.

I18n Everywhere simplifies localisation for mods that lack their own pipeline. Integrate it to let translators contribute without touching your code.

## 1. Add The Dependency

- Declare I18n Everywhere in your `mod.json`.
- On load, confirm the dependency is present. If not, warn the player and fall back to English strings so the mod still functions.

## 2. Ship Embedded Locale Files

- Create a `lang/` folder alongside your assembly.
- Place JSON files named by locale (`en-US.json`, `zh-Hans.json`, etc.) inside the folder.
- I18n Everywhere automatically reads and registers these files, removing the need for custom loaders.

## 3. Support Centralised Packs

- Encourage translators to submit improvements via the community repositories (GitHub, Discord, ParatransZ). Updates are bundled into regular releases of I18n Everywhere so your mod benefits without publishing a new version.

## 4. Publish Custom Language Packs

- If you need total control, include an `i18n.json` manifest that points to your own locale bundle.
- List I18n Everywhere as a dependency so the manifest is processed at runtime.
- Follow the same folder structure as the upstream localisation repo to keep contributions compatible.

## 5. Keep Contributors In The Loop

- Credit translators in your README and changelog.
- Mention that strings are hosted on a shared Crowdin project, so prospective contributors know where to help.
