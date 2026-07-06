---
FrontmatterVersion: 1
DocumentType: Guide
Title: "How-to: Add a custom runtime React panel (vs Options UI)"
diataxis: how-to
source_version: "~1.5.x (write-everywhere@13c70eb; date-pinned, static source only)"
last_reverified: "2026-07-04"
canonical_mods:
  - write-everywhere@13c70eb04e6bed152257c516a982455148f591a5
technique_applicability: [ui]
status: source-verified
Created: 2026-07-03
Updated: 2026-07-03
Owners:
  - codex
---

# Add a custom runtime React panel (vs Options UI)

> Decide whether your mod needs a bespoke in-world React panel/dashboard rendered by
> Gameface, or whether the game's declarative **Options** page is enough - then, if you
> do need a runtime panel, mount it into the live HUD through the UI module registry and
> back it with C# data bindings.

## Problem
You want your mod to show information or controls *during play* - a status dashboard, a
tool option row, an overlay, a toggle button on the HUD - not just a settings screen.
Cities: Skylines II gives you two very different routes to on-screen UI, and picking the
wrong one costs you a maintenance burden (a full React/Gameface bundle) you did not need,
or boxes you into a settings form that cannot express what you want.

## Solution
Choose by *what the UI has to do*, not by habit:

| Need | Route | What it costs |
| --- | --- | --- |
| Sliders, toggles, dropdowns, buttons, read-only text - a **configuration** surface | Declarative **Options UI**: subclass `ModSetting`, decorate with `[SettingsUI*]`, call `RegisterInOptionsUI()` | No React, no CSS, no bundle. See [Build a mod Options-menu UI](options-ui.md). |
| Bespoke **in-world** UI: live dashboards, tool-option rows, overlays, HUD buttons, custom panels reading ECS/service state each frame | Custom **Gameface/React module** injected via the UI module registry, fed by C# bindings | A React/TypeScript bundle you build, ship, and keep working against the Gameface subset. |

The rule of thumb: **reach for a runtime React panel only when the settings system cannot
express the UI.** A read-only status summary, a directory picker, or a "reset" button
belongs in the Options page (the settings attribute family - see
[options attributes](../../reference/options-attributes.md) - covers read-only text and
file/dir pickers too). A live crime/economy dashboard, a tool-option strip, or a HUD
overlay is what runtime React is for.

Before committing to React, read [The Gameface UI Runtime](../../explanation/gameface-runtime.md):
CS2 UI is a *subset* of the web platform (flex-first layout, curated CSS, a pared-down JS
runtime), so a runtime panel is more work to get right than a web widget of the same shape.

## Steps & Code

### 1. Confirm you actually need a runtime panel

If every control you need is a setting, stop here and use
[Build a mod Options-menu UI](options-ui.md). The Options route needs no bundle and no
Gameface knowledge. Only continue if you need in-world UI the settings system cannot draw.

### 2. Register your React entry point with the module registry

A runtime UI module exports a `ModRegistrar` as its default. The game hands it a
`moduleRegistry`, and you mount your components onto named targets. Adding a HUD button is
a single `append` onto a mount point - Write Everywhere adds its top-left toolbar button
exactly this way:

```ts
const register: ModRegistrar = (moduleRegistry) => {
    // ...extend calls for tool-options and panels...
    moduleRegistry.append('GameTopLeft', WEButton);   // custom HUD button, live in-game
}
export default register;
```
Source: `../../../vice-and-order-research/mods/dossiers/write-everywhere/repo/_Frontends/UI/k45-we-vuio/src/index.tsx#L9-L16,#L47` (@13c70eb04e6bed152257c516a982455148f591a5)

The mechanics of `append` / `extend` / `override` - how you place a whole panel, wrap a
vanilla component, or replace one - are their own task: see
[Inject React into vanilla UI with the module registry](module-registry.md).

### 3. Back the panel with C# state, not UI-local state

A runtime panel is a **projection** of simulation state, not the owner of it. The panel
reads named values your C# `UISystemBase` publishes and fires named triggers to ask C# to
change things. That binding channel is the subject of
[C#/React communication](../../explanation/ui-cs-communication.md) - build the panel's data
flow on `ValueBinding<T>` (C# -> React) and `TriggerBinding` (React -> C#), and keep the
payloads small and flat.

### 4. Develop, style, and test the bundle

Once the panel exists, iterate on it with the React dev loop
([Set up the React UI development loop](react-development.md)) and gate it before shipping
with [Lint, build, and smoke-test a React UI bundle](react-testing.md) plus the
[UI QA checklist](../operations/ui-testing-checklist.md).

## Pitfalls & gotchas

- **Do not build a React panel for a settings form.** The Options route is far cheaper and
  is the right tool for configuration. A runtime bundle is only justified by in-world UI the
  settings system cannot express.
- **A runtime panel lives in the Gameface subset.** Layout is flex-first, CSS is curated,
  and the JS runtime lacks `Intl`/Fetch. Assume nothing works until you have seen it in the
  in-engine inspector, and watch the UI log for dropped-rule warnings. See
  [The Gameface UI Runtime](../../explanation/gameface-runtime.md).
- **The UI is never the source of truth.** Store gameplay state in C#/ECS and publish a copy;
  a panel that keeps its own authoritative state desyncs from the simulation.
- **The exact set of valid mount targets is version-specific.** `GameTopLeft` and the other
  hook targets Write Everywhere uses are verified from its source at the pinned commit, but the
  full catalogue of mount points and vanilla module paths is discovered against the running
  game, not enumerable from one mod - treat any target not shown here as **Needs Verification**.

## Variations

- **Read-only summary without React.** If all you need is a live-ish text readout (status,
  a computed total, a diagnostic line), a read-only text row on the Options page can serve
  instead of a Gameface panel - see the settings attribute family in
  [options attributes](../../reference/options-attributes.md) and the read-only `string`
  property pattern in [Build a mod Options-menu UI](options-ui.md).
- **Tool-option row instead of a free-floating window.** Rather than a standalone panel, you
  can `extend` the vanilla tool-options component so your controls appear inside the existing
  tool UI (Write Everywhere does this for its world-picker tool) - see
  [module registry](module-registry.md).
- **Degrade when a UI dependency is missing.** If the panel uses a shared icon or
  localization library, probe for it and fall back to plain text -
  [Handle a missing UI dependency](dependency-handling.md).

## See also
- How-to: [Build a mod Options-menu UI](options-ui.md) - the no-React route and when it is
  enough; [Inject React into vanilla UI with the module registry](module-registry.md) -
  how the panel actually mounts; [Set up the React UI development loop](react-development.md);
  [Lint, build, and smoke-test a React UI bundle](react-testing.md).
- Operations: [UI QA checklist](../operations/ui-testing-checklist.md).
- Explanation: [The Gameface UI Runtime](../../explanation/gameface-runtime.md) - why the
  panel runs in a web-platform subset; [C#/React communication](../../explanation/ui-cs-communication.md)
  - the binding channel that feeds it.
- Index: [technique index](../../technique-index.md) - family P (`UISystemBase` / React binding).

## Sources
- Canonical mods (dossier + repo):
  - `write-everywhere` @13c70eb04e6bed152257c516a982455148f591a5 - `repo/_Frontends/UI/k45-we-vuio/src/index.tsx` (runtime panel/button registration)
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding
