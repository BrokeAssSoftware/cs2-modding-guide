---
FrontmatterVersion: 1
DocumentType: Guide
Title: Conditional Execution and Diagnostics
Summary: How CS2 mods keep a DOTS system from running when it should not - RequireForUpdate gating, GameMode/state guards, the enable-run-once-disable idiom, and diagnostics behind a flag - explained against real mod source.
diataxis: explanation
source_version: "1.6.0f1 (magic-mail@6fb3d2b; date-pinned, static source only)"
last_reverified: "2026-07-04"
canonical_mods:
  - magic-mail@6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47
  - road-speed-adjuster@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed
  - traffic-tool-essentials@10973595ac9ed37f47ca24eb788d4421dc295fa0
  - magic-garbage-truck@1b6a478753e1ef4e43ac9b90d567f3d7183c7be2
  - realistic-path-finding@50645fa6a078181365e36a42e2e27b96699bf02a
  - realistic-jobsearch@7a096b2ab974bb03cc4cf0937f250bf1d7671f31
status: source-verified
Created: 2026-07-03
Updated: 2026-07-05
Owners:
  - codex
References:
  - Label: System Scheduling (companion concept)
    Path: ./system-scheduling.md
  - Label: Lifecycle and Initialization
    Path: ./mod-lifecycle.md
  - Label: Recipe - Periodic UpdateInterval system
    Path: ../how-to/recipes/periodic-updateinterval-system.md
  - Label: Technique Index
    Path: ../technique-index.md
---

# Conditional Execution and Diagnostics

A system that is scheduled runs every time its phase fires - even in the main menu,
even when the feature is switched off, even when there is no work to do. That is
wasteful and, worse, unsafe: touching ECS data before a city is loaded or a feature is
enabled is a common source of null-reference and stale-data bugs.

This page explains the *why* of keeping a system quiet: the gates CS2 gives you, in
rough order of cheapness, and the diagnostics that let you see what a gated system is
(or is not) doing. It is a concept page. For the full periodic-system recipe, see
[`recipes/periodic-updateinterval-system.md`](../how-to/recipes/periodic-updateinterval-system.md);
for how systems slot into the loop at all, see [System Scheduling](./system-scheduling.md).

## The gates, cheapest first

Prefer the cheapest gate that expresses your condition. Each layer below runs *less*
of your code when the condition is unmet.

### 1. RequireForUpdate - skip the whole system

`RequireForUpdate<T>()` (or `RequireForUpdate(query)`) tells the scheduler not to call
`OnUpdate` at all unless the requirement is satisfied. This is cheaper than early-returning
inside `OnUpdate`, because the system is skipped before your code runs. Register the
requirement once in `OnCreate`.

Magic Mail's one-shot capacity system requires both of its prefab queries before it will
run ([`magic-mail` `repo/Systems/MailCapacitySystem.cs#L58-L59`](../../vice-and-order-research/mods/dossiers/magic-mail/repo/Systems/MailCapacitySystem.cs), commit `6fb3d2b`):

```csharp
RequireForUpdate(m_PostFacilitiesQuery);
RequireForUpdate(m_PostVansQuery);
```

Its periodic sibling does the same with its post-facilities query
([`magic-mail` `repo/Systems/MagicMailSystem.cs#L100`](../../vice-and-order-research/mods/dossiers/magic-mail/repo/Systems/MagicMailSystem.cs), commit `6fb3d2b`).
If the queries are empty (no matching prefabs in this save), the system simply never
fires - no per-tick guard needed.

#### Seed the singleton you require, so there is no bootstrap dependency

A system that requires a *config singleton* usually depends on some earlier bootstrap step
having created that entity. You can remove that dependency by seeding the singleton
yourself in `OnCreate`. Realistic Job Search's acceptance-gate system checks whether its
params singleton exists and, if not, creates and defaults it from settings
([`realistic-jobsearch` `repo/RealisticJobSearch/Systems/GravityAcceptanceGateSystem.cs#L34-L44`](../../vice-and-order-research/mods/dossiers/realistic-jobsearch/repo/RealisticJobSearch/Systems/GravityAcceptanceGateSystem.cs), commit `7a096b2`):

```csharp
m_ParamsQ = GetEntityQuery(ComponentType.ReadOnly<GravityAcceptParams>());
if (m_ParamsQ.IsEmptyIgnoreFilter)
{
    EntityManager.CreateEntity(typeof(GravityAcceptParams));
    EntityManager.SetComponentData(m_ParamsQ.GetSingletonEntity(), new GravityAcceptParams
    {
        AlphaJobs = Mod.m_Setting.alpha_jobs,
        // ... remaining tuning fields defaulted from settings ...
    });
}
```

It then `RequireForUpdate`s the *results* query rather than the config
([`#L53`](../../vice-and-order-research/mods/dossiers/realistic-jobsearch/repo/RealisticJobSearch/Systems/GravityAcceptanceGateSystem.cs), commit `7a096b2`),
so the gate is skipped only when there is genuinely no work - never merely because nobody
seeded its configuration.

### 2. GameMode / game-state guards - run only in a real city

Even a scheduled system can fire in the main menu or the asset editor. Guard on the
`GameManager` state and `GameMode` so simulation logic only runs in an actual game.

Magic Mail's capacity system enables itself only when a real game finishes loading, by
overriding `OnGameLoadingComplete` and checking both the mode and the load purpose
([`magic-mail` `repo/Systems/MailCapacitySystem.cs#L65-L77`](../../vice-and-order-research/mods/dossiers/magic-mail/repo/Systems/MailCapacitySystem.cs), commit `6fb3d2b`):

```csharp
protected override void OnGameLoadingComplete(Purpose purpose, GameMode mode)
{
    base.OnGameLoadingComplete(purpose, mode);

    bool isRealGame =
        mode == GameMode.Game &&
        (purpose == Purpose.NewGame || purpose == Purpose.LoadGame);

    if (isRealGame)
    {
        Enabled = true;
    }
}
```

Road Speed Adjuster gates on both the `GameManager.State` and the `GameMode` inside
`OnUpdate` before initialising its persistent storage, and resets when the player returns
to the main menu ([`road-speed-adjuster` `repo/Systems/RoadSpeedSaveDataSystem.cs#L36-L44`](../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/Systems/RoadSpeedSaveDataSystem.cs), commit `e0c0c0b`):

```csharp
if (!_initialized && _gameManager != null &&
    _gameManager.state >= GameManager.State.WorldReady &&
    (_gameManager.gameMode == GameMode.Game || _gameManager.gameMode == GameMode.Editor))
{
    InitializePersistentStorage();
    _initialized = true;
}
else if (_initialized && _gameManager.gameMode == GameMode.MainMenu)
{
    _initialized = false;
    _lastCityName = null;
}
```

The `state >= GameManager.State.WorldReady` check is the important part: it prevents the
system from touching a world that has not finished loading.

#### A shared mid-load no-op gate

`WorldReady` is a one-time threshold, but a save can still be *streaming* - loading a game
or applying serialized state - after a system has begun updating. Realistic Path Finding
centralises this into one tiny static helper that every one of its systems consults at the
top of `OnUpdate` ([`realistic-path-finding` `repo/RealisticPathFinding/Systems/SimulationGuard.cs#L7-L11`](../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Systems/SimulationGuard.cs), commit `50645fa`):

```csharp
public static bool IsGameLoading()
{
    var gameManager = GameManager.instance;
    return gameManager != null && gameManager.isGameLoading;
}
```

Each system then bails early while a load is in progress - for example
`if (SimulationGuard.IsGameLoading()) return;`
([`repo/RealisticPathFinding/Systems/ScaleWaitingTimeSystem.cs#L16`](../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Systems/ScaleWaitingTimeSystem.cs), commit `50645fa`).
Factoring the check into one place keeps the guard consistent across a large system fleet
and makes "should this run mid-load?" a single, testable decision.

### 3. Enable / run-once / disable - the one-shot idiom

Apply-on-change work (recompute capacities when a setting changes, run once after a load)
should not poll every tick. The idiom is: start `Enabled = false`, flip `Enabled = true`
from the event that needs it, do the work in `OnUpdate`, then set `Enabled = false` again.

Magic Mail's capacity system starts disabled in `OnCreate`
([`repo/Systems/MailCapacitySystem.cs#L62`](../../vice-and-order-research/mods/dossiers/magic-mail/repo/Systems/MailCapacitySystem.cs))
and returns `GetUpdateInterval => 1` so that once enabled it runs on the next tick
([`#L82-L85`](../../vice-and-order-research/mods/dossiers/magic-mail/repo/Systems/MailCapacitySystem.cs)).
Its `OnUpdate` re-checks the conditions and disables itself the moment they do not hold
([`#L87-L101`](../../vice-and-order-research/mods/dossiers/magic-mail/repo/Systems/MailCapacitySystem.cs), commit `6fb3d2b`):

```csharp
protected override void OnUpdate()
{
    GameManager gm = GameManager.instance;
    if (gm == null || !gm.gameMode.IsGame())
    {
        Enabled = false;
        return;
    }

    Setting? settings = Mod.Settings;
    if (settings == null)
    {
        Enabled = false;
        return;
    }
    // ... do the apply-on-change work ...
}
```

The early `return` after `Enabled = false` is the belt-and-braces layer: even when the
system is briefly enabled in a state it should not act in, it costs one cheap check and
switches itself back off. See the periodic-system recipe for the full write-up.

#### Restore your mutation *before* you disable

