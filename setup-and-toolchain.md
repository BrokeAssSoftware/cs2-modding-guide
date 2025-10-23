# Setup And Toolchain

Prime every workstation with the official Cities: Skylines II toolchain before touching Vice & Order modules. These steps assume Windows because that is the only supported build target today.

## Launch Parameters & Profiles
- **Steam**  
  1. In Steam, right-click *Cities: Skylines II* → *Properties*.  
  2. Under *Launch Options*, paste:  
     ```
     --developerMode --uiDeveloperMode
     ```  
  3. Launch the game from Steam. The developer menu (`Tab`) and object browser (`Home`) confirm the flag is active; `http://localhost:9444/` should open the Gameface inspector.
- **Xbox app / PC Game Pass**  
  1. In the Xbox app, open *Cities: Skylines II* → *Manage* → *Files* → *Browse*. This opens the install folder (typically `C:\XboxGames\Cities Skylines II\Content\`).  
  2. Right-click `Cities2.exe` → *Create shortcut*. Windows places the shortcut on the desktop.  
  3. Edit the shortcut target to include the launch parameters, for example:  
     ```
     "C:\XboxGames\Cities Skylines II\Content\Cities2.exe" --developerMode --uiDeveloperMode
     ```  
     If your install path differs, copy the path from the shortcut’s *Start in* field.  
  4. Launch the game using that shortcut to ensure the flags are honoured (look for the developer menu and UI debugger as above).
- **Direct executable / desktop shortcut**  
  1. Browse to `...\Steam\steamapps\common\Cities Skylines II\`.  
  2. Right-click `Cities2.exe` → *Create shortcut*.  
  3. Edit the shortcut target to:  
     ```
     "C:\Program Files (x86)\Steam\steamapps\common\Cities Skylines II\Cities2.exe" --developerMode --uiDeveloperMode
     ```  
  4. Rename it to `Cities II (Dev)` so QA builds stay clean.
- If you need a “shipping” profile (no flags), keep a second shortcut with no launch options for parity testing.

## Install The Toolchain (In-Game Workflow)
1. Launch the game with the dev profile and open *Options → Modding*.  
2. Work down the dependency list until every entry shows a green check mark:  
   - **Unity 2022.3.7f1** – click *Install*. Unity Hub will open once to activate your license; this also seeds the Unity-based Burst toolchain.  
   - **Unity Mod Project** – select *Install*. The toolchain copies a template project to `%LocalAppData%\Colossal Order\Cities Skylines II\Modding` and opens it once; let it complete.  
   - **.NET SDK 8.0** – install if the entry shows *Missing*. Verify with `dotnet --list-sdks` afterwards.  
   - **Node.js 18+** – install or repair from the same panel. Restart any shells/IDEs after the install so they pick up the Node path.  
3. Use the *Repair* button if any entry reports corruption after an update.  
4. Optional dependencies such as Git or VS Code are best installed manually so you can manage versions centrally.

## Bootstrap A Code Mod
1. Open *Options → Modding → Projects* and click *Create*.  
2. Pick a template (`Code Mod`) and enter the module ID (`VNO.Core`, etc.).  
3. The toolchain writes a solution under `%LocalAppData%\Colossal Order\Cities Skylines II\Mods\Local\<ModuleId>\`. Move it into the Vice & Order repo (e.g., `vno-core/`) and add it to source control.  
4. Update `Properties/PublishConfiguration.xml` immediately with placeholder metadata. Publishing scripts fail if the file is blank.  
5. Open the solution in Rider or Visual Studio; restore packages and build once (`dotnet build` or IDE build button) to confirm the pipeline is healthy.

## Bootstrap A UI Project Beside The Code Mod
1. Open a terminal in the code mod directory (the folder containing `<Module>.csproj`).  
2. Run the scaffold command:  
   ```powershell
   npx create-csii-ui-mod
   ```  
   Accept the defaults unless you have a reason to change them—aligning with Colossal’s template keeps the asset pipeline predictable.  
3. The script creates a folder (default `ui`) containing `mod.json`, `package.json`, and a `src/` tree. Ensure `mod.json.id` matches the code mod assembly name.  
4. Install dependencies if the scaffold did not run `npm install` automatically:  
   ```powershell
   cd ui
   npm install
   ```  
5. Add a `Mods/` symlink or `mod.json` copy if you plan to ship the UI as a separate bundle; for combined modules keep it co-located with the code project.

## Wire The UI Build Into MSBuild
Add a build target to the code mod project so every `dotnet build` also bundles the UI:

```xml
<!-- Inside <Project> of <Module>.csproj -->
<Target Name="BuildUI" AfterTargets="AfterBuild">
  <Exec Command="npm run build" WorkingDirectory="$(ProjectDir)ui" />
</Target>

<Target Name="CopyUIAssets" AfterTargets="DeployWIP">
  <ItemGroup>
    <UIBundle Include="ui\dist\**\*.*" />
  </ItemGroup>
  <Copy SourceFiles="@(UIBundle)"
        DestinationFiles="@(UIBundle->'$(DeployDir)\%(RecursiveDir)%(Filename)%(Extension)')" />
</Target>
```

Key tips:
- Keep the working directory relative to the `.csproj`; avoid absolute paths (CI runners may map drives differently).  
- Add `ui/dist/` to `.gitignore`; the bundle is rebuilt on demand.  
- If multiple modules share the UI project, switch the working directory to a repo-relative path and use `$(MSBuildThisFileDirectory)` to keep paths stable.

## Daily Build Loop
1. **Compile C#** – run `dotnet build` from the module root or the solution root. This triggers `ModPostProcessor.exe` and copies the output to `%AppData%\LocalLow\Colossal Order\Cities Skylines II\Mods\<ModuleId>\`.  
2. **Run the UI watcher (optional)** – inside the UI folder run `npm run dev`. With `--uiDeveloperMode`, the game requests assets from the dev server instead of the packaged bundle, giving you hot reload.  
3. **Start the game** with your dev shortcut. Verify the dev menu and UI inspector are active, then load a test city or the main menu depending on what the module needs.  
4. **Validate logs** – check `%AppData%\LocalLow\Colossal Order\Cities Skylines II\Logs\Log_<date>.txt` for dependency warnings before shipping changes.

## Publishing Workflow
1. Build in `Release` configuration to avoid debug-only symbols (`dotnet build -c Release`).  
2. Launch the game, sign into your Paradox account, and run the template publisher (`PublishNewMod`, `PublishNewVersion`, or `UpdatePublishedConfiguration`) from Rider or Visual Studio.  
3. Cross-check that `PublishConfiguration.xml` lists every hard dependency (ExtraLib, I18n Everywhere, etc.) and that the version matches the intended release.  
4. Keep signed `PublishConfiguration.xml` files in the repo but never commit authentication tokens; the toolchain handles auth via the game session.

## Environment Health Checks
- Ensure `ModsSettings/<ModuleId>` and `ModsData/<ModuleId>` folders exist after the first run; missing directories usually mean the mod failed to load.  
- Install the Gameface developer certificate (run the scaffold’s certificate script—currently `npm run install-cert`) once per machine to avoid HTTPS warnings when opening `http://localhost:9444/`.  
- Maintain at least one clean sandbox save with all unlocks enabled for integration testing.  
- Document any additional local prerequisites (database dumps, telemetry proxies) in the module’s `Agents.md`.

With these pieces in place, new contributors can clone the repo, run the toolchain, and begin building both code and UI mods with the same workflow automation we rely on across Vice & Order.
