---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Validate Prerequisites and Recover Safely in a CS2 Mod"
Summary: How to make a CS2 mod validate its prerequisites before it runs, gate on game mode and load purpose, tear down cleanly when state must reset, and how to exercise all of that against regression saves before you ship.
diataxis: how-to
source_version: "1.6.0f1 (magic-mail@6fb3d2b; date-pinned, static source only)"
last_reverified: "2026-07-04"
canonical_mods:
  - magic-mail@6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47
  - road-speed-adjuster@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed
  - realistic-path-finding@50645fa6a078181365e36a42e2e27b96699bf02a
technique_applicability: [core, simulation]
status: source-verified
Created: 2026-07-03
Updated: 2026-07-03
Owners:
  - codex
References:
  - Label: System Scheduling (RequireForUpdate, phases, cadence)
    Path: ../../explanation/system-scheduling.md
  - Label: "Recipe: System skeleton (OnCreate gating)"
    Path: ../recipes/system-template.md
  - Label: Log and Debug (regression saves, Player.log)
    Path: ./logging-and-debugging.md
  - Label: Release Checklist (the ship gate this feeds)
    Path: ./release-checklist.md
  - Label: Memory and Performance (guarded dispose on teardown)
    Path: ./memory-and-performance.md
  - Label: Technique Index
    Path: ../../technique-index.md
---

# Validate Prerequisites and Recover Safely in a CS2 Mod

A robust CS2 mod does three things a fragile one does not: it refuses to run until its
inputs actually exist, it runs only in the game contexts it was built for, and it tears its
state down cleanly when the player leaves a city. This page shows how to build those three
behaviours - prerequisite validation, game-mode gating, and safe teardown - with cited real
mod source, and then how to *verify* them against regression saves before you ship.

It builds on the system shape in [Recipe: System skeleton](../recipes/system-template.md)
and the scheduling model in [System Scheduling](../../explanation/system-scheduling.md), and
it feeds the [Release Checklist](./release-checklist.md).

---

## 1. Validate prerequisites before a system runs

The cheapest failed run is the one that never happens. Gate a system on its inputs in
`OnCreate` so DOTS skips it entirely when they are absent - that is cheaper and safer than
an early `return` inside `OnUpdate`.

- **`RequireForUpdate` on a query or component.** A system with an unmet `RequireForUpdate`
  is not scheduled at all that tick. Magic Mail's capacity system requires both prefab
  queries it depends on, and deliberately starts disabled:

  ```csharp
  RequireForUpdate(m_PostFacilitiesQuery);
  RequireForUpdate(m_PostVansQuery);

  // Run only when settings change or after a city load.
  Enabled = false;
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/magic-mail/repo/Systems/MailCapacitySystem.cs#L58-L62` (@6fb3d2b)

- **Ensure a singleton exists before you read it.** When a system depends on a config or
  data singleton, check for it rather than assuming it. Realistic Path Finding's congestion
  system creates its config singleton with sane defaults if none is present, so a later read
  cannot fail:

  ```csharp
  if (!SystemAPI.HasSingleton<CarCongestionConfig>())
  {
      var e = EntityManager.CreateEntity(typeof(CarCongestionConfig));
      EntityManager.SetComponentData(e, new CarCongestionConfig { /* defaults */ });
  }
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Systems/CarCongestionEwmaSystem.cs#L86-L98` (@50645fa)

- **Read another module's data defensively.** When you consume a singleton produced by a
  *different* mod or a vanilla system that may not be present, use the try-variant of the
  singleton read (`SystemAPI.TryGetSingleton(out var value)`) and fall back gracefully when
  it returns false, rather than throwing. `Needs Verification`: the exact
  `TryGetSingleton` overload surface is a Unity Entities API - confirm against the
  [Entities docs](https://docs.unity3d.com/Packages/com.unity.entities@latest) for the SDK
  version you build against; the *principle* (check, then fall back) is what matters.

## 2. Gate on game mode and load purpose

A system placed in a simulation phase can be created in the main menu, the map editor, and
a live city. Most gameplay logic should run only in a real game. Gate it:

- **Enable only for a real game, on load completion.** Magic Mail's capacity system starts
  disabled (section 1) and enables itself from `OnGameLoadingComplete` **only** when the
  mode is `GameMode.Game` and the load is a new or loaded game - not the editor, not a
  reload of the same session:

  ```csharp
  protected override void OnGameLoadingComplete(Purpose purpose, GameMode mode)
  {
      base.OnGameLoadingComplete(purpose, mode);

      bool isRealGame =
          mode == GameMode.Game &&
          (purpose == Purpose.NewGame || purpose == Purpose.LoadGame);

      if (isRealGame)
          Enabled = true;
  }
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/magic-mail/repo/Systems/MailCapacitySystem.cs#L65-L76` (@6fb3d2b)

  This is the "enable -> run once -> disable" pattern: pair it with `GetUpdateInterval => 1`
  so the system does its apply-on-change work a single time per enable instead of polling
  every tick (see [System Scheduling](../../explanation/system-scheduling.md#tuning-cadence-getupdateinterval-and-getupdateoffset)).

- **Initialize only when the world is ready.** Road Speed Adjuster's save-data system
  initializes its persistent storage only once the game manager reports `WorldReady` and the
  mode is `Game` or `Editor` - never in the main menu:

  ```csharp
  if (!_initialized && _gameManager != null &&
      _gameManager.state >= GameManager.State.WorldReady &&
      (_gameManager.gameMode == GameMode.Game || _gameManager.gameMode == GameMode.Editor))
  {
      InitializePersistentStorage();
      _initialized = true;
  }
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/Systems/RoadSpeedSaveDataSystem.cs#L36-L42` (@e0c0c0b)

  Gating on the editor context matters both ways: editor-only work should run in `Editor`,
  and runtime-only work should not. A system that quietly executes gameplay logic while the
  player is in the map editor is a common bug.

## 3. Tear down and recover cleanly

State that was set up on load must be undone when the player leaves - otherwise it leaks
into the next city or the main menu. Recovery is the mirror image of initialization.

- **Reset on return to the main menu.** The same Road Speed Adjuster system that
  initialized on `WorldReady` clears its state when the player goes back to the main menu,
  so the next city load reinitializes from scratch:

  ```csharp
  else if (_initialized && _gameManager.gameMode == GameMode.MainMenu)
  {
      _initialized = false;
      _lastCityName = null;
  }
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/Systems/RoadSpeedSaveDataSystem.cs#L44-L49` (@e0c0c0b)

- **Dispose native state in `OnDestroy`.** Any `Allocator.Persistent` collection a system
  holds must be released, guarded by `IsCreated`, when the system is destroyed - see
  [Memory and Performance](./memory-and-performance.md#2-dispose-every-native-collection-you-allocate).

- **Unpatch on unload.** If your mod applies Harmony patches or disables vanilla systems,
  undo that on unload so toggling the mod off restores vanilla behaviour cleanly (verified
  in section 4 below).

## 4. Verify it: regression saves and pre-release recovery checks

Building the guards above is half the job; the other half is proving they work before you
ship. Keep a small library of **regression saves** - saves that reliably reproduce the
conditions your mod touches (for example a heavy-traffic city, a budget-collapse city, and
whatever domain stress your feature responds to) - each with a short note describing the
expected behaviour so anyone can tell pass from fail. See
[Log and Debug](./logging-and-debugging.md#regression-saves).

Before every release, run this recovery sweep (it complements the
[Release Checklist](./release-checklist.md)):

1. **Run tests if you have them.** `dotnet test` for any unit/integration coverage.
2. **Load each regression save and watch CPU time and `Player.log`.** Profile the heavy
   ones - a per-entity cost is invisible on an empty map
   (see [Memory and Performance](./memory-and-performance.md#5-profile-before-you-optimize)).
3. **Toggle the mod off, then on.** Confirm systems unpatch cleanly (vanilla behaviour
   returns) and re-patch without duplicating state or throwing.
4. **Launch without each optional shared dependency** (for example a localization helper or
   a shared icon library) and confirm graceful degradation - the dependent feature disables
   itself, everything else keeps running, and the log has no per-frame errors. This is the
   run-time half of prerequisite validation.
5. **Open the map editor.** Confirm your game-mode gate (section 2) actually holds -
   runtime-only systems must not execute in `Editor`, and editor-only work must not leak
   into a live city.

## Pitfalls & gotchas

- **No `RequireForUpdate`, so the system runs every eligible tick even when idle.** Gate in
  `OnCreate`; it is cheaper than an early `return` in `OnUpdate`.
- **Reading a singleton that may not exist.** Ensure-or-create it (section 1) or use the
  try-variant and fall back; a hard `GetSingleton` throws when it is absent.
- **Running gameplay logic in the editor or main menu.** A simulation-phase system is
  created in every context; gate on `GameMode` / load `Purpose`.
- **Initializing without tearing down.** State set up on load but never reset on return to
  the main menu leaks into the next city. Recovery mirrors setup.
- **Not testing the absence path.** A missing optional dependency, a toggled-off mod, and
  the editor context are exactly the states that are green on your machine and broken for a
  player - test them explicitly.

## See also

- Concept: [System Scheduling](../../explanation/system-scheduling.md) - `RequireForUpdate`,
  phases, and the enable/run-once/disable cadence.
- Recipe: [System skeleton](../recipes/system-template.md) - the `OnCreate` gating this page
  cites in context.
- Operations: [Memory and Performance](./memory-and-performance.md) - guarded dispose on
  teardown and profiling saves; [Incident Response](./incident-response.md) - when a guard
  fails in the wild; [Release Checklist](./release-checklist.md) - the ship gate.
- Reference: [Technique Index](../../technique-index.md) (families L, M, S).

## Sources

- Canonical mods (dossier + repo, pinned commits):
  - `magic-mail` @6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47
  - `road-speed-adjuster` @e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed
  - `realistic-path-finding` @50645fa6a078181365e36a42e2e27b96699bf02a
- Official references (link out, do not duplicate): Unity Entities (systems, singletons,
  `RequireForUpdate`) - https://docs.unity3d.com/Packages/com.unity.entities@latest
