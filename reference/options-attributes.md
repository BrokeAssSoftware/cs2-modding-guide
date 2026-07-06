---
FrontmatterVersion: 1
DocumentType: Guide
Title: SettingsUI Attribute Reference
Summary: Lookup table of the [SettingsUI*] / [FileLocation] attributes used to build a CS2 mod Options page declaratively on a ModSetting class - each entry cross-checked against real mod source at a pinned commit, with attributes not seen in the corpus flagged Needs Verification.
diataxis: reference
source_version: "1.5.x-1.6.0f1 settings corpus (magic-mail@6fb3d2b, anarchy@a6311e8, better-bulldozer@4408466, vno-debug@8d1cbb8; static source only)"
last_reverified: "2026-07-05"
status: needs-verification
canonical_mods:
  - magic-mail@6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47
  - anarchy@a6311e898d20a775368668b234aaa32f06e3e1eb
  - better-bulldozer@4408466f226db811159d92859479ae1e1c28ba06
  - vno-debug@8d1cbb848850d152430c40aa374d970dc7618bae
Created: 2026-07-03
Updated: 2026-07-05
Owners:
  - codex
References:
  - Label: Recipe - Settings UI patterns (how to compose these)
    Path: ../how-to/recipes/settings-patterns.md
  - Label: Settings and Data (when to use settings vs runtime data)
    Path: ../explanation/settings-and-data.md
  - Label: Official CS2 modding wiki (authoritative)
    Path: https://cs2.paradoxwikis.com/Modding
  - Label: Technique Index
    Path: ../technique-index.md
---

# SettingsUI Attribute Reference

A dry lookup for the attributes that turn a `Game.Settings.ModSetting` subclass into an Options
page - no React, no custom UI. Each public property becomes a control based on its type
(`bool` -> checkbox, `enum` -> dropdown, `float`/`int` + `[SettingsUISlider]` -> slider,
write-only `bool` + `[SettingsUIButton]` -> button); the attributes below place, group, gate,
and annotate those controls.

**Scope of verification.** Every attribute in the "Verified in corpus" tables was confirmed
present in real mod source at a pinned commit (citations inline). Attributes in the
"Needs Verification" table appear in older community cheatsheets but were **not** observed in
this handbook's source corpus; they may exist, be renamed, or be superseded - confirm against
the official wiki before relying on them. For the official, exhaustive list, link out to the
[CS2 modding wiki](https://cs2.paradoxwikis.com/Modding) rather than trusting a duplicate here.

For *how* to compose these into a working page (defaults, presets, `Apply()`, persistence,
localization), see the recipe [Settings UI patterns](../how-to/recipes/settings-patterns.md).

Corpus and pinned commits used for verification:

- `magic-mail` @`6fb3d2b` - `repo/Settings/Settings.cs`
- `anarchy` @`a6311e8` - `repo/Anarchy/Settings/AnarchyModSettings.cs`
- `better-bulldozer` @`4408466` - `repo/BetterBulldozer/Settings/BetterBulldozerModSettings.cs`
- `vno-debug` @`8d1cbb848850d152430c40aa374d970dc7618bae` - `repo/Settings.cs`

## File location

| Attribute | Level | Purpose | Verified usage |
| --- | --- | --- | --- |
| `[FileLocation(path)]` | class | Names the on-disk settings file the game reads/writes (under the user's `ModsSettings` tree). Path may be nested (`"ModsSettings/MagicMail/MagicMail"`) or a flat token (`"Mods_Yenyang_Anarchy"`). | `magic-mail/repo/Settings/Settings.cs#L26`; `anarchy/repo/Anarchy/Settings/AnarchyModSettings.cs#L19`; `better-bulldozer/repo/BetterBulldozer/Settings/BetterBulldozerModSettings.cs#L19` |

## Layout and grouping

| Attribute | Level | Purpose | Verified usage |
| --- | --- | --- | --- |
| `[SettingsUISection(tab, group)]` | property | Places a control on a named tab and group (both are string constants on the class). | `magic-mail/.../Settings.cs#L140`; `anarchy/.../AnarchyModSettings.cs#L113` |
| `[SettingsUITabOrder(...)]` | class | Declares the order of tabs. | `magic-mail/.../Settings.cs#L27`; `anarchy/.../AnarchyModSettings.cs#L20` |
| `[SettingsUIGroupOrder(...)]` | class | Declares the order of groups within a section. | `magic-mail/.../Settings.cs#L29`; `anarchy/.../AnarchyModSettings.cs#L21` |
| `[SettingsUIShowGroupName(...)]` | class | Forces group headers to render (they are hidden by default in some layouts). | `magic-mail/.../Settings.cs#L36` |

## Value inputs

| Attribute | Level | Purpose | Verified usage |
| --- | --- | --- | --- |
| `[SettingsUISlider(min, max, step, scalarMultiplier, unit)]` | property (`float`/`int`) | Renders a slider. Arguments are passed by **name**; `unit` is a `Unit` enum (verified values: `Unit.kPercentage`, `Unit.kFloatTwoFractions`, `Unit.kInteger`). | `magic-mail/.../Settings.cs#L191`; `anarchy/.../AnarchyModSettings.cs#L192` (float `0-1.75`, `kFloatTwoFractions`), `#L205` (int `1-600`, `kInteger`) |
| `[SettingsUITextInput]` | property (read/write `string`) | Renders a free-text entry box bound to the string property (unlike a get-only `string`, which renders read-only). Corpus uses it for free-text filter patterns. | `vno-debug/repo/Settings.cs#L121` (`NamespaceInclude`), `#L125` (`NamespaceExclude`), `#L129` (`TypeNamePattern`) |

Controls with **no** attribute: a plain `bool` property renders as a checkbox, and a public
`enum` property renders as a dropdown, without any dedicated value-input attribute (place them
with `[SettingsUISection]` and gate them like any other control). A get-only `string` property
renders as a read-only text row. This is how the corpus produces checkboxes, dropdowns, and
"About" text - see the recipe's variations section.

## Actions and buttons

| Attribute | Level | Purpose | Verified usage |
| --- | --- | --- | --- |
| `[SettingsUIButton]` | property (write-only `bool`) | Renders a clickable button; the `set` accessor runs on click (guard with `if (value)`). | `magic-mail/.../Settings.cs#L141`; `better-bulldozer/.../BetterBulldozerModSettings.cs#L51` |
| `[SettingsUIConfirmation]` | property | Gates the action behind a yes/no confirmation dialog. | `anarchy/.../AnarchyModSettings.cs#L318`; `better-bulldozer/.../BetterBulldozerModSettings.cs#L52` |
| `[SettingsUIButtonGroup(id)]` | property | Lays multiple buttons on one row under a shared group id. | `magic-mail/.../Settings.cs#L139` |
| `[SettingsUISetter(type, method)]` | property | Runs a named method the instant the control's value changes (no wait for `Apply()`). | `better-bulldozer/.../BetterBulldozerModSettings.cs#L45`; `anarchy/.../AnarchyModSettings.cs#L126` |

## Visibility and state gating

| Attribute | Level | Purpose | Verified usage |
| --- | --- | --- | --- |
| `[SettingsUIHidden]` | property | Keeps a field in the settings file but draws no control (bookkeeping flags, deprecated fields kept for ID stability). | `magic-mail/.../Settings.cs#L85`; `anarchy/.../AnarchyModSettings.cs#L215`; `better-bulldozer/.../BetterBulldozerModSettings.cs#L104` |
| `[SettingsUIHideByCondition(type, member, value?)]` | property | Hides a control when another member (a property's expected value, or a predicate method) matches. | `magic-mail/.../Settings.cs#L197` (property + expected `true`); `anarchy/.../AnarchyModSettings.cs#L143` (predicate method) |
| `[SettingsUIDisableByCondition(type, member)]` | property | Greys out (but still shows) a control based on another member. | `better-bulldozer/.../BetterBulldozerModSettings.cs#L71`; `anarchy/.../AnarchyModSettings.cs#L303` |

## Input binding (key/mouse rebinds)

| Attribute | Level | Purpose | Verified usage |
| --- | --- | --- | --- |
| `[SettingsUIKeyboardAction(name, type, usages)]` | class | Declares a rebindable keyboard action for the mod. | `anarchy/.../AnarchyModSettings.cs#L23` |
| `[SettingsUIKeyboardBinding(key, actionName, modifiers)]` | property | Binds a default key (with ctrl/alt/etc.) to a declared action, exposed as a rebindable row. | `anarchy/.../AnarchyModSettings.cs#L257` |
| `[SettingsUIMouseAction(name, ...)]` | class | Declares a rebindable mouse action. | `anarchy/.../AnarchyModSettings.cs#L22` |
| `[SettingsUIMouseBinding(actionName)]` | property | Binds/exposes a mouse action as a rebindable row. | `anarchy/.../AnarchyModSettings.cs#L212` |
| `[SettingsUIBindingMimic(map, action)]` | property | Mirrors an existing game input map/action so the rebind row reflects the vanilla binding. | `anarchy/.../AnarchyModSettings.cs#L214` |

## Needs Verification (not seen in this corpus)

These appear in older community cheatsheets but were **not** observed in the source corpus at
the pinned commits. They may exist under a different name, be deprecated, or require additional
UI-mod support. Confirm against the [official wiki](https://cs2.paradoxwikis.com/Modding) before
use, and treat them as **Needs Verification**:

| Attribute / helper | Cheatsheet-claimed purpose | Note |
| --- | --- | --- |
| `[SettingsUICheckbox]` / `[SettingsUIToggle]` | explicit boolean control styling | A plain `bool` already renders as a checkbox in the corpus with **no** such attribute; these explicit forms unconfirmed. |
| `[SettingsUIEnum]` | enum dropdown | A public `enum` already renders as a dropdown with no such attribute in the corpus. |
| `[SettingsUIText]` / `[SettingsUIMultilineText]` | read-only text display | A get-only `string` already renders as a read-only row in the corpus with no such attribute. |
| `[SettingsUIMultilineInput]` / `[SettingsUITextArea]` | multi-line free-text entry | Not seen in corpus. (`[SettingsUITextInput]` for single-line free text **is** verified - see the Value inputs table.) |
| `[SettingsUIDirPicker]` / `[SettingsUIFilePicker]` | filesystem pickers | Not seen in corpus. |
| `[SettingsUINumericInput]` | direct numeric entry | Not seen in corpus. |
| `[SettingsUIDescription(key)]` | per-property description override | Not seen in corpus (labels/descriptions come from the localization dictionary source). |
| `[SettingsUIGamepadAction(...)]` | gamepad rebind metadata | Corpus uses keyboard/mouse binding attributes only. |
| `[ProxyBinding(...)]` | rebindable-action marker | Corpus uses `[SettingsUIKeyboardBinding]` / `[SettingsUIMouseBinding]` instead. |
| `[SettingsUITransportNetwork]` | transport-network picker | Not seen in corpus. |
| `[SettingsUICustomControl]` | custom Gameface component host | Not seen in corpus; would require a custom UI module (see [Gameface runtime](../explanation/gameface-runtime.md)). |
| `GetBindingMapLocaleID` / `GetBindingKeyLocaleID` | localized names for binding rows | These are localization *methods*, not attributes; resolve them via the mod's `IDictionarySource`. |
| `Unit.kInt` / `Unit.kFloatSingleFraction` | slider unit formats | Corpus verified `Unit.kInteger`, `Unit.kPercentage`, `Unit.kFloatTwoFractions`; these two spellings unconfirmed. |

## See also

- Recipe: [Settings UI patterns](../how-to/recipes/settings-patterns.md) - how these attributes
  compose into a real Options page (defaults, presets, `Apply()`, persistence, localization).
- Explanation: [Settings and Data](../explanation/settings-and-data.md) - when a value belongs
  in settings versus runtime ECS data versus a side-car file.
- Explanation: [The Gameface UI Runtime](../explanation/gameface-runtime.md) - the custom-UI
  route these attributes let you avoid.
- Official [CS2 modding wiki](https://cs2.paradoxwikis.com/Modding) - authoritative attribute
  list; link out rather than duplicating.
- [Technique Index](../technique-index.md) - family N (SettingsUI slider/section/hide-by-condition).
