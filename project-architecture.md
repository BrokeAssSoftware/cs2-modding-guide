# Project Architecture

Vice & Order modules follow the standard Cities: Skylines II code-mod layout but add conventions that keep our micro-mod ecosystem maintainable. Use this playbook whenever you scaffold a new module or refactor an existing one.

## Module Layout
- Assembly naming: keep the assembly, namespace root, and published mod ID identical (for example `vno-core`). This avoids type lookup issues and keeps dependency manifests consistent.
- Folder structure:
  ```
  vno-module/
    Mod.cs
    Setting.cs
    Systems/
    Services/
    UI/
    Localization/
    Properties/PublishConfiguration.xml
    Icons/ (optional, see shared icon guide)
    lang/ (for I18n Everywhere embedded locales)
  ```
- Shared MSBuild props: import `Directory.Build.props` (or similar) to centralise references to ExtraLib, UIL, Harmony, and Unity assemblies.
- Environment paths: resolve everything via `EnvPath` or `ToolchainSettings`. Never hard-code absolute paths; this keeps the project portable across Steam, Game Pass, and dev installations.

## Lifecycle Flow
Every `Mod` class should execute these steps in order:
1. Create module folders with `Directory.CreateDirectory` for `ModsSettings/<Module>` and `ModsData/<Module>` so first-run users do not hit IO exceptions.
2. Load settings by instantiating your `Setting` class, calling `AssetDatabase.global.LoadSettings`, and registering the Options UI immediately.
3. Register localization before the Options UI renders so labels resolve on first paint.
4. Detect dependencies (ExtraLib, UIL, I18n Everywhere, etc.) and set fallback flags when they are missing.
5. Disable vanilla systems you intend to replace by setting `Enabled = false` on the existing system instance.
6. Schedule custom systems via `updateSystem.UpdateAt` / `UpdateAfter` / `UpdateBefore` with explicit `SystemUpdatePhase` enums.
7. Apply Harmony patches last, log the patched methods, and store the Harmony instance for unpatching.

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

## Settings, Data, and State
- Settings: derive from `ModSetting`, store user preferences only, and keep runtime caches elsewhere. Persist changes via `AssetDatabase.global.SaveSettings`.
- Long-lived data: store simulation caches, analytics, and save-independent data in `ModsData/<Module>`. Treat the folder like a cache; ship reset buttons to purge safely.
- Session data: anything disposable goes under `ModsDataTemp/<Module>`. Clear it on unload to avoid cruft.
- Configuration schema: document JSON/YAML formats under `docs/modules/` when you release data packs so automation can validate them.

## Localization and Logging
- Register per-locale dictionary sources (`IDictionarySource`). Use your `Setting` instance to derive option keys and keep naming consistent.
- Add at least an English fallback before other locales. When I18n Everywhere is present, let it override keys via JSON to reduce churn.
- Keep a single static logger per module. Disable UI error popups by default and expose debug toggles in settings.
- Log the executable asset path and dependency versions during load. These breadcrumbs are invaluable during support.

## System Scheduling and Replacement
- Disable vanilla systems before registering replacements to avoid double execution. If you may re-enable them later, keep a reference.
- Use `UpdateAt` with explicit phases (`GameSimulation`, `UIUpdate`, `EditorSimulation`, `Deserialize`, `PrefabUpdate`). Deterministic ordering is critical when multiple VNO modules share the same world.
- When chaining systems, prefer `updateSystem.UpdateAfter<TDependency, TSystem>()` so reorderings remain explicit.
- For multi-phase systems, guard inside `OnUpdate`:
  ```csharp
  protected override void OnUpdate()
  {
      if (WorldUnmanaged.Time.DeltaTime == 0f) return; // during deserialize
      if (!Application.isPlaying && !RunInEditor) return;
      // Simulation logic here
  }
  ```
- Use `ComponentLookup` and `SystemHandle` fields to cache queries. Initialise them in `OnCreate` and refresh with `Update(ref state)` patterns to avoid repeated lookups.

## Dependency Strategy
- Treat ExtraLib, Unified Icon Library, I18n Everywhere, and Write Everywhere as required runtime dependencies for modules that rely on them.
- Check assemblies at runtime and set feature flags (`HasExtraLib`, `HasI18n`) so systems can downgrade gracefully.
- House shared wrappers (icon hosts, prefab loaders, notification helpers) in ExtraLib or another dedicated dependency to avoid copy/paste between modules.
- When interacting with third-party mods (for example Time2Work), expose interop helpers that wrap their public API and guard against missing assemblies.

## Performance and Telemetry Targets
- Reference hardware: Intel i7-11700K class CPU, RTX 3070 GPU, 32 GB RAM, 1440p, medium graphics preset.
- Frame budgets: aim for simulation systems under 5 ms per frame, UI systems under 2 ms, and background analytics under 1 ms. Reference this section when writing acceptance criteria.
- Benchmarks: keep deterministic save files for high-crime districts, budget stress tests, and vice escalation scenarios. Run profiling on these saves before shipping features.
- Instrumentation: drop `ProfilerMarker` scopes around heavy jobs and expose developer commands to dump component summaries in profiling builds.

## Shared Terminology
Use these canonical definitions across design docs, stories, and telemetry payloads:
- Identity triangle: tuple `(loyalty, fear, opportunity)` in range `0.0 - 1.0`, default `0.5`. Serialised under `vno_economy.identity`.
- Ideology vector: six-element array `(reformist, developer, lawAndOrder, unionist, populist, viceAligned)`, normalised to sum to `1.0`.
- Influence metric: structure with `current`, `decayRate`, `maxCapacity`, floats between `0.0` and `100.0`.
- Heat index: scalar `0.0 - 100.0`, escalates tiers at `25`, `50`, and `75`.
- Legitimacy / Trust: civic sentiment scores `0.0 - 100.0`, stored in `vno_order.legitimacy` and `vno_governance.trust`.
Add new terms here as soon as they appear in the backlog to keep future stories unambiguous.

## Draft Event and API Contracts
Promote entries from this table into `docs/API_REFERENCE.md` once the implementation stabilises.

| Contract | Summary | Status |
| --- | --- | --- |
| `HeatChangedEvent` | `{ factionId: Guid, previous: float, current: float, delta: float, timestamp: long }` | Draft |
| `GangEvent` | `{ type: enum, territoryId: Guid, actors: Guid[], loyaltyDelta: float, liquidityDelta: float }` | Draft |
| `IFinanceService` | `GetBalances(factionId)`, `InitiateLaunder(job)`, `SetAutoPolicy(policyId, enabled)` | Draft |
| `IVoteService` | `ScheduleSession(config)`, `GetForecast(sessionId)`, `SubmitInfluence(action)` | Draft |
| `IHealthService` | `GetStress(districtId)`, `RegisterProgram(programConfig)`, `ReportOutcome(outcome)` | Draft |

Keep this document close while planning new features; it captures the architectural guardrails that let our micro-mods interoperate without surprises.