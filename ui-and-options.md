# UI and Options

Vice & Order exposes most configuration through the built-in Options UI plus custom panels backed by Gameface React. This guide walks through building a complete settings pipeline, handling localization, wiring key bindings, and synchronising UI changes with simulation systems.

## Build a Settings Class Step by Step
1. **Derive from `ModSetting`**
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
2. **Define properties**
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
3. **Load and register in `Mod.OnLoad`**
   ```csharp
   _setting = new Setting(this);
   AssetDatabase.global.LoadSettings(Id, _setting, new Setting(this));
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

       private static KeyValuePair<string, string> Map(string key, string value) =>
           new KeyValuePair<string, string>(key, value);
   }
   ```
2. Register before calling `RegisterInOptionsUI`:
   ```csharp
   LocalizationManager.AddSource("en-US", new LocaleEN(_setting));
   LocalizationManager.AddSource("fr-FR", new LocaleFR(_setting));
   ```
3. Listen for dictionary changes when UI state must refresh immediately:
   ```csharp
   GameManager.instance.localizationManager.onActiveDictionaryChanged += OnLocaleChanged;
   ```
   Rebuild cached UI labels when this event fires.

## Synchronise Settings with Systems
- Inject your `Setting` instance into simulation systems through singletons or constructor parameters.
- Use `ComponentLookup` or shared singletons to propagate runtime values. Example:
  ```csharp
  protected override void OnUpdate()
  {
      if (!_settings.EnableViceLoop) return;
      var multiplier = _settings.HeatMultiplier;
      // Apply multiplier in simulation logic
  }
  ```
- For expensive changes (e.g., toggling entire systems) register callbacks on the setting so you can enable/disable systems dynamically without restarting.

## Key Binding Pipeline
1. Decorate the setting property:
   ```csharp
   [ProxyBinding("VNO.Core.ToggleVicePanel")]
   [SettingsUIKeyboardAction("Toggle Vice Panel", DefaultKey = KeyCode.V)]
   [SettingsUIGamepadAction("Toggle Vice Panel", DefaultButton = GamepadButton.DPadUp)]
   public Binding ToggleVicePanel { get; set; }
   ```
2. Provide localized captions:
   ```csharp
   _setting.GetBindingMapLocaleID(nameof(Setting.ToggleVicePanel)) => "Bindings.MAP[VNO.Core]";
   _setting.GetBindingKeyLocaleID(nameof(Setting.ToggleVicePanel)) => "Bindings.KEY[VNO.Core.ToggleVicePanel]";
   ```
3. Resolve conflicts by watching `InputManager.instance.onBindingConflict` and surfacing a toast or inline warning when the player selects a conflicting key.
4. Implement an action handler in a UI or simulation system:
   ```csharp
   InputManager.instance.AddBindingHandler(
       "VNO.Core.ToggleVicePanel",
       _ => ToggleViceDashboard());
   ```

## Options UX Guidelines
- Group sliders, toggles, and actions by functional area (e.g., Simulation, Analytics, Debug) and keep sections short.
- Show current values in the section header when small adjustments are expected (for example `Heat multiplier: 1.25x`).
- Always provide a Reset button for complex sections and confirm destructive actions with `[SettingsUIConfirmation]`.
- When options affect live systems, apply changes immediately and display a short success message so players know the update landed.

## Runtime UI Beyond Options
- **Read-only summaries** – use `[SettingsUIMultilineText]` to display analytics or current system status inside the options view.
- **Dynamic panels** – for custom dashboards, build Gameface React components that subscribe to ECS buffers or query services via message channels (see `ui-react-pipeline.md`).
- **File pickers** – `[SettingsUIDirPicker]` and `[SettingsUIFilePicker]` are ideal for export locations or importing data packs; validate paths before saving.

## Handling Dependency Failures Gracefully
- Detect missing ExtraLib, UIL, or I18n Everywhere instances during `OnLoad` and set boolean flags in your settings class.
- Disable UI sections that require absent dependencies and display a short explanation inside the options menu rather than throwing.
- Provide a button that opens the dependency mod page or documentation so players can install the missing requirement easily.

## Testing Checklist
1. Build in Debug configuration and open the Options menu with translation sets (English plus at least one non-Latin locale) to ensure labels resolve.
2. Change each setting and confirm persistence after restarting the game.
3. Rebind every key and validate that conflicts raise visible warnings.
4. Run with dependencies removed to confirm fallbacks behave correctly.
5. Trigger developer mode and confirm debug-only sections stay hidden in release builds.

Following this workflow ensures every Vice & Order module ships with predictable configuration, clear localization, and responsive UI panels that integrate cleanly with the underlying simulation.
