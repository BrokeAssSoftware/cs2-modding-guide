# Shared Library Pattern (ExtraLib)

Reference mod: `ExtraLib`.

When multiple mods share UI components, localisation helpers, or utility code, ship a dedicated dependency library so maintenance stays centralised.

## 1. Treat The Library As A Mod

- Publish the library as its own code mod with a stable ID and version number.
- Keep the README focused on compatibility, contributors, and dependency instructions rather than end-user features.
- Provide translation coverage for the library UI (if any) via a shared localisation platform (ExtraLib uses Crowdin).

## 2. Expose A Clean API Surface

- Group reusable helpers (UI widgets, asset management, common services) under clear namespaces.
- Avoid leaking implementation details from consumer mods. Instead, provide extension points or service interfaces.
- Document any required initialisation order so dependent mods can call your setup routines at the right time.

## 3. Version And Communicate Changes

- Adopt semantic versioning and publish release notes whenever you add or change APIs.
- Add defensive checks: if a dependent mod loads an incompatible version, log a descriptive error and disable only the affected features.

## 4. Simplify Dependency Installation

- Mention the dependency explicitly in each consumer mod's `mod.json` and README.
- In the dependent mod's `OnLoad`, verify that the library is installed; if not, display a user-facing warning so the player knows what to download.

## 5. Share UI And Graphics Assets

- Store shared thumbnails, icons, and other UI resources in the library. This reduces duplicated bundles across dependent mods and keeps visual styling consistent.
- Credit designers (ExtraLib thanks CityRat for thumbnails) and keep assets under compatible licences.

