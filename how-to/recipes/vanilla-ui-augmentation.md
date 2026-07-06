---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: Augment vanilla UI via moduleRegistry.extend"
recipe: vanilla-ui-augmentation
technique_family: "AU - Augment vanilla UI via moduleRegistry.extend (toolbar / button / InfoSection / nested categories)"
diataxis: how-to
source_version: "~1.6.0f1 (advanced-simulation-speed@d224017; date-pinned, static source only)"
last_reverified: "2026-07-05"
status: source-verified
canonical_mods:
  - advanced-simulation-speed@d224017f15b6be7384ffcc34528553f86b22280d
  - smooth-left-hand-traffic@37b2850b10ced1cd17457c25778409e57e00a03e
  - extra-lib@4879487b7df62e83676030a87f92d4b102955eee
technique_applicability: [ui]
Created: 2026-07-05
Updated: 2026-07-05
Owners:
  - codex
---

# Augment vanilla UI via moduleRegistry.extend

> Decorate a vanilla toolbar field, tool-options panel, or category tab bar by wrapping
> the game's own React component instead of replacing it - so your addition rides on top
> of the vanilla element and survives its restyling.

## Problem
You want to add to an existing vanilla panel - a readout next to the time-controls
field, an extra button in a tool's options row, a nested tab in the asset category bar -
without forking the vanilla component. Mounting a brand-new panel (`moduleRegistry.append`)
puts your UI *somewhere else*; you specifically want your control to appear *inside* the
vanilla element, keyed to its layout and lifecycle. You also must not break when the game
re-authors that panel between patches.

## Solution
`moduleRegistry.extend(modulePath, exportName, hoc)` registers a **higher-order
component**: the game calls your `hoc` with the vanilla `Component`, and you return a new
component that renders the vanilla one plus your additions. This is decoration, not
replacement - the original still renders; you wrap or append around it. The HOC is typed
`ModuleRegistryExtend` (`(Component) => (props) => JSX`). Two support calls make it
practical: `getModule(path, exportName)` pulls a real vanilla sub-component (e.g. the
toolbar `Divider`) or its SCSS `classes` map so your addition matches the vanilla look,
and you register the **same** HOC against both the legacy and the rewritten module path so
one build covers both toolbar layouts. For a Selected-Info panel section the parallel
managed-side tool is `ExtendedInfoSectionBase` (see Variations).

## Steps & Code

### 1. Register the HOC against the vanilla module (toolbar field)

`register` (the `ModRegistrar` default export) receives the `moduleRegistry`. Call
`extend` with the vanilla module path, the export name, and your HOC. Advanced Simulation
Speed registers the **same** `AdvancedTimeControls` HOC twice - against the legacy
`time-controls.tsx` / `TimeControls` and the 1.6 `time-controls-new.tsx` / `TimeControlsNew`:

```tsx
const register: ModRegistrar = (moduleRegistry) => {
  moduleRegistry.extend(
    "game-ui/game/components/toolbar/bottom/time-controls/time-controls.tsx",
    "TimeControls",
    AdvancedTimeControls
  );
  moduleRegistry.extend(
    "game-ui/game/components/toolbar/bottom/time-controls/time-controls-new.tsx",
    "TimeControlsNew",
    AdvancedTimeControls
  );
}
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-simulation-speed/repo/UI/src/index.tsx#L5-L20` (@d224017f15b6be7384ffcc34528553f86b22280d)

### 2. Pull real vanilla sub-components / SCSS with `getModule`

To match the vanilla field's look, resolve the vanilla `Divider` component and the
field's SCSS `classes` map from their modules. These helpers wrap `getModule` from
`cs2/modding` and cast the result:

```ts
export const getModuleComponent = <Props = any>(
  modulePath: string,
  exportName: string,
) => getModule(modulePath, exportName) as Component<Props>;

export const getModuleClasses = <T extends object = object>(
  modulePath: string,
) => getModule(modulePath, "classes") as Classes<T>;
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-simulation-speed/repo/UI/src/mods/utils/utils.ts#L5-L13` (@d224017f15b6be7384ffcc34528553f86b22280d)

```ts
const fieldStyles = getModuleClasses<{ field: any; content: any }>(
  "game-ui/game/components/toolbar/components/field/field.module.scss",
);
const Divider = getModuleComponent(toolbarFieldPath, "Divider");
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-simulation-speed/repo/UI/src/mods/simSpeedComponent.tsx#L12-L15` (@d224017f15b6be7384ffcc34528553f86b22280d)

### 3. Write the HOC: render the vanilla component, then your addition

The HOC signature is `ModuleRegistryExtend = (Component) => (props) => JSX`. Split
`children` off `props`, render `<Component>` unchanged, and place your decoration next to
it - here a sibling `<div>` carrying the prev/next buttons and the speed readout, styled
with the vanilla `fieldStyles.field` class so it sits flush with the toolbar:

```tsx
export const AdvancedTimeControls: ModuleRegistryExtend =
  (Component) => (props) => {
    const { children, ...otherProps } = props || {};
    // ...bindings + button JSX (left / center / right) omitted...
    return (
      <>
        <Component {...otherProps}>{children}</Component>
        <div className={classNames(styles.simSpeedControls, {[fieldStyles.field]: legacyUI, ...})}>
          {!displayOnlyMode && left}
          {center}
          {!displayOnlyMode && right}
        </div>
      </>
    );
  };
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-simulation-speed/repo/UI/src/mods/simSpeedComponent.tsx#L30-L106` (@d224017f15b6be7384ffcc34528553f86b22280d)

### 4. Tool button: resolve vanilla exports up front, throw if missing

For a tool-options button, Smooth Left-Hand Traffic resolves the vanilla `Section`,
`ToolButton`, and the tool-button SCSS class from the registry *before* extending, and
throws a labelled error if any export is absent - so a vanilla rename fails loudly at
registration instead of rendering a broken panel:

```tsx
function getRegistryExport<T>(moduleRegistry, modulePath, exportName): T {
    const resolvedValue = moduleRegistry.get(modulePath, exportName);
    if (!resolvedValue) {
        throw new Error(`[SmoothLHT] Missing vanilla UI export ${exportName} from ${modulePath}`);
    }
    return resolvedValue as T;
}

const register: ModRegistrar = (moduleRegistry) => {
    const toolOptionsComponents = resolveToolOptionsComponents(moduleRegistry);
    moduleRegistry.extend(
        MOUSE_TOOL_OPTIONS_MODULE, "MouseToolOptions",
        createInvertLHTTool(toolOptionsComponents)
    );
};
```
Source: `../../../vice-and-order-research/mods/dossiers/smooth-left-hand-traffic/repo/SmoothLHT/UI/SmoothLHT/src/index.tsx#L8-L34` (@37b2850b10ced1cd17457c25778409e57e00a03e)

### 5. Tool button HOC: call the vanilla component, then append a child section

This HOC differs from step 3: it *invokes* the vanilla `Component(props)` to get its
rendered element, then clones it with an extra child (`cloneElement` + `Children.toArray`)
so your `<Section>` appends into the vanilla options row. It also early-returns the
untouched result when the tool is not showing:

```tsx
function withAppendedToolSection(result, section) {
    if (!isValidElement(result)) return result;
    const nextChildren = [...Children.toArray(result.props?.children), section];
    return cloneElement(result, { ...result.props, children: nextChildren });
}

export function createInvertLHTTool(components): ModuleRegistryExtend {
    return (Component) => (props) => {
        const result = Component(props) as ExtendableComponentResult;
        if (!useValue(SHOW_BINDING)) return result;
        return withAppendedToolSection(result, <InvertToggleSection .../>);
    };
}
```
Source: `../../../vice-and-order-research/mods/dossiers/smooth-left-hand-traffic/repo/SmoothLHT/UI/SmoothLHT/src/mods/InvertLHTTool.tsx#L52-L110` (@37b2850b10ced1cd17457c25778409e57e00a03e)

### 6. Nested categories: feed the vanilla tab bar from a C# binding

Decorating the category tab bar happens on the managed side. Extra Lib's
`AssetMultiCategory` (a `UISystemBase`) publishes a `RawValueBinding` under the vanilla
`"el"` group so the vanilla toolbar UI reads extra category rows from your JSON:

```csharp
AddBinding(_AssetMultiCategoriesBinding = new RawValueBinding(
    "el", "AssetMultiCategories", new Action<IJsonWriter>(this.WriteAssetMultiCategories)));
AddBinding(_SelectedAssetMultiCategoriesBinding = new GetterValueBinding<List<Entity>>(
    "el", "SelectedAssetMultiCategories", () => _SelectedAssetMultiCategories, new ListWriter<Entity>()));
```
Source: `../../../vice-and-order-research/mods/dossiers/extra-lib/repo/MOD/Systems/UI/AssetMultiCategory.cs#L49-L50` (@4879487b7df62e83676030a87f92d4b102955eee)

Each row is emitted as a `toolbar.AssetCategory` typed object (entity, name, icon, locked,
uiTag, highlight) - the same shape vanilla category tabs use - so the tab bar renders them
natively:

```csharp
writer.TypeBegin("toolbar.AssetCategory");
writer.PropertyName("entity"); writer.Write(entity);
writer.PropertyName("name");   writer.Write(prefab.name);
writer.PropertyName("icon");   writer.Write(ImageSystem.GetIcon(prefab) ?? _ImageSystem.placeholderIcon);
writer.PropertyName("locked"); writer.Write(base.EntityManager.HasEnabledComponent<Locked>(entity));
writer.PropertyName("uiTag");  writer.Write(prefab.uiTag);
writer.TypeEnd();
```
Source: `../../../vice-and-order-research/mods/dossiers/extra-lib/repo/MOD/Systems/UI/AssetMultiCategory.cs#L96-L109` (@4879487b7df62e83676030a87f92d4b102955eee)

The binding is refreshed off Harmony post-fixes on the vanilla `ToolbarUISystem`
selection methods, so your rows track the player's live tab selection:

```csharp
[HarmonyPatch(typeof(ToolbarUISystem), "SelectAssetMenu")]
class SelectAssetMenu {
    static void Postfix(Entity assetMenu) {
        if (assetMenu != Entity.Null && EL.m_EntityManager.HasComponent<UIAssetMenuData>(assetMenu))
            AssetMultiCategory.instance.OnSelectAssetMenu(assetMenu);
    }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/extra-lib/repo/MOD/Patches/ToolbarUISystemPatch.cs#L13-L29` (@4879487b7df62e83676030a87f92d4b102955eee)

## Pitfalls & gotchas

- **`extend` vs `append`.** `moduleRegistry.extend` is an HOC that *decorates* the named
  vanilla export - the vanilla component still renders and yours wraps or appends around
  it. `append` mounts a wholly new element elsewhere. If your control must appear *inside*
  a vanilla panel keyed to its layout, `extend` is the only option.

- **Dual legacy + new registration is required through the 1.6 toolbar rewrite.** The
  time-controls field was re-authored: the same HOC must be registered against *both*
  `time-controls.tsx`/`TimeControls` and `time-controls-new.tsx`/`TimeControlsNew`
  (step 1). Register only the legacy path and your field silently vanishes on the new
  toolbar; register only the new path and it vanishes on any layout still serving the
  legacy module. `legacyUI$` is bound so the same component styles itself for whichever
  path rendered it (`simSpeedComponent.tsx#L24`).

- **Unguarded `getModule` casts degrade silently on a vanilla rename.** Advanced
  Simulation Speed's `getModuleComponent(toolbarFieldPath, "Divider")` and the
  `field.module.scss` `classes` pull are cast with no null check
  (`utils.ts#L5-L13`, `simSpeedComponent.tsx#L12-L15`). If a patch renames the `Divider`
  export or the SCSS class, the cast yields `undefined`, the vanilla styling drops, and
  the wrapper renders unstyled or throws at render time with no registration-time warning.
  Smooth LHT's `getRegistryExport` throw-on-missing (step 4) is the defensive counterpart -
  prefer it when a missing export should fail loud.

- **Vanilla-shape coupling.** The nested-category path emits `toolbar.AssetCategory` with
  the exact field set vanilla expects and reflects into private `ToolbarUISystem` members
  via Harmony `Traverse` (`AssetMultiCategory.cs#L29-L32, #L249-L262`). A vanilla rename of
  those fields/methods breaks it at runtime, not compile time.

- **Whether the decorated element renders at the intended position / z-order in the live
  1.6 toolbar is `Needs Verification (in-game)`** - the source shows the wrapping JSX and
  bindings but not the final on-screen placement.

## Variations

- **Append a child instead of a sibling.** Step 3 renders the vanilla component and a
  *sibling* `<div>`; step 5 instead invokes `Component(props)` and `cloneElement`s an extra
  *child* into it. Use the sibling form to sit beside a self-contained vanilla field; use
  the child-append form to inject into a container that maps over its `children`
  (`InvertLHTTool.tsx#L52-L62`).

- **Add a Selected-Info panel section (managed side).** For a section in the
  Selected-Info panel rather than the toolbar, subclass `ExtendedInfoSectionBase`, put it in
  the vanilla group with `[UpdateInGroup(typeof(SelectedInfoUISystem))]`, and register it with
  `AddMiddleSection(this)` in `OnCreate`. The override surface is small:

  ```csharp
  [UpdateInGroup(typeof(SelectedInfoUISystem))]
  public partial class RoadSpeedToolUISystem : ExtendedInfoSectionBase
  {
      protected override string group => "RoadSpeedAdjuster.Systems.RoadSpeedToolUISystem"; // unique group id
      protected override bool displayForUnderConstruction => false;

      protected override void OnCreate()
      {
          base.OnCreate();
          m_InfoUISystem.AddMiddleSection(this);   // mount into the Selected-Info panel
      }

      protected override void OnProcess() { }       // per-selection recompute (empty here; work in OnUpdate)
      public override void OnWriteProperties(IJsonWriter writer) { /* push this section's bindings */ }
  }
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/Systems/RoadSpeedToolUISystem.cs#L24-L57` (@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed);
  `OnProcess`/`OnWriteProperties` at `#L161-L163,#L241`. The React side consumes the section's
  bindings exactly like the toolbar case - see [UISystemBase React binding](uisystembase-react-binding.md)
  for the `ValueBinding`/`TriggerBinding` wiring and the full worked example in the
  [road-speed-adjuster case study](../../case-studies/road-speed-adjuster.md).

- **Fail loud vs fail soft.** Choose per surface: `getRegistryExport`'s throw (step 4) for
  exports whose absence should abort registration; the silent cast (step 2) only where a
  missing vanilla piece can be tolerated as an unstyled fallback.

## See also
- Related recipes: [UISystemBase React binding](uisystembase-react-binding.md) (the C# side
  of a `RawValueBinding`/`GetterValueBinding` feeding UI, as in step 6).
- Reference: [module registry](../ui/module-registry.md) (extend / append / get semantics),
  [React UI](../../explanation/react-ui.md) (how the game's React tree is composed).
- Case studies demonstrating it: [advanced-simulation-speed](../../case-studies/advanced-simulation-speed.md).

## Sources
- Canonical mods (dossier + repo):
  - `advanced-simulation-speed` @d224017f15b6be7384ffcc34528553f86b22280d - `repo/UI/src/index.tsx`, `repo/UI/src/mods/simSpeedComponent.tsx`, `repo/UI/src/mods/utils/utils.ts`
  - `smooth-left-hand-traffic` @37b2850b10ced1cd17457c25778409e57e00a03e - `repo/SmoothLHT/UI/SmoothLHT/src/index.tsx`, `repo/SmoothLHT/UI/SmoothLHT/src/mods/InvertLHTTool.tsx`
  - `extra-lib` @4879487b7df62e83676030a87f92d4b102955eee - `repo/MOD/Systems/UI/AssetMultiCategory.cs`, `repo/MOD/Patches/ToolbarUISystemPatch.cs`
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding
