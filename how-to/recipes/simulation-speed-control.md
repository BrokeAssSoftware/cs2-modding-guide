---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: Simulation-speed control via SimulationSystem.selectedSpeed"
recipe: simulation-speed-control
technique_family: "AV - Simulation-speed control via SimulationSystem.selectedSpeed"
diataxis: how-to
source_version: "~1.6.0f1 (advanced-simulation-speed@d224017; date-pinned, static source only)"
last_reverified: "2026-07-05"
status: source-verified
canonical_mods:
  - advanced-simulation-speed@d224017f15b6be7384ffcc34528553f86b22280d
technique_applicability: [simulation, ui]
Created: 2026-07-05
Updated: 2026-07-05
Owners:
  - codex
---

# Simulation-speed control via SimulationSystem.selectedSpeed

> Read and set the game's simulation speed - beyond the vanilla 1x/2x/3x buttons -
> by writing the vanilla `SimulationSystem.selectedSpeed` field from a UI system, with
> no Harmony and no prefab edits.

## Problem
You want a mod that changes how fast the simulation runs - custom speed presets,
finer stepping, a "doubled" mode, or just a live speed readout - and you want to drive
it from a React panel. The vanilla speed control is a fixed three-button set; there is
no settings field for "run at 5x" or "step by 0.25". You need to reach the actual speed
value the game simulates at, read it for a display, and write new values back safely.

## Solution
`Game.Simulation.SimulationSystem` exposes a public `selectedSpeed` field (the speed the
user has chosen) and a `smoothSpeed` field (the eased, currently-applied speed). Grab the
system once with `GetOrCreateSystemManaged<SimulationSystem>()`, wrap it in a mod-owned
property, and expose getter bindings + write triggers to your UI. Reads are plain field
reads; writes are either **absolute** (`setSpeed` sets `selectedSpeed = v`) or **relative
stepped** (`selectSpeed` computes a new value, snaps through 1x, and clamps to `[0, 8]`).
No patching - `selectedSpeed` is a writable public field, so the whole technique is field
access from an ordinary `UISystemBase`.

## Steps & Code

### 1. Resolve `SimulationSystem` and wrap the field

Get the managed `SimulationSystem` once in `OnCreate`, then wrap `selectedSpeed` in a
property so every read/write goes through one place:

```csharp
private SimulationSystem _simulationSystem;

private float SelectedSpeed
{
    get => _simulationSystem.selectedSpeed;
    set => _simulationSystem.selectedSpeed = value;
}

private float GetActualSpeed() => _simulationSystem.smoothSpeed;
private float GetSelectedSpeed() => _simulationSystem.selectedSpeed;
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-simulation-speed/repo/Systems/AdvancedSimSpeedUISystem.cs#L22-L31` (@d224017f15b6be7384ffcc34528553f86b22280d)

```csharp
_simulationSystem = World.GetOrCreateSystemManaged<SimulationSystem>();
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-simulation-speed/repo/Systems/AdvancedSimSpeedUISystem.cs#L47` (@d224017f15b6be7384ffcc34528553f86b22280d)

Note the two distinct fields: `selectedSpeed` is what the user picked; `smoothSpeed` is
the eased value the sim is actually running at right now. Expose `smoothSpeed` for a live
readout, `selectedSpeed` for the setting.

### 2. Expose a mod-owned getter binding for the speed

Bind a getter that reads `selectedSpeed` directly. This is the key move for a robust
readout - your UI reads the field the sim actually uses, not a frontend proxy:

```csharp
CreateBinding("selectedSpeed", GetSelectedSpeed);
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-simulation-speed/repo/Systems/AdvancedSimSpeedUISystem.cs#L54` (@d224017f15b6be7384ffcc34528553f86b22280d)

### 3. Add an absolute-write trigger

The simplest write path: a `Trigger<float>` that sets the speed directly. Because it goes
through the `SelectedSpeed` setter, one line covers preset buttons ("set 4x"):

```csharp
CreateTrigger<float>("setSpeed", v => SelectedSpeed = v);
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-simulation-speed/repo/Systems/AdvancedSimSpeedUISystem.cs#L63` (@d224017f15b6be7384ffcc34528553f86b22280d)

### 4. Add a relative stepping trigger with snap-at-1x and clamp

