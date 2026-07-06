---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: Rewrite pathfinding candidate buffers via a CompleteSetup prefix"
recipe: pathfind-candidate-rewrite
technique_family: "AH - Rewrite pathfinding candidate buffers via a CompleteSetup prefix"
diataxis: how-to
source_version: "~1.6.0f1 (realistic-jobsearch@7a096b2; date-pinned, static source only)"
last_reverified: "2026-07-05"
status: source-verified
canonical_mods:
  - realistic-jobsearch@7a096b2ab974bb03cc4cf0937f250bf1d7671f31
technique_applicability: [economy, simulation]
Created: 2026-07-05
Updated: 2026-07-05
Owners:
  - codex
---

# Rewrite pathfinding candidate buffers via a CompleteSetup prefix

> Re-rank, filter, or trim the vanilla pathfinding candidate set in place - without
> replacing the system that consumes it - by prefixing `PathfindSetupSystem.CompleteSetup`,
> reaching its private `NativeList` buffers through compiled field refs, and mutating
> the per-action `UnsafeList<PathTarget>` with no allocations.

## Problem
The game builds pathfinding "setup" work items (origin + candidate destinations) in a
private `NativeList` on `PathfindSetupSystem`, then hands them to consumers like
`FindJobSystem`. You want to change *which* candidates survive or *what order* they are
in - for example, make job seekers prefer nearby workplaces with free openings instead
of the vanilla near-random pick - but you do **not** want to rewrite the whole consuming
system, duplicate its query wiring, or fight its job scheduling. You need to reach inside
the vanilla buffers at the exact moment they are complete and mutate them in place.

## Solution
Harmony-prefix `PathfindSetupSystem.CompleteSetup`. That method is where the game
finalizes the setup list, so a prefix runs with the buffers populated and about to be
consumed. The private members you need (`m_SetupList`, `m_SetupDependencies`) are reached
with `AccessTools.FieldRefAccess`, which compiles a direct ref accessor once (hot-path
safe, unlike `Traverse`/`FieldInfo` per-call reflection). Inside the prefix you first
**force-complete** the setup `JobHandle` so the native buffers are safe to touch on the
main thread, then walk the list, and for each end-target buffer of the type you care
about you overwrite its `UnsafeList<PathTarget>` in place using no-alloc helpers that
poke `list.Length` and `list.ElementAt(i)` directly. You never replace the system and
never reallocate the buffer.

## Steps & Code

### 1. Attach the prefix to `CompleteSetup`, not `FindTargets`

The patch targets the *setup finalization* method by string name. This is the key
distinction from candidate-replacement recipes that prefix the target-*finding* method
and swap `__result` (family U): here you mutate the shared setup buffers the consumer
will read, leaving the consumer untouched.

```csharp
[HarmonyPatch(typeof(PathfindSetupSystem), "CompleteSetup")]
public static class Patch_CompleteSetup_FilterJobSeekerTargets
{
    static readonly float AlphaJobs = Mod.m_Setting.alpha_jobs;
    static readonly float BetaMinute = Mod.m_Setting.beta_minute;
    static readonly int TopK = 12;       // cap list size for performance
    static readonly float MinKeepP = 0.05f;
```
Source: `../../../vice-and-order-research/mods/dossiers/realistic-jobsearch/repo/RealisticJobSearch/Patches/Patch_CompleteSetup_FilterJobSeekerTargets.cs#L15-L24` (@7a096b2ab974bb03cc4cf0937f250bf1d7671f31)

### 2. Build compiled field refs into the private members

`AccessTools.FieldRefAccess<TInstance, TField>("name")` returns a delegate you invoke as
`ref field(__instance)`. Building it once as a `static readonly` means each hot-path
access is a direct field read - no `FieldInfo.GetValue` boxing, no `Traverse` walk.

```csharp
static readonly AccessTools.FieldRef<PathfindSetupSystem, NativeList<PathfindSetupSystem.SetupListItem>> _setupListRef =
    AccessTools.FieldRefAccess<PathfindSetupSystem, NativeList<PathfindSetupSystem.SetupListItem>>("m_SetupList");

static readonly AccessTools.FieldRef<PathfindSetupSystem, Unity.Jobs.JobHandle> _setupDepsRef =
    AccessTools.FieldRefAccess<PathfindSetupSystem, Unity.Jobs.JobHandle>("m_SetupDependencies");
```
Source: `../../../vice-and-order-research/mods/dossiers/realistic-jobsearch/repo/RealisticJobSearch/Patches/Patch_CompleteSetup_FilterJobSeekerTargets.cs#L27-L31` (@7a096b2ab974bb03cc4cf0937f250bf1d7671f31)

### 3. In the prefix, force-complete the dependency before touching native memory

The buffers are filled by scheduled jobs. Reading or writing them from the main thread
before those jobs finish is a data race. The prefix grabs the setup `JobHandle` by ref
and calls `Complete()` - the same thing the original method would do - so the mutation is
safe. Then it early-outs on an empty list.

