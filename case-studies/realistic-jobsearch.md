---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Case study: Realistic Job Search"
Summary: How Realistic Job Search reshapes citizen job assignment without replacing FindJobSystem - a Harmony CompleteSetup prefix that re-ranks the pathfind candidate buffer, a Burst acceptance gate that vetoes vanilla by stripping PathInformation, a retry-throttle cooldown, and the handbook's first source-verified CSV telemetry example.
case_study: realistic-jobsearch
mod: "Realistic Job Search"
dossier: ../../vice-and-order-research/mods/dossiers/realistic-jobsearch/
repo_commit: 7a096b2ab974bb03cc4cf0937f250bf1d7671f31
source_version: "1.6.0f1 (realistic-jobsearch@7a096b2; date-pinned, static source only)"
last_reverified: "2026-07-05"
diataxis: explanation
techniques: [AH, AI, AS]
technique_applicability: [economy, simulation, operations]
status: source-verified
Created: 2026-07-05
Updated: 2026-07-05
Owners:
  - codex
---

# Realistic Job Search - case study

> A job-assignment overhaul that never replaces `FindJobSystem`. Instead it
> intercepts the shared pathfind setup buffer with a Harmony prefix to re-rank
> workplace candidates, then gates the finished paths with a Burst job that
> vetoes vanilla by stripping `PathInformation` - a compact tour of how to bend
> an economy loop through its seams rather than cloning it.

All code claims below are cited to `repo/<path>#Lxx` at pinned commit
`7a096b2ab974bb03cc4cf0937f250bf1d7671f31` (ModId 123500, ModVersion 0.2.1),
surfaced through the dossier at
`../../vice-and-order-research/mods/dossiers/realistic-jobsearch/`. This is the
handbook's **first source-verified CSV telemetry example** - two earlier
telemetry write-ups were fabricated and retracted; the schema here is read
directly from source.

## What it does / why it's instructive

Realistic Job Search (RJS, author ruzbeh0) makes citizens prefer nearby
workplaces while still letting large employers pull workers from across the map -
a gravity model layered onto vanilla job assignment. Vanilla CS2 ignores commute
distance when assigning jobs; RJS injects distance-and-mass scoring into the
selection without owning the job-search system.

It is instructive because it is the **opposite maintenance bet** from a
system-replacement mod like Realistic Path Finding. Rather than disabling
`FindJobSystem` and shipping a decompiled clone, RJS leaves every vanilla system
running and inserts itself at three seams:

1. It **re-ranks the pathfind candidate buffer** with a Harmony prefix on
   `PathfindSetupSystem.CompleteSetup`, reaching into private fields to rewrite
   the target list in place before the pathfinder consumes it.
2. It **gates the results** with a Burst `IJobChunk` that removes
   `PathInformation` from seekers whose commute scores too low - a
   component-strip *veto* that makes vanilla `FindJobSystem` behave as if the
   path never finished.
3. It **throttles retries** with a per-seeker cooldown marker.

Because it patches the seam instead of the system, the whole mod is ~5 source
files and carries almost no decompiled surface - at the cost of depending on the
exact private field names and buffer types of `PathfindSetupSystem`.

## Architecture at a glance

### Load-time scheduling

