# Asset Pack Management

Reference mod: `CS2-AssetPacksManager`.

If you plan to distribute large collections of assets (buildings, props, surfaces) outside official playsets, provide tooling that helps players understand what is installed and what is missing.

## 1. Mirror Playset Awareness

- Enumerate the player's active playsets and list which packs come from Paradox Mods versus local folders.
- Highlight discrepancies (pack enabled in playset but missing locally, or vice versa) so users can resolve conflicts quickly.

## 2. Provide Metadata Fallbacks

- Many legacy packs lack thumbnails or localisation. Generate thumbnails on the fly or fall back to generic images, and allow community translations for pack names/descriptions.
- Cache the generated assets so the UI remains responsive during browsing.

## 3. Diagnose Broken Assets

- Offer a reporting tool that scans packs for missing dependencies, outdated references, or corrupt files.
- Export the report to disk so players can share it when requesting support.
- Surface warnings inline (e.g., red badge next to a pack) so issues are visible without opening the report.

## 4. Keep Local Assets Optional

- Warn players that locally installed assets may not receive updates and might bypass Skyve or Galaxy-style validation.
- Encourage creators to publish through the Paradox toolchain whenever possible, while still supporting local packs for advanced users.

## 5. Integrate With Community Support

- Link to the relevant Discord channels or forums where players can get help (`#apm-general`, Cities: Skylines Modding Discord).
- Maintain a concise README that explains how to install new packs, how to refresh metadata, and how to submit bug reports.

