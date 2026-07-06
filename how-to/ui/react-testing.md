---
FrontmatterVersion: 1
DocumentType: Guide
Title: "How-to: Lint, build, and smoke-test a React UI bundle"
diataxis: how-to
source_version: "~1.5.x (write-everywhere@13c70eb; date-pinned, static source only)"
last_reverified: "2026-07-04"
canonical_mods:
  - write-everywhere@13c70eb04e6bed152257c516a982455148f591a5
technique_applicability: [ui, tooling]
status: source-verified
Created: 2026-07-03
Owners:
  - codex
Updated: 2026-07-03
---

# Lint, build, and smoke-test a React UI bundle

> Before you publish a mod with a Gameface/React UI, run a gate: lint and typecheck the
> TypeScript, produce the production bundle, load it in-game to confirm it renders, and let the
> C# Release build carry the bundle into the published archive.

## Problem
A UI bundle that compiles in watch mode can still ship broken: a type error hidden behind
incremental builds, a dev-only bundle that never got a production build, a component that works
in the browser preview but is blank in Gameface, or a `dist` folder that never made it into the
uploaded archive. You want a repeatable pre-ship gate that catches each of these.

> **Scope note.** This is a process/tooling topic. The npm script *names* differ by scaffold and
> are not all present in every UI project, so this page gives the gate's shape and cites Write
> Everywhere's real build config where it grounds a step; steps that depend on scripts a scaffold
> may not include are labelled **Needs Verification**.

## Solution
Run the checks in increasing cost order - static analysis first (lint, typecheck), then the
production build, then an in-game smoke test, then multi-language and missing-dependency passes -
and finish by letting the C# Release build produce and place the bundle so publishing cannot ship
a stale one.

## Steps & Code

### 1. Lint and typecheck the TypeScript

Catch errors statically before building. **Needs Verification:** not every scaffold ships `lint`
/ `typecheck` npm scripts - Write Everywhere's `package.json` defines `build` / `dev` but no
`lint` or `typecheck` script:

```json
"scripts": {
  "build:webpack": "webpack --env production",
  "build": "webpack",
  "dev": "webpack --watch"
}
```
Source: `../../../vice-and-order-research/mods/dossiers/write-everywhere/repo/_Frontends/UI/k45-we-vuio/package.json` (@13c70eb04e6bed152257c516a982455148f591a5)

So if your project has them, run `npm run lint` and `npm run typecheck` (locally and in CI). If
it does not, run the tools directly: `tsc --noEmit` for a typecheck (the project ships a
`tsconfig.json` with `strict: true`, so this is meaningful -
`.../tsconfig.json`), and ESLint if configured. The point is a static pass, not a specific
script name.

### 2. Produce the production bundle

Build the real, minified bundle - not the watch build - so you test what players get. In Write
Everywhere that is `build:webpack` (`webpack --env production`); other scaffolds expose an
equivalent `build`. Confirm the build emits output into the expected `dist`/`build` directory
with no errors or unresolved imports.

### 3. Smoke-test the bundle in-game

Load the production bundle in the actual game and confirm your UI mounts and renders. Because
Gameface implements only a subset of the web platform, a component can pass a browser preview and
still fail in-engine - so the authoritative smoke test is **in the game**:

- Launch with the production bundle (no watch/dev server) so you exercise the shipped artifact.
- Open every surface your mod adds (HUD button, tool-option row, panel) and confirm it appears
  and is interactive.
- Watch the UI log for parser "unsupported" warnings and clear them - each is a rule Gameface
  silently dropped (see [The Gameface UI Runtime](../../explanation/gameface-runtime.md)).

**Needs Verification:** opening the built `dist/index.html` directly in a browser as a quick
smoke test (a habit from the archived draft) only exercises plain-browser rendering, *not*
Gameface - use it at most to catch gross build breakage, and treat in-engine as the real check.

### 4. Test multiple languages and a missing-dependency fallback

- Switch the game to at least two languages and confirm labels resolve (no raw locale keys) and
  layouts still fit. Localized strings generally come from the C# side, not the UI.
- Run once **without** any shared UI library your mod soft-depends on (an icon pack, a
  localization helper) and confirm the UI degrades to text/neutral assets instead of erroring -
  see [Handle a missing UI dependency](dependency-handling.md).

The full manual sweep (controller focus, screen-reader labels, dependency-missing behaviour)
lives in the [UI QA checklist](../operations/ui-testing-checklist.md); run it before a release.

### 5. Let the C# Release build carry the bundle

Publishing runs off the C# build, not npm. A correctly wired mod builds and copies the UI bundle
during `dotnet build -c Release`, so:

- Run `dotnet build -c Release` and confirm it (re)produces and places the current bundle.
- Inspect the published archive and confirm it actually contains the UI `dist`/`build` output -
  a missing or stale bundle is the classic "UI didn't update" ship failure.

**Needs Verification:** the exact MSBuild target that runs `npm run build` and copies the output
is mod-specific (see [Set up the React UI development loop](react-development.md), step 3) -
verify it in your `.csproj`. Fold this into the mod-wide [Release Checklist](../operations/release-checklist.md).

## Pitfalls & gotchas

- **Watch build != production build.** The dev/watch bundle can differ from the minified
  production one (dead-code elimination, env flags). Always smoke-test the production build.
- **Browser-green, Gameface-blank.** Passing a browser preview proves nothing about Gameface's
  subset. The in-engine load is the only authoritative smoke test.
- **Stale bundle in the archive.** If the C# build did not rebuild/copy the UI, you publish an
  old bundle. Verify the archive's UI files after `dotnet build -c Release`.
- **Assuming lint/typecheck scripts exist.** Many scaffolds omit them. Run `tsc --noEmit` (and
  ESLint if configured) directly rather than relying on a script name - **Needs Verification**
  per project.
- **Skipping the multi-language / no-dependency passes.** These are the two smoke tests most
  likely to be green on your machine and broken on a player's - do not skip them.

## Variations

- **CI gate.** Run lint + typecheck + production build in CI on every UI change so a broken bundle
  never reaches a release branch; keep the in-engine smoke test as a manual pre-publish step.
- **Bundle-size budget.** Profile the built bundle with the bundler's analyzer and the in-engine
  inspector to keep the UI light (see [Set up the React UI development loop](react-development.md)).

## See also
- How-to: [Set up the React UI development loop](react-development.md) - the build wiring this
  gate depends on; [Add a custom runtime React panel](runtime-ui.md);
  [Inject React into vanilla UI with the module registry](module-registry.md);
  [Handle a missing UI dependency](dependency-handling.md) - the fallback you test in step 4.
- Operations: [UI QA checklist](../operations/ui-testing-checklist.md) - the manual sweep;
  [Release Checklist](../operations/release-checklist.md) - where this gate sits in shipping.
- Explanation: [The Gameface UI Runtime](../../explanation/gameface-runtime.md) - why the in-engine
  smoke test is authoritative; [C#/React communication](../../explanation/ui-cs-communication.md).
- Index: [technique index](../../technique-index.md).

## Sources
- Canonical mods (dossier + repo):
  - `write-everywhere` @13c70eb04e6bed152257c516a982455148f591a5 - `repo/_Frontends/UI/k45-we-vuio/package.json` (build scripts; no lint/typecheck script), `repo/_Frontends/UI/k45-we-vuio/tsconfig.json` (`strict` typecheck config)
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding
