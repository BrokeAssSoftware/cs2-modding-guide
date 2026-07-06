---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Case study: Magic Garbage Truck"
case_study: magic-garbage-truck
mod: "Magic Garbage Truck"
dossier: ../../vice-and-order-research/mods/dossiers/magic-garbage-truck/
repo_commit: 1b6a478753e1ef4e43ac9b90d567f3d7183c7be2
source_version: "1.6.0f1 (magic-garbage-truck@1b6a478; date-pinned, static source only)"
last_reverified: "2026-07-05"
diataxis: explanation
techniques: [AM, J, L, AX]
technique_applicability: [simulation, economy, ui]
status: source-verified
Created: 2026-07-05
Updated: 2026-07-05
Owners:
  - codex
---

# Magic Garbage Truck - case study

> A garbage-service override suite built from a handful of tiny, near-zero-idle-cost
> ECS systems - a permanently-enabled burst auto-cleaner, several one-shot prefab
> rescalers that sleep the instant they finish, a self-restoring live-parameter
> assist, and a status panel that computes only while the Options menu is open.

All code claims below are cited to `repo/<path>#Lxx` at pinned commit
`1b6a478753e1ef4e43ac9b90d567f3d7183c7be2` (mod id `MagicGarbage`, tag `[MG]`),
surfaced through the dossier at
`../../vice-and-order-research/mods/dossiers/magic-garbage-truck/`. Every type and
method named here was verified via `git show <pin>:<path>` at that commit.

## What it does / why it's instructive

Magic Garbage (author River-Mochi) offers three player-facing modes that are
mutually exclusive at the setting level: **Total Magic** (garbage simply
disappears on a cadence, for players who do not want to manage waste at all),
**Trash Boss** (leave vanilla garbage mechanics on but rescale trucks, facilities,
thresholds, and accumulation so the service copes), and a pure **Status** report
mode where neither override runs and the mod only observes. It is a service-tuning
mod, not a simulation replacement: it never disables a vanilla system and ships no
Harmony patch.

It is instructive precisely because it is *small*. Where a maximal mod clones and
replaces vanilla systems, Magic Garbage shows the opposite discipline - a set of
cheap `GameSystemBase` systems that each do exactly one job and then get out of the
way. It is a clean tour of four "good citizen" techniques: a burst job on a long
`GetUpdateInterval`, one-shot prefab multipliers that self-disable, an override
that restores its own baseline *before* it sleeps, and a read-only status binding
that has an empty `OnUpdate` and only builds a snapshot when the UI reads it. These
are the patterns that keep a mod's idle CPU cost effectively nil.

## Architecture at a glance

`Mod.OnLoad` registers seven systems, all into `SystemUpdatePhase.GameSimulation`
via `updateSystem.UpdateAfter<T>` (repo/Mod.cs#L124-L133):
`GarbageTruckCapacitySystem`, `GarbageFacilityCapacitySystem`,
`GarbageThresholdSystem`, `GarbagePriorityAssistSystem`,
`GarbageAccumulationRateSystem`, `GarbageStatusSystem`, and `TotalMagicSystem`.
There is no explicit ordering beyond registration order, and a source comment warns
against adding pre-dispatch garbage-request suppression for Total Magic because it
floods `Player.log` with "UpdateFrame added to unsupported type" spam
(repo/Mod.cs#L134). Settings are constructed and loaded before the systems are
scheduled (repo/Mod.cs#L101-L121).

The systems fall into four behavioural classes:

- **Permanently-enabled burst cleaner** - `TotalMagicSystem` never self-sleeps
  (`Enabled = true` in `OnCreate`, repo/Systems/TotalMagicSystem.cs#L71); it relies
  on a long update interval plus a cheap early-return instead.
- **One-shot prefab rescalers** - `GarbageTruckCapacitySystem`,
  `GarbageFacilityCapacitySystem`, `GarbageAccumulationRateSystem`,
  `GarbageThresholdSystem` all start disabled, wake on city load or a settings
  callback, apply once, and set `Enabled = false` again.
- **Self-restoring live assist** - `GarbagePriorityAssistSystem` runs on a frame
  cadence while active and restores the vanilla threshold before it disables.
- **Zero-steady-state observer** - `GarbageStatusSystem` has an empty `OnUpdate`
  and builds its snapshot only when the Options UI pulls it.

### Burst auto-clean (Total Magic)

`TotalMagicSystem` runs on a coarse cadence: `GetUpdateInterval` returns
`TicksPerDay / UpdatesPerDay` = `262144 / 64` = 4096 ticks (~22.5 in-game minutes
per the source comment), and `UpdatesPerDay` is a tunable knob to raise if warning
icons slip through (repo/Systems/TotalMagicSystem.cs#L33-L44). Because the system
stays enabled, `OnUpdate` guards with a cheap early-return when settings are
missing or Total Magic is not the active mode (repo/Systems/TotalMagicSystem.cs#L81-L89).

When active it schedules a `[BurstCompile] IJobChunk` over every non-`Deleted`/
`Destroyed`/`Temp` `GarbageProducer` (repo/Systems/TotalMagicSystem.cs#L52-L64,
repo/Systems/TotalMagicSystem.cs#L124-L125). The job zeroes producer state and
removes the notification icon, using the producer->writer->dependency handshake
with the game's `IconCommandSystem`:

```csharp
IconCommandBuffer iconBuffer = m_IconCommandSystem.CreateCommandBuffer();
GarbageParameterData garbageParams =
    m_GarbageParamsQuery.GetSingleton<GarbageParameterData>();
JobHandle cleanHandle = new MagicJob
{
    m_EntityType = GetEntityTypeHandle(),
    m_GarbageProducerType = GetComponentTypeHandle<GarbageProducer>(false),
    m_IconCommandBuffer = iconBuffer,
    m_GarbageParameters = garbageParams
}.ScheduleParallel(m_GarbageProducerQuery, Dependency);
m_IconCommandSystem.AddCommandBufferWriter(cleanHandle);
Dependency = cleanHandle;
```
(repo/Systems/TotalMagicSystem.cs#L91-L105)

Inside the job, each producer has `m_Garbage`, `m_CollectionRequest`, and
`m_DispatchIndex` reset and the `GarbagePilingUpWarning` flag cleared, then
`m_IconCommandBuffer.Remove(building, m_GarbageParameters.m_GarbageNotificationPrefab)`
retires the icon; an already-clean producer still gets an idempotent icon removal
(repo/Systems/TotalMagicSystem.cs#L144-L161). The notification-prefab reference is
read from the `GarbageParameterData` singleton, not hard-coded
(repo/Systems/TotalMagicSystem.cs#L66, repo/Systems/TotalMagicSystem.cs#L93-L94).

### One-shot prefab rescalers (Trash Boss / Power User)

`GarbageTruckCapacitySystem` and `GarbageFacilityCapacitySystem` are the model
one-shot. Each starts disabled (`Enabled = false` in `OnCreate`,
repo/Systems/GarbageTruckCapacitySystem.cs#L46), wakes on a real city load
(`OnGameLoadingComplete` gated to `GameMode.Game` + `NewGame`/`LoadGame`,
repo/Systems/GarbageTruckCapacitySystem.cs#L49-L67), and uses
`GetUpdateInterval => 1` so it runs the very next simulation tick
(repo/Systems/GarbageTruckCapacitySystem.cs#L69-L72). The apply pass reads the
**authoring baseline** off the prefab rather than the current live value, so
rescaling is non-stacking:

```csharp
if (!m_PrefabSystem.TryGetPrefab(prefabEntity, out PrefabBase prefabBase) || prefabBase == null)
    return false;
if (!prefabBase.TryGet(out Game.Prefabs.GarbageTruck authoring))
    return false;
capacity = authoring.m_GarbageCapacity;
unloadRate = authoring.m_UnloadRate;
```
(repo/Systems/GarbageTruckCapacitySystem.cs#L128-L146)

The target percent defaults to `100` (a full restore to vanilla) and only takes the
slider value when Trash Boss is active and Total Magic is not
(repo/Systems/GarbageTruckCapacitySystem.cs#L89-L93). `ScalePercentKeepZero` scales
the baseline but leaves a genuinely-zero authoring value at zero and floors non-zero
results at 1 (repo/Systems/GarbageTruckCapacitySystem.cs#L148-L156). After writing,
the system sets `Enabled = false` (repo/Systems/GarbageTruckCapacitySystem.cs#L125).
`GarbageFacilityCapacitySystem` follows the identical shape over `GarbageFacilityData`
(vehicle/processing/storage, repo/Systems/GarbageFacilityCapacitySystem.cs#L100-L143)
and `GarbageAccumulationRateSystem` over `ConsumptionData.m_GarbageAccumulation`,
reading its baseline from either `ServiceConsumption` or `ZoneServiceConsumption`
authoring (repo/Systems/GarbageAccumulationRateSystem.cs#L124-L146).

`GarbageThresholdSystem` is the singleton-parameter variant: instead of a prefab
baseline it captures the live `GarbageParameterData` fields once into private fields
and treats those as vanilla (repo/Systems/GarbageThresholdSystem.cs#L75-L82). It
clamps the four Power-User values, writes them back to the singleton, and self-sleeps
once the singleton already equals the target (repo/Systems/GarbageThresholdSystem.cs#L84-L146).

### Self-restoring live assist

`GarbagePriorityAssistSystem` is the only override that stays live on a cadence
(`GetUpdateInterval => 128` frames, repo/Systems/GarbagePriorityAssistSystem.cs#L33,
repo/Systems/GarbagePriorityAssistSystem.cs#L70-L73). When any building crosses the
critical garbage threshold it temporarily raises `m_CollectionGarbageLimit` to the
live request limit to pull overloaded buildings forward
(repo/Systems/GarbagePriorityAssistSystem.cs#L240-L266). The instructive part is how
it turns *off*: it computes the correct "normal" collection value first, writes that
back to the singleton, and only then disables - so toggling the assist off never
leaves the threshold stuck-raised:

```csharp
if (data.m_CollectionGarbageLimit != targetCollect)
{
    data.m_CollectionGarbageLimit = targetCollect;
}

if (!assistAllowed && data.m_CollectionGarbageLimit == normalCollect)
{
    Enabled = false;
}
```
(repo/Systems/GarbagePriorityAssistSystem.cs#L263-L271)

When the assist is disallowed, `targetCollect` is already `normalCollect`
(repo/Systems/GarbagePriorityAssistSystem.cs#L241), so the restore write happens on
the same pass that flips `Enabled = false` - the guard on the disable is what makes
the restore-before-sleep ordering safe.

### Zero-steady-state status observer (AX)

`GarbageStatusSystem` starts disabled and has a deliberately **empty** `OnUpdate` -
it never does simulation work:

```csharp
// Snapshot system only in Options UI, no auto sim work needed on purpose, does not affect city performance.
protected override void OnUpdate()
{
}
```
(repo/Systems/GarbageStatusSystem.cs#L296-L299, disabled in `OnCreate` at
repo/Systems/GarbageStatusSystem.cs#L293)

All work lives in `BuildSnapshot()`, an on-demand ECS scan of producers, requests,
trucks, and facilities that returns an immutable `Snapshot` struct
(repo/Systems/GarbageStatusSystem.cs#L301-L729), plus `GetCriticalBuildings()`
(repo/Systems/GarbageStatusSystem.cs#L779-L807). A transfer-probe extension in a
separate file adds `GetGarbageTransferProbeEntries()` as a `partial` continuation of
the same system, kept split out so it can be removed cleanly later
(repo/Systems/GarbageTransferProbe.cs#L34, repo/Systems/GarbageTransferProbe.cs#L109-L120).

The bridge to the UI is the static `GarbageStatus` cache. `RefreshIfNeeded()` is
throttled twice: it early-returns if already refreshed this frame, and otherwise
enforces a 10-second minimum age before rebuilding
(repo/Systems/GarbageStatus.cs#L28, repo/Systems/GarbageStatus.cs#L57-L94). Every
Options-menu getter calls `RefreshIfNeeded()` and then returns the cached string, so
the snapshot is computed only while the panel is on screen
(repo/Settings/Setting.Status.cs#L24-L91); the report button forces
`GarbageStatus.RefreshNow(writeToLog: true)` and dumps a full report to the log
(repo/Settings/Setting.Status.cs#L110).

### Settings and mode wiring

The two mode toggles are mutually exclusive by construction: enabling Total Magic
clears Trash Boss and vice versa in the property setters
(repo/Settings/Setting.cs#L133-L135, repo/Settings/Setting.cs#L157-L159). Every
tuning control carries `[SettingsUISetter(...)]`; the callbacks re-wake the sleeping
one-shot systems by setting `Enabled = true`, e.g. `OnModeToggleChanged` wakes all
six override systems at once (repo/Settings/Setting.cs#L514-L556). Preset buttons are
grouped with `[SettingsUIButtonGroup]`; "Recommended" writes the recommended-constant
set and then calls `EnableTuningSystemsOnce()` + `Apply()`
(repo/Settings/Setting.cs#L216-L237, repo/Settings/Setting.cs#L671-L707).

## Techniques demonstrated

- [Periodic GameSystemBase w/ UpdateInterval tuning](../how-to/recipes/periodic-updateinterval-system.md)
  (family L) - `TotalMagicSystem.GetUpdateInterval` returns `262144 / UpdatesPerDay`
  (4096 ticks) and the system stays permanently enabled, leaning on the coarse
  cadence + an early-return rather than self-sleeping
  (repo/Systems/TotalMagicSystem.cs#L33-L44, repo/Systems/TotalMagicSystem.cs#L71,
  repo/Systems/TotalMagicSystem.cs#L81-L89).
- [Notification-icon via IconCommandBuffer](../how-to/recipes/icon-command-buffer.md)
  (family AM) - the burst clean job removes the garbage-pile-up icon through
  `IconCommandSystem.CreateCommandBuffer()` -> `AddCommandBufferWriter(handle)` ->
  `Dependency = handle`, using the notification prefab read from the
  `GarbageParameterData` singleton (repo/Systems/TotalMagicSystem.cs#L91-L105,
  repo/Systems/TotalMagicSystem.cs#L149).
- [One-shot non-stacking prefab multiplier](../how-to/recipes/one-shot-prefab-multiplier.md)
  (family J) - truck/facility/accumulation systems read the `PrefabBase` authoring
  baseline via `PrefabSystem.TryGetPrefab` + `prefabBase.TryGet<...>`, scale with
  `ScalePercentKeepZero`, and set `Enabled = false` after one pass; re-woken by
  `SettingsUISetter` callbacks (repo/Systems/GarbageTruckCapacitySystem.cs#L128-L156,
  repo/Systems/GarbageTruckCapacitySystem.cs#L125, repo/Settings/Setting.cs#L514-L556).
- [Read-only status/observability binding](../how-to/recipes/status-observability-binding.md)
  (family AX) - `GarbageStatusSystem` has an empty `OnUpdate`; the snapshot is built
  on demand and cached behind a per-frame + 10-second throttle that Options getters
  drive (repo/Systems/GarbageStatusSystem.cs#L296-L299,
  repo/Systems/GarbageStatus.cs#L57-L94, repo/Settings/Setting.Status.cs#L24-L91).

The self-restoring disable in `GarbagePriorityAssistSystem` is a supporting variation
of the reversible-override idea (see
[reversible override baseline](../how-to/recipes/reversible-override-baseline.md)):
restore the vanilla value *before* the system sleeps
(repo/Systems/GarbagePriorityAssistSystem.cs#L263-L271).

## Key decisions & tradeoffs

- **Coarse cadence over self-sleep for the auto-cleaner.** Total Magic must react to
  any building that reaccumulates garbage, so it cannot sleep the way the one-shots
  do. Instead it runs a burst pass only every 4096 ticks and pays a near-free
  early-return the rest of the time; `UpdatesPerDay` is the exposed dial if warning
  icons ever appear between sweeps (repo/Systems/TotalMagicSystem.cs#L33-L44).
- **Authoring baseline, never the live value.** Every rescaler multiplies the prefab
  *authoring* value, so repeated applies and mode flips are idempotent and always
  restorable to 100% - there is no accumulating drift from reading a
  previously-scaled value (repo/Systems/GarbageTruckCapacitySystem.cs#L128-L146,
  repo/Systems/GarbageTruckCapacitySystem.cs#L89-L93).
- **Restore-before-sleep.** The priority assist writes the normal collection limit
  back to the `GarbageParameterData` singleton and only disables once the singleton
  matches it, so turning the assist off can never strand the threshold in its raised
  state (repo/Systems/GarbagePriorityAssistSystem.cs#L263-L271).
- **Compute status only when the Options panel is open.** The status system does zero
  per-frame work; the snapshot is a UI-pull with a double throttle, so an idle mod in
  a paused menu-less session costs nothing (repo/Systems/GarbageStatusSystem.cs#L296-L299,
  repo/Systems/GarbageStatus.cs#L57-L94).
- **No Harmony, no system disable.** The mod coexists with vanilla garbage mechanics
  by mutating shared prefab/singleton data rather than replacing systems, which keeps
  its per-patch maintenance surface tiny - at the cost of last-write-wins interactions
  with any peer that touches the same data (see pitfalls).

## Pitfalls / upstream-watch

- **Double-clamp of the same thresholds.** Both `GarbageThresholdSystem` and
  `GarbagePriorityAssistSystem` independently clamp `GarbageDispatchRequestThreshold`
  / `GarbagePickupThreshold` against the same `Setting` constants and re-apply the
  "pickup cannot exceed request" rule; the clamp logic is duplicated in two systems
  and must be kept in sync (repo/Systems/GarbageThresholdSystem.cs#L89-L115,
  repo/Systems/GarbagePriorityAssistSystem.cs#L151-L173).
- **Last-write-wins on `GarbageParameterData`.** `GarbageThresholdSystem`,
  `GarbagePriorityAssistSystem`, and any external mod all write
  `m_CollectionGarbageLimit` on the same singleton. The two MG systems are ordered
  only by registration (threshold before priority assist,
  repo/Mod.cs#L126-L127), so the last writer in a tick wins; a peer mod editing the
  same singleton can silently override MG or be overridden by it
  (repo/Systems/GarbageThresholdSystem.cs#L134-L137,
  repo/Systems/GarbagePriorityAssistSystem.cs#L263-L266).
- **Notification-icon leak if the interval is too long.** Because the burst cleaner
  only removes icons on its sweep, a long enough interval can let the
  `GarbagePilingUpWarning` icon appear between sweeps; the source itself flags
  `UpdatesPerDay` as the knob to raise (repo/Systems/TotalMagicSystem.cs#L34).
- **Do not add pre-dispatch request suppression.** A first-party source comment warns
  that suppressing garbage requests before dispatch for Total Magic causes severe
  `Player.log` spam ("UpdateFrame added to unsupported type") - a trap for anyone
  extending the auto-cleaner (repo/Mod.cs#L134).
- **Discord/Paradox URL drift.** The About links are hard-coded consts
  (`UrlDiscord = "https://discord.gg/gwXgvtyhjc"`,
  repo/Settings/Setting.cs#L89-L90; `UrlParadox` at repo/Settings/Setting.cs#L86-L87)
  and will silently rot if the invite or author page moves.

Needs Verification (in-game): the actual in-game minutes per sweep (the ~22.5 figure
is a source comment, repo/Systems/TotalMagicSystem.cs#L33), whether a given
`UpdatesPerDay` fully prevents warning icons on a large city, and the real-world
outcome of two mods contending for `GarbageParameterData` in the same tick.

## Source pointers

- Dossier:
  `../../vice-and-order-research/mods/dossiers/magic-garbage-truck/`
  (`index.md`, `source.md`, `modding.md`, `guide.md`, `notes/`).
- Repo @ `1b6a478753e1ef4e43ac9b90d567f3d7183c7be2`, key files:
  - `repo/Mod.cs` - `OnLoad` system registration into `GameSimulation`.
  - `repo/Systems/TotalMagicSystem.cs` - permanently-enabled burst cleaner + icon buffer.
  - `repo/Systems/GarbageTruckCapacitySystem.cs`,
    `.../GarbageFacilityCapacitySystem.cs`,
    `.../GarbageAccumulationRateSystem.cs` - one-shot authoring-baseline rescalers.
  - `repo/Systems/GarbageThresholdSystem.cs` - singleton-parameter one-shot.
  - `repo/Systems/GarbagePriorityAssistSystem.cs` - self-restoring live assist.
  - `repo/Systems/GarbageStatusSystem.cs`, `.../GarbageStatus.cs`,
    `.../GarbageTransferProbe.cs` - empty-`OnUpdate` observer + cached UI binding + probe.
  - `repo/Settings/Setting.cs`, `.../Setting.Status.cs` - mutually-exclusive modes,
    `SettingsUISetter` wakeups, preset button groups, status getters.
