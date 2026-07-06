---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: Platform achievement control + windowed self-terminating enforcement"
recipe: platform-achievement-control
technique_family: "AR - Platform achievement control + windowed self-terminating enforcement"
diataxis: how-to
source_version: "~1.6.0f1 (achievement-fixer@4d3a1ec; date-pinned, static source only)"
last_reverified: "2026-07-05"
status: source-verified
canonical_mods:
  - achievement-fixer@4d3a1eccc64a7a09de65a23f6abbb3daa998fd0d
technique_applicability: [platform, core]
Created: 2026-07-05
Updated: 2026-07-05
Owners:
  - codex
---

# Platform achievement control + windowed self-terminating enforcement

> Read, unlock, clear, and reset platform achievements through `PlatformManager`,
> and hold a game-owned flag (`achievementsEnabled`) true through a noisy startup by
> re-asserting it every frame for a bounded window, then going idle.

## Problem
You want to touch the platform (Steam/PDX) achievement layer directly - list the
achievements the platform knows about, unlock or clear a specific one, or wipe them
all - from mod settings. Separately, you have a game-owned flag that some vanilla
system flips to a value you don't want during load (here: the game disables
achievements when mods are active), and a single write at load time loses the race:
the flag is set true, then flipped false a few frames later. You need the flag to
*stay* at your value across the unpredictable startup window without patching vanilla
methods and without running your system forever.

## Solution
Two reusable pieces live in one small mod.

1. **The `PlatformManager` API surface.** `PlatformManager.instance` exposes the whole
   achievement layer: `EnumerateAchievements()` to list, `GetAchievement(id, out ...)`
   to read one, `UnlockAchievement(id)` / `ClearAchievement(id)` to mutate one, and
   `ResetAchievements()` to wipe all. Resolve an `AchievementId` from a stable string
   key by scanning `EnumerateAchievements()`; gate destructive buttons behind
   `[SettingsUIConfirmation]`.

2. **Windowed re-assert then self-terminate.** On `OnGameLoadingComplete` for
   `GameMode.Game`, open a fixed frame window (`kAssertFrames = 1800`) and enable the
   system. Each `OnUpdate`, idempotently re-write the flag only if it has drifted, then
   decrement the counter; when it hits zero, set `Enabled = false` and stop consuming
   CPU. Schedule the system **after** the vanilla `AchievementTriggerSystem` in
   `MainLoop` so your re-assert runs after the frame's achievement bookkeeping.

## Steps & Code

### 1. Schedule after the vanilla achievement system, in the main loop

Ordering matters: you want your re-assert to land *after* the game's own achievement
work each frame. Register with `UpdateAfter<Self, Anchor>(phase)`:

```csharp
// Ensure AF system runs after the game's trigger during the main loop.
updateSystem.UpdateAfter<AchievementFixerSystem, AchievementTriggerSystem>(SystemUpdatePhase.MainLoop);
```
Source: `../../../vice-and-order-research/mods/dossiers/achievement-fixer/repo/Mod.cs#L76` (@4d3a1eccc64a7a09de65a23f6abbb3daa998fd0d)

### 2. Start the system idle so it never ticks until a real load

Define the window length as a frame count and disable the system in `OnCreate`. A
disabled `GameSystemBase` is not scheduled, so it costs nothing until a load re-enables
it:

```csharp
private const int kAssertFrames = 1800;  // ~30s @ 60FPS or ~60s @ 30FPS
private int m_FramesLeft;                 // counts down from kAssertFrames to 0

protected override void OnCreate()
{
    base.OnCreate();
    m_FramesLeft = 0;
    Enabled = false;   // idle until a real game load occurs
}
```
Source: `../../../vice-and-order-research/mods/dossiers/achievement-fixer/repo/Systems/AchievementFixerSystem.cs#L14-L32` (@4d3a1eccc64a7a09de65a23f6abbb3daa998fd0d)

### 3. Open the window on `GameMode.Game` load only

`OnGameLoadingComplete` fires for menu and editor too. Guard on `mode == GameMode.Game`,
reset the counter, re-enable, and enforce once immediately so the flag is correct on the
very first tick:

```csharp
protected override void OnGameLoadingComplete(Purpose purpose, GameMode mode)
{
    base.OnGameLoadingComplete(purpose, mode);
    if (mode != GameMode.Game) { Enabled = false; return; }  // skip menu/editor

    m_FramesLeft = kAssertFrames;   // open the assert window
    Enabled = true;                 // start ticking
    ForceEnableIfNeeded("OnGameLoadingComplete");  // enforce immediately
}
```
Source: `../../../vice-and-order-research/mods/dossiers/achievement-fixer/repo/Systems/AchievementFixerSystem.cs#L34-L58` (@4d3a1eccc64a7a09de65a23f6abbb3daa998fd0d)

### 4. Re-assert each frame, decrement, and self-terminate

`OnUpdate` is the whole windowed pattern: if the window is spent, disable and bail;
otherwise re-assert and count down. Once `Enabled = false` the system stops being
scheduled - it terminates itself:

```csharp
protected override void OnUpdate()
{
    if (m_FramesLeft <= 0) { Enabled = false; return; }  // window spent -> go idle

    ForceEnableIfNeeded("OnUpdate");   // keep the flag true; cheap & robust
    m_FramesLeft--;                    // advance the window
}
```
Source: `../../../vice-and-order-research/mods/dossiers/achievement-fixer/repo/Systems/AchievementFixerSystem.cs#L60-L84` (@4d3a1eccc64a7a09de65a23f6abbb3daa998fd0d)

### 5. Make the re-assert idempotent (write only on drift)

The enforcer reads the current flag and writes *only if it drifted*. That keeps the
per-frame cost to a comparison in the common case and avoids redundant writes:

```csharp
private static bool ForceEnableIfNeeded(string source)
{
    PlatformManager pm = PlatformManager.instance;
    if (pm == null) return false;

    if (!pm.achievementsEnabled)
    {
        Mod.s_Log.Info($"{source}: ATTN: detected game flipped achievementsEnabled == FALSE. Forcing TRUE now");
        pm.achievementsEnabled = true;   // write only when drifted
        Mod.s_Log.Info($"{source}: achievementsEnabled is now TRUE.");
        return true;
    }
    return false;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/achievement-fixer/repo/Systems/AchievementFixerSystem.cs#L86-L107` (@4d3a1eccc64a7a09de65a23f6abbb3daa998fd0d)

### 6. Build a data-driven achievement dropdown from `EnumerateAchievements()`

For the control surface, populate a settings dropdown from the platform's own list.
The stable dropdown *value* is `internalName` (falling back to `id.ToString()`); the
display label goes through a localization helper:

```csharp
public static DropdownItem<string>[] GetAchievementChoices()
{
    PlatformManager pm = PlatformManager.instance;
    if (pm == null) return Array.Empty<DropdownItem<string>>();

    return pm.EnumerateAchievements()
        .Select(a => a.internalName ?? a.id.ToString())
        .OrderBy(id => AchievementDisplay.Get(id), StringComparer.CurrentCultureIgnoreCase)
        .Select(id => new DropdownItem<string> { value = id, displayName = AchievementDisplay.Get(id) })
        .ToArray();
}
```
Source: `../../../vice-and-order-research/mods/dossiers/achievement-fixer/repo/Settings/Settings.cs#L282-L303` (@4d3a1eccc64a7a09de65a23f6abbb3daa998fd0d)

The dropdown property itself binds the choices provider by name:

```csharp
[SettingsUISection(AdvancedTab, AdvRowActions)]
[SettingsUIDropdown(typeof(Settings), nameof(GetAchievementChoices))]
public string SelectedAchievement { get; set; } = "";
```
Source: `../../../vice-and-order-research/mods/dossiers/achievement-fixer/repo/Settings/Settings.cs#L132-L134` (@4d3a1eccc64a7a09de65a23f6abbb3daa998fd0d)

### 7. Resolve a string key to an `AchievementId`, bailing on empty

Never trust the raw dropdown string. `TryGetAchievementId` scans the platform list,
matching `internalName` first and `id.ToString()` as a fallback, and returns `false`
(a safe no-op) if `PlatformManager` is null or nothing matches:

```csharp
private static bool TryGetAchievementId(string selectedValue, out AchievementId id)
{
    id = default;
    PlatformManager pm = PlatformManager.instance;
    if (pm == null) return false;

    foreach (IAchievement? a in pm.EnumerateAchievements())
    {
        if (!string.IsNullOrEmpty(a.internalName) &&
            string.Equals(a.internalName, selectedValue, StringComparison.OrdinalIgnoreCase))
        { id = a.id; return true; }

        if (string.Equals(a.id.ToString(), selectedValue, StringComparison.OrdinalIgnoreCase))
        { id = a.id; return true; }
    }
    return false;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/achievement-fixer/repo/Settings/Settings.cs#L305-L333` (@4d3a1eccc64a7a09de65a23f6abbb3daa998fd0d)

### 8. Mutate one achievement: unlock / clear, with a post-check read

Once you have an `AchievementId`, `UnlockAchievement` / `ClearAchievement` do the write;
`GetAchievement(id, out IAchievement?)` reads it back so you can log whether the state
actually changed:

```csharp
pm.UnlockAchievement(id);
var ok = pm.GetAchievement(id, out IAchievement? a) && a.achieved;
Mod.s_Log.Info($"UnlockSelected: \"{AchievementDisplay.Get(SelectedAchievement)}\" -> {(ok ? "Enabled" : "No change")}");
```
Source: `../../../vice-and-order-research/mods/dossiers/achievement-fixer/repo/Settings/Settings.cs#L167-L174` (@4d3a1eccc64a7a09de65a23f6abbb3daa998fd0d)

The clear path is symmetric - `pm.ClearAchievement(id)` then a `!a.achieved` post-check
(`Settings.cs#L215-L222`).

### 9. Gate destructive actions behind `[SettingsUIConfirmation]`

The "reset everything" button is unrecoverable, so it carries a confirmation attribute
and only calls `ResetAchievements()` after the user confirms:

```csharp
[SettingsUIButton]
[SettingsUIConfirmation]    // Yes/No modal before the setter runs
[SettingsUIButtonGroup(AdvRowDebug)]
[SettingsUISection(AdvancedTab, AdvRowDebug)]
public bool ResetAllAchievements
{
    set
    {
        if (!value) return;
        PlatformManager pm = PlatformManager.instance;
        if (pm == null) return;
        pm.ResetAchievements();   // wipes ALL platform achievements
    }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/achievement-fixer/repo/Settings/Settings.cs#L239-L277` (@4d3a1eccc64a7a09de65a23f6abbb3daa998fd0d)

`ClearSelectedAchievement` also carries `[SettingsUIConfirmation]`
(`Settings.cs#L186`); `UnlockSelectedAchievement` does not, because unlocking is not
destructive (`Settings.cs#L137-L140`).

## Pitfalls & gotchas

- **A single load-time write loses the race.** The whole reason for the frame window is
  that some vanilla path flips `achievementsEnabled` *after* your first write. Writing
  once in `OnGameLoadingComplete` and stopping is not enough; you must keep re-asserting
  across the startup window (`AchievementFixerSystem.cs#L48-L53, #L60-L84`). The mod
  picks 1800 frames (`kAssertFrames`, `#L15`) as a fixed budget - that is ~30s at 60 FPS
  but only ~15s at 120 FPS or ~60s at 30 FPS, so the *real-time* window scales with frame
  rate. If your target flag can drift later than the budget, raise the constant.

- **Forgetting to self-terminate leaves a forever-system.** The pattern's value is that
  it stops: once `m_FramesLeft <= 0` the system sets `Enabled = false` and is no longer
  scheduled (`AchievementFixerSystem.cs#L63-L66`). Omit that and you have re-added a
  permanent per-frame `PlatformManager.instance` read for the rest of the session.

- **Guard `PlatformManager.instance` on every access.** Every entry point in the source
  null-checks `pm` and no-ops if it is null (`ForceEnableIfNeeded` at `#L88-L95`;
  `GetAchievementChoices` returns an empty array at `Settings.cs#L285-L288`;
  `TryGetAchievementId` returns false at `Settings.cs#L308-L312`). The platform layer is
  not guaranteed to be present when your code runs.

- **Empty / unresolvable selection must be a safe no-op.** `TryGetAchievementId` returns
  `false` when the string does not match any `internalName` or `id.ToString()`, and the
  button setters bail with a warning rather than calling into `PlatformManager` with a
  `default` id (`Settings.cs#L151-L155, #L199-L203`).

- **`GameMode` guard is mandatory.** `OnGameLoadingComplete` also fires for menu and
  editor; without the `mode != GameMode.Game` early-out the window would open in
  contexts where the flag is irrelevant (`AchievementFixerSystem.cs#L39-L46`).

- **Whether unlock/clear/reset actually persist to the platform is runtime behaviour.**
  The source calls the API and reads back via `GetAchievement`, but the end-to-end effect
  on Steam/PDX (and whether the game re-disables achievements again outside the window)
  is `Needs Verification (in-game)`.

## Variations

- **Enforce a different flag, or a value other than `true`.** The window mechanism is
  agnostic to what it writes. Swap the body of `ForceEnableIfNeeded` for any "read
  current, write only if drifted" comparison against your own target value; the
  `OnGameLoadingComplete` -> countdown -> `OnUpdate` -> self-disable skeleton is
  unchanged.

- **Time-based instead of frame-based window.** `kAssertFrames` couples the window to
  frame rate (see pitfalls). If you need a wall-clock window, decrement by
  `SystemAPI.Time.DeltaTime` against a seconds budget instead of by 1 per frame; the
  self-terminate check (`accumulated >= budget -> Enabled = false`) is the same shape.

- **Event-driven instead of polling.** If the platform exposes a change event for your
  flag, subscribe and re-assert on the callback rather than every frame. Polling a frame
  window is the robust fallback the source chose because it makes no assumption about
  *when* the drift happens (`AchievementFixerSystem.cs#L68-L70`).

- **Read-only achievement inspector.** Drop the mutation buttons and keep only
  `EnumerateAchievements()` + `GetAchievement` to surface achievement state in your UI
  (progress, `achieved` flag) without ever writing.

## See also
- Related recipes: [settings patterns](settings-patterns.md) (dropdown providers,
  `[SettingsUIConfirmation]`, button setters), [localization helper](localization-helper.md)
  (resolving display names for the dropdown).
- Reference: [system scheduling](../../explanation/system-scheduling.md)
  (`UpdateAfter` / `SystemUpdatePhase.MainLoop` and why ordering after the vanilla
  system matters).
- Case study demonstrating it: [achievement-fixer](../../case-studies/achievement-fixer.md).

## Sources
- Canonical mods (dossier + repo):
  - `achievement-fixer` @4d3a1eccc64a7a09de65a23f6abbb3daa998fd0d - `repo/Systems/AchievementFixerSystem.cs`, `repo/Settings/Settings.cs`, `repo/Mod.cs`
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Achievements
