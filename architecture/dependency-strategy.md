# Dependency Strategy

Vice & Order relies on shared libraries such as Unified Icon Library, I18n Everywhere, and Write Everywhere. Coordinate these dependencies so modules degrade gracefully when they are missing and our tooling can reason about the stack. ExtraLib is no longer a baseline requirement but the same patterns below apply if a module opts into it (see `dependencies/shared-library-extra.md`).

## Standard Dependency Workflow
1. **Declare** the dependency in publishing metadata (`PublishConfiguration.xml`, `mod.json`, documentation).
2. **Reference** the assembly for compilation without bundling it inside the module.
3. **Validate at runtime** during `Mod.OnLoad` (or the module's bootstrap entry point) and capture the result in module state.
4. **Downgrade gracefully** when the dependency is absent and surface a single warning.
5. **Automate** validation in CI to catch missing declarations or stale paths.

Treat this workflow as the checklist every time a new shared library is introduced or updated.

## Declaring Dependencies

- **Publish configuration** â€“ declare every required mod so the in-game publisher installs prerequisites automatically:
  ```xml
  <Publish>
    <!-- other metadata -->
    <Dependency Id="74417" DisplayName="Unified Icon Library" />
    <Dependency Id="75426" DisplayName="I18n Everywhere" />
  </Publish>
  ```
- **UI module `mod.json`** â€“ reflect the same dependency list for Gameface packages:
  ```json
  {
    "id": "VNO.UI",
    "dependencies": [
      "algernon.UnifiedIconLibrary",
      "baka.I18NEverywhere"
    ]
  }
  ```
- **Release notes / READMEs** â€“ mark each dependency as "Required" with a link to the Paradox Mods page to reduce support churn.
- **MSBuild references** - point project files at the shared DLL location with `<Private>false</Private>` so the dependency is resolved at compile time but not duplicated in the output. If you add optional libraries (for example ExtraLib), mirror the guidance in `docs/cs2-modding-guide/dependencies/shared-library-extra.md`.

Keep declarations synchronized across every file when adding or removing a dependency.

## Runtime Validation

Metadata alone cannot guarantee load order. Run a defensive assembly check during `OnLoad` and persist the status for downstream systems:

```csharp
using System;
using System.Linq;
using Colossal.Logging;
using Game;
using HarmonyLib;

public sealed class Mod : IMod
{
    private static readonly ILog Log = LogManager
        .GetLogger("VNO.Core.Mod")
        .SetShowsErrorsInUI(false);

    private readonly DependencyStatus _dependencies = new();

    public void OnLoad(UpdateSystem updateSystem)
    {
        _dependencies.Refresh();

        if (!_dependencies.HasUnifiedIconLibrary)
        {
            Log.Warn("Unified Icon Library missing. Falling back to text labels.");
        }

        if (!_dependencies.HasI18nEverywhere)
        {
            Log.Warn("I18n Everywhere missing. Localisation will default to embedded English strings.");
        }

        // Gate feature registration on dependency availability.
        if (_dependencies.HasUnifiedIconLibrary)
        {
            UilBootstrap.Register();
        }
        else
        {
            FallbackBootstrap.Register();
        }
    }
}

internal sealed class DependencyStatus
{
    public bool HasUnifiedIconLibrary { get; private set; }
    public bool HasI18nEverywhere { get; private set; }

    public void Refresh()
    {
        HasUnifiedIconLibrary = IsAssemblyLoaded("UnifiedIconLibrary");
        HasI18nEverywhere = IsAssemblyLoaded("I18NEverywhere");
    }

    private static bool IsAssemblyLoaded(string assemblyName) =>
        AppDomain.CurrentDomain
            .GetAssemblies()
            .Any(a => a.GetName().Name.Equals(assemblyName, StringComparison.OrdinalIgnoreCase));
}
```

Use the pattern above as the baseline and extend `DependencyStatus` as new shared modules come online. When multiple modules need the same guard, promote the helper into a shared utility assembly (for example `vno-core`) to avoid drift.

## Downgrading Behaviour

- **Feature flags** â€“ expose `HasUIL`, `HasI18n`, and similar booleans on settings classes so UI layers can branch cleanly.
- **UI fallbacks** â€“ replace icon URIs with plain text labels or neutral assets when UIL is unavailable.
- **System scheduling** â€“ skip registration of systems that rely on a missing dependency instead of letting them throw inside `OnUpdate`.
- **Logging** â€“ log one warning per missing dependency during `OnLoad`; avoid spamming the log on every frame.

## Inter-Mod Hooks

- Wrap third-party APIs (for example Time2Work) behind typed adapters that check for assembly presence and guard every entry point.
- Expose an explicit capability list via events or interfaces so other mods can detect when Vice & Order features are available (and vice versa).

## Dependency Hygiene

- Keep third-party DLLs in `dependencies/<Vendor>/<Version>/` and never commit them under `bin/` or `obj/`.
- Version dependencies explicitly and avoid floating to "latest" without coordinated testing across modules.
- Add CI checks that verify:
  - Declared dependencies exist in `PublishConfiguration.xml` and `mod.json`.
  - Every assembly referenced in code has a corresponding publish declaration.
  - Shared DLL hint paths still resolve.

Following this checklist keeps dependency handling predictable, reduces runtime surprises, and makes it easier for automation (and future contributors) to reason about the Vice & Order stack.
