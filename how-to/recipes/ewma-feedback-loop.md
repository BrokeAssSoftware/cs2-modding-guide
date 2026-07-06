---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: Telemetry-fed feedback loop (asymmetric EWMA)"
recipe: ewma-feedback-loop
technique_family: "AJ - Telemetry-fed feedback loop (asymmetric EWMA)"
diataxis: how-to
source_version: "1.6.0f1 (realistic-path-finding@50645fa; date-pinned, static source only)"
last_reverified: "2026-07-05"
status: source-verified
canonical_mods:
  - realistic-path-finding@50645fa6a078181365e36a42e2e27b96699bf02a
technique_applicability: [simulation]
Created: 2026-07-05
Updated: 2026-07-05
Owners:
  - codex
---

# Telemetry-fed feedback loop (asymmetric EWMA)

> Sample a noisy per-frame signal, smooth it with an exponential-weighted moving
> average that rises fast but recovers slow, then step-limit the smoothed value
> back into a cost/density field - so a single spike can never destabilize the graph.

## Problem
You want a live simulation signal (traffic congestion, queue length, demand
pressure) to feed back into a cost field the game already reads - but the raw
signal is jittery. Vehicles change lanes, samples arrive in bursts, and one slow
truck should not spike a whole edge. If you write the raw sample straight back you
get oscillation: the field jumps up, path costs change, traffic reroutes, the
signal drops, the field jumps down, traffic returns - a feedback loop that never
settles. You need smoothing, and you need the smoothing to model real hysteresis
(congestion builds fast, clears slow), plus a hard limit on how far one update can
move the field.

## Solution
Realistic Path Finding's `CarCongestionEwmaSystem` is a single system that does the
whole loop: a Burst job times each car on its current lane and emits one sample when
the car changes lane; the main thread drains those samples into a per-lane average,
runs an **asymmetric EWMA** (a larger `alpha` when the signal is *rising*, a smaller
`alpha` when it is *falling*, so recovery is deliberately slower than build-up),
then converts the smoothed travel time into a density delta and writes it into the
path graph **step-limited** - clamped to a small max step and applied as a delta
against the last value it pushed, never re-stacked. The asymmetry is the whole point:
rise fast, recover slow, and cap the per-update movement.

## Steps & Code

### 1. Pre-allocate persistent scratch maps once; drain them fresh each frame

Allocate the per-lane aggregator and the "last value I pushed" map once in
`OnCreate` at a fixed capacity (here 4096 entries) with `Allocator.Persistent`, and
dispose them in `OnDestroy`. Reusing the allocation avoids per-frame GC/alloc churn;
you clear (not reallocate) the aggregator each frame.

```csharp
_sampleQueue = new NativeQueue<Sample>(Allocator.Persistent);
_lastDensityAdd = new NativeParallelHashMap<Entity, float>(4096, Allocator.Persistent);
_agg = new NativeParallelHashMap<Entity, Agg>(4096, Allocator.Persistent);
```
Source: `../../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Systems/CarCongestionEwmaSystem.cs#L81-L83` (@50645fa6a078181365e36a42e2e27b96699bf02a)

The config singleton carries the tunables with "sane default" seeds - note
`Alpha = 0.20f` (the *rise* rate) and the caps that later bound the feedback:

```csharp
Alpha = 0.20f,
UpdateThresholdSec = 0.5f,   // skip writes smaller than this
MaxSlowdownRatio = 3.0f,     // cap EWMA/freeflow
MaxDensityAdd = 0.50f,       // cap the density add
```
Source: `../../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Systems/CarCongestionEwmaSystem.cs#L91-L94` (@50645fa6a078181365e36a42e2e27b96699bf02a)

### 2. Sample the signal in a Burst job - emit only meaningful samples

The sampler accumulates elapsed time while a car stays on a lane and emits **one**
sample when the car leaves that lane, gated by `MinEmitSec` so micro-samples are
dropped. Emitting per lane-change (not per frame) is what keeps the sample stream
sane. This is an `IJobEntity` (see the gotcha on job kinds below):

```csharp
void Execute(Entity v, ref LaneSampleState st, in CarCurrentLane cur)
{
    Entity laneNow = cur.m_Lane;
    if (laneNow == Entity.Null || laneNow == st.Lane) { st.ElapsedSec += DeltaTime; return; }
    if (st.Lane != Entity.Null && st.ElapsedSec >= MinEmitSec)
        Out.Enqueue(new Sample { Owner = st.Lane, TravelSec = st.ElapsedSec });
    st.Lane = laneNow;      // start timing the new lane
    st.ElapsedSec = 0f;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Systems/CarCongestionEwmaSystem.cs#L266-L286` (@50645fa6a078181365e36a42e2e27b96699bf02a)

### 3. Drain samples into a per-lane average

`Clear()` (do not reallocate) the aggregator, then dequeue every sample into a
`(sum, count)` per lane. `math.max(0.01f, ...)` floors each sample so a zero can
never poison the average:

```csharp
_agg.Clear();
while (_sampleQueue.TryDequeue(out var sample))
{
    if (!_agg.TryGetValue(sample.Owner, out var a)) a = default;
    a.Sum += math.max(0.01f, sample.TravelSec);
    a.Cnt += 1;
    _agg[sample.Owner] = a;
}
if (_agg.Count() == 0) return;
```
Source: `../../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Systems/CarCongestionEwmaSystem.cs#L161-L170` (@50645fa6a078181365e36a42e2e27b96699bf02a)

### 4. Run the asymmetric EWMA (rise fast, recover slow)

This is the heart of the recipe. Pick the EWMA weight by direction: full `Alpha`
when the new sample is *above* the smoothed value (congestion rising), but
`Alpha * 0.7` when it is *below* (recovering). A smaller alpha weights the old
value more, so the smoothed value decays back toward free-flow **slower** than it
climbed - modelling how real congestion builds quickly and clears gradually.

```csharp
float sampleAvg = a.Sum / math.max(1, a.Cnt);
float old = t.EwmaSec;
float alphaUp = cfg.Alpha;
float alphaDown = cfg.Alpha * 0.7f;                 // slower recovery
float alpha = (sampleAvg > old) ? alphaUp : alphaDown;
float ewma = alpha * sampleAvg + (1f - alpha) * old;
```
Source: `../../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Systems/CarCongestionEwmaSystem.cs#L203-L208` (@50645fa6a078181365e36a42e2e27b96699bf02a)

Then a **dead-band**: if the smoothed value barely moved (below
`UpdateThresholdSec`), skip the write entirely. This stops a stream of tiny
adjustments from churning the graph:

```csharp
if (math.abs(ewma - old) < cfg.UpdateThresholdSec)
    continue;                                       // small change -> skip write
t.EwmaSec = ewma;
```
Source: `../../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Systems/CarCongestionEwmaSystem.cs#L210-L216` (@50645fa6a078181365e36a42e2e27b96699bf02a)

### 5. Map smoothed value to a density add, clamped

Convert the smoothed travel time into a ratio against free-flow, apply the global
mode bias, and clamp both the ratio and the resulting density add. The clamps keep
the feedback bounded no matter how extreme the sample:

```csharp
float ratio = ewma / math.max(0.01f, t.FreeflowSec);
ratio += (CarModeWeight - 1f);                       // global mode bias
ratio = math.clamp(ratio, 0f, cfg.MaxSlowdownRatio);
float densityAdd = math.clamp(ratio - 1f, -cfg.MaxSlowdownRatio, cfg.MaxDensityAdd);
```
Source: `../../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Systems/CarCongestionEwmaSystem.cs#L222-L234` (@50645fa6a078181365e36a42e2e27b96699bf02a)

### 6. Step-limit the write-back and apply a delta, not a stack

Do not write `densityAdd` straight onto the field. Remember the last value you
pushed for this lane (`_lastDensityAdd`), compute the **delta** since then, clamp
that delta to a small `maxStep` (here `0.08`), and only apply if it clears a tiny
epsilon. This is what prevents a single spike from destabilizing the graph - the
field can only move a little per update, and re-running with the same `densityAdd`
produces a zero delta (idempotent, no stacking):

```csharp
float prev = _lastDensityAdd.TryGetValue(lane, out var p) ? p : 0f;
float delta = densityAdd - prev;
float maxStep = 0.08f;
delta = math.clamp(delta, -maxStep, maxStep);
if (math.abs(delta) >= 1e-4f)
{
    ref float density = ref data.SetDensity(eid);
    density += delta;
    _lastDensityAdd[lane] = densityAdd;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Systems/CarCongestionEwmaSystem.cs#L239-L248` (@50645fa6a078181365e36a42e2e27b96699bf02a)

## Pitfalls & gotchas

- **The asymmetry is the recipe - do not make it symmetric.** `alphaDown =
  cfg.Alpha * 0.7f` is the deliberate hysteresis: the smoothed value climbs at full
  `Alpha` but decays at 70% of it, so congestion "sticks" a little after traffic
  clears (`CarCongestionEwmaSystem.cs#L205-L207`). A symmetric EWMA (same alpha both
  directions) will flap up and down as traffic reroutes around the cost you just
  raised. That the asymmetry actually damps real oscillation in a running city is
  `Needs Verification (in-game)`.

- **Skip the write-back guards and you get a feedback storm.** Three independent
  brakes must all stay in: the dead-band (`UpdateThresholdSec`,
  `#L210-L214`), the ratio/density clamps (`#L230-L234`), and the per-update
  `maxStep = 0.08f` on the delta (`#L241-L242`). Remove any one and the field can
  move far enough in a single pass to reroute traffic hard, invalidating the next
  sample. The step-limit is the last line of defence against a single spike.

- **Apply a delta against your own last write, never re-add.** `_lastDensityAdd`
  stores what this system last pushed per lane so it applies `densityAdd - prev`,
  not `densityAdd` (`#L239-L247`). Without it, every update stacks onto the field
  and the density runs away. This is the same non-stacking discipline as prefab
  overrides - derive the write from a remembered baseline, not the live value.

- **Fixed 4096 capacity is not auto-growing.** `_agg` and `_lastDensityAdd` are
  allocated at 4096 (`#L82-L83`). `NativeParallelHashMap` does not silently resize;
  a city whose per-interval active-lane set exceeds capacity is an edge case this
  source does not handle. `Needs Verification (in-game)` whether large cities ever
  exceed it at this update cadence.

- **`Dependency.Complete()` right after scheduling stalls the worker threads.** The
  system schedules the Burst sampler then immediately blocks to drain the queue on
  the main thread (`#L157-L158`). That is a deliberate simplification (drain now,
  apply now), but it forfeits job overlap - acceptable only because the cadence is
  coarse (see next).

- **This runs on a coarse cadence, not every frame.**
  `GetUpdateInterval` returns `262144 / 64` (commented "~every 7.5 in-game
  minutes", `#L111-L115`). The whole loop - sample, smooth, feed back - is designed
  for infrequent correction, not per-frame control. Tune your own cadence to your
  signal; a per-frame version would need the guards even more.

- **Job kind: this sampler is `IJobEntity`.** `SampleJob` is declared
  `IJobEntity` and scheduled with `ScheduleParallel`
  (`#L258`, `#L157`). RPF mixes job kinds across its systems and the dossier flags
  which are `IJobEntity` vs `IJobChunk` as `Needs Verification` per system - so
  confirm the kind in *any other* RPF system before copying its scheduling shape.
  For the chunk-iteration alternative see the burst-ijobchunk recipe.

## Variations

- **Symmetric EWMA (no hysteresis).** Set `alphaDown = alphaUp` (i.e. drop the
  `* 0.7f`) for a signal where rise and fall should smooth identically - e.g. a
  demand meter with no physical "sticky" behaviour. You lose oscillation damping, so
  keep the dead-band and step-limit.

- **Tune the asymmetry ratio to the domain.** `0.7` is RPF's choice for road
  congestion; a slower-recovering signal (e.g. reputation, pollution decay) wants a
  smaller multiplier (stickier), a fast-clearing one a larger multiplier (closer to
  symmetric). It is a single constant at `#L206`.

- **Different feedback target.** RPF writes into the path-graph density via
  `data.SetDensity(eid)` (`#L245`). The same sample -> smooth -> step-limit shape
  applies to any writable field: a prefab component, a cost buffer, a settings-driven
  multiplier. Only the write-back line changes; keep the EWMA + delta + clamp core.

- **Emit-rate control lives in the sampler, not the smoother.** RPF gates samples by
  `MinEmitSec` and emits per lane-change (`#L278-L285`). If your signal is denser,
  down-sample there (drop or bucket samples) rather than loosening the EWMA - the
  smoother should see a clean stream.

## See also
- Related recipes: [Burst IJobChunk](burst-ijobchunk.md) (the chunk-iteration job
  shape and when to prefer it over `IJobEntity`).
- Reference: [ECS fundamentals](../../explanation/ecs-fundamentals.md) (components,
  queries, jobs, and `ComponentLookup`).
- Operations: [Memory and performance](../operations/memory-and-performance.md)
  (persistent `NativeParallelHashMap` allocation and per-frame drain discipline).
- Case study demonstrating it: [realistic-path-finding](../../case-studies/realistic-path-finding.md).

## Sources
- Canonical mods (dossier + repo):
  - `realistic-path-finding` @50645fa6a078181365e36a42e2e27b96699bf02a - `repo/RealisticPathFinding/Systems/CarCongestionEwmaSystem.cs`
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding
