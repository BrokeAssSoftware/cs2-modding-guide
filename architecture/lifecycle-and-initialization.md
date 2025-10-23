# Lifecycle and Initialization

Follow these steps inside every `Mod` implementation to ensure modules load consistently and cleanly.

## Standard Load Sequence
1. **Create module folders** - call `Directory.CreateDirectory` for `ModsSettings/<Module>` and `ModsData/<Module>` to avoid first-run IO errors.
2. **Load settings** - instantiate the `Setting` class, call `AssetDatabase.global.LoadSettings`, and register the Options UI immediately.
3. **Register localisation** - add dictionary sources before the Options UI renders so labels resolve on first paint.
4. **Detect dependencies** - check for ExtraLib, Unified Icon Library, I18n Everywhere, and other shared libraries; set fallback flags and warnings when they are missing.
5. **Disable vanilla systems** - fetch the Unity world and set `Enabled = false` on systems you will replace.
6. **Schedule custom systems** - register your DOTS systems with explicit `SystemUpdatePhase` configuration.
7. **Apply Harmony patches** - patch last, log the patched methods, and store the Harmony instance so you can unpatch in `OnDispose`.

## Sample `Mod` Implementation
```csharp
public sealed class Mod : IMod
{
    private const string ModuleId = "VNO.Core";
    private const string HarmonyId = "vno.core";
    private static readonly ILog Log = LogManager
        .GetLogger("VNO.Core.Mod")
        .SetShowsErrorsInUI(false);

    private Harmony? _harmony;
    private Setting? _setting;

    public void OnLoad(UpdateSystem updateSystem)
    {
        EnsureDirectories();

        _setting = new Setting(this);
        AssetDatabase.global.LoadSettings(ModuleId, _setting, new Setting(this));
        _setting.RegisterInOptionsUI();

        RegisterLocales(_setting);

        if (!IsAssemblyLoaded("ExtraLib"))
        {
            Log.Warn("ExtraLib missing. Advanced UI features will be disabled.");
            _setting.HasExtraLib = false;
        }

        var world = World.DefaultGameObjectInjectionWorld;
        world.GetOrCreateSystemManaged<Game.Simulation.ResidentAISystem>().Enabled = false;

        updateSystem.UpdateAt<VnoCoreSystem>(SystemUpdatePhase.GameSimulation);
        updateSystem.UpdateAfter<VnoCoreSystem, Game.Simulation.StatisticSystem>(SystemUpdatePhase.GameSimulation);

        _harmony = new Harmony(HarmonyId);
        _harmony.PatchAll(typeof(Mod).Assembly);
        foreach (var method in _harmony.GetPatchedMethods())
        {
            Log.Info($"Patched: {method.Module.Name}:{method.Name}");
        }
    }

    public void OnDispose()
    {
        _setting?.UnregisterInOptionsUI();
        _setting = null;
        _harmony?.UnpatchAll(HarmonyId);
        _harmony = null;
    }

    private static void EnsureDirectories()
    {
        Directory.CreateDirectory(Path.Combine(EnvPath.kUserDataPath, "ModsSettings", ModuleId));
        Directory.CreateDirectory(Path.Combine(EnvPath.kUserDataPath, "ModsData", ModuleId));
    }

    private static void RegisterLocales(Setting setting)
    {
        var manager = GameManager.instance.localizationManager;
        manager.AddSource("en-US", new LocaleEN(setting));
        manager.AddSource("fr-FR", new LocaleFR(setting));
    }

    private static bool IsAssemblyLoaded(string assemblyName) => AppDomain.CurrentDomain
        .GetAssemblies()
        .Any(a => a.GetName().Name.Equals(assemblyName, StringComparison.OrdinalIgnoreCase));
}
```

Keep this template close when scaffolding new modules so every load sequence follows the same conventions.
