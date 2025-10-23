# I18n Everywhere Integration

I18n Everywhere is the localization backbone for Vice & Order. It mounts locale dictionaries directly into the Cities: Skylines II runtime, watches for JSON updates, and lets community translators ship fixes without us cutting a new release. Treat it as a required dependency for every playable module.

## Why We Rely On It
- Loads locale JSON from our `lang/` folders automatically; no custom loaders or asset bundles.
- Pulls weekly community updates from GitHub/ParatransZ so end users stay current.
- Exposes runtime tools (export, reload, key logging) that make QA and contributor workflows painless.
- Gives us a single contract across C# systems, React UI packages, and Write Everywhere overlays.

## Quick Start Checklist
- Declare `baka.I18NEverywhere` wherever the game checks dependencies (`mod.json`, `PublishConfiguration.xml`, release notes).
- Ship at least one complete `en-US.json` under `lang/` for each module and include the folder in your build output.
- Wrap user-facing strings in translation keys (Options UI, tooltips, notifications, UI React components).
- Provide a fallback path or warning if I18n Everywhere is missing so the mod remains usable in English.
- Document how translators can contribute (Crowdin, Discord threads, upstream localization repo pull requests).

## Step-by-Step Integration

### 1. Wire The Dependency
- **`mod.json`** – add `baka.I18NEverywhere` to the dependency list used by your template (`dependencies`, `modDependencies`, etc.). Mark it required so CS2 installs it automatically.
- **`Properties/PublishConfiguration.xml`** – add the Paradox Mods ID (`75426`) so publishing pulls I18n Everywhere as a prerequisite:

```xml
<Publish>
  ...
  <Dependency Id="75426" DisplayName="I18n Everywhere" />
</Publish>
```

- **Runtime guard (optional)** – if you still want to boot without the dependency (e.g., developer builds), check for the loaded assembly and downgrade to embedded English strings. Emit a single warning via the module logger so players know localization is limited.

### 2. Package Locale Bundles
1. Create a `lang/` folder next to the module assembly (`vno-core/lang`, `vno-vice/lang`, etc.).
2. Add locale JSON files named with BCP-47 tags (`en-US.json`, `fr-FR.json`, `zh-HANS.json`).
3. Ensure the folder is copied into the published mod:

```xml
<!-- In your <Project> file -->
<ItemGroup>
  <Content Include="lang\**\*.json">
    <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
  </Content>
</ItemGroup>
```

4. When the project builds, verify the output contains:  
`%AppData%/LocalLow/Colossal Order/Cities Skylines II/Mods/<ModuleId>/lang/<locale>.json`

I18n Everywhere scans that directory on load and merges dictionaries into the active locale.

### 3. Author Translation Keys
- **Prefix everything** with a stable namespace: `VNO.Core`, `VNO.Vice`, `VNO.UI.Campaigns`, etc. This prevents clashes with other mods and keeps exports tidy.
- **Follow vanilla categories** so the UI knows where to render strings:
  - `Options.SECTION[Namespace.Settings]` – section titles in the options menu.
  - `Options.GROUP[Namespace.Settings.General]` – group headers.
  - `Options.OPTION[Namespace.Settings.FlagName]` / `Options.OPTION_DESCRIPTION[...]` – setting labels and tooltips.
  - `Tools.INFO[...]`, `Tutorials.*`, `Notifications.*`, `UI.*` – match the vanilla surface you are extending.
- **Use `string.Format` tokens** (`{0}`, `{1:P0}`, `{2:N0}`) for variables; I18n Everywhere forwards them untouched so you can format via `Translate(..., args)`.

Example `lang/en-US.json`:

```json
{
  "Options.SECTION[VNO.Core.Settings]": "Vice & Order",
  "Options.GROUP[VNO.Core.Settings.General]": "Simulation",
  "Options.OPTION[VNO.Core.Settings.EnableViceLoop]": "Enable Vice loop",
  "Options.OPTION_DESCRIPTION[VNO.Core.Settings.EnableViceLoop]": "Turns on the systemic vice economy and police response loop.",
  "Notifications.TITLE[VNO.Order.SquadDispatched]": "Rapid response deployed",
  "Notifications.DESCRIPTION[VNO.Order.SquadDispatched]": "{0} squad en route to {1}."
}
```

Mirror that file for other locales and only include keys that are already populated in English—blank strings block UI rendering.

### 4. Consume Localized Strings In Code
- **Options UI / settings** – attributes take locale keys directly:

```csharp
[SettingsUISection("Options.SECTION[VNO.Core.Settings]", "Options.GROUP[VNO.Core.Settings.General]")]
[SettingsUICheckbox]
public bool EnableViceLoop { get; set; }
```

- **C# helpers** – add a tiny extension so gameplay systems can translate on demand while still functioning if the dictionary is absent:

```csharp
using Colossal.Localization;
using Game.SceneFlow;

namespace VNO.Core.Localization;

internal static class Locale
{
    private static LocalizationDictionary Active => GameManager.instance.localizationManager.activeDictionary;

    internal static string Translate(string key, params object[] args)
        => Active.TryGetValue(key, out var value)
            ? string.Format(value, args)
            : key;
}
```

Usage:

```csharp
var message = Locale.Translate("Notifications.DESCRIPTION[VNO.Order.SquadDispatched]", squadName, districtName);
```

- **React UI** – import `LocalizedString` from `cs2/l10n` and pass the same keys:

```tsx
import { LocalizedString } from "cs2/l10n";

export function SquadBanner({ squad, district }: Props) {
  return (
    <LocalizedString
      localeKey="Notifications.DESCRIPTION[VNO.Order.SquadDispatched]"
      args={[squad, district]}
    />
  );
}
```

I18n Everywhere keeps the dictionaries synchronized, so switching languages in-game updates both C# and UI surfaces immediately.

### 5. Central Packs & `i18n.json`
- To subscribe to the community pack maintained at <https://github.com/baka-gourd/I18NEverywhere.Localization>, point an `i18n.json` manifest at the folder structure you wish to consume. The manifest lives beside your DLL and lists one or more locale feeds (follow the upstream repo format for key names).
- When you need a custom pack (e.g., internal playtests), host the bundle yourself, add an entry in `i18n.json`, and keep the folder hierarchy identical to the upstream repository so translators can reuse their tooling.
- Always leave the embedded `lang/` folder in place—language packs are additive and provide overrides but the game falls back to your local copy if the pack is unavailable.

### 6. Contributor Workflow
- Document translation points in each module’s `Agents.md` so writers know which features to cover.
- Link the shared Crowdin project (or preferred tooling) through `<ExternalLink Type="crowdin" ... />` in `PublishConfiguration.xml`.
- Credit translators in `CHANGELOG.md` and Paradox Mods release notes and mention how to request string exports (`ModsData/<ModuleId>/Localization/` after using the I18n Everywhere export button).
- When accepting pull requests, run a quick JSON validation (`npm exec ajv` or similar) to ensure files remain UTF-8 and properly formatted.

### 7. QA & Troubleshooting
- Launch the game with `--developerMode --uiDeveloperMode` to get access to the I18n Everywhere diagnostics tab (reload dictionaries, export keys, toggle key logging).
- If strings fail to load, confirm the `lang/` directory shipped with the DLL and that filenames use the correct casing (`zh-HANS.json`, not `zh-Hans.json`).
- Missing translations show the raw key—scan the log for `[I18NE] Missing key:` entries to trace typos.
- Use the extension method above to guard gameplay text; returning the key name is preferable to throwing `KeyNotFoundException`.
- For regression testing, export the merged dictionary via I18n Everywhere and diff against previous builds to confirm no keys were dropped.

### 8. References & Further Reading
- Official wiki: [Localize your mod](https://cs2.paradoxwikis.com/Localize_your_mod)
- Community repo: <https://github.com/baka-gourd/I18NEverywhere.Localization>
- Vice & Order linkage: see `docs/cs2-modding-guide/options-attributes.md` for settings usage and `docs/vision/index.md` for tone constraints that affect copywriting.

Keeping these steps in sync ensures both humans and automation (agents, CI validators, translation bots) can reason about localization across the entire Vice & Order stack.
