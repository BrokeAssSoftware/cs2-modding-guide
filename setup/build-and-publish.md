# Build and Publish Workflow

Integrate the UI bundle into MSBuild, follow a daily development loop, and publish releases through the toolchain.

## Wire the UI Build into MSBuild
Add build targets to the code mod project so `dotnet build` also bundles the UI:
```xml
<Target Name="BuildUI" AfterTargets="AfterBuild">
  <Exec Command="npm run build" WorkingDirectory="$(ProjectDir)ui" />
</Target>

<Target Name="CopyUIBundle" AfterTargets="DeployWIP">
  <ItemGroup>
    <UIBundle Include="ui\dist\**\*.*" />
  </ItemGroup>
  <Copy SourceFiles="@(UIBundle)"
        DestinationFiles="@(UIBundle->'$(DeployDir)\%(RecursiveDir)%(Filename)%(Extension)')" />
</Target>
```
- Keep the working directory relative to the `.csproj` file.
- Add `ui/dist/` to `.gitignore`; regenerate the bundle during builds.
- If multiple modules share a UI project, adjust the working directory to a repo-relative path and rely on `$(MSBuildThisFileDirectory)`.

## Daily Build Loop
1. Run `dotnet build` from the module or solution root. The toolchain executes `ModPostProcessor.exe` and copies outputs to `%AppData%\LocalLow\Colossal Order\Cities Skylines II\Mods/<ModuleId>`.
2. Inside the UI project, run `npm run dev` when you need hot reload. With `-uiDeveloperMode`, the game loads assets from the dev server.
3. Launch the game with your developer shortcut, load an appropriate test save, and verify logs.

## Publishing
1. Build in Release mode (`dotnet build -c Release`) to strip debug symbols. Run `npm run build` if you maintain UI bundles separately.
2. Launch the game, sign into your Paradox account, and execute `PublishNewMod`, `PublishNewVersion`, or `UpdatePublishedConfiguration` from Rider or Visual Studio.
3. Confirm `PublishConfiguration.xml` lists every hard dependency (ExtraLib, Unified Icon Library, I18n Everywhere, etc.) and that the version matches the release notes.
4. Keep signed `PublishConfiguration.xml` files in the repository but never commit authentication tokens; the toolchain authenticates via the current game session.
