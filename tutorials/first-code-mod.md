---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Your First Code Mod"
Summary: Scaffold a C# code mod from the official CS2 template, give it a proper mod ID, move it into a Git repository, and build it once to confirm the pipeline works.
diataxis: tutorial
source_version: "n/a - not source-verified (toolchain commands not run this pass)"
last_reverified: "2026-07-04"
status: needs-verification
Created: 2026-07-02
Updated: 2026-07-02
Owners:
  - codex
References:
  - Label: Getting Started (prerequisite)
    Path: ./getting-started.md
  - Label: First UI Mod (next tutorial)
    Path: ./first-ui-mod.md
  - Label: Build and Publish (daily loop)
    Path: ./build-and-publish.md
  - Label: Lifecycle and Initialization (concept)
    Path: ../explanation/mod-lifecycle.md
  - Label: Recipes index
    Path: ../how-to/recipes/README.md
---

# Your First Code Mod

In this tutorial you will generate a working C# code mod from the official template,
name it, put it under source control, and build it once so you know the toolchain
compiles and deploys cleanly. The result is an empty-but-valid mod you can grow from.

> **Before you start**, complete [Getting Started](getting-started.md). You need the
> toolchain fully installed (all green in **Options -> Modding**), `dotnet` on your
> `PATH`, and a code editor (Rider, Visual Studio, or VS Code).

We use the placeholder mod ID `MyMod.Core` throughout. Substitute your own - a dotted
`Vendor.Purpose` name (for example `Acme.Traffic`) is the common convention.

---

## Step 1: Create the project from the template

The template lives inside the game's Modding panel, not a `git clone`.

1. In-game, open **Options -> Modding -> Projects** and click **Create**.
2. Choose the **Code Mod** template.
3. Enter your mod ID: `MyMod.Core`.
4. The toolchain writes a ready-to-build solution into your local mods folder:
   ```
   %LocalAppData%\Colossal Order\Cities Skylines II\Mods\Local\MyMod.Core
   ```

That folder now contains a `.csproj`, a `Mod` class that implements `IMod`, and the
scaffolding that hooks your mod into the game's build/deploy pipeline.

> **What is `IMod`?** It is the tiny interface every code mod implements - the game
> calls `OnLoad` when your mod activates and `OnDispose` when it unloads. You do not
> need to understand it to finish this tutorial, but read
> [Lifecycle and Initialization](../explanation/mod-lifecycle.md) before you add real
> behavior.

---

## Step 2: Move it into a repository

The template drops the project in the *local mods* folder, which is not where you want
to develop long-term. Move it into a real repository so you get version history and can
collaborate.

1. Create (or open) a Git repo for your mod, for example a `my-mod/` folder.
2. Move the generated project into it - a clean layout is one project per subfolder:
   ```
   my-mod/
     MyMod.Core/
       MyMod.Core.csproj
       Mod.cs
       Properties/
   ```
3. Initialize source control and make the first commit:
   ```powershell
   cd my-mod
   git init
   git add .
   git commit -m "Scaffold MyMod.Core from CS2 code-mod template"
   ```

> **Why move it?** The build targets in the template resolve the game's SDK through the
> `CSII_*` environment variables the toolchain sets, so the project still builds from
> its new home - it is not tied to the `Mods\Local` path. Keeping it in a repo means
> you edit in your repo and the build *deploys* into the game folder, rather than
> editing live game files.

---

## Step 3: Fill in publish metadata

Even though you are not publishing yet, the publish scripts expect certain fields to
exist. Populate them now with placeholders so a future build or publish does not fail on
missing data.

1. Open `Properties/PublishConfiguration.xml` in the project.
2. Fill in placeholder values for the display name, short description, and version
   fields. You will replace these with real content in
   [Build and Publish](build-and-publish.md).
3. Leave the dependency list empty for now - this mod has none yet.

> **Never commit tokens.** `PublishConfiguration.xml` belongs in the repo, but it must
> not contain authentication tokens. The toolchain authenticates through your live game
> session at publish time, so no secret is ever stored in the file.

---

## Step 4: Build once to verify the pipeline

Now confirm the whole chain works: restore, compile, post-process, deploy.

1. Open the project in your editor and let it restore NuGet packages.
2. From the project (or solution) folder, run:
   ```powershell
   dotnet build
   ```
3. A successful build runs the game's post-processor and copies the output to the
   deployed mods location:
   ```
   %AppData%\LocalLow\Colossal Order\Cities Skylines II\Mods\MyMod.Core
   ```

If the build fails, the two most common causes are:

- **The toolchain is not fully installed.** Re-check that every row in
  **Options -> Modding** is green (see [Getting Started](getting-started.md)).
- **A stale shell.** If `dotnet` is not found, open a fresh terminal so it picks up the
  `PATH` set during toolchain install.

---

## Step 5: Load it in-game

1. Launch the game with your developer shortcut (the one carrying `-developerMode`).
2. Enable **MyMod.Core** in the mods list and load a test save.
3. Confirm the mod loaded: after first run, a settings folder and a data folder named
   after your mod ID appear under
   `%AppData%\LocalLow\Colossal Order\Cities Skylines II\`. If they do not, the mod
   failed to load - open the latest `Logs\Log_<date>.txt` and look for the error.

You now have a valid, source-controlled, building, loading mod. It does nothing yet -
that is exactly the right starting point.

---

## Where to go next

- **Add a UI:** [First UI Mod](first-ui-mod.md) scaffolds a Gameface/React bundle and
  wires it into this project's build.
- **Set up your daily loop:** [Build and Publish](build-and-publish.md) covers the
  edit-build-test cycle and shipping to PDX Mods.
- **Write real behavior:** start from [Lifecycle and Initialization](../explanation/mod-lifecycle.md),
  then pick a technique from the [Recipes](../how-to/recipes/README.md).
