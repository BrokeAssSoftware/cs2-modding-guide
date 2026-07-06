---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: Debug-gated CSV telemetry + reflection ECS-type introspection"
recipe: debug-telemetry-introspection
technique_family: "AS - Debug-gated CSV telemetry + reflection ECS-type introspection"
diataxis: how-to
source_version: "~1.6.0f1 (realistic-jobsearch@7a096b2; date-pinned, static source only)"
last_reverified: "2026-07-05"
status: source-verified
canonical_mods:
  - realistic-jobsearch@7a096b2ab974bb03cc4cf0937f250bf1d7671f31
  - vno-debug@8d1cbb848850d152430c40aa374d970dc7618bae
technique_applicability: [operations]
Created: 2026-07-05
Updated: 2026-07-05
Owners:
  - codex
---

# Debug-gated CSV telemetry + reflection ECS-type introspection

> Export diagnostic data - a wide CSV of simulation metrics, or a reflection scan of
> every ECS system/component type loaded - behind a debug toggle, so the machinery
> costs nothing when the toggle is off.

## Problem
You want to *measure* what your mod is doing - commute distances, accept rates,
histogram tails - or *discover* what ECS types the game exposes, and you want that
data in a file you can open in a spreadsheet or grep, not scattered across the game
log. But diagnostics must never tax a normal play session: a per-frame CSV writer or
a full-AppDomain reflection scan running unconditionally is a performance bug. The
technique is to put every expensive path behind a settings gate and a cap, so the
system idles (or is never even scheduled) when diagnostics are off.

> Note on provenance: this handbook previously **retracted** two fabricated
> CSV-exporter claims (the time2work / RPF "case studies"). The Realistic Job Search
> block below is the first *source-verified* CSV exporter in the handbook - every
> column, the truncate-on-run behavior, and the cadence are read from real source at a
> pinned commit, not inferred. The vno-debug dump likewise emits real log lines and
> real files; none of it is an in-game behavioral claim.

## Solution
Two complementary halves, both gated:

1. **CSV telemetry (Realistic Job Search).** Only *schedule* the metrics system when a
   debug setting is set, so with the toggle off the system never runs at all. When on,
   it accumulates rolling stats and writes a fixed-schema CSV under
   `%UserData%/ModsData/<Mod>/`, truncating the file on each run and appending one row
   per cadence tick.
2. **Reflection introspection + JSON logging (VNO Debug).** A lightweight system that
   returns immediately unless enabled, then on request scans
   `AppDomain.CurrentDomain.GetAssemblies()` for ECS system and component types
   (matched by interface `FullName`), groups them by namespace, caps the output at a
   settings value, and emits each snapshot as a one-line JSON envelope stamped with a
   per-run id.

Both halves share one discipline: **gate + cap first, work second.**

## Steps & Code

### 1. Schedule the telemetry system only when the debug toggle is set

Do the gating at the coarsest level you can - here, the metrics system is not even
registered into the update loop unless `m_Setting.debug` is true:

```csharp
if (m_Setting.debug)
{
    log.Info($"Debug CSV output enabled");
    updateSystem.UpdateBefore<MetricsSystem,
                              Game.Simulation.FindJobSystem>(SystemUpdatePhase.GameSimulation);
    updateSystem.UpdateAfter<MetricsSystem,
                             GravityAcceptanceGateSystem>(SystemUpdatePhase.GameSimulation);
}
```
Source: `../../../vice-and-order-research/mods/dossiers/realistic-jobsearch/repo/RealisticJobSearch/Mod.cs#L39-L46` (@7a096b2ab974bb03cc4cf0937f250bf1d7671f31)

The toggle itself is an ordinary bool setting defaulting to `false`
(`Setting.cs#L33,L37`).

### 2. Put the file where the game expects mod data

Build the output directory from `EnvPath.kUserDataPath` + `ModsData` + your mod name,
so it lands next to every other mod's data and survives reinstalls:

```csharp
public static string outputPath = Path.Combine(EnvPath.kUserDataPath, "ModsData", nameof(RealisticJobSearch));
```
Source: `../../../vice-and-order-research/mods/dossiers/realistic-jobsearch/repo/RealisticJobSearch/Mod.cs#L19` (@7a096b2ab974bb03cc4cf0937f250bf1d7671f31)

The system composes the filename off that: `Path.Combine(Mod.outputPath, "RealisticJobSearch_metrics.csv")`
(`Systems/MetricSystem.cs#L36`).

