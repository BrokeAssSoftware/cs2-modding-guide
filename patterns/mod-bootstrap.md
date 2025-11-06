# Mod Bootstrap Pattern

```csharp
using Colossal.IO.AssetDatabase;
using Colossal.Logging;
using Colossal.PSI.Environment;
using Game;
using Game.Modding;
using Game.SceneFlow;
using HarmonyLib;
using Unity.Entities;

namespace MyModule;

public sealed class Mod : IMod
{
    private const string HarmonyId = "MyModule";
    private static readonly ILog Log = LogManager
        .GetLogger($"{nameof(MyModule)}.{nameof(Mod)}")
        .SetShowsErrorsInUI(false);

    private Harmony? _harmony;
    private Setting? _settings;

    public void OnLoad(UpdateSystem updateSystem)
    {
        Log.Info(nameof(OnLoad));

        var settingsPath = Path.Combine(EnvPath.kUserDataPath, "ModsSettings", nameof(MyModule));
        Directory.CreateDirectory(settingsPath);

        _settings = new Setting(this);
        _settings.RegisterInOptionsUI();

        GameManager.instance.localizationManager.AddSource("en-US", new LocaleEN(_settings));
        AssetDatabase.global.LoadSettings(nameof(MyModule), _settings, new Setting(this));

        var world = World.DefaultGameObjectInjectionWorld;
        world.GetOrCreateSystemManaged<Game.Simulation.SomeVanillaSystem>().Enabled = false;

        updateSystem.UpdateAt<MyCustomSystem>(SystemUpdatePhase.GameSimulation);
        updateSystem.UpdateAfter<AnotherCustomSystem, MyCustomSystem>(SystemUpdatePhase.GameSimulation);

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
```

**Highlights**
- Settings are initialised and registered before `LoadSettings` so localisation keys resolve on first render.
- Vanilla systems are disabled before scheduling replacements.
- Harmony patches are tracked and unpatched in `OnDispose` to support hot reload.
