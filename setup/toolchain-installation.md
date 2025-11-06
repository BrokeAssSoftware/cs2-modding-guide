# Toolchain Installation

Install the official toolchain through the in-game Modding panel before building any Vice & Order module.

1. Launch the game with the developer launch profile and open *Options -> Modding*.
2. Work down the dependency list until every entry shows a green check mark:
   - **Unity 2022.3.7f1** - click *Install*. Unity Hub opens to activate your license and seeds the Burst/IL post processors.
   - **Unity Mod Project** - click *Install*. The toolchain copies a template project to `%LocalAppData%\Colossal Order\Cities Skylines II\Modding` and opens it once; let it finish.
   - **.NET SDK 8.0** - install if the panel reports *Missing*. Verify with `dotnet --list-sdks` afterward.
   - **Node.js 18+** - install or repair. Restart shells and IDEs so the PATH update is recognised.
3. Use *Repair* if any entry shows as corrupted after a game update.
4. Optional dependencies such as Git or VS Code are best installed manually so you control their versions.

Once the toolchain is fully green, move on to [Bootstrap a Code Mod](code-mod-bootstrap.md).