For +/- buttons, compute a new speed from the old one, then **snap through 1x** (if a step
would cross the 1.0 boundary, land exactly on 1x first) and **clamp to `[0, 8]`**:

```csharp
CreateTrigger<float>("selectSpeed", n =>
{
    var oldSpeed = GetSelectedSpeed();
    var newSpeed = Mod.setting.Type switch
    {
        Type.Doubled => SelectedSpeed * math.pow(2, n),
        Type.Fixed   => SelectedSpeed + (n * ((oldSpeed, n) switch
        {
            (_, 0)         => 0,
            (1, > 0)       => Mod.setting.Step - 1,       // stepping UP off exactly 1x
            (1, < 0)       => Mod.setting.IfLessThanOne,  // stepping DOWN off exactly 1x
            ( > 1, _)      => Mod.setting.Step,           // coarse step above 1x
            ( < 1, _)      => Mod.setting.IfLessThanOne,  // fine step below 1x
            (var a, var b) => throw new Exception($"Unreachable: ({a}, {b})")
        })),
        var e => throw new Exception($"Invalid enum value {e}")
    };
    if (math.min(oldSpeed, newSpeed) < 1 && math.max(oldSpeed, newSpeed) > 1)
    {
        newSpeed = 1;                       // snap onto exactly 1x when crossing it
    }
    SelectedSpeed = math.clamp(newSpeed, 0, 8);
});
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-simulation-speed/repo/Systems/AdvancedSimSpeedUISystem.cs#L64-L86` (@d224017f15b6be7384ffcc34528553f86b22280d)

The `Type.Fixed` step is a nested `switch` on `(oldSpeed, n)` that picks `Mod.setting.Step`
above 1x and `Mod.setting.IfLessThanOne` below it (with `-1`/`+` fudges at the boundary),
so stepping is coarse at high speed and fine below 1x
(`AdvancedSimSpeedUISystem.cs#L70-L78`).

### 5. Poll `smoothSpeed` on an interval for a live telemetry binding

`smoothSpeed` changes every frame, so instead of a per-frame binding push, throttle it. The
mod ticks a small `IntervalTimer` from `OnUpdate` and refreshes the `actualSpeed` binding
every 500 ms:

```csharp
_actualSpeed = CreateBinding("actualSpeed", GetActualSpeed());
_IntervalTimer = IntervalTimer.EveryMilliseconds(500, () => _actualSpeed.Value = GetActualSpeed());
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-simulation-speed/repo/Systems/AdvancedSimSpeedUISystem.cs#L57-L58` (@d224017f15b6be7384ffcc34528553f86b22280d)

The timer accumulates `deltaTime` and fires when it reaches its duration; `Trigger()` runs
the action and resets the accumulator, so it re-arms cleanly:

```csharp
public void Trigger()
{
    _elapsed = 0;
    _action?.Invoke();
}

public void Update(float deltaTime)
{
    if (!_isRunning) return;
    _elapsed += deltaTime;
    if (_elapsed >= _duration) { Trigger(); }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-simulation-speed/repo/Util/IntervalTimer.cs#L46-L62` (@d224017f15b6be7384ffcc34528553f86b22280d)

Drive it from the system's `OnUpdate` (this is a `UISystemBase`, so `OnUpdate` runs on the
UI update tick):

```csharp
protected override void OnUpdate()
{
    _IntervalTimer.Update(World.Time.DeltaTime);
    _isLegacyUI.Value = _interfaceSettings.useLegacyInterface;
    base.OnUpdate();
}
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-simulation-speed/repo/Systems/AdvancedSimSpeedUISystem.cs#L92-L97` (@d224017f15b6be7384ffcc34528553f86b22280d)

### 6. Theme the UI to match the active interface skin

Read `SharedSettings.instance.userInterface.useLegacyInterface` and bind it so the React
side can pick classic-vs-current glyphs. It is refreshed each `OnUpdate` (line 95 above)
because the user can switch skins at runtime:

```csharp
InterfaceSettings _interfaceSettings = SharedSettings.instance.userInterface;
// ...
_isLegacyUI = CreateBinding("legacyUI", _interfaceSettings.useLegacyInterface);
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-simulation-speed/repo/Systems/AdvancedSimSpeedUISystem.cs#L36,L60` (@d224017f15b6be7384ffcc34528553f86b22280d)

## Pitfalls & gotchas

- **Read `selectedSpeed`/`smoothSpeed`, not the frontend speed binding.** Expose a
  mod-owned getter off `_simulationSystem.selectedSpeed`
  (`AdvancedSimSpeedUISystem.cs#L31,L54`). Do not source your readout from the vanilla
  frontend `time.simulationSpeed$` binding: while the game is paused that value reflects
  the pre-pause speed, not the live selected speed, so a mod reading it would show the
  wrong number when paused. The exact vanilla value returned while paused is not visible
  in this mod's source - **Needs Verification (in-game)** - but the mod deliberately binds
  its own getter off the field rather than the frontend proxy.

- **The snap-through-1x uses STRICT inequalities.** The guard is
  `math.min(old,new) < 1 && math.max(old,new) > 1`
  (`AdvancedSimSpeedUISystem.cs#L81-L84`). A step that lands *exactly* on the boundary (a
  value of 1) does not satisfy `> 1`, so it is not snapped. That is fine here because
  landing on 1 is already the desired snap target, but if you copy this logic with a
  different snap point, remember exact-boundary values fall through unsnapped.

- **`Fixed` mode can step down to 0x (full stop).** The final write is
  `math.clamp(newSpeed, 0, 8)` (`AdvancedSimSpeedUISystem.cs#L85`) - the lower bound is
  **0**, not a positive minimum. In `Type.Fixed`, repeatedly stepping down subtracts from
  `selectedSpeed` and can reach 0, which sets the simulation to a dead stop. If you do not
  want a 0x "paused-but-not-paused" state, clamp the lower bound above 0 or special-case
  it on your buttons.

- **A `switch` write-trigger with a missing case THROWS.** Both the mode `switch`
  (`var e => throw new Exception($"Invalid enum value {e}")`, line 79) and the nested
  step `switch` (`throw new Exception($"Unreachable...")`, line 77) throw on an
  unhandled tuple. A trigger action that throws surfaces as an exception on the write
  path. Guard invalid states on the **frontend** (do not send a bad speed) *and* add a
  backend no-op default so an unexpected value degrades to "do nothing" instead of
  throwing.

- **Throttle `smoothSpeed`, don't bind it per frame.** `smoothSpeed` eases every tick;
  pushing it to the UI each frame is wasted churn. The mod polls it on a 500 ms
  `IntervalTimer` (`AdvancedSimSpeedUISystem.cs#L58`). Whether the game clamps `selectedSpeed`
  writes internally to any range beyond this mod's own `[0,8]` clamp is **Needs
  Verification (in-game)**.

## Variations

- **Doubled vs Fixed stepping.** `Type.Doubled` multiplies by `2^n`
  (`SelectedSpeed * math.pow(2, n)`), giving geometric 1x -> 2x -> 4x steps;
  `Type.Fixed` adds a configurable linear step. A third mode, `Type.Display`, is a
  read-only readout (the enum is `Doubled, Fixed, Display`)
  (`../../../vice-and-order-research/mods/dossiers/advanced-simulation-speed/repo/Domain/Type.cs#L3-L8`,
  @d224017f15b6be7384ffcc34528553f86b22280d).

- **Absolute-only control.** If you only need preset buttons, skip the stepping trigger
  entirely and expose just `setSpeed` (step 3) plus the `selectedSpeed` getter. The
  snap/clamp logic only matters for relative +/- stepping.

- **Display-only widget.** For a HUD that shows speed without changing it, bind
  `smoothSpeed` (via the interval poller) and `selectedSpeed` (getter) and register no
  write triggers - a pure telemetry panel.

## See also
- Related recipes: [vanilla UI augmentation](vanilla-ui-augmentation.md) (adding your
  panel to the game HUD), [time-tick-rate override](time-tick-rate-override.md) (changing
  how much sim-time each tick advances, a different lever than playback speed).
- Explanation: [React UI](../../explanation/react-ui.md) (how bindings/triggers connect
  C# systems to the frontend).
- Case study demonstrating it: [advanced-simulation-speed](../../case-studies/advanced-simulation-speed.md).

## Sources
- Canonical mods (dossier + repo):
  - `advanced-simulation-speed` @d224017f15b6be7384ffcc34528553f86b22280d - `repo/Systems/AdvancedSimSpeedUISystem.cs`, `repo/Util/IntervalTimer.cs`, `repo/Domain/Type.cs`
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding
