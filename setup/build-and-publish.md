# Build and Publish Workflow

Integrate the UI bundle into MSBuild, follow a daily development loop, and publish releases through the toolchain.

## Wire the UI Build into MSBuild
Add build targets to the code mod project so `dotnet build` also bundles the UI and refreshes the deployed copy:
```xml
<PropertyGroup>
  <UIBuildSourceDir>$(CSII_USERDATAPATH)\Mods\<ModuleId></UIBuildSourceDir>
</PropertyGroup>

<ItemGroup>
  <LegacyModOutput Include="$(CSII_USERDATAPATH)\Mods\<LegacyId>" />
</ItemGroup>

<Target Name="CleanLegacyModOutputs" BeforeTargets="BuildUI">
  <RemoveDir Directories="@(LegacyModOutput)"
             Condition="Exists('%(LegacyModOutput.Identity)')" />
  <RemoveDir Directories="$(UIBuildSourceDir)"
             Condition="Exists('$(UIBuildSourceDir)')" />
</Target>

<Target Name="BuildUI" AfterTargets="AfterBuild">
  <Exec Command="npm run build" WorkingDirectory="$(ProjectDir)ui" />
</Target>

<Target Name="CopyUIBundle" AfterTargets="DeployWIP">
  <ItemGroup>
    <UIBundle Include="$(UIBuildSourceDir)\**\*.*" />
  </ItemGroup>
  <Copy SourceFiles="@(UIBundle)"
        DestinationFiles="@(UIBundle->'$(DeployDir)\%(RecursiveDir)%(Filename)%(Extension)')" />
</Target>
```
- Replace `<ModuleId>` with the mod's published identifier (for example `vno-ui`) so MSBuild points at the deployed bundle location under `%AppData%\LocalLow\Colossal Order\Cities Skylines II\Mods`.
- Remove or expand `<LegacyId>` entries as you retire older module IDs; the cleanup target prevents stale bundles from coexisting with the current build.
- Keep the working directory relative to the `.csproj` file. If multiple modules share a UI project, swap in an absolute or repo-relative path and rely on `$(MSBuildThisFileDirectory)`.
- Add `ui/dist/` (or the toolchain output path) to `.gitignore`; regenerate bundles during builds rather than storing them in Git.

## Daily Build Loop
1. Run `dotnet build` from the module or solution root. The toolchain executes `ModPostProcessor.exe` and copies outputs to `%AppData%\LocalLow\Colossal Order\Cities Skylines II\Mods/<ModuleId>`.
2. Inside the UI project, run `npm run dev` when you need hot reload. With `-uiDeveloperMode`, the game loads assets from the dev server.
3. Launch the game with your developer shortcut, load an appropriate test save, and verify logs.

## Publishing
1. Build in Release mode (`dotnet build -c Release`) to strip debug symbols. Run `npm run build` if you maintain UI bundles separately.
2. Launch the game, sign into your Paradox account, and execute `PublishNewMod`, `PublishNewVersion`, or `UpdatePublishedConfiguration` from Rider or Visual Studio.
3. Confirm `PublishConfiguration.xml` lists every hard dependency (ExtraLib, Unified Icon Library, I18n Everywhere, etc.) and that the version matches the release notes.
4. Keep signed `PublishConfiguration.xml` files in the repository but never commit authentication tokens; the toolchain authenticates via the current game session.

## VS Code Build and Attach Tasks
VS Code can mirror the Visual Studio build pipeline by invoking the same MSBuild targets.

1. Create `.vscode/tasks.json` at the repository root:
   ```json
   {
     "version": "2.0.0",
     "tasks": [
       {
         "label": "Build Vice & Order (Debug)",
         "type": "shell",
         "command": "powershell.exe",
         "args": [
           "-NoProfile",
           "-ExecutionPolicy",
           "Bypass",
           "-Command",
           "$vswhere = Join-Path ${Env:ProgramFiles(x86)} 'Microsoft Visual Studio\\Installer\\vswhere.exe'; if (!(Test-Path $vswhere)) { throw 'vswhere.exe not found; install Visual Studio Build Tools.' }; $installationPath = & $vswhere -latest -products * -requires Microsoft.Component.MSBuild -property installationPath; if ([string]::IsNullOrWhiteSpace($installationPath)) { throw 'Unable to locate MSBuild via vswhere.' }; $msbuild = Join-Path $installationPath 'MSBuild\\Current\\Bin\\MSBuild.exe'; & $msbuild 'ViceAndOrder.sln' /t:Build /p:Configuration=Debug /m"
         ],
         "problemMatcher": "$msCompile",
         "group": {
           "kind": "build",
           "isDefault": true
         },
         "presentation": {
           "reveal": "always",
           "panel": "shared"
         }
       },
       {
         "label": "Build Vice & Order (Release)",
         "type": "shell",
         "command": "powershell.exe",
         "args": [
           "-NoProfile",
           "-ExecutionPolicy",
           "Bypass",
           "-Command",
           "$vswhere = Join-Path ${Env:ProgramFiles(x86)} 'Microsoft Visual Studio\\Installer\\vswhere.exe'; if (!(Test-Path $vswhere)) { throw 'vswhere.exe not found; install Visual Studio Build Tools.' }; $installationPath = & $vswhere -latest -products * -requires Microsoft.Component.MSBuild -property installationPath; if ([string]::IsNullOrWhiteSpace($installationPath)) { throw 'Unable to locate MSBuild via vswhere.' }; $msbuild = Join-Path $installationPath 'MSBuild\\Current\\Bin\\MSBuild.exe'; & $msbuild 'ViceAndOrder.sln' /t:Build /p:Configuration=Release /m"
         ],
         "problemMatcher": "$msCompile",
         "group": "build",
         "presentation": {
           "reveal": "always",
           "panel": "shared"
         }
       }
     ]
   }
   ```
   - Substitute the solution path if you relocate files.
   - `Ctrl+Shift+B` runs the default Debug build; use **Tasks: Run Task** to trigger the Release build when needed.
2. Add `.vscode/launch.json` for one-click rebuild-and-attach debugging:
   ```json
   {
     "version": "0.2.0",
     "configurations": [
       {
         "name": "Attach to Cities: Skylines II",
         "type": "coreclr",
         "request": "attach",
         "processId": "${command:pickProcess}",
         "preLaunchTask": "Build Vice & Order (Debug)",
         "justMyCode": false
       }
     ]
   }
   ```
3. With these files in place, the VS Code build task executes the same MSBuild targets as Visual Studio—including the UI cleanup—and the launch profile rebuilds before offering the `Cities2.exe` process picker.
