# UI React Pipeline

Vice & Order UI layers run on the Gameface React runtime that ships with Cities: Skylines II. This document describes how to scaffold a UI project, hook it into the code mod, develop with hot reload, and ship a production bundle that stays in sync with the C# side.

## Scaffold and Install Dependencies
1. Open a terminal inside the code mod folder (the directory containing `<Module>.csproj`).
2. Run the official template:
   ```powershell
   npx create-csii-ui-mod
   ```
   Accept the defaults so the generated project matches Colossal's expectations.
3. Change into the UI directory (default `ui/`) and install dependencies if the scaffold skipped it:
   ```powershell
   cd ui
   npm install
   ```
4. Align identifiers:
   - `package.json` `name` should match your module ID.
   - `mod.json` `id` must exactly match the code mod assembly.
   - Declare required dependencies (ExtraLib, UIL, etc.) in `mod.json`.

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
Gameface resolves modules through the `ModRegistrar` exported from `src/index.ts`:
```ts
import { ModRegistrar } from "cs2/modding";
import { registerVicePanel } from "./vice-panel";

const registrar: ModRegistrar = (registry) => {
  registry.append("Game.UI.GameMenu", registerVicePanel);
};

export default registrar;
```
- Use `registry.append` to add content, `registry.extend` to wrap existing components, and `registry.override` to replace them entirely.
- Inspect the registry with `registry.find(/pattern/)` or via the Gameface inspector to avoid overriding unexpected modules.
- Log registrations once during init so you can confirm they ran.

## Recommended React Patterns
- Import hooks from `cs2/api` such as `useGameSimulation`, `useLocalization`, and `useUpdate` to synchronise with simulation ticks and locale changes.
- Scope heavy computation with `useMemo` and `useCallback`; the runtime runs on the main thread, so wasted renders impact performance.
- Use CSS modules to avoid leaking styles globally. The runtime supports modern flexbox but not every CSS feature, so test frequently.
- Prefer SVG assets (served from UIL or your own COUI host); raster images increase bundle size and memory usage.

## Development Workflow with Hot Reload
1. Run the dev server inside the UI folder:
   ```powershell
   npm run dev
   ```
2. Launch the game with `-uiDeveloperMode`.
3. Gameface loads assets from the dev server instead of the bundled files, enabling hot reload.
4. Open `http://localhost:9444` in a Chromium-based browser to inspect the component tree, props, hooks, and network traffic.
5. Use the inspector's reload button when you change global styles or module registration logic.

## Communicating with C# Systems
- Share simulation data via DTO buffers or message channels exposed by the C# side.
- For simple settings, read `ModsSettings` through `cs2/modding.ModSettings` during UI boot so React components start with correct defaults.
- When UI actions need to trigger simulation work, expose a service method (for example `ViceCommandBus.Enqueue`) and call it via a thin API wrapper.
- Keep payloads small; Gameface marshals data via JSON.

## Testing and QA
- Add linting and type-check scripts (`npm run lint`, `npm run typecheck`) and run them in CI.
- Build locally with `npm run build` and open `dist/index.html` in a browser for a quick smoke test.
- Launch the game without `npm run dev` to verify the production bundle loads correctly.
- Test in multiple languages and without shared dependencies (UIL, ExtraLib) to confirm fallbacks.
- Profile with the Gameface inspector or Chromium dev tools and keep components light.

## Publishing Checklist
1. Run `npm run build` (Release configuration for bundlers).
2. Run `dotnet build -c Release` to package the code mod and copy the UI bundle.
3. Launch the game without the dev server to confirm the production assets load.
4. Publish via the in-game toolchain and verify the uploaded archive contains the UI `dist/` files.
5. Tag the release and note the bundle version in release notes so automated update scripts can track compatibility.

By following this pipeline, every Vice & Order UI feature ships with a reliable development loop, consistent module registration, and bundles that stay in lockstep with the simulation logic.
