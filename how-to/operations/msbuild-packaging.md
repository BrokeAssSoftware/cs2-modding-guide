---
FrontmatterVersion: 1
DocumentType: Guide
Title: "MSBuild build & packaging automation"
Summary: How CS2 mods fold the React/TSX UI build, publish-metadata injection, single-source version sync, and a compile-time Burst/managed toggle into a plain `dotnet build`, so one command produces a publishable mod - shown from real, pinned mod csproj files.
diataxis: how-to
source_version: "~1.5.x-1.6.0f1 (multiple mods; date-pinned)"
last_reverified: "2026-07-05"
status: source-verified
canonical_mods:
  - traffic-tool-essentials@10973595ac9ed37f47ca24eb788d4421dc295fa0
  - anarchy@a6311e898d20a775368668b234aaa32f06e3e1eb
  - unified-icon-library@b200d8901346457f97c032e86e1d8437a12c0cc8
  - magic-garbage-truck@1b6a478753e1ef4e43ac9b90d567f3d7183c7be2
  - abandoned-building-remover-deviance-fix@a515bfe588965cbc988cba6a974242f66e5386b7
  - i18n-everywhere@d9285c2490079c6d67303da536207b47b106ce64
technique_applicability: [core, content]
Created: 2026-07-05
Updated: 2026-07-05
Owners:
  - codex
---

# MSBuild build & packaging automation

> Make one `dotnet build` produce a complete, publishable CS2 mod: compile the C#,
> build the React/TSX UI bundle, inject the publish metadata, sync the version into
> the publish XML, and (optionally) flip a Burst/managed code path - all wired into
> MSBuild targets so nothing is a manual side-step.

## Problem
A CS2 mod is more than a DLL. A UI mod also has a React/TSX bundle that must be built
with npm; every mod has a `PublishConfiguration.xml` whose `LongDescription`,
`ChangeLog`, and `ModVersion` have to match the code and changelog you are shipping;
and some mods carry a Burst-compiled hot path that must degrade to a managed path when
Burst is off. If any of these is a separate manual step, it drifts: you ship a stale UI
bundle, a description that does not match the build, a publish XML version that lags the
assembly, or a Burst path that no longer matches its managed twin. This page shows how
real mods pull all of it into `dotnet build` through MSBuild `Target`s and properties.

## Solution
Attach each concern to a build phase with a `<Target>` and the right
`BeforeTargets`/`AfterTargets` hook, and drive the mutable metadata from a single
property (`$(Version)`) so hand-edited copies cannot skew:

- **UI bundle** -> a target that runs `npm run build` before/after the C# compile, with
  an opt-out flag so Node-less CI agents can skip it.
- **Publish metadata** -> a `BeforeBuild` target that uses the `XmlPoke` task to poke
  `LongDescription.md`, the changelog, and `$(Version)` into `PublishConfiguration.xml`.
- **Version** -> one `<Version>` element; a target rewrites the publish XML from it.
- **Burst** -> one `<BurstCompile>` property that conditionally adds `USE_BURST` to
  `DefineConstants`, selecting a `#if USE_BURST` code path at compile time.

Because MSBuild runs these every build, the packaged output is always consistent with
source. The only discipline left is keeping the two Burst code branches in sync (see
Pitfalls).

## Steps & Code

### 1. Build the UI bundle inside `dotnet build`

Add a target that runs `npm run build` in your frontend folder and hook it to a build
phase. Traffic Tool Essentials runs it **before** `CoreCompile`, only installs
`node_modules` when it is missing, copies the built `.mjs` into `$(OutDir)`, and - key
for CI - guards the whole target behind a `DisableBuildFrontend` flag that defaults to
`false` but can be overridden on a build agent with no Node:

```xml
<!-- Escape hatch: defaults false; set DisableBuildFrontend=true on Node-less agents -->
<DisableBuildFrontend Condition="'$(DisableBuildFrontend)' == ''">false</DisableBuildFrontend>
<CleanFrontendCache   Condition="'$(CleanFrontendCache)' == ''">false</CleanFrontendCache>
```
Source: `../../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/TrafficToolEssentials.csproj#L17-L18` (@10973595ac9ed37f47ca24eb788d4421dc295fa0)

```xml
<Target Name="BuildFrontend" BeforeTargets="CoreCompile" Condition="'$(DisableBuildFrontend)' != 'true'">
  <RemoveDir Directories="..\TLEFrontend\node_modules\.cache" Condition="'$(CleanFrontendCache)' == 'true'" />
  <Exec Command="npm install --prefer-offline --no-audit" WorkingDirectory="..\TLEFrontend\"
        Condition="!Exists('..\TLEFrontend\node_modules')" ContinueOnError="false" />
  <Exec Command="npm run build" WorkingDirectory="..\TLEFrontend\" ContinueOnError="false" />
  <Copy SourceFiles="..\TLEFrontend\dist\C2VM.TrafficToolEssentials.mjs" DestinationFolder="$(OutDir)\" />
</Target>
```
Source: `../../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/TrafficToolEssentials.csproj#L121-L145` (@10973595ac9ed37f47ca24eb788d4421dc295fa0)

The minimal shape of the same idea is Anarchy's `InstallUI` target, which just runs the
build **after** `AfterBuild` in its `UI/` folder - note it has **no** opt-out condition
(a deliberate contrast; see Pitfalls):

```xml
<Target Name="InstallUI" AfterTargets="AfterBuild">
  <Exec Command="npm run build" WorkingDirectory="$(ProjectDir)/UI" />
</Target>
```
Source: `../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/Anarchy.csproj#L201-L203` (@a6311e898d20a775368668b234aaa32f06e3e1eb)

### 2. Inject publish metadata with `XmlPoke`

Do not hand-edit `PublishConfiguration.xml`. A `BeforeBuild` target reads your Markdown
files and pokes them into the XML nodes, and pushes `$(Version)` into the `ModVersion`
attribute. Unified Icon Library's `SetDescription` is the canonical shape:

```xml
<Target Name="SetDescription" BeforeTargets="BeforeBuild">
  <XmlPoke XmlInputPath="$(PublishConfigurationPath)" Value="$([System.IO.File]::ReadAllText($(ProjectDir)/Properties/LongDescription.md))" Query="//LongDescription" />
  <XmlPoke XmlInputPath="$(PublishConfigurationPath)" Value="$([System.IO.File]::ReadAllText($(ProjectDir)/Properties/LatestChangelog.md))" Query="//ChangeLog" />
  <XmlPoke XmlInputPath="$(PublishConfigurationPath)" Value="$(Version)" Query="//ModVersion/@Value" />
</Target>
```
Source: `../../../vice-and-order-research/mods/dossiers/unified-icon-library/repo/UnifiedIconLibrary.csproj#L56-L60` (@b200d8901346457f97c032e86e1d8437a12c0cc8)

The same csproj mirrors non-compiled assets into the deploy folder (`CopyIcons`, running
after `AfterBuild` over glob ItemGroups)...

```xml
<Target Name="CopyIcons" AfterTargets="AfterBuild">
  <Copy SourceFiles="@(_Standard)" DestinationFolder="$(DeployDir)/Icons/Standard" />
  <Copy SourceFiles="@(_Dark)"     DestinationFolder="$(DeployDir)/Icons/Dark" />
  <Copy SourceFiles="@(_Colored)"  DestinationFolder="$(DeployDir)/Icons/Colored" />
</Target>
```
Source: `../../../vice-and-order-research/mods/dossiers/unified-icon-library/repo/UnifiedIconLibrary.csproj#L45-L54` (@b200d8901346457f97c032e86e1d8437a12c0cc8)

...and trims files that must not ship (the raw `.xml` template and `.pdb`s) from the
deploy folder with a `Cleanup` target:

```xml
<Target Name="Cleanup" AfterTargets="AfterBuild">
  <ItemGroup>
    <CleanTargets Include="$(DeployDir)/$(ProjectName).xml" />
    <CleanTargets Include="$(DeployDir)/*.pdb" />
  </ItemGroup>
  <Delete Files="@(CleanTargets)" />
</Target>
```
Source: `../../../vice-and-order-research/mods/dossiers/unified-icon-library/repo/UnifiedIconLibrary.csproj#L62-L68` (@b200d8901346457f97c032e86e1d8437a12c0cc8)

### 3. Drive the publish XML from a single `<Version>`

Declare the version once and let a target rewrite the publish XML from it, so
`AssemblyVersion`, `FileVersion`, and `PublishConfiguration.xml` can never disagree.
Magic Garbage Truck derives everything from one `<Version>`:

```xml
<!-- Single source of truth for version number -->
<GameVersion>1.6.*</GameVersion>
<Version>1.3.1</Version>
<AssemblyVersion>$(Version).0</AssemblyVersion>
<FileVersion>$(Version).0</FileVersion>
```
Source: `../../../vice-and-order-research/mods/dossiers/magic-garbage-truck/repo/MagicGarbage.csproj#L19-L23` (@1b6a478753e1ef4e43ac9b90d567f3d7183c7be2)

A `BeforeTargets="PrepareForBuild"` target then runs a helper script that syncs
`ModVersion` and `GameVersion` (and a nested `mod.json`) from `$(Version)`/`$(GameVersion)`.
The target is `Condition`-guarded so it silently no-ops if the script or XML is absent:

```xml
<Target Name="UpdatePublishConfigVersion" BeforeTargets="PrepareForBuild"
        Condition="Exists('$(PublishConfigScript)') AND Exists('$(MSBuildProjectDirectory)\$(PublishConfigurationPath)')">
  <Exec Command="powershell.exe -NoProfile -ExecutionPolicy Bypass -File &quot;$(PublishConfigScript)&quot;
        -Path &quot;$(MSBuildProjectDirectory)\$(PublishConfigurationPath)&quot;
        -Version &quot;$(Version)&quot; -GameVersion &quot;$(GameVersion)&quot; -Eol lf -LeftAlignBlocks" />
</Target>
```
Source: `../../../vice-and-order-research/mods/dossiers/magic-garbage-truck/repo/MagicGarbage.csproj#L108-L113` (@1b6a478753e1ef4e43ac9b90d567f3d7183c7be2)
(`$(PublishConfigScript)` is defined as `...\Scripts\Update-PublishConfig.ps1`,
`MagicGarbage.csproj#L14`.)

Note the difference in mechanism from step 2: Unified Icon Library pokes the XML nodes
directly with the built-in `XmlPoke` task; Magic Garbage Truck delegates to an external
PowerShell script (which also normalizes EOL/BOM). Both start from a single `<Version>`.

### 4. Toggle Burst vs managed from one MSBuild property

Compile-time branching lets you ship a Burst-jobified hot path while keeping a managed
fallback. Abandoned Building Remover turns one `<BurstCompile>` property into a
`USE_BURST` preprocessor symbol via a conditional `DefineConstants`:

```xml
<BurstCompile>true</BurstCompile>
...
<PropertyGroup Condition="$(BurstCompile)">
  <DefineConstants>$(DefineConstants);USE_BURST</DefineConstants>
</PropertyGroup>
```
Source: `../../../vice-and-order-research/mods/dossiers/abandoned-building-remover-deviance-fix/repo/AbandonedBuildingRemover.csproj#L8` and `#L24-L26` (@a515bfe588965cbc988cba6a974242f66e5386b7)

The system then guards its Burst-only `using`s and picks a code path with
`#if USE_BURST` / `#if !USE_BURST`. `OnUpdate` schedules an `IJob` when Burst is on, and
runs a plain `EntityManager` loop when it is off - two implementations of the same work:

```csharp
protected override void OnUpdate()
{
#if USE_BURST
    AbandonedBuildingRemoverJob job = default;
    job.m_entityTypeHandle = SystemAPI.GetEntityTypeHandle();
    job.m_entityCommandBuffer = _endFrameBarrier.CreateCommandBuffer();
    job.m_abandonedBuildingsChunk = _abandonedBuildingQuery.ToArchetypeChunkListAsync(World.UpdateAllocator.ToAllocator, out _);
    // ... buffer lookups ...
    JobHandle handle = job.Schedule(Dependency);
    _endFrameBarrier.AddJobHandleForProducer(handle);
    Dependency = handle;
#endif
#if !USE_BURST
    var abandonedBuildings = _abandonedBuildingQuery.ToEntityArray(Allocator.Temp);
    foreach (var entity in abandonedBuildings)
    {
        if (EntityManager.TryGetBuffer<SubArea>(entity, false, out var subareas))
            foreach (var subArea in subareas) EntityManager.AddComponent<Deleted>(subArea.m_Area);
        // ... SubNet, SubLane ...
        EntityManager.AddComponent<Deleted>(entity);
    }
#endif
}
```
Source: `../../../vice-and-order-research/mods/dossiers/abandoned-building-remover-deviance-fix/repo/AbandonedBuildingRemoverSystem.cs#L54-L104` (@a515bfe588965cbc988cba6a974242f66e5386b7)

The `[BurstCompile]` job struct that the Burst path schedules is itself wrapped in
`#if USE_BURST`, so it does not even compile when Burst is disabled
(`AbandonedBuildingRemoverSystem.cs#L109-L158`); the Burst-only `using Unity.Jobs; using
Unity.Burst;` block is guarded the same way (`#L11-L14`). Even the log line differs per
path - "Using Burst" vs "NOT using Burst"
(`AbandonedBuildingRemoverSystem.cs#L42-L47`), which is a cheap runtime confirmation of
which branch compiled.

## Pitfalls & gotchas

- **An unconditional npm target breaks Node-less CI.** Traffic Tool Essentials guards
  `BuildFrontend` with `Condition="'$(DisableBuildFrontend)' != 'true'"`
  (`TrafficToolEssentials.csproj#L121`), so a build agent without Node can pass
  `-p:DisableBuildFrontend=true`. Anarchy's `InstallUI` has **no** such guard
  (`Anarchy/Anarchy.csproj#L201-L203`): a plain `dotnet build` on an agent without npm
  fails at that `<Exec>`. If your CI compiles C# without building the UI, you need the
  escape-hatch flag, not the bare target. See [Automate Build Validation](./ci-automation.md).

- **`XmlPoke` rewrites the file in place.** `SetDescription` and `UpdatePublishConfigVersion`
  mutate the committed `PublishConfiguration.xml` on every build
  (`UnifiedIconLibrary.csproj#L56-L60`). That is the point (metadata always matches
  source), but it also means the file shows up as dirty in git after a build even when
  you changed nothing meaningful - expect churn, and treat the Markdown/`<Version>`
  sources as authoritative, not the generated XML.

- **The version-sync target can silently skip.** Magic Garbage Truck's target only runs
  when both the script and the XML exist (`MagicGarbage.csproj#L108`). If a checkout is
  missing `Scripts\Update-PublishConfig.ps1`, the build still succeeds but the publish
  XML is never synced - a silent no-op, not an error. Verify the sync actually happened
  before publishing.

- **The two Burst branches must be maintained together.** The whole `USE_BURST` scheme
  compiles only one branch, so a change to the managed loop that is not mirrored into the
  Burst `IJob` (or vice-versa) ships a divergence that is invisible in whichever build you
  did not run. The managed and Burst paths in Abandoned Building Remover do the same
  deletions two different ways (`AbandonedBuildingRemoverSystem.cs#L54-L104` vs
  `#L109-L158`); keeping them in lockstep is a manual discipline MSBuild cannot enforce.

- **Hook ordering matters.** `SetDescription` runs at `BeforeBuild` so the XML is correct
  before packaging; `CopyIcons`/`Cleanup` run at `AfterBuild` so the deploy folder exists
  first; the frontend build runs before `CoreCompile` (Traffic Tool Essentials) or after
  `AfterBuild` (Anarchy). Pick the phase by what must already exist when the target runs;
  a target hung on the wrong phase either copies nothing or races the build.

- Anything not shown in the cited csproj/source is `Needs Verification (in-game)`. In
  particular, whether a given `BeforeTargets`/`AfterTargets` choice interacts cleanly with
  the CS2 `Mod.props`/`Mod.targets` import (each project `Import`s the SDK targets) is not
  proven from these files alone and should be confirmed against your SDK version.

