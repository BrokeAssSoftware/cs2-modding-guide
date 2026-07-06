---
FrontmatterVersion: 1
DocumentType: Guide
Title: "UI QA Checklist for a CS2 Mod"
Summary: A manual QA gate for a CS2 mod's UI - render in multiple languages, verify settings persistence and key rebinds, exercise missing-dependency fallbacks, hide debug-only sections in Release, and (for any custom modal/panel) check controller focus and screen-reader labels.
diataxis: how-to
source_version: "n/a - concept/process page, no pinned source"
last_reverified: "2026-07-04"
status: needs-verification
Created: 2026-07-03
Updated: 2026-07-03
Owners:
  - codex
References:
  - Label: Lint, build, and smoke-test a React UI bundle (the automated gate)
    Path: ../ui/react-testing.md
  - Label: Release Checklist (where UI QA sits in shipping)
    Path: ./release-checklist.md
  - Label: Handle a missing UI dependency (the fallback this checklist exercises)
    Path: ../ui/dependency-handling.md
  - Label: The Gameface UI Runtime (why in-engine is authoritative)
    Path: ../../explanation/gameface-runtime.md
  - Label: Technique Index
    Path: ../../technique-index.md
---

# UI QA Checklist for a CS2 Mod

Run this manual sweep before shipping any UI change. It complements the automated gate in
[Lint, build, and smoke-test a React UI bundle](../ui/react-testing.md) (which handles lint /
typecheck / build): this page is the *in-game* pass that only a human driving the UI can do.
Work top to bottom; do not skip a step because "that part didn't change."

Every step is exercised **in the running game**, not a browser preview - CS2 UI runs in the
Gameface web-platform subset, so browser behaviour does not guarantee in-engine behaviour (see
[The Gameface UI Runtime](../../explanation/gameface-runtime.md)).

---

## Core UI

1. **Multi-language render.** Build in Debug and confirm the mod's Options page (and any custom
   UI) renders correctly in **at least two languages** - no raw locale keys, no clipped or
   overflowing labels, no broken layout when strings get longer.

2. **Settings persistence.** Change each setting, restart the game, and confirm every value
   persisted (and that a "reset to defaults" action restores the intended defaults).

3. **Key rebinds and conflicts.** Rebind every key the mod declares and confirm a conflicting
   binding produces a **visible warning** rather than silently double-binding.

4. **Missing-dependency fallback.** Launch **without** each shared UI dependency the mod
   soft-depends on (an icon library, a localization helper) and confirm graceful degradation -
   text labels instead of missing icons, bundled strings instead of a helper's catalogue - with
   no per-frame errors. See [Handle a missing UI dependency](../ui/dependency-handling.md).

5. **Debug-only sections hidden in Release.** With developer mode enabled, verify any debug-only
   UI section appears; then confirm those same sections are **hidden in a Release build**, so
   players never see debug controls.

## Custom modal / panel (only if your mod adds one)

Steps 6-9 apply when your mod adds bespoke in-world UI beyond the Options page - a modal launched
from a settings row, a HUD panel, a multi-section dashboard. If your mod is settings-only, skip
this section. Run each check with **both** a controller/gamepad and the keyboard.

6. **Controller focus round-trip.** From the control that launches your modal/panel (for example
   an Options-page row or a HUD button), open and then close the modal with a controller. Confirm
   focus **returns to the launching control** (or the list it lives in) after closing, and that
   the launch control advertises its focus state (a visible focus ring / highlight) so a
   controller user can find it.

7. **Screen-reader labels on the entry points.** Turn on the screen-reader bridge and confirm the
   launch control, the modal's **close button**, and its **primary action** all announce
   meaningful text - real labels / ARIA titles, not empty or generic strings.

8. **Navigation sweep of a multi-section panel.** For a panel with its own navigation (a nav rail,
   tabs, a list of sections), sweep it with gamepad/keyboard and verify **sequential focus order**,
   sensible **wrap-around** at the ends, and that a **"back" affordance** returns focus to wherever
   the user entered from (the launching control or the previous view).

9. **Screen-reader pass on the panel body.** Run the screen reader over the panel's **section
   headers, navigation items, action buttons, and any stateful controls** (buttons whose meaning
   changes - e.g. a "download" vs "installed" or "upgrade" state) and confirm each announces an
   **actionable** description, not just its raw text.

---

## Notes and caveats

- **Accessibility behaviour in Gameface is engine-version-specific.** How screen-reader
  announcements, ARIA titles, `:focus`/`:focus-visible` styling, and controller focus traversal
  actually behave inside a Gameface view depends on the engine build the current patch ships and
  can only be confirmed by driving it. Treat the exact a11y/focus outcomes of steps 6-9 as
  **Needs Verification (in-game)** and re-check them each game update.
- **Watch the UI log while you test.** Gameface writes "unsupported" parser warnings when it
  drops a CSS rule; a control that looks wrong is often a silently-dropped style, not a logic bug
  (see [The Gameface UI Runtime](../../explanation/gameface-runtime.md)).
- **This is a gate, not a suggestion.** The multi-language, missing-dependency, and focus
  round-trip checks are the ones most likely to be green on your machine and broken for a player.

## See also
- How-to: [Lint, build, and smoke-test a React UI bundle](../ui/react-testing.md) - the automated
  half of UI QA; [Add a custom runtime React panel](../ui/runtime-ui.md);
  [Inject React into vanilla UI with the module registry](../ui/module-registry.md);
  [Handle a missing UI dependency](../ui/dependency-handling.md).
- Operations: [Release Checklist](./release-checklist.md) - the mod-wide pre-publish gate this
  feeds into.
- Explanation: [The Gameface UI Runtime](../../explanation/gameface-runtime.md);
  [C#/React communication](../../explanation/ui-cs-communication.md).
- Index: [technique index](../../technique-index.md).