The idiom has a trap when the system has *changed* shared state: self-disabling while that
change is still applied leaves the state stuck. Magic Garbage Truck's priority-assist
system raises a city-wide garbage collection limit while critical buildings need help. When
assist is no longer allowed, it first writes the limit back to normal and only then permits
itself to disable - the `Enabled = false` is guarded on the limit already equalling normal
([`magic-garbage-truck` `repo/Systems/GarbagePriorityAssistSystem.cs#L263-L271`](../../vice-and-order-research/mods/dossiers/magic-garbage-truck/repo/Systems/GarbagePriorityAssistSystem.cs), commit `1b6a478`):

```csharp
if (data.m_CollectionGarbageLimit != targetCollect)
{
    data.m_CollectionGarbageLimit = targetCollect;   // targetCollect == normalCollect once assist ends
}

if (!assistAllowed && data.m_CollectionGarbageLimit == normalCollect)
{
    Enabled = false;                                 // only self-disable once the limit is restored
}
```

Because the disable is conditioned on `m_CollectionGarbageLimit == normalCollect`, toggling
the feature off can never leave the threshold stuck-raised: the system keeps running until
it has undone its own mutation, then goes quiet.

## Diagnostics for gated systems

A gated system is easy to mis-debug: "nothing happened" could mean the gate correctly
skipped it, or that a bug skipped it. Two practical tactics:

### Debug logging behind a settings flag

Verbose per-tick logging is itself a performance cost, so gate it behind a debug toggle
and keep only lifecycle events always-on. Traffic Tool Essentials splits its logging into
`LogDebug` (only when the debug setting is on) and `LogInfo` (always)
([`traffic-tool-essentials` `repo/TrafficToolEssentials/Mod.cs#L27-L46`](../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Mod.cs), commit `1097359`):

```csharp
public static bool IsDebugLoggingEnabled => m_Settings?.m_DebugLogging == true;

public static void LogDebug(string message)
{
    if (IsDebugLoggingEnabled)
        m_Log.Info(message);
}

public static void LogInfo(string message)
{
    m_Log.Info(message);
}
```

### Run faster while debugging, throttle in release

A periodic system can pick its cadence from the build configuration. Magic Mail's main
system drops to the vanilla facility interval under `#if DEBUG` and throttles to its real
cadence in release ([`magic-mail` `repo/Systems/MagicMailSystem.cs#L59-L67`](../../vice-and-order-research/mods/dossiers/magic-mail/repo/Systems/MagicMailSystem.cs), commit `6fb3d2b`):

```csharp
public override int GetUpdateInterval(SystemUpdatePhase phase)
{
#if DEBUG
    return 256;                 // vanilla PostFacilityAISystem interval, for debugging
#else
    return 262144 / UpdatesPerDay;
#endif
}
```

### ProfilerMarker and developer-mode dumps - `Needs Verification`

Two further diagnostic techniques are worth knowing but are **not source-verified in the
current corpus**, so treat them as `Needs Verification` until a dossier demonstrates them:

- **`ProfilerMarker` scopes** around heavy jobs, captured in profiling builds, to attribute
  frame cost to your system. No dossier in the corpus currently uses `ProfilerMarker`.
- **Developer-mode commands** (behind CS2's `-developerMode` / the in-game developer UI)
  that dump component or analytics state for QA. Not demonstrated in a corpus dossier.

Do not present either as an established CS2 mod pattern without a cited example.

## Common pitfalls

- **Guarding in `OnUpdate` when `RequireForUpdate` would do.** If the condition is a query
  or singleton being present, prefer `RequireForUpdate` so the system is skipped entirely.
- **Polling every tick for apply-on-change work.** Use the enable/run-once/disable idiom,
  not a permanently-enabled `GetUpdateInterval => 1` system.
- **Touching the world before it is ready.** Always gate on `GameManager.State` (e.g.
  `>= WorldReady`) and `GameMode` before reading ECS data on load.
- **Always-on verbose logging.** Frequent logs are a real cost; gate them behind a debug
  flag and keep only lifecycle events unconditional.
- **Self-disabling before you undo your mutation.** If a system has raised a limit or
  written shared state, restore it *first* and condition `Enabled = false` on the restore,
  or toggling the feature off leaves the state stuck.
- **Trusting `WorldReady` alone.** A save can still be streaming after the world is ready;
  add a shared mid-load no-op (an `isGameLoading` check) for systems that run during load.

## See also

- Concept: [System Scheduling](./system-scheduling.md) - phases, cadence tuning, and the
  caching/gating properties of well-behaved systems.
- Concept: [Lifecycle and Initialization](./mod-lifecycle.md) - `OnCreate`/`OnGameLoadingComplete`
  ordering, where these gates are installed.
- How-to: [Periodic UpdateInterval system](../how-to/recipes/periodic-updateinterval-system.md)
  - the full enable/run-once/disable recipe.
- Reference: [Technique Index](../technique-index.md) - family L (periodic
  `GameSystemBase` with `UpdateInterval` tuning).
