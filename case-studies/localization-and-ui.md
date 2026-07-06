---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Case study: The localization & UI dependency stack (UIL + I18n Everywhere)"
case_study: localization-and-ui
mod: "Unified Icon Library (74417) + I18n Everywhere (75426)"
dossier: ../../vice-and-order-research/mods/dossiers/unified-icon-library/
repo_commit: "unified-icon-library@b200d890 + i18n-everywhere@d9285c2"
canonical_mods:
  - unified-icon-library@b200d8901346457f97c032e86e1d8437a12c0cc8
  - i18n-everywhere@d9285c2490079c6d67303da536207b47b106ce64
source_version: "~1.5.x (unified-icon-library@b200d89; date-pinned, static source only)"
last_reverified: "2026-07-04"
diataxis: explanation
techniques: [H, O, D]
technique_applicability: [ui, content]
status: source-verified
Created: 2026-07-03
Updated: 2026-07-03
Owners:
  - codex
---

# The localization & UI dependency stack - case study

> Two of the most-subscribed mods on Paradox Mods do almost nothing on their own: Unified
> Icon Library ships shared UI *art* behind one `AddHostLocation` call, and I18n Everywhere
> ships shared *translations* behind one Harmony prefix on the game's dictionary lookup.
> Both exist so other mods can *delegate* instead of *duplicate* - the defining move of a
> dependency-stack library.

## What it does / why it's instructive

This is a **combined** case study of two standalone dependency mods that consumer mods lean
on constantly:

- **Unified Icon Library (UIL)** (algernon, Paradox Mods id `74417`) - a curated catalogue
  of SVG icons matching the CS2 UI, pre-registered with the Gameface runtime under one COUI
  host named `uil`, so any mod references `coui://uil/Standard/ArrowLeft.svg` instead of
  bundling sprites.
- **I18n Everywhere** (Baka-gourd, Paradox Mods id `75426`) - a localization injector that
  intercepts *every* string lookup in the game and answers from its own merged dictionaries,
  so a mod can ship JSON locale files (and let community translators ship fixes) without
  cutting a release.

They are worth studying together because they solve the same architectural problem -
"stop every mod from re-shipping the same shared resource" - in two instructive, opposite
ways: UIL adds a resource *host* the consumer opts into by URI; I18n Everywhere *hooks a
vanilla method* so consumers benefit with no API call at all. Both also model good
dependency-library hygiene: tiny runtime surface, no gratuitous ECS systems, and a clear
declare-and-degrade contract for the mods that depend on them.

## Architecture at a glance

### Unified Icon Library: one host registration, then get out of the way

UIL's entire runtime is a single `IMod` whose `OnLoad` caches the mod's install directory
and mounts its `/Icons/` folder as a COUI host - there are no ECS systems and no Harmony
(`repo/Code/Mod.cs#L20`, `#L72-L87`):

```csharp
UIManager.defaultUISystem.AddHostLocation("uil", AssemblyPath + "/Icons/");
```
(`repo/Code/Mod.cs#L86`)

`AssemblyPath` is resolved robustly - it queries `AssetDatabase.global` for the executable
asset matching the running assembly rather than hard-coding a path, so the icon lookup
survives Paradox relocating the install (`repo/Code/Mod.cs#L38-L61`). `OnDispose` clears
the static instance but deliberately never calls `RemoveHostLocation`, so the `uil` host
persists for the process lifetime (`repo/Code/Mod.cs#L92-L96`). The catalogue is three
style folders (`Standard`, `Dark`, `Colored`, 268 icons each), addressed as
`coui://uil/<Style>/<Icon>.svg`. The project targets `net48`
(`repo/UnifiedIconLibrary.csproj#L12`) and pins its Paradox id in publish config
(`repo/Properties/PublishConfiguration.xml#L2`, `ModId 74417`). The full URI scheme,
styles, and CSS layer-colour tinting are catalogued in the
[UIL reference](../reference/shared-libraries/unified-icon-library.md).

### I18n Everywhere: one Harmony prefix, a three-phase load pipeline

I18n Everywhere's `IMod` (`repo/I18NEverywhere/Mod.cs#L36`) applies a single Harmony prefix
to `LocalizationDictionary.TryGetValue` (Harmony id `Nptr.I18nEverywhere`,
`repo/I18NEverywhere/Mod.cs#L84`), so every key the game or any mod resolves is checked
against I18n Everywhere's dictionaries first
(`repo/I18NEverywhere/HookLocalizationDictionary.cs#L14`). The prefix resolves current
locale, then fallback, and only returns control to vanilla when neither holds the key
(`repo/I18NEverywhere/HookLocalizationDictionary.cs#L53-L54`):

```csharp
if (!I18NEverywhere.CurrentLocaleDictionary.TryGetValue(entryID, out result) &&
    !I18NEverywhere.FallbackLocaleDictionary.TryGetValue(entryID, out result)) return true;
```

The dictionaries are built by `LoadLocales` in a fixed three-phase order, then swapped in
atomically (`repo/I18NEverywhere/Mod.cs#L113-L119`):

```csharp
LoadEmbedLocales(...);        // 1) per-mod lang/ folders   (lowest priority)
LoadCentralizedLocales(...);  // 2) this mod's own bundle
LoadLanguagePacks(...);       // 3) dedicated translation mods (highest priority)
```

Every phase merges through one policy method. `MergeDictionary` skips null keys/values,
and - because `Restrict` defaults to false - lets a later source overwrite an earlier one,
so language packs (loaded last) win by default (`repo/I18NEverywhere/Mod.cs#L147-L177`).
Pack discovery does not trust the in-game mod list; it queries the Paradox backend for the
active playset (`repo/I18NEverywhere/Mod.cs#L467-L470`):

```csharp
PdxSdkPlatform manager = PlatformManager.instance.GetPSI<PdxSdkPlatform>("PdxSdk");
HashSet<Mod> playsetResult = manager.GetModsInActivePlayset()...GetResult();
```

Behaviour is governed by three General toggles - `Overwrite` (default true), `Restrict`
(default false), `LoadLanguagePacks` (default true)
(`repo/I18NEverywhere/Setting.cs#L26`, `#L28`, `#L32`) - persisted via
`[FileLocation(@"ModsSettings\I18NEverywhere\I18NEverywhere")]`
(`repo/I18NEverywhere/Setting.cs#L18`). The mod uses `Lib.Harmony 2.2.2`
(`repo/I18NEverywhere/I18NEverywhere.csproj#L124`) and reads `.lb` bundles as zip archives
(`repo/I18NEverywhere/Models/LanguageBundle.cs#L68-L74`). The full dependency-consumer
workflow is in [how-to: I18n Everywhere integration](../how-to/localization/i18n-integration.md).

### The shared pattern

Both mods are *pure delegation targets*: a consumer declares the dependency, then references
a URI (UIL) or ships a JSON file (I18n Everywhere) and gets the shared resource with no
copying. Neither runs simulation work; UIL runs no ECS systems at all, and I18n Everywhere's
only continuous cost is a per-frame initializer that defers the first locale load until the
main menu. That minimal footprint is what lets a million-subscriber dependency sit under a
long tail of consumer mods without becoming a compatibility hazard.

## Techniques demonstrated

- **COUI icon/host registration (family H)** - UIL's single
  `UIManager.defaultUISystem.AddHostLocation("uil", ...)` is the canonical static-mount
  example (`repo/Code/Mod.cs#L86`). Deep-dive and the three-arg watch/teardown variant are in
  the [UIL reference](../reference/shared-libraries/unified-icon-library.md) and
  [ExtraLib reference](../reference/shared-libraries/extralib.md); background in
  [The Gameface UI Runtime](../explanation/gameface-runtime.md); catalogued in the
  [Technique Index](../technique-index.md) (H).
- **Localization multi-locale registration (family O)** - I18n Everywhere's embed ->
  centralized -> pack pipeline with a single overwrite/`Restrict` merge policy
  (`repo/I18NEverywhere/Mod.cs#L113-L177`). The dependency path is
  [how-to: I18n Everywhere integration](../how-to/localization/i18n-integration.md); the
  vanilla `IDictionarySource` path you own end-to-end is the
  [Localization helper recipe](../how-to/recipes/localization-helper.md); Technique Index (O).
- **Harmony redirector over a vanilla method (family D)** - I18n Everywhere reroutes the
  game's own `LocalizationDictionary.TryGetValue` through a Harmony prefix rather than
  registering sources the vanilla way (`repo/I18NEverywhere/Mod.cs#L84`,
  `repo/I18NEverywhere/HookLocalizationDictionary.cs#L14-L57`). This is the interception
  end of family D (the reverse-patch/bridge end is shown in the
  [Write Everywhere case study](./write-everywhere-ecosystem.md)); Technique Index (D).

## Key decisions & tradeoffs

- **Opt-in host (UIL) vs. transparent interception (I18n Everywhere).** UIL requires the
  consumer to *choose* a `coui://uil/...` URI - explicit, discoverable, but the consumer
  must know the icon exists. I18n Everywhere requires *nothing* from the consumer's code:
  because it patches the vanilla lookup, existing option keys and `activeDictionary`
  reads transparently pick up merged strings. Transparency is powerful but means a single
  Harmony prefix now sits on the hottest string path in the game
  (`repo/I18NEverywhere/HookLocalizationDictionary.cs#L14`).