```csharp
static void Prefix(PathfindSetupSystem __instance)
{
    // Ensure the jobs that filled m_SetupList buffers are finished.
    ref var setupDeps = ref _setupDepsRef(__instance);
    setupDeps.Complete();

    ref var setupList = ref _setupListRef(__instance);
    if (setupList.Length == 0) return;

    var tfRO = __instance.GetComponentLookup<Transform>(isReadOnly: true);
    var wpRO = __instance.GetComponentLookup<WorkProvider>(isReadOnly: true);
    var freeRO = __instance.GetComponentLookup<FreeWorkplaces>(isReadOnly: true);
```
Source: `../../../vice-and-order-research/mods/dossiers/realistic-jobsearch/repo/RealisticJobSearch/Patches/Patch_CompleteSetup_FilterJobSeekerTargets.cs#L49-L60` (@7a096b2ab974bb03cc4cf0937f250bf1d7671f31)

### 4. Walk the list; select only the buffers you own by target type

Setup items come in start/end pairs keyed by `m_ActionIndex`. This mod first records
each action's origin position from its `CurrentLocation` start item, then processes only
the end items whose target type is `JobSeekerTo`, leaving every other kind of pathfinding
request untouched. `setupList.ElementAt(i)` yields a `ref` to the item so `dst.m_Buffer`
below is the live buffer, not a copy.

```csharp
for (int i = 0; i < setupList.Length; i++)
{
    ref var dst = ref setupList.ElementAt(i);
    if (dst.m_ActionStart) continue;                              // only end-target buffers
    if (dst.m_Target.m_Type != SetupTargetType.JobSeekerTo) continue;
    if (!originByAction.TryGetValue(dst.m_ActionIndex, out var originXZ)) continue;

    var buf = dst.m_Buffer;
    if (buf.Length == 0) continue;
```
Source: `../../../vice-and-order-research/mods/dossiers/realistic-jobsearch/repo/RealisticJobSearch/Patches/Patch_CompleteSetup_FilterJobSeekerTargets.cs#L82-L91` (@7a096b2ab974bb03cc4cf0937f250bf1d7671f31)

### 5. Rewrite the buffer in place with no-alloc `UnsafeList` helpers

After scoring the candidates (the scoring itself is a separate concern - see
[heuristic scoring & selection](heuristic-scoring-selection.md)), the mod writes the
chosen candidate into slot 0, appends the next-best up to `TopK`, then truncates. Every
mutation goes through helpers that set `list.Length` and assign via `list.ElementAt(idx)`
- no `Add`/`RemoveAt` that would reallocate or shuffle.

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
Source: `../../../vice-and-order-research/mods/dossiers/realistic-jobsearch/repo/RealisticJobSearch/Patches/Patch_CompleteSetup_FilterJobSeekerTargets.cs#L158-L170` (@7a096b2ab974bb03cc4cf0937f250bf1d7671f31)

### 6. The no-alloc helper definitions

These are the whole trick for mutating `UnsafeList<PathTarget>` without allocating.
Truncating is just lowering `Length`; growing by one is raising `Length` then writing the
new tail via `ElementAt`. `AggressiveInlining` keeps them free in the hot loop.

```csharp
[MethodImpl(MethodImplOptions.AggressiveInlining)]
static void Clear(ref UnsafeList<PathTarget> list) => list.Length = 0;

[MethodImpl(MethodImplOptions.AggressiveInlining)]
static void Truncate(ref UnsafeList<PathTarget> list, int newLen)
    => list.Length = math.clamp(newLen, 0, list.Length);

[MethodImpl(MethodImplOptions.AggressiveInlining)]
static void SetAt(ref UnsafeList<PathTarget> list, int idx, in PathTarget val)
{
    if (idx >= list.Length) list.Length = idx + 1;
    list.ElementAt(idx) = val;
}

[MethodImpl(MethodImplOptions.AggressiveInlining)]
static void Add(ref UnsafeList<PathTarget> list, in PathTarget val)
{
    int idx = list.Length;
    list.Length = idx + 1;
    list.ElementAt(idx) = val;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/realistic-jobsearch/repo/RealisticJobSearch/Patches/Patch_CompleteSetup_FilterJobSeekerTargets.cs#L180-L202` (@7a096b2ab974bb03cc4cf0937f250bf1d7671f31)
(Namespaces shortened for readability; the source qualifies these as `Unity.Collections.LowLevel.Unsafe.UnsafeList<PathTarget>`.)

### 7. Companion ordered systems run before the consumer

The prefix does not stand alone. The mod registers its ECS systems to run *before* the
vanilla consumer via `UpdateBefore<..., FindJobSystem>(SystemUpdatePhase.GameSimulation)`
- three unconditionally (`GravityPreFilterSystem`, `GravityAcceptanceGateSystem`,
`RetryThrottleSystem`), plus a fourth (`MetricsSystem`) only when debug output is on.
Ordering before `FindJobSystem` is what makes prefixing *its* setup source meaningful.

```csharp
updateSystem.UpdateBefore<GravityPreFilterSystem,
                         Game.Simulation.FindJobSystem>(SystemUpdatePhase.GameSimulation);
updateSystem.UpdateBefore<GravityAcceptanceGateSystem,
                         Game.Simulation.FindJobSystem>(SystemUpdatePhase.GameSimulation);
updateSystem.UpdateBefore<RetryThrottleSystem,
                         Game.Simulation.FindJobSystem>(SystemUpdatePhase.GameSimulation);
```
Source: `../../../vice-and-order-research/mods/dossiers/realistic-jobsearch/repo/RealisticJobSearch/Mod.cs#L33-L39` (@7a096b2ab974bb03cc4cf0937f250bf1d7671f31)

The prefix is installed with a plain `harmony.PatchAll(typeof(Mod).Assembly)` in `OnLoad`
(`Mod.cs#L51-L53`).

## Pitfalls & gotchas

- **You are mutating shared vanilla buffers - complete the dependency first.** The setup
  list is filled by scheduled jobs. Skipping `m_SetupDependencies.Complete()` before
  touching `m_SetupList` is a race that can crash or corrupt (`Patches/...#L52-L53`).
  This prefix explicitly re-does the completion the original would perform.

- **`FieldRefAccess`, not `Traverse`/`FieldInfo`, on the hot path.** `CompleteSetup` runs
  every simulation tick. `AccessTools.FieldRefAccess` compiles the accessor once into a
  `static readonly` delegate (`Patches/...#L27-L31`); resolving the field per call with
  `Traverse` or `FieldInfo.GetValue` would allocate/box and be far slower. If you need the
  reflection-driven read/write shape for a cold path instead, see
  [Harmony Traverse private fields](harmony-traverse-private-fields.md).

- **`ElementAt` returns a ref into unmanaged memory - do not hold it across a resize.**
  The `SetAt`/`Add` helpers write `list.ElementAt(idx)` immediately after adjusting
  `Length`. Caching that ref and then changing `Length` would leave you pointing at the
  wrong slot (`Patches/...#L189-L202`).

- **Setting `Length` up does not zero the new slots.** `Add`/`SetAt` raise `Length` then
  always overwrite the new tail in the same breath; if you grow without writing, the slot
  holds stale `PathTarget` data. Never leave a raised `Length` unassigned.

- **String-named method target is brittle across patches.** `[HarmonyPatch(typeof(
  PathfindSetupSystem), "CompleteSetup")]` binds by name (`Patches/...#L15`); a game update
  that renames or re-signatures `CompleteSetup`, `m_SetupList`, or `m_SetupDependencies`
  breaks the patch (a missing field ref throws at static-init time). Re-verify these member
  names every CS2 release. **Needs Verification (in-game)** that the current live 1.6.0f1
  binary still exposes these exact members.

- **This is not `FindTargets`/`__result` replacement.** Prefixing the target-*finding*
  method and overwriting its result buffer is a different technique (family U). Here the
  consumer (`FindJobSystem`) is left entirely intact and only its *input* setup buffers are
  rewritten - do not conflate the two or you will patch the wrong method.

- **Whether the rewritten ordering actually changes in-game job assignment** (i.e. the
  consumer honors buffer order/contents as expected) is **Needs Verification (in-game)** -
  the source shows the mutation, not the downstream effect.

## Variations

- **Filter-only (drop candidates, keep order).** Instead of re-ranking, just skip
  candidates that fail a predicate and `Truncate` to the survivors, or `Clear` the buffer
  to reject all of them for that action (`Patches/...#L120`, `#L181`). Same helpers, no
  scoring.

- **Different target type.** Swap `SetupTargetType.JobSeekerTo` (`Patches/...#L86`) for
  another setup target type to intercept a different pathfinding intent (e.g. shopping,
  leisure) with the same prefix skeleton.

- **Pair with a pre-filter ECS system instead of doing everything in the prefix.** This
  mod splits work: ordered `UpdateBefore<..., FindJobSystem>` systems (`Mod.cs#L33-L39`)
  plus the buffer-rewrite prefix. You can push more logic into the ordered systems and keep
  the prefix minimal, or vice versa.

## See also
- Explanation: [Harmony patching](../../explanation/harmony-patching.md) (prefix/postfix
  model, `PatchAll`, patch lifetime).
- Related recipes: [Harmony Traverse private fields](harmony-traverse-private-fields.md)
  (the cold-path reflection alternative to `FieldRefAccess`);
  [heuristic scoring & selection](heuristic-scoring-selection.md) (the scoring that decides
  which candidates win before this recipe writes them back).
- Case study demonstrating it: [realistic-jobsearch](../../case-studies/realistic-jobsearch.md).

## Sources
- Canonical mods (dossier + repo):
  - `realistic-jobsearch` @7a096b2ab974bb03cc4cf0937f250bf1d7671f31 -
    `repo/RealisticJobSearch/Patches/Patch_CompleteSetup_FilterJobSeekerTargets.cs`,
    `repo/RealisticJobSearch/Mod.cs`
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding
