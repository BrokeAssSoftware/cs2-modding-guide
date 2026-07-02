# Options Attribute Cheatsheet

Attribute coverage distilled from the Options UI wiki and reference mods. Use this as a quick lookup while wiring settings classes.

## Layout & Grouping

- `[FileLocation("ModsSettings\\<Mod>\\<Mod>")]` - controls where the `.coc` settings file is stored.
- `[SettingsUIGroupOrder("GroupA", "GroupB")]` - defines the render order of groups inside a section.
- `[SettingsUIShowGroupName("GroupA", ...)]` - forces group headers to display even when the section name is already visible.
- `[SettingsUISection("SectionName", "GroupName")]` - places a property inside a UI section and group.
- `[SettingsUIDescription("locale-key")]` - associates a description override with a property (use sparingly; localization dictionaries are preferred).

## Value Inputs

- `[SettingsUISlider(min, max, step, unit, scalarMultiplier)]` - renders a slider; pair with `Unit.kFloatSingleFraction`, `Unit.kFloatTwoFractions`, `Unit.kInt`, etc.
- `[SettingsUICheckbox]` / `[SettingsUIToggle]` - boolean switches (checkbox vs toggle styling).
- `[SettingsUIEnum]` - renders an enum dropdown; combine with `[SettingsUIHidden]` on deprecated entries to keep IDs stable.
- `[SettingsUIText]` / `[SettingsUIMultilineText]` - display read-only strings (useful for summaries, version info).
- `[SettingsUITextInput]` / `[SettingsUIMultilineInput]` - capture user-entered text.
- `[SettingsUIDirPicker]` / `[SettingsUIFilePicker]` - filesystem pickers; validate paths before writing.
- `[SettingsUITextArea]` - legacy alias for multiline text; prefer `[SettingsUIMultilineInput]`.

## Actions & Buttons

- `[SettingsUIButton]` - converts a boolean property with setter into a button; logic lives in the setter.
- `[SettingsUIConfirmation("locale-key")]` - adds a yes/no confirmation before invoking the button action.
- `[SettingsUIButtonGroup("GroupId")]` - groups multiple buttons on one row.
- `[SettingsUIHidden]` - hides a property from the Options UI without removing it from the settings file.

## Key Binding Metadata

- `[ProxyBinding("BindingId")]` - marks a property as a rebindable action.
- `[SettingsUIKeyboardAction("usage", DefaultKey = KeyCode.K)]` - declares the keyboard binding metadata.
- `[SettingsUIGamepadAction("usage", DefaultButton = GamepadButton.X)]` - declares the gamepad metadata.
- `GetBindingMapLocaleID` / `GetBindingKeyLocaleID` - provide localized names for the binding category and key.

## Advanced

- `[SettingsUITransportNetwork]` - specialized picker for transport networks (used sparingly in vanilla).
- `[SettingsUINumericInput]` - accentuates direct number entry for sliders that also support text overrides.
- `[SettingsUICustomControl]` - attaches custom Gameface components; requires UI mod support.

Refer back to the `docs/cs2-modding-guide/ui-and-options.md` playbook for usage examples, and mirror the localization patterns from `code-patterns.md` when wiring labels.

