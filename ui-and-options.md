# UI and Options

Vice & Order exposes most configuration through the built-in Options UI plus custom panels backed by Gameface React. This guide walks through building a settings pipeline, handling localization, wiring key bindings, and synchronising UI changes with simulation systems.

## Build a Settings Class
1. Derive from `ModSetting` and apply layout attributes:
   ```csharp
   [FileLocation("ModsSettings\\VNO.Core\\VNO.Core")]
   [SettingsUIGroupOrder(GeneralGroup, DebugGroup)]
   [SettingsUIShowGroupName(GeneralGroup)]
   public sealed class Setting : ModSetting
   {
       public const string GeneralGroup = "General";
       public const string DebugGroup = "Debug";

       public Setting(IMod mod) : base(mod) { }
   }
   ```
2. Define properties with UI metadata:
   ```csharp
   [SettingsUISlider(0.1f, 5.0f, 0.1f, Unit.kFloatSingleFraction)]
   [SettingsUISection("Simulation", GeneralGroup)]
   public float HeatMultiplier { get; set; } = 1.0f;

   [SettingsUICheckbox]
   [SettingsUISection("Simulation", GeneralGroup)]
   public bool EnableViceLoop { get; set; } = true;

   [SettingsUIButton]
   [SettingsUISection("Developer", DebugGroup)]
   [SettingsUIConfirmation("Options.CONFIRM[VNO.Core.Reset]")]
   public bool ResetDefaults
   {
       set
       {
           SetDefaults();
           AssetDatabase.global.SaveSettings(Mod, this);
       }
   }
   ```
3. Load and register in `Mod.OnLoad`:
   ```csharp
   _setting = new Setting(this);
   AssetDatabase.global.LoadSettings(ModuleId, _setting, new Setting(this));
   _setting.RegisterInOptionsUI();
   ```

## Localize Labels and Tooltips
1. Create a locale provider per language:
   ```csharp
   public sealed class LocaleEN : IDictionarySource
   {
       private readonly Setting _setting;
       public LocaleEN(Setting setting) => _setting = setting;
       public string name => "VNO Core Locale EN";

       public IEnumerable<KeyValuePair<string, string>> Pairs
       {
           get
           {
               yield return Map(_setting.GetOptionLabelLocaleID(nameof(Setting.HeatMultiplier)), "Heat multiplier");
               yield return Map(_setting.GetOptionDescLocaleID(nameof(Setting.HeatMultiplier)), "Scales heat gain per simulation tick.");
           }
       }

       private static KeyValuePair<string, string> Map(string key, string value) => new(key, value);
   }
   ```
2. Register locales before calling `RegisterInOptionsUI`:
   ```csharp
   var manager = GameManager.instance.localizationManager;
   manager.AddSource("en-US", new LocaleEN(_setting));
   manager.AddSource("fr-FR", new LocaleFR(_setting));
   ```
3. Listen for dictionary changes when UI state must refresh immediately:
   ```csharp
   GameManager.instance.localizationManager.onActiveDictionaryChanged += OnLocaleChanged;
   ```

## Synchronise Settings with Systems
- Inject the `Setting` instance into systems via constructors, singletons, or shared services.
- Apply changes immediately and surface confirmation messages so players know updates landed.
- For costly toggles, enable or disable systems dynamically by flipping `Enabled` flags.

## Key Binding Pipeline
1. Decorate the binding property:
   ```csharp
   [ProxyBinding("VNO.Core.ToggleVicePanel")]
   [SettingsUIKeyboardAction("Toggle Vice Panel", DefaultKey = KeyCode.V)]
   [SettingsUIGamepadAction("Toggle Vice Panel", DefaultButton = GamepadButton.DPadUp)]
   public Binding ToggleVicePanel { get; set; }
   ```
2. Provide localized captions via `GetBindingMapLocaleID` / `GetBindingKeyLocaleID`.
3. Watch `InputManager.instance.onBindingConflict` and show inline warnings when conflicts occur.
4. Handle the action in code:
   ```csharp
   InputManager.instance.AddBindingHandler("VNO.Core.ToggleVicePanel", _ => ToggleViceDashboard());
   ```

## Options UX Tips
- Group related toggles and sliders by functional area (Simulation, Analytics, Debug).
- Display current values in section headers for complex sliders (for example `Heat multiplier: 1.25x`).
- Always provide a Reset button with confirmation for destructive options.
- Apply changes without requiring a game restart whenever possible.

## Runtime UI Beyond Options
- Use `[SettingsUIMultilineText]` to show read-only summaries (status, telemetry).
- For dynamic dashboards, build Gameface React components that read data from ECS buffers or service APIs.
- `[SettingsUIDirPicker]` and `[SettingsUIFilePicker]` are ideal for export/import paths; validate the path before saving.

## Dependency Handling
- Detect missing ExtraLib, Unified Icon Library, or I18n Everywhere during `OnLoad` and set flags so UI sections can disable themselves.
- Provide links or guidance that help players install missing dependencies without leaving the game confused.

## Testing Checklist
1. Build in Debug configuration and verify the Options menu renders correctly in at least two languages.
2. Change each setting, reload the game, and confirm persistence.
3. Rebind every key and ensure conflicts display warnings.
4. Launch without shared dependencies to confirm graceful fallbacks.
5. Enable `-developerMode` and verify debug-only sections stay hidden in release builds.

Follow this workflow to deliver predictable configuration experiences across the Vice & Order module stack.
