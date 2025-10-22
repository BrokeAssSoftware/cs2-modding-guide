# Code Patterns

Reusable snippets distilled from community mods and the vanilla templates. Treat these as starting points and adapt naming to the Vice & Order module that owns the behaviour.

## `Mod.cs` Bootstrap

```csharp
using Colossal.IO.AssetDatabase;
using Colossal.Logging;
using Colossal.PSI.Environment;
using Game;
using Game.Modding;
using Game.SceneFlow;
using HarmonyLib;
using Unity.Entities;

namespace MyModule
{
    public sealed class Mod : IMod
    {
        private const string HarmonyId = "MyModule";
        private static readonly ILog Log =
            LogManager.GetLogger($"{nameof(MyModule)}.{nameof(Mod)}")
                      .SetShowsErrorsInUI(false);

        private Harmony? _harmony;
        private Setting? _settings;

        public void OnLoad(UpdateSystem updateSystem)
        {
            Log.Info(nameof(OnLoad));

            // Ensure per-module folders exist.
            var settingsPath = Path.Combine(EnvPath.kUserDataPath, "ModsSettings", nameof(MyModule));
            Directory.CreateDirectory(settingsPath);

            // Hydrate settings and register options.
            _settings = new Setting(this);
            _settings.RegisterInOptionsUI();
            GameManager.instance.localizationManager.AddSource("en-US", new LocaleEN(_settings));
            AssetDatabase.global.LoadSettings(nameof(MyModule), _settings, new Setting(this));

            // Disable vanilla systems before scheduling replacements.
            var world = World.DefaultGameObjectInjectionWorld;
            world.GetOrCreateSystemManaged<Game.Simulation.SomeVanillaSystem>().Enabled = false;

            updateSystem.UpdateAt<MyCustomSystem>(SystemUpdatePhase.GameSimulation);
            updateSystem.UpdateAfter<AnotherCustomSystem, MyCustomSystem>(SystemUpdatePhase.GameSimulation);

            // Apply Harmony patches and log the result.
            _harmony = new Harmony(HarmonyId);
            _harmony.PatchAll(typeof(Mod).Assembly);
            foreach (var method in _harmony.GetPatchedMethods())
            {
                Log.Info($"Patched: {method.DeclaringType?.FullName}.{method.Name}");
            }
        }

        public void OnDispose()
        {
            Log.Info(nameof(OnDispose));

            _settings?.UnregisterInOptionsUI();
            _settings = null;

            _harmony?.UnpatchAll(HarmonyId);
            _harmony = null;
        }
    }
}
```

**Highlights**
- Matches the Realistic Path Finding pattern: settings first, localization before `LoadSettings`, then system scheduling.
- Stores the Harmony instance so it can be unpatched on dispose (useful for hot reload).
- Uses `Directory.CreateDirectory` defensively in case the user deletes the settings folder.

## `Setting.cs` Essentials

```csharp
using Colossal.IO.AssetDatabase;
using Game.Modding;
using Game.Settings;
using Game.UI;

[FileLocation("ModsSettings\\MyModule\\MyModule")]
[SettingsUIGroupOrder(BalanceGroup, DebugGroup)]
[SettingsUIShowGroupName(BalanceGroup)]
public sealed class Setting : ModSetting
{
    public const string BalanceGroup = "Balance";
    public const string DebugGroup = "Debug";

    public Setting(IMod mod) : base(mod)
    {
        if (Modifier <= 0) SetDefaults();
    }

    public override void SetDefaults()
    {
        Modifier = 1.0f;
        EnableDebug = false;
    }

    [SettingsUISlider(min: 0.1f, max: 3.0f, step: 0.1f, unit: Unit.kFloatSingleFraction)]
    [SettingsUISection("Gameplay", BalanceGroup)]
    public float Modifier { get; set; }

    [SettingsUICheckbox]
    [SettingsUISection("Developer", DebugGroup)]
    public bool EnableDebug { get; set; }

    [SettingsUIButton]
    [SettingsUISection("Developer", DebugGroup)]
    public bool ResetDefaults
    {
        set
        {
            SetDefaults();
            AssetDatabase.global.SaveSettings(Mod, this);
        }
    }
}
```

**Highlights**
- Uses group attributes to keep related sliders together, mirroring Realistic Path Finding's approach.
- Shows how to expose a reset button that persists the new values.
- Settings constructor calls `SetDefaults` when fields are at default CLR values (mirrors the community mods).

## Localization Dictionary

```csharp
using Colossal.Localization;

public sealed class LocaleEN : IDictionarySource
{
    private readonly Setting _setting;

    public LocaleEN(Setting setting) => _setting = setting;

    public string name => "MyModule Locale (EN)";

    public IEnumerable<KeyValuePair<string, string>> Pairs
    {
        get
        {
            yield return Map(_setting.GetOptionLabelLocaleID(nameof(Setting.Modifier)),
                             "Simulation modifier");
            yield return Map(_setting.GetOptionDescLocaleID(nameof(Setting.Modifier)),
                             "Scales all simulated values driven by this module.");
        }
    }

    public void Unload() { }

    private static KeyValuePair<string, string> Map(string key, string value) =>
        new KeyValuePair<string, string>(key, value);
}
```

**Highlights**
- Mirrors `AchievementFixer` and `RealisticPathFinding` by sourcing localization keys from the setting instance.
- Additional locales can inherit from the same pattern, swapping text only.

## Time2Work-Style Multi-Phase System Registration

```csharp
updateSystem.UpdateAt<MySimulationSystem>(SystemUpdatePhase.GameSimulation);
updateSystem.UpdateAt<MySimulationSystem>(SystemUpdatePhase.Deserialize);
updateSystem.UpdateAt<MyUISystem>(SystemUpdatePhase.UIUpdate);
```

- Registering the same system in multiple phases ensures consistent data when the world loads or when running inside the editor.
- Always confirm the system guards side effects by checking the active phase or application state.

Use these fragments to jump-start new modules, and keep the full implementations in the module repository for deeper context.
