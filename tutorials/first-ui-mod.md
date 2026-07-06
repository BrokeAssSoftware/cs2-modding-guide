---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Your First UI Mod"
Summary: Scaffold a React (SCSS/TypeScript) UI bundle for a CS2 code mod, align its ID with the code project, wire the UI build into MSBuild, and get hot reload working.
diataxis: tutorial
source_version: "n/a - toolchain/tutorial, wiki-sourced (Modding_Toolchain stale-verified 1.1.12f1)"
last_reverified: "2026-07-04"
status: needs-verification
Created: 2026-07-02
Updated: 2026-07-02
Owners:
  - codex
References:
  - Label: First Code Mod (prerequisite)
    Path: ./first-code-mod.md
  - Label: Build and Publish (daily loop)
    Path: ./build-and-publish.md
  - Label: Getting Started (toolchain + cert)
    Path: ./getting-started.md
  - Label: Recipes index
    Path: ../how-to/recipes/README.md
  - Label: Technique Index
    Path: ../technique-index.md
---

# Your First UI Mod

Cities: Skylines II builds its interface with **React** (SCSS for styles, TypeScript for
code), per the official [UI Modding](https://cs2.paradoxwikis.com/UI_Modding) wiki page.
The Chromium-based runtime it renders into is commonly called "Gameface" (or "Coherent
Gameface") in the community; the wiki itself only names the "Chrome Debugger" inspector,
so treat "Gameface" as community terminology.

This tutorial is the **UI layer** of the official CS2 mod-creation route:

1. Install the toolchain from the in-game **Modding Options** panel - it installs and
   manages the IDE, Unity Editor (2022.3.7f1), .NET SDK (8 recommended), and Node.js.
   See [Getting Started](getting-started.md).
2. Scaffold the **code mod** from the IDE template (Rider / Visual Studio) and edit
   `PublishConfiguration.xml`. See [First Code Mod](first-code-mod.md).
3. Add the **UI layer** with `npx create-csii-ui-mod` and wire its build into MSBuild -
   this page.

Here you will scaffold a UI bundle onto that existing code mod, keep the two projects'
identifiers aligned, wire the UI build into the C# build so `dotnet build` bundles both,
and see your changes hot-reload live in the running game.

> **Provenance.** The `npx create-csii-ui-mod` scaffolder, the `--uiDeveloperMode` launch
> flag, the `http://localhost:9444` Chrome Debugger inspector, and the auto-copy-to-Mods
> build step are all **officially documented** on the CS2 wiki
> ([Modding_Toolchain](https://cs2.paradoxwikis.com/Modding_Toolchain),
> [Creating UI and Code Mods](https://cs2.paradoxwikis.com/Creating_UI_And_Code_Mods)) -
> they are not community-only tricks. Caveat: the Modding_Toolchain page is still stamped
> "Verified ... for version 1.1.12f1" while the live game is 1.6.0, so verify every
> command, flag, version, and path against the current client before relying on it.

> **Before you start:**
> - Finish [First Code Mod](first-code-mod.md) - this tutorial adds UI to that project
>   (`MyMod.Core`, in a repo at `my-mod/`).
> - Node.js must be installed (from [Getting Started](getting-started.md)); confirm with
>   `node --version`.

We use the placeholder UI mod ID `MyMod.UI`. Substitute your own, but read Step 2 first
- the ID has to match the code mod.

---

## Step 1: Scaffold the UI bundle

The official toolchain ships a scaffolder run through `npx`. `npx create-csii-ui-mod` is
the **officially documented** UI scaffolder - it appears on both the
[Modding_Toolchain](https://cs2.paradoxwikis.com/Modding_Toolchain) and
[Creating UI and Code Mods](https://cs2.paradoxwikis.com/Creating_UI_And_Code_Mods) wiki
pages - not a community-only trick. (Whether the npm package itself is first-party or
community-maintained is not settled by the sources; the command is documented regardless.)

1. Open a terminal in the code mod directory - the folder that contains
   `MyMod.Core.csproj`.
2. Run the scaffolder and accept the defaults so the output matches what the game
   expects:
   ```powershell
   npx create-csii-ui-mod
   ```
3. It creates a UI directory (default `ui/`) containing:
   - `mod.json` - the UI bundle's manifest (this holds the ID).
   - `package.json` - npm scripts and dependencies.
   - `src/` - your React source.
4. If the scaffolder did not install dependencies automatically, do it yourself:
   ```powershell
   cd ui
   npm install
   ```

Your project now looks like:

```
my-mod/
  MyMod.Core/
    MyMod.Core.csproj
    ui/
      mod.json
      package.json
      src/
```

---

## Step 2: Align the UI ID with the code mod

This is the step people miss. The game pairs a UI bundle to its code mod by **ID**, so
the two must match exactly.

1. Open `ui/mod.json`.
2. Set its `id` to match the code mod's assembly name exactly:
   ```json
   { "id": "MyMod.Core" }
   ```

If the IDs disagree, the game loads the code mod and the UI separately (or not at all)
and your panel never appears. Fix this before you build.

> **Shipping UI separately?** If you ever ship the UI as its own mod, you would give it
> its own ID (`MyMod.UI`) and reference it as a dependency instead. For a single
> combined mod, keep the UI beside the code and share the one ID as above.

---

## Step 3: Wire the UI build into MSBuild

You want one command - `dotnet build` - to compile the C#, build the React bundle, and
deploy both together. Add MSBuild targets to `MyMod.Core.csproj` that run the UI build
and copy its output into the deployed mod folder. The
`<Target Name="BuildUI" AfterTargets="AfterBuild">` running `npm run build` below is the
exact wiring shown on the official
[Creating UI and Code Mods](https://cs2.paradoxwikis.com/Creating_UI_And_Code_Mods) page;
the toolchain also copies the finished mod to the local Mods folder (`CSII_LOCALMODSPATH`)
as the last build step.

```xml
<PropertyGroup>
  <!-- Where the deployed bundle for this mod lives -->
  <UIBuildSourceDir>$(CSII_USERDATAPATH)\Mods\MyMod.Core</UIBuildSourceDir>
</PropertyGroup>

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

What each piece does:

- **`UIBuildSourceDir`** points at your mod's deployed folder using the
  `CSII_USERDATAPATH` environment variable the toolchain sets - not a hard-coded path.
- **`BuildUI`** runs `npm run build` in the `ui/` folder after the C# compile, so the
  React bundle is always fresh.
- **`CopyUIBundle`** copies the built assets into the deploy directory so the game picks
  them up.

Two housekeeping notes:

- **Keep `WorkingDirectory` relative to the `.csproj`** (`$(ProjectDir)ui`). If several
  code projects share one UI project, use `$(MSBuildThisFileDirectory)` with a repo-
  relative path instead so the build does not depend on which project triggered it.
- **Do not commit the built bundle.** Add the UI output folder (e.g. `ui/dist/`) to
  `.gitignore`; the bundle is regenerated on every build.

> `Needs Verification`: the exact deploy-target names (`AfterBuild`, `DeployWIP`,
> `$(DeployDir)`) come from the template's own targets and can be renamed across
> toolchain versions. Open the template's imported `.targets` file to confirm the names
> your version uses.

Now build the whole thing once:

```powershell
dotnet build
```

The C# compiles, `npm run build` runs, and the bundle deploys into
`%AppData%\LocalLow\Colossal Order\Cities Skylines II\Mods\MyMod.Core`. Note the path
still uses the **"Colossal Order"** folder name (the wiki's documented `CSII_LOCALMODSPATH`
target) despite the Iceflake Studios handover - the folder name is unchanged.

---

## Step 4: Get hot reload working

For UI work you do not want a full rebuild after every tweak. The dev server serves your
React bundle live, and the game loads from it when UI developer mode is on. The launch
flag and the `http://localhost:9444` **Chrome Debugger** inspector are both **officially
documented** on the [Modding_Toolchain](https://cs2.paradoxwikis.com/Modding_Toolchain)
wiki page.

> `Needs Verification` (flag spelling): the wiki prints the flag with **two dashes**
> (`--uiDeveloperMode`), while community shortcuts and older docs commonly use **one**
> (`-uiDeveloperMode`). Unity / Colossal Order command-line flags are historically
> single-dash, so the wiki's double dash may be a typo. Confirm the exact spelling against
> the live 1.6.0 client before committing it to a shortcut. This page uses
> `--uiDeveloperMode` to match the wiki.

1. Install the UI developer certificate once per machine (skips the HTTPS warning on the
   inspector). From the `ui/` folder:
   ```powershell
   npm run install-cert
   ```
2. Start the dev server in the `ui/` folder:
   ```powershell
   npm run dev
   ```
3. Launch the game with your developer shortcut (it must include `--uiDeveloperMode` -
   see the flag-spelling note above; from [Getting Started](getting-started.md)) and load
   a test save.
4. Edit a file under `ui/src/` and save. The panel updates in the running game without a
   rebuild.
5. Open `http://localhost:9444/` - the officially documented **Chrome Debugger**
   inspector - to inspect the live UI DOM and debug styles the same way you would inspect
   a web page.

> **If hot reload does nothing:** the usual cause is that the game was launched *without*
> `--uiDeveloperMode`, so it loaded the deployed static bundle instead of the dev server.
> Relaunch with the dev shortcut.

---

## You're done

You have a UI bundle that shares its code mod's ID, builds and deploys through a single
`dotnet build`, and hot-reloads during development.

Next steps:

- **Daily loop and publishing:** [Build and Publish](build-and-publish.md).
- **Browse UI techniques:** the [Recipes](../how-to/recipes/README.md) and
  [Technique Index](../technique-index.md) (see the `ui` applicability tag) for panels,
  bindings, and C#-to-React communication.
