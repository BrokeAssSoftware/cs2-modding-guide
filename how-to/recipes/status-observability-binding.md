---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: Read-only status / observability binding (zero steady-state cost)"
recipe: status-observability-binding
technique_family: "AX - Read-only status / observability binding (zero steady-state cost)"
diataxis: how-to
source_version: "~1.6.0f1 (magic-garbage-truck@1b6a478; date-pinned, static source only)"
last_reverified: "2026-07-05"
status: source-verified
canonical_mods:
  - magic-mail@6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47
  - magic-garbage-truck@1b6a478753e1ef4e43ac9b90d567f3d7183c7be2
technique_applicability: [ui, operations]
Created: 2026-07-05
Updated: 2026-07-05
Owners:
  - codex
---

# Read-only status / observability binding (zero steady-state cost)

> Surface live mod/sim stats to the Options UI without paying a per-frame cost:
> compute the numbers only when the settings menu is actually open (or when a sim
> tick that already runs publishes them), and never in a hot in-city loop.

## Problem
You want an Options-menu "Status" panel that shows what your mod is doing right now -
facility counts, capacity totals, city-wide accumulation, truck states. The naive
approach runs an `OnUpdate` every frame to keep those numbers fresh, which burns CPU
in-city even though nobody is looking at the panel 99.9% of the time. You want the
readout to be correct when opened but to cost **nothing** in steady state.

## Solution
Decouple *producing* the stat from *displaying* it, and make production lazy. Two
cadences cover almost every case:

- **Static-field bridge** - a system that *already ticks* (on its own throttled
  interval) writes its latest numbers into `internal static` fields; read-only string
  getters on the settings object format those fields on demand. The UI never triggers
  work; it just reads the last published values. (Magic Mail.)
- **Refresh-on-read** - the system does **no** automatic work (empty `OnUpdate`,
  `Enabled = false`); each settings string getter calls a throttled
  `RefreshIfNeeded()` that recomputes an ECS snapshot at most once per frame and at
  most once per N seconds of wall-clock. Closing the menu stops all work. (Magic
  Garbage Truck.)

Both guard against a not-yet-loaded city so an early menu open never NREs.

## Steps & Code

### 1. (Static-field bridge) Declare the published statics on the sim system

The system owns a block of `internal static` fields - one per stat. `static` so the
settings getters can read them without holding a system reference:

```csharp
// ---- STATUS FIELDS (read by Setting.Status* properties) ----
internal static int s_LastFacilityCount;
internal static int s_LastPostOfficeCount;
internal static int s_LastSortingFacilityCount;
internal static int s_LastPostVanCapacityTotal;
internal static int s_LastPostTruckCapacityTotal;
internal static int s_LastPostOfficeGets;
internal static int s_LastSortingGets;
internal static int s_LastOverflowClamps;
```
Source: `../../../vice-and-order-research/mods/dossiers/magic-mail/repo/Systems/MagicMailSystem.cs#L44-L53` (@6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47)

### 2. (Static-field bridge) Publish at the end of the system's existing tick

Magic Mail's system already runs on a throttled interval (32 updates/day, not
per-frame). At the end of that same tick it copies its accumulators into the statics -
so the publish is free-riding on work that had to happen anyway:

```csharp
// Publish status for the Status tab.
s_LastFacilityCount = facilityCount;
s_LastPostOfficeCount = postOfficeCount;
s_LastSortingFacilityCount = sortingFacilityCount;
s_LastPostVanCapacityTotal = totalPostVanCapacity;
s_LastPostTruckCapacityTotal = totalPostTruckCapacity;
s_LastPostOfficeGets = postOfficeGets;
s_LastSortingGets = sortingGets;
s_LastOverflowClamps = overflowClamps;
```
Source: `../../../vice-and-order-research/mods/dossiers/magic-mail/repo/Systems/MagicMailSystem.cs#L201-L209` (@6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47)

### 3. (Static-field bridge) Read-only settings getters format the statics

A `[SettingsUISection(...)]` string property with a getter-only body reads the
statics and formats them. Crucially it **falls back** to a friendly "not ready" string
when the count is still zero, so an early open shows guidance instead of "0 0 0":

```csharp
[SettingsUISection(kStatusTab, StatusSummaryGroup)]
public string StatusFacilitySummary
{
    get
    {
        if (MagicMailSystem.s_LastFacilityCount == 0)
            return L(StatusNoFacilitiesKey,
                "No postal facilities processed yet. Open a city and let the simulation run.");

        return string.Format(
            L(StatusSummaryKey, "{0} post offices | {1} post-vans | {2} sorting buildings | {3} post trucks"),
            MagicMailSystem.s_LastPostOfficeCount, MagicMailSystem.s_LastPostVanCapacityTotal,
            MagicMailSystem.s_LastSortingFacilityCount, MagicMailSystem.s_LastPostTruckCapacityTotal);
    }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/magic-mail/repo/Settings/Settings.cs#L390-L411` (@6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47)

### 4. (Static-field bridge) Fall back gracefully when a vanilla stat source is absent

Magic Mail also mirrors two city-wide stats off the vanilla
`MailAccumulationSystem`. It resolves that system inside a `try/catch` -
`GetExistingSystemManaged<T>()` throws `InvalidOperationException` if the system does
not exist - and logs a warning once instead of crashing:

```csharp
private void TryResolveMailAccumulationSystem()
{
    try
    {
        m_MailAccumulationSystem = World.GetExistingSystemManaged<MailAccumulationSystem>();
    }
    catch (System.InvalidOperationException)
    {
        if (m_MailAccumulationSystem == null)
            Mod.s_Log.Warn("MailAccumulationSystem not found; city mail stats unavailable.");
    }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/magic-mail/repo/Systems/MagicMailSystem.cs#L478-L491` (@6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47)

### 5. (Refresh-on-read) Make the status system do NO automatic work

Magic Garbage Truck's snapshot system disables itself in `OnCreate` and leaves
`OnUpdate` empty on purpose - it exists only to build a snapshot when asked, never as
a background tick:

```csharp
    Enabled = false;
}

// Snapshot system only in Options UI, no auto sim work needed on purpose, does not affect city performance.
protected override void OnUpdate()
{
}
```
Source: `../../../vice-and-order-research/mods/dossiers/magic-garbage-truck/repo/Systems/GarbageStatusSystem.cs#L293-L299` (@1b6a478753e1ef4e43ac9b90d567f3d7183c7be2)

### 6. (Refresh-on-read) Settings getters trigger a throttled refresh, then read

Each status property calls `RefreshIfNeeded()` before returning the cached UI string.
Opening the panel (Options polls these getters) is what drives recomputation; closing
it stops all work:

```csharp
[SettingsUISection(ActionsTab, StatusGrp)]
public string StatusGarbageServiceRating
{
    get
    {
        GarbageStatus.RefreshIfNeeded();
        return GarbageStatus.GetUiGarbageServiceRating();
    }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/magic-garbage-truck/repo/Settings/Setting.Status.cs#L23-L32` (@1b6a478753e1ef4e43ac9b90d567f3d7183c7be2)

### 7. (Refresh-on-read) Self-throttle: per-frame dedupe, force on entry, else 10s

`RefreshIfNeeded` is a small state machine. Options may poll several getters in one
frame - `Time.frameCount` dedupe collapses them to one recompute. First frame in a
city force-refreshes; after that it only recomputes when the last snapshot is older
than `AutoRefreshSeconds` (10s) of wall-clock:

```csharp
public static void RefreshIfNeeded()
{
    int frame = Time.frameCount;
    if (frame == s_LastUiFrame) return;   // already refreshed this frame
    s_LastUiFrame = frame;

    if (!IsGameMode()) { s_WasInGame = false; SetNoCityUi(); return; }

    if (!s_WasInGame) { s_WasInGame = true; RefreshNow(writeToLog: false); return; } // force on city entry

    long nowUtc = DateTime.UtcNow.Ticks;
    if (s_LastRefreshUtcTicks > 0)
    {
        long minTicks = AutoRefreshSeconds * TimeSpan.TicksPerSecond;   // AutoRefreshSeconds = 10
        if (nowUtc - s_LastRefreshUtcTicks < minTicks) return;          // too soon; keep cached
    }
    RefreshNow(writeToLog: false);
}
```
Source: `../../../vice-and-order-research/mods/dossiers/magic-garbage-truck/repo/Systems/GarbageStatus.cs#L57-L94` (@1b6a478753e1ef4e43ac9b90d567f3d7183c7be2) (const `AutoRefreshSeconds = 10` at `#L28`)

### 8. (Refresh-on-read) Build the ECS snapshot only when a refresh actually fires

`RefreshNow` gets-or-creates the snapshot system and calls `BuildSnapshot()`, which
aggregates the producer/request/truck queries into an immutable `Snapshot` struct.
Guard the parameter singleton with `SystemAPI.TryGetSingleton` so an early load (no
city) returns an empty snapshot instead of throwing:

```csharp
GarbageStatusSystem sys = world.GetOrCreateSystemManaged<GarbageStatusSystem>();
GarbageStatusSystem.Snapshot snap = sys.BuildSnapshot();
// inside BuildSnapshot:
bool haveParams = SystemAPI.TryGetSingleton(out GarbageParameterData gp);
```
Source: `../../../vice-and-order-research/mods/dossiers/magic-garbage-truck/repo/Systems/GarbageStatus.cs#L122-L123` and `.../Systems/GarbageStatusSystem.cs#L316` (@1b6a478753e1ef4e43ac9b90d567f3d7183c7be2)

The builder also computes a true median by sorting the producer garbage values
(odd -> middle element, even -> rounded mean of the two middles) rather than an average
that outliers would skew:

```csharp
garbageValues.Sort();
int middle = garbageValues.Count / 2;
if ((garbageValues.Count & 1) == 1)
    producerMedianGarbage = garbageValues[middle];
else
    producerMedianGarbage = (int)Math.Round(
        (garbageValues[middle - 1] + garbageValues[middle]) / 2.0, MidpointRounding.AwayFromZero);
```
Source: `../../../vice-and-order-research/mods/dossiers/magic-garbage-truck/repo/Systems/GarbageStatusSystem.cs#L477-L489` (@1b6a478753e1ef4e43ac9b90d567f3d7183c7be2)

## Pitfalls & gotchas

- **The whole point is zero steady-state cost.** Do NOT recompute stats in a
  per-frame `OnUpdate` "so the panel is always fresh." Either publish from a tick that
  already runs (Magic Mail, step 2) or recompute lazily on read (Magic Garbage Truck,
  step 5). Magic Garbage Truck's `OnUpdate` is deliberately empty with `Enabled =
  false` (`GarbageStatusSystem.cs#L293-L299`) - the readout costs nothing until the
  menu is opened.

- **Pick the cadence that matches your data.** Static-field bridge fits stats a
  throttled sim tick computes anyway (near-free publish, but the readout is only as
  fresh as the last tick). Refresh-on-read fits expensive aggregates you would rather
  not compute unless someone is looking (fresh when opened, but the getter now does
  real work). A third cadence - a polled fixed-interval (~500ms) binding - is covered
  in the poll/interval family (see AV); use it when a live panel must update while
  visible without a menu-open trigger.

- **Guard singleton / system absence or early loads NRE.** During load or in the main
  menu the city singleton and parameter singletons may not exist. Magic Garbage Truck
  uses `SystemAPI.TryGetSingleton(out GarbageParameterData gp)` and treats
  `city == Entity.Null` as "not in game" (`GarbageStatusSystem.cs#L304, #L316`); Magic
  Mail wraps `GetExistingSystemManaged<MailAccumulationSystem>()` in `try/catch`
  (`MagicMailSystem.cs#L478-L491`). Skipping these guards crashes the Options menu on
  early open.

- **Always ship a "not ready yet" fallback string.** A zero-count getter should return
  guidance, not raw zeros. Magic Mail returns "No postal facilities processed yet..."
  when `s_LastFacilityCount == 0` (`Settings.cs#L395-L399`); Magic Garbage Truck seeds
  every UI field to `"-"` (`GarbageStatus.cs#L34-L40`) and `SetNoCityUi()` restores the
  placeholder text out of game (`GarbageStatus.cs#L648-L659`).

- **`static` publish fields are process-global.** The `s_Last*` pattern works because
  a mod is effectively a singleton per process, but the fields survive between
  city loads - reset them (or gate reads on an in-game flag) so a stale count from a
  previous save is not shown before the first tick of a new one. Magic Garbage Truck's
  `ResetUi()` does this for its cached strings (`GarbageStatus.cs#L42-L55`).

- Whether the ~10s throttle "feels" live enough, and the exact frequency Options polls
  these getters, are runtime behaviours not provable from source: `Needs Verification
  (in-game)`.

## Variations

- **Force-refresh + log dump on a button.** `RefreshNow(writeToLog: true)` recomputes
  and writes a full report to the mod log; wire it to an Options button for on-demand
  diagnostics without changing the passive readout
  (`GarbageStatus.cs#L96` onward).

- **Mirror a vanilla system's stat.** Instead of computing your own numbers, resolve a
  vanilla stat system and read its public properties (Magic Mail reads
  `MailAccumulationSystem.LastAccumulatedMail` / `.LastProcessedMail`,
  `MagicMailSystem.cs#L219-L220`) - with the try/catch fallback of step 4 so a game
  patch that removes the system degrades gracefully.

- **Polled fixed-interval binding (~500ms).** When the panel is a persistent HUD rather
  than a settings tab, a timer-driven binding refreshed every ~500ms is the right
  cadence (family AV). It trades a small always-on cost for liveness while visible.

## See also
- Related recipes: [settings patterns](settings-patterns.md) (the
  `[SettingsUISection]` string-getter surface these bind to),
  [UISystemBase React binding](uisystembase-react-binding.md) (pushing status to a
  custom HUD instead of the Options menu).
- Operations: [logging and debugging](../operations/logging-and-debugging.md) (the
  force-refresh log-dump variant).
- Case studies demonstrating it: [magic-garbage-truck](../../case-studies/magic-garbage-truck.md).

## Sources
- Canonical mods (dossier + repo):
  - `magic-mail` @6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47 - `repo/Systems/MagicMailSystem.cs`, `repo/Settings/Settings.cs`
  - `magic-garbage-truck` @1b6a478753e1ef4e43ac9b90d567f3d7183c7be2 - `repo/Systems/GarbageStatus.cs`, `repo/Systems/GarbageStatusSystem.cs`, `repo/Settings/Setting.Status.cs`
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding
