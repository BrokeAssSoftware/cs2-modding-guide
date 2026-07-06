---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Reference: SDK, MSBuild props/targets, and mod assemblies"
Summary: "Lookup of the CS2 modding build surface a code mod depends on - the Mod.props/Mod.targets import via CSII_TOOLPATH, the Colossal/Game/Unity assemblies referenced from real csproj files, and the IMod OnLoad/OnDispose entry points, each cited to a mod at a pinned commit."
diataxis: reference
source_version: "1.6.0f1 (realistic-path-finding@50645fa; date-pinned, static source only)"
last_reverified: "2026-07-04"
status: source-verified
canonical_mods:
  - realistic-path-finding@50645fa6a078181365e36a42e2e27b96699bf02a
  - magic-mail@6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47
  - outside-traffic-adjuster@42afd29638267c9dc116b040515ceaf41eb9a9e1
technique_applicability: [core]
Created: 2026-07-04
Updated: 2026-07-04
Owners:
  - codex
References:
  - Label: Official CS2 Modding Toolchain wiki (authoritative)
    Path: https://cs2.paradoxwikis.com/Modding_Toolchain
  - Label: "Explanation: Module layout"
    Path: ../explanation/module-layout.md
  - Label: "Explanation: Mod lifecycle"
    Path: ../explanation/mod-lifecycle.md
  - Label: "Tutorial: Getting started"
    Path: ../tutorials/getting-started.md
---

# Reference: SDK, MSBuild props/targets, and mod assemblies

Dry lookup of the build/SDK surface a CS2 **code** mod depends on: how the csproj pulls in
the SDK, which engine assemblies it references, and the `IMod` entry points the game calls.
Every row cites real mod source at a pinned commit. For the authoritative, install-side
toolchain details (installing the SDK, `CSII_TOOLPATH` setup, publishing), link out to the
official [CS2 Modding Toolchain wiki](https://cs2.paradoxwikis.com/Modding_Toolchain) rather
than trusting a duplicate here.

Corpus and pinned commits used for verification:

- `realistic-path-finding` @`50645fa` - `repo/RealisticPathFinding/RealisticPathFinding.csproj`,
  `repo/RealisticPathFinding/Mod.cs` (the fullest assembly-reference exhibit)
- `magic-mail` @`6fb3d2b` - `repo/MagicMail.csproj`
- `outside-traffic-adjuster` @`42afd29` - `repo/OutsideTrafficAdjuster.csproj`

Re-verify any cite with (Git-Bash paths):

```
git -C ../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo \
    show 50645fa6a078181365e36a42e2e27b96699bf02a:RealisticPathFinding/RealisticPathFinding.csproj
```

## 1. SDK import mechanism (`Mod.props` / `Mod.targets` via `CSII_TOOLPATH`)

A CS2 code mod is a plain `Microsoft.NET.Sdk` project that imports two SDK-supplied MSBuild
files from the toolchain install directory. The install path is read at evaluation time from
the **user** environment variable `CSII_TOOLPATH`; the two `Import` elements MUST come after
the `PropertyGroup` block.

| Element | Purpose | Cite (verified in all three csproj) |
| --- | --- | --- |
| `<Project Sdk="Microsoft.NET.Sdk">` | Standard .NET SDK-style project. | `realistic-path-finding/repo/RealisticPathFinding/RealisticPathFinding.csproj#L1`; `magic-mail/repo/MagicMail.csproj#L2`; `outside-traffic-adjuster/repo/OutsideTrafficAdjuster.csproj#L1` |
| `<Import ...CSII_TOOLPATH...\Mod.props />` | Pulls in SDK properties (assembly search paths, `$(ModPropsFile)`/`$(ModTargetsFile)`, output/deploy locations). | `realistic-path-finding/.../RealisticPathFinding.csproj#L12`; `magic-mail/repo/MagicMail.csproj#L31`; `outside-traffic-adjuster/repo/OutsideTrafficAdjuster.csproj#L13` |
| `<Import ...CSII_TOOLPATH...\Mod.targets />` | Pulls in SDK build/deploy targets (compile against game assemblies, copy the built mod into the game's mods tree). | `realistic-path-finding/.../RealisticPathFinding.csproj#L13`; `magic-mail/repo/MagicMail.csproj#L32`; `outside-traffic-adjuster/repo/OutsideTrafficAdjuster.csproj#L14` |
| `<None Include="$(ModPropsFile)" .../>` and `<None Include="$(ModTargetsFile)" .../>` | Surfaces the imported `Mod.props`/`Mod.targets` in the project tree (as `Properties\Mod.props` / `Properties\Mod.targets`) for reference. | `realistic-path-finding/.../RealisticPathFinding.csproj#L79-L80`; `magic-mail/repo/MagicMail.csproj#L80-L81`; `outside-traffic-adjuster/repo/OutsideTrafficAdjuster.csproj#L71-L72` |
| `<PublishConfigurationPath>Properties\PublishConfiguration.xml</PublishConfigurationPath>` | Points the SDK's publish tooling at the mod's PDX Mods metadata file. | `realistic-path-finding/.../RealisticPathFinding.csproj#L7`; `magic-mail/repo/MagicMail.csproj#L8`; `outside-traffic-adjuster/repo/OutsideTrafficAdjuster.csproj#L9` |

The exact literal import expression (identical across the corpus):

```xml
<!--Imports must be after PropertyGroup block-->
<Import Project="$([System.Environment]::GetEnvironmentVariable('CSII_TOOLPATH', 'EnvironmentVariableTarget.User'))\Mod.props" />
<Import Project="$([System.Environment]::GetEnvironmentVariable('CSII_TOOLPATH', 'EnvironmentVariableTarget.User'))\Mod.targets" />
```

`realistic-path-finding/repo/RealisticPathFinding/RealisticPathFinding.csproj#L11-L13`.

Notes on properties seen in source:

- `<AllowUnsafeBlocks>true</AllowUnsafeBlocks>` when the mod uses pointer/ref code -
  `realistic-path-finding/.../RealisticPathFinding.csproj#L8`.
- `TargetFramework` may be set explicitly (`<TargetFramework>net472</TargetFramework>`,
  `outside-traffic-adjuster/repo/OutsideTrafficAdjuster.csproj#L5`) or left to the SDK default
  (RPF and magic-mail do not set it in their csproj).
- Harmony is added as an ordinary NuGet `PackageReference` where used:
  `<PackageReference Include="Lib.Harmony" Version="2.2.2" />` -
  `realistic-path-finding/.../RealisticPathFinding.csproj#L83-L84` (magic-mail and
  outside-traffic-adjuster do not declare a Harmony `PackageReference` in their csproj).
- **`CSII_USERMODSPATH`**: not present in any of these three csproj files. It is defined inside
  the SDK's `Mod.props`/`Mod.targets` (which live under `CSII_TOOLPATH`, outside the mod repos),
  so it is `Needs Verification (in-game)` from this corpus - confirm against the
  [Modding Toolchain wiki](https://cs2.paradoxwikis.com/Modding_Toolchain).

## 2. Engine assembly references

Each `<Reference Include="...">` in an `<ItemGroup>` names a game/engine assembly resolved by
the SDK (via `Mod.props` search paths) and marked `<Private>false</Private>` so it is NOT
copied into the mod's output (the game already ships it). The table lists every assembly
referenced in the corpus, what it provides, and a csproj that references it.

| Assembly | Provides (namespace roots used by mods) | Cite |
| --- | --- | --- |
| `Game` | The base game: `Game.*` systems, prefabs, components, `Game.Modding.IMod`, `Game.SceneFlow`, `Game.Simulation`, `Game.Economy`, etc. | `realistic-path-finding/.../RealisticPathFinding.csproj#L25`; `magic-mail/repo/MagicMail.csproj#L38`; `outside-traffic-adjuster/repo/OutsideTrafficAdjuster.csproj#L17` |
| `Colossal.Core` | Core engine utilities (`Colossal.*` base types, environment/paths). | `realistic-path-finding/.../RealisticPathFinding.csproj#L28`; `magic-mail/repo/MagicMail.csproj#L41`; `outside-traffic-adjuster/repo/OutsideTrafficAdjuster.csproj#L20` |
| `Colossal.Logging` | Logging (`ILog`, `LogManager`). | `realistic-path-finding/.../RealisticPathFinding.csproj#L31`; `magic-mail/repo/MagicMail.csproj#L44`; `outside-traffic-adjuster/repo/OutsideTrafficAdjuster.csproj#L23` |
| `Colossal.IO.AssetDatabase` | Asset/settings persistence (`AssetDatabase`, `LoadSettings`). | `realistic-path-finding/.../RealisticPathFinding.csproj#L34`; `magic-mail/repo/MagicMail.csproj#L47`; `outside-traffic-adjuster/repo/OutsideTrafficAdjuster.csproj#L26` |
| `Colossal.Localization` | Localization sources (`IDictionarySource`, locale registration). | `realistic-path-finding/.../RealisticPathFinding.csproj#L43`; `magic-mail/repo/MagicMail.csproj#L35`; `outside-traffic-adjuster/repo/OutsideTrafficAdjuster.csproj#L35` |
| `Colossal.Collections` | Native/engine collection types. | `realistic-path-finding/.../RealisticPathFinding.csproj#L16` |
| `Colossal.Mathematics` | Engine math helpers beyond `Unity.Mathematics`. | `realistic-path-finding/.../RealisticPathFinding.csproj#L19` |
| `Colossal.PSI.Common` | Platform Services Integration common layer (platform/user-data services). | `realistic-path-finding/.../RealisticPathFinding.csproj#L22` |
| `Colossal.UI` | UI host (Gameface/CEF integration surface). | `realistic-path-finding/.../RealisticPathFinding.csproj#L37`; `outside-traffic-adjuster/repo/OutsideTrafficAdjuster.csproj#L29` |
| `Colossal.UI.Binding` | C#<->UI data binding (`IBinding`, value/trigger bindings). | `realistic-path-finding/.../RealisticPathFinding.csproj#L40`; `outside-traffic-adjuster/repo/OutsideTrafficAdjuster.csproj#L32` |
| `Unity.Entities` | Unity DOTS ECS (`World`, `SystemBase`, `EntityManager`, components/buffers). | `realistic-path-finding/.../RealisticPathFinding.csproj#L58`; `magic-mail/repo/MagicMail.csproj#L59`; `outside-traffic-adjuster/repo/OutsideTrafficAdjuster.csproj#L50` |
| `Unity.Collections` | Native containers (`NativeArray`, `NativeList`, etc.). | `realistic-path-finding/.../RealisticPathFinding.csproj#L55`; `magic-mail/repo/MagicMail.csproj#L56`; `outside-traffic-adjuster/repo/OutsideTrafficAdjuster.csproj#L47` |
| `Unity.Mathematics` | `float3`, `int2`, math functions used in ECS jobs. | `realistic-path-finding/.../RealisticPathFinding.csproj#L61`; `magic-mail/repo/MagicMail.csproj#L62`; `outside-traffic-adjuster/repo/OutsideTrafficAdjuster.csproj#L53` |
| `Unity.Burst` | Burst compiler attributes/types for jobs. | `realistic-path-finding/.../RealisticPathFinding.csproj#L52`; `magic-mail/repo/MagicMail.csproj#L53`; `outside-traffic-adjuster/repo/OutsideTrafficAdjuster.csproj#L44` |
| `Unity.InputSystem` | Input actions/bindings (keyboard/mouse rebinds). | `realistic-path-finding/.../RealisticPathFinding.csproj#L46`; `outside-traffic-adjuster/repo/OutsideTrafficAdjuster.csproj#L38` |
| `UnityEngine.CoreModule` | Core UnityEngine types (`Debug`, `Vector*`, `MonoBehaviour` surface). | `realistic-path-finding/.../RealisticPathFinding.csproj#L49`; `magic-mail/repo/MagicMail.csproj#L50`; `outside-traffic-adjuster/repo/OutsideTrafficAdjuster.csproj#L41` |

BCL references are pinned as `<Reference Update="...">` with `<Private>false</Private>`
(`System`, `System.Core`, `System.Data`) so they are not privately copied:
`realistic-path-finding/.../RealisticPathFinding.csproj#L66-L76`;
`magic-mail/repo/MagicMail.csproj#L67-L77`;
`outside-traffic-adjuster/repo/OutsideTrafficAdjuster.csproj#L58-L68`.

**Reference set is a subset per mod.** `realistic-path-finding` references the widest surface
(all 16 assemblies above); `magic-mail` and `outside-traffic-adjuster` reference only what they
use. Add a `<Reference>` only when the mod actually consumes that assembly's types.

## 3. The `IMod` entry surface

The game discovers a mod through a single class implementing `Game.Modding.IMod`. The two
lifecycle methods below are the entire required surface; the game calls `OnLoad` at load and
`OnDispose` at unload.

| Member | Signature | Purpose | Cite |
| --- | --- | --- | --- |
| `IMod` (implemented by `public class Mod : IMod`) | - | Mod entry class; `IMod` comes from `Game.Modding`. | `realistic-path-finding/repo/RealisticPathFinding/Mod.cs#L5,L16` |
| `OnLoad` | `public void OnLoad(UpdateSystem updateSystem)` | Called on mod load: create/load settings, register locale source, register/order ECS systems on `updateSystem`, apply Harmony patches. | `realistic-path-finding/repo/RealisticPathFinding/Mod.cs#L24` |
| `OnDispose` | `public void OnDispose()` | Called on unload: tear down (e.g. unregister settings from the Options UI). | `realistic-path-finding/repo/RealisticPathFinding/Mod.cs#L94` |

Minimal shape as seen in source:

```csharp
public class Mod : IMod
{
    public void OnLoad(UpdateSystem updateSystem)
    {
        log.Info(nameof(OnLoad));
        m_Setting = new Setting(this);
        AssetDatabase.global.LoadSettings(nameof(RealisticPathFinding), m_Setting, new Setting(this));
        m_Setting.RegisterInOptionsUI();
        GameManager.instance.localizationManager.AddSource("en-US", new LocaleEN(m_Setting));
        // ... register systems on updateSystem, apply Harmony patches ...
    }

    public void OnDispose()
    {
        log.Info(nameof(OnDispose));
        if (m_Setting != null)
        {
            m_Setting.UnregisterInOptionsUI();
            m_Setting = null;
        }
    }
}
```

`realistic-path-finding/repo/RealisticPathFinding/Mod.cs#L16-L102` (elided). The `using` for
`IMod`/`UpdateSystem` is `Game.Modding` (`#L5`); `UpdateSystem` and system registration/ordering
(`UpdateAt`/`UpdateAfter`/`UpdateBefore`) are covered under
[mod-lifecycle.md](../explanation/mod-lifecycle.md).

## SDK / tooling version

- **SDK version numbers are not stated in mod source** (the version lives in the toolchain
  install under `CSII_TOOLPATH`, not in the repos). Treat any specific SDK version as
  `Needs Verification (in-game)` unless read from the install.
- The current SDK at time of writing is **2.1.2** (per the official
  [Modding Toolchain wiki](https://cs2.paradoxwikis.com/Modding_Toolchain)) - cited to the wiki,
  not asserted from these repos.
- `magic-mail` declares game-version/mod-version metadata in its csproj as build properties
  (`<GameVersion>1.6.*</GameVersion>`, `<Version>1.1.5</Version>`) that it syncs into
  `PublishConfiguration.xml` via a PowerShell target -
  `magic-mail/repo/MagicMail.csproj#L15-L16,L84-L90`. This is a per-mod convention, not an SDK
  requirement.

## Needs Verification (in-game / install-side)

- `CSII_USERMODSPATH` and the internal contents of `Mod.props`/`Mod.targets` (assembly search
  roots, deploy target) - defined under `CSII_TOOLPATH`, outside these repos.
- The concrete SDK version (2.1.2 is per the wiki, not source).
- The full authoritative assembly list the SDK exposes - the table above is the subset the
  corpus references, not the complete set. See the official wiki.

## See also

- [explanation/module-layout.md](../explanation/module-layout.md) - how a mod's assemblies and
  files are organized.
- [explanation/mod-lifecycle.md](../explanation/mod-lifecycle.md) - what `OnLoad`/`OnDispose`
  and system registration do at runtime.
- [tutorials/getting-started.md](../tutorials/getting-started.md) - first-mod walkthrough,
  including toolchain install.
- Official [CS2 Modding Toolchain wiki](https://cs2.paradoxwikis.com/Modding_Toolchain) -
  authoritative install, `CSII_TOOLPATH` setup, SDK version, and publishing steps.