### 3. Truncate and write the header once per run

Open the file with `append: false` so each game run starts a fresh file - old rows are
not carried across sessions. The header is a fixed 40-column schema (9 stat columns +
31 histogram bins); wrap the whole thing in try/catch so an IO failure logs and
degrades instead of throwing into the simulation:

```csharp
Directory.CreateDirectory(Path.GetDirectoryName(_csvPath)!);
using var sw = new StreamWriter(_csvPath, append: false); // overwrite / create
sw.WriteLine("timestamp,frame,count,avg_meters,avg_minutes,p50_km,p90_km,p95_km,p99_km,hist_0_1km,...,hist_29_30km,hist_30km_plus");
_wroteHeader = true;
```
Source: `../../../vice-and-order-research/mods/dossiers/realistic-jobsearch/repo/RealisticJobSearch/Systems/MetricSystem.cs#L166-L185` (@7a096b2ab974bb03cc4cf0937f250bf1d7671f31)
(the middle histogram columns `hist_1_2km` ... `hist_28_29km` are elided above; the real header at `#L173` spells out all 31 bins.)

### 4. Throttle writes with GetUpdateInterval, not a manual counter

A `GameSystemBase` lets you set the cadence declaratively. This writer runs 32 times
per in-game day rather than every frame:

```csharp
public override int GetUpdateInterval(SystemUpdatePhase phase)
{
    // One day (or month) in-game is '262144' ticks
    return TimeSystem.kTicksPerDay / 32;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/realistic-jobsearch/repo/RealisticJobSearch/Systems/MetricSystem.cs#L46-L50` (@7a096b2ab974bb03cc4cf0937f250bf1d7671f31)

### 5. Append one row per tick; skip empty ticks

Each row opens the file with `append: true`, writes an ISO timestamp + frame + rolling
aggregates + histogram-derived quantiles, and bails early when there is nothing to
report (`if (_acceptedCount == 0) return;`):

```csharp
using var sw = new StreamWriter(_csvPath, append: true);
sw.Write(DateTime.Now.ToString("s"));
sw.Write(','); sw.Write(frame.ToString(CultureInfo.InvariantCulture));
sw.Write(','); sw.Write(_acceptedCount.ToString(CultureInfo.InvariantCulture));
sw.Write(','); sw.Write((_sumMeters / _acceptedCount).ToString("F2", CultureInfo.InvariantCulture));
// ... p50/p90/p95/p99, then the 31-bin histogram ...
for (int i = 0; i < _hist.Length; i++) { sw.Write(','); sw.Write(_hist[i].ToString(CultureInfo.InvariantCulture)); }
sw.WriteLine();
```
Source: `../../../vice-and-order-research/mods/dossiers/realistic-jobsearch/repo/RealisticJobSearch/Systems/MetricSystem.cs#L188-L222` (@7a096b2ab974bb03cc4cf0937f250bf1d7671f31)

Quantiles are estimated from the histogram (cumulative-count walk, mid-bin estimate),
not a stored sample array - cheap and bounded:

```csharp
long target = (long)Math.Ceiling(q * _acceptedCount);
long cum = 0;
for (int i = 0; i < _hist.Length; i++)
{
    cum += _hist[i];
    if (cum >= target) return i == _hist.Length - 1 ? i : (i + 0.5f); // mid-bin estimate
}
```
Source: `../../../vice-and-order-research/mods/dossiers/realistic-jobsearch/repo/RealisticJobSearch/Systems/MetricSystem.cs#L224-L235` (@7a096b2ab974bb03cc4cf0937f250bf1d7671f31)

A final flush in `OnDestroy` writes the last partial interval on shutdown
(`Systems/MetricSystem.cs#L237-L242`).

### 6. (Introspection half) Return immediately unless enabled

The VNO Debug system's `OnUpdate` starts with the gate. Everything after it -
heartbeat, dump - is skipped when the mod is disabled, so the system is effectively
free when off:

```csharp
var settings = Mod.Settings;
if (settings == null || !settings.Enabled)
    return;

// Refill line budget per second (very simple token bucket)
s_LineBudget = Math.Min(settings.MaxLinesPerSecond, s_LineBudget + settings.MaxLinesPerSecond);
```
Source: `../../../vice-and-order-research/mods/dossiers/vno-debug/repo/VnoDebugSystem.cs#L37-L42` (@8d1cbb848850d152430c40aa374d970dc7618bae)

