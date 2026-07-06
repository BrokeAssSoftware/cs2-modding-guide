---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Getting Started: Install the CS2 Modding Toolchain"
Summary: A first-run tutorial that installs the official Cities Skylines II modding toolchain, enables developer flags, and verifies the environment is healthy before you write any code.
diataxis: tutorial
source_version: "n/a - not source-verified (toolchain commands not run this pass)"
last_reverified: "2026-07-04"
status: needs-verification
Created: 2026-07-02
Updated: 2026-07-02
Owners:
  - codex
References:
  - Label: First Code Mod (next tutorial)
    Path: ./first-code-mod.md
  - Label: Lifecycle and Initialization (concept)
    Path: ../explanation/mod-lifecycle.md
  - Label: Recipes index
    Path: ../how-to/recipes/README.md
  - Label: Technique Index
    Path: ../technique-index.md
---

# Getting Started: Install the CS2 Modding Toolchain

This tutorial walks you from a plain copy of Cities: Skylines II to a workstation
that can build mods. By the end you will have the official toolchain installed,
developer tools switched on in-game, and a short checklist confirming everything is
wired up correctly. No code yet - that is the [next tutorial](first-code-mod.md).

You will do four things, in order:

1. Turn on developer flags so the game exposes its modding menus and tools.
2. Install the toolchain (Unity, the mod project template, .NET, Node) from the
   in-game Modding panel.
3. Install the UI developer certificate so the browser-based inspector opens cleanly.
4. Run a quick health check so you know the environment is sound.

> **Who this is for.** A modder setting up for the first time on Windows. You need
> the base game installed and a Paradox account. Everything else is installed below.

---

## Step 1: Turn on developer flags

Two launch flags unlock the modding experience:

- `-developerMode` enables the in-game developer menu, the object/entity browser, and
  the Modding options page.
- `-uiDeveloperMode` enables the browser-based UI inspector served on
  `http://localhost:9444/`.

Pick the profile that matches how you installed the game. You only need one.

### Steam

1. In your Steam library, right-click **Cities: Skylines II** and choose **Properties**.
2. In **Launch Options**, paste:
   ```
   -developerMode -uiDeveloperMode
   ```
3. Launch the game from Steam.

### Xbox App / PC Game Pass

The Game Pass build launches through the store, so add the flags to a desktop shortcut
instead of a launch-options box.

1. In the Xbox app, open **Cities: Skylines II -> Manage -> Files -> Browse** to reveal
   the install folder (commonly `C:\XboxGames\Cities Skylines II\Content`).
2. Right-click `Cities2.exe` and choose **Create shortcut**.
3. Edit the shortcut's **Target** so the flags follow the quoted path:
   ```
   "C:\XboxGames\Cities Skylines II\Content\Cities2.exe" -developerMode -uiDeveloperMode
   ```
   Adjust the path if your install lives elsewhere.
4. Launch the game with this shortcut.

### Direct executable shortcut (any install)

If you would rather keep a dedicated "dev" launcher:

1. Browse to the game folder, e.g.
   `C:\Program Files (x86)\Steam\steamapps\common\Cities Skylines II`.
2. Right-click `Cities2.exe` -> **Create shortcut**.
3. Set the shortcut **Target** to:
   ```
   "C:\Program Files (x86)\Steam\steamapps\common\Cities Skylines II\Cities2.exe" -developerMode -uiDeveloperMode
   ```
4. Rename it something obvious like `Cities II (Dev)`.

> **Keep a plain launcher too.** Make a second shortcut *without* the flags. When you
> need to reproduce how the mod behaves for regular players, launch that one - developer
> mode changes some behavior and adds overlays.

### Confirm the flags took effect

With the game running:

- Press **Tab** to open the developer menu.
- Press **Home** to open the object browser.
- In any browser, visit `http://localhost:9444/` - you should reach the UI debugger.
  (A certificate warning here is expected until Step 3.)

If none of these appear, the flags are not being passed. Re-check the exact spelling
and the quoting of the path.

---

## Step 2: Install the toolchain from the Modding panel

The game ships its own dependency installer. Do not hunt down these components
manually - the panel installs versions the game expects.

1. In-game, open **Options -> Modding**.
2. Work down the dependency list, clicking **Install** on each entry until every row
   shows a green check mark:
   - **Unity** - opens Unity Hub to activate your license and seeds the Burst / IL
     post-processors the game needs to build mod assemblies.
   - **Unity Mod Project** - copies a template project into your user-data folder and
     opens it once. Let that first open finish before moving on.
   - **.NET SDK** - the C# build toolchain. Install it here if the panel reports it
     missing.
   - **Node.js** - required for building UI (Gameface/React) bundles. After it
     installs, restart any open terminals and your IDE so the updated `PATH` is picked
     up.
3. If a game update ever leaves an entry showing **Corrupted**, use **Repair** on that
   row rather than reinstalling from scratch.

> **Versions come from the panel, not this page.** The exact Unity / .NET / Node
> versions are whatever the current game build pins. Trust the panel's list over any
> version number written down elsewhere - it moves with each CS2 release.
> `Needs Verification`: exact pinned versions for 1.6.x.

Optional tools - **Git** and **VS Code** (or Rider / Visual Studio) - are best
installed yourself so you control the versions. You will want one code editor before
the next tutorial.

### The user-data folder

The toolchain reads and writes a per-user folder. It is worth knowing where it is,
because health checks and build output land there:

- **Templates & local mods:**
  `%LocalAppData%\Colossal Order\Cities Skylines II\`
- **Deployed mods, settings, saves, and logs:**
  `%AppData%\LocalLow\Colossal Order\Cities Skylines II\`

`Needs Verification`: exact subfolder names can shift between game versions; confirm
against your own install.

---

## Step 3: Install the UI developer certificate

The UI inspector at `http://localhost:9444/` is served over a local dev certificate.
Installing it once per machine stops the browser HTTPS warning. If you scaffold a UI
project later (see [First UI Mod](first-ui-mod.md)), that scaffold ships a helper:

```powershell
npm run install-cert
```

Run it from the UI project folder once. If you are not doing UI work yet, you can skip
this and come back when you scaffold your first UI bundle.

---

## Step 4: Verify your environment

Run this checklist now, and re-run it whenever something starts misbehaving - most
"my mod won't load" problems are really environment drift.

- [ ] **Toolchain is all green.** Every row in **Options -> Modding** shows a check
      mark.
- [ ] **.NET works.** In a fresh terminal:
      ```powershell
      dotnet --list-sdks
      ```
      lists at least one SDK.
- [ ] **Node works.** `node --version` prints a version. If it errors, you did not
      restart the shell after installing Node.
- [ ] **Developer tools respond.** Tab opens the dev menu; `http://localhost:9444/`
      loads without a certificate warning (after Step 3).
- [ ] **A test save exists.** Keep at least one sandbox save with everything unlocked.
      You will reload it constantly to test changes.
- [ ] **Logs are clean-ish.** During any test session, watch
      `%AppData%\LocalLow\Colossal Order\Cities Skylines II\Logs\Log_<date>.txt`
      and address *new* warnings promptly.

> **After your first mod runs**, two more folders appear once a mod loads (named after
> your mod's ID): a settings folder and a data folder under the user-data path. If they
> never appear, the mod failed to load - check the log. You will see these named in the
> [First Code Mod](first-code-mod.md) tutorial.

---

## You're done

Your workstation can now build and run mods. Next steps:

- **Build your first mod:** [First Code Mod](first-code-mod.md) scaffolds a C# mod from
  the template and builds it once.
- **Add a UI:** [First UI Mod](first-ui-mod.md) scaffolds a Gameface/React bundle.
- **Understand what happens at load time:** [Lifecycle and Initialization](../explanation/mod-lifecycle.md).
- **Browse techniques:** the [Technique Index](../technique-index.md) and
  [Recipes](../how-to/recipes/README.md).
