# UI And Options

Combine the automatic Options UI system with localized strings and input bindings to keep Vice & Order modules discoverable.

## Options UI Essentials

- Derive settings from `ModSetting` and use `[SettingsUISlider]` for numeric ranges, `[SettingsUICheckbox]` or `[SettingsUIToggle]` for booleans, `[SettingsUIEnum]` for enums (hide obsolete entries with `[SettingsUIHidden]`), and `[SettingsUIButton]` plus `[SettingsUIConfirmation]` for actions.
- Group related properties with `[SettingsUISection("SectionName", "Group")]`, and keep the menu predictable by defining `[SettingsUIGroupOrder]`.
- Display group titles explicitly with `[SettingsUIShowGroupName]` when multiple groups share a section (pattern from `RealisticPathFinding`).

## Localization Pipeline

- Create locale classes that implement `IDictionarySource` and populate label/description IDs with `GetOptionLabelLocaleID` and `GetOptionDescLocaleID`.
- Register locales for every supported language before calling `RegisterInOptionsUI()`.
- Listen to `localizationManager.onActiveDictionaryChanged` when options must rebuild after a language change (`AchievementFixer` pattern).
- Keep translations ASCII-friendly; place multi-line copy in dictionary sources rather than inline attributes.

## Key Bindings

- Decorate properties with `[ProxyBinding]` and pair keyboard/gamepad metadata via `[SettingsUIKeyboardAction]` and `[SettingsUIGamepadAction]` (see [Mod Key Binding](https://cs2.paradoxwikis.com/Mod_Key_Binding)).
- Assign unique action IDs and usage strings to prevent conflicts and expose user-friendly names.
- Provide localized captions through `GetBindingMapLocaleID` and `GetBindingKeyLocaleID`.
- Handle binding conflicts by listening to `InputManager.instance` callbacks, surfacing warnings, and exposing reset actions.

## Options UX Guidance

- Mirror the `RealisticPathFinding` approach for dense sliders by clustering under logical sections such as Vehicles, Pedestrians, or Taxi and provide concise tooltips explaining outcomes.
- Keep defaults clear; expose reset buttons with `[SettingsUIButton]` and confirm destructive actions.
- When options affect system activation, toggle systems by setting `Enabled` flags or updating module configuration at runtime so players see changes immediately.

## Runtime Panels

- Use `[SettingsUIDirPicker]` for filesystem selectors when storing exports or logs.
- Expose read-only summaries with `[SettingsUIMultilineText]` for quick player feedback.
- For dynamic UI beyond the options menu, route data through shared services or serialized singletons and update via ECS UI systems.
