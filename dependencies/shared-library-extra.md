# Shared Library Pattern (ExtraLib)

ExtraLib is the canonical example of how we ship a shared dependency in the Vice & Order ecosystem. It bundles icon hosts, localisation helpers, UI systems, entity utilities, and Harmony patches that multiple micro-mods need. This guide covers how to wire the dependency, reference its API safely, and share assets without duplicating work.

## Quick Start Checklist
- Install the matching ExtraLib release and store the DLL under `dependencies/ExtraLib/<version>/ExtraLib.dll`.
- Add a shared MSBuild reference with `<Private>false</Private>` so the DLL is used at compile time but not copied into module outputs.
- Declare ExtraLib as a hard dependency in code and UI modules (`PublishConfiguration.xml`, `mod.json`, README).
- At runtime, verify the assembly is present and downgrade gracefully when it is missing.
- Use the public hooks (`EL.AddOnInitialize`, `Icons.LoadIconsFolder`, etc.) instead of re-implementing their behaviour.

## Declare the Dependency
- **Publish configuration**
  ```xml
  <Publish>
    ...
    <Dependency Id="75724" DisplayName="ExtraLib" />
  </Publish>
  ```
- **UI module `mod.json`**
  ```json
  {
    "id": "VNO.UI",
    "dependencies": ["algernon.ExtraLib"]
  }
  ```
  Confirm the dependency identifier and casing against the current ExtraLib release.
- **Documentation** - list ExtraLib as required in module READMEs and release notes so players install it alongside Vice & Order modules.

## Runtime Guard
Even with metadata in place, add a defensive check before using the API:
```csharp
private static bool IsExtraLibAvailable()
{
    return AppDomain.CurrentDomain
        .GetAssemblies()
        .Any(a => a.GetName().Name.Equals("ExtraLib", StringComparison.OrdinalIgnoreCase));
}
```
When `IsExtraLibAvailable()` returns `false`, swap icons or panels for text fallbacks and log a single warning.

## Reference the Assembly from Projects
Use a shared `Directory.Build.props` to hook the DLL into every module:
```xml
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
Setting `<Private>false</Private>` ensures the DLL is referenced at compile time but not bundled with the mod, because players install ExtraLib separately.

## Use the Provided Helpers
- **Icons** - store shared SVGs in ExtraLib and reference them via `coui://extralib/...`. Update the dependency when you add icons so every consumer picks them up.
- **Localization** - call `ExtraLocalization.LoadLocalization(Logger, Assembly.GetExecutingAssembly())` for embedded dictionaries before I18n Everywhere attaches.
- **Panels and notifications** - use `ExtraPanelsUISystem.AddExtraPanel<T>` and `EL.m_NotificationUISystem.AddOrUpdateNotification` rather than recreating menu scaffolding.
- **Entity edit queue** - queue prefab or entity mutations with `EL.AddOnEditEnities` so ExtraLib handles progress notifications and coroutine timing.

Document any new conventions (icon folder names, embedded resource layout) in the relevant module `Agents.md` so future work stays consistent.

## Dependency Hygiene for Micro-Mods
- Keep third-party DLLs outside project directories (`dependencies/<Vendor>/<Version>/`) and never commit them under `bin/` or `obj/`.
- Version dependencies explicitly; avoid floating to "latest" without testing the entire module stack.
- Run `dotnet clean` and rebuild after upgrades to clear stale metadata.
- Enforce dependency declarations via CI by linting `PublishConfiguration.xml`, `mod.json`, and README badges.
- When replacing a shared library, stage the change: add the new dependency, migrate modules, then remove the old reference once every consumer is updated.

## QA and Troubleshooting
- **Missing dependency warning in-game** - confirm `PublishConfiguration.xml` and Paradox Mods metadata list ExtraLib. The launcher only auto-installs declared dependencies.
- **`FileNotFoundException: ExtraLib` during load** - check the `<HintPath>` and ensure the DLL is present. Relative paths that cross drive letters often break on CI runners.
- **Runtime API changes** - ExtraLib follows semantic versioning. When upgrading (for example `1.4.x` to `1.5.x`), re-run smoke tests for every module that uses its APIs.
- **Icon placeholders** - confirm the ExtraLib release contains the new SVG and reference it with `Icons.COUIBaseLocation`. Cached bundles sometimes require a game restart after icon updates.
- **Performance** - icons are vector files; keep them lightweight (< 50 KB). Run `svgo` or similar optimisers in CI.

## References
- ExtraLib repository: <https://github.com/AlphaGaming7780/ExtraLib>
- Set-up steps for shared libraries: [Architecture Overview](../architecture/overview.md)
- Icon usage patterns: [Unified Icon Library](unified-icon-library.md)

Follow these practices when introducing or updating shared dependencies so both humans and automation can reason about the Vice & Order stack.


