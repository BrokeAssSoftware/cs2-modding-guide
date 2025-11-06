# I18n Everywhere Integration

I18n Everywhere is the localisation backbone for Vice & Order. It mounts locale dictionaries directly into the Cities: Skylines II runtime, watches for JSON updates, and lets community translators ship fixes without us cutting a new release.

## Quick Start Checklist
- Declare `baka.I18NEverywhere` as a dependency in every module that uses shared localisation.
- Ship at least one complete `en-US.json` under `lang/` and make sure MSBuild copies the folder into the deploy directory.
- Wrap user-facing strings in translation keys (Options UI, tooltips, notifications, UI React components).
- Provide a fallback path when the dependency is missing so the mod remains usable in English.
- Document how translators can contribute (Crowdin, Discord, upstream localisation repository).

## Wire the Dependency
- **`mod.json`** - add `baka.I18NEverywhere` to the dependencies array.
- **`Properties/PublishConfiguration.xml`** - add the Paradox Mods ID (`75426`) so publishing installs I18n Everywhere automatically.
- **Runtime guard (optional)** - if you want to boot without the dependency (for example developer builds), detect the assembly and fall back to embedded English strings with a single warning.

## Package Locale Bundles
1. Create a `lang/` folder next to the module assembly.
2. Add locale JSON files named by BCP-47 tags (`en-US.json`, `fr-FR.json`, `zh-HANS.json`).
3. Copy the folder during build:
   ```xml
   <ItemGroup>
     <Content Include="lang\**\*.json">
       <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
     </Content>
   </ItemGroup>
   ```
4. Verify the output contains `%AppData%/LocalLow/Colossal Order/Cities Skylines II/Mods/<ModuleId>/lang/<locale>.json` after build.

I18n Everywhere scans the directory on load and merges dictionaries into the active locale.

## Author Translation Keys
- Prefix every key with a stable namespace (`VNO.Core`, `VNO.Vice`, `VNO.UI.Campaigns`).
- Follow vanilla categories so the UI knows where to render strings:
  - `Options.SECTION[Namespace.Settings]` - section titles
  - `Options.GROUP[Namespace.Settings.General]` - group headers
  - `Options.OPTION[Namespace.Settings.FlagName]` / `Options.OPTION_DESCRIPTION[...]` - option labels and descriptions
  - `Tools.INFO[...]`, `Tutorials.*`, `Notifications.*`, `UI.*` - match the vanilla surface you are extending
- Use `string.Format` tokens (`{0}`, `{1:P0}`, `{2:N0}`) for variables; I18n Everywhere forwards them untouched.

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
Mirror the file for other locales and only include keys that already have English copy.

## Consume Localised Strings
- **Options UI** - attributes accept locale keys directly:
  ```csharp
  [SettingsUISection("Options.SECTION[VNO.Core.Settings]", "Options.GROUP[VNO.Core.Settings.General]")]
  [SettingsUICheckbox]
  public bool EnableViceLoop { get; set; }
  ```
- **C# helpers** - add a helper so gameplay systems can translate on demand:
  ```csharp
  internal static class LocaleHelper
  {
      private static LocalizationDictionary Active => GameManager.instance.localizationManager.activeDictionary;

      internal static string Translate(string key, params object[] args)
          => Active.TryGetValue(key, out var value)
              ? string.Format(value, args)
              : key;
  }
  ```
- **React UI** - import `LocalizedString` from `cs2/l10n`:
  ```tsx
  <LocalizedString
    localeKey="Notifications.DESCRIPTION[VNO.Order.SquadDispatched]"
    args={[squad, district]}
  />
  ```

## Language Packs and `i18n.json`
- To subscribe to the community pack, add an `i18n.json` manifest that points to the repositories or folders you want to consume. Follow the upstream folder structure so translators can reuse their tooling.
- Keep the embedded `lang/` folder even when using external packs; language packs are additive and provide overrides when available.

## Contributor Workflow
- Document translation touchpoints in each module's `Agents.md` so writers know what to cover.
- Link the shared Crowdin project (or preferred tool) through `<ExternalLink Type="crowdin" ... />` in `PublishConfiguration.xml`.
- Credit translators in changelogs and note where string exports are stored (`ModsData/<ModuleId>/Localization/`).
- When accepting pull requests, validate JSON formatting (for example `npm exec ajv`) to avoid corrupted files.

## QA and Troubleshooting
- Launch the game with `-developerMode -uiDeveloperMode` to access the I18n Everywhere diagnostics tab (reload dictionaries, export keys, toggle key logging).
- Confirm the `lang/` directory ships with the DLL and filenames use correct casing (`zh-HANS.json`, not `zh-Hans.json`).
- Missing translations show the raw key; scan logs for `[I18NE] Missing key:` entries to trace typos.
- Guard gameplay text with the helper above so missing keys return the key name instead of throwing.
- Export the merged dictionary via I18n Everywhere and diff against previous builds to confirm no keys were dropped.

## References
- Official wiki: <https://cs2.paradoxwikis.com/Localize_your_mod>
- Community localisation repo: <https://github.com/baka-gourd/I18NEverywhere.Localization>
- Options attribute guide: [Options Attribute Reference](../ui/reference/options-attributes.md)

With these steps in place, both humans and automation can keep localisation consistent across the entire Vice & Order stack.


