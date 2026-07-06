---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: Localization helper (multi-locale)"
recipe: localization-helper
technique_family: "O - Localization multi-locale registration"
diataxis: how-to
source_version: "1.6.0f1 (magic-mail@6fb3d2b; date-pinned, static source only)"
last_reverified: "2026-07-04"
canonical_mods:
  - magic-mail@6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47
  - anarchy@a6311e898d20a775368668b234aaa32f06e3e1eb
  - i18n-everywhere@d9285c2490079c6d67303da536207b47b106ce64
  - achievement-fixer@4d3a1eccc64a7a09de65a23f6abbb3daa998fd0d
  - custom-chirps@f018ac382e93e0b56cd7b974d1ce0b155d23d2b4
  - specialized-industrial-zones@8d9153c3fda72e3b79a133bee6c83f9ba7df8b5a
technique_applicability: [media, ui]
status: source-verified
Created: 2026-07-03
Updated: 2026-07-05
Owners:
  - codex
---

# Localization helper (multi-locale)

> Give every settings label, tab, tooltip, and UI string a translated value by
> implementing `IDictionarySource` and registering one source per locale through
> `GameManager.instance.localizationManager.AddSource(localeId, source)`.

## Problem
Your mod's Options page and UI show raw locale-ID keys
(`Options.OPTION[MyMod...]`) instead of readable text, and you want it to display
correctly in English plus the other languages the game supports. CS2 resolves every
on-screen string through a localization dictionary keyed by locale ID; until you feed
it key -> text pairs for your keys, there is nothing to show. You need a place to
author those pairs and a way to hand them to the game for each language.

## Solution
Write a class implementing `Colossal.IDictionarySource`. Its `ReadEntries(...)` returns
`KeyValuePair<string,string>` entries: the key is a locale ID generated from your
settings object (`GetSettingsLocaleID`, `GetOptionLabelLocaleID`,
`GetOptionDescLocaleID`, `GetOptionTabLocaleID`, `GetOptionGroupLocaleID`,
`GetEnumValueLocaleID`, ...), and the value is the translated text. In `IMod.OnLoad`,
call `localizationManager.AddSource("en-US", new LocaleEN(setting))` once per locale.
Deriving keys from the settings object keeps labels in lockstep with property names, so
a rename can't silently orphan a translation. For many languages, either write one
`IDictionarySource` subclass per locale, or author en-US in code and load the rest from
embedded JSON.

## Steps & Code

### 1. Implement `IDictionarySource` for your base locale

Hold a reference to the settings object so keys derive from real property names. Return
the pairs from `ReadEntries`; leave `Unload()` empty for static dictionaries:

```csharp
public sealed class LocaleEN : IDictionarySource
{
    private readonly Setting m_Setting;
    public LocaleEN(Setting setting) { m_Setting = setting; }

    public IEnumerable<KeyValuePair<string, string>> ReadEntries(
        IList<IDictionaryEntryError> errors, Dictionary<string, int> indexCounts)
    {
        return new Dictionary<string, string>
        {
            { m_Setting.GetSettingsLocaleID(), "Magic Mail [MM]" },
            { m_Setting.GetOptionTabLocaleID(Setting.kActionsTab), "Actions" },
            { m_Setting.GetOptionGroupLocaleID(Setting.PostVanGroup), "Post vans & trucks" },
            // ... one entry per label/desc/tab/group ...
        };
    }

    public void Unload() { }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/magic-mail/repo/Localization/LocaleEN.cs#L19-L60,#L274-L276` (@6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47) - class + `ReadEntries` signature L19-L40, `Unload` L274-L276.

### 2. Emit a label + description for each control

Every setting control wants two keys: `GetOptionLabelLocaleID` (the caption) and
`GetOptionDescLocaleID` (the tooltip). Pass `nameof(...)` so the key tracks the property:

```csharp
{ m_Setting.GetOptionLabelLocaleID(nameof(Setting.PO_GetLocalMail)), "Fix low local mail" },
{ m_Setting.GetOptionDescLocaleID(nameof(Setting.PO_GetLocalMail)),
    "Post offices automatically request local mail when it runs low." },
```
Source: `../../../vice-and-order-research/mods/dossiers/magic-mail/repo/Localization/LocaleEN.cs#L62-L66` (@6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47)

