# Placement Validation Overrides

Reference mod: `Anarchy`.

Bypassing tool errors lets designers place assets in otherwise restricted scenarios (tight curves, shoreline violations, overlapping props). Follow these steps to keep the experience safe and reversible.

## 1. Patch Tool Error Pipelines

- Use Harmony to intercept the validation methods in `NetToolSystem`, `ObjectToolSystem`, and `BulldozeToolSystem`.
- After the base game builds its `ToolBase.Error` collection, remove entries that match your opt-in list (e.g., `ToolErrorType.Overlap`, `ToolErrorType.InWater`, `ToolErrorType.LongDistance`).
- Only skip errors when your override is enabled; fall back to vanilla to preserve the default safety net.

## 2. Share Toggle State

- Maintain a singleton service that tracks whether anarchy is active. Expose it to:
  - The UI (toolbar button, flaming chirper, tooltip).
  - The Harmony postfix/prefix methods that decide whether to strip errors.
  - Hotkeys (e.g., `Ctrl+A`) that let players toggle the mode quickly.
- Persist state in your settings so the tool remembers the previous choice.

## 3. Add Per-Tool Safeguards

- Some workflows (tree brushing, line tool painting) should auto-disable validation overrides. Detect the active tool via `ToolSystem.activeTool` and temporarily suspend anarchy when the tool is on the opt-out list.
- Provide settings so advanced players can tailor which tools auto-disable the mode.

## 4. Support Elevation Tweaks

- Expose keyboard shortcuts for incremental elevation adjustments while the object or line tool is active.
- Modify the placement command's elevation fields before it is enqueued (for example by editing the struct passed to `ObjectToolSystem.Submit`).
- Offer a reset action that snaps the elevation back to zero so players can recover quickly.

## 5. Surface Warnings

- Display in-world feedback (icon, toast, status text) when validation overrides are active.
- Reset to vanilla behaviour on tool change, on load, or after a configurable timeout to avoid accidental long-term misuse.

