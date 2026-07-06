---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Gameface / React UI: the consume side"
Summary: "How a CS2 mod's TSX component mounts into the game UI and consumes a C# binding - useValue, engine.on, engine.call, and engine.trigger - explained against a real React frontend."
diataxis: explanation
source_version: "~1.5.x (write-everywhere@13c70eb; date-pinned, static source only)"
last_reverified: "2026-07-04"
status: source-verified
canonical_mods:
  - write-everywhere@13c70eb04e6bed152257c516a982455148f591a5
  - advanced-simulation-speed@d224017f15b6be7384ffcc34528553f86b22280d
  - time-weather-anarchy@71128d3958c31b03163a2cba981909655a38bc83
technique_applicability: [ui]
Created: 2026-07-04
Updated: 2026-07-05
Owners:
  - codex
References:
  - Label: How-to - mount React via the module registry
    Path: ../how-to/ui/module-registry.md
  - Label: Concept - the Gameface runtime
    Path: ./gameface-runtime.md
  - Label: Concept - UI <-> C# communication
    Path: ./ui-cs-communication.md
  - Label: Recipe - UISystemBase React binding (the C# side)
    Path: ../how-to/recipes/uisystembase-react-binding.md
  - Label: Case study - write-everywhere ecosystem
    Path: ../case-studies/write-everywhere-ecosystem.md
---

# Gameface / React UI: the consume side

Cities: Skylines II renders its entire interface with **Gameface** (Coherent Labs'
HTML/CSS/JS engine) driving a **React** tree. A mod does not draw pixels; it ships a
JavaScript bundle that React mounts into that same tree, and its components read game
state through **data bindings** that the C# simulation publishes. This page explains the
*consume* side: once your TSX is mounted, what does a component actually render, and how
does it pull a value out of C# and push a command back?

This is the wall most agents hit. Getting React *mounted* is a solved, mechanical step
(see the how-to [mount React via the module registry](../how-to/ui/module-registry.md)).
What is unobvious is the C# <-> TSX data-flow: there is no `fetch`, no REST, no shared
memory you can poke. There is a small, specific bridge API, and this page grounds it in a
real frontend - **write-everywhere**, whose TSX lives at
`_Frontends/UI/k45-we-vuio/`. Every claim below is cited from that codebase at commit
`13c70eb`.

This is a concept page. The **C# half** of the bridge - how a `UISystemBase` publishes a
`ValueBinding`/`TriggerBinding` - is the how-to recipe
[UISystemBase React binding](../how-to/recipes/uisystembase-react-binding.md); the
runtime it all sits on is [the Gameface runtime](./gameface-runtime.md). This page stays
on the consume side.

## The mount point: where your component enters the tree

The game exposes its React module graph to mods through a **module registry**. Your mod's
entry point is a `ModRegistrar` - a function handed the registry - and it has exactly two
verbs for getting a component onto screen: `append` (add your component to a named host
slot) and `extend` (wrap/replace a named export of a vanilla module). write-everywhere's
entry file uses both ([write-everywhere repo/_Frontends/UI/k45-we-vuio/src/index.tsx#L9-L16](../../vice-and-order-research/mods/dossiers/write-everywhere/repo/_Frontends/UI/k45-we-vuio/src/index.tsx), commit `13c70eb`):

```tsx
const register: ModRegistrar = (moduleRegistry) => {
    moduleRegistry.extend("game-ui/game/components/tool-options/tool-options-panel.tsx", 'useToolOptionsVisible', WriteEverywhereToolOptionsVisibility);
    moduleRegistry.extend("game-ui/game/components/tool-options/mouse-tool-options/mouse-tool-options.tsx", 'MouseToolOptions', WriteEverywhereToolOptions);
    moduleRegistry.extend("game-ui/game/data-binding/game-bindings.ts", 'GamePanelType', RegisterWePanelType);
    moduleRegistry.extend("game-ui/game/components/game-panel-renderer.tsx", 'gamePanelComponents', RegisterWePanel);
    moduleRegistry.extend("game-ui/editor/components/toolbar/toolbar.tsx", 'Toolbar', WePanelEditor);
    moduleRegistry.append('GameTopLeft', WEButton);
}
```

`append('GameTopLeft', WEButton)` inserts the floating toolbar button into the vanilla
`GameTopLeft` host; the `extend(...)` calls hook named exports of specific vanilla `.tsx`
modules by path. The registry's contract is small - the signatures are the whole surface
([write-everywhere repo/_Frontends/UI/k45-we-vuio/types/modding.d.ts#L11-L13, #L21](../../vice-and-order-research/mods/dossiers/write-everywhere/repo/_Frontends/UI/k45-we-vuio/types/modding.d.ts), commit `13c70eb`):

```ts
extend(modulePath: string, exportNameOrSCSSValue: string | any, extendCb?: ModuleRegistryExtend): void;
append(modulePath: string, exportName: string, appendedComponent?: ModuleRegistryAppend, index?: number): void;
append(target: AppendHookTargets, appendedComponent: ModuleRegistryAppend, index?: number, _?: never): void;
export type ModRegistrar = (moduleRegistry: ModuleRegistry) => void;
```

An `extend` callback receives the vanilla component and returns a replacement, which is how
write-everywhere splices its panel into `MouseToolOptions`: it calls the original, then
conditionally unshifts its own child ([write-everywhere repo/_Frontends/UI/k45-we-vuio/src/toolOptions/WriteEverywhereToolOptions.tsx#L91-L100](../../vice-and-order-research/mods/dossiers/write-everywhere/repo/_Frontends/UI/k45-we-vuio/src/toolOptions/WriteEverywhereToolOptions.tsx), commit `13c70eb`):

```tsx
export const WriteEverywhereToolOptions: ModuleRegistryExtend = (Component: any) => {
    return () => {
        const toolActive = useValue(tool.activeTool$).id == "K45_WE_WEWorldPickerTool";
        const result = Component();
        if (toolActive) {
            result.props.children ??= []
            result.props.children.unshift(<WEWorldPickerToolPanel />);
        }
        return result;
```

The step-by-step of *choosing* hosts and paths is the how-to
[module-registry](../how-to/ui/module-registry.md); the concept to hold onto is that once
`register` runs, your React component is a first-class node in the game's tree and re-renders
on the same clock as everything else.

### `extend` to decorate, `append` to mount - and register twice to survive a rewrite

The two verbs carry different intent. `append` *mounts alongside* - it drops your component
into a named host slot (`GameTopLeft` above). `extend` *decorates in place* - it replaces a
named export of a vanilla module, so the game renders your version where the original stood.
The decorate case is the fragile one: you are binding to a specific vanilla module *path*,
and the game can rebuild that module between versions. advanced-simulation-speed swaps the
game's whole time-controls widget, and because the 1.6 toolbar rewrite moved that widget, it
registers the **same** replacement against *both* the legacy and the new path so one of them
always matches ([advanced-simulation-speed repo/UI/src/index.tsx#L7-L19](../../vice-and-order-research/mods/dossiers/advanced-simulation-speed/repo/UI/src/index.tsx), commit `d224017`):

```tsx
moduleRegistry.extend(
  "game-ui/game/components/toolbar/bottom/time-controls/time-controls.tsx",
  "TimeControls", AdvancedTimeControls);
moduleRegistry.extend(
  "game-ui/game/components/toolbar/bottom/time-controls/time-controls-new.tsx",
  "TimeControlsNew", AdvancedTimeControls);
```

The legacy `time-controls.tsx#TimeControls` and the rewritten `time-controls-new.tsx#TimeControlsNew`
are two different module targets; whichever the running game exposes takes the decorator, and
the other silently no-ops. When you `extend` a vanilla component the game may rewrite, dual-register
against old and new paths rather than betting on a single one.

### Pitfall: unguarded `getModule` casts degrade silently on a rename

`extend`/`append` hook whole exports; when you instead need to *reuse* a single vanilla piece -
a component or its SCSS class map - you reach for `getModule(path, export)`, and its return is an
unchecked `as` cast. advanced-simulation-speed pulls the toolbar `Divider` and the field style
classes this way ([advanced-simulation-speed repo/UI/src/mods/utils/utils.ts#L5-L13](../../vice-and-order-research/mods/dossiers/advanced-simulation-speed/repo/UI/src/mods/utils/utils.ts), [repo/UI/src/mods/simSpeedComponent.tsx#L12-L15](../../vice-and-order-research/mods/dossiers/advanced-simulation-speed/repo/UI/src/mods/simSpeedComponent.tsx), commit `d224017`):

```ts
export const getModuleComponent = <Props = any>(modulePath: string, exportName: string) =>
  getModule(modulePath, exportName) as Component<Props>;
// ...consumed as:
const fieldStyles = getModuleClasses<{ field: any; content: any }>(
  "game-ui/game/components/toolbar/components/field/field.module.scss");
const Divider = getModuleComponent(toolbarFieldPath, "Divider");
```

The cast satisfies the compiler but guarantees nothing at runtime. If a game update renames
`Divider` or moves `field.module.scss`, `getModule` returns `undefined`, the `as` cast hides
that, and the component renders a blank where the divider or styled field was - no exception,
no console error, just missing UI. This is a sharper failure than a missing *binding* (which
has `api.d.ts`'s defined fallback-or-throw contract): a missing *module* fails quietly. The
exact on-screen symptom for a specific rename is `Needs Verification (in-game)`; the structural
hazard - an unchecked cast over a version-fragile string path - is plain in the source. Guard
these lookups (null-check the result, or feature-detect the export) before trusting the cast.

## The bridge API: `cs2/api`

C# and TSX talk over a narrow, typed bridge exported from the virtual module `cs2/api`.
Four functions carry almost all traffic. Their signatures are the contract
([write-everywhere repo/_Frontends/UI/k45-we-vuio/types/api.d.ts#L28, #L36, #L37, #L39](../../vice-and-order-research/mods/dossiers/write-everywhere/repo/_Frontends/UI/k45-we-vuio/types/api.d.ts), commit `13c70eb`):

```ts
export function bindValue<T>(group: string, name: string, fallbackValue?: T): ValueBinding<T>;
export function trigger(group: string, name: string, ...args: any[]): void;
export function call<T>(group: string, name: string, ...args: any[]): Promise<T>;
/** Subscribe to a ValueBinding. Return fallback value or throw ... if the binding is not registered on the C# side */
export function useValue<V>(binding: ValueBinding<V>): V;
```

(The `.d.ts` literally reads `export export function` - a harmless artefact of how the
game's type stubs were generated; treat it as one `export`.)

The direction of each:

- **`bindValue(group, name)`** resolves a **handle** to a C# `ValueBinding` published under
  a `(group, name)` pair. It does *not* give you the value - it gives you a subscribable
  object.
- **`useValue(binding)`** is the React hook that subscribes to that handle and returns the
  current value, re-rendering the component whenever C# pushes a new one. This is the
  read path.
- **`trigger(group, name, ...args)`** is fire-and-forget: it invokes a C# `TriggerBinding`.
  This is the write path for "do something", nothing returned.
- **`call<T>(group, name, ...args)`** is a request/response: it returns a `Promise<T>`
  resolved by C#. Use it when you need an answer back.

A `ValueBinding<T>` is just a readonly value plus a `subscribe`
([write-everywhere repo/_Frontends/UI/k45-we-vuio/types/api.d.ts#L4-L8](../../vice-and-order-research/mods/dossiers/write-everywhere/repo/_Frontends/UI/k45-we-vuio/types/api.d.ts), commit `13c70eb`):

```ts
export interface ValueBinding<T> {
	readonly value: T;
	subscribe(listener?: BindingListener<T>): ValueSubscription<T>;
	dispose(): void;
}
```

`useValue` is the ergonomic wrapper over that `subscribe`; inside a React render you almost
always want the hook, not manual subscribe/dispose.

## Reading a C# value in a component

The minimal read is one line. write-everywhere's tool-options wrapper reads the game's
*active tool* - a vanilla binding, not one of its own - to decide whether to show its panel
([write-everywhere repo/_Frontends/UI/k45-we-vuio/src/toolOptions/WriteEverywhereToolOptions.tsx#L2-L3, #L93](../../vice-and-order-research/mods/dossiers/write-everywhere/repo/_Frontends/UI/k45-we-vuio/src/toolOptions/WriteEverywhereToolOptions.tsx), commit `13c70eb`):

```tsx
import { useValue } from "cs2/api";
import { tool } from "cs2/bindings";
// ...inside the component:
const toolActive = useValue(tool.activeTool$).id == "K45_WE_WEWorldPickerTool";
```

Two things to read off this. First, `useValue(...)` returns the *unwrapped* value (here a
tool descriptor with an `.id`) and re-renders when C# changes the active tool - you never
poll. Second, `tool.activeTool$` comes from **`cs2/bindings`**: the game pre-exports a large
library of ready-made bindings for vanilla state (tools, budgets, entities, ...), so you
frequently consume game data without publishing anything yourself. The `$` suffix is the
game's convention for "this export is a `ValueBinding`."

For **your own** state, you would instead call `bindValue("myGroup", "myValue")` to get a
handle, then `useValue` it - the group/name being the same string pair your C#
`UISystemBase` registered the binding under. That C# registration is the recipe
[UISystemBase React binding](../how-to/recipes/uisystembase-react-binding.md); here it is
enough to know the TSX side only needs the matching strings.

### How a value reaches you: polled, dirtied, flushed per frame

`useValue` re-renders whenever C# *pushes* a new value - but a well-behaved publisher does not
push on every simulation tick, and understanding the cadence explains why a "live" number does
not storm React. advanced-simulation-speed's binding wrapper carries a `dirty` flag: assigning
`helper.Value = x` only stashes the value and marks it dirty; the actual `Binding.Update` runs
later in `ForceUpdate`, and only if dirty ([advanced-simulation-speed repo/Extensions/ValueBindingHelper.cs#L15-L39](../../vice-and-order-research/mods/dossiers/advanced-simulation-speed/repo/Extensions/ValueBindingHelper.cs), commit `d224017`):

```csharp
public T? Value {
    get => dirty ? valueToUpdate : Binding.value;
    set { dirty = true; valueToUpdate = value; }
}
public void ForceUpdate() {
    if (dirty) { Binding.Update(valueToUpdate); dirty = false; }
}
```

The system flushes every helper's `ForceUpdate` exactly once per frame from `OnUpdate`
([advanced-simulation-speed repo/Extensions/ExtendedUISystemBase.cs#L13-L21](../../vice-and-order-research/mods/dossiers/advanced-simulation-speed/repo/Extensions/ExtendedUISystemBase.cs)),
and the *source* value is re-sampled only on an interval - a 500 ms `IntervalTimer` polls the
actual speed and assigns `helper.Value` ([advanced-simulation-speed repo/Systems/AdvancedSimSpeedUISystem.cs#L58, #L92-L96](../../vice-and-order-research/mods/dossiers/advanced-simulation-speed/repo/Systems/AdvancedSimSpeedUISystem.cs), [repo/Util/IntervalTimer.cs#L52-L62](../../vice-and-order-research/mods/dossiers/advanced-simulation-speed/repo/Util/IntervalTimer.cs)):

```csharp
_IntervalTimer = IntervalTimer.EveryMilliseconds(500, () => _actualSpeed.Value = GetActualSpeed());
// OnUpdate: _IntervalTimer.Update(World.Time.DeltaTime);  // ticks the poll; ForceUpdate flushes
```

So a continuously-changing value reaches your TSX as: poll on an interval -> mark dirty -> coalesce
to at most one push per frame. The consume-side takeaway is that `useValue` re-renders are already
rate-limited by the publisher's poll-and-dirty batching, so reading a moving number is cheap on the
React side - the throttling lives in C#, not in your component.

## Writing back: triggers and calls

To push a command back to C#, the component fires a trigger. write-everywhere's main panel
changes tabs by triggering an event the C# side listens for, via the low-level `engine`
object ([write-everywhere repo/_Frontends/UI/k45-we-vuio/src/mainUI/WEMainUI.tsx#L42](../../vice-and-order-research/mods/dossiers/write-everywhere/repo/_Frontends/UI/k45-we-vuio/src/mainUI/WEMainUI.tsx), commit `13c70eb`):

```tsx
const onSelect = (i: string) => { engine.trigger("k45::we.main.setTabActive", tabs.indexOf(i as any)) }
```

`engine.trigger(name, ...args)` (from `cohtml/cohtml`) is the same fire-and-forget channel
that `trigger` from `cs2/api` wraps - a single dotted event name here rather than a
`(group, name)` pair. Its mirror is `engine.on`, used to *subscribe* the TSX side to an
event C# raises; the same panel registers a handler so C# can drive the tab selection back
([write-everywhere repo/_Frontends/UI/k45-we-vuio/src/index.tsx#L34-L35, #L40](../../vice-and-order-research/mods/dossiers/write-everywhere/repo/_Frontends/UI/k45-we-vuio/src/index.tsx), commit `13c70eb`):

```tsx
const [tabActive, setTabActive] = useState(0)
engine.on("k45::we.main.setTabActive", setTabActive)
// ...
{bindResult.value === "k45__we_MainWindow" && <Portal>
```

For request/response, this codebase leans heavily on `engine.call` (the wrapper behind
`cs2/api`'s `call<T>`). Every service class is a thin typed faade over named C# calls -
for example the module-options service ([write-everywhere repo/_Frontends/UI/k45-we-vuio/src/services/WEModulesService.tsx#L4-L6](../../vice-and-order-research/mods/dossiers/write-everywhere/repo/_Frontends/UI/k45-we-vuio/src/services/WEModulesService.tsx), commit `13c70eb`):

```tsx
static async listAllOptions(): Promise<Record<string, Record<string, WEModuleOptionFieldTypes & number>>> { return engine.call("k45::we.modules.listAllOptions"); }
static async getFieldValue<T>(moduleId: string, field: string): Promise<T> { return engine.call("k45::we.modules.getFieldValue", moduleId, field); }
static async setFieldValue<T>(moduleId: string, field: string, value: T): Promise<void> { return engine.call("k45::we.modules.setFieldValue", moduleId, field, "" + value); }
```

The same pattern repeats across `FontService`, `FileService`, and the rest - each method is
`engine.call("<dotted C# name>", ...args): Promise<T>`
([write-everywhere repo/_Frontends/UI/k45-we-vuio/src/services/FontService.tsx#L5-L13](../../vice-and-order-research/mods/dossiers/write-everywhere/repo/_Frontends/UI/k45-we-vuio/src/services/FontService.tsx), commit `13c70eb`). Collecting these into
service classes (rather than sprinkling raw `engine.call` through components) is the pattern
to copy: it keeps the C# name strings in one place and gives components a typed API.

### When the raw subscription is worth it

`useValue` covers component render. But when you want a **non-render side effect** on change
- clear some editor state, bump a rebuild counter - you subscribe to the binding directly and
run a callback. write-everywhere wraps its bindings in a helper class and subscribes each one
to a refresh function ([write-everywhere repo/_Frontends/UI/k45-we-vuio/src/services/WorldPickerService.tsx#L311-L323](../../vice-and-order-research/mods/dossiers/write-everywhere/repo/_Frontends/UI/k45-we-vuio/src/services/WorldPickerService.tsx), commit `13c70eb`):

```tsx
registerBindings(refreshFn: () => any) {
    Object.values(this.bindingList).map(y => {
        Object.values(y).map(z => z.subscribe(async () => refreshFn()));
    })
    this.refreshFnRegistered.push(refreshFn);
    this.bindingList.picker.CurrentSubEntity.subscribe(async () => {
        this.clearCurrentEditingFormulaeParam();
    })
    this.bindingList.picker.CurrentEntity.subscribe(async () => {
        this.clearCurrentEditingFormulaeParam()
    });
}
```

Every `subscribe` returns a subscription you are responsible for disposing - the class pairs
this with a `disposeBindings()` that walks the same list and calls `.dispose()`
(`WorldPickerService.tsx`, same file). The rule: `useValue` inside render (React manages the
lifecycle for you); manual `subscribe`/`dispose` only for imperative side effects, and always
tear them down.

## The import surface: vanilla modules you build against

Your TSX imports from a handful of game-provided virtual modules. Across write-everywhere's
files the recurring ones are:

- **`cs2/api`** - the binding bridge (`bindValue`, `useValue`, `trigger`, `call`) covered
  above.
- **`cs2/ui`** - the vanilla component library. Panels, buttons, tooltips, portals come from
  here, so your UI matches the game's look for free
  ([write-everywhere repo/_Frontends/UI/k45-we-vuio/src/mainUI/WEMainUI.tsx#L3, #L16-L26](../../vice-and-order-research/mods/dossiers/write-everywhere/repo/_Frontends/UI/k45-we-vuio/src/mainUI/WEMainUI.tsx), commit `13c70eb`):

  ```tsx
  import { Button, Panel, Tooltip } from "cs2/ui";
  export const WEButton = () => {
      return (
          <Tooltip tooltip="Write Everywhere">
              <Button
                  src={icon}
                  variant="floating"
                  onSelect={() => VanillaComponentResolver.instance.toggleGamePanel(WeMainPanelId)}
              />
          </Tooltip>
      );
  }
  ```

- **`cs2/bindings`** - the pre-exported vanilla binding library (`tool.activeTool$` above;
  types like `EnumField`, `DropdownItem` are imported from here too).
- **`cs2/modding`** - `ModRegistrar`, `ModuleRegistryExtend`, `ModuleRegistryAppend` (the
  registry types).
- **`cs2/input`** - focus management for interactive panels (`FocusDisabled`, `FocusBoundary`),
  used widely in the tool-option views.
- **`cohtml/cohtml`** - the low-level `engine` object (`engine.on/off/trigger/call/translate`)
  that `cs2/api` is built on.

Note the **`cs2/l10n`** slot: write-everywhere does *not* import it. It localizes through
`engine.translate` wrapped in a small helper instead
([write-everywhere repo/_Frontends/UI/k45-we-vuio/src/utils/translate.ts#L3-L7](../../vice-and-order-research/mods/dossiers/write-everywhere/repo/_Frontends/UI/k45-we-vuio/src/utils/translate.ts), commit `13c70eb`):

```ts
export const translate = function (key: string, fallback?: string) {
    const fullKey = `K45::WE.vuio[${key}]`;
    const tr = engine.translate(fullKey);
    if (tr === fullKey) { /* record missing key / return fallback */ }
    return tr;
}
```

So localization is another consume-of-C# path: `engine.translate(key)` reads a string the C#
locale source registered under that key. Whether a given mod reaches for `cs2/l10n` or wraps
`engine.translate` directly is a style choice; both resolve against the same locale data.

## Formatting: store canonical, convert in the view layer

A value the player reads in *their* units should still cross the bridge in one canonical unit;
the component converts at render time. time-weather-anarchy holds temperature in Celsius and
time as a raw hour number, and publishes the player's *preference* as a separate int binding
read straight from the game's own settings ([time-weather-anarchy repo/TimeWeatherAnarchy/TimeWeatherAnarchy/Code/System/TimeWeatherAnarchyUISystem.cs#L317-L329](../../vice-and-order-research/mods/dossiers/time-weather-anarchy/repo/TimeWeatherAnarchy/TimeWeatherAnarchy/Code/System/TimeWeatherAnarchyUISystem.cs), commit `71128d3`):

```csharp
temperaturePreference = GameManager.instance.settings.userInterface.temperatureUnit switch {
    InterfaceSettings.TemperatureUnit.Celsius    => Domain.TemperaturePreference.Celsius,
    InterfaceSettings.TemperatureUnit.Fahrenheit => Domain.TemperaturePreference.Fahrenheit,
    InterfaceSettings.TemperatureUnit.Kelvin     => Domain.TemperaturePreference.Kelvin,
    _ => temperaturePreference
};
```

`GetTimePreference` does the same against `userInterface.timeFormat` (12h vs 24h). The TSX
`useValue`s that preference alongside the canonical value and formats in the view - Celsius
in, `F`/`K` or a 12/24-hour string out ([time-weather-anarchy repo/TimeWeatherAnarchy/TimeWeatherAnarchy/UI/src/mods/panel/time-weather-panel.tsx#L130-L150, #L191-L192, #L630](../../vice-and-order-research/mods/dossiers/time-weather-anarchy/repo/TimeWeatherAnarchy/TimeWeatherAnarchy/UI/src/mods/panel/time-weather-panel.tsx), commit `71128d3`):

```tsx
const temperaturePreference = useValue(TemperaturePreferenceValueBinding);
// convertTemperature(celsius, pref): F is *9/5+32, K is +273.15, C passthrough
{translate("TimeWeatherAnarchy.Temperature")}: {convertTemperature(currentTemperature, temperaturePreference)}
```

Two bindings - one canonical value plus one preference - beat pre-formatting on the C# side.
The same Celsius number renders as C, F, or K purely in the component, and it re-formats the
instant the player flips the game's unit setting: the preference binding pushes, the component
re-renders, and the simulation never has to resend the temperature. Reading the game's own
`userInterface` settings (rather than duplicating a units toggle in your mod) also means your
display tracks the vanilla preference for free.

## The mental model, in one paragraph

C# is the source of truth. A `UISystemBase` on the C# side publishes named `ValueBinding`s
(state you can read) and `TriggerBinding`s (commands you can invoke), plus `call` handlers
(request/response). Your TSX, once mounted through the module registry, holds *handles* to
those names: it `useValue`s a binding to render current state and re-render on change, and it
`trigger`s / `call`s to act. Nothing crosses the boundary except values keyed by string
`(group, name)` pairs. Get the strings matched on both sides and the data flows; mismatch them
and `useValue` returns the fallback (or throws) with no other error. That last failure mode is
`Needs Verification (in-game)` for the exact symptom, but the design intent is visible in the
`api.d.ts` doc comment above: *"Return fallback value or throw ... if the binding is not
registered on the C# side."*

## See also

- How-to: [mount React via the module registry](../how-to/ui/module-registry.md) - the
  mechanical steps for `append`/`extend` and choosing host paths.
- Concept: [the Gameface runtime](./gameface-runtime.md) - what Gameface is and how the JS
  bundle is loaded and executed.
- Concept: [UI <-> C# communication](./ui-cs-communication.md) - the boundary from both sides
  at a higher level.
- Recipe: [UISystemBase React binding](../how-to/recipes/uisystembase-react-binding.md) - the
  **C# half**: registering the `ValueBinding`/`TriggerBinding` this page consumes.
- Case study: [write-everywhere ecosystem](../case-studies/write-everywhere-ecosystem.md) -
  the full mod this page's source is drawn from.
