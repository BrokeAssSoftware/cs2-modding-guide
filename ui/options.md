# Options UI Patterns

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
2. Define properties with metadata so the Options UI renders correctly:
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

## Localise Labels and Tooltips
- Create a locale provider per language and map option labels/descriptions using `GetOptionLabelLocaleID` and `GetOptionDescLocaleID`.
- Register locales before calling `RegisterInOptionsUI` so labels resolve on first render.
- Listen for `localizationManager.onActiveDictionaryChanged` when the UI must refresh after a language change.

## Synchronise Settings with Systems
- Inject the `Setting` instance into simulation systems or service singletons.
- Apply changes immediately and surface confirmation messages where appropriate.
- For expensive toggles, enable/disable systems dynamically by flipping their `Enabled` flags rather than requiring restarts.

## Options UX Tips
- Group related controls by functional area (Simulation, Analytics, Debug).
- Display current values in headers when useful (for example `Heat multiplier: 1.25x`).
- Always provide Reset buttons for complex sections and confirm destructive operations.
- Apply changes without requiring a restart whenever possible.

See [Options Attribute Reference](reference/options-attributes.md) for attribute details and examples.