## Variations

- **Auto-increment a build counter and copy loose asset folders.** I18N Everywhere keeps
  a `build.counter` file (`I18NEverywhere.csproj#L4`) and a target that reads, increments,
  and writes it back so each build gets a unique fourth version part:

  ```xml
  <Target Name="IncrementBuildCount">
    <!-- read + parse current count, compute BuildCount -->
    <ItemGroup>
      <CounterContent Include="$(BaseVersion).$(BuildCount)" />
    </ItemGroup>
    <WriteLinesToFile File="$(BuildCounterFile)" Lines="@(CounterContent)" Overwrite="true" />
  </Target>
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/i18n-everywhere/repo/I18NEverywhere/I18NEverywhere.csproj#L186-L212` (@d9285c2490079c6d67303da536207b47b106ce64)

  It also copies its `lang/**` translation tree into the output via a `Content` item with
  `PreserveNewest`, so localization files ship without a manual copy step:

  ```xml
  <Content Include="lang\**">
    <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
  </Content>
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/i18n-everywhere/repo/I18NEverywhere/I18NEverywhere.csproj#L237-L239` (@d9285c2490079c6d67303da536207b47b106ce64)

- **Inline the XmlPoke instead of a separate metadata target.** Anarchy folds the same
  `LongDescription`/`ChangeLog`/`ModVersion` pokes into a `SetupAttributes` target gated on
  a non-Debug configuration (`Condition="'$(Configuration)' != 'Debug'"`), so metadata is
  only rewritten for Release builds:

  ```xml
  <Target Name="SetupAttributes" BeforeTargets="BeforeBuild" Condition="'$(Configuration)' != 'Debug'">
    <XmlPoke XmlInputPath="$(PublishConfigurationPath)" Value="$([System.IO.File]::ReadAllText(Properties/$(Configuration)/LongDescription.md))" Query="//LongDescription" />
    <XmlPoke XmlInputPath="$(PublishConfigurationPath)" Value="$(Version)" Query="//ModVersion/@Value" />
  </Target>
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/Anarchy.csproj#L205-L208` (@a6311e898d20a775368668b234aaa32f06e3e1eb)

  Gating on configuration keeps the committed publish XML from being rewritten during
  everyday Debug builds - a middle ground between "always poke" (Unified Icon Library) and
  hand editing.

## See also
- Tutorials: [Build your first UI mod](../../tutorials/first-ui-mod.md) (where the
  React/TSX bundle comes from), [Build, Test, and Publish](../../tutorials/build-and-publish.md)
  (the commands these targets wrap).
- Operations: [Automate Build Validation](./ci-automation.md) (running these builds on CI,
  and the `DisableBuildFrontend` escape hatch), [Release Checklist](./release-checklist.md)
  (the manual gate the version/metadata sync supports).

## Sources
- Canonical mods (dossier + repo):
  - `traffic-tool-essentials` @10973595ac9ed37f47ca24eb788d4421dc295fa0 - `repo/TrafficToolEssentials/TrafficToolEssentials.csproj`
  - `anarchy` @a6311e898d20a775368668b234aaa32f06e3e1eb - `repo/Anarchy/Anarchy.csproj`
  - `unified-icon-library` @b200d8901346457f97c032e86e1d8437a12c0cc8 - `repo/UnifiedIconLibrary.csproj`
  - `magic-garbage-truck` @1b6a478753e1ef4e43ac9b90d567f3d7183c7be2 - `repo/MagicGarbage.csproj`
  - `abandoned-building-remover-deviance-fix` @a515bfe588965cbc988cba6a974242f66e5386b7 - `repo/AbandonedBuildingRemover.csproj`, `repo/AbandonedBuildingRemoverSystem.cs`
  - `i18n-everywhere` @d9285c2490079c6d67303da536207b47b106ce64 - `repo/I18NEverywhere/I18NEverywhere.csproj`
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding ; MSBuild `XmlPoke`/`Target` docs:
  https://learn.microsoft.com/visualstudio/msbuild/xmlpoke-task
