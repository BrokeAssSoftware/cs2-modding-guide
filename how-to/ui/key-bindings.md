---
FrontmatterVersion: 1
DocumentType: Guide
Title: "How-to: Declare key bindings and read input"
diataxis: how-to
source_version: "~1.5.9 (anarchy@a6311e8; date-pinned, static source only)"
last_reverified: "2026-07-04"
canonical_mods:
  - anarchy@a6311e898d20a775368668b234aaa32f06e3e1eb
technique_applicability: [core, ui]
status: source-verified
Created: 2026-07-03
Updated: 2026-07-03
Owners:
  - codex
---

# Declare key bindings and read input

> Give your mod rebindable keyboard shortcuts that show up in the game's Options
> **Keybinds** section, then read them each frame from a system - using
> `[SettingsUIKeyboardAction]` + `[SettingsUIKeyboardBinding]` on `ProxyBinding`
> properties and `GetAction(...)` on the settings object.

## Problem
Your mod needs a hotkey - toggle a tool, nudge a value, open a panel. You want the
default key to appear in the vanilla **Options -> Keybinds** UI, be rebindable by the
player, persist, and be readable from your systems without polling raw `KeyCode`s.

## Solution
Key bindings ride on the same `ModSetting` machinery as the rest of your options.
Declare **named actions** with class-level `[SettingsUIKeyboardAction]` attributes,
then expose one `ProxyBinding` property per binding decorated with
`[SettingsUIKeyboardBinding(...)]` (which sets the default key and modifiers and ties
the property to its action name). At runtime, resolve each action once with
`settings.GetAction(actionName)` to get a `ProxyAction`, gate it with
`shouldBeEnabled`, and test `WasPerformedThisFrame()` (or `ReadValue<float>()` for an
axis) inside a system's `OnUpdate`. Bindings appear on a `[SettingsUISection]` you name
"Keybinds"; a reset button calls the inherited `ResetKeyBindings()`.

## Steps & Code

### 1. Declare named actions on the settings class

Each hotkey is a **named action**. Declare them once as class-level attributes,
giving the action name, its `ActionType`, and the usage contexts in which it is live
(tool usage, a custom map, etc.). Keep the names as constants so the property and the
runtime lookup can share them:

```csharp
[SettingsUIKeyboardAction(ToggleAnarchyActionName, ActionType.Button, new string[] { Usages.kToolUsage })]
[SettingsUIKeyboardAction(ResetElevationActionName, ActionType.Button, new string[] { Usages.kToolUsage })]
[SettingsUIKeyboardAction(ElevationActionName, ActionType.Button, new string[] { "Anarchy" })]
public class AnarchyModSettings : ModSetting
{
    public const string ToggleAnarchyActionName  = "ToggleAnarchy";
    public const string ResetElevationActionName = "ResetElevation";
    public const string ElevationActionName      = "Elevation";
    // ...
}
```
Source: `../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/Settings/AnarchyModSettings.cs#L23-L27` (action declarations) and `#L78-L98` (name constants) (@a6311e898d20a775368668b234aaa32f06e3e1eb).

### 2. Expose a `ProxyBinding` property per binding with its default key

Each binding is a `ProxyBinding` property with `[SettingsUIKeyboardBinding(key,
actionName, modifiers)]`. The attribute sets the **default** key (players can rebind)
and links the property to its action name. Modifiers are named args (`ctrl:`, `alt:`,
`shift:`). Place bindings on a section named for the Keybinds tab/group:

```csharp
[SettingsUISection(Keybinds, Stable)]
[SettingsUIKeyboardBinding(BindingKeyboard.A, actionName: ToggleAnarchyActionName, ctrl: true)]
public ProxyBinding ToggleAnarchy { get; set; }

[SettingsUISection(Keybinds, Stable)]
[SettingsUIKeyboardBinding(BindingKeyboard.R, actionName: ResetElevationActionName, alt: true)]
public ProxyBinding ResetElevation { get; set; }
```
Source: `../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/Settings/AnarchyModSettings.cs#L256-L272` (@a6311e898d20a775368668b234aaa32f06e3e1eb) - Ctrl+A toggle L256-258, Alt+R reset L263-265, Alt+E step L270-272.

For a two-direction control (increase/decrease), bind two `ProxyBinding`s to the
**same** action with `AxisComponent.Positive` / `AxisComponent.Negative`:

```csharp
[SettingsUIKeyboardBinding(BindingKeyboard.PageUp, AxisComponent.Positive, actionName: ElevationActionName)]
public ProxyBinding IncreaseElevation { get; set; }

[SettingsUIKeyboardBinding(BindingKeyboard.PageDown, AxisComponent.Negative, actionName: ElevationActionName)]
public ProxyBinding DecreaseElevation { get; set; }
```
Source: `../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/Settings/AnarchyModSettings.cs#L301-L312` (@a6311e898d20a775368668b234aaa32f06e3e1eb).

### 3. Localize the binding labels

Binding rows need localized captions like any control - supply them from the same
`IDictionarySource` your settings use, via `GetBindingKeyLocaleID(actionName)`:

```csharp
{ m_Setting.GetBindingKeyLocaleID(AnarchyModSettings.ToggleAnarchyActionName), "Press key" },
```
Source: `../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/Settings/LocaleEN.cs#L96` (@a6311e898d20a775368668b234aaa32f06e3e1eb). See [localization helper](../recipes/localization-helper.md).

### 4. Resolve the action and read it from a system

In a system's `OnCreate`, resolve each action once from the settings object into a
`ProxyAction` field. Gate it with `shouldBeEnabled` (so the binding is only live in the
right game mode), then poll it in `OnUpdate`:

```csharp
// OnCreate: resolve once
m_ToggleAnarchy = AnarchyMod.Instance.Settings.GetAction(AnarchyModSettings.ToggleAnarchyActionName);

// OnUpdate: gate + read
m_ToggleAnarchy.shouldBeEnabled = mode.IsGameOrEditor();
if (m_ToggleAnarchy.WasPerformedThisFrame())
{
    AnarchyToggled();
}
```
Source: `../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/Systems/Common/AnarchyUISystem.cs#L281` (resolve), `#L345` (`shouldBeEnabled`), `#L381-L383` (read + act) (@a6311e898d20a775368668b234aaa32f06e3e1eb).

For an axis action, `ReadValue<float>()` gives the signed magnitude (+1 / -1) so you
can act on direction:

```csharp
if (m_ElevationKey.WasPerformedThisFrame())
{
    ChangeElevation(m_ElevationStep.Value * m_ElevationKey.ReadValue<float>());
}
```
Source: `../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/Systems/Common/AnarchyUISystem.cs#L398-L400` (@a6311e898d20a775368668b234aaa32f06e3e1eb).

### 5. Offer a "reset keybinds" button

A write-only `bool` + `[SettingsUIButton]` that calls the inherited
`ResetKeyBindings()` then `ApplyAndSave()` lets players revert to defaults:

```csharp
[SettingsUIButton]
[SettingsUIConfirmation]
[SettingsUISection(Keybinds, Reset)]
public bool ResetKeybindSettings
{
    set
    {
        UseElevationMimics = true;
        ResetKeyBindings();
        ApplyAndSave();
    }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/Settings/AnarchyModSettings.cs#L317-L327` (@a6311e898d20a775368668b234aaa32f06e3e1eb). `ResetKeyBindings()` is an inherited `ModSetting` member.

## Handling conflicts

The safest way to avoid clashing with a vanilla shortcut is to **mimic** the game's
own binding rather than claim a fresh key. `[SettingsUIBindingMimic(map, actionName)]`
(paired with a `[SettingsUIKeyboardBinding]`/`[SettingsUIMouseBinding]` and usually
`[SettingsUIHidden]`) makes your action follow an existing engine binding, so it moves
with the vanilla key and never double-binds:

```csharp
[SettingsUIKeyboardBinding(AxisComponent.Positive, actionName: ElevationMimicActionName)]
[SettingsUIBindingMimic(InputManager.kShortcutsMap, "Change Elevation")]
[SettingsUIHidden]
public ProxyBinding IncreaseElevationMimic { get; set; }
```
Source: `../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/Settings/AnarchyModSettings.cs#L278-L281` (@a6311e898d20a775368668b234aaa32f06e3e1eb); a mouse-binding mimic of "Secondary Apply" at `#L214-L216`.

`[SettingsUIDisableByCondition]` is also used to grey out the manual bindings while the
mimic set is active, so only one of the two paths is editable at a time
(`AnarchyModSettings.cs#L302-L312`, disabled by `UseElevationMimics`).

> **Needs Verification (in-game).** A runtime conflict-notification API - listening for
> a "binding conflict" event to surface an inline warning when a player picks an
> already-used key - is **not** demonstrated in the cited sources. Anarchy avoids
> conflicts structurally (mimic + disable-by-condition) rather than reacting to a
> conflict event. The exact name/shape of any conflict callback and how the vanilla
> rebind dialog reports a clash is unverified here; treat conflict *avoidance* (mimic
> vanilla bindings, prefer modified keys) as the source-verified guidance.

## Pitfalls & gotchas

- **Action name must match between attribute and lookup.** The string in
  `[SettingsUIKeyboardAction]` / `[SettingsUIKeyboardBinding(actionName:)]` and the one
  passed to `GetAction(...)` must be identical - share a `const string` to avoid typos.

- **Resolve once, read every frame.** `GetAction` returns a `ProxyAction`; cache it in
  `OnCreate` and only test `WasPerformedThisFrame()` / `ReadValue<T>()` in `OnUpdate`.
  Re-resolving per frame is wasteful.

- **Gate with `shouldBeEnabled`.** An action left always-enabled fires in menus and
  loading screens. Anarchy sets `shouldBeEnabled = mode.IsGameOrEditor()` each update
  (`AnarchyUISystem.cs#L345`).

- **`[SettingsUIKeyboardBinding]` sets the *default*, not a lock.** Players rebind in
  Options; your default is just the starting value and the reset target.

- **Rendered rebind UI is not source-provable.** The attributes and runtime reads are
  verifiable; the exact Keybinds-row rendering and the rebind capture dialog are
  `Needs Verification (in-game)`.

## Variations

- **Gamepad bindings.** Keyboard actions have gamepad counterparts declared the same
  way (a gamepad action attribute + a binding). Not exercised in the cited Anarchy
  source at this commit -> `Needs Verification`.
- **Mouse bindings.** `[SettingsUIMouseBinding]` + `[SettingsUIMouseAction]` bind mouse
  buttons; Anarchy uses a hidden mimic of the tool's "Secondary Apply"
  (`AnarchyModSettings.cs#L214-L216`).
- **Multiple usage contexts.** The `usages` array on `[SettingsUIKeyboardAction]`
  scopes an action to a tool/map so it only competes for the key while that context is
  active (`AnarchyModSettings.cs#L23-L27`).

## See also
- How-to: [Build a mod Options-menu UI](options-ui.md) - the settings page these
  bindings live on.
- Recipe: [Settings UI patterns](../recipes/settings-patterns.md) - the wider
  `[SettingsUI*]` attribute family; [localization helper](../recipes/localization-helper.md)
  for binding captions.
- Explanation: [UI <-> C# communication](../../explanation/ui-cs-communication.md)
  (forward reference) - how input flows between the UI layer and systems.
- Index: [technique index](../../technique-index.md).

## Sources
- Canonical mods (dossier + repo):
  - `anarchy` @a6311e898d20a775368668b234aaa32f06e3e1eb - `repo/Anarchy/Settings/AnarchyModSettings.cs`, `repo/Anarchy/Settings/LocaleEN.cs`, `repo/Anarchy/Systems/Common/AnarchyUISystem.cs`
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding
