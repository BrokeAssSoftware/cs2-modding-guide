# Bootstrap a UI Project

Use the Gameface React template to add UI to a Vice & Order module and keep the identifiers aligned with the code mod.

1. Open a terminal in the code mod directory (the folder containing `<Module>.csproj`).
2. Scaffold the UI project:
   ```powershell
   npx create-csii-ui-mod
   ```
   Accept the defaults so the scaffold matches Colossal's expectations.
3. The script creates a directory (default `ui/`) containing `mod.json`, `package.json`, and a `src/` tree.
4. Ensure `mod.json.id` matches the code mod assembly name exactly.
5. Install dependencies if the scaffold did not run `npm install` automatically:
   ```powershell
   cd ui
   npm install
   ```
6. If you plan to ship the UI separately, add a `Mods/` symlink or copy of `mod.json`. For combined modules keep the UI project beside the code project.

Continue with [Build and Publish Workflow](build-and-publish.md) to hook the bundle into MSBuild.
