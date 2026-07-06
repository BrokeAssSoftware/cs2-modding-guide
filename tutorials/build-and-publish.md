---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Build, Test, and Publish Your Mod"
Summary: The daily edit-build-test loop for a CS2 mod, optional VS Code build/attach tasks, and a first walkthrough of publishing to PDX Mods with PublishConfiguration.xml.
diataxis: tutorial
source_version: "n/a - not source-verified (build/publish commands not run this pass)"
last_reverified: "2026-07-04"
status: needs-verification
Created: 2026-07-02
Updated: 2026-07-02
Owners:
  - codex
References:
  - Label: First Code Mod (prerequisite)
    Path: ./first-code-mod.md
  - Label: First UI Mod (UI build wiring)
    Path: ./first-ui-mod.md
  - Label: Getting Started (dev flags + toolchain)
    Path: ./getting-started.md
  - Label: Lifecycle and Initialization (concept)
    Path: ../explanation/mod-lifecycle.md
  - Label: Recipes index
    Path: ../how-to/recipes/README.md
---

# Build, Test, and Publish Your Mod

This tutorial gives you a repeatable rhythm for developing a mod day to day, and then
walks you through publishing it to **PDX Mods** for the first time. By the end you will
know the fast inner loop, an optional one-key build-and-attach setup for VS Code, and
the publish steps and their guardrails.

> **Before you start**, you should have a building mod from
> [First Code Mod](first-code-mod.md) (and optionally a UI bundle from
> [First UI Mod](first-ui-mod.md)). We keep using `MyMod.Core` as the placeholder ID.

---

## Part 1: The daily build loop

This is the cycle you repeat all day: edit, build, run, check logs.

1. **Build.** From the mod (or solution) root:
   ```powershell
   dotnet build
   ```
   The toolchain compiles, runs its post-processor, and copies the output to the
   deployed mods folder:
   ```
   %AppData%\LocalLow\Colossal Order\Cities Skylines II\Mods\MyMod.Core
   ```
2. **Run the UI dev server (only if you have UI).** In the `ui/` folder:
   ```powershell
   npm run dev
   ```
   With `-uiDeveloperMode` set, the game loads UI assets live from this server so you
   skip rebuilds for UI-only changes. (See [First UI Mod](first-ui-mod.md).)
3. **Launch and test.** Start the game with your developer shortcut (the one carrying
   `-developerMode`), load your regression save, and exercise the change.
4. **Watch the log.** Keep an eye on
   `%AppData%\LocalLow\Colossal Order\Cities Skylines II\Logs\Log_<date>.txt` for new
   warnings or exceptions and fix them before moving on.

> **Tip:** keep a sandbox save with everything unlocked purely for testing, so you can
> reproduce behavior quickly and consistently.

---

## Part 2: One-key build-and-attach in VS Code (optional)

Rider and Visual Studio drive the build and debugger out of the box. VS Code can do the
same by calling the same MSBuild targets. Set this up once and `Ctrl+Shift+B` builds,
plus you get one-click rebuild-and-attach debugging.

### tasks.json

Create `.vscode/tasks.json` at the repo root. This task locates MSBuild via `vswhere`
and builds your solution in Debug:

```json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "Build MyMod (Debug)",
      "type": "shell",
      "command": "powershell.exe",
      "args": [
        "-NoProfile", "-ExecutionPolicy", "Bypass", "-Command",
        "$vswhere = Join-Path ${Env:ProgramFiles(x86)} 'Microsoft Visual Studio\\Installer\\vswhere.exe'; if (!(Test-Path $vswhere)) { throw 'vswhere.exe not found; install Visual Studio Build Tools.' }; $installationPath = & $vswhere -latest -products * -requires Microsoft.Component.MSBuild -property installationPath; if ([string]::IsNullOrWhiteSpace($installationPath)) { throw 'Unable to locate MSBuild via vswhere.' }; $msbuild = Join-Path $installationPath 'MSBuild\\Current\\Bin\\MSBuild.exe'; & $msbuild 'MyMod.sln' /t:Build /p:Configuration=Debug /m"
      ],
      "problemMatcher": "$msCompile",
      "group": { "kind": "build", "isDefault": true },
      "presentation": { "reveal": "always", "panel": "shared" }
    }
  ]
}
```

Substitute `MyMod.sln` with your solution path. Duplicate the task with
`Configuration=Release` for a Release variant, and run it through **Tasks: Run Task**.

Because this invokes the same MSBuild targets as Visual Studio, it also runs your UI
cleanup/bundle targets from [First UI Mod](first-ui-mod.md).

### launch.json

Create `.vscode/launch.json` for rebuild-then-attach:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Attach to Cities: Skylines II",
      "type": "coreclr",
      "request": "attach",
      "processId": "${command:pickProcess}",
      "preLaunchTask": "Build MyMod (Debug)",
      "justMyCode": false
    }
  ]
}
```

Now starting the debugger rebuilds first, then offers the `Cities2.exe` process picker.
Set `justMyCode` to `false` so you can step into game/framework frames when needed.

---

## Part 3: Publish to PDX Mods

When the mod is ready to share, publish it through the toolchain. Publishing
authenticates against your live game session - you do not paste tokens anywhere.

1. **Build in Release.** Strip debug symbols with:
   ```powershell
   dotnet build -c Release
   ```
   Run `npm run build` too if you maintain the UI bundle separately from the C# build.
2. **Fill in `PublishConfiguration.xml`.** In the mod's `Properties/` folder, confirm:
   - Display name, description, and thumbnail are real (not the placeholders from
     [First Code Mod](first-code-mod.md)).
   - The **version** matches your release notes.
   - Every **hard dependency** is listed by both name and mod ID (for example a shared
     icon library or a localization framework your mod requires). A missing dependency
     entry means the mod loads for you but breaks for players who lack it.
3. **Publish from your IDE.** Launch the game, sign into your Paradox account, then run
   the appropriate command from Rider or Visual Studio:
   - **`PublishNewMod`** - the very first publish; creates the PDX Mods listing.
   - **`PublishNewVersion`** - ship an update to an already-published mod.
   - **`UpdatePublishedConfiguration`** - change listing metadata (description,
     dependencies, thumbnail) without shipping new code.

   `Needs Verification`: the exact command names/menu entries are provided by the
   toolchain's MSBuild targets and may be renamed across versions - confirm against the
   targets your template imports.

### Publishing guardrails

- **Commit the config, never the secrets.** Keep the signed `PublishConfiguration.xml`
  in your repo so releases are reproducible, but it must never contain authentication
  tokens - the toolchain authenticates through the running game session at publish time.
- **Bump the version every release.** Match the version in `PublishConfiguration.xml`
  to your changelog so players can tell what they are getting.
- **Re-verify dependencies each release.** Dependencies drift; a hard dependency added
  mid-development but missing from the config is the classic "works on my machine"
  failure.

---

## You've shipped

You now have a daily loop, optional editor integration, and a repeatable publish path.

Where to go next:

- **Understand load order and initialization** before adding more systems:
  [Lifecycle and Initialization](../explanation/mod-lifecycle.md).
- **Pick your next technique** from the [Recipes](../how-to/recipes/README.md) and the
  [Technique Index](../technique-index.md).
