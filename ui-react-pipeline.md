# UI React Pipeline

Vice & Order UI layers sit on the Gameface React runtime (`cs2/*` packages). Follow this workflow to wire UI bundles to code mods.

## Project Setup

- Run `npx create-csii-ui-mod` inside the code mod folder; accept the defaults so the scaffold matches Colossal's expectations.
- Align `mod.json.id` with the C# assembly name; mismatch prevents the runtime from linking bundles.
- Add a custom MSBuild target in the code mod `.csproj` to invoke `npm run build` during `AfterBuild`.
- Keep `npm run dev` available for hot reload; it serves assets to the running game when `--uiDeveloperMode` is enabled.

## Module Registry

- Export a default `ModRegistrar` function and call `registry.append`, `registry.extend`, or `registry.override` to hook game modules.
- Investigate host module IDs with `registry.find(/pattern/)` before overriding; log matches to avoid blind overrides.
- Respect lifecycle order: initialization happens once, `onWillMount`/`onUnmount` each time the component mounts, and `onUpdate` on every state change.

## React Patterns

- Use `cs2/api` hooks (`useGameSimulation`, `useUpdate`) to synchronize with simulation ticks.
- Pull shared styles via CSS modules; Gameface supports standard flexbox but not every modern CSS feature.
- Prefer SVG assets; raster textures require explicit imports and add bundle weight.
- Wrap heavy data fetches in `useMemo` or `useCallback`; the runtime runs on the main thread.

## Debugging

- With `--uiDeveloperMode`, open the Gameface inspector at `http://localhost:9444/` to inspect component trees and network calls.
- Enable `SetShowsErrorsInUI(true)` temporarily when debugging UI errors to surface toast notifications in-game.
- Log through `cs2/utils` logger helpers to keep output consistent with the game console.

## Communicating With Code Mods

- Share state via ECS components or custom message channels exposed through `cs2/api`.
- For simple cases, inject settings through the `ModsSettings` file and read them from the UI bundle during boot.
- When UI needs live simulation data, expose read-only buffers via a `SystemBase` that writes to `NativeList` and mirror it in the UI system with marshalled DTOs.

## Packaging

- Keep `dist/` out of source control; rebuild during MSBuild or CI.
- Minimize bundle size by code-splitting large overlays and lazy-loading seldom used panels.
- Version the UI bundle alongside the code mod so publish scripts can confirm alignment.