- **Packs-win load order.** Running language packs last, with `Restrict` off by default, is
  a deliberate choice so community translators can override a mod's own embedded strings
  without the mod author's involvement - the whole point of the ecosystem
  (`repo/I18NEverywhere/Mod.cs#L147-L177`). The tradeoff is that a careless pack can shadow
  correct strings; `Restrict` exists to flip the policy to log-and-skip.
- **Never removing the host.** UIL leaking the `uil` host after a hot-disable
  (`repo/Code/Mod.cs#L92-L96`) is a pragmatic call - icons that other mods reference keep
  resolving - but it means a two-arg `AddHostLocation` host cannot be cleanly unmounted
  mid-session; ExtraLib's three-arg overload with `RemoveHostLocation` is the removable
  alternative (see [ExtraLib](../reference/shared-libraries/extralib.md)).
- **Tiny surface, hard dependency.** Both keep near-zero runtime logic so they are safe to
  depend on, but both are *hard* dependencies at runtime: reference `coui://uil` without UIL
  loaded and icons render blank; ship only non-English JSON without I18n Everywhere and those
  locales never load. Consumers must declare the dependency and degrade gracefully - see
  [Dependency Strategy](../explanation/dependency-strategy.md).
- **Query the playset, not the mod list.** I18n Everywhere reads the active playset through
  the PDX SDK rather than the in-game mod manager, so packs shipped outside the enabled set
  are found correctly (`repo/I18NEverywhere/Mod.cs#L467-L470`); the cost is a dependence on
  an SDK call that returns an empty set on error (with a `LegacyCacheMods` fallback).

## Pitfalls / upstream-watch

- **Missing key -> raw locale ID on screen.** Any key absent from the active and fallback
  dictionaries falls through to the game and renders its ID; cover every control, group,
  and enum value (`repo/I18NEverywhere/HookLocalizationDictionary.cs#L53-L54`).
- **Harmony-prefix collisions.** Any other mod that also prefixes
  `LocalizationDictionary.TryGetValue` can reorder or bypass I18n Everywhere's dictionaries;
  cross-test with other localization injectors (`repo/I18NEverywhere/Mod.cs#L84`).
- **`Overwrite` off changes interception, not just merging.** When `Overwrite` is off and
  the instance already holds an id, the prefix returns early and falls through to vanilla -
  a hook behaviour distinct from the `Restrict` merge policy
  (`repo/I18NEverywhere/HookLocalizationDictionary.cs#L22-L26`).
- **Icon-name typos resolve to nothing.** A misspelled `coui://uil/...` URI silently renders
  blank rather than erroring; confirm the icon exists in the chosen style (UIL's catalogue
  grows over time) (`repo/Code/Mod.cs#L86`).
- **Duplicate-host collision semantics are `Needs Verification`.** What happens if two mods
  register the same host name lives in the closed-source `Colossal.UI` assembly and cannot be
  confirmed from mod source; a survey found UIL is currently the only mod claiming `uil`.
- **Version drift between source and storefront.** These pins are the source tips
  (`unified-icon-library@b200d890` = v1.0.14; `i18n-everywhere@d9285c2` = BaseVersion 1.5.0);
  the live storefront builds add CS2 1.6 support with no source delta beyond these commits.
  Re-verify the Harmony target and host call on each game patch.

## Source pointers

- Dossiers: `../../vice-and-order-research/mods/dossiers/unified-icon-library/` and
  `../../vice-and-order-research/mods/dossiers/i18n-everywhere/` (index / source / modding).
- UIL repo @ `b200d8901346457f97c032e86e1d8437a12c0cc8` (branch master, tag v1.0.14) -
  `repo/Code/Mod.cs`, `repo/UnifiedIconLibrary.csproj`, `repo/Properties/PublishConfiguration.xml`.
- I18n Everywhere repo @ `d9285c2490079c6d67303da536207b47b106ce64` (branch main,
  BaseVersion 1.5.0) - `repo/I18NEverywhere/Mod.cs` (LoadLocales, MergeDictionary, Harmony
  patch, playset discovery), `repo/I18NEverywhere/HookLocalizationDictionary.cs` (TryGetValue
  prefix), `repo/I18NEverywhere/Setting.cs`, `repo/I18NEverywhere/Models/LanguageBundle.cs`.
- Storefront: UIL id `74417` (userModVersion 1.0.14); I18n Everywhere id `75426`
  (1.5.1+build.1, requiredVersion `1.6.*`).
- Related handbook pages: [UIL reference](../reference/shared-libraries/unified-icon-library.md),
  [ExtraLib reference](../reference/shared-libraries/extralib.md),
  [I18n Everywhere integration](../how-to/localization/i18n-integration.md),
  [Localization helper recipe](../how-to/recipes/localization-helper.md),
  [Dependency Strategy](../explanation/dependency-strategy.md),
  [Write Everywhere case study](./write-everywhere-ecosystem.md),
  [Technique Index](../technique-index.md) (families H, O, D).
