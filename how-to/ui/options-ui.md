---
FrontmatterVersion: 1
DocumentType: Guide
Title: "How-to: Build a mod Options-menu UI"
diataxis: how-to
source_version: "1.6.0f1 (magic-mail@6fb3d2b; date-pinned, static source only)"
last_reverified: "2026-07-04"
canonical_mods:
  - magic-mail@6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47
  - road-speed-adjuster@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed
technique_applicability: [core, ui]
status: source-verified
Created: 2026-07-03
Updated: 2026-07-03
Owners:
  - codex
---

# Build a mod Options-menu UI

> Give your mod a page inside the game's own **Options** panel - tabs, groups,
> sliders, checkboxes, dropdowns, and buttons - using only a `ModSetting`
> subclass, `[SettingsUI*]` attributes, one `RegisterInOptionsUI()` call, and a
> localization source for the labels. No React, no Gameface markup, no Harmony.

## Problem
You have a working mod and you want players to configure it. You want the settings
to appear in the vanilla **Options** menu (not a bespoke window), to persist between
sessions, and to show readable, localized labels. You do not want to write any UI
markup or touch the Gameface/React layer to get there.

## Solution
Cities: Skylines II ships a declarative settings API. Subclass
`Game.Settings.ModSetting`, decorate each public property with `[SettingsUI*]`
attributes (the attribute picks the control - slider, checkbox, dropdown, button),
place controls onto tabs and groups with `[SettingsUISection]`, and register the
whole object in `IMod.OnLoad` with `setting.RegisterInOptionsUI()`. Persist with the
game's `AssetDatabase` load/save. Labels come from a paired localization source that
you register **before** the settings resolve. This page walks the end-to-end task;
for the full control-by-control attribute catalogue (every slider arg, conditional
hide/disable, confirmations, presets, setter callbacks) see the companion recipe
[Settings UI patterns](../recipes/settings-patterns.md).

## Steps & Code

### 1. Subclass `ModSetting` and lay out tabs + groups on the class

`[FileLocation(...)]` names the on-disk settings file. Three class-level attributes
declare the page skeleton: `[SettingsUITabOrder]` lists the tabs left-to-right,
`[SettingsUIGroupOrder]` orders the groups within tabs, and `[SettingsUIShowGroupName]`
makes a group render its header. Tabs and groups are just `string` constants you
reference from each control:

```csharp
[FileLocation("ModsSettings/MagicMail/MagicMail")]
[SettingsUITabOrder(kActionsTab, kStatusTab, kAboutTab)]
[SettingsUIGroupOrder(ResetGroup, PostVanGroup, PostOfficeGroup, /* ... */ kAboutLinksGroup)]
[SettingsUIShowGroupName(ResetGroup, PostVanGroup, PostOfficeGroup, /* ... */ kAboutLinksGroup)]
public sealed class Setting : ModSetting
{
    public const string kActionsTab = "Actions";
    public const string kStatusTab  = "Status";
    public const string kAboutTab   = "About";
    public const string PostVanGroup = "PostVan";
    // ...
}
```
Source: `../../../vice-and-order-research/mods/dossiers/magic-mail/repo/Settings/Settings.cs#L26-L55` (@6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47) - `[FileLocation]` L26, tab/group ordering L27-42, class decl L43, tab constants L47, group constant L55.

### 2. Seed defaults in the constructor and the required `SetDefaults()`

The constructor runs on first construction; `SetDefaults()` is a mandatory override
that defines both the initial state and the reset target. A first-run guard keeps you
from clobbering saved values on later loads:

```csharp
public Setting(IMod mod) : base(mod)
{
    if (!NotFirstTime) { SetDefaults(); NotFirstTime = true; }
}

public override void SetDefaults() { SetToVanilla(); }
```
Source: `../../../vice-and-order-research/mods/dossiers/magic-mail/repo/Settings/Settings.cs#L95-L104` (@6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47); `SetDefaults` override L509-L512.

> `SetDefaults()` must set **every** field. A property you forget keeps its zero
> value - a `0%` slider, a wrong enum. See the recipe's pitfalls for details.

### 3. Declare each control as a property + `[SettingsUISection]`

The property type plus attribute chooses the control. A numeric with
`[SettingsUISlider]` is a slider; a `bool` is a checkbox; an `enum` is a dropdown; a
write-only `bool` with `[SettingsUIButton]` is a clickable action.
`[SettingsUISection(tab, group)]` places it:

```csharp
[SettingsUISection(kActionsTab, PostVanGroup)]
[SettingsUISlider(min = 100, max = 500, step = 10,
    scalarMultiplier = 1, unit = Unit.kPercentage)]
public int PostVanMailLoadPercentage { get; set; }
```
Source: `../../../vice-and-order-research/mods/dossiers/magic-mail/repo/Settings/Settings.cs#L190-L201` (@6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47) (slider + section; the third attribute on this property is a conditional-hide covered in the recipe).

An `enum` property renders as a dropdown with no extra attribute - each value is
localized separately:

