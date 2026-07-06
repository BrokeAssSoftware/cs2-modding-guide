---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: One-shot non-stacking prefab multiplier"
recipe: one-shot-prefab-multiplier
technique_family: "J - One-shot non-stacking prefab multiplier (PrefabBase baseline)"
diataxis: how-to
source_version: "1.6.0f1 (magic-mail@6fb3d2b; date-pinned, static source only)"
last_reverified: "2026-07-04"
status: source-verified
canonical_mods:
  - magic-mail@6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47
  - traffic-tool-essentials@10973595ac9ed37f47ca24eb788d4421dc295fa0
  - better-bulldozer@4408466f226db811159d92859479ae1e1c28ba06
  - magic-garbage-truck@1b6a478753e1ef4e43ac9b90d567f3d7183c7be2
  - realistic-path-finding@50645fa6a078181365e36a42e2e27b96699bf02a
technique_applicability: [economy, simulation]
Created: 2026-07-04
Updated: 2026-07-05
Owners:
  - codex
---

# One-shot non-stacking prefab multiplier (PrefabBase baseline)

> Scale a prefab field by a user multiplier that can be re-applied any number of
> times and never compounds - by reading the vanilla baseline off the immutable
> authoring `PrefabBase` each time, then applying it once from a self-disabling
> system.

## Problem
You want a slider that multiplies a prefab value (post-van mail capacity, sorting
rate, facility fleet size) and takes effect the instant the user drags it. The
naive version reads the live ECS component, multiplies *that*, and writes it back -
so x1.5 applied twice becomes x2.25, and every settings re-apply drifts the value
further. You need the multiply to be **idempotent**: applying x1.5 ten times must
still be exactly x1.5 of vanilla. You also do not want a system burning cycles every
frame just to be ready for an occasional slider change.

## Solution
Never scale the live value - scale an **immutable baseline** that you re-read every
time. The vanilla authoring numbers live on the managed `PrefabBase` object
(`prefabBase.TryGet(out PostVan van)` gives you `van.m_MailCapacity` as authored),
which the game does not mutate when you edit the ECS component. So each apply pass:
resolve `PrefabBase` from the prefab entity, read the authoring baseline, multiply
*that*, floor the result at 1, and write onto the ECS component. Because the source
of truth is the untouched `PrefabBase`, re-applying is exact and drift-free. Run it
from a system that is `Enabled = false` by default, gets flipped on by
`Setting.Apply()` (and on city load), does its one pass, and disables itself again.

This is the drift-resistant sibling of [prefab-field override](prefab-field-override.md)
(family A): family A multiplies a **hard-coded vanilla literal** (`0.3f * mult`);
this recipe multiplies the **live authoring baseline off `PrefabBase`**, so a patch
that changes vanilla values cannot silently make your scaling wrong.

## Steps & Code

### 1. Cache `PrefabSystem` and RW prefab queries; start disabled

In `OnCreate`, grab `PrefabSystem`, build queries over the prefab-data components you
will scale, and set `Enabled = false` so nothing runs until settings or a load wakes
you:

```csharp
m_PrefabSystem = World.GetOrCreateSystemManaged<PrefabSystem>();

m_PostVansQuery = SystemAPI.QueryBuilder()
    .WithAll<PrefabData>()
    .WithAllRW<PostVanData>()
    .Build();

RequireForUpdate(m_PostFacilitiesQuery);
RequireForUpdate(m_PostVansQuery);

// Run only when settings change or after a city load.
Enabled = false;
```
Source: `../../../vice-and-order-research/mods/dossiers/magic-mail/repo/Systems/MailCapacitySystem.cs#L44-L62` (@6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47)

### 2. Read the vanilla baseline off `PrefabBase` (the whole trick)

This is what makes the multiply non-stacking. Resolve the managed `PrefabBase` from
the prefab entity, pull the authored component, and return its field as the
baseline. The game never overwrites this authoring value, so it is a stable origin
on every pass:

```csharp
private bool TryGetPostVanBaseMailCapacity(Entity prefabEntity, out int baseMailCapacity)
{
    baseMailCapacity = 0;
    if (!m_PrefabSystem.TryGetPrefab(prefabEntity, out PrefabBase prefabBase))
        return false;
    if (!prefabBase.TryGet(out Game.Prefabs.PostVan postVan))
        return false;
    baseMailCapacity = postVan.m_MailCapacity;   // immutable vanilla authoring value
    return baseMailCapacity > 0;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/magic-mail/repo/Systems/MailCapacitySystem.cs#L212-L228` (@6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47)

For a facility, the same shape reads a whole struct of baselines
(`m_PostVanCapacity`, `m_PostTruckCapacity`, `m_MailStorageCapacity`, `m_SortingRate`)
off `PostFacility` via one `prefabBase.TryGet(out Game.Prefabs.PostFacility ...)`
(`MailCapacitySystem.cs#L230-L253`).

### 3. Scale the baseline (never the live value) and floor at 1

Multiply the baseline by the percent, round, and clamp to a minimum of 1 so a
positive value can never collapse to a broken zero:

```csharp
private static int ScalePercentMin1(int baseValue, int percent)
{
    if (baseValue <= 0)
        return 0;
    return math.max(1, (int)math.round(baseValue * percent / 100f));
}
```
Source: `../../../vice-and-order-research/mods/dossiers/magic-mail/repo/Systems/MailCapacitySystem.cs#L255-L263` (@6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47)

### 4. Write the scaled value onto the live component via `RefRW`

Iterate the prefab entities, read the baseline for each, scale it, and write onto the
component through `RefRW.ValueRW`. The write is gated on inequality so an
already-correct prefab is not re-touched:

```csharp
foreach ((RefRW<PostVanData> vanRef, Entity prefabEntity) in SystemAPI
             .Query<RefRW<PostVanData>>()
             .WithAll<PrefabData>()
             .WithEntityAccess())
{
    ref PostVanData vanData = ref vanRef.ValueRW;
    if (!TryGetPostVanBaseMailCapacity(prefabEntity, out int baseMailCapacity))
        continue;                                   // silent skip on prefab miss

    int newMailCapacity = ScalePercentMin1(baseMailCapacity, vanMailPercent);
    if (vanData.m_MailCapacity != newMailCapacity)
        vanData.m_MailCapacity = newMailCapacity;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/magic-mail/repo/Systems/MailCapacitySystem.cs#L134-L151` (@6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47)

This writes through the query's `RefRW` on the prefab-data component directly, not
`PrefabSystem.AddComponentData` as in family A - both commit prefab changes; the
query path is natural when you are already iterating.

### 5. Do the pass once, then disable yourself again

`OnUpdate` bails out (and disables) outside a real game, does the two apply calls,
then sets `Enabled = false` on the way out so it costs nothing until next woken:

```csharp
ApplyPostVanPayload(vanMailPercent);
ApplyPostFacilityValues(vanFleetPercent, truckFleetPercent,
    sortingSpeedPercent, sortingStoragePercent);

// Back to disabled until Setting.Apply() wakes us again.
Enabled = false;
```
Source: `../../../vice-and-order-research/mods/dossiers/magic-mail/repo/Systems/MailCapacitySystem.cs#L121-L129` (@6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47)

`GetUpdateInterval` returns `1` so that when enabled it runs on the very next tick
rather than waiting out a long phase interval
(`MailCapacitySystem.cs#L82-L85`), and `OnGameLoadingComplete` only flips
`Enabled = true` for `GameMode.Game` with `Purpose.NewGame`/`Purpose.LoadGame`
(`MailCapacitySystem.cs#L65-L77`).

### 6. Wake the system from `Setting.Apply()`

The settings object re-enables the system every time the user applies settings, which
is what makes slider changes feel instant:

```csharp
public override void Apply()
{
    base.Apply();
    World? world = World.DefaultGameObjectInjectionWorld;
    if (world == null || !world.IsCreated) return;

    MailCapacitySystem? capacitySystem =
        world.GetExistingSystemManaged<MailCapacitySystem>();
    if (capacitySystem != null)
        capacitySystem.Enabled = true;   // one pass, then it disables itself again
}
```
Source: `../../../vice-and-order-research/mods/dossiers/magic-mail/repo/Settings/Settings.cs#L108-L133` (@6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47)

## Pitfalls & gotchas

- **Scaling the live value stacks; scaling the `PrefabBase` baseline does not.** This
  is the entire reason the recipe exists. If you read `vanData.m_MailCapacity`,
  multiply it, and write it back, every apply compounds. Magic Mail instead re-reads
  `postVan.m_MailCapacity` off `PrefabBase` (untouched by your edits) as the origin
  each pass, so ten applies of x1.5 all land on exactly 1.5x vanilla
  (`MailCapacitySystem.cs#L226`, `#L146`).

- **Floor positive values at 1.** A percent slider plus integer rounding can drive a
  small baseline to 0, which can break the facility. `ScalePercentMin1` clamps with
  `math.max(1, ...)` but only for `baseValue > 0`, so a genuinely-zero baseline stays
  zero (`MailCapacitySystem.cs#L255-L263`). Note: at this pin `ScalePercentMin1` and
  `ScalePercentKeepZero` are byte-for-byte identical implementations
  (`MailCapacitySystem.cs#L265-L273`) - the two names signal *intent* only, so do not
  assume `KeepZero` behaves differently here.

- **Silent prefab-miss.** `TryGetPostVanBaseMailCapacity` returning false just
  `continue`s the loop - no log, no fallback. If a DLC/patch renames or removes the
  prefab, or the baseline is <= 0, the scaling silently no-ops
  (`MailCapacitySystem.cs#L141-L144`). Add a log on the false branch if you need to
  know.

- **Only two mods in the canonical set read the true baseline off `PrefabBase`.**
  Magic Mail (this recipe) and Magic Garbage Truck (see Variations) both resolve the
  authoring component off `PrefabBase` to make the multiply non-stacking. Traffic Tool
  Essentials and Better Bulldozer demonstrate the *self-disabling one-shot lifecycle*
  (below) but do not scale a baseline. Do not assume other prefab-editing mods are
  drift-resistant - most multiply a literal (family A).

- **You are editing the PREFAB - all instances change.** As with any prefab edit this
  mutates the shared template, so every current and future post van/facility picks up
  the scaled value. That is intended for balance sliders, not for targeting one placed
  building.

- **Re-verify the authoring field names each game version.** `m_MailCapacity`,
  `m_PostVanCapacity`, `m_MailStorageCapacity`, `m_SortingRate` are read off the
  vanilla `PostVan`/`PostFacility` prefab components; a patch renaming or retyping them
  would break `TryGet`. Confirm against `Game.Prefabs` at the target version. Whether
  the scaled capacity behaves correctly in the running simulation is `Needs
  Verification (in-game)`.

## Variations

- **Self-disabling one-shot lifecycle without a multiplier (Better Bulldozer).** The
  same "disabled by default -> flipped on by a settings button -> does one pass ->
  disables itself" shape drives a one-time cleanup instead of a scale. `OnCreate` sets
  `Enabled = false`; a `[SettingsUIButton]` setter flips it on; `OnUpdate` schedules
  its job once and self-disables:

  ```csharp
  // Settings button turns the one-shot system on:
  public bool RemovedOwnedGrassSurfaces
  {
      set => World.DefaultGameObjectInjectionWorld
          .GetOrCreateSystemManaged<RemoveExistingOwnedGrassSurfaces>().Enabled = true;
  }
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/better-bulldozer/repo/BetterBulldozer/Settings/BetterBulldozerModSettings.cs#L57` (@4408466f226db811159d92859479ae1e1c28ba06)

  The system itself sets `Enabled = false` in `OnCreate`
  (`RemoveExistingOwnedGrassSurfaces.cs#L86`) and again at the end of `OnUpdate` after
  scheduling its delete job (`RemoveExistingOwnedGrassSurfaces.cs#L119`) - identical
  lifecycle, non-scaling payload.

- **Self-disable as a fail-safe, not just completion (Traffic Tool Essentials).** A
  system can also disable itself defensively when a required dependency is missing, so
  it never runs half-wired:

  ```csharp
  if (m_PathfindSetupSystem == null || m_DepotZoneSystem == null || m_SimulationSystem == null)
  {
      Mod.LogError("[DepotZoneDispatch] V401.12: CRITICAL - System reference is null, disabling!");
      Enabled = false;
      return;
  }
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Systems/DepotZone/DepotZoneDispatchSystem.cs#L297-L301` (@10973595ac9ed37f47ca24eb788d4421dc295fa0)

  Same `Enabled = false` mechanism, opposite trigger: this one guards against a broken
  setup rather than marking work complete.

