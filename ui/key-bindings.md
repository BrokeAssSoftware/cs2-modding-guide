# Key Binding Pipeline

1. Decorate binding properties with metadata:
   ```csharp
   [ProxyBinding("VNO.Core.ToggleVicePanel")]
   [SettingsUIKeyboardAction("Toggle Vice Panel", DefaultKey = KeyCode.V)]
   [SettingsUIGamepadAction("Toggle Vice Panel", DefaultButton = GamepadButton.DPadUp)]
   public Binding ToggleVicePanel { get; set; }
   ```
2. Provide localised captions via `GetBindingMapLocaleID` and `GetBindingKeyLocaleID`.
3. Watch `InputManager.instance.onBindingConflict` to display inline warnings when conflicts occur.
4. Handle the action in code:
   ```csharp
   InputManager.instance.AddBindingHandler("VNO.Core.ToggleVicePanel", _ => ToggleViceDashboard());
   ```
5. Offer reset actions so players can revert to defaults.
