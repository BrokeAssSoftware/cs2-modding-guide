---
FrontmatterVersion: 1
DocumentType: Guide
Title: The Gameface UI Runtime
Summary: What Coherent Gameface (Cohtml) is, why CS2 mod UI runs inside a curated web-platform subset (flex-only layout, restricted CSS, a pared-down JS runtime), and where the C#/React boundary sits - a concept page for anyone building React UI for a Cities Skylines II mod.
diataxis: explanation
source_version: "~1.5.x (better-bulldozer@4408466; date-pinned, static source only)"
last_reverified: "2026-07-04"
canonical_mods:
  - better-bulldozer@4408466f226db811159d92859479ae1e1c28ba06
status: source-verified
Created: 2026-07-03
Updated: 2026-07-03
Owners:
  - codex
References:
  - Label: C#/React communication (the binding boundary)
    Path: ./ui-cs-communication.md
  - Label: Options attributes (declarative settings UI, no React)
    Path: ../reference/options-attributes.md
  - Label: Technique Index
    Path: ../technique-index.md
  - Label: Coherent Gameface supported-features tables (authoritative)
    Path: https://docs.coherent-labs.com/unity-gameface/content_development/supported_features_tables/
---

# The Gameface UI Runtime

Cities: Skylines II does not render its interface with Unity's UI systems. It renders with
**Coherent Gameface** (the *Cohtml* engine), an embedded HTML/CSS/JS runtime. Every panel you
see in-game - and every panel a mod adds - is a web view: React components rendered into a
Chromium-derived engine that implements a *subset* of the web platform.

This page explains what that means for you as a mod author: why your UI is "web but not quite",
what the runtime does and does not support, and where the boundary between your C# simulation
code and your React UI actually is. It is a concept page - the "why the UI layer behaves this
way" - not a step-by-step recipe. For the mechanism that carries data across the boundary, see
[C#/React communication](./ui-cs-communication.md).

## What Gameface is

Gameface is a commercial UI middleware from Coherent Labs that embeds a browser-like engine
into a game. CS2 uses it to author its HUD, menus, tool options, and info panels as HTML/CSS
with React on top. For a modder the practical consequences are:

- **You write React/TypeScript, not Unity UI.** Mod UI is a JavaScript bundle that mounts
  React components into a Gameface *view*, the same way a web app mounts into a browser tab.
- **It is a subset, not a full browser.** Gameface implements a curated list of HTML elements,
  CSS properties, and JS APIs tuned for game UI performance. Anything outside that list either
  silently no-ops or logs a parser warning and is dropped. This is the single most important
  mental model: **assume nothing works until you have seen it work in-engine.**
- **Assets load over a custom scheme.** UI assets (JS, CSS, images, icons) are served through
  an engine-internal URL scheme (`coui://`) rather than from disk paths or `http(s)`. Your
  build tooling emits bundles into the mod's deploy directory so the engine can resolve them.

Because we do not have Gameface's engine source, the *exact* capability set is defined by
Coherent's own [supported-features tables](https://docs.coherent-labs.com/unity-gameface/content_development/supported_features_tables/),
which are versioned to the Gameface release CS2 ships. Treat that page as the authority and
this one as orientation. Every engine-version-specific claim below is labelled
**Needs Verification** because it depends on the Gameface build in the current patch and on
in-engine observation, neither of which we can source-verify from mod code.

## Why it is "web but not quite": the three constraints

### 1. Layout is flex-first

Flexbox is the supported, preferred layout model. The classic web alternatives are not safe to
rely on:

- **CSS Grid support is limited or absent** in the Gameface builds modders have historically
  targeted; `display: grid` may parse but not lay out as a browser would. **Needs
  Verification** against the current engine version.
- **Inline-flow display modes** (`inline-block`, `inline-flex`) and some spacing shorthands
  (`gap`) have been reported unreliable in older engine builds. **Needs Verification.**

The durable takeaway is not the specific list - it is the *posture*: **design flex-first**, keep
layouts explicit, and verify any layout feature in the in-engine inspector before depending on
it. Build spacing with explicit margins or utility patterns rather than assuming a shorthand
resolves.

### 2. CSS is a curated subset

A number of CSS features that "just work" in a browser have historically been dropped or
ignored by Gameface. Community-reported examples (all **Needs Verification** against the current
engine) include math helpers (`min()`/`max()`/`clamp()`), `backdrop-filter`, `object-fit`,
`vertical-align`, `border-radius: inherit`, `list-style`, and advanced selectors
(`:focus-visible`, `:is()`, `:not()`, `:disabled`). When a rule is unsupported, the engine
typically discards it and writes a parser warning to the UI log rather than throwing.

The way to work with this is diagnostic, not memorised:

- **Read the UI log.** Gameface prints explicit "unsupported" entries when it discards a rule.
  The UI log lives under the game's user data (`Logs/` in the CS2 user directory). Watch it
  while iterating and treat every parser warning as a rule that is silently not applying.
- **Toggle helper classes from React instead of leaning on pseudo-class logic.** If
  `:disabled` or `:focus-visible` is unreliable, drive the same state from a React-controlled
  class name, which always works because it is plain DOM.
- **Prefer explicit values over computed shorthands.** Where a browser lets you write terse
  computed CSS, Gameface rewards explicitness.

### 3. The JavaScript runtime is pared down

Gameface ships a reduced JS runtime. Core ES is present, but some browser APIs are missing or
stubbed. Community-reported gaps (**Needs Verification** against the current engine) include the
`Intl` internationalisation APIs (`Intl.DateTimeFormat`) and the Fetch/network APIs inside the
sandbox. The consequences that matter:

- **Do not format locale-sensitive values in JS.** If `Intl` is unavailable, format
  dates/numbers/currency on the C# side and pass finished strings across the binding, or format
  manually. (This is also why localization is generally handled through the C# dictionary-source
  system rather than in-UI.)
- **Do not make network calls from UI.** There is no browser to reach the internet through the
  game's sandbox. Anything the UI needs from "the outside" comes from C# through a binding.
- **Compile through TypeScript targeting a conservative output.** Modern language syntax
  (optional chaining and similar) is fine to *write* because it transpiles away; the risk is
  runtime *APIs*, not language syntax.

## The C#/React boundary

The most important architectural fact about CS2 mod UI is that **the UI has no direct access to
the simulation.** React runs in the Gameface sandbox; the game world (DOTS/ECS) runs in C#. The
two never share memory. They communicate only through an explicit, named **binding** channel,
and everything that crosses it is serialized (marshalled as JSON).

This is source-verifiable from mod code even though the engine internals are not. A mod's UI
lives in a C# system deriving from `UISystemBase` (the game's base for UI-facing systems), which
registers named bindings that the React side reads and calls. Better Bulldozer's UI system
imports `Colossal.UI.Binding` and holds its state as typed `ValueBinding<T>` fields that it
pushes to React, plus `TriggerBinding` entries that let React invoke C# callbacks
([`better-bulldozer` `repo/BetterBulldozer/Systems/BetterBulldozerUISystem.cs#L15,#L49-L60,#L302-L326`](../../vice-and-order-research/mods/dossiers/better-bulldozer/repo/BetterBulldozer/Systems/BetterBulldozerUISystem.cs), commit `4408466f`). The values are written out through an
`IJsonWriter`, which is the concrete evidence that the boundary is a JSON marshalling boundary,
not a shared-object one
([`better-bulldozer` `repo/BetterBulldozer/Extensions/GenericUIWriter.cs#L15-L17`](../../vice-and-order-research/mods/dossiers/better-bulldozer/repo/BetterBulldozer/Extensions/GenericUIWriter.cs)).

The design rules that follow from a JSON marshalling boundary:

- **Keep payloads small and flat.** Every update is serialized and deserialized; large or deep
  objects cost more per tick. Send primitives and small records, not whole simulation graphs.
- **The UI is a projection, not the source of truth.** C# owns the state; React renders a copy
  pushed to it and asks C# to change things by name.
- **Two directions, two mechanisms.** C#-to-React is a *value* you publish and update;
  React-to-C# is a *trigger* (an RPC by name). That split - and the exact `ValueBinding` /
  `TriggerBinding` / `CreateBinding` / `CreateTrigger` API - is the subject of its own page:
  [C#/React communication](./ui-cs-communication.md).

## When you do not need Gameface at all

Not every mod needs a React bundle. If all you need is a settings/options page - sliders,
toggles, dropdowns, buttons - the game gives you a **declarative** path: subclass `ModSetting`
and decorate properties with `[SettingsUI*]` attributes, and the game renders the controls in
its own Options panel for you. No React, no CSS, no Gameface bundle to maintain. Reach for a
custom Gameface/React module only when you need bespoke in-world UI (tool option rows, info
panels, overlays) that the settings system cannot express. See
[options attributes](../reference/options-attributes.md) for the declarative route.

## Common pitfalls

- **Assuming browser parity.** A rule that works in Chrome may be silently dropped by Gameface.
  Verify in the in-engine inspector and watch the UI log for "unsupported" entries.
- **Relying on `Intl` or Fetch.** Format locale-sensitive text and fetch all data on the C#
  side; pass results across the binding.
- **Treating the UI as authoritative.** The simulation state lives in C#/ECS; the UI is a
  serialized projection. Never store gameplay truth only in React.
- **Trusting an unverified feature list.** The supported-features set is engine-version-specific.
  Confirm against Coherent's tables for the Gameface build in the current CS2 patch, and label
  anything you have not observed in-engine **Needs Verification**.

## See also

- [C#/React communication](./ui-cs-communication.md) - the `ValueBinding` / `TriggerBinding`
  model that carries data across the boundary described here.
- [Options attributes](../reference/options-attributes.md) - the no-React declarative route for
  a settings page.
- [Technique Index](../technique-index.md) - coverage ledger, including the UI binding family
  (P - `UISystemBase` / React binding).
- Coherent Gameface [supported-features tables](https://docs.coherent-labs.com/unity-gameface/content_development/supported_features_tables/) -
  the authoritative, version-specific capability reference.
