---
FrontmatterVersion: 1
DocumentType: Guide
Title: Module Layout
Summary: How a CS2 code mod is named and organised - one identifier shared by assembly, namespace, and mod ID; a predictable folder layout; and the MSBuild toolchain import that resolves game DLLs without bundling them - explained against real mod source.
diataxis: explanation
source_version: "~1.5.10f1 (advanced-road-naming@559e72c; date-pinned, static source only)"
last_reverified: "2026-07-04"
canonical_mods:
  - advanced-road-naming@559e72cdb3180e3e71869643367e094a22eed988
  - road-speed-adjuster@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed
status: source-verified
Created: 2026-07-02
Updated: 2026-07-02
Owners:
  - codex
References:
  - Label: Lifecycle and Initialization (companion concept)
    Path: ./mod-lifecycle.md
  - Label: Dependency Strategy (companion concept)
    Path: ./dependency-strategy.md
  - Label: Technique Index
    Path: ../technique-index.md
---

# Module Layout

Before a Cities: Skylines II mod does anything interesting, it has to be *named* and
*organised*. Those two decisions look cosmetic but they are load-bearing: the game keys
several subsystems off your identifiers (the mod ID that publishing and dependencies use,
the assembly name reflection matches against, the logger channel your support logs are
filed under), and the toolchain keys its build off where your project sits relative to the
installed modding tools.

This page explains the *why* behind a predictable module layout - the naming convention,
the folder shape, and the MSBuild configuration that ties a project to the CS2 toolchain -
so that tooling, automation, and other modders can reason about your mod without special
cases. It is a concept page, not a step-by-step setup guide.

## One identifier, three roles

The single most useful convention is to use the **same identifier** for the assembly name,
the namespace root, and the published mod ID. Real mods do this with `nameof(...)` so the
three can never drift apart. Advanced Road Naming declares its namespace `AdvancedRoadNaming`,
its mod `Id` as `nameof(AdvancedRoadNaming)`, and namespaces its logger channel with the same
token
([`advanced-road-naming` `repo/Mod.cs#L14-L19`](../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Mod.cs), commit `559e72c`):

```csharp
namespace AdvancedRoadNaming
{
    public class Mod : IMod
    {
        public static readonly string Id = nameof(AdvancedRoadNaming);
        private static readonly ILog BaseLog =
            LogManager.GetLogger($"{nameof(AdvancedRoadNaming)}.{nameof(Mod)}")
                      .SetShowsErrorsInUI(false);
```

Road Speed Adjuster follows the identical shape - namespace `RoadSpeedAdjuster`, `Id = "RoadSpeedAdjuster"`,
and `LoadSettings(nameof(RoadSpeedAdjuster), ...)` - so the settings file, the assembly, and
the namespace all agree
([`road-speed-adjuster` `repo/Mod.cs#L10-L33`](../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/Mod.cs), commit `e0c0c0b`).

Why it matters:

- **The mod ID is the key other systems reference you by.** It appears in your
  `PublishConfiguration.xml` (`<ModId>`), is the string passed to
  `AssetDatabase.global.LoadSettings(id, ...)` so your `.coc` settings file is named after
  it, and is what a *dependent* mod names when it declares a dependency on you.
- **The assembly name is what reflection matches.** Mods that soft-detect each other scan
  `AppDomain.CurrentDomain.GetAssemblies()` by assembly name (see
  [Dependency Strategy](./dependency-strategy.md)); an unpredictable assembly name makes your
  mod undetectable.
- **The namespaced logger channel** (`"MyMod.Core.Mod"`) makes your lines greppable in the
  shared game log.

For a generic mod, pick one PascalCase identifier - say `MyMod.Core` - and use it in all
three places.

## A predictable folder shape

Type folders keep a mod navigable as it grows, and let shared automation find things by
convention. Advanced Road Naming's top level separates concerns into `Systems/`,
`Services/`, `Components/`, `Domain/`, `UI/`, `Settings/`, and `L10N/`, with `Properties/`
holding publish metadata
([`advanced-road-naming` tree at commit `559e72c`](../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/)).
A generic layout distilled from that:

```
MyMod.Core/
  Mod.cs                     # IMod entry point (OnLoad / OnDispose)
  Setting.cs                 # Setting subclass + Options UI attributes
  Systems/                   # DOTS/ECS systems (one per file)
  Services/                  # plain-C# helpers systems delegate to
  Components/                # IComponentData / buffer element types
  UI/                        # C# UI systems (paired with a JS/TS package)
  Localization/              # IDictionarySource locale classes / lang files
  Properties/
    PublishConfiguration.xml # ModId, DisplayName, dependencies, tags
```

Not every mod needs every folder - a background simulation mod may have no `UI/`, a tool mod
adds tool systems - but the *principle* is one type per file, grouped by role. The
`Mod.cs` at the root is the fixed point: it is the `IMod` implementation the game
instantiates (see [Lifecycle and Initialization](./mod-lifecycle.md)).

## The MSBuild toolchain import

A CS2 mod project does not vendor Unity, DOTS, or the game DLLs. Instead it imports the
installed modding toolchain via an environment variable and references game assemblies with
`<Private>false</Private>` so they are used at compile time but **not copied** into the
mod's output. Road Speed Adjuster's project shows the canonical form
([`road-speed-adjuster` `repo/RoadSpeedAdjuster.csproj#L10-L41`](../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/RoadSpeedAdjuster.csproj), commit `e0c0c0b`):

```xml
<!-- Imports must be after the PropertyGroup block -->
<Import Project="$([System.Environment]::GetEnvironmentVariable('CSII_TOOLPATH', 'EnvironmentVariableTarget.User'))\Mod.props" />
<Import Project="$([System.Environment]::GetEnvironmentVariable('CSII_TOOLPATH', 'EnvironmentVariableTarget.User'))\Mod.targets" />
...
<Reference Include="Game">
    <Private>false</Private>
</Reference>
<Reference Include="Colossal.Core">
    <Private>false</Private>
</Reference>
```

Two ideas are doing the work here:

- **`CSII_TOOLPATH` indirection.** The path to the installed toolchain is resolved from a
  per-user environment variable, not hard-coded. That is what lets the same project build on
  a different machine, a different game install (Steam vs. Xbox / PC Game Pass), or a CI
  runner without editing paths. `Mod.props` and `Mod.targets` (provided by the toolchain)
  inject the game's reference assemblies, output-to-mods-folder behaviour, and packaging
  targets.
- **`<Private>false</Private>` on game/engine references.** These DLLs ship with the game;
  bundling copies of them into your mod would bloat it and risk version mismatches. The flag
  means "compile against this, do not copy it." The same discipline applies to *shared
  library* dependencies - reference them, do not bundle them - which is the subject of
  [Dependency Strategy](./dependency-strategy.md).

If multiple related mods share settings, the common pattern is to lift shared references and
properties into a `Directory.Build.props` at the repository root so every project inherits
them, keeping optional libraries scoped to only the projects that use them. `Needs
Verification`: this multi-project `Directory.Build.props` arrangement is a general MSBuild
convention rather than something confirmed in a single-project dossier above; the toolchain
import and `<Private>false</Private>` discipline are the source-verified parts.

## Why predictability pays off

- **Discoverability by convention.** When systems live in `Systems/` and components in
  `Components/`, a reader (human or agent) can navigate an unfamiliar mod immediately.
- **Toolable identifiers.** One identifier across assembly/namespace/mod-ID means a script
  can derive the settings file name, the log channel, and the publish `ModId` from a single
  input.
- **Portable builds.** The `CSII_TOOLPATH` import and non-private references mean the project
  carries no machine-specific state, so it builds anywhere the toolchain is installed.

## See also

- [Lifecycle and Initialization](./mod-lifecycle.md) - what the `Mod.cs` entry point does
  once the game instantiates it.
- [Dependency Strategy](./dependency-strategy.md) - referencing shared libraries without
  bundling them, and detecting them at runtime.
- [Technique Index](../technique-index.md) - coverage ledger for these techniques.
