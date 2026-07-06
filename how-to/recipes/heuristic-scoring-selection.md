---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: Heuristic scoring & (pseudo-)stochastic selection + component-strip veto"
recipe: heuristic-scoring-selection
technique_family: "AI - Heuristic scoring & (pseudo-)stochastic selection + component-strip veto"
diataxis: how-to
source_version: "~1.6.0f1 (realistic-jobsearch@7a096b2; date-pinned, static source only)"
last_reverified: "2026-07-05"
status: source-verified
canonical_mods:
  - realistic-jobsearch@7a096b2ab974bb03cc4cf0937f250bf1d7671f31
  - realistic-path-finding@50645fa6a078181365e36a42e2e27b96699bf02a
technique_applicability: [simulation]
Created: 2026-07-05
Updated: 2026-07-05
Owners:
  - codex
---

# Heuristic scoring & (pseudo-)stochastic selection + component-strip veto

> Score a set of candidates with a hand-tuned utility function, pick one either by a
> softmax draw or a deterministic hash, and - when you want to *block* a vanilla
> decision instead of steering it - strip the component the vanilla system reads so
> its downstream logic never fires.

## Problem
The base game picks job targets, transit routes, and vehicle ownership with its own
built-in heuristics, and you want to bias or override those choices: prefer bigger /
closer workplaces, spread commuters across a top-K instead of always taking the single
best, or cap how many citizens own bikes. You need three things the vanilla systems do
not hand you: a **scoring** step you control, a **selection** step that is either
randomised (variety) or reproducible across save/reload (determinism), and - for a
hard override - a way to make a vanilla system simply *not act* on a candidate.

## Solution
Build a utility score `U` per candidate from cheap features (mass, distance/time),
sort descending, then choose. Two selection idioms appear in the canonical mods and
they are **not interchangeable**:

- **Stochastic softmax draw** (Realistic Job Search): weight the top pool by
  `exp((U - maxU) / Tau)` and draw one with an RNG seeded from the in-game clock. Gives
  run-to-run variety; the RNG state matters.
- **Deterministic hash** (Realistic Path Finding): derive the "random" number from the
  entity's own identity (`Entity.Index`, or a mixed seed) so the same citizen always
  lands the same side of a threshold - no RNG state to save, reproducible across
  reloads.

For a **veto** rather than a bias, do not fight the vanilla system in a loop: run a
Burst `IJobChunk` over the finished results, and on reject `RemoveComponent` the
component the vanilla system's query requires, via an `EntityCommandBuffer`. With its
input gone, the vanilla system skips that entity entirely.

## Steps & Code

### 1. Score each candidate with a gravity-style utility

Realistic Job Search patches `PathfindSetupSystem.CompleteSetup` and, for each
job-seeker target, converts distance to rough minutes and blends workplace counts into
a "mass", then scores `U = alpha * log(1 + mass) - beta * minutes`:

```csharp
float minutes = 0f;
if (TryXZ(targetEnt, tfRO, out var txz))
{
    var meters = math.distance(originXZ, txz);
    minutes = (meters / 7f) / 60f;              // 7 m/s ~ 25 km/h
}
float total = GetTotalWorkplaces(wpRO, targetEnt);
float free  = GetFreeWorkplaces(freeRO, targetEnt);
float mass  = math.max(1f, WTotal * total + WFree * free);
float U = AlphaJobs * math.log(1f + mass) - BetaMinute * minutes;
scored.Add((pt, U));
```
Source: `../../../vice-and-order-research/mods/dossiers/realistic-jobsearch/repo/RealisticJobSearch/Patches/Patch_CompleteSetup_FilterJobSeekerTargets.cs#L103-L117` (@7a096b2ab974bb03cc4cf0937f250bf1d7671f31)

The tuning constants (`AlphaJobs`, `BetaMinute`, weights, and `TopK = 12`) are pulled
from settings once as `static readonly` fields:
`Patch_CompleteSetup_FilterJobSeekerTargets.cs#L18-L24`.

### 2. Sort descending and seed the draw from the game clock

```csharp
scored.Sort((a, b) => b.U.CompareTo(a.U));      // best first
```
Source: `../../../vice-and-order-research/mods/dossiers/realistic-jobsearch/repo/RealisticJobSearch/Patches/Patch_CompleteSetup_FilterJobSeekerTargets.cs#L123` (@7a096b2ab974bb03cc4cf0937f250bf1d7671f31)

The RNG seed is built from the current in-game minute/hour/year, so the draw is
stochastic but tied to sim time:

```csharp
uint seed = (uint)(currentDateTime.Minute * 1000 + currentDateTime.Hour * 100 + currentDateTime.Year);
Unity.Mathematics.Random random = Unity.Mathematics.Random.CreateFromIndex(seed);
```
Source: `../../../vice-and-order-research/mods/dossiers/realistic-jobsearch/repo/RealisticJobSearch/Patches/Patch_CompleteSetup_FilterJobSeekerTargets.cs#L62-L65` (@7a096b2ab974bb03cc4cf0937f250bf1d7671f31)

### 3. Softmax-draw one winner (Tau = 0.45), numerically stabilised

Shift every score by `maxU` before exponentiating so `exp` never overflows, then draw
one candidate proportional to its weight:

```csharp
const float Tau = 0.45f;                         // lower = greedier
int poolCount = math.min(TopK * 2, scored.Count);
// ... maxU = max over pool ...
double sum = 0d;
var weights = new double[poolCount];
for (int p = 0; p < poolCount; p++)
{
    double w = Math.Exp((scored[p].U - maxU) / Tau);
    weights[p] = w; sum += w;
}
double r = random.NextDouble() * sum;
int chosen = 0; double acc = 0d;
for (; chosen < poolCount; chosen++) { acc += weights[chosen]; if (acc >= r) break; }
```
Source: `../../../vice-and-order-research/mods/dossiers/realistic-jobsearch/repo/RealisticJobSearch/Patches/Patch_CompleteSetup_FilterJobSeekerTargets.cs#L127-L155` (@7a096b2ab974bb03cc4cf0937f250bf1d7671f31)

### 4. Write the winner first, then keep the next-best up to TopK

The chosen target is written to slot 0; the rest of the buffer is filled with the
remaining best candidates (skipping the one already picked) and truncated to `TopK`:

```csharp
SetAt(ref dst.m_Buffer, 0, scored[chosen].t);
int writeCountSoft = 1;
int maxKeep = math.min(TopK, scored.Count);
for (int s = 0; s < scored.Count && writeCountSoft < maxKeep; s++)
{
    if (s == chosen) continue;
    Add(ref dst.m_Buffer, scored[s].t);
    writeCountSoft++;
}
Truncate(ref dst.m_Buffer, writeCountSoft);
```
Source: `../../../vice-and-order-research/mods/dossiers/realistic-jobsearch/repo/RealisticJobSearch/Patches/Patch_CompleteSetup_FilterJobSeekerTargets.cs#L157-L170` (@7a096b2ab974bb03cc4cf0937f250bf1d7671f31)

### 5. Veto a vanilla decision by stripping its input component

For a hard override, run a Burst `IJobChunk` over finished path results
(`JobSeeker` + `Owner` + `PathInformation`) and re-score with a gravity *acceptance*
term `accept = pow(jobs, alpha) * exp(-beta * minutes)`, clamped to `[MinAccept,
MaxAccept]`:

```csharp
float minutes = math.max(0.01f, info.m_Duration / 60f);   // duration is seconds
float mass    = math.pow(jobs, m_Params.AlphaJobs);
float accept  = mass * math.exp(-m_Params.BetaMinute * minutes);
accept = math.clamp(accept, m_Params.MinAccept, m_Params.MaxAccept);

// Reject at the floor: strip PathInformation so vanilla won't start the job.
if (accept <= m_Params.MinAccept + 1e-4f)
{
    m_ECB.RemoveComponent<PathInformation>(unfilteredChunkIndex, seeker);
    // ... bump a RefusedLongCommute counter ...
}
```
Source: `../../../vice-and-order-research/mods/dossiers/realistic-jobsearch/repo/RealisticJobSearch/Systems/GravityAcceptanceGateSystem.cs#L113-L121` (@7a096b2ab974bb03cc4cf0937f250bf1d7671f31)

Because vanilla `FindJobSystem` reads `PathInformation` to start the job, removing that
component makes it skip the seeker. The job is scheduled parallel and its writes are
deferred through the `EndFrameBarrier` command buffer:

```csharp
var job = new GateJob
{
    m_EntityType   = GetEntityTypeHandle(),
    m_PathInfoType = GetComponentTypeHandle<PathInformation>(true),
    m_ECB          = m_EndBarrier.CreateCommandBuffer().AsParallelWriter(),
    // ...
};
Dependency = job.ScheduleParallel(m_ResultsQ, Dependency);
m_EndBarrier.AddJobHandleForProducer(Dependency);
```
Source: `../../../vice-and-order-research/mods/dossiers/realistic-jobsearch/repo/RealisticJobSearch/Systems/GravityAcceptanceGateSystem.cs#L61-L73` (@7a096b2ab974bb03cc4cf0937f250bf1d7671f31)

The reject query is declared in `OnCreate` as the vanilla result shape
(`JobSeeker`, `Owner`, `PathInformation`, `Exclude<Deleted>`):
`GravityAcceptanceGateSystem.cs#L47-L51`.

### 6. Deterministic alternative: hash the entity instead of drawing (RPF)

Realistic Path Finding never touches an RNG for its bike-ownership share. It maps each
citizen's `Entity.Index` through a fixed LCG constant into `0..99` and compares to a
per-age-group threshold - same entity, same result, every reload:

```csharp
uint idx   = (uint)e.Index;
uint hash  = idx * 1103515245u + 12345u;         // classic LCG constants
uint value = hash % 100u;                         // 0..99
bool shouldHaveBike = value < (uint)threshold;
CommandBuffer.SetComponentEnabled<BicycleOwner>(unfilteredChunkIndex, e, shouldHaveBike);
```
Source: `../../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Systems/BicycleOwnerLimiterSystem.cs#L137-L150` (@50645fa6a078181365e36a42e2e27b96699bf02a)

### 7. Deterministic stochasticity for route choice (Gumbel, clamped)

RPF also injects controlled randomness into car-route cost without an RNG-state problem:
it derives a seed from `(citizen, pathId, salt)`, samples a Gumbel(0,1) draw from it,
and converts that to a *bounded* multiplicative weight on car cost:

```csharp
public static float SampleGumbel01(uint seed)
{
    var rng = new Unity.Mathematics.Random(seed == 0 ? 1u : seed);
    float u = math.clamp(rng.NextFloat(), 1e-6f, 1f - 1e-6f);
    return -math.log(-math.log(u));               // Gumbel(0,1)
}

public static float CarWeightFromGumbel(float epsSeconds, float typicalSecs = 300f)
{
    float x = math.saturate(math.abs(epsSeconds)) * math.sign(epsSeconds) / math.max(30f, typicalSecs);
    return math.clamp(math.exp(x), 0.85f, 1.15f); // tame: +/-15% at most
}
```
Source: `../../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Utils/RPFRouteUtils.cs#L35-L65` (@50645fa6a078181365e36a42e2e27b96699bf02a)

The seed mixer (`MakeChoiceSeed`) folds `Entity.Index`, `Entity.Version`, the path id
and a user salt through an integer hash, so the "noise" is a pure function of the trip -
reproducible: `RPFRouteUtils.cs#L44-L54`.

## Pitfalls & gotchas

- **Stochastic vs deterministic is a real design fork.** The softmax draw (RJS, step 3)
  depends on `random.NextDouble()`, so the same city state can pick a *different*
  winner after a reload unless that RNG is reconstructed identically - here it is only
  re-seeded from the sim clock (`CreateFromIndex(seed)`,
  `Patch_CompleteSetup_FilterJobSeekerTargets.cs#L65`), so timing decides the seed. The
  hash approach (RPF, steps 6-7) has **no state to persist** and reproduces exactly.
  Choose deterministic when you need identical results across save/reload; choose the
  draw when you want run-to-run variety.

- **`Entity.Index` is not a stable citizen ID across sessions.** RPF's bike share keys
  off `(uint)e.Index` (`BicycleOwnerLimiterSystem.cs#L137`). Index values are recycled
  by the ECS and are *not* guaranteed to survive a save/reload for the same citizen, so
  "deterministic within a session" is what the code proves - persistence of the *same
  citizen's* assignment across reloads is **Needs Verification (in-game)**.

- **Clamp-then-reject makes the floor a hard cutoff.** In the veto (step 5) `accept` is
  clamped to `MinAccept` *before* the `accept <= MinAccept + 1e-4f` test
  (`GravityAcceptanceGateSystem.cs#L116-L119`). Every candidate whose raw score fell at
  or below `MinAccept` collapses onto the floor and is rejected together - `MinAccept`
  is not a soft nudge, it is a guillotine. Raising it rejects strictly more seekers.

- **Component-strip veto is a distinct idiom - and it is destructive.**
  `RemoveComponent<PathInformation>` (`GravityAcceptanceGateSystem.cs#L121`) does not
  "tell vanilla no", it deletes the data vanilla needs, so any *other* system reading
  `PathInformation` on that entity is also affected. Strip only a component you are sure
  is the specific input of the behaviour you want suppressed, and prefer the deferred
  `EntityCommandBuffer` path (as here) so removal happens at the barrier, not mid-chunk.

- **Numerical stability is mandatory for softmax.** RJS shifts by `maxU` before
  `Math.Exp` (`Patch_CompleteSetup_FilterJobSeekerTargets.cs#L133-L142`). Skip that and
  large `U` overflows `exp` to infinity and the draw breaks. Always subtract the max.

- **Whether these biases produce the intended macro behaviour** (commuters actually
  spreading across the TopK, bike modal share landing near the configured percent) is
  **Needs Verification (in-game)** - the source proves the math, not the emergent
  outcome.

## Variations

- **Greedy (argmax) instead of a draw.** Lower `Tau` toward zero and the softmax
  collapses onto the single best candidate; RJS exposes `Tau = 0.45` as the knob
  (`Patch_CompleteSetup_FilterJobSeekerTargets.cs#L127`). For pure greedy, skip steps
  3-4 and just take `scored[0]`.

- **Bias vs veto.** Steps 1-4 *reorder* the vanilla candidate buffer (a bias); step 5
  *removes* a component so vanilla never acts (a veto). Use the bias when you want to
  nudge among options, the veto when a decision must be blocked outright.

- **Additive noise -> multiplicative weight.** RPF converts an additive Gumbel time
  bias (seconds) into a bounded cost multiplier via `exp(x)` clamped to `[0.85, 1.15]`
  (`RPFRouteUtils.cs#L59-L65`), keeping the perturbation small and preventing a lucky
  draw from dominating the route cost. Pick the clamp to bound how much randomness can
  swing the outcome.

- **Threshold-by-category.** RPF selects the bike threshold per `CitizenAge` group
  before the hash test (`BicycleOwnerLimiterSystem.cs#L109-L127`), so one deterministic
  rule yields different shares per cohort. Any per-candidate category can gate the
  threshold the same way.

## See also
- Related recipes: [pathfind candidate rewrite](pathfind-candidate-rewrite.md)
  (rewriting the setup buffer these scores feed), [Burst IJobChunk](burst-ijobchunk.md)
  (the parallel-job shape the veto uses).
- Reference: [ECS fundamentals](../../explanation/ecs-fundamentals.md) (components,
  queries, and command buffers).
- Case studies demonstrating it:
  [realistic-jobsearch](../../case-studies/realistic-jobsearch.md).

## Sources
- Canonical mods (dossier + repo):
  - `realistic-jobsearch` @7a096b2ab974bb03cc4cf0937f250bf1d7671f31 -
    `repo/RealisticJobSearch/Patches/Patch_CompleteSetup_FilterJobSeekerTargets.cs`,
    `repo/RealisticJobSearch/Systems/GravityAcceptanceGateSystem.cs`
  - `realistic-path-finding` @50645fa6a078181365e36a42e2e27b96699bf02a -
    `repo/RealisticPathFinding/Utils/RPFRouteUtils.cs`,
    `repo/RealisticPathFinding/Systems/BicycleOwnerLimiterSystem.cs`
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding
