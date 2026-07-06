---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Use I18n Everywhere as a Localization Backbone (JSON Bundles, Load Order)"
Summary: How to localize a CS2 mod through the I18n Everywhere dependency - ship BCP-47 JSON locale bundles that I18n Everywhere merges into the live runtime by intercepting every key lookup, so community translators can fix strings without you cutting a release.
diataxis: how-to
source_version: "~1.5.10f1 (i18n-everywhere@d9285c2; date-pinned, static source only)"
last_reverified: "2026-07-04"
canonical_mods:
  - i18n-everywhere@d9285c2490079c6d67303da536207b47b106ce64
technique_applicability: [ui, content]
status: source-verified
Created: 2026-07-03
Updated: 2026-07-03
Owners:
  - codex
References:
  - Label: "Recipe: Localization helper (vanilla IDictionarySource path)"
    Path: ../recipes/localization-helper.md
  - Label: "Technique O - Localization multi-locale registration"
    Path: ../../technique-index.md
  - Label: "Reference: shared libraries (forward-ref)"
    Path: ../../reference/shared-libraries/README.md
---

# Use I18n Everywhere as a Localization Backbone (JSON Bundles, Load Order)

**I18n Everywhere** (Paradox Mods id `75426`, dependency id `baka.I18NEverywhere`, source
`github.com/baka-gourd/I18NEverywhere`) is a localization dependency you can build on: it
mounts locale dictionaries into the Cities: Skylines II runtime and reroutes *every* string
lookup through them, so a mod can ship JSON locale bundles and let community translators
ship fixes independently.

This how-to is the **dependency path**. It complements - and does not replace - the
[localization helper recipe](../recipes/localization-helper.md), which covers the vanilla
path you own end-to-end (implement `IDictionarySource`, call
`localizationManager.AddSource(localeId, source)` per locale). Reach for I18n Everywhere
when you want:

- **externalized JSON** translators can edit without a code change or a rebuild, and
- **community language packs** that override or extend your strings at runtime.

Everything below is cited to I18n Everywhere at pinned commit
`d9285c2490079c6d67303da536207b47b106ce64` (BaseVersion 1.5.0; the live storefront build
adds CS2 1.6 support with no source delta beyond this commit).

---

## How it works: one Harmony prefix over every lookup

You do not register sources with I18n Everywhere the way you do with the vanilla API.
Instead it applies a single **Harmony prefix to `LocalizationDictionary.TryGetValue`**, so
every key the game (or any mod) resolves is first checked against I18n Everywhere's own
dictionaries:

```csharp
Harmony harmony = new("Nptr.I18nEverywhere");
MethodInfo originalMethod =
    typeof(LocalizationDictionary).GetMethod("TryGetValue", BindingFlags.Public | BindingFlags.Instance);
MethodInfo prefix =
    typeof(HookLocalizationDictionary).GetMethod("Prefix", BindingFlags.Public | BindingFlags.Static);
harmony.Patch(originalMethod, new HarmonyMethod(prefix));
```
Source: `../../../vice-and-order-research/mods/dossiers/i18n-everywhere/repo/I18NEverywhere/Mod.cs#L78-L84` (@d9285c2490079c6d67303da536207b47b106ce64)

The prefix resolves from `CurrentLocaleDictionary`, then `FallbackLocaleDictionary`, and
only returns control to the game's own data when neither holds the key:

```csharp
if (!I18NEverywhere.CurrentLocaleDictionary.TryGetValue(entryID, out result) &&
    !I18NEverywhere.FallbackLocaleDictionary.TryGetValue(entryID, out result)) return true;
```
Source: `../../../vice-and-order-research/mods/dossiers/i18n-everywhere/repo/I18NEverywhere/HookLocalizationDictionary.cs#L53-L54` (@d9285c2490079c6d67303da536207b47b106ce64)

**Consequence for you:** because the interception is on the vanilla lookup, your existing
options-attribute keys and any `activeDictionary.TryGetValue(...)` reads (see the
[localization helper](../recipes/localization-helper.md)) transparently pick up I18n
Everywhere's merged strings - you supply the JSON, it handles delivery.

## Step 1: Declare the dependency

- **`mod.json`** - add `baka.I18NEverywhere` to the dependencies array.
- **`Properties/PublishConfiguration.xml`** - add the Paradox Mods id `75426` so publishing
  installs I18n Everywhere automatically (I18n Everywhere's own config pins
  `<ModId Value="75426" />`, `i18n-everywhere/repo/I18NEverywhere/Properties/PublishConfiguration.xml#L3`,
  @d9285c2).
- **Fallback (optional).** If you want the mod usable when the dependency is absent, keep a
  complete embedded English `IDictionarySource` (the vanilla path) so English still resolves;
  I18n Everywhere then only *adds* other locales on top.

## Step 2: Package locale bundles as JSON

1. Create a `lang/` folder next to your module assembly.
2. Add one JSON file per locale, named by its BCP-47 tag: `en-US.json`, `fr-FR.json`,
   `zh-HANS.json`, ... Each is a flat `{ "<localeKey>": "<translated text>" }` map.
3. Copy the folder into the build output. I18n Everywhere's own csproj copies `lang/**`
   with `PreserveNewest` - mirror that in your module:

   ```xml
   <ItemGroup>
     <Folder Include="lang\" />
     <Content Include="lang\**">
       <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
     </Content>
   </ItemGroup>
   ```
   Source (I18n Everywhere's own build wiring): `../../../vice-and-order-research/mods/dossiers/i18n-everywhere/repo/I18NEverywhere/I18NEverywhere.csproj#L237-L239` (@d9285c2490079c6d67303da536207b47b106ce64)

4. Confirm the output contains
   `%AppData%/LocalLow/Colossal Order/Cities Skylines II/Mods/<ModuleId>/lang/<locale>.json`.
   I18n Everywhere discovers per-mod `lang/` folders and merges them (this is the "embed"
   phase in Step 4).

## Step 3: Name keys with vanilla category prefixes and a stable namespace

Keys are just locale IDs. Use the same vanilla category shapes the game already renders (the
[localization helper recipe](../recipes/localization-helper.md) generates these exact shapes
via `GetSettingsLocaleID` / `GetOptionLabelLocaleID` / `GetOptionDescLocaleID` etc.), and
put a **stable namespace** in the bracket so a rename cannot silently orphan a translation:

```json
{
  "Options.SECTION[MyMod.Settings]": "My Mod",
  "Options.GROUP[MyMod.Settings.General]": "General",
  "Options.OPTION[MyMod.Settings.EnableFeature]": "Enable feature",
  "Options.OPTION_DESCRIPTION[MyMod.Settings.EnableFeature]": "Turns the feature on.",
  "Notifications.DESCRIPTION[MyMod.Events.Dispatched]": "{0} unit en route to {1}."
}
```

- Match the vanilla surface you extend (`Options.*`, `Tools.INFO[...]`, `Tutorials.*`,
  `Notifications.*`, `UI.*`) so the string renders in the right place.
- Use `string.Format` tokens (`{0}`, `{1:P0}`, `{2:N0}`) for runtime values; the
  localization system stores the template and you fill it at read time.
- Mirror each key across locales, but only include keys that already have a translation -
  missing keys fall through (see load order below).

## Step 4: Understand the load order (embed -> centralized -> packs)

I18n Everywhere's `LoadLocales` is the central pipeline. It builds two fresh dictionaries
and fills them in a **fixed three-phase order**, then swaps them in atomically:

```csharp
LoadEmbedLocales(...);        // 1) per-mod lang/ folders (lowest priority)
LoadCentralizedLocales(...);  // 2) this mod's own Localization/ bundle
LoadLanguagePacks(...);       // 3) dedicated translation mods (highest priority)
```
Source: `../../../vice-and-order-research/mods/dossiers/i18n-everywhere/repo/I18NEverywhere/Mod.cs#L113-L119` (@d9285c2490079c6d67303da536207b47b106ce64)

Each phase merges through one policy method, `MergeDictionary`, governed by the `Overwrite`
and `Restrict` settings:

```csharp
foreach (KeyValuePair<string, string> kv in source)
{
    if (kv.Key is null || kv.Value is null) { /* warn + skip */ continue; }
    if (target.ContainsKey(kv.Key))
    {
        if (restrict) { /* overlap: log + skip */ continue; }
        target[kv.Key] = kv.Value;   // default: later source wins
    }
    else target.Add(kv.Key, kv.Value);
}
```
Source: `../../../vice-and-order-research/mods/dossiers/i18n-everywhere/repo/I18NEverywhere/Mod.cs#L147-L177` (@d9285c2490079c6d67303da536207b47b106ce64)

Because language packs run **last** and `Restrict` defaults to **false**, a community pack
can override both your embedded `lang/` strings and any centralized bundle. That is the
whole point: translators ship a pack, players get the fix, you cut no release.

## Step 5: Let community language packs plug in

A language pack is itself a mod carrying an `i18n.json` manifest. I18n Everywhere classifies
each discovered mod into either a `lang/`-folder mod (embed) or an `i18n.json`-bearing
**language pack**, and loads packs through the same centralized loader with the pack's own
path. Keep your embedded `lang/` folder even when packs exist - packs are additive and only
override where a translation is provided.

---

## Pitfalls & gotchas

- **Missing key -> raw locale ID on screen.** Any key without an entry in the active or
  fallback locale falls through to the game and renders its ID string. Cover every control,
  group, tab, and enum value.
- **Casing matters.** File names must match the BCP-47 casing the game uses
  (`zh-HANS.json`, not `zh-Hans.json`); a mis-cased file will not be discovered.
- **`Overwrite` off changes interception, not just merging.** When `Overwrite` is off and
  the instance already contains an id, the prefix returns early and falls through to vanilla
  data - a source-verified behaviour of the hook, distinct from the `Restrict` merge policy
  (`i18n-everywhere/repo/I18NEverywhere/HookLocalizationDictionary.cs#L14-L57`, @d9285c2).
- **Null keys/values are stripped defensively.** The pipeline validates entries and skips
  null key/value pairs so they cannot NRE the Harmony prefix; malformed JSON that yields
  nulls is silently dropped, not surfaced. Validate your JSON before shipping.
- **Load timing is deferred.** The first real locale load waits until the game reaches the
  main menu (it is scheduled as a per-frame updater), so do not expect merged strings at the
  instant your `OnLoad` returns (`i18n-everywhere/repo/I18NEverywhere/Mod.cs#L91-L94`, @d9285c2).
- **The exact precedence when the *same* key exists in multiple sources of one phase is
  `Needs Verification (in-game)`** beyond the documented "later source wins unless
  `Restrict`" rule.

## Variations

- **Vanilla-only, no dependency.** If you do not need externalized JSON or community packs,
  skip I18n Everywhere entirely and register `IDictionarySource`s yourself - see the
  [localization helper recipe](../recipes/localization-helper.md), which cites I18n Everywhere
  among its canonical mods for the hot-reload-on-locale-switch variation.
- **Hot-reload on locale switch.** I18n Everywhere subscribes to
  `localizationManager.onActiveDictionaryChanged` and rebuilds on language change, unhooking
  in teardown (`i18n-everywhere/repo/I18NEverywhere/Mod.cs#L72,#L708`, @d9285c2). If you use
  the vanilla path, the same hook is the reload mechanism.

## See also

- **The vanilla path you own:** [Localization helper recipe](../recipes/localization-helper.md).
- **Coverage ledger (family O):** [Technique Index](../../technique-index.md).
- **Shared libraries this depends on:** [reference](../../reference/shared-libraries/README.md)
  (forward-ref).

## Sources

- Canonical mod (dossier + repo): `i18n-everywhere`
  @d9285c2490079c6d67303da536207b47b106ce64 (BaseVersion 1.5.0) -
  `repo/I18NEverywhere/Mod.cs` (LoadLocales pipeline, MergeDictionary, Harmony patch),
  `repo/I18NEverywhere/HookLocalizationDictionary.cs` (TryGetValue prefix),
  `repo/I18NEverywhere/Setting.cs` (Overwrite/Restrict/LoadLanguagePacks),
  `repo/I18NEverywhere/I18NEverywhere.csproj` (`lang/**` packaging),
  `repo/I18NEverywhere/Properties/PublishConfiguration.xml` (ModId 75426).
- Complementary recipe: [`how-to/recipes/localization-helper.md`](../recipes/localization-helper.md).
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Localize_your_mod ,
  https://github.com/baka-gourd/I18NEverywhere.Localization
