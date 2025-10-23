# Shared Library Pattern (ExtraLib)

ExtraLib is the canonical example of how we ship a shared dependency in the Vice & Order ecosystem. It bundles icon hosts, localisation glue, UI systems, entity utilities, and Harmony patches that multiple micro-mods need. This guide covers both the ExtraLib-specific integration steps and the conventions we follow for any shared runtime dependency.

## Why ExtraLib Sits In Our Stack
- Provides common UI and notification systems (`ExtraPanelsUISystem`, icons, toolbar helpers) so feature mods stay lean.
- Exposes localisation utilities (`ExtraLocalization`) that bridge embedded resources before I18n Everywhere loads.
- Centralises Harmony patches and debug logging so we do not duplicate boilerplate across modules.
- Ships as its own Paradox mod (ID `75724`) with active Crowdin translations and icon assets, which we would otherwise have to maintain per module.

## Quick Start Checklist
- Download the matching ExtraLib release and store the DLL under `dependencies/ExtraLib/<version>/ExtraLib.dll`.
- Add a shared MSBuild reference with `<Private>false</Private>` so the DLL is used at compile time but not copied into any module output.
- Declare ExtraLib as a hard dependency in every code/UI module you publish (PublishConfiguration, `mod.json`, README).
- At runtime, verify the assembly is present and short-circuit optional features with a clear log warning if it is missing.
- Use the public hooks (`EL.AddOnInitialize`, `Icons.LoadIconsFolder`, etc.) instead of re-implementing their behaviour.

## Step-by-Step Integration

### 1. Bring ExtraLib Into The Workspace
1. Download the latest release from <https://mods.paradoxplaza.com/mods/75724/Windows> or GitHub.
2. Drop the compiled DLL and accompanying assets into a versioned folder:
   ```
   dependencies/
     ExtraLib/
       1.4.4/
         ExtraLib.dll
         release-notes.md
   ```
3. Track the version in `dependencies/manifest.yml` (or the ledger you maintain) so automation knows which build is in use.
4. When upgrading, keep the previous folder until all consuming mods are verified against the new API.

### 2. Reference The Assembly From Every Module
Use a central `Directory.Build.props` so all micro-mod projects inherit the dependency:

```xml
<!-- Directory.Build.props at repo root -->
<Project>
  <PropertyGroup>
    <RepoRoot>$([System.IO.Path]::GetFullPath('$(MSBuildThisFileDirectory)'))</RepoRoot>
    <ExtraLibDll>$(RepoRoot)dependencies\ExtraLib\1.4.4\ExtraLib.dll</ExtraLibDll>
  </PropertyGroup>

  <ItemGroup Condition="Exists('$(ExtraLibDll)')">
    <Reference Include="ExtraLib">
      <HintPath>$(ExtraLibDll)</HintPath>
      <Private>false</Private>
    </Reference>
  </ItemGroup>
</Project>
```

If you cannot use a common props file, add the same `<Reference>` block to the individual `.csproj`, pointing the `HintPath` to the shared DLL. Setting `<Private>false</Private>` ensures the compiler sees ExtraLib but the deploy step does not redistribute it (players receive the dependency through Paradox Mods).

### 3. Declare Runtime Dependencies For Publication
- **Code mods (`PublishConfiguration.xml`)**
  ```xml
  <Publish>
    ...
    <Dependency Id="75724" DisplayName="ExtraLib" />
  </Publish>
  ```
- **UI mods / Gameface bundles (`mod.json`)** – add the dependency ID from ExtraLib’s `mod.json` in the `dependencies` array. Always verify the identifier from the current release package so it matches the exact casing used by Paradox Mods.
- **README / workshop copy** – list ExtraLib under “Required Mods” to eliminate support churn.

Our `build` and `publish` scripts should fail fast if any module omits the dependency declaration; keep that validator up to date when you add new micro-mods.

### 4. Guard And Initialise At Runtime
Even with metadata in place, add a defensive check before you touch the API. This keeps developer builds usable when the dependency is absent.

```csharp
using System;
using System.Linq;
using Colossal.Logging;
using ExtraLib;
using ExtraLib.Helpers;
using ExtraLib.Systems;

private static readonly ILog Log = LogManager.GetLogger("VNO.Vice").SetShowsErrorsInUI(false);

public void OnLoad(UpdateSystem updateSystem)
{
    var extraLibAssembly = AppDomain.CurrentDomain
        .GetAssemblies()
        .FirstOrDefault(a => a.GetName().Name == "ExtraLib");

    if (extraLibAssembly == null)
    {
        Log.Warn("ExtraLib missing. Vice loop features that rely on shared UI will stay disabled.");
        return;
    }

    EL.AddOnInitialize(() =>
    {
        // Runs once the main menu finishes initialising.
        RegisterVicePanels(updateSystem.World.GetOrCreateSystemManaged<ExtraPanelsUISystem>());
    });

    EL.AddOnEditEnities(entities =>
    {
        // Example: patch prefabs once the notification system is ready.
        ApplyPrefabOverrides(entities);
    },
    new EntityQueryDesc
    {
        All = new[] { ComponentType.ReadOnly<MyPrefabTag>() }
    });
}
```

Prefer the helpers that ExtraLib ships (`ExtraLocalization.LoadLocalization`, `Icons.LoadIconsFolder`, `MainSystem.AddOnInitialize`, etc.) over custom implementations—this keeps behaviour consistent across modules and simplifies upgrades.

### 5. Reuse Assets And Utilities
- **Icons** – store shared SVGs in ExtraLib and reference them via `coui://extralib/...`. When you add new icons, update the ExtraLib release so every consumer sees them without bundling duplicates.
- **Localization** – for embedded strings, call `ExtraLocalization.LoadLocalization(Logger, Assembly.GetExecutingAssembly())` before I18n Everywhere attaches. This is handy for debug menus or fallback copy.
- **Panels & Notifications** – use `ExtraPanelsUISystem.AddExtraPanel<T>` and `EL.m_NotificationUISystem.AddOrUpdateNotification` rather than reinventing menu scaffolding.
- **Entity edit queue** – schedule expensive prefab or entity mutations through `EL.AddOnEditEnities` so ExtraLib handles progress notifications and coroutine timing.

Document any new conventions you add (naming, icon folders, embedded resource layout) in the module’s `Agents.md` so future contributors follow the same structure.

## Dependency Hygiene For Micro-Mods
- Keep third-party DLLs outside project directories (`dependencies/<Vendor>/<Version>/`) and never check them into `bin/` or `obj/`.
- Version every dependency explicitly; do not “float” to latest without coordinated testing across the module suite.
- Run `dotnet clean` + `dotnet build` after upgrades to ensure no project caches stale metadata about the DLL.
- Enforce dependency declarations via CI (lint `PublishConfiguration.xml`, `mod.json`, and README badges).
- When removing or replacing a shared library, stage the change: drop new dependency first, migrate modules, then remove the old reference once all downstream mods are rebuilt.

## QA & Troubleshooting
- **Missing dependency warning in-game** – confirm the dependency ID is in both `PublishConfiguration.xml` and the Paradox Mods upload form. The player installer only auto-downloads declared dependencies.
- **`FileNotFoundException: ExtraLib` during load** – check the `<HintPath>` in your project file and make sure the DLL is present. A relative path that crosses drive letters often breaks on CI machines.
- **Runtime API changes** – ExtraLib follows semantic versioning. If you upgrade from `1.4.x` to `1.5.x`, re-run automated smoke tests for every feature mod that touches icons, notifications, or prefab hooks.
- **Icon placeholders showing** – ensure the ExtraLib release contains the new SVG and that you referenced the path via `Icons.COUIBaseLocation`. Cached bundles sometimes require a game restart after icon updates.

## References & Further Reading
- ExtraLib repository: <https://github.com/AlphaGaming7780/ExtraLib>
- Dependency overview: `docs/cs2-modding-guide/project-architecture.md`
- UI conventions: `docs/cs2-modding-guide/ui-and-options.md`
- Localization playbook: `docs/cs2-modding-guide/localization/i18n-integration.md`

Follow these steps whenever you introduce or update a shared library so both humans and automation can reason about the dependency graph that powers Vice & Order.