- **Second `PrefabBase`-baseline exemplar with a per-slider wake (Magic Garbage
  Truck).** Same trick, different domain: `TryGetAuthoringBase` resolves the prefab,
  reads `GarbageTruck.m_GarbageCapacity`/`.m_UnloadRate` off the authoring component,
  and scales *that* through `ScalePercentKeepZero` (identical guard shape to Magic
  Mail's `ScalePercentMin1` - `<= 0` returns 0, positives floor at 1):

  ```csharp
  if (!prefabBase.TryGet(out Game.Prefabs.GarbageTruck authoring))
      return false;
  capacity = authoring.m_GarbageCapacity;   // immutable vanilla authoring value
  unloadRate = authoring.m_UnloadRate;
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/magic-garbage-truck/repo/Systems/GarbageTruckCapacitySystem.cs#L128-L146` (@1b6a478753e1ef4e43ac9b90d567f3d7183c7be2)

  It self-disables at the end of the apply pass
  (`GarbageTruckCapacitySystem.cs#L125`), and where Magic Mail wakes on the global
  `Setting.Apply()`, this mod wakes *per slider* via `[SettingsUISetter]`: the
  `GarbageTruckCapacityMultiplier` property carries
  `[SettingsUISetter(typeof(Setting), nameof(OnTruckSliderChanged))]`
  (`Settings/Setting.cs#L170-L174`), and `OnTruckSliderChanged` flips only the truck
  system's `Enabled = true` (`Settings/Setting.cs#L558-L570`) - a tighter wake that
  re-runs just the affected one-shot instead of every settings system.

- **No `PrefabBase` to read? Capture the live baseline once (singleton PARAMETER
  tuning).** When your target is a singleton component rather than a prefab template,
  there is no immutable authoring copy to re-read. Magic Garbage Truck's
  `GarbageThresholdSystem` scales the `GarbageParameterData` singleton, so it snapshots
  the vanilla values on the *first* pass under an `m_HaveBase` guard and treats that
  cache as the origin thereafter:

  ```csharp
  ref GarbageParameterData data = ref parameters.ValueRW;
  if (!m_HaveBase)
  {
      m_BaseCollectLimit = math.max(1, data.m_CollectionGarbageLimit);
      m_BaseRequestLimit = math.max(1, data.m_RequestGarbageLimit);
      m_HaveBase = true;                     // snapshot once; never re-read the live value
  }
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/magic-garbage-truck/repo/Systems/GarbageThresholdSystem.cs#L73-L82` (@1b6a478753e1ef4e43ac9b90d567f3d7183c7be2)

  Each pass then clamps the user sliders into vanilla..max ranges and writes off the
  cached baseline (`GarbageThresholdSystem.cs#L89-L114`), so re-applying is still
  non-stacking - the guarantee comes from the one-time capture instead of from
  `PrefabBase`. Caveat: the snapshot is only reset on `OnGameLoadingComplete`
  (`GarbageThresholdSystem.cs#L44-L54`), so if another mod has already mutated the
  singleton before the first pass, the "baseline" you capture is that mutated value.

- **Epsilon-suppressed writes when the target is a live graph, not a prefab (Realistic
  Path Finding).** The integer write-gate in step 4 (`!=` before write) has a float
  analog for live-simulation data. RPF rescales the live pedestrian pathfinding graph,
  whose lane edges keep no stored original, so it caches the last-applied factor per
  lane and compares floats through an epsilon helper rather than `==`:

  ```csharp
  internal const float DefaultEpsilon = 1e-4f;
  internal static bool AlmostEqual(float a, float b, float epsilon = DefaultEpsilon)
      => math.abs(a - b) < epsilon;
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Utils/PathfindCostUtils.cs#L8-L19` (@50645fa6a078181365e36a42e2e27b96699bf02a)

  A settings pass that resolves to the same factor is skipped entirely
  (`AlmostEqual(newFactor, _lastKnownFactor)` before flipping `Enabled = true`,
  `PedestrianWalkCostFactorSystem.cs#L73-L80`), and the per-lane cache
  (`_prevFactorByLane`, `PedestrianWalkCostFactorSystem.cs#L36-L38`) stands in for the
  missing `PrefabBase` so writes apply only the delta. Use `AlmostEqual`, never `==`,
  as the write-gate whenever the scaled field is a float.

- **Hard-coded literal baseline instead (family A).** If the vanilla value is a single
  known constant and you accept re-verifying it per patch, skip the `PrefabBase` read
  and multiply the literal directly (`0.3f * mult`). Simpler, but drifts if vanilla
  changes. See [prefab-field override](prefab-field-override.md).

## See also
- Related recipes: [prefab-field override](prefab-field-override.md) (family A - the
  hard-coded-baseline sibling of this technique),
  [periodic update-interval system](periodic-updateinterval-system.md) (the opposite
  lifecycle - a system that runs on a cadence rather than once).
- Reference: [technique index](../../technique-index.md) (family J coverage ledger).
- Case studies demonstrating it: [magic-mail](../../case-studies/magic-mail.md).

## Sources
- Canonical mods (dossier + repo):
  - `magic-mail` @6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47 - `repo/Systems/MailCapacitySystem.cs`, `repo/Settings/Settings.cs`
  - `better-bulldozer` @4408466f226db811159d92859479ae1e1c28ba06 - `repo/BetterBulldozer/Systems/RemoveExistingOwnedGrassSurfaces.cs`, `repo/BetterBulldozer/Settings/BetterBulldozerModSettings.cs`
  - `traffic-tool-essentials` @10973595ac9ed37f47ca24eb788d4421dc295fa0 - `repo/TrafficToolEssentials/Systems/DepotZone/DepotZoneDispatchSystem.cs`
  - `magic-garbage-truck` @1b6a478753e1ef4e43ac9b90d567f3d7183c7be2 - `repo/Systems/GarbageTruckCapacitySystem.cs`, `repo/Systems/GarbageThresholdSystem.cs`, `repo/Settings/Setting.cs`
  - `realistic-path-finding` @50645fa6a078181365e36a42e2e27b96699bf02a - `repo/RealisticPathFinding/Utils/PathfindCostUtils.cs`, `repo/RealisticPathFinding/Systems/PedestrianWalkCostFactorSystem.cs`
- Official/community references (link out, do not duplicate): https://cs2.paradoxwikis.com/Modding
