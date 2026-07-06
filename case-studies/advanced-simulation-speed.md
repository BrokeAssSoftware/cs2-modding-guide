---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Case study: Advanced Simulation Speed"
case_study: advanced-simulation-speed
mod: "Advanced Simulation Speed"
dossier: ../../vice-and-order-research/mods/dossiers/advanced-simulation-speed/
repo_commit: d224017f15b6be7384ffcc34528553f86b22280d
source_version: "1.6.0f1 (advanced-simulation-speed@d224017; date-pinned, static source only)"
last_reverified: "2026-07-05"
diataxis: explanation
techniques: [AU, AV, AX]
technique_applicability: [simulation, ui]
status: source-verified
Created: 2026-07-05
Updated: 2026-07-05
Owners:
  - codex
---

# Advanced Simulation Speed - case study

> A no-Harmony time-control mod that writes the vanilla speed field directly,
> grafts custom stepping onto the toolbar clock via `moduleRegistry.extend`, and
> keeps a live actual-speed readout on a 500 ms dirty-flag binding - a compact
> tour of how far you can push time control and toolbar UI without patching a
> single method.

All code claims below are cited to `repo/<path>#Lxx` at pinned commit
`d224017f15b6be7384ffcc34528553f86b22280d`, surfaced through the dossier at
`../../vice-and-order-research/mods/dossiers/advanced-simulation-speed/`. Only the
mod's own source is cited; vanilla `SimulationSystem` internals are referenced
only as the mod uses them.

## What it does / why it's instructive

Advanced Simulation Speed (ASS, namespace `AdvancedSimulationSpeed`) replaces the
game's three fixed speed buttons with a stepper that can hold non-integer speeds,
double or add fixed increments, snap through 1x, and optionally show the *actual*
smoothed simulation speed next to the *selected* one. It ships two speed modes
(`Doubled`, `Fixed`) plus a passive `Display` mode that only reads out speed with
no controls (repo/Domain/Type.cs#L3-L8).

It is instructive because it is a **maximal example of the "touch nothing you
don't have to" school**: no Harmony, no decompiled clones, no new simulation
system. The entire mod is one `UISystemBase` scheduled into `UIUpdate` that
read/writes `SimulationSystem.selectedSpeed` directly, plus a React component
grafted onto the vanilla toolbar clock. That minimalism is exactly what makes its
maintenance seams sharp: because it leans on vanilla field names and vanilla UI
module paths, the 1.6 toolbar rewrite is visible in the code as a *pair* of
registrations, and several unguarded casts are one Colossal rename away from
breaking. It is a clean study of the three UI-side technique families - direct
speed control, toolbar augmentation, and read-only telemetry binding - combined
in one small mod.

## Architecture at a glance

There is exactly one system. `Mod.OnLoad` registers settings, loads locales, then
schedules the single UI system into `SystemUpdatePhase.UIUpdate`
(repo/Mod.cs#L40) - not a simulation phase, because it only reads/writes state and
drives bindings:

```csharp
updateSystem.UpdateAt<AdvancedSimSpeedUISystem>(SystemUpdatePhase.UIUpdate);
```
(repo/Mod.cs#L40)

`AdvancedSimSpeedUISystem` extends a small `ExtendedUISystemBase` binding helper
(repo/Systems/AdvancedSimSpeedUISystem.cs#L13; repo/Extensions/ExtendedUISystemBase.cs#L9).
It resolves the managed `SimulationSystem` in `OnCreate`
(repo/Systems/AdvancedSimSpeedUISystem.cs#L47) and exposes the vanilla speed as a
property pair - this is the whole of the time-control mechanism, no patching:

```csharp
private float SelectedSpeed
{
    get => _simulationSystem.selectedSpeed;
    set => _simulationSystem.selectedSpeed = value;
}
private float GetActualSpeed() => _simulationSystem.smoothSpeed;
private float GetSelectedSpeed() => _simulationSystem.selectedSpeed;
```
(repo/Systems/AdvancedSimSpeedUISystem.cs#L24-L31)

Data flow per frame: `OnUpdate` ticks the interval timer with `World.Time.DeltaTime`
and re-pushes the interface flag; the base `OnUpdate` then flushes every dirty
binding (repo/Systems/AdvancedSimSpeedUISystem.cs#L92-L97;
repo/Extensions/ExtendedUISystemBase.cs#L13-L21). The React side reads the
`selectedSpeed` / `actualSpeed` bindings and calls the `selectSpeed` trigger back
into the system (repo/UI/src/mods/simSpeedComponent.tsx#L19-L28).

Two binding *kinds* coexist and the distinction matters:

- `selectedSpeed` is a `GetterValueBinding` (the `Func<T>` overload), polled every
  frame via `AddUpdateBinding` - always live
  (repo/Systems/AdvancedSimSpeedUISystem.cs#L54; repo/Extensions/ExtendedUISystemBase.cs#L47-L54).
- `actualSpeed` is a `ValueBindingHelper<float>` dirty-flag binding, written only
  when the 500 ms timer fires and flushed once per frame
  (repo/Systems/AdvancedSimSpeedUISystem.cs#L57-L58;
  repo/Extensions/ValueBindingHelper.cs#L15-L39).

## Techniques demonstrated

- [Simulation-speed control via `SimulationSystem.selectedSpeed`](../how-to/recipes/simulation-speed-control.md)
  (family AV) - the mod writes the vanilla speed field directly through the
  `SelectedSpeed` property and computes the next value in the `selectSpeed`
  trigger: `Doubled` multiplies by `2^n`, `Fixed` adds a mode-dependent step, then
  it snaps through 1x and clamps to `[0, 8]`
  (repo/Systems/AdvancedSimSpeedUISystem.cs#L64-L86). Crucially it exposes its own
  `selectedSpeed` binding rather than the vanilla `time.simulationSpeed$`, because
  the latter returns `m_SpeedBeforePause` while paused - the component's own
  comment documents this (repo/UI/src/mods/simSpeedComponent.tsx#L17-L19).

```csharp
if (math.min(oldSpeed, newSpeed) < 1 && math.max(oldSpeed, newSpeed) > 1)
{
    newSpeed = 1;
}
SelectedSpeed = math.clamp(newSpeed, 0, 8);
```
(repo/Systems/AdvancedSimSpeedUISystem.cs#L81-L85)

- [Augment vanilla UI via `moduleRegistry.extend`](../how-to/recipes/vanilla-ui-augmentation.md)
  (family AU) - `index.tsx` registers the *same* `AdvancedTimeControls`
  `ModuleRegistryExtend` against BOTH the legacy `time-controls.tsx` (`TimeControls`)
  and the new 1.6 `time-controls-new.tsx` (`TimeControlsNew`), so the augmentation
  survives the toolbar rewrite regardless of which control the running build mounts
  (repo/UI/src/index.tsx#L7-L19). The component appends its stepper after the
  vanilla clock via the wrapped `<Component>` (repo/UI/src/mods/simSpeedComponent.tsx#L97-L104).
  Glyph theming is interface-aware: the backend pushes
  `SharedSettings.instance.userInterface.useLegacyInterface` as a `legacyUI`
  binding every frame (repo/Systems/AdvancedSimSpeedUISystem.cs#L36,#L60,#L95), and
  the component swaps between UIL arrow glyphs (legacy) and a rotated
  `SimulationPlay.svg` (new) off that flag
  (repo/UI/src/mods/simSpeedComponent.tsx#L64-L66,#L92,#L100).

```tsx
moduleRegistry.extend(
  "game-ui/game/components/toolbar/bottom/time-controls/time-controls.tsx",
  "TimeControls", AdvancedTimeControls);
moduleRegistry.extend(
  "game-ui/game/components/toolbar/bottom/time-controls/time-controls-new.tsx",
  "TimeControlsNew", AdvancedTimeControls);
```
(repo/UI/src/index.tsx#L7-L19)

- [Read-only status / observability binding](../how-to/recipes/status-observability-binding.md)
  (family AX) - the live "actual speed" readout is polled off a hand-rolled
  `IntervalTimer.EveryMilliseconds(500, ...)` that writes `_actualSpeed.Value` on
  each fire (repo/Systems/AdvancedSimSpeedUISystem.cs#L58; repo/Util/IntervalTimer.cs#L27-L28,#L52-L62).
  Writes land on a dirty-flag `ValueBindingHelper`, so the value is batched and the
  binding only calls `Binding.Update` once per frame when actually dirty, keeping
  steady-state UI cost low (repo/Extensions/ValueBindingHelper.cs#L15-L39;
  repo/Extensions/ExtendedUISystemBase.cs#L23-L32). A `refresh` trigger lets the
  frontend force an immediate re-push after a speed change
  (repo/Systems/AdvancedSimSpeedUISystem.cs#L62; repo/UI/src/mods/simSpeedComponent.tsx#L28,#L41).

Settings round out the surface: a `SettingsUISlider` `Step` field gated by
`SettingsUIDisableByCondition` (disabled unless `Type == Fixed`), a confirmed
`Reset` button that calls `SetDefaults` + `ApplyAndSave`, and an `onSettingsApplied`
handler that live-re-pushes the display flags without a restart
(repo/Setting.cs#L25-L49; repo/Systems/AdvancedSimSpeedUISystem.cs#L48-L53). See the
[settings patterns recipe](../how-to/recipes/settings-patterns.md).

## Key decisions & tradeoffs

- **Write the vanilla speed field directly; no Harmony.** The mod never patches a
  method. Time control is a property write to `_simulationSystem.selectedSpeed`
  (repo/Systems/AdvancedSimSpeedUISystem.cs#L24-L28), and the UI is grafted via
  `moduleRegistry.extend` rather than by patching the toolbar. This is the entire
  reason the mod is ~one system: it inherits vanilla pause/speed semantics for
  free and has no decompiled clone to re-diff each patch. The cost is total
  dependence on vanilla field names (`selectedSpeed`, `smoothSpeed`) and module
  paths staying stable.
- **Expose a bespoke speed binding to dodge pause aliasing.** Rather than reading
  vanilla `time.simulationSpeed$`, the mod publishes its own `selectedSpeed`
  getter binding precisely because the vanilla stream reports the pre-pause speed
  while paused (repo/UI/src/mods/simSpeedComponent.tsx#L17-L19). A small decision
  with a real correctness payoff for a speed HUD.
- **Two registrations, not one, for the 1.6 toolbar rewrite.** Instead of
  detecting the game build, the mod simply extends both the old and new clock
  modules with the same component (repo/UI/src/index.tsx#L7-L19). Cheap forward/
  backward compatibility - at the price of two module-path strings that both have
  to keep matching.
- **Split binding strategy by volatility.** Selected speed (changes only on user
  action, but must be instantly correct) is a per-frame getter; actual speed
  (changes constantly, tolerant of ~500 ms latency) is a throttled dirty-flag push
  (repo/Systems/AdvancedSimSpeedUISystem.cs#L54,#L57-L58). This keeps the
  high-frequency telemetry off the per-frame path.
- **`Display` mode as a passive escape hatch.** When `Type == Display` the frontend
  hides both stepper buttons, leaving only the readout
  (repo/UI/src/mods/simSpeedComponent.tsx#L101-L103) - so the mod doubles as a pure
  speed HUD with zero control surface.

## Pitfalls / upstream-watch

- **Strict-inequality snap footgun.** The snap-through-1x guard fires only when the
  step *strictly straddles* 1 (`min < 1 && max > 1`)
  (repo/Systems/AdvancedSimSpeedUISystem.cs#L81). A step whose endpoint lands
  exactly on the boundary is not treated as straddling, so with float arithmetic a
  transition that touches 1 exactly can pass without the intended 1x pause. Any
  edit to the stepping math must re-check this boundary case.
- **`Fixed` mode can land on 0x.** In `Fixed` mode the next value is
  `SelectedSpeed + n * step`, then `math.clamp(newSpeed, 0, 8)` with a *floor of 0*
  (repo/Systems/AdvancedSimSpeedUISystem.cs#L70-L85). Stepping down far enough
  clamps to 0, i.e. a full pause via the speed stepper - surprising if the user
  expected a minimum positive speed.
- **A missing `Display` case in the write trigger throws - guard both ends.** The
  `selectSpeed` trigger switches only on `Doubled` and `Fixed`; any other value
  (including `Display`) hits `throw new Exception($"Invalid enum value {e}")`
  (repo/Systems/AdvancedSimSpeedUISystem.cs#L67-L79). The only thing keeping this
  from throwing in `Display` mode is the frontend hiding the buttons
  (repo/UI/src/mods/simSpeedComponent.tsx#L101-L103). Adding a new speed mode means
  updating BOTH the backend switch and the frontend gate, or the trigger throws.
- **Unguarded `getModuleComponent` cast for the vanilla `Divider`.** The component pulls the
  toolbar `Divider` straight out of `field.tsx` with no existence check
  (repo/UI/src/mods/simSpeedComponent.tsx#L15). If a future patch renames or moves
  that export - the same kind of rename the dual toolbar registration exists to
  survive - the cast yields undefined and the stepper render breaks.
- **UIL glyphs consumed with no fallback.** Legacy-interface arrows load from the
  Unified Icon Library host (`coui://uil/Standard/ArrowLeftTriangle.svg`,
  `ArrowRightTriangle.svg`) with no graceful degradation if UIL is absent
  (repo/UI/src/mods/simSpeedComponent.tsx#L66,#L92); the buttons would render with
  missing icons in legacy mode without that dependency present.

Needs Verification (in-game): the exact in-game speed produced by each step and
mode, whether the 1x snap feels correct across rapid stepping, the actual on-screen
behaviour when `Divider` or the UIL glyphs are missing, and whether `smoothSpeed`
tracks perceived speed closely enough for the readout - none are confirmable from
static source.

## Source pointers

- Dossier:
  `../../vice-and-order-research/mods/dossiers/advanced-simulation-speed/`
  (`index.md`, `source.md`, `modding.md`, `guide.md`, `notes/`).
- Repo @ `d224017f15b6be7384ffcc34528553f86b22280d`, key files:
  - `repo/Mod.cs` - settings/locale load + single `UIUpdate` registration.
  - `repo/Systems/AdvancedSimSpeedUISystem.cs` - direct speed read/write, stepping
    math, bindings, 500 ms interval timer, `onSettingsApplied` re-push.
  - `repo/Domain/Type.cs` - `Doubled` / `Fixed` / `Display` modes.
  - `repo/Setting.cs` - slider + `SettingsUIDisableByCondition` + `Reset` button.
  - `repo/Util/IntervalTimer.cs` - hand-rolled polled timer.
  - `repo/Extensions/ValueBindingHelper.cs`, `repo/Extensions/ExtendedUISystemBase.cs`
    - dirty-flag binding helper + `GetterValueBinding` overloads.
  - `repo/UI/src/index.tsx` - dual `moduleRegistry.extend` (legacy + new toolbar).
  - `repo/UI/src/mods/simSpeedComponent.tsx` - stepper component, interface-aware
    glyphs, pause-safe binding note, unguarded `Divider` cast.
