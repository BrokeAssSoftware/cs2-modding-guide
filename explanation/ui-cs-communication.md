---
FrontmatterVersion: 1
DocumentType: Guide
Title: C#/React Communication (UISystemBase Bindings)
Summary: The two-way binding model CS2 mods use to connect C# to a Gameface/React UI - ValueBinding pushes typed state C#-to-React, TriggerBinding is a React-to-C# RPC by name - explained against real UISystemBase mod source.
diataxis: explanation
source_version: "~1.5.x (better-bulldozer@4408466; date-pinned, static source only)"
last_reverified: "2026-07-04"
canonical_mods:
  - better-bulldozer@4408466f226db811159d92859479ae1e1c28ba06
  - road-speed-adjuster@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed
status: source-verified
Created: 2026-07-03
Updated: 2026-07-03
Owners:
  - codex
References:
  - Label: The Gameface UI Runtime (why the boundary exists)
    Path: ./gameface-runtime.md
  - Label: Options attributes (declarative settings, no bindings)
    Path: ../reference/options-attributes.md
  - Label: Technique Index
    Path: ../technique-index.md
---

# C#/React Communication (UISystemBase Bindings)

A Cities: Skylines II mod's UI runs in the Gameface sandbox and cannot touch the simulation
directly (see [The Gameface UI Runtime](./gameface-runtime.md)). React and C# never share
memory. They communicate only through an explicit, named **binding** channel, and every value
that crosses it is serialized (marshalled as JSON).

This page explains that channel: the two directions of data flow, the two binding primitives
that carry them, and the small helper layer most mods build on top. It is a concept page - the
"how the two halves of a mod talk" - not a step-by-step recipe.

## The model: two directions, two primitives

The binding channel is deliberately asymmetric, because the two directions do different jobs:

| Direction | Primitive | What it is | Analogy |
| --- | --- | --- | --- |
| C# -> React | **`ValueBinding<T>`** | A named, typed piece of state C# *publishes*. C# calls `.Update(newValue)` and every React reader re-renders. | Server-pushed state / a prop |
| React -> C# | **`TriggerBinding`** | A named C# callback React can *invoke*, optionally with arguments. | A remote procedure call (RPC) |

Both live in `Colossal.UI.Binding` and are registered from a C# system that derives from the
game's `UISystemBase`. The mental model is: **C# owns the state and publishes a projection of
it; React renders that projection and asks C# to change things by name.** The UI is never the
source of truth.

## Where the bindings are registered

A mod's UI-facing logic lives in a `UISystemBase` subclass. It declares its published state as
`ValueBinding<T>` fields, and in `OnCreate` it registers each binding (and each trigger) with
`AddBinding(...)`. Better Bulldozer's UI system is a clear example: typed `ValueBinding<int>` /
`ValueBinding<bool>` fields for tool state, registered alongside `TriggerBinding` callbacks for
each button
([`better-bulldozer` `repo/BetterBulldozer/Systems/BetterBulldozerUISystem.cs#L49-L60,#L280-L326`](../../vice-and-order-research/mods/dossiers/better-bulldozer/repo/BetterBulldozer/Systems/BetterBulldozerUISystem.cs), commit `4408466f`):

```csharp
// Fields: the state this system publishes to React
private ValueBinding<int> m_RaycastTarget;
private ValueBinding<bool> m_BypassConfirmation;
// ...

protected override void OnCreate()
{
    base.OnCreate();
    // C# -> React: publish named, typed state with default values.
    AddBinding(m_RaycastTarget = new ValueBinding<int>(ModId, "RaycastTarget", (int)RaycastTarget.Vanilla));
    AddBinding(m_BypassConfirmation = new ValueBinding<bool>(ModId, "BypassConfirmation", false));

    // React -> C#: register callbacks React can invoke by name.
    AddBinding(new TriggerBinding(ModId, "BypassConfirmationButton", BypassConfirmationToggled));
    AddBinding(new TriggerBinding(ModId, "GameplayManipulationButton", GameplayManipulationToggled));
}
```

Each binding is keyed by a **(mod id, name)** pair. That name string is the contract with the
React side: the TypeScript reads the value or calls the trigger by exactly that name. There is
no compiler enforcing that the C# name and the React name match - a typo is a silent no-op, one
of the most common UI bugs.

## C# -> React: publishing and updating state

A `ValueBinding<T>` starts at the default value you pass in its constructor. To push a new value
to React, call `.Update(newValue)`; every React component subscribed to that name re-renders
with the new value. Better Bulldozer's toggle handlers do exactly this - they mutate a bulldoze
tool field and then `Update` the binding so the UI reflects it
([`better-bulldozer` `repo/BetterBulldozer/Systems/BetterBulldozerUISystem.cs#L465-L482`](../../vice-and-order-research/mods/dossiers/better-bulldozer/repo/BetterBulldozer/Systems/BetterBulldozerUISystem.cs)):

```csharp
private void BypassConfirmationToggled()
{
    m_BypassConfirmation.Update(!m_BypassConfirmation.value);
    m_BulldozeToolSystem.debugBypassBulldozeConfirmation = m_BypassConfirmation.value;
}
```

`.value` reads the current value; `.Update(...)` sets it and notifies React. Publish only what
the UI needs, and keep it small - every update is serialized across the boundary.

## React -> C#: triggers as RPCs

A `TriggerBinding` names a C# callback React can invoke. The parameterless form maps a button
click to a method; generic forms (`TriggerBinding<T1>`, `TriggerBinding<T1,T2>`, ...) carry
arguments the UI supplies. In the block above, `"BypassConfirmationButton"` is invoked from
React when the player clicks the corresponding UI button, and C# runs `BypassConfirmationToggled`.

Triggers are how the UI *asks* the simulation to do something. The UI never mutates game state
itself; it fires a named trigger and lets C# decide what happens. That keeps the simulation
authoritative and the UI a thin projection.

## The common helper layer: CreateBinding / CreateTrigger

Registering `new ValueBinding<T>(...)` and `new TriggerBinding(...)` by hand is verbose, so most
mods wrap `UISystemBase` in a small base class that exposes `CreateBinding` / `CreateTrigger`
helpers. Better Bulldozer's `ExtendedUISystemBase` is representative: `CreateBinding<T>` builds
the binding, registers it, and returns a helper you can assign; `CreateTrigger` does the same
for a trigger, with overloads for 0-4 arguments
([`better-bulldozer` `repo/BetterBulldozer/Extensions/ExtendedUISystemBase.cs#L11-L82`](../../vice-and-order-research/mods/dossiers/better-bulldozer/repo/BetterBulldozer/Extensions/ExtendedUISystemBase.cs)):

```csharp
public abstract partial class ExtendedUISystemBase : UISystemBase
{
    public ValueBindingHelper<T> CreateBinding<T>(string key, T initialValue) { /* builds + AddBinding */ }
    public TriggerBinding CreateTrigger(string key, Action action) { /* builds + AddBinding */ }
    public TriggerBinding<T1> CreateTrigger<T1>(string key, Action<T1> action) { /* ... */ }
}
```

Road Speed Adjuster uses the same helper shape from its info-panel UI system. Its `OnCreate`
reads as a compact manifest of the whole C#/React contract - state bindings on top, triggers
below
([`road-speed-adjuster` `repo/Systems/RoadSpeedToolUISystem.cs#L66-L82`](../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/Systems/RoadSpeedToolUISystem.cs), commit `e0c0c0b3`):

```csharp
_initialSpeedBinding    = CreateBinding("INFOPANEL_ROAD_SPEED", 50f);
_toolActiveBinding      = CreateBinding("TOOL_ACTIVE", false);
_selectionCounterBinding = CreateBinding("SELECTION_COUNTER", 0);
// ...
CreateTrigger<float>("APPLY_SPEED", HandleApplySpeed);   // React sends a float; C# applies it
CreateTrigger("RESET_SPEED", HandleResetSpeed);
CreateTrigger<bool>("ACTIVATE_TOOL", HandleActivateTool);
```

Here `"APPLY_SPEED"` shows the round trip end to end: C# publishes the current speed via
`_initialSpeedBinding`, React lets the player edit it, and React fires the `"APPLY_SPEED"`
trigger with the new float, which C# receives in `HandleApplySpeed`. The helper's returned
object (`ValueBindingHelper<T>`) exposes a `.Value` setter as an ergonomic alternative to
`.Update(...)`; Road Speed Adjuster seeds its bindings that way right after creating them
(`repo/Systems/RoadSpeedToolUISystem.cs#L83-L88`).

## Marshalling: it is JSON, so keep it small

The boundary is a serialization boundary, not a shared-object one. Binding values are written
through an `IJsonWriter` - Better Bulldozer's `GenericUIWriter<T>.Write(IJsonWriter, T)` is the
concrete proof
([`better-bulldozer` `repo/BetterBulldozer/Extensions/GenericUIWriter.cs#L15-L17`](../../vice-and-order-research/mods/dossiers/better-bulldozer/repo/BetterBulldozer/Extensions/GenericUIWriter.cs)).
The design consequences:

- **Send primitives and small records, not simulation graphs.** Every `Update` serializes the
  whole value. Deep or large objects cost per publish.
- **Publish derived, display-ready data.** Do the shaping in C# and push a small, flat result -
  the UI should not have to reconstruct state from raw components.
- **Batch where you can.** Prefer one binding carrying a small struct over many chatty bindings
  updated separately each frame.

## Common pitfalls

- **Name mismatch between C# and React.** The binding name is a stringly-typed contract; a typo
  compiles fine and silently does nothing. Keep names in shared constants where possible and
  check them first when a binding "doesn't fire".
- **Reading a binding before its first update.** React reading a value that C# has not published
  yet can error. Road Speed Adjuster seeds values and forces an update in `OnCreate` explicitly
  to avoid an "update was not called before getValueUnsafe" error
  (`road-speed-adjuster/repo/Systems/RoadSpeedToolUISystem.cs#L83-L92`). Publish sensible
  defaults up front.
- **Treating the UI as authoritative.** Never store gameplay truth only in React; fire a trigger
  and let C#/ECS own the change, then publish the result back.
- **Over-publishing every frame.** A `ValueBinding` updated each tick with an unchanged value is
  wasted serialization. Update only when the value actually changes.
- **The React-side half lives on its own page.** How the TypeScript subscribes to these names
  (the `useValue` / `bindValue` / `trigger` / `engine.on` API surface, the `cs2/api` and `cs2/ui`
  imports, and `moduleRegistry`) is source-verified from a real frontend in
  [React UI (consume side)](./react-ui.md) - read it for the exact call signatures rather than
  guessing. The C#-side `(group, name)` binding contract on this page and the TSX consumer there
  must use the identical strings.

## See also

- [The Gameface UI Runtime](./gameface-runtime.md) - why the C#/React boundary exists and why
  it is a serialized channel.
- [Options attributes](../reference/options-attributes.md) - the declarative settings route that
  needs no bindings at all.
- [Technique Index](../technique-index.md) - coverage ledger; this is family P (`UISystemBase` /
  React binding), canonical mods better-bulldozer, road-speed-adjuster, write-everywhere,
  traffic-tool-essentials.
