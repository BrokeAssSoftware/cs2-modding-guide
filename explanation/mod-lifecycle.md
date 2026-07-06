---
FrontmatterVersion: 1
DocumentType: Guide
Title: Lifecycle and Initialization
Summary: How a CS2 mod bootstraps through IMod.OnLoad(UpdateSystem) and tears down in OnDispose, explained against real mod source.
diataxis: explanation
source_version: "1.6.0f1 (realistic-path-finding@50645fa; date-pinned, static source only)"
last_reverified: "2026-07-04"
canonical_mods:
  - realistic-path-finding@50645fa6a078181365e36a42e2e27b96699bf02a
  - magic-mail@6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47
  - outside-traffic-adjuster@42afd29638267c9dc116b040515ceaf41eb9a9e1
  - road-speed-adjuster@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed
  - traffic-tool-essentials@10973595ac9ed37f47ca24eb788d4421dc295fa0
status: source-verified
Created: 2026-07-01
Updated: 2026-07-01
Owners:
  - codex
References:
  - Label: System Scheduling (companion concept)
    Path: ./system-scheduling.md
  - Label: Module Layout (companion concept)
    Path: ./module-layout.md
  - Label: Dependency Strategy (companion concept)
    Path: ./dependency-strategy.md
  - Label: Settings and Data Management (pending rework)
    Path: ./settings-and-data.md
  - Label: Recipe - Periodic UpdateInterval system
    Path: ../how-to/recipes/periodic-updateinterval-system.md
  - Label: Recipe - ECS system replacement ordering
    Path: ../how-to/recipes/ecs-system-replacement-ordering.md
  - Label: Technique Index
    Path: ../technique-index.md
---

# Lifecycle and Initialization

Every Cities: Skylines II code mod is a class that implements `Game.Modding.IMod`.
The game calls exactly two methods on it: `OnLoad(UpdateSystem updateSystem)` when the
mod is activated during world initialization, and `OnDispose()` when it is unloaded.
Understanding what belongs in each - and the order it must happen in - is the foundation
for everything else in this handbook.

This page explains the *shape* of a well-behaved `OnLoad` and *why* the ordering matters.
It is a concept page, not a step-by-step tutorial; for a scaffolded first mod see
the code-mod bootstrap tutorial (pending rework) or the [Technique Index](../technique-index.md).

## The two entry points

`IMod` is deliberately tiny. The whole contract is:

- `OnLoad(UpdateSystem updateSystem)` - the game hands you the `UpdateSystem` scheduler
  for the active ECS `World`. This is your one chance to build settings, register
  localization, and slot your DOTS systems into the simulation loop.
- `OnDispose()` - symmetric teardown. Unregister anything global you added (Options UI,
  Harmony patches) so the mod can be cleanly reloaded without leaking state.

Because `OnLoad` receives the scheduler, mod initialization and system scheduling are the
same act. The rest of this page walks the canonical order; the scheduling half is expanded
in [System Scheduling](./system-scheduling.md).

## The canonical OnLoad order

Across our source-verified corpus the same skeleton recurs, and the ordering is not
cosmetic - each step depends on state established by the previous one:

1. **Log a load banner** (and optionally resolve the executable asset path). Cheap, and
   the first thing you want in a support log. Realistic Path Finding logs the banner and
   then resolves its own asset path via
   `GameManager.instance.modManager.TryGetExecutableAsset(this, out var asset)`
   ([`realistic-path-finding` `repo/Mod.cs#L26-L34`](../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Mod.cs), commit `50645fa`),
   which is the idiom for finding where your mod's DLL and bundled assets live on disk.
2. **Create the `Setting` instance.** The settings object is the backing store the Options
   UI and every locale source read from, so it must exist before either.
3. **Add locale sources** via `localizationManager.AddSource(localeId, source)`. Register
   these before the Options UI paints so labels resolve on first render.
4. **Load persisted settings** with `AssetDatabase.global.LoadSettings(id, setting, defaults)`.
   This overwrites the fresh `Setting` with the player's saved `.coc` values (or seeds
   defaults on first run).
5. **Register the Options UI** with `setting.RegisterInOptionsUI()`.
6. **Detect dependencies / disable vanilla systems** you intend to replace.
7. **Schedule your systems** with `updateSystem.UpdateAt/UpdateBefore/UpdateAfter<T>(phase)`.
8. **Apply Harmony patches last** (only if the mod uses Harmony), then log the patched
   methods so the support log records exactly what was hooked.

Two subtleties are worth calling out because real mods disagree on them:

- **Locale-vs-LoadSettings order is not universal.** MagicMail adds every locale source
  *before* `LoadSettings`
  ([`magic-mail` `repo/Mod.cs#L88-L108`](../../vice-and-order-research/mods/dossiers/magic-mail/repo/Mod.cs), commit `6fb3d2b`),
  while Outside Traffic Adjuster registers the Options UI and the `en-US` source *before*
  `LoadSettings`
  ([`outside-traffic-adjuster` `repo/Mod.cs#L24-L27`](../../vice-and-order-research/mods/dossiers/outside-traffic-adjuster/repo/Mod.cs), commit `42afd29`).
  Both work because `LoadSettings` mutates the *same* `Setting` object the sources already
  captured by reference; what must not vary is that `new Setting(...)` comes first.
- **Some mods do file-system setup in `OnLoad`.** Realistic Path Finding creates its
  `ModsSettings/<id>` folder and force-writes a default `.coc` if none exists
  ([`realistic-path-finding` `repo/Mod.cs#L28-L31`, `#L104-L112`](../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Mod.cs), commit `50645fa`).
  This is a defensive measure so downstream code can assume the settings file is present.

## Worked example: a minimal two-system mod (MagicMail)

MagicMail is a compact, current (game 1.6.0f1, commit `6fb3d2b`) example of the full
sequence. Its `OnLoad`
([`magic-mail` `repo/Mod.cs#L67-L115`](../../vice-and-order-research/mods/dossiers/magic-mail/repo/Mod.cs)):

- logs a one-time banner and null-checks `GameManager.instance` (`#L70-L81`);
- constructs `Setting setting = new Setting(this)` and stashes it in a static property so
  systems and locale classes can read it (`#L84-L85`);
- registers thirteen locale sources through a defensive wrapper - `AddLocaleSource` catches
  exceptions so a fragile third-party locale mod cannot crash the load
  (`#L88-L102`, helper at `#L139-L161`);
- calls `AssetDatabase.global.LoadSettings(ModId, setting, new Setting(this))` (`#L105`) -
  note the third argument is a *separate* defaults instance;
- calls `setting.RegisterInOptionsUI()` (`#L108`);
- schedules its two systems into `GameSimulation` (`#L113-L114`).

The static `Settings` property (`#L50-L53`) is the idiom that lets systems reach
configuration without a service locator: the mod owns the single instance, everything else
reads it.

## Worked example: a six-system tool mod (Road Speed Adjuster)

Road Speed Adjuster shows the same skeleton scaling to a tool + rendering + save-data mod
([`road-speed-adjuster` `repo/Mod.cs#L21-L54`](../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/Mod.cs), commit `e0c0c0b`).
The settings block is identical in spirit (`new Setting`, `RegisterInOptionsUI`, `AddSource`,
`LoadSettings` at `#L28-L33`), and then six systems are placed across five distinct phases
(`#L36-L51`). It also demonstrates `SetShowsErrorsInUI(true)` on its logger (`#L16-L17`) -
a deliberate choice for a tool where surfacing errors to the player is useful, versus the
`false` most background mods use.

The system-placement details of that example belong to [System Scheduling](./system-scheduling.md);
what matters here is that the *initialization frame* is unchanged - more systems, same order.

## Getting the world and disabling vanilla systems

When a mod replaces vanilla behaviour it usually fetches the running world and disables the
stock system before scheduling its replacement. The world is
`World.DefaultGameObjectInjectionWorld`, and a system is disabled by resolving it and
setting `.Enabled = false`. Realistic Path Finding disables four vanilla systems this way
before scheduling its own
([`realistic-path-finding` `repo/Mod.cs#L54-L58`](../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Mod.cs), commit `50645fa`):

```csharp
World.DefaultGameObjectInjectionWorld.GetOrCreateSystemManaged<Game.Simulation.ResidentAISystem>().Enabled = false;
World.DefaultGameObjectInjectionWorld.GetOrCreateSystemManaged<Game.Simulation.TripNeededSystem>().Enabled = false;
World.DefaultGameObjectInjectionWorld.GetOrCreateSystemManaged<Game.Simulation.ResourceBuyerSystem>().Enabled = false;
```

Traffic Tool Essentials caches `updateSystem.World` and resolves systems from it, then
toggles the vanilla and patched systems together in a `SetCompatibilityMode` helper
([`traffic-tool-essentials` `repo/Mod.cs#L120-L125`, `#L247-L256`](../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Mod.cs), commit `1097359`).
Using `GetOrCreateSystemManaged` (rather than `GetExistingSystemManaged`) is intentional:
it guarantees the target system exists before you flip its flag or order against it.

The full disable-then-replace pattern - and the ordering guarantees that make it safe - is
the subject of [System Scheduling](./system-scheduling.md) and the recipe
[`recipes/ecs-system-replacement-ordering.md`](../how-to/recipes/ecs-system-replacement-ordering.md).

## Harmony bootstrap (only if the mod patches methods)

Not every mod uses Harmony - scheduling and disabling vanilla systems handles most needs -
but when a mod must rewrite a method body it applies patches as the *last* step of `OnLoad`,
after settings and systems are in place. The bootstrap shape is: create a `Harmony` instance
keyed by a stable id, `PatchAll` the mod's assembly, then log what got patched. Realistic
Path Finding does exactly this
([`realistic-path-finding` `repo/Mod.cs#L18`, `#L81-L86`](../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Mod.cs), commit `50645fa`):

```csharp
public static readonly string harmonyID = "RealisticPathFinding";
// ...in OnLoad, after systems are scheduled:
var harmony = new Harmony(harmonyID);
harmony.PatchAll(typeof(Mod).Assembly);
var patchedMethods = harmony.GetPatchedMethods().ToArray();
log.Info($"Plugin {harmonyID} made patches! Patched methods: " + patchedMethods);
```

Two things matter about this block:

- **A stable, mod-unique `harmonyID`.** It doubles as the key you pass to `UnpatchAll` in
  `OnDispose`, so reloading the mod removes exactly its own patches and nothing else.
- **Logging the patched methods** records in the support log precisely what was hooked -
  invaluable when a later game patch moves or removes a target method.

Patching last is deliberate: the methods you hook may depend on systems or settings that the
earlier `OnLoad` steps established.

## OnDispose: symmetric teardown

`OnDispose` should undo the *global* registrations `OnLoad` performed - not the ECS
systems, which the world tears down on its own. In practice this is short. The common
minimum is to unregister the Options UI and drop the settings reference:

```csharp
public void OnDispose()
{
    log.Info(nameof(OnDispose));
    if (m_Setting != null)
    {
        m_Setting.UnregisterInOptionsUI();
        m_Setting = null;
    }
}
```

This exact shape appears in Outside Traffic Adjuster
([`repo/Mod.cs#L32-L40`](../../vice-and-order-research/mods/dossiers/outside-traffic-adjuster/repo/Mod.cs), commit `42afd29`),
Road Speed Adjuster
([`repo/Mod.cs#L56-L64`](../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/Mod.cs), commit `e0c0c0b`),
Realistic Path Finding
([`repo/Mod.cs#L94-L102`](../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Mod.cs), commit `50645fa`),
and MagicMail
([`repo/Mod.cs#L120-L129`](../../vice-and-order-research/mods/dossiers/magic-mail/repo/Mod.cs), commit `6fb3d2b`).

If the mod applied Harmony patches it should also `UnpatchAll(harmonyId)` here, using the
same id it patched with, so a reload does not double-patch. Realistic Path Finding patches in
`OnLoad` at
[`repo/Mod.cs#L81-L86`](../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Mod.cs)
but its `OnDispose` at commit `50645fa` only unregisters the Options UI
([`repo/Mod.cs#L94-L102`](../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Mod.cs)) -
it does *not* unpatch. That is survivable because CS2 rebuilds the world on reload, but the
symmetric `UnpatchAll(harmonyId)` is the safer, leak-free posture and what this handbook
recommends.

Notably, none of these four mods restore the vanilla systems they disabled in `OnDispose`.
In current CS2 practice the world is rebuilt on mod reload, so re-enabling disabled systems
is generally unnecessary - but if you keep a reference to a disabled system for a debug
toggle, this is where you would flip it back.

## Common pitfalls

- **Reading settings before `LoadSettings`.** The `Setting` object exists after `new
  Setting(...)`, but its fields hold *code defaults* until `LoadSettings` runs. Any logic
  that branches on user preferences must come after step 4.
- **Registering the Options UI before locales.** Labels resolve at paint time; if the
  source is not yet added they fall back to raw locale IDs.
- **Forgetting the defaults argument.** `LoadSettings(id, setting, defaults)` needs a
  distinct defaults instance (`new Setting(this)`) so a "reset to defaults" action has a
  clean baseline to copy from.
- **Non-symmetric teardown.** Anything global you register in `OnLoad` (Options UI, Harmony)
  must be undone in `OnDispose`, or reloading the mod leaks or double-registers.

## See also

- [System Scheduling](./system-scheduling.md) - the second half of `OnLoad`: phases,
  ordering, and update-interval tuning.
- [Module Layout](./module-layout.md) - how the `Mod.cs` entry point is named and organised.
- [Dependency Strategy](./dependency-strategy.md) - the detect-dependencies step of `OnLoad`
  in depth.
- Settings and Data Management (pending rework) - the `Setting` class and `.coc`
  persistence in depth.
- Recipes: [ECS system replacement ordering](../how-to/recipes/ecs-system-replacement-ordering.md),
  [Periodic UpdateInterval system](../how-to/recipes/periodic-updateinterval-system.md).
- [Technique Index](../technique-index.md) - coverage ledger for these techniques.