`Mod.OnLoad` registers settings, then schedules three systems - all
`UpdateBefore<..., FindJobSystem>` in `SystemUpdatePhase.GameSimulation`:
`GravityPreFilterSystem`, `GravityAcceptanceGateSystem`, and
`RetryThrottleSystem` (repo/RealisticJobSearch/Mod.cs#L33-L38). A fourth system,
`MetricsSystem`, is scheduled **only when `debug` is enabled** - both
`UpdateBefore<..., FindJobSystem>` and `UpdateAfter<..., GravityAcceptanceGateSystem>`
(repo/RealisticJobSearch/Mod.cs#L39-L46). It then applies all Harmony patches in
the assembly with `harmony.PatchAll(...)` (repo/RealisticJobSearch/Mod.cs#L51-L53).
Harmony (Lib.Harmony 2.2.2) is a real, load-bearing dependency
(repo/RealisticJobSearch/RealisticJobSearch.csproj#L88).

Data flows in three stages relative to vanilla `FindJobSystem`:

- **Setup time (Harmony prefix):** when the pathfinder assembles job-seeker
  targets, the prefix re-ranks the candidate buffer in place.
- **Result time (`GravityAcceptanceGateSystem`, pre-`FindJobSystem`):** finished
  paths are accepted or vetoed by stripping `PathInformation`.
- **Retry time (`RetryThrottleSystem`, pre-`FindJobSystem`):** vetoed seekers get
  a cooldown, then are released to try again.

### The candidate-rewrite prefix

`Patch_CompleteSetup_FilterJobSeekerTargets` prefixes
`PathfindSetupSystem.CompleteSetup`
(repo/RealisticJobSearch/Patches/Patch_CompleteSetup_FilterJobSeekerTargets.cs#L15-L16).
It reaches into two private members via Harmony `AccessTools.FieldRefAccess`: the
`NativeList<PathfindSetupSystem.SetupListItem> m_SetupList` and the
`JobHandle m_SetupDependencies`
(repo/RealisticJobSearch/Patches/Patch_CompleteSetup_FilterJobSeekerTargets.cs#L27-L31).
The prefix first force-completes the setup dependencies (`setupDeps.Complete()`)
so it can safely read the buffers the vanilla code is about to consume
(repo/RealisticJobSearch/Patches/Patch_CompleteSetup_FilterJobSeekerTargets.cs#L52-L56),
pairs each `JobSeekerTo` target buffer with its origin `CurrentLocation` by
`m_ActionIndex` (repo/RealisticJobSearch/Patches/Patch_CompleteSetup_FilterJobSeekerTargets.cs#L67-L87),
scores every candidate, and rewrites the `UnsafeList<PathTarget>` buffer in place
through four `[MethodImpl(AggressiveInlining)]` no-alloc helpers - `Clear`,
`Truncate`, `SetAt`, `Add` - that mutate `list.Length` and `list.ElementAt(...)`
directly (repo/RealisticJobSearch/Patches/Patch_CompleteSetup_FilterJobSeekerTargets.cs#L179-L202).

### The acceptance gate

`GravityAcceptanceGateSystem` is a `GameSystemBase` that schedules a Burst
`IJobChunk` (`GateJob`) over job seekers that have a finished `PathInformation`
(repo/RealisticJobSearch/Systems/GravityAcceptanceGateSystem.cs#L47-L77). It reads
its knobs from a `GravityAcceptParams` singleton it creates in `OnCreate` from
`Mod.m_Setting` (repo/RealisticJobSearch/Systems/GravityAcceptanceGateSystem.cs#L33-L44).
For each finished path it computes a gravity acceptance probability and, if the
seeker scores at or below the floor, **removes `PathInformation` via the
EndFrameBarrier command buffer** so vanilla `FindJobSystem.StartWorkingJob` sees
no completed path and skips the hire
(repo/RealisticJobSearch/Systems/GravityAcceptanceGateSystem.cs#L118-L138).

### The retry throttle

`RetryThrottleSystem` runs a Burst `IJobEntity` (`ThrottleJob`) over seekers
carrying the `RefusedLongCommute` marker the gate added. It clears the marker once
the seeker is both under its daily-retry cap and past the cooldown window
(`framesPerHour = 1024`, cooldown = `RetryCooldownHours * 1024` frames), releasing
the seeker to be re-evaluated
(repo/RealisticJobSearch/Systems/RetryThrottleSystem.cs#L46-L83). Its
`SpatialSamplerParams` singleton defaults `MaxDailyRetries = 1` and
`RetryCooldownHours = 2f`
(repo/RealisticJobSearch/Systems/RetryThrottleSystem.cs#L31-L42).

### Debug CSV telemetry

`MetricsSystem` (in `MetricSystem.cs`) is a plain `GameSystemBase` that samples
accepted job paths on a periodic cadence - `GetUpdateInterval` returns
`TimeSystem.kTicksPerDay / 32`
(repo/RealisticJobSearch/Systems/MetricSystem.cs#L46-L50). It accumulates
origin->destination distance, duration, and a 31-bin distance histogram, then
appends a row to `RealisticJobSearch_metrics.csv` under
`%UserData%/ModsData/RealisticJobSearch/`
(repo/RealisticJobSearch/Mod.cs#L19; repo/RealisticJobSearch/Systems/MetricSystem.cs#L36).
The header is a **40-column schema** - 9 fixed columns
(`timestamp,frame,count,avg_meters,avg_minutes,p50_km,p90_km,p95_km,p99_km`) plus
31 histogram bins (`hist_0_1km` .. `hist_30km_plus`)
(repo/RealisticJobSearch/Systems/MetricSystem.cs#L173). The file is **truncated on
every run** - `TryWriteHeader` opens the writer with `append: false`
(repo/RealisticJobSearch/Systems/MetricSystem.cs#L166-L179).

## Techniques demonstrated

- [Pathfind candidate-buffer rewrite (CompleteSetup prefix)](../how-to/recipes/pathfind-candidate-rewrite.md)
  (family AH) - a Harmony prefix on `PathfindSetupSystem.CompleteSetup` uses
  `AccessTools.FieldRefAccess` into private `m_SetupList` / `m_SetupDependencies`,
  force-completes the setup jobs, and rewrites each `UnsafeList<PathTarget>` in
  place with inlined no-alloc helpers
  (repo/RealisticJobSearch/Patches/Patch_CompleteSetup_FilterJobSeekerTargets.cs#L15-L31,
  repo/RealisticJobSearch/Patches/Patch_CompleteSetup_FilterJobSeekerTargets.cs#L179-L202).
- [Heuristic scoring & stochastic selection + component-strip veto](../how-to/recipes/heuristic-scoring-selection.md)
  (family AI) - a gravity utility `U = alpha*log(1+mass) - beta*minutes` feeds a
  softmax draw (`Tau = 0.45`, pool up to `TopK*2`, keep `TopK = 12`) so the picked
  workplace is proportional to `exp(U/Tau)` rather than always the top score
  (repo/RealisticJobSearch/Patches/Patch_CompleteSetup_FilterJobSeekerTargets.cs#L113-L170).
  The post-path `GateJob` re-scores with a multiplicative gravity model
  (`mass = pow(jobs, alpha)`, `accept = mass * exp(-beta*minutes)`, clamped) and
  vetoes low scorers by `RemoveComponent<PathInformation>`
  (repo/RealisticJobSearch/Systems/GravityAcceptanceGateSystem.cs#L108-L138). A
  retry-throttle marker (`RefusedLongCommute`) plus cooldown prevents thrashing
  (repo/RealisticJobSearch/Systems/RetryThrottleSystem.cs#L70-L83).
- [Debug-gated CSV telemetry](../how-to/recipes/debug-telemetry-introspection.md)
  (family AS) - `MetricsSystem` is registered only when `Setting.debug` is true,
  writes a fixed 40-column CSV to `%UserData%/ModsData/<Mod>/`, and truncates the
  file at startup
  (repo/RealisticJobSearch/Mod.cs#L39-L46;
  repo/RealisticJobSearch/Systems/MetricSystem.cs#L166-L179,
  repo/RealisticJobSearch/Systems/MetricSystem.cs#L173).

## Key decisions & tradeoffs

- **Re-rank candidates; don't replace the system.** RJS never disables
  `FindJobSystem`. It edits the candidate buffer the pathfinder is about to
  consume (`CompleteSetup` prefix) and vetoes finished results by stripping
  `PathInformation`
  (repo/RealisticJobSearch/Patches/Patch_CompleteSetup_FilterJobSeekerTargets.cs#L49-L56;
  repo/RealisticJobSearch/Systems/GravityAcceptanceGateSystem.cs#L118-L121). The
  win is a tiny maintenance surface with no decompiled clone; the cost is a hard
  dependency on `PathfindSetupSystem`'s private layout (`m_SetupList`,
  `m_SetupDependencies`, `SetupListItem`, the `UnsafeList<PathTarget>` buffer).
- **Veto by removing a component.** Instead of forcing a hire, the gate *subtracts*
  a completed path so the vanilla consumer simply finds nothing to do - a clean way
  to say "no" to another system without patching it
  (repo/RealisticJobSearch/Systems/GravityAcceptanceGateSystem.cs#L118-L138).
- **Two different gravity forms on purpose.** The setup-time patch uses an additive
  log-utility with a softmax draw (variety among good options); the result-time
  gate uses a multiplicative accept probability (a hard floor cutoff). They are
  deliberately different scoring shapes at different stages
  (repo/RealisticJobSearch/Patches/Patch_CompleteSetup_FilterJobSeekerTargets.cs#L113-L115;
  repo/RealisticJobSearch/Systems/GravityAcceptanceGateSystem.cs#L113-L116).
- **Retry with cooldown, not infinite re-roll.** A refused seeker is marked, then
  released only after `MaxDailyRetries` and a frame cooldown, bounding churn
  (repo/RealisticJobSearch/Systems/RetryThrottleSystem.cs#L54-L83).
- **Telemetry is debug-gated and disposable.** The CSV system is scheduled only
  under `debug`, and the file is truncated each run - it is an analysis aid, not a
  persisted log (repo/RealisticJobSearch/Mod.cs#L39-L46;
  repo/RealisticJobSearch/Systems/MetricSystem.cs#L170-L173).

## Pitfalls / upstream-watch

- **Settings frozen at type-init (anti-pattern).** The patch caches slider values
  into `static readonly` fields (`AlphaJobs`, `BetaMinute`, `WTotal`, `WFree`)
  evaluated once when the type initializes
  (repo/RealisticJobSearch/Patches/Patch_CompleteSetup_FilterJobSeekerTargets.cs#L19-L22),
  and the gate seeds its `GravityAcceptParams` singleton once in `OnCreate`
  (repo/RealisticJobSearch/Systems/GravityAcceptanceGateSystem.cs#L36-L43). There
  is no `onSettingsApplied` handler in `Setting.cs`, so slider edits do **not**
  take effect until the game reloads - the opposite of the event-driven
  re-application idiom other mods use.
- **`GravityPreFilterSystem` is dormant.** It is scheduled
  (repo/RealisticJobSearch/Mod.cs#L33-L34) and fully implemented, but it
  `RequireForUpdate` a `ProposedJobPath` query
  (repo/RealisticJobSearch/Systems/GravityPreFilterSystem.cs#L34-L44) and nothing
  in the mod ever creates a `ProposedJobPath` entity, so its body never runs. The
  `RjsBypassPrefilter` tag it would add
  (repo/RealisticJobSearch/Systems/GravityPreFilterSystem.cs#L147-L148) is also
  never checked by the active prefix. Treat it as inert scaffolding, not a live
  path.
- **Private-field fragility.** The prefix binds by string field names
  (`"m_SetupList"`, `"m_SetupDependencies"`) and assumes the buffer is
  `UnsafeList<PathTarget>`
  (repo/RealisticJobSearch/Patches/Patch_CompleteSetup_FilterJobSeekerTargets.cs#L27-L31,
  repo/RealisticJobSearch/Patches/Patch_CompleteSetup_FilterJobSeekerTargets.cs#L181).
  If Colossal renames or retypes any of these, `FieldRefAccess` throws at type
  init and the whole patch fails.
- **Managed scratch allocations per call.** Despite the "no-alloc" UnsafeList
  helpers, the prefix allocates a `Dictionary`, a `List`, and a `double[]` on the
  managed heap on every invocation
  (repo/RealisticJobSearch/Patches/Patch_CompleteSetup_FilterJobSeekerTargets.cs#L69,
  repo/RealisticJobSearch/Patches/Patch_CompleteSetup_FilterJobSeekerTargets.cs#L94,
  repo/RealisticJobSearch/Patches/Patch_CompleteSetup_FilterJobSeekerTargets.cs#L139);
  profile on large cities where `CompleteSetup` runs often.
- **Version drift.** The committed `PublishConfiguration.xml` declares
  `GameVersion 1.5.*`
  (repo/RealisticJobSearch/Properties/PublishConfiguration.xml#L37-L39), but the
  live Paradox storefront reports `requiredVersion` 1.6.* (dossier
  `notes/paradox-api-20260624.json`), so the published build is ahead of the
  committed metadata.
- **Building-position fallback is a hack.** When a destination lacks a `Transform`,
  `MetricsSystem` synthesizes a pseudo-position from `Building.m_CurvePosition`
  (`new float2(cp, cp * 19f)`)
  (repo/RealisticJobSearch/Systems/MetricSystem.cs#L146-L154); those rows are not
  real world coordinates and will skew the distance histogram.

Needs Verification (in-game): the actual routing/commute change per slider, the
real frequency and cost of the `CompleteSetup` prefix under load, and whether the
acceptance-gate veto interacts cleanly with other job/economy mods - none of these
can be confirmed from static source.

## Source pointers

- Dossier:
  `../../vice-and-order-research/mods/dossiers/realistic-jobsearch/`
  (`index.md`, `source.md`, `modding.md`, `guide.md`, `notes/`).
- Repo @ `7a096b2ab974bb03cc4cf0937f250bf1d7671f31`, key files:
  - `repo/RealisticJobSearch/Mod.cs` - scheduling + debug gating + Harmony PatchAll.
  - `repo/RealisticJobSearch/Patches/Patch_CompleteSetup_FilterJobSeekerTargets.cs`
    - the private-field candidate-buffer rewrite + softmax draw.
  - `repo/RealisticJobSearch/Systems/GravityAcceptanceGateSystem.cs` - Burst
    IJobChunk acceptance gate; `PathInformation`-strip veto.
  - `repo/RealisticJobSearch/Systems/RetryThrottleSystem.cs` - cooldown/retry cap.
  - `repo/RealisticJobSearch/Systems/MetricSystem.cs` - 40-column CSV telemetry.
  - `repo/RealisticJobSearch/Systems/GravityPreFilterSystem.cs` - dormant
    scaffolding (never triggered).
  - `repo/RealisticJobSearch/Setting.cs` - sliders (no `onSettingsApplied`).
