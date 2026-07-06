---
FrontmatterVersion: 1
DocumentType: Guide
Title: "How-to: Set up the React UI development loop"
diataxis: how-to
source_version: "n/a - toolchain/tutorial, wiki-sourced (Modding_Toolchain stale-verified 1.1.12f1)"
last_reverified: "2026-07-04"
canonical_mods:
  - write-everywhere@13c70eb04e6bed152257c516a982455148f591a5
technique_applicability: [ui, tooling]
status: needs-verification
Created: 2026-07-03
Updated: 2026-07-03
Owners:
  - codex
---

# Set up the React UI development loop

> Get a fast iteration loop for a CS2 mod's React UI: scaffold the UI project, wire its
> build into the C# build so `dotnet build` also produces the bundle, run a watch build for
> hot iteration, scope your styles, and keep components light so per-frame updates stay cheap.

## Problem
You are building bespoke React UI for a CS2 mod and want to iterate without a full rebuild
every change: edit a component, see it in-game, keep the C# side and the UI bundle in sync,
and avoid the performance traps that make a Gameface panel stutter. The exact npm scripts and
dev-server details differ between UI scaffolds, so you need the *shape* of the loop and the
few things that are true across toolchains.

> **Scope note.** The CS2 UI is officially **React, with SCSS for styles and TypeScript**,
> per the [UI Modding](https://cs2.paradoxwikis.com/UI_Modding) wiki page (verified for
> 1.5.7f1). The official debugging tool is the **Chrome Debugger** at
> `http://localhost:9444`, reached by launching the game with `--uiDeveloperMode` (both
> documented on [Modding_Toolchain](https://cs2.paradoxwikis.com/Modding_Toolchain)) - so
> the port and the developer-mode flag are **not** unverified guesses. The Chromium-based
> runtime the UI renders into is commonly called "Gameface" (or "Coherent Gameface") in the
> community, but the wiki names only the "Chrome Debugger", so treat "Gameface" as community
> terminology. What genuinely varies by scaffold (exact npm script names, bundler choice)
> stays labelled **Needs Verification**. Where a claim is grounded in a real mod's pinned UI
> project it is cited to Write Everywhere's UI build config.

## Solution
Treat the UI as an npm sub-project inside your mod that the C# build drives. A one-time
scaffold generates the project; MSBuild targets run the UI build and copy the bundle into the
mod's deploy directory; a watch build gives you the fast loop; and you keep the React side thin
because everything it shows is serialized across the C#/React binding boundary.

## Steps & Code

### 1. Scaffold the UI project inside the mod

The community toolchain scaffolds a UI project (entry point, `mod.json`, build config, and the
`cs2/*` type stubs) with a generator. **Needs Verification:** the exact scaffold command and
the toolchain it emits differ by template - historically `npx create-csii-ui-mod` - so confirm
against the generator version you install. Whatever it emits, make sure `mod.json.id` lines up
with the C# assembly so the bundle is served under the right host.

### 2. Know your npm scripts (they vary by scaffold)

The generated `package.json` defines the build and watch scripts. There is no single canonical
set - Write Everywhere, for example, drives a **webpack** build directly rather than the default
template's bundler:

```json
"scripts": {
  "build:webpack": "webpack --env production",
  "build": "webpack",
  "dev": "webpack --watch",
  "update": "npx create-csii-ui-mod update",
  "clean": "npx create-csii-ui-mod clean"
}
```
Source: `../../../vice-and-order-research/mods/dossiers/write-everywhere/repo/_Frontends/UI/k45-we-vuio/package.json` (@13c70eb04e6bed152257c516a982455148f591a5)

The durable takeaway: there is a **production build** (`build` / `build:webpack`), a **watch
build** for iteration (`dev`), and template-maintenance scripts (`update` / `clean`). The
literal names (`build` vs `build:webpack` vs a `vite build`) are **Needs Verification** for your
scaffold - read your own `package.json`.

### 3. Wire the UI build into the C# build

You want `dotnet build` to produce the UI bundle and drop it where the game serves mod assets,
so you never ship a stale bundle. The general pattern is an MSBuild target that runs the UI
build (`npm run build`) and copies the output into the deploy directory alongside the compiled
assembly. **Needs Verification:** the exact target names, `Exec` invocation, and copy paths are
scaffold- and mod-specific; confirm them in your `.csproj`. The invariant is: the C# build owns
producing and placing the bundle, so a plain `dotnet build -c Release` is enough to publish a
current UI (this is why the release gate only runs the C# build - see
[Lint, build, and smoke-test a React UI bundle](react-testing.md)).

### 4. Run a watch build for the fast loop

For iteration, run the watch script (`npm run dev` -> `webpack --watch` in Write Everywhere's
config, above) so the bundle rebuilds on save. Launch the game with `--uiDeveloperMode` and
open the **Chrome Debugger** at `http://localhost:9444` to inspect component state and
computed styles live; both the flag and the port are officially documented on
[Modding_Toolchain](https://cs2.paradoxwikis.com/Modding_Toolchain). (The wiki spells the
flag with two dashes, `--uiDeveloperMode`; some community shortcuts use a single dash,
`-uiDeveloperMode` - **Needs Verification** against the live client.)

### 5. Scope your styles

Keep component styles from leaking into vanilla UI. Two common approaches:

- **Scoped stylesheets imported per component.** Write Everywhere ships plain SCSS imported
  into the components that use it (compiled by `sass-loader` in its webpack config):

  ```ts
  import "style/mainUi/mainUi.scss";
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/write-everywhere/repo/_Frontends/UI/k45-we-vuio/src/mainUI/WEMainUI.tsx#L6` (@13c70eb04e6bed152257c516a982455148f591a5); SCSS/CSS loaders (`sass-loader`, `css-loader`, `style-loader`) declared in `.../package.json` devDependencies.

- **CSS Modules** (`import styles from "./x.module.scss"`, referencing `styles.foo`) give per-
  component class-name hashing. **Needs Verification:** whether your scaffold enables CSS Modules
  depends on its loader config - Write Everywhere uses *global* SCSS imports, not `.module.scss`.
  Pick one convention and be consistent; either way, remember Gameface implements only a curated
  CSS subset (see [The Gameface UI Runtime](../../explanation/gameface-runtime.md)), so verify
  layout in-engine rather than in a browser.

### 6. Keep components light

Every value the UI shows is serialized across the binding boundary, and Gameface is a game
runtime, so runtime cost matters:

- **Memoize** derived values and callbacks (`useMemo` / `useCallback`) so re-renders do not
  recompute or re-allocate needlessly. (General React practice - **Needs Verification** against
  your own profiling.)
- **Publish small, flat, display-ready data** from C#; do not push simulation graphs the UI has
  to reshape. This is a property of the JSON binding boundary - see
  [C#/React communication](../../explanation/ui-cs-communication.md).
- **Update bindings only when values change**, not every tick with an unchanged value.

## Pitfalls & gotchas

- **The bundle can go stale silently.** If the C# build does not run the UI build + copy, you
  ship yesterday's UI. Wire the build so `dotnet build` regenerates and places the bundle
  (step 3), and confirm the deployed archive contains the current `dist`/`build` output.
- **Browser parity is a trap.** A rule or JS API that works in `npm run dev`'s browser preview
  may be dropped by Gameface. Verify in the in-engine inspector and read the UI log for
  "unsupported" parser warnings. See [Gameface runtime](../../explanation/gameface-runtime.md).
- **`mod.json.id` mismatch breaks asset resolution.** Keep the UI project's id aligned with the
  C# assembly so the bundle serves under the expected host location.
- **Dev-loop specifics can drift.** The `--uiDeveloperMode` flag and the
  `http://localhost:9444` inspector port are officially documented, but on a version-stale
  wiki page, and the exact flag dash-count is **Needs Verification** against the live client
  - re-check both rather than assuming they never change.

## Variations

- **Different bundlers.** Templates ship different bundlers (Write Everywhere runs webpack; other
  scaffolds default to another). The loop shape is identical - a production build plus a watch
  build - only the tool differs.
- **Type-only iteration.** For pure logic/typing changes, a `tsc --noEmit` typecheck is faster
  than a full bundle - fold it into the pre-ship gate in
  [Lint, build, and smoke-test a React UI bundle](react-testing.md).

## See also
- How-to: [Add a custom runtime React panel](runtime-ui.md) - what you are iterating on;
  [Inject React into vanilla UI with the module registry](module-registry.md) - how it mounts;
  [Lint, build, and smoke-test a React UI bundle](react-testing.md) - the pre-ship gate.
- Operations: [UI QA checklist](../operations/ui-testing-checklist.md).
- Explanation: [The Gameface UI Runtime](../../explanation/gameface-runtime.md) - the web-subset
  constraints your components run under; [C#/React communication](../../explanation/ui-cs-communication.md)
  - why payloads must stay small.
- Index: [technique index](../../technique-index.md) - family P (`UISystemBase` / React binding).

## Sources
- Canonical mods (dossier + repo):
  - `write-everywhere` @13c70eb04e6bed152257c516a982455148f591a5 - `repo/_Frontends/UI/k45-we-vuio/package.json` (build/watch scripts, loaders), `repo/_Frontends/UI/k45-we-vuio/src/mainUI/WEMainUI.tsx` (scoped SCSS import)
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/UI_Modding (React/SCSS/TypeScript stack; Chrome Debugger, verified 1.5.7f1),
  https://cs2.paradoxwikis.com/Modding_Toolchain (--uiDeveloperMode, http://localhost:9444; stale-verified 1.1.12f1),
  https://cs2.paradoxwikis.com/Modding
