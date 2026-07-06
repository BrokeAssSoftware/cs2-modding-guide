---
FrontmatterVersion: 1
DocumentType: Guide
Title: "How-to: Inject React into vanilla UI with the module registry"
diataxis: how-to
source_version: "~1.5.x (write-everywhere@13c70eb; date-pinned, static source only)"
last_reverified: "2026-07-04"
canonical_mods:
  - write-everywhere@13c70eb04e6bed152257c516a982455148f591a5
  - smooth-left-hand-traffic@37b2850b10ced1cd17457c25778409e57e00a03e
  - extra-lib@4879487b7df62e83676030a87f92d4b102955eee
technique_applicability: [ui]
status: source-verified
Created: 2026-07-03
Updated: 2026-07-05
Owners:
  - codex
---

# Inject React into vanilla UI with the module registry

> Make your mod's React appear *inside* the game's existing UI - a HUD button, a tool-option
> row, a custom panel, or a replaced vanilla component - by exporting a `ModRegistrar` and
> calling `append` / `extend` / `override` on the `moduleRegistry` the game passes you.

## Problem
A CS2 mod UI is a JavaScript bundle, but the game already owns the DOM - the HUD, tool options,
menus, and panels are all vanilla React modules. You cannot just render into `document.body`;
you have to hook your components onto the game's module graph so they show up in the right place
and survive the game's own re-renders. You want to *add* to a spot, *wrap* an existing component,
or *replace* one - without patching game files.

## Solution
Your UI entry point exports a **`ModRegistrar`**: a function the game calls once at UI load,
handing it a **`moduleRegistry`**. The registry both indexes every vanilla module (by path +
export name) and gives you verbs to modify them: `append` adds a component to a mount target,
`extend` wraps an existing export with a higher-order component, and `override` swaps an export
out entirely. `find` / `get` help you discover targets. This is source-verifiable from Write
Everywhere's UI project, whose bundled `cs2/modding` type declaration is the API contract and
whose entry point exercises `append` and `extend` against real vanilla module paths.

## Steps & Code

### 1. Export a `ModRegistrar` as the module default

The game imports your bundle and calls its default export with the registry. Everything you hook
happens inside this one function:

```ts
import { ModRegistrar } from "cs2/modding";

const register: ModRegistrar = (moduleRegistry) => {
    moduleRegistry.extend("game-ui/game/components/tool-options/mouse-tool-options/mouse-tool-options.tsx", 'MouseToolOptions', WriteEverywhereToolOptions);
    moduleRegistry.append('GameTopLeft', WEButton);
}
export default register;
```
Source: `../../../vice-and-order-research/mods/dossiers/write-everywhere/repo/_Frontends/UI/k45-we-vuio/src/index.tsx#L3,#L9-L16,#L47` (@13c70eb04e6bed152257c516a982455148f591a5)

The full registry surface is declared in the bundled `cs2/modding` types the mod builds against:

```ts
export type ModuleRegistry = {
  get(modulePath: string, exportName: string): any;
  add(modulePath: string, module: Record<string, any>): void;
  override(modulePath: string, exportName: string, newValue: any): void;
  extend(modulePath: string, exportNameOrSCSSValue: string | any, extendCb?: ModuleRegistryExtend): void;
  append(modulePath: string, exportName: string, appendedComponent?: ModuleRegistryAppend, index?: number): void;
  append(target: AppendHookTargets, appendedComponent: ModuleRegistryAppend, index?: number, _?: never): void;
  registry: Map<string, Record<string, any>>;
  find(query: string | RegExp): [path: string, ...exports: string[]][];
  reset(): void;
};
```
Source: `../../../vice-and-order-research/mods/dossiers/write-everywhere/repo/_Frontends/UI/k45-we-vuio/types/modding.d.ts#L4-L20` (@13c70eb04e6bed152257c516a982455148f591a5) - this is the type declaration the mod compiles against, i.e. the API surface; live calls to `override` / `add` / `find` / `reset` are not exercised in this mod's entry point, so treat their *runtime* behaviour as **Needs Verification**.

### 2. `append` - add a component to a mount target

`append` has two shapes (both in the type above). The short form takes a named **mount target**
from a fixed set (`AppendHookTargets` = `"Menu" | "Editor" | "Game" | "GameTopLeft" |
"GameTopRight" | "GameBottomRight"`) and drops your component there. Write Everywhere adds its
toolbar button to the top-left of the in-game HUD:

```ts
moduleRegistry.append('GameTopLeft', WEButton);
```
Source: `../../../vice-and-order-research/mods/dossiers/write-everywhere/repo/_Frontends/UI/k45-we-vuio/src/index.tsx#L15` (@13c70eb04e6bed152257c516a982455148f591a5); target set: `.../types/modding.d.ts#L6`.

The long form (`append(modulePath, exportName, component, index?)`) appends into a specific
vanilla module's export rather than a global mount point. Use the short form for HUD/menu
corners; use the long form to inject into a named component's children.

### 3. `extend` - wrap an existing vanilla component

`extend` takes a module path, the export name to wrap, and a **`ModuleRegistryExtend`**
higher-order function `(Component) => (props) => JSX`. Your callback receives the original
component and returns a replacement that usually renders the original and adds to it. Write
Everywhere wraps the vanilla mouse-tool-options component and *unshifts* its own panel into the
original's children only while its tool is active:

```ts
export const WriteEverywhereToolOptions: ModuleRegistryExtend = (Component: any) => {
    return () => {
        const toolActive = useValue(tool.activeTool$).id == "K45_WE_WEWorldPickerTool";
        const result = Component();                 // render the vanilla component
        if (toolActive) {
            result.props.children ??= []
            result.props.children.unshift(<WEWorldPickerToolPanel />);   // add ours
        }
        return result;
    };
}
```
Source: `../../../vice-and-order-research/mods/dossiers/write-everywhere/repo/_Frontends/UI/k45-we-vuio/src/toolOptions/WriteEverywhereToolOptions.tsx#L91-L101` (@13c70eb04e6bed152257c516a982455148f591a5); registered at `.../src/index.tsx#L11`.

The key discipline: **call `Component()` and return valid JSX.** If your wrapper forgets to render
the original, you delete the vanilla UI you hooked. `extend` is the safest verb because it composes
with other mods that also extend the same component.

### 4. `override` - replace an export entirely

`override(modulePath, exportName, newValue)` swaps a vanilla export for your own. It is the
heaviest hammer: whatever you replace is gone, and any other mod that expected the original (or
that also overrides it) conflicts with you. Prefer `extend` unless you truly must replace the
component. **Needs Verification:** Write Everywhere does not call `override` at this commit, so
its runtime semantics beyond the declared signature are unverified here - confirm in-engine.

### 5. Discover mount points before you hook them

You cannot hook a module path you do not know. The registry exposes discovery so you do not guess:

- `find(query)` returns matching `[path, ...exports]` tuples for a string or `RegExp`.
- `get(modulePath, exportName)` returns the current export (to inspect before wrapping).
- `registry` is the raw `Map` of every indexed module.

Source (signatures): `../../../vice-and-order-research/mods/dossiers/write-everywhere/repo/_Frontends/UI/k45-we-vuio/types/modding.d.ts#L8,#L14-L19` (@13c70eb04e6bed152257c516a982455148f591a5). Use `find(/tool-options/)` (or the in-engine Gameface inspector) to locate the real path and export name for the vanilla component you want, then hook it. The concrete module-path strings in Write Everywhere's entry (`game-ui/game/components/tool-options/...`) are verified for *this* game version; the full catalogue of vanilla paths is **Needs Verification** and version-specific - always discover, never hard-code from memory.

## Pitfalls & gotchas

- **The module path + export name is a stringly-typed contract.** A wrong path or export name is
  a silent no-op (or a crash if the module resolves but the export does not). Copy paths from
  `find` output, not from memory, and re-check them each game update.
- **`extend` must render the original.** A callback that returns its own JSX without calling
  `Component()` erases the vanilla UI it wrapped. Render the original and add to it (step 3).
- **`override` collides with other mods.** Two mods overriding the same export cannot coexist;
  the last registrar wins. Reserve `override` for cases `extend` cannot express.
- **`append` targets are a closed set.** The short-form mount points are exactly the
  `AppendHookTargets` union; anything else must go through the long form against a real module
  path (`.../types/modding.d.ts#L6,#L12-L13`).
- **Vanilla paths drift between game versions.** The `game-ui/...` paths that resolve today may
  move; discovery via `find` keeps you honest. Treat any path not re-verified against the current
  build as **Needs Verification**.

## Variations

- **Register a game-panel type + renderer.** Beyond HUD corners, Write Everywhere `extend`s the
  panel-type binding and the panel-renderer maps to register a full custom panel
  (`.../src/index.tsx#L12-L13`) - the same registry verbs, applied to the panel subsystem.
- **Feed the hooked component with C# state.** A registry hook only *places* React; its data
  comes from the binding channel. Back your appended/extended component with `ValueBinding` /
  `TriggerBinding` - see [C#/React communication](../../explanation/ui-cs-communication.md).
- **Editor vs game targets.** The `"Editor"` and `"Menu"` mount targets place UI in the map
  editor and main menu respectively, not just the in-game HUD.
- **A simpler `extend` of the same mouse-tool-options module.** Smooth Left-Hand Traffic hooks the
  identical `mouse-tool-options.tsx` / `MouseToolOptions` export as Write Everywhere (step 3), but
  with a leaner shape: it first `get`s the real vanilla `Section` and `ToolButton` exports up front
  - throwing if the game has moved them - then `extend`s with a factory that closes over those
  resolved components:
  ```ts
  const MOUSE_TOOL_OPTIONS_MODULE = "game-ui/game/components/tool-options/mouse-tool-options/mouse-tool-options.tsx";
  // ...resolve real vanilla exports before hooking (fail loudly if missing)
  Section: getRegistryExport(moduleRegistry, MOUSE_TOOL_OPTIONS_MODULE, "Section"),
  ToolButton: getRegistryExport(moduleRegistry, TOOL_BUTTON_MODULE, "ToolButton"),
  // ...
  const register: ModRegistrar = (moduleRegistry) => {
      const toolOptionsComponents = resolveToolOptionsComponents(moduleRegistry);
      moduleRegistry.extend(MOUSE_TOOL_OPTIONS_MODULE, "MouseToolOptions", createInvertLHTTool(toolOptionsComponents));
  };
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/smooth-left-hand-traffic/repo/SmoothLHT/UI/SmoothLHT/src/index.tsx#L4-L5,#L8-L15,#L21-L22,#L27-L34` (@37b2850b10ced1cd17457c25778409e57e00a03e).
  Resolving `Section` / `ToolButton` with `get` and erroring when an export is missing is the step-5
  discovery discipline applied at load time - decorate the vanilla tool options, never rebuild them.
- **Nested asset-menu categories: `extend` plus a `ToolbarUISystem` patch.** Extra Lib augments the
  vanilla asset-category tab bar rather than replacing it, spanning both languages. On the React
  side it `extend`s `asset-category-tab-bar.tsx` / `AssetCategoryTabBar` with a HOC that renders the
  original `<Component .../>` and appends extra tab bars driven by two `el` bindings; on the C# side
  an `AssetMultiCategory : UISystemBase` publishes those bindings, and Harmony `Postfix` patches on
  `ToolbarUISystem.SelectAssetMenu` / `SelectAssetCategory` feed it the current selection. Same
  "decorate, don't replace" idiom as `extend`, extended across the binding channel.
  Source: extend at `../../../vice-and-order-research/mods/dossiers/extra-lib/repo/UI/src/index.tsx#L12`;
  HOC at `.../extra-lib/repo/UI/src/mods/AssetMultiCategory.tsx#L21-L40`; C# bindings at
  `.../extra-lib/repo/MOD/Systems/UI/AssetMultiCategory.cs#L16,#L49-L50`; patches at
  `.../extra-lib/repo/MOD/Patches/ToolbarUISystemPatch.cs#L13-L41` (@4879487b7df62e83676030a87f92d4b102955eee).
  See the recipe [Augment vanilla UI via moduleRegistry.extend](../recipes/vanilla-ui-augmentation.md)
  for the full pattern (toolbar / button / InfoSection / nested categories).

## See also
- How-to: [Add a custom runtime React panel](runtime-ui.md) - deciding to build one in the first
  place; [Set up the React UI development loop](react-development.md) - building the bundle you
  register; [Lint, build, and smoke-test a React UI bundle](react-testing.md);
  [Augment vanilla UI via moduleRegistry.extend](../recipes/vanilla-ui-augmentation.md) - the
  recipe collecting the "decorate, don't replace" `extend` pattern across mods.
- Explanation: [The Gameface UI Runtime](../../explanation/gameface-runtime.md) - the web-subset
  your injected components run in; [C#/React communication](../../explanation/ui-cs-communication.md)
  - the binding channel that supplies their data.
- Index: [technique index](../../technique-index.md) - family P (`UISystemBase` / React binding);
  Write Everywhere is a canonical mod there.

## Sources
- Canonical mods (dossier + repo):
  - `write-everywhere` @13c70eb04e6bed152257c516a982455148f591a5 -
    `repo/_Frontends/UI/k45-we-vuio/types/modding.d.ts` (the `moduleRegistry` API contract),
    `repo/_Frontends/UI/k45-we-vuio/src/index.tsx` (`ModRegistrar` with live `append`/`extend`),
    `repo/_Frontends/UI/k45-we-vuio/src/toolOptions/WriteEverywhereToolOptions.tsx` (`ModuleRegistryExtend` HOC)
  - `smooth-left-hand-traffic` @37b2850b10ced1cd17457c25778409e57e00a03e -
    `repo/SmoothLHT/UI/SmoothLHT/src/index.tsx` (a simpler `get` + `extend` of `MouseToolOptions`)
  - `extra-lib` @4879487b7df62e83676030a87f92d4b102955eee -
    `repo/UI/src/index.tsx` + `repo/UI/src/mods/AssetMultiCategory.tsx` (`extend` of `AssetCategoryTabBar`),
    `repo/MOD/Systems/UI/AssetMultiCategory.cs` (the `el` bindings behind it),
    `repo/MOD/Patches/ToolbarUISystemPatch.cs` (Harmony postfixes on `ToolbarUISystem` selection)
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding
