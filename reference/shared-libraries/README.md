---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Shared libraries reference"
diataxis: reference
source_version: "n/a - concept/process page, no pinned source"
last_reverified: "2026-07-04"
status: needs-verification
Created: 2026-07-03
Updated: 2026-07-03
Owners:
  - codex
---

# Shared libraries

Reference pages for third-party libraries that CS2 mods commonly take a dependency on.
Declare each dependency by its numeric mod ID in `PublishConfiguration.xml` so the in-game
publisher installs it for players; see [dependency strategy](../../explanation/dependency-strategy.md)
for the detect-and-degrade pattern.

## Documented libraries

- [Unified Icon Library](unified-icon-library.md) - mod ID 74417. A shared icon set mounted at
  `coui://uil/...`; use it instead of shipping your own icons.
- [ExtraLib](extralib.md) - mod ID 75724. A shared runtime library (COUI icon host, notification
  UI, the ExtraPanels framework, localization helper) used by asset-import and detailing mods.

## Other libraries you may encounter

Some ecosystems ship their own shared "commons" assembly rather than depending on the libraries
above - for example Write Everywhere bundles its own `CS2-BelzontCommons` (Klyte commons) as a git
submodule for its reverse-patch bridge machinery. Those are documented in the relevant case study
([Write Everywhere ecosystem](../../case-studies/write-everywhere-ecosystem.md)) rather than here,
because they are not general-purpose player-facing dependencies. When you meet an unfamiliar shared
assembly, verify its mod ID and public API against its own source before depending on it.

## See also

- [Dependency strategy](../../explanation/dependency-strategy.md) - declaring, referencing, and degrading gracefully.
- [External resources](../external-resources.md) - official and community references.
- [Technique index](../../technique-index.md) - including COUI host registration (family H).