For an enum dropdown, add a `GetEnumValueLocaleID` entry per value; for a confirmation
button, `GetOptionWarningLocaleID` supplies the dialog prompt:

```csharp
{ m_Setting.GetEnumValueLocaleID(Setting.SpeedUnit.Metric), "Metric (km/h)" },
{ m_Setting.GetOptionWarningLocaleID(nameof(Setting.ClearAllCustomSpeeds)),
    "Are you sure you want to clear all custom speed limits? ..." },
```
Source: `../../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/sETTING.cs#L169-L176` (@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed)

### 3. Register the source(s) in `OnLoad` - one `AddSource` per locale

Construct the settings first (keys derive from it), then add a source for each language.
Magic Mail registers **13 locales**, each its own `IDictionarySource` subclass:

```csharp
Setting setting = new Setting(this);

AddLocaleSource("en-US", new LocaleEN(setting));
AddLocaleSource("de-DE", new LocaleDE(setting));
AddLocaleSource("fr-FR", new LocaleFR(setting));
AddLocaleSource("ja-JP", new LocaleJA(setting));
AddLocaleSource("zh-HANS", new LocaleZH_CN(setting));   // Simplified Chinese
// ... es, it, ko, pl, pt-BR, zh-HANT, th, vi ...
```
Source: `../../../vice-and-order-research/mods/dossiers/magic-mail/repo/Mod.cs#L84-L102` (@6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47)

### 4. Wrap `AddSource` so a bad locale can't crash your mod

`AddSource` is the actual game call. Wrap it to null-check the manager and swallow
per-locale failures - a fragile translation shouldn't take down the whole mod:

```csharp
private static void AddLocaleSource(string localeId, IDictionarySource source)
{
    if (string.IsNullOrEmpty(localeId)) return;
    LocalizationManager? lm = GameManager.instance?.localizationManager;
    if (lm == null) return;
    try { lm.AddSource(localeId, source); }
    catch (Exception ex) { s_Log.Warn($"AddSource for '{localeId}' failed: {ex.Message}"); }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/magic-mail/repo/Mod.cs#L139-L160` (@6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47)

### 5. Order matters: settings, then locale sources, then load

The registration sequence in `OnLoad` is: build settings -> add every locale source ->
`LoadSettings` -> `RegisterInOptionsUI`. Magic Mail's own comment: "Settings must exist
before locales so labels resolve correctly."
Source: `../../../vice-and-order-research/mods/dossiers/magic-mail/repo/Mod.cs#L83-L109` (@6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47)

## Pitfalls & gotchas

- **Missing key -> raw locale ID on screen.** Any control/tab/group/enum value without
  a matching entry renders its key string. Cover every property, including read-only
  "About" rows and enum values.

- **Register locales after the settings object exists.** Locale keys are generated from
  the settings instance; if you `AddSource` before constructing `Setting`, the
  `GetOptionLabelLocaleID(...)` calls have nothing to key against. Magic Mail constructs
  settings first, by design (`magic-mail/repo/Mod.cs#L83-L88`).

- **Unregister sources you can unregister; unhook events you hooked.** Static
  `IDictionarySource` objects with an empty `Unload()` need no teardown, but if you
  subscribe to `localizationManager` events (see the reload variation), unsubscribe in
  `OnDispose` - i18n Everywhere detaches `onActiveDictionaryChanged`
  (`i18n-everywhere/repo/I18NEverywhere/Mod.cs#L708`).

- **A locale the player never selects is registered but idle.** `AddSource` for
  `th-TH`/`vi-VN` etc. is harmless when unused; the game only reads the active locale's
  merged dictionary. But some locales require the matching community language mod to be
  installed at all - Magic Mail flags `th-TH`/`vi-VN` as "requires \[language\] mod"
  (`magic-mail/repo/Mod.cs#L101-L102`).

- **String formatting for runtime values is your job.** Keys can hold `{0}` placeholders
  filled via `string.Format` at read time (Magic Mail's Status summaries,
  `magic-mail/repo/Settings/Settings.cs#L203-L210`); the localization system stores the
  template, not the filled string.

- **The precise fallback order when a key is missing across locales is
  `Needs Verification (in-game)`.** The game merges the active dictionary with a
  fallback locale, but the exact precedence between multiple sources on the same locale
  is not provable from these sources alone.

## Variations

- **Author en-US in code, load the rest from embedded JSON.** Instead of one subclass
  per language, register `LocaleEN` for en-US, then iterate the game's supported locales
  and load a matching embedded `l10n/<localeId>.json` resource into a `MemorySource`.
  This scales to many languages without a class each:

  ```csharp
  GameManager.instance.localizationManager.AddSource("en-US", new LocaleEN(Settings));
  foreach (string localeID in GameManager.instance.localizationManager.GetSupportedLocales())
  {
      string resourceName = $"{thisAssembly.GetName().Name}.l10n.{localeID}.json";
      if (!resourceNames.Contains(resourceName)) continue;
      using StreamReader reader = new(thisAssembly.GetManifestResourceStream(resourceName));
      var translations = Colossal.Json.JSON.Load(reader.ReadToEnd()).Make<Dictionary<string, string>>();
      GameManager.instance.localizationManager.AddSource(localeID, new MemorySource(translations));
  }
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/AnarchyMod.cs#L95,#L170-L195` (@a6311e898d20a775368668b234aaa32f06e3e1eb)

- **Hot-reload on locale switch.** Subscribe to
  `localizationManager.onActiveDictionaryChanged` (and `settings.userInterface.onSettingsApplied`)
  to rebuild dictionaries when the player changes language at runtime, rather than only
  once at load. i18n Everywhere does this and unhooks the handler in `OnDispose`:

  ```csharp
  GameManager.instance.localizationManager.onActiveDictionaryChanged += ChangeCurrentLocale;
  GameManager.instance.settings.userInterface.onSettingsApplied += ChangeCurrentLocale;
  GameManager.instance.localizationManager.AddSource("en-US", new LocaleEN(Setting));
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/i18n-everywhere/repo/I18NEverywhere/Mod.cs#L72-L91` (@d9285c2490079c6d67303da536207b47b106ce64); teardown at `Mod.cs#L708`.

- **Override one built-in key install-once; let the manager replay it on rebuild.** To
  retranslate a single existing game key (not add your own controls), register a tiny
  `IDictionarySource` that returns just that one pair, once per locale. You do **not** need
  to subscribe to `onActiveDictionaryChanged` - when the player switches language the
  localization manager rebuilds and re-reads every registered source, so a source added at
  load "wins" for its locale on each rebuild. Achievement Fixer overrides the vanilla
  `Menu.ACHIEVEMENTS_WARNING_MODS` banner this way, guarding each `AddSource` and marking
  the locale installed only on success so it is not double-added:

  ```csharp
  const string kWarningKey = "Menu.ACHIEVEMENTS_WARNING_MODS";  // game key to override
  var entries = new Dictionary<string, string> { [kWarningKey] = LocaleBannerText.For(localeId) };
  if (TryAddLocaleSource(localeId, new LocaleOverrideSource(entries), "EnsureWarningOverrideFor"))
      s_InstalledLocales.Add(localeId);
  ```
  The source is a minimal dictionary wrapper - `ReadEntries` returns the stored pairs and
  `Unload()` is empty:

  ```csharp
  internal sealed class LocaleOverrideSource : IDictionarySource
  {
      private readonly Dictionary<string, string> m_Entries;
      public LocaleOverrideSource(Dictionary<string, string> entries) { m_Entries = entries; }
      public IEnumerable<KeyValuePair<string, string>> ReadEntries(
          IList<IDictionaryEntryError> errors, Dictionary<string, int> indexCounts) => m_Entries;
      public void Unload() { }
  }
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/achievement-fixer/repo/Mod.cs#L124-L167` (@4d3a1eccc64a7a09de65a23f6abbb3daa998fd0d) - `AddWarningOverrideSources`/`EnsureWarningOverrideFor`, "rebuild normally ... without ... subscribing to onActiveDictionaryChanged" comment L124-L128; and `.../achievement-fixer/repo/Locale/AchievementLocaleHelpers.cs#L46-L64` (same commit) - `LocaleOverrideSource`.

- **Inject text at read time with a Harmony prefix - no source, no reload.** When strings
  are generated at runtime (per-object, user-authored) and re-registering sources on every
  change is impractical, patch `Colossal.Localization.LocalizationDictionary.TryGetValue`
  with a prefix that short-circuits your own key namespace. Custom Chirps returns runtime
  text for any `customchirps:`-prefixed key and skips the original method, so the string
  appears with no `AddSource` and no `ReloadActiveLocale`:

  ```csharp
  [HarmonyPatch(typeof(LocalizationDictionary), nameof(LocalizationDictionary.TryGetValue))]
  internal static class RuntimeChirpLocalizationPatch
  {
      [HarmonyPriority(Priority.First)]
      public static bool Prefix(string entryID, ref string value, ref bool __result)
      {
          if (!string.IsNullOrEmpty(entryID) &&
              entryID.StartsWith("customchirps:", System.StringComparison.Ordinal) &&
              RuntimeChirpLocalization.TryGetValue(entryID, out var runtimeValue))
          {
              value = runtimeValue; __result = true; return false;  // skip original
          }
          return true;
      }
  }
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/custom-chirps/repo/CustomChirps/UI/RuntimeChirpLocalizationPatch.cs#L7-L24` (@f018ac382e93e0b56cd7b974d1ce0b155d23d2b4)

- **Derive your strings from vanilla prefab titles at runtime.** A locale source can read
  live data when `ReadEntries` runs instead of holding literals. Specialized Industrial
  Zones builds each specialized zone's title/description by looking up the base prefab's
  title via `PrefabUISystem.GetTitleAndDescription`, reading it from the active dictionary,
  and appending a suffix - so its labels track whatever the base game (or a language mod)
  calls the parent zone:

  ```csharp
  _prefabUISystem.GetTitleAndDescription(_industrialZoneEntity, out var baseTitleID, out _);
  _prefabUISystem.GetTitleAndDescription(_specializedZoneEntity, out var titleID, out _);
  var activeDict = GameManager.instance.localizationManager.activeDictionary;
  if (activeDict.TryGetValue(baseTitleID, out var baseTitle))
      yield return new KeyValuePair<string, string>(titleID, baseTitle + $" [{_spec.Name}]");
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/specialized-industrial-zones/repo/src/SpecializedIndustryZones/SpecializedIndustrialZoneLocaleDictionary.cs#L10-L46` (@8d9153c3fda72e3b79a133bee6c83f9ba7df8b5a)

- **Co-locate the locale with the settings class.** For a small mod, nest `LocaleEN` in
  the same file as the `ModSetting` subclass - Road Speed Adjuster keeps both in
  `sETTING.cs` (`road-speed-adjuster/repo/sETTING.cs#L134-L191`). Fine for one or two
  languages; split into per-locale files once the list grows (Magic Mail's approach).

## See also
- Related recipes: [settings UI patterns](settings-patterns.md) (family N - the
  controls whose labels these locale entries supply).
- Reference: [technique index](../../technique-index.md) (family O coverage ledger).
- Explanation: [mod lifecycle](../../explanation/mod-lifecycle.md) (where locale
  registration sits in `OnLoad`).
- Case studies demonstrating it: [anarchy](../../case-studies/anarchy.md).

## Sources
- Canonical mods (dossier + repo):
  - `magic-mail` @6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47 - `repo/Mod.cs`, `repo/Localization/LocaleEN.cs`
  - `anarchy` @a6311e898d20a775368668b234aaa32f06e3e1eb - `repo/Anarchy/AnarchyMod.cs`, `repo/Anarchy/Settings/LocaleEN.cs`
  - `i18n-everywhere` @d9285c2490079c6d67303da536207b47b106ce64 - `repo/I18NEverywhere/Mod.cs`
  - `achievement-fixer` @4d3a1eccc64a7a09de65a23f6abbb3daa998fd0d - `repo/Mod.cs`, `repo/Locale/AchievementLocaleHelpers.cs` (install-once single-key override)
  - `custom-chirps` @f018ac382e93e0b56cd7b974d1ce0b155d23d2b4 - `repo/CustomChirps/UI/RuntimeChirpLocalizationPatch.cs` (read-time Harmony injection)
  - `specialized-industrial-zones` @8d9153c3fda72e3b79a133bee6c83f9ba7df8b5a - `repo/src/SpecializedIndustryZones/SpecializedIndustrialZoneLocaleDictionary.cs` (derive from vanilla prefab titles)
  - `road-speed-adjuster` @e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed - `repo/sETTING.cs` (supporting)
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding
