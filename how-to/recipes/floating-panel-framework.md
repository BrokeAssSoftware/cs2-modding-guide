---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: Draggable / resizable / collapsible floating-panel framework"
recipe: floating-panel-framework
technique_family: "AW - Draggable / resizable / collapsible floating-panel framework"
diataxis: how-to
source_version: "~1.6.0f1 (extra-lib@4879487; date-pinned, static source only)"
last_reverified: "2026-07-05"
status: source-verified
canonical_mods:
  - extra-lib@4879487b7df62e83676030a87f92d4b102955eee
technique_applicability: [ui]
Created: 2026-07-05
Updated: 2026-07-05
Owners:
  - codex
---

# Draggable / resizable / collapsible floating-panel framework

> Build a real movable/resizable/collapsible floating window - not just a
> fixed-position panel - by splitting persistent state (location, size,
> expanded, full-screen) onto a C# `UISystemBase` subclass and driving the
> window chrome (drag handle, resize edges, header buttons, z-order) from
> React, wired together with triggers and one raw binding.

## Problem
The basic runtime-UI recipe stops at "register a fixed panel": you get a React
component bound to a system, but the window cannot be moved, resized, or rolled
up, and its position does not survive a mode change. A floating tool window
needs authoritative, server-side state (where it is, how big, whether it is
open/expanded/full-screen), a way for the UI to push interactive changes back,
and a discipline for *when* the panel bothers to recompute its contents. This
recipe is the build-your-own-window-chrome layer on top of a plain UI system.

## Solution
Model each window as a subclass of an abstract base system that owns the
floating-window state and serialises it to the UI. Extra Lib's `ExtraPanelBase`
holds `PanelLocation`, `PanelSize`, `PanelMinSize`, `IsExpanded` and
`IsFullScreen`, exposes them through `IJsonWritable.Write`, and receives
interactive edits through triggers (`Open`/`Close`, `Collapse`/`Expand`,
`SetFullScreen`, `LocationChanged`, `SizeChanged`) fanned out by a single
manager system (`ExtraPanelsUISystem`). A panel is only "visible" when it is
*both* shown and expanded, so a collapsed panel skips its content update
entirely; `OnProcess` runs only when the panel is visible **and** dirty, so
subclasses call `RequestUpdate()` whenever their data changes. The React side
renders the chrome: a drag handle on the header, resize handles on the edges,
header buttons for collapse/full-screen/close, and per-panel z-index for
bring-to-front.

## Steps & Code

### 1. Subclass the base panel system and implement `OnProcess`

Your window is a `UISystemBase` (via `ExtraPanelBase`). Its identity is its
type name, and it must implement the single abstract hook `OnProcess`:

```csharp
public abstract partial class ExtraPanelBase : UISystemBase, IJsonWritable
{
    public string ID => GetType().FullName;
    public override GameMode gameMode => GameMode.Game;
    public virtual string Icon => "Media/Placeholder.svg";
    ...
    protected abstract void OnProcess();
}
```
Source: `../../../vice-and-order-research/mods/dossiers/extra-lib/repo/MOD/Systems/UI/ExtraPanels/ExtraPanelBase.cs#L8-L57` (@4879487b7df62e83676030a87f92d4b102955eee)

A minimal concrete panel just overrides `gameMode`, optionally opts into
full-screen, and implements `OnProcess`:

```csharp
internal partial class TestExtraPanel : ExtraPanelBase
{
    public override GameMode gameMode => GameMode.Game | GameMode.Editor;
    protected override bool m_CanFullScreen => true;
    protected override void OnProcess() { }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/extra-lib/repo/MOD/Systems/UI/ExtraPanels/TestExtraPanel.cs#L11-L26` (@4879487b7df62e83676030a87f92d4b102955eee)

### 2. Own the floating-window state on the base class

The base class holds location/size/expanded/full-screen as private-set
properties plus a minimum size. Note `PanelMinSize` is virtual so a subclass
can raise the floor:

```csharp
public float2 PanelLocation { get; private set; }
public float2 PanelSize { get; private set; }
public virtual float2 PanelMinSize => new float2(48, 48);

public bool IsExpanded { get; private set; } = true;
public bool IsFullScreen { get; private set; } = false;
```
Source: `../../../vice-and-order-research/mods/dossiers/extra-lib/repo/MOD/Systems/UI/ExtraPanels/ExtraPanelBase.cs#L24-L29` (@4879487b7df62e83676030a87f92d4b102955eee)

### 3. Serialise the panel state to the UI via `IJsonWritable`

The panel writes itself as a typed object keyed by `ID`; every field the React
chrome needs (visibility, expansion, full-screen, location, size, min size)
goes over the wire here:

```csharp
public void Write(IJsonWriter writer)
{
    writer.TypeBegin(ID);
    writer.PropertyName("visible");      writer.Write(m_Visible);
    writer.PropertyName("isExpanded");   writer.Write(IsExpanded);
    writer.PropertyName("canFullScreen");writer.Write(m_CanFullScreen);
    writer.PropertyName("isFullScreen"); writer.Write(IsFullScreen);
    writer.PropertyName("panelLocation");writer.Write(PanelLocation);
    writer.PropertyName("panelSize");    writer.Write(PanelSize);
    writer.PropertyName("panelMinSize"); writer.Write(PanelMinSize);
    writer.TypeEnd();
}
```
Source: `../../../vice-and-order-research/mods/dossiers/extra-lib/repo/MOD/Systems/UI/ExtraPanels/ExtraPanelBase.cs#L59-L82` (@4879487b7df62e83676030a87f92d4b102955eee) (abridged)

### 4. Gate `OnProcess` behind visible-and-dirty; expose `RequestUpdate()`

This is the update-cost discipline. `Visible()` is `m_Visible && IsExpanded`,
so a collapsed panel is treated as not visible and its `OnProcess` never runs.
And even when visible, `OnProcess` fires only when `m_Dirty`, which the panel
sets via `RequestUpdate()`:

```csharp
public void PerformUpdate()
{
    Update();
    if (Visible())
    {
        Reset();
        OnPreProcess();
        if (m_Dirty) { m_Dirty = false; OnProcess(); }
    }
}
public void RequestUpdate() { m_Dirty = true; }
public bool Visible() { return m_Visible && IsExpanded; }
```
Source: `../../../vice-and-order-research/mods/dossiers/extra-lib/repo/MOD/Systems/UI/ExtraPanels/ExtraPanelBase.cs#L38-L103` (@4879487b7df62e83676030a87f92d4b102955eee) (abridged)

`SetVisible` and `SetExpanded` both call `RequestUpdate()` on the way *in* (so a
re-shown or re-expanded panel recomputes once), then ask the manager to refresh
the binding:

```csharp
public void SetVisible(bool visible)
{
    if (visible == m_Visible) return;
    m_Visible = visible;
    if (visible) RequestUpdate();
    m_ExtraPanelsUISystem.RequestBindingUpdate();
}
```
Source: `../../../vice-and-order-research/mods/dossiers/extra-lib/repo/MOD/Systems/UI/ExtraPanels/ExtraPanelBase.cs#L89-L110` (@4879487b7df62e83676030a87f92d4b102955eee) (abridged)

### 5. Register triggers on the manager system

`ExtraPanelsUISystem` owns one `RawValueBinding` that writes the panel array,
plus one trigger per interactive action. This is the full control surface:

```csharp
AddBinding(m_PanelsBinding = new RawValueBinding("el", "ExtraPanels", WritePanels));
AddBinding(new TriggerBinding<string>("el", "OpenExtraPanel", OpenExtraPanel));
AddBinding(new TriggerBinding<string>("el", "CloseExtraPanel", CloseExtraPanel));
AddBinding(new TriggerBinding<string>("el", "CollapseExtraPanel", CollapseExtraPanel));
AddBinding(new TriggerBinding<string>("el", "ExpandExtraPanel", ExpandExtraPanel));
AddBinding(new TriggerBinding<string, bool>("el", "SetFullScreenExtraPanel", SetFullScreenExtraPanel));
AddBinding(new TriggerBinding<string, float2>("el", "LocationChanged", UpdatePanelLocation));
AddBinding(new TriggerBinding<string, float2>("el", "SizeChanged", UpdatePanelSize));
```
Source: `../../../vice-and-order-research/mods/dossiers/extra-lib/repo/MOD/Systems/UI/ExtraPanels/ExtraPanelsUISystem.cs#L62-L72` (@4879487b7df62e83676030a87f92d4b102955eee)

Every trigger resolves the target panel by its `ID` string. `FindPanelByID`
matches on `ExtraPanelBase.ID == id`, so the panel type name is the routing key:

```csharp
public ExtraPanelBase FindPanelByID(string id)
{
    return m_ValidPanels.Find((ExtraPanelBase) => { return ExtraPanelBase.ID == id; });
}
```
Source: `../../../vice-and-order-research/mods/dossiers/extra-lib/repo/MOD/Systems/UI/ExtraPanels/ExtraPanelsUISystem.cs#L161-L164` (@4879487b7df62e83676030a87f92d4b102955eee)

### 6. Clamp SIZE server-side; leave LOCATION to the client

`SizeChanged` clamps the incoming size up to `PanelMinSize` before storing it -
this is the authoritative floor. `LocationChanged` does **not** clamp: it stores
whatever the client sent:

```csharp
private void UpdatePanelSize(string id, float2 newSize)
{
    if (!TryToFindPanelByID(id, out ExtraPanelBase extraPanelBase)) { ...return; }
    if (newSize.x < extraPanelBase.PanelMinSize.x || newSize.y < extraPanelBase.PanelMinSize.y)
        newSize = math.max(newSize, extraPanelBase.PanelMinSize);
    extraPanelBase.SetPanelSize(newSize);
}
```
Source: `../../../vice-and-order-research/mods/dossiers/extra-lib/repo/MOD/Systems/UI/ExtraPanels/ExtraPanelsUISystem.cs#L237-L250` (@4879487b7df62e83676030a87f92d4b102955eee) (abridged)

### 7. Drive the panels each frame from the manager's `OnUpdate`

The manager iterates only the *valid* panels (those matching the current game
mode) and calls `PerformUpdate()`; the raw binding is re-pushed only when marked
dirty:

```csharp
protected override void OnUpdate()
{
    base.OnUpdate();
    UpdatePanels();                    // calls PerformUpdate() on each valid panel
    if (m_DirtyPanelBinding)
    {
        m_DirtyPanelBinding = false;
        m_PanelsBinding.Update();
        ...
    }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/extra-lib/repo/MOD/Systems/UI/ExtraPanels/ExtraPanelsUISystem.cs#L83-L102` (@4879487b7df62e83676030a87f92d4b102955eee) (abridged)

### 8. On the React side, define the trigger wrappers keyed by `__Type`

Each C# trigger has a thin TS wrapper that fires it with the panel's `__Type`
(the serialised `ID`). Location/size changes push `Number2` payloads back:

```typescript
export function SetPanelPosition(p: ExtraPanelType, newPos: Number2) { trigger("el", "LocationChanged", p.__Type, newPos) }
export function SetPanelSize(p: ExtraPanelType, newSize: Number2)    { trigger("el", "SizeChanged", p.__Type, newSize) }
export const CollapseExtraPanel = (p: ExtraPanelType) => { trigger("el", "CollapseExtraPanel", p.__Type) }
export const ExpandExtraPanel   = (p: ExtraPanelType) => { trigger("el", "ExpandExtraPanel", p.__Type) }
export const SetFullScreenExtraPanel = (p: ExtraPanelType, fs: boolean) => { trigger("el", "SetFullScreenExtraPanel", p.__Type, fs) }
```
Source: `../../../vice-and-order-research/mods/dossiers/extra-lib/repo/UI/src/mods/ExtraPanels/ExtraPanelType.tsx#L17-L27` (@4879487b7df62e83676030a87f92d4b102955eee)

### 9. Render the chrome: drag handle, resize, and clamp location client-side

The window component wraps its header in a drag handle, enables edge resizing
(disabled when full-screen), and commits the new location/size on drag/resize
*end*. Location is clamped to the viewport here, in the client:

```typescript
// inside getTranslate(): clamp the panel origin to the viewport
var clampedPanelPosx = Math.min(Math.max(finalPanelX, 0), window.innerWidth);
var clampedPanelPosy = Math.min(Math.max(finalPanelY, 0), window.innerHeight);
...
const onDragEnd = (b: BetterDragEventData) => {
    const { x, y } = getTranslate(b);
    extraPanel.panelLocation = { x: extraPanel.panelLocation.x + x, y: extraPanel.panelLocation.y + y };
    SetPanelPosition(extraPanel, extraPanel.panelLocation);   // -> LocationChanged trigger
};
```
Source: `../../../vice-and-order-research/mods/dossiers/extra-lib/repo/UI/src/mods/ExtraPanels/ExtraPanel/ExtraPanel.tsx#L41-L66` (@4879487b7df62e83676030a87f92d4b102955eee) (abridged)

The header's drag comes from a custom `BetterDragHandle` (NOT `@dnd-kit`) that
clones the game's own `useMouseDragEvents` hook onto the child's `onMouseDown`:

```typescript
const { handleMouseDown } = useMouseDragEvents({ handleDragStart, handleDragging, handleDragEnd });
return <>{React.Children.map(children, (e) =>
    React.isValidElement(e)
        ? React.cloneElement(e as any, { onMouseDown: handleMouseDown })
        : <div onMouseDown={handleMouseDown}>{e}</div>)}</>;
```
Source: `../../../vice-and-order-research/mods/dossiers/extra-lib/repo/UI/src/mods/Utilities/BetterDragHandle.tsx#L54-L64` (@4879487b7df62e83676030a87f92d4b102955eee)

Resizing is the reusable `Panel` component's `useResize`: it installs
document-level `mousemove`/`mouseup` listeners, clamps width/height to
`[min,max]`, and reports origin deltas when dragging a left/top edge so the
window can shift as it shrinks:

```typescript
const clampedW = Math.min(Math.max(newW, opts.minWidth), opts.maxWidth);
const clampedH = Math.min(Math.max(newH, effectiveMinHeight), opts.maxHeight);
if (direction.includes("w")) deltaX = startW - clampedW;   // move origin on left-edge resize
if (direction.includes("n")) deltaY = startH - clampedH;   // move origin on top-edge resize
return { width: clampedW, height: clampedH, deltaX, deltaY };
```
Source: `../../../vice-and-order-research/mods/dossiers/extra-lib/repo/UI/src/mods/ExtraPanels/ExtraPanel/Panel.tsx#L164-L209` (@4879487b7df62e83676030a87f92d4b102955eee) (abridged)

### 10. Bring-to-front z-order and collapse via header buttons

The root renderer keeps a per-`__Type` z-index map and bumps the clicked panel
above the current max on `onBringToFront`:

```typescript
const bringToFront = useCallback((panelId: string) => {
    setZIndices(prev => {
        const currentMax = Math.max(10, ...Object.values(prev));
        if (prev[panelId] === currentMax) return prev;
        zCounterRef.current = currentMax + 1;
        return { ...prev, [panelId]: zCounterRef.current };
    });
}, []);
```
Source: `../../../vice-and-order-research/mods/dossiers/extra-lib/repo/UI/src/mods/ExtraPanels/ExtraPanelsRoot/ExtraPanelsRoot.tsx#L20-L45` (@4879487b7df62e83676030a87f92d4b102955eee) (abridged)

The header's collapse toggle fires `Collapse`/`Expand` and, like every chrome
button, swallows `onMouseDown` so a button press does not also start a drag:

```typescript
onSelect={() => extraPanel.isExpanded ? CollapseExtraPanel(extraPanel) : ExpandExtraPanel(extraPanel)}
onMouseDown={(e) => { e.preventDefault(); e.stopPropagation(); }}
```
Source: `../../../vice-and-order-research/mods/dossiers/extra-lib/repo/UI/src/mods/ExtraPanels/ExtraPanel/Header/ExtraPanelHeader.tsx#L39-L47` (@4879487b7df62e83676030a87f92d4b102955eee)

## Pitfalls & gotchas

- **A collapsed panel stops recomputing.** `Visible()` is `m_Visible &&
  IsExpanded`, and `PerformUpdate` runs `OnProcess` only when `Visible()` is
  true. So rolling a panel up doesn't just hide the body - it freezes the panel:
  no `OnProcess`, no data refresh. That is the intended cost-saving, but if your
  subclass needs to keep computing while collapsed, this framework will not do
  it for you (`ExtraPanelBase.cs#L38-L103`).

- **You MUST call `RequestUpdate()` or `OnProcess` never fires again.**
  `OnProcess` is gated on `m_Dirty`, which is set true only by `RequestUpdate()`
  (and implicitly by `SetVisible(true)`/`SetExpanded(true)`). A subclass that
  changes its data without calling `RequestUpdate()` will render stale content
  forever (`ExtraPanelBase.cs#L44-L49, #L84-L87`).

- **Panel identity is `GetType().FullName` - one instance per type.** `ID`
  is the type's full name (`ExtraPanelBase.cs#L10`) and `FindPanelByID` matches
  on it (`ExtraPanelsUISystem.cs#L161-L164`). Every trigger, the React
  `__Type`, and the z-index map all key off it. Two panels of the same concrete
  type would share one ID and collide; give each window its own subclass.

- **Location is only clamped in the client, not on the server.**
  `SizeChanged` clamps up to `PanelMinSize` in C#
  (`ExtraPanelsUISystem.cs#L245-L249`), but `LocationChanged` stores the raw
  `float2` with no bounds check (`ExtraPanelsUISystem.cs#L225-L235`). The
  on-screen clamp to `window.innerWidth/innerHeight` happens only in
  `ExtraPanel.tsx#L41-L42`. Anything that fires `LocationChanged` directly (not
  through the drag handle) can push a window off-screen.

- **The drag handle is custom, not a library.** Despite the "@dnd-kit"
  shorthand you may see referenced, this framework uses `BetterDragHandle` built
  on the game's `useMouseDragEvents`
  (`BetterDragHandle.tsx#L2, #L54`), and resize is a hand-rolled `useResize`
  with document-level listeners (`Panel.tsx#L136-L209`). There is no external
  drag/resize dependency to add.

- **Chrome buttons must stop propagation.** Header buttons (collapse,
  full-screen, close) each call `e.preventDefault(); e.stopPropagation()` in
  `onMouseDown` (`ExtraPanelHeader.tsx#L44-L47`) precisely because the header is
  also the drag handle; without it, clicking a button would begin a drag.

- **`OnProcess` gate depends on the `Reset()`/`OnPreProcess()` order.**
  `PerformUpdate` calls `Reset()` then `OnPreProcess()` before the dirty check,
  every frame the panel is visible (`ExtraPanelBase.cs#L41-L49`). Heavy work
  belongs in `OnProcess` (dirty-gated), not `OnPreProcess`/`Reset` (run every
  visible frame). Whether the per-frame `Reset`/`OnPreProcess` cost matters in
  practice is `Needs Verification (in-game)`.

- **Persistence of location/size across sessions is not shown here.** The state
  lives on the system instance; nothing in the cited source serialises it to a
  save or settings file. Survival across game restart is `Needs Verification
  (in-game)`.

## Variations

- **Opt into full-screen.** Override `m_CanFullScreen => true` on the subclass
  (`TestExtraPanel.cs#L15`); `SetFullScreen` is a no-op unless that flag is set
  (`ExtraPanelBase.cs#L111-L116`). Dragging a full-screen window auto-exits
  full-screen first (`ExtraPanel.tsx#L54-L58`).

- **Raise the minimum size per window.** Override `PanelMinSize` (it is
  `virtual`, default `48x48`); the server clamp in `SizeChanged` then enforces
  your floor (`ExtraPanelBase.cs#L26`, `ExtraPanelsUISystem.cs#L245-L249`).

- **Register a fixed panel instead.** If you don't need drag/resize/collapse,
  skip this framework and register a plain bound panel - see
  [runtime UI](../ui/runtime-ui.md). This recipe is specifically the
  window-chrome layer above that.

- **Restrict a window to certain game modes.** Override `gameMode` on the
  subclass; the manager only ticks panels whose mode intersects the current
  one via `IsValidPanel` (`ExtraPanelsUISystem.cs#L118-L122`), so a
  `Game`-only window silently drops out in the editor.

## See also
- Related recipes:
  [runtime UI (register a panel)](../ui/runtime-ui.md),
  [UISystemBase React binding](uisystembase-react-binding.md).
- Explanation: [React UI in CS2](../../explanation/react-ui.md).
- Reference: [Extra Lib shared library](../../reference/shared-libraries/extralib.md);
  [technique index](../../technique-index.md) (family AW).

## Sources
- Canonical mods (dossier + repo):
  - `extra-lib` @4879487b7df62e83676030a87f92d4b102955eee -
    `repo/MOD/Systems/UI/ExtraPanels/ExtraPanelBase.cs`,
    `repo/MOD/Systems/UI/ExtraPanels/ExtraPanelsUISystem.cs`,
    `repo/MOD/Systems/UI/ExtraPanels/TestExtraPanel.cs`,
    `repo/UI/src/mods/ExtraPanels/ExtraPanelType.tsx`,
    `repo/UI/src/mods/ExtraPanels/ExtraPanel/ExtraPanel.tsx`,
    `repo/UI/src/mods/ExtraPanels/ExtraPanel/Panel.tsx`,
    `repo/UI/src/mods/ExtraPanels/ExtraPanel/Header/ExtraPanelHeader.tsx`,
    `repo/UI/src/mods/ExtraPanels/ExtraPanelsRoot/ExtraPanelsRoot.tsx`,
    `repo/UI/src/mods/Utilities/BetterDragHandle.tsx`
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding
