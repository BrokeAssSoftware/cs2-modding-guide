# UI React Pipeline

Vice & Order UI layers run on the Gameface React runtime that ships with Cities: Skylines II. This document describes how to scaffold a UI project, hook it into the code mod, develop with hot reload, and ship a production bundle that stays in sync with the C# side.

## Scaffold and Install Dependencies
1. Open a terminal inside the code mod folder (the directory containing `<Module>.csproj`).
2. Run the official template:
   ```powershell
   npx create-csii-ui-mod
   ```
   Accept the defaults so the generated project matches Colossal’s expectations.
3. Change into the UI directory (default `ui/`) and install dependencies if the scaffold skipped it:
   ```powershell
   cd ui
   npm install
   ```
4. Align identifiers:
   - `package.json` → `name`: `"vno-ui-module"`
   - `mod.json` → `id`: must exactly match the code mod assembly (for example `VNO.UI`).
   - `mod.json` → `dependencies`: declare any required mods (ExtraLib, Unified Icon Library, etc.).

## Wire the Bundle into MSBuild
Add build targets to the C# project so `dotnet build` produces and deploys the UI bundle automatically:
```xml
<Target Name="BuildUI" AfterTargets="AfterBuild">
  <Exec Command="npm run build" WorkingDirectory="$(ProjectDir)ui" />
</Target>

<Target Name="CopyUIBundle" AfterTargets="DeployWIP">
  <ItemGroup>
    <UIBundle Include="ui\dist\**\*.*" />
  </ItemGroup>
  <Copy SourceFiles="@(UIBundle)"
        DestinationFiles="@(UIBundle->'$(DeployDir)\%(RecursiveDir)%(Filename)%(Extension)')" />
</Target>
```
Keep `ui/dist/` out of source control and rely on the build step to regenerate it.

## Understand the Module Registry
Gameface resolves modules through the `ModRegistrar` exported from `src/index.ts` (or similarly named file):
```ts
import { ModRegistrar } from "cs2/modding";
import { registerVicePanel } from "./vice-panel";

const registrar: ModRegistrar = (registry) => {
  registry.append("Game.UI.GameMenu", registerVicePanel);
};

export default registrar;
```
- Use `registry.append("Host.Component", handler)` to add content, `registry.extend` to wrap existing components, and `registry.override` to replace them entirely.
- Inspect the current registry with `registry.find(/pattern/)` or via the Gameface inspector to avoid overriding unexpected modules.
- Log your registrations once during init so you can confirm they ran (use `console.log` or the `cs2/utils` logger).

## Recommended React Patterns
- Import hooks from `cs2/api` such as `useGameSimulation`, `useLocalization`, and `useUpdate` to synchronise with simulation ticks and locale changes.
- Scope heavy computation with `useMemo` and `useCallback`; Gameface runs on the main thread, so wasted renders hurt simulation performance.
- Use CSS modules (`panel.module.scss`) to avoid leaking styles globally. The runtime supports modern flexbox but not every CSS feature—test frequently.
- Prefer SVG assets (served from UIL or your own COUI host); raster images increase bundle size and memory usage.

## Development Workflow with Hot Reload
1. Run the dev server inside the UI folder:
   ```powershell
   npm run dev
   ```
2. Launch the game using the dev shortcut with `--uiDeveloperMode`.
3. Gameface will load assets from `http://localhost:5173` (default Vite port) instead of the bundled files.
4. Open `http://localhost:9444` in a Chromium-based browser to inspect the component tree, props, hooks, and network traffic.
5. Use the inspector’s “Reload bundle” button when you change global styles or module registration logic.

## Communicating with C# Systems
- **DTO buffers** – expose simulation state via `NativeList` or `DynamicBuffer` and mirror it into a managed singleton that the UI polls through a minimal API (`viceDataService.getSnapshot()`).
- **Message channels** – when UI needs to trigger simulation commands, expose a service class on the C# side (for example `ViceCommandBus.Enqueue`) and call it through a shared `cs2/api` binding.
- **Settings bridge** – read `ModsSettings` values inside the UI on boot (using `cs2/modding.ModSettings`) so React components start with the same defaults as C#.
- **Latency consideration** – keep payloads small; Gameface marshals data via JSON. For high-frequency data (e.g., heat maps), send deltas or aggregated buckets.

## Testing and QA
- **Lint and type-check** – add scripts (`npm run lint`, `npm run typecheck`) and wire them into CI to catch regressions early.
- **Smoke test** – run `npm run build` locally and open `dist/index.html` in a browser to sanity check the layout outside the game.
- **Game pass** – open the dev tools, navigate through each panel, and ensure localization tokens, icons, and tooltips resolve in at least two languages.
- **Dependency failure** – launch without UIL or ExtraLib; verify the UI warns gracefully and falls back to text badges or neutral icons.
- **Performance** – profile with Gameface’s performance overlay or Chromium dev tools. Aim for lightweight components that do not trigger layout thrashing on every tick.

## Publishing Checklist
1. Build the UI bundle in Release mode (`npm run build`).
2. Run `dotnet build -c Release` to package the code mod and copy the UI into the deploy directory.
3. Launch the game without `npm run dev` to verify the production bundle loads.
4. Publish via the in-game toolchain and confirm the uploaded archive contains the UI `dist/` files under the mod directory.
5. Tag the release and note the UI bundle version in release notes so automated update scripts can track compatibility.

By following this pipeline, every Vice & Order UI feature ships with a reliable development loop, consistent module registration, and bundles that stay in lockstep with the underlying simulation logic.