```csharp
[SettingsUISection(kMainTab, kDisplayGroup)]
public SpeedUnit SpeedUnitPreference { get; set; } = SpeedUnit.Auto;
// ...
public enum SpeedUnit { Auto, Metric, Imperial }
```
Source: `../../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/sETTING.cs#L31-L32` (property) and `#L126` (enum) (@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed).

A read-only `string` property renders as a static text row - handy for a version or
credits line on an About tab:

```csharp
[SettingsUISection(kMainTab, kAboutGroup)]
public string Version => "Version 1.0.3";
```
Source: `../../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/sETTING.cs#L62` (@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed).

(Buttons, confirmations, presets, `Apply()`, and conditional visibility are the same
attribute family - see [Settings UI patterns](../recipes/settings-patterns.md) for
each.)

### 4. Register, localize, and load in `IMod.OnLoad`

Three things must happen in `OnLoad`: the `Setting` must exist, a localization source
must be added (so labels resolve to text instead of raw keys), and the object must be
registered in the Options UI and loaded from disk. Order matters - construct the
settings first so the locale source can reference it:

```csharp
// Settings must exist before locales so labels resolve correctly.
Setting setting = new Setting(this);
Settings = setting;

AddLocaleSource("en-US", new LocaleEN(setting));   // one per language

// Load persisted settings or create defaults on first run.
AssetDatabase.global.LoadSettings(ModId, setting, new Setting(this));

// Register in Options UI.
setting.RegisterInOptionsUI();
```
Source: `../../../vice-and-order-research/mods/dossiers/magic-mail/repo/Mod.cs#L83-L108` (@6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47) - "settings before locales" comment L83, construct L84, locale source L88, `LoadSettings` L105, `RegisterInOptionsUI` L108.

The localization source that supplies every label/tooltip is its own technique - see
the [localization helper](../recipes/localization-helper.md) recipe. Without it the
Options page shows raw locale-ID keys.

## Pitfalls & gotchas

- **No localized label -> raw key on screen.** Every property, tab, and group ID needs
  a matching entry in an `IDictionarySource`. Register the settings object and its
  locale source in `OnLoad`; Magic Mail comments the ordering explicitly ("Settings
  must exist before locales so labels resolve correctly",
  `magic-mail/repo/Mod.cs#L83`).

- **`SetDefaults()` is mandatory and must set every field.** It doubles as the reset
  target; a missed field keeps its zero value.

- **Registration-vs-locale ordering has two valid shapes.** Magic Mail adds locale
  sources, `LoadSettings`, then `RegisterInOptionsUI` (`Mod.cs#L87-L108`); Road Speed
  Adjuster registers *first*, then adds the locale source and loads
  (`road-speed-adjuster/repo/Mod.cs#L28-L33`). Both work because the `Setting`
  instance exists before the locale source is built - that is the real constraint.

- **The rendered panel is not source-provable.** The attribute inputs are verifiable;
  the exact pixel layout, dropdown styling, and confirmation-dialog wording the game
  draws from them are `Needs Verification (in-game)`. This page documents the C# side
  you control, not the Gameface render.

## Variations

- **Immediate apply instead of restart.** Override `ModSetting.Apply()` to push
  changes into live systems the moment an option changes - see step 6 of
  [Settings UI patterns](../recipes/settings-patterns.md).
- **Action buttons, presets, and reset.** Write-only `bool` + `[SettingsUIButton]`
  (optionally `[SettingsUIConfirmation]`), extra preset methods, and `ApplyAndSave()`
  for persistence - all in the recipe.
- **Conditional controls.** `[SettingsUIHideByCondition]` /
  `[SettingsUIDisableByCondition]` gate a control on another property or predicate -
  recipe step 3.
- **Key-binding rows on the same page.** Bindings are declared as `ProxyBinding`
  properties with `[SettingsUIKeyboardBinding]` and live on their own Keybinds
  section - see [Declare key bindings](key-bindings.md).

## See also
- Recipe (depth): [Settings UI patterns](../recipes/settings-patterns.md) - the full
  attribute catalogue this task builds on.
- Recipe: [localization helper](../recipes/localization-helper.md) - the label source
  every control needs.
- How-to: [Declare key bindings](key-bindings.md) - binding rows on the Options page.
- How-to: [Handle a missing UI dependency](dependency-handling.md) - degrade the
  page when a shared UI library is absent.
- Explanation: [mod lifecycle](../../explanation/mod-lifecycle.md) - where `OnLoad`
  registration sits; [UI <-> C# communication](../../explanation/ui-cs-communication.md)
  and [Gameface runtime](../../explanation/gameface-runtime.md) (forward references)
  for what happens above the C# layer.
- Index: [technique index](../../technique-index.md).

## Sources
- Canonical mods (dossier + repo):
  - `magic-mail` @6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47 - `repo/Settings/Settings.cs`, `repo/Mod.cs`
  - `road-speed-adjuster` @e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed - `repo/sETTING.cs`, `repo/Mod.cs`
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding
