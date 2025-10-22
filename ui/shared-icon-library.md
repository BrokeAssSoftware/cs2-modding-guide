# Shared Icon Library

Reference mod: `UnifiedIconLibrary`.

Reuse high-quality, game-matching icons without bundling your own SVG sets.

## 1. Dependency Setup

- Add Unified Icon Library as a required dependency in `mod.json`.
- During `OnLoad`, verify the library is active. If not, warn the player and fall back to safe defaults (e.g., text labels instead of icons).

## 2. Referencing Icons

- Every icon is pre-injected into the UI under `coui://uil/<Style>/<IconName>.svg`.
- Styles include `Standard`, `Dark`, and `Colored`. Pick the style that matches your UI theme.
- Example usage in a Gameface template:
  ```html
  <uil-icon src="coui://uil/Standard/ArrowLeft.svg" />
  ```

## 3. Recolouring Icons

- For coloured icon sets, override the layer colour via CSS:
  ```css
  .warning-icon {
    color: var(--uil-red);
  }
  ```
- Predefined colour tokens: `black`, `white`, `grey`, `brown`, `violet`, `red`, `orange`, `yellow`, `green`, `blue`. Hex codes are also supported.

## 4. Discovering Assets

- Browse the repository's `Icons/` directory to see all names and styles.
- Check `Properties/Previews/` for quick visual sheets that document the available icons.

## 5. Support And Custom Requests

- Reach out via the Cities: Skylines modding Discord for tailored icon advice or to request new styles. The maintainer actively supports fellow modders.

