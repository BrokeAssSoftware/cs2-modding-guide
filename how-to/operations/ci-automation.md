---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Automate Build Validation and Checks for a CS2 Mod"
Summary: What you can and cannot put in continuous integration for a CS2 mod - compile and bundle validation, frontmatter/manifest schema linting, dependency-declaration checks, and version-bump automation - with an honest note that CS2 has no official headless test harness.
diataxis: how-to
source_version: "n/a - concept/process page, no pinned source"
last_reverified: "2026-07-04"
status: needs-verification
Created: 2026-07-03
Updated: 2026-07-03
Owners:
  - codex
References:
  - Label: Release Checklist (the manual gate CI supports, not replaces)
    Path: ./release-checklist.md
  - Label: Build, Test, and Publish (the build and publish commands)
    Path: ../../tutorials/build-and-publish.md
  - Label: Lint, build, and smoke-test a React UI bundle (the UI half)
    Path: ../ui/react-testing.md
  - Label: Dependency Strategy (declared vs actual dependencies)
    Path: ../../explanation/dependency-strategy.md
  - Label: Technique Index
    Path: ../../technique-index.md
---

# Automate Build Validation and Checks for a CS2 Mod

Continuous integration catches the mistakes a checklist relies on a human to notice: a
build that no longer compiles, a manifest missing a field, a dependency declared in code
but not in the publish config. This page is what you can realistically automate for a CS2
mod on a plain CI runner (GitHub Actions, or a pre-commit / pre-push hook), and - just as
important - what you cannot.

> **CS2 has no official CI or headless test harness.** The game does not run on a headless
> build agent, and publishing authenticates through a live, signed-in game session (see
> [Build, Test, and Publish](../../tutorials/build-and-publish.md)). So CI validates the
> *artifacts and metadata* of a mod; it cannot run the mod inside the simulation. Anything
> that needs the running game stays in the manual [Release Checklist](./release-checklist.md)
> and [UI QA Checklist](./ui-testing-checklist.md). CI **supports** those gates; it does
> not replace them.

---

## 1. Build validation (the reliable core)

The C# and UI builds are standard toolchains and run fine on a runner. Gate every pull
request on them:

- **Compile in both configurations.** Run `dotnet build -c Debug` and
  `dotnet build -c Release`. Release matters because conditional compilation (`#if DEBUG`)
  means the shipped binary is not the one you develop against - a Release-only compile
  error is invisible until you build it.
- **Build the UI bundle.** If the mod ships UI, run `npm ci` then `npm run build`, and run
  the lint/typecheck steps described in
  [Lint, build, and smoke-test a React UI bundle](../ui/react-testing.md). That page is the
  automatable half of UI validation; the in-engine half stays manual.
- **Fail the build on warnings that matter.** Treat new compiler warnings as signal, not
  noise, so they do not accumulate until one hides a real problem.

A runner needs the CS2 modding SDK / game assemblies to resolve references. Vendoring or
restoring those on the agent is the one non-trivial setup step; without them the compile
cannot see the game's types. `Needs Verification`: the exact way to provision the game/SDK
assemblies on a runner depends on your toolchain and licensing and is not something an
official pipeline provides - confirm against your template's reference-resolution setup.

## 2. Manifest and frontmatter schema linting

Metadata errors load fine on your machine and break for players (or fail a publish). Lint
the structured files on every change:

- **Validate manifests against a schema.** Check `PublishConfiguration.xml`, any `mod.json`
  / module manifest, and JSON/YAML config packs against a schema so a missing or misspelled
  required field fails CI instead of a player's load.
- **Lint documentation frontmatter.** If your repo carries structured docs (like this
  handbook's YAML frontmatter), assert required keys are present and well-formed - a schema
  or a small script over the frontmatter block catches a dropped `version` or malformed
  date early.
- **Validate localization files.** Parse each locale file and assert it is well-formed and
  that keys line up across locales, so a broken translation file does not ship.

## 3. Dependency-declaration check

The classic "works on my machine" failure is a dependency you reference in code but forgot
to declare. Automate the comparison:

- **Diff declared vs. actual.** Compare the dependencies listed in `PublishConfiguration.xml`
  against the mod's actual project/assembly references, and fail if code references a mod
  that the publish config does not declare (by both name and mod ID). This is the automated
  form of the manual re-verify-dependencies step in the
  [Release Checklist](./release-checklist.md); the reasoning behind it is in
  [Dependency Strategy](../../explanation/dependency-strategy.md).

## 4. Version-bump and release automation

Version, changelog, and tag drift apart the moment they are maintained by hand. Automate
the linkage:

- **Assert version consistency.** Fail CI if the version in `PublishConfiguration.xml`, the
  changelog's top entry, and the intended git tag disagree - the
  [Release Checklist](./release-checklist.md) requires them to match, so let CI enforce it.
- **Script the bump.** A small release script that increments the version, prepends a
  changelog entry, commits, and creates the git tag removes the step most often forgotten
  under release pressure.
- **Do not automate the publish itself.** The publish command authenticates through the
  running game session; it is a human action from your IDE, not a CI job. CI can prepare and
  validate the release; a person publishes it.

## What CI cannot do for a CS2 mod (be honest)

These belong on a manual checklist, not a pipeline. Treat any pitch to automate them as
`Needs Verification` until CS2 provides the harness that would make them possible:

- **Load a save and assert in-game behaviour.** There is no supported headless game runner,
  so "load a scripted city on the agent and assert the log error count" is not achievable
  with the shipped tooling today. This is the speculative idea to be most skeptical of. In
  reality this stays a human step: load [regression saves](./logging-and-debugging.md) and
  read `Player.log` (Release Checklist steps 2-3).
- **In-engine UI / accessibility checks.** CS2 UI runs in the Gameface runtime; browser CI
  does not reproduce it (see [UI QA Checklist](./ui-testing-checklist.md)).
- **Publish and post-publish verification.** Publishing needs a live session; a bot cannot
  do it.

---

## Notes

- **CI raises the floor; the checklists set the bar.** Automating build, schema, dependency,
  and version checks frees the manual [Release Checklist](./release-checklist.md) and
  [UI QA Checklist](./ui-testing-checklist.md) to focus on what only a human driving the
  running game can verify. Neither replaces the other.
- **Start small.** Even a pre-push hook that just runs `dotnet build -c Release` and the
  version-consistency check catches the two most common release failures.

## Where to go next

- The manual gate CI supports: [Release Checklist](./release-checklist.md) and
  [UI QA Checklist](./ui-testing-checklist.md).
- The build and publish commands CI wraps: [Build, Test, and Publish](../../tutorials/build-and-publish.md).
- Why dependencies must be declared: [Dependency Strategy](../../explanation/dependency-strategy.md).
- Pick your next technique: [Technique Index](../../technique-index.md).
