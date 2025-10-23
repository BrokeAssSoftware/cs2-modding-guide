# Launch Profiles and Developer Flags

Enable developer flags so you can access debugging tools, the object browser, and the UI inspector. Use the profile that matches your installation.

## Steam
1. In Steam, right-click *Cities: Skylines II* and choose *Properties*.
2. Under *Launch Options*, enter:
   ```
   -developerMode -uiDeveloperMode
   ```
3. Start the game from Steam. Press `Tab` to open the developer menu and `Home` to open the object browser. Visit `http://localhost:9444/` in a browser to confirm the UI debugger is active.

## Xbox App / PC Game Pass
1. In the Xbox app, open *Cities: Skylines II* -> *Manage* -> *Files* -> *Browse* to open the install folder (typically `C:\XboxGames\Cities Skylines II\Content`).
2. Right-click `Cities2.exe` and select *Create shortcut*. Windows places the shortcut on the desktop.
3. Edit the shortcut target so it includes:
   ```
   "C:\XboxGames\Cities Skylines II\Content\Cities2.exe" -developerMode -uiDeveloperMode
   ```
   Adjust the path if your install directory differs.
4. Launch the game with the shortcut and verify the developer tools as above.

## Direct Executable Shortcut
1. Browse to `C:\Program Files (x86)\Steam\steamapps\common\Cities Skylines II`.
2. Right-click `Cities2.exe` -> *Create shortcut*.
3. Edit the shortcut target to:
   ```
   "C:\Program Files (x86)\Steam\steamapps\common\Cities Skylines II\Cities2.exe" -developerMode -uiDeveloperMode
   ```
4. Rename the shortcut (for example, `Cities II (Dev)`) so you can distinguish it from shipping builds.

Keep a second shortcut without the flags when you need to reproduce shipping behaviour.