`s_LineBudget` is a token-bucket counter capped at `MaxLinesPerSecond` and refilled per
update; in this snapshot the refill/cap are wired but no line-emit call decrements it,
so treat it as the rate-limit scaffold rather than a proven throttle
(**Needs Verification (in-game)** that emission is actually gated by the budget).

### 7. Scan the AppDomain for types by predicate, filtered and capped

The core introspection primitive iterates every loaded assembly, guards `GetTypes()`
with try/catch (some assemblies throw `ReflectionTypeLoadException`), applies the
settings namespace/name filters, then the caller-supplied predicate:

```csharp
foreach (var asm in AppDomain.CurrentDomain.GetAssemblies())
{
    Type[] types;
    try { types = asm.GetTypes(); }
    catch { continue; }

    foreach (var t in types)
    {
        try
        {
            // NamespaceInclude / NamespaceExclude / TypeNamePattern filters elided
            if (predicate(t)) list.Add(t);
        }
        catch { /* ignore */ }
    }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/vno-debug/repo/VnoDebugSystem.cs#L197-L226` (@8d1cbb848850d152430c40aa374d970dc7618bae)

### 8. Match ECS types by interface FullName, group by namespace, cap by setting

Systems match on `ComponentSystemBase`/`SystemBase`; components match by the three ECS
data-interface names. Matching on the interface `FullName` string (not `typeof`) means
the scan does not need a hard reference to `Unity.Entities` types at those sites:

```csharp
bool IsComponent(Type t)
{
    return t.GetInterfaces().Any(it => it.FullName == "Unity.Entities.IComponentData")
        || t.GetInterfaces().Any(it => it.FullName == "Unity.Entities.IBufferElementData")
        || t.GetInterfaces().Any(it => it.FullName == "Unity.Entities.ISharedComponentData");
}

var types = EnumerateTypesMatch(IsComponent);
var limited = types.Take(settings.MaxItemsPerDump).ToArray();
```
Source: `../../../vice-and-order-research/mods/dossiers/vno-debug/repo/VnoDebugSystem.cs#L166-L189` (@8d1cbb848850d152430c40aa374d970dc7618bae)

The systems variant groups the capped set by namespace and reports `count` vs `shown`
so a truncated dump is visible as such:
`byNs = limited.GroupBy(t => t.Namespace ?? "").ToDictionary(...)`
(`VnoDebugSystem.cs#L142-L158`).

### 9. Emit each snapshot as a one-line JSON envelope with a run id

Rather than a file, VNO Debug writes a single structured log line per snapshot -
greppable and stamped with `ts` / `run` / `cat` / `name` / `payload` / `ctx`:

```csharp
var sb = new StringBuilder(256);
sb.Append(Prefix);                       // "[VNO-DEBUG] "
sb.Append('{');
AppendProp(sb, "ts", DateTime.UtcNow.ToString("o", CultureInfo.InvariantCulture)); sb.Append(',');
AppendProp(sb, "run", Mod.RunId());      sb.Append(',');
AppendProp(sb, "src", "mod");            sb.Append(',');
AppendProp(sb, "cat", cat);              sb.Append(',');
AppendProp(sb, "name", name);            sb.Append(',');
sb.Append("\"payload\":"); sb.Append(ObjectToJson(payload)); sb.Append(',');
sb.Append("\"ctx\":");     sb.Append(ObjectToJson(ctx));
sb.Append('}');
Mod.Log.Info(sb.ToString());
```
Source: `../../../vice-and-order-research/mods/dossiers/vno-debug/repo/Utilities/LogJson.cs#L12-L33` (@8d1cbb848850d152430c40aa374d970dc7618bae)

The run id groups every line from one session; it is computed once and cached:

```csharp
var ts = DateTime.UtcNow.ToString("yyyyMMdd-HHmmss");
var rand = Guid.NewGuid().ToString("N").Substring(0, 3).ToUpperInvariant();
s_RunId = $"{ts}-{rand}";
```
Source: `../../../vice-and-order-research/mods/dossiers/vno-debug/repo/Mod.cs#L49-L57` (@8d1cbb848850d152430c40aa374d970dc7618bae)

## Pitfalls & gotchas

- **The CSV gate is decided at mod-load, not live.** Realistic Job Search reads
  `m_Setting.debug` inside `OnLoad` to decide whether to *schedule* `MetricsSystem`
  (`Mod.cs#L39-L46`). Toggling the setting mid-session does not start or stop the
  writer - it takes effect only on the next load. That is the right trade for a
  zero-cost-when-off design, but it surprises anyone expecting a live checkbox.
- **The CSV truncates every run.** `append: false` on the header write
  (`MetricSystem.cs#L172`) recreates the file each session, so you get the *current*
  run only. If you need history, copy the file off between runs or change the open mode
  - but then you own de-duping the header.
- **Rows are skipped when there is no data.** `if (_acceptedCount == 0) return;`
  (`MetricSystem.cs#L191`) means gaps in the timestamp column are expected during quiet
  periods; do not treat a missing row as a dropped tick.
- **Reflection scans are unbounded without the cap.** `AppDomain...GetAssemblies()`
  touches every loaded assembly and `asm.GetTypes()` can be thousands of types. The
  `.Take(MaxItemsPerDump)` cap (`VnoDebugSystem.cs#L150,L178`) is what keeps a dump
  cheap; the `count` vs `shown` fields tell you when you are seeing a truncated view.
  Never run a full uncapped scan on the simulation thread.
- **`GetTypes()` throws on some assemblies.** The per-assembly `try { } catch { continue; }`
  (`VnoDebugSystem.cs#L203-L204`) is load-bearing: a `ReflectionTypeLoadException` from
  one assembly must not abort the whole scan. Keep it.
- **Interface matching is by `FullName` string.** `it.FullName == "Unity.Entities.IComponentData"`
  is resilient to not referencing the type directly, but it also means a namespace
  rename in a future Unity/game update silently matches nothing rather than failing
  loudly. Re-verify the three interface names per game version.
- **JSON here is a hand-rolled stringifier.** `LogJson.ObjectToJson` handles anonymous
  types, primitives, arrays and `IEnumerable` only (`LogJson.cs#L51-L104`); it is not a
  general serializer. Feed it flat anonymous payloads, not arbitrary object graphs.

## Variations

- **File sink instead of log line.** The CSV half writes a real file under
  `ModsData/<Mod>/`; the introspection half writes log lines. Either sink works for
  either payload - route your reflection dump to a file with the Step 2/3 pattern if
  you want it spreadsheet-friendly, or emit metrics as JSON log lines if you only need
  to grep.
- **Introspection as a corpus generator.** The ECS-type dump (Steps 7-8) is exactly the
  scan you run once to *generate* a components catalog: capture the emitted
  `ecs.components` / `ecs.systems` snapshots and fold the type list into
  [reference/ecs-components-catalog.md](../../reference/ecs-components-catalog.md).
  Set `MaxItemsPerDump` high and a `NamespaceInclude` of `Game.` for a focused pass.
- **On-request vs periodic.** VNO Debug supports both: a `RequestDump()` static flag
  consumed once per update, and a `HeartbeatSeconds` periodic emit
  (`VnoDebugSystem.cs#L44-L62`). Use request-driven dumps for expensive scans and the
  heartbeat only for a cheap liveness ping.
- **Coarse gate vs early-return gate.** Realistic Job Search gates by *not scheduling*
  the system (cheapest - the system never ticks); VNO Debug gates by early-return
  inside `OnUpdate`. Prefer the not-scheduled form when the toggle rarely changes;
  prefer early-return when you want a live checkbox.

## See also
- Related recipes: [json-sidecar-persistence](json-sidecar-persistence.md) (structured
  JSON payloads to a companion file).
- Operations: [logging-and-debugging](../operations/logging-and-debugging.md).
- Reference: [ecs-components-catalog](../../reference/ecs-components-catalog.md) (the
  corpus the introspection dump feeds).
- Case studies demonstrating it: [realistic-jobsearch](../../case-studies/realistic-jobsearch.md).

## Sources
- Canonical mods (dossier + repo):
  - `realistic-jobsearch` @7a096b2ab974bb03cc4cf0937f250bf1d7671f31 - `repo/RealisticJobSearch/Mod.cs`, `repo/RealisticJobSearch/Systems/MetricSystem.cs`
  - `vno-debug` @8d1cbb848850d152430c40aa374d970dc7618bae - `repo/VnoDebugSystem.cs`, `repo/Utilities/LogJson.cs`, `repo/Mod.cs`
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding
