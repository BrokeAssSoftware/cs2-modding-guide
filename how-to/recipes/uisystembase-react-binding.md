---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: UISystemBase C# <-> React binding (ValueBinding / TriggerBinding)"
recipe: uisystembase-react-binding
technique_family: "P - UISystemBase C# <-> React binding (ValueBinding / TriggerBinding)"
diataxis: how-to
source_version: "~1.5.x (better-bulldozer@4408466; date-pinned, static source only)"
last_reverified: "2026-07-04"
status: source-verified
canonical_mods:
  - better-bulldozer@4408466f226db811159d92859479ae1e1c28ba06
  - write-everywhere@13c70eb04e6bed152257c516a982455148f591a5
  - traffic-tool-essentials@10973595ac9ed37f47ca24eb788d4421dc295fa0
  - road-speed-adjuster@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed
  - advanced-road-naming@559e72cdb3180e3e71869643367e094a22eed988
  - time2work-realistic-trips@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39
technique_applicability: [ui]
Created: 2026-07-04
Updated: 2026-07-05
Owners:
  - codex
---

# UISystemBase C# <-> React binding (ValueBinding / TriggerBinding)

> Expose C# state to your React/TSX panel with `ValueBinding<T>` (C# -> UI) and
> receive button clicks / value edits with `TriggerBinding` (UI -> C#), all
> registered from a `UISystemBase` subclass via `AddBinding`.

## Problem
Your mod has a custom panel written in React (TSX) and you need the two sides to
talk: push a live number/flag/string from simulation into the panel, and run C#
when the user clicks a button or drags a slider. The bridge is a set of **named
bindings** owned by a C# `UISystemBase` system. The catch that trips people up is
that mods do not all extend the same base class - some extend raw
`UISystemBase`, some ship a local `ExtendedUISystemBase` helper, and info-panel
mods extend `InfoSectionBase`. Picking the wrong base (or copying the wrong
`AddBinding` idiom) is the usual first failure.

## Solution
Create a system that inherits a UI base class, and in `OnCreate` register bindings
by `(group, name)`:

- **State out (C# -> React):** `ValueBinding<T>` holds a value; call `.Update(v)`
  to push a new value to the UI. `GetterValueBinding<T>` instead wraps a getter
  `Func<T>` and re-reads it whenever you call `.Update()` (no argument).
- **Events in (React -> C#):** `TriggerBinding` (no payload) or
  `TriggerBinding<T...>` (typed payload) invokes a C# `Action` when the UI fires
  `trigger(group, name, ...)`.

Every binding is registered with `AddBinding(...)`. The `group` + `name` you pass
in C# are the exact strings the React side uses in `useValue`/`trigger` (the
consume side is covered in [explanation/react-ui.md](../../explanation/react-ui.md)).
Which base class you extend is a real decision - see the gotcha below and pick
from the two-to-three worked examples here.

## Steps & Code

### 1. Subclass a UI base class

The simplest, most portable choice is to extend the game's `UISystemBase`
directly. Traffic Tool Essentials does exactly this:

```csharp
public partial class UISystem : UISystemBase
{
    // ...
    protected override void OnCreate()
    {
        base.OnCreate();
        // ... queries ...
        AddUIBindings();   // register all bindings (see step 2)
    }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Systems/UI/UISystem.cs#L21` and `#L148,#L191` (@10973595ac9ed37f47ca24eb788d4421dc295fa0)

### 2. Register value bindings (C# -> React) with `AddBinding`

A plain `ValueBinding<T>` is `new ValueBinding<T>(group, name, initialValue)`, then
handed to `AddBinding`. A `GetterValueBinding<T>` takes a getter delegate instead of
a stored value. Traffic Tool Essentials uses both:

```csharp
private void AddUIBindings()
{
    // getter-backed: re-reads GetMainPanel() each time you call .Update()
    AddBinding(m_MainPanelBinding = new GetterValueBinding<string>("C2VM.TLE", "GetMainPanel", GetMainPanel));
    // value-backed: holds an int, initial value -1
    AddBinding(m_ActiveEditingCustomPhaseIndexBinding = new ValueBinding<int>("C2VM.TLE", "GetActiveEditingCustomPhaseIndex", -1));
}
```
Source: `../../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Systems/UI/UISystem.UIBindings.cs#L101-L109` (@10973595ac9ed37f47ca24eb788d4421dc295fa0)

The first argument (`"C2VM.TLE"`) is the **group** and the second (`"GetMainPanel"`)
is the **name**; React reads this exact pair. A `GetterValueBinding` does not push on
its own - you call `m_MainPanelBinding.Update()` (no argument) whenever the underlying
data changes and it re-invokes the getter:

```csharp
protected string GetMainPanel() { /* build + return the panel JSON/string */ }
// ... later, after mutating state:
m_MainPanelBinding.Update();
```
Source: `../../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Systems/UI/UISystem.UIBindings.cs#L253` and `#L1099` (@10973595ac9ed37f47ca24eb788d4421dc295fa0)

### 3. Register trigger bindings (React -> C#)

A `TriggerBinding` runs an `Action` when the UI fires it; `TriggerBinding<T>` carries a
typed payload the UI sends. Same `(group, name)` addressing:

```csharp
// bool payload - runs the delegate with the value the UI sent
AddBinding(new TriggerBinding<bool>("C2VM.TLE", "SetDashboardActive", (active) => { m_GreenWaveDashboardActive = active; }));
// typed payload (Entity) - passed straight to a handler method
AddBinding(new TriggerBinding<Entity>("C2VM.TLE", "NavigateToIntersection", NavigateToIntersection));
```
Source: `../../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Systems/UI/UISystem.UIBindings.cs#L159` and `#L169` (@10973595ac9ed37f47ca24eb788d4421dc295fa0)

Write Everywhere shows the same idiom on a raw `UISystemBase` - one `TriggerBinding<int>`
that drives a panel switch:

```csharp
public partial class WEMainUISystem : UISystemBase
{
    // ...
    AddBinding(new TriggerBinding<int>("k45::we.main", "setTabActive", (x) => panelSystem.ShowPanel<WEMainPanel>(x)));
}
```
Source: `../../../vice-and-order-research/mods/dossiers/write-everywhere/repo/BelzontWE/Systems/WEMainUISystem.cs#L13` and `#L19` (@13c70eb04e6bed152257c516a982455148f591a5)

### 4. Push updated values back on user action

A value binding is only as live as your calls to `.Update(value)`. A trigger handler
typically reads the current value, flips it, and pushes it back so React re-renders.
Better Bulldozer's confirmation toggle is the canonical shape:

```csharp
private void BypassConfirmationToggled()
{
    m_BypassConfirmation.Update(!m_BypassConfirmation.value);
}
```
Source: `../../../vice-and-order-research/mods/dossiers/better-bulldozer/repo/BetterBulldozer/Systems/BetterBulldozerUISystem.cs#L465-L467` (@4408466f226db811159d92859479ae1e1c28ba06)

Read the current value off `.value` and push the next with `.Update(...)`; that single
call is what re-renders the panel.

## Pitfalls & gotchas

- **The base class VARIES - pick deliberately (this is the main trap).** Three
  live patterns, all in canonical mods:
  - **Raw `UISystemBase` + verbose `AddBinding`.** You construct every
    `ValueBinding`/`GetterValueBinding`/`TriggerBinding` by hand. Most portable,
    most boilerplate. Traffic Tool Essentials
    (`.../traffic-tool-essentials/repo/TrafficToolEssentials/Systems/UI/UISystem.cs#L21`)
    and Write Everywhere
    (`.../write-everywhere/repo/BelzontWE/Systems/WEMainUISystem.cs#L13`).
  - **Locally-shipped `ExtendedUISystemBase` helper.** A thin abstract subclass of
    `UISystemBase` that adds `CreateBinding`/`CreateTrigger` sugar so you skip the
    `AddBinding(new ...)` ceremony. Better Bulldozer ships its own:
    ```csharp
    public partial class BetterBulldozerUISystem : ExtendedUISystemBase { /* ... */ }
    ```
    Source: `../../../vice-and-order-research/mods/dossiers/better-bulldozer/repo/BetterBulldozer/Systems/BetterBulldozerUISystem.cs#L33` (@4408466f226db811159d92859479ae1e1c28ba06)

    The helper is a per-mod file, NOT a shared library - copying Better Bulldozer's
    call sites without its `Extensions/ExtendedUISystemBase.cs` will not compile.
  - **`ExtendedInfoSectionBase` (info-panel middle section).** For a section inside
    the vanilla selected-info panel, extend an `InfoSectionBase`-derived helper, not
    a bare `UISystemBase`. Road Speed Adjuster:
    ```csharp
    public partial class RoadSpeedToolUISystem : ExtendedInfoSectionBase
    ```
    Source: `../../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/Systems/RoadSpeedToolUISystem.cs#L25` (@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed)

- **`GetterValueBinding` does NOT auto-push.** It re-evaluates its getter only when
  you call `.Update()` (no argument). If your panel shows stale data, you forgot the
  `.Update()` call after mutating the backing state - Traffic Tool Essentials calls
  `m_MainPanelBinding.Update()` from every mutation path
  (`.../UISystem.UIBindings.cs#L1099`).

- **`group`/`name` are string contracts - a typo is a silent dead binding.** The C#
  `(group, name)` must match the React `useValue`/`trigger` strings exactly. There is
  no compile-time link between the two sides; a mismatch just never fires. Note the
  groups differ per mod: `"C2VM.TLE"`, `"k45::we.main"`, or the mod id passed by a
  helper - agree on the string once and reuse a constant.

- **Order: register in `OnCreate`, before the UI reads.** Read a value binding before
  its first `.Update()` and the game throws "update was not called before
  getValueUnsafe". Road Speed Adjuster seeds initial values and then calls
  `RequestUpdate()` in `OnCreate` specifically to avoid this
  (`.../road-speed-adjuster/repo/Systems/RoadSpeedToolUISystem.cs#L83-L92`). The exact
  runtime error text and timing is `Needs Verification (in-game)`.

- **The exact React-side hook names and re-render timing are UI-runtime behaviour.**
  What the TSX side must import and how fast a `.Update()` reaches the DOM is
  `Needs Verification (in-game)` from C# source alone - see
  [explanation/react-ui.md](../../explanation/react-ui.md) for the consume side.

- **Do NOT call `.Update()` every frame from `OnUpdate` - throttle it.** A value binding
  driven straight from `OnUpdate` re-serializes and re-pushes on every simulation frame,
  even when the payload rarely changes (a clock, a day-of-week label). Time2Work realistic
  trips wraps the push in a tiny time-accumulator so the string binding only updates about
  once per second:

  ```csharp
  public void Update(float delta)
  {
      Elapsed += delta;
      if (Elapsed >= Duration) { InvokeAction(); }   // resets Elapsed, runs Action
  }
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Utils/Throttle.cs#L38-L46` (@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39)

  It is created once in `OnCreate` around the binding's `.Update()` and pumped with
  `World.Time.DeltaTime` from `OnUpdate`:

  ```csharp
  _weekUpdateThrottle = Throttle.BySeconds(1, () => { _weekDay.Update(dateOutput); });
  // ... in OnUpdate:
  _weekUpdateThrottle.Update(World.Time.DeltaTime);
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Systems/Time2WorkUISystem.cs#L47` and `#L70` (@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39)

  `ValueBinding` already skips pushes when the value is equal (see the equality-comparer
  overload used below), but the throttle avoids rebuilding the payload at all - cheaper when
  the string is expensive to assemble.

## Variations

- **Helper sugar over raw `AddBinding` (Better Bulldozer).** The local
  `ExtendedUISystemBase` collapses the boilerplate into `CreateBinding`/`CreateTrigger`
  that build the binding and register it for you:

  ```csharp
  public ValueBindingHelper<T> CreateBinding<T>(string key, T initialValue)
  {
      var helper = new ValueBindingHelper<T>(new (BetterBulldozerMod.Id, key, initialValue, new GenericUIWriter<T>()));
      AddBinding(helper.Binding);
      return helper;
  }
  public GetterValueBinding<T> CreateBinding<T>(string key, Func<T> getterFunc) { /* AddBinding(...) */ }
  public TriggerBinding CreateTrigger(string key, Action action) { /* AddBinding(...) */ }
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/better-bulldozer/repo/BetterBulldozer/Extensions/ExtendedUISystemBase.cs#L11-L46` (@4408466f226db811159d92859479ae1e1c28ba06)

  The `ValueBindingHelper<T>` wraps `.value`/`.Update()` behind an ergonomic `.Value`
  property (`get => Binding.value; set => Binding.Update(value)`):

  ```csharp
  public T Value { get => Binding.value; set => Binding.Update(value); }
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/better-bulldozer/repo/BetterBulldozer/Extensions/ValueBindingHelper.cs#L16` (@4408466f226db811159d92859479ae1e1c28ba06)

  Call sites then read as one-liners: `AddBinding(m_RaycastTarget = new ValueBinding<int>(...))`
  for the raw path or `m_SelectionMode = CreateBinding("SelectionMode", ...)` for the sugar,
  and triggers as `AddBinding(new TriggerBinding(ModId, "BypassConfirmationButton", BypassConfirmationToggled))`
  (`.../BetterBulldozerUISystem.cs#L302-L327`).

- **Info-section binding with a group prefix (Road Speed Adjuster).** The
  `ExtendedInfoSectionBase` variant namespaces every binding with `BINDING:` /
  `TRIGGER:` prefixes and auto-pairs a setter trigger with each value binding:

  ```csharp
  var helper  = new ValueBindingHelper<T>(new(Mod.Id, $"BINDING:{key}", initialValue, new GenericUIWriter<T>()), updateCallBack);
  var trigger = new TriggerBinding<T>(Mod.Id, $"TRIGGER:{key}", helper.UpdateCallback, new GenericUIReader<T>());
  AddBinding(helper.Binding);
  AddBinding(trigger);
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/Extensions/ExtendedInfoSectionBase.cs#L20-L39` (@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed)

  Usage stays terse - `_initialSpeedBinding = CreateBinding("INFOPANEL_ROAD_SPEED", 50f)`
  and `CreateTrigger<float>("APPLY_SPEED", HandleApplySpeed)` - and the section declares
  its `group` override so the panel routes to it
  (`.../RoadSpeedToolUISystem.cs#L47,#L66-L79`).

- **Request/response with `CallBinding<TIn,TResult>` + a static clipboard (Traffic Tool
  Essentials).** When the UI must ship *structured* ECS data to C#, act on it, and read a
  result back synchronously, a `TriggerBinding` (fire-and-forget, no return) is the wrong
  shape. Traffic Tool Essentials instead registers `CallBinding<string, string>` handlers -
  the UI passes a JSON string, C# deserializes it, mutates ECS buffers, and returns a JSON
  result - and keeps the copied phases in a plain `static` field across calls, exposing only
  the *count* to React through a separate `ValueBinding<int>`:

  ```csharp
  private static List<CopiedPhaseData> m_PhaseClipboard = new List<CopiedPhaseData>();
  private ValueBinding<int> m_ClipboardCountBinding;
  // ... in the binding-registration method:
  AddBinding(m_ClipboardCountBinding = new ValueBinding<int>("C2VM.TLE", "GetClipboardCount", 0));
  AddBinding(new CallBinding<string, string>("C2VM.TLE", "CallCopyPhases", CallCopyPhases));
  AddBinding(new CallBinding<string, string>("C2VM.TLE", "CallPastePhases", CallPastePhases));
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Systems/UI/UISystem.UIBindings.cs#L39-L40` and `#L140-L142` (@10973595ac9ed37f47ca24eb788d4421dc295fa0)

  The copy handler reads the selected entity's `CustomPhaseData`/`EdgeGroupMask` buffers into
  the static list, then pushes the new count so any "paste" button can enable itself:

  ```csharp
  protected string CallCopyPhases(string jsonString)
  {
      var input = JsonConvert.DeserializeObject<PhaseIndicesInput>(jsonString);
      // ... fill m_PhaseClipboard from EntityManager.TryGetBuffer<CustomPhaseData>(...) ...
      m_ClipboardCountBinding.Update(m_PhaseClipboard.Count);
      return JsonConvert.SerializeObject(new { success = true, count = m_PhaseClipboard.Count });
  }
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Systems/UI/UISystem.UIBindings.cs#L1675-L1725` (@10973595ac9ed37f47ca24eb788d4421dc295fa0)

  The `static` clipboard survives across separate `CallBinding` invocations (and even tool
  re-selection); the `ValueBinding<int>` is the only piece React observes, so the heavy ECS
  payload never crosses the bridge as a live value.

- **One `ValueBinding<string>` for the whole panel state (Advanced Road Naming).** At the
  opposite extreme from many small bindings: a selected-info tool serializes its *entire*
  panel state into a single pipe-delimited string and pushes it through ONE
  `ValueBinding<string>`, paired with ~18 `TriggerBinding`s for the inbound commands - no
  `GetterValueBinding` anywhere.

  ```csharp
  _stateBinding = new ValueBinding<string>(PanelBindingGroup, "state", _lastState,
      ValueWriters.Create<string>(), System.Collections.Generic.EqualityComparer<string>.Default);
  AddBinding(_stateBinding);
  AddBinding(new TriggerBinding(PanelBindingGroup, "activate", ActivateTool));
  AddBinding(new TriggerBinding<string>(PanelBindingGroup, "setMode", SetMode, ValueReaders.Create<string>()));
  AddBinding(new TriggerBinding<long>(PanelBindingGroup, "selectSavedRoute", SelectSavedRoute, ValueReaders.Create<long>()));
  // ... ~18 triggers total, all under the same PanelBindingGroup ...
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Systems/RoadRouteToolUISystem.cs#L31-L51` (@559e72cdb3180e3e71869643367e094a22eed988)

  `OnUpdate` rebuilds the string with `string.Join("|", new[] { Escape(...), ... })` and only
  calls `_stateBinding.Update(state)` when it differs from `_lastState`; the passed
  `EqualityComparer<string>` makes that diff cheap. Each field is run through an `Escape`
  helper that replaces `|` with `\p` so the delimiter never collides with data:

  ```csharp
  return string.Join("|", new[] { Escape("1"), Escape(_toolSystem?.Mode.ToString()), /* ... */ });
  // Escape: (value ?? "").Replace("\\", "\\\\").Replace("|", "\\p").Replace("\n", "\\n") ...
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Systems/RoadRouteToolUISystem.cs#L313-L336` and `#L443` (@559e72cdb3180e3e71869643367e094a22eed988)

  This trades type-safety and the "one binding per field" convention for a single push per
  frame-that-changed and a trivially versioned wire format (field 0 is a "panel open" flag),
  at the cost of a matching split/unescape parser on the TSX side (`Needs Verification
  (in-game)` - the consume-side parser is not in this C# source).

## See also
- Consume side (TSX): [explanation/react-ui.md](../../explanation/react-ui.md).
- Why the split exists: [explanation/ui-cs-communication.md](../../explanation/ui-cs-communication.md).
- Wiring the module into the UI: [how-to/ui/module-registry.md](../ui/module-registry.md).
- Case studies: [better-bulldozer](../../case-studies/better-bulldozer.md),
  [write-everywhere-ecosystem](../../case-studies/write-everywhere-ecosystem.md).
- Reference: [technique index](../../technique-index.md) (family P coverage ledger).

## Sources
- Canonical mods (dossier + repo):
  - `better-bulldozer` @4408466f226db811159d92859479ae1e1c28ba06 - `repo/BetterBulldozer/Systems/BetterBulldozerUISystem.cs`, `repo/BetterBulldozer/Extensions/ExtendedUISystemBase.cs`, `repo/BetterBulldozer/Extensions/ValueBindingHelper.cs`
  - `traffic-tool-essentials` @10973595ac9ed37f47ca24eb788d4421dc295fa0 - `repo/TrafficToolEssentials/Systems/UI/UISystem.cs`, `repo/TrafficToolEssentials/Systems/UI/UISystem.UIBindings.cs`
  - `road-speed-adjuster` @e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed - `repo/Systems/RoadSpeedToolUISystem.cs`, `repo/Extensions/ExtendedInfoSectionBase.cs`
  - `write-everywhere` @13c70eb04e6bed152257c516a982455148f591a5 - `repo/BelzontWE/Systems/WEMainUISystem.cs`
  - `advanced-road-naming` @559e72cdb3180e3e71869643367e094a22eed988 - `repo/Systems/RoadRouteToolUISystem.cs`
  - `time2work-realistic-trips` @d42921ffb1f6bbbf2c2b086bb17727bf0f637c39 - `repo/NightShift/Utils/Throttle.cs`, `repo/NightShift/Systems/Time2WorkUISystem.cs`
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding
