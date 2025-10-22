# Setup And Toolchain

Prime every workstation with the official toolchain before touching Vice & Order modules.

## Launch Parameters

- Add `--developerMode` to unlock the developer menu (`Tab`) and object browser (`Home`); see [Developer mode](https://cs2.paradoxwikis.com/Developer_mode).
- Add `--uiDeveloperMode` to enable the UI debugger at `http://localhost:9444/`.
- Keep both flags in a dedicated development shortcut so QA builds stay clean.

## Required Dependencies

- IDE: Visual Studio 2022 17.8+ or JetBrains Rider 2021.3.3+ for code mods; VS Code or Rider for UI projects.
- Unity Editor 2022.3.7f1 with Entities and Burst; activate the license in Unity Hub once per machine.
- Unity mod project: the toolchain copies a template project to `%LocalAppData%\Colossal Order\Cities Skylines II\Modding`; opening it once seeds Burst post processors.
- .NET SDK 8.0+ (6.0 minimum) provides CLI tooling and MSBuild targets.
- Node.js 18+ (toolchain installs 20.11) for UI builds; restart shells after installation to load environment variables.
- Keep the in-game Modding Options panel green; it offers install/repair/uninstall actions per dependency.

## Project Scaffolding

- Generate code mod templates through the in-game Modding panel or the solution wizard inside Rider/Visual Studio.
- For UI work, run `npx create-csii-ui-mod` inside the code mod folder and align `mod.json.id` with the assembly name.
- Configure `PublishConfiguration.xml` as soon as the mod ID is reserved to avoid last-minute publish blockers.

## Build Pipeline Overview

1. MSBuild compiles the C# project (Roslyn) with source generators enabled.
2. `ModPostProcessor.exe` performs IL weaving, Burst compilation, and asset packaging.
3. Output lands under `%AppData%/LocalLow/Colossal Order/Cities Skylines II/Mods/<ModName>`.
4. Optional `<Target Name="BuildUI" AfterTargets="AfterBuild">` runs `npm run build` to bundle React assets; keep the working directory relative to the `.csproj`.

## Publishing Workflow

- Authenticate through the game client before running `PublishNewMod`, `PublishNewVersion`, or `UpdatePublishedConfiguration`.
- Maintain human-readable release notes alongside module changelogs; match semver tags across code and `mod.json`.
- Verify the bundle with the in-game publisher checklist (dependencies resolved, version metadata, tags).
- Store the signed `PublishConfiguration.xml` in a secure repo location; never commit auth tokens.

## Environment Checklists

- Confirm `ModsSettings`, `ModsData`, and `Logs` folders exist per module namespace using `EnvPath` helpers.
- Set up a development save file with sandbox unlocks; keep a clean baseline for regression testing.
- Install the UI debugger certificate once per machine to avoid browser trust warnings.
