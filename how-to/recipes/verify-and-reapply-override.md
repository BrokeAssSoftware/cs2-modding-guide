---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: Verify-and-reapply self-healing override of a recomputed ECS value"
recipe: verify-and-reapply-override
technique_family: "AE - Verify-and-reapply self-healing override of a recomputed ECS value"
diataxis: how-to
source_version: "~1.5.10f1 (advanced-road-naming@559e72c; date-pinned, static source only)"
last_reverified: "2026-07-05"
status: source-verified
canonical_mods:
  - advanced-road-naming@559e72cdb3180e3e71869643367e094a22eed988
technique_applicability: [core, simulation]
Created: 2026-07-05
Updated: 2026-07-05
Owners:
  - codex
---

# Verify-and-reapply self-healing override of a recomputed ECS value

> Make a granular edit stick on an ECS value the engine keeps **recomputing** -
> schedule your work around the vanilla producer, then run a multi-tick
> verify-and-reapply loop so a re-merge or recompute never silently reverts you.
> No Harmony.

## Problem
A [prefab-field override](prefab-field-override.md) works because prefab data is
static: you write it once and nothing else touches it. But some values are **derived,
not authored** - the engine recomputes them every time its inputs change. Cities:
Skylines II's `AggregateSystem` continuously rebuilds road *aggregates* (the grouped
runs of edges that share one name) from raw `Edge`/`Aggregated` topology. Split one
aggregate to give a subsection its own name and, a few ticks later, the vanilla
producer re-merges it and your edit is gone. Writing harder or more often does not
help - you are fighting a system that runs after you and wins.

This is Advanced Road Naming's own stated top contribution: making a granular edit
**stick on a recomputed ECS aggregate**. The technique generalises far beyond road
naming to any value a vanilla system re-derives on its own schedule.

## Solution
Treat your override as something to *defend*, not just *write*. Three moving parts:

1. **Schedule around the vanilla producer.** Register one system to run **before**
   `AggregateSystem` (to protect intended state going in) and one to run **after** it
   (to inspect the result and heal it), so you always get the last word each frame.
2. **Verify with a bidirectional readback** on a **multi-tick stability window** -
   re-read the live ECS state, confirm your edit survived *and* that nothing you did
   not intend leaked in, and only declare victory after it has held stable for N
   consecutive checks (so you never reapply mid-recompute).
3. **Reapply adaptively and with a cap** when the readback fails: if the current
   owner is exactly your selected set, a cheap re-set suffices; otherwise re-do the
   full re-partition. Bound every loop with a named attempt cap so a value that can
   never be made to stick eventually gives up instead of spinning forever.

## Steps & Code

### 1. Schedule one system before and one after the vanilla producer

Order your systems around `AggregateSystem` in `ModificationEnd`. The metadata system
runs *after* it (to verify and heal); a lightweight protection system runs *before*
it (to mark intended state so vanilla is less likely to clobber it in the first
place):

```csharp
updateSystem.UpdateAfter<SegmentMetadataSystem, AggregateSystem>(SystemUpdatePhase.ModificationEnd);
updateSystem.UpdateBefore<RoadAggregateProtectionSystem, AggregateSystem>(SystemUpdatePhase.ModificationEnd);
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Mod.cs#L56-L57` (@559e72cdb3180e3e71869643367e094a22eed988)

The "before" system is a thin shell that delegates to the metadata system's
pre-vanilla hook, so all state lives in one place:

```csharp
protected override void OnUpdate()
{
    _metadataSystem?.ProtectModAggregatesBeforeVanilla();
}
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Systems/RoadAggregateProtectionSystem.cs#L15-L18` (@559e72cdb3180e3e71869643367e094a22eed988)

### 2. Name every cap and delay as a constant

Do not scatter magic numbers through the loop. The stability window, the retry
cadence, and the give-up thresholds are all named constants at the top of the system:

```csharp
private const int AggregateStabilityInitialDelayTicks = 2;
private const int AggregateStabilityRetryDelayTicks = 10;
private const int AggregateStabilityStableChecksRequired = 3;
private const int AggregateStabilityMaxReapplyAttempts = 3;
private const int DeferredNameReapplyDelayTicks = 5;
private const int DeferredNameReapplyRetryDelayTicks = 10;
private const int DeferredNameReapplyMaxAttempts = 10;
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Systems/SegmentMetadataSystem.cs#L22-L28` (@559e72cdb3180e3e71869643367e094a22eed988)

### 3. Register a stability check right after you make the edit

When the edit succeeds, do not assume it will hold. Enqueue a check object seeded with
the initial delay and the attempt budget; the loop in step 4 drains it:

```csharp
var check = new AggregateSplitStabilityCheck
{
    SourceAggregate = sourceAggregate,
    SelectedFinalName = selectedFinalName ?? string.Empty,
    OriginalVisibleName = originalVisibleName ?? string.Empty,
    Operation = operation ?? string.Empty,
    TicksUntilNextCheck = AggregateStabilityInitialDelayTicks,
    ReapplyAttemptsRemaining = AggregateStabilityMaxReapplyAttempts
};
check.SelectedEdges.AddRange(selectedEdges);
check.RemainderEdges.AddRange(remainderEdges);
_aggregateStabilityChecks.Add(check);
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Systems/SegmentMetadataSystem.cs#L360-L371` (@559e72cdb3180e3e71869643367e094a22eed988)

### 4. Run the verify loop each tick with a stability gate

This is the heart of the technique. Each due check re-reads live state; **only after
`AggregateStabilityStableChecksRequired` consecutive clean reads** does it retire the
check. A single failure resets the counter and triggers a bounded reapply. The
counter is what stops you reapplying *during* an in-progress recompute:

```csharp
var stable = CheckAggregateStability(check, "PostUpdateTick");
if (stable)
{
    check.StableChecks++;
    if (check.StableChecks >= AggregateStabilityStableChecksRequired)
        _aggregateStabilityChecks.RemoveAt(i);   // held N ticks -> done
    else
        check.TicksUntilNextCheck = AggregateStabilityRetryDelayTicks;
    continue;
}

check.StableChecks = 0;                            // regressed -> restart the window
if (check.ReapplyAttemptsRemaining <= 0) { _aggregateStabilityChecks.RemoveAt(i); continue; }
check.ReapplyAttemptsRemaining--;
ReapplyAggregateStabilityCheck(check);
check.TicksUntilNextCheck = AggregateStabilityRetryDelayTicks;
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Systems/SegmentMetadataSystem.cs#L396-L424` (@559e72cdb3180e3e71869643367e094a22eed988)

### 5. Verify with a bidirectional readback

A one-way check ("did my value survive?") is not enough - a recompute can also drag
in edges you excluded. The verify walks **both** sides: every selected edge must now
be owned by the selected set under the expected name, and every remainder edge must
*not* be. This slice shows the selected-side assertion (the remainder side mirrors it
with inverted conditions):

```csharp
var owner = GetAuthoritativeNameEntity(edge);
var ownerName = GetCurrentAuthoritativeName(owner, string.Empty);
if (owner == Entity.Null || !string.Equals(ownerName, check.SelectedFinalName, System.StringComparison.Ordinal))
{
    ok = false;   // my edit did NOT survive on this edge
    Mod.log.Warn(() => $"Aggregate stability selected-edge mismatch. ...");
}
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Systems/SegmentMetadataSystem.cs#L447-L455` (@559e72cdb3180e3e71869643367e094a22eed988)

The method then cross-checks ownership the other direction - it fetches each owner's
current edge list and fails if a selected owner now contains remainder edges (or vice
versa), returning `ok && validSelected > 0`
(`SegmentMetadataSystem.cs#L474-L508`). That "at least one still-valid edge" clause
means a fully-deleted selection retires the check rather than looping on ghosts.

### 6. Reapply adaptively - cheap re-set vs. full re-partition

When you do reapply, match effort to the damage. If the aggregate the engine produced
already contains exactly your selected edges, just re-stamp the name (cheap). Only if
it re-merged your edges back into a larger aggregate do you pay for a full
re-partition:

```csharp
var ownerEdges = GetAggregateRoadEdges(owner);
if (ownerEdges.Count == 0 || AllEdgesSelected(ownerEdges, selectedSet))
{
    SetAuthoritativeName(owner, check.SelectedFinalName, "PostUpdateReapply:" + check.Operation, "SelectedAggregateReapply", pair.Value.Count, ownerEdges.Count);
    continue;   // owner == selected set -> cheap re-set
}

TryPartitionAggregateForSelectedEdges(owner, pair.Value, selectedSet, "PostUpdateReapply:" + check.Operation, check.SelectedFinalName, check.OriginalVisibleName, false);
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Systems/SegmentMetadataSystem.cs#L539-L546` (@559e72cdb3180e3e71869643367e094a22eed988)

### 7. Defer the post-load reapply behind a "world ready" gate

The biggest recompute event is loading a save: the game rebuilds all aggregates from
scratch, so your persisted overrides must be re-stamped afterward - but not *during*
deserialisation. Queue the reapply, tick down a delay, and only run once a safety
predicate says the world is genuinely playable; on exception, retry up to
`DeferredNameReapplyMaxAttempts`:

```csharp
if (_pendingPostLoadNameReapplyDelayTicks > 0) { _pendingPostLoadNameReapplyDelayTicks--; return; }
if (!IsSafeForPostLoadNameWrites(out var reason))
{
    _pendingPostLoadNameReapplyDelayTicks = DeferredNameReapplyRetryDelayTicks;
    return;   // world not ready - back off and re-poll
}
_pendingPostLoadNameReapplyAttempts++;
try { CleanupOrphanedMetadata(); ReapplyAllResolvedNames(out var reapplied, out var skipped); _pendingPostLoadNameReapply = false; }
catch (System.Exception ex)
{
    if (_pendingPostLoadNameReapplyAttempts < DeferredNameReapplyMaxAttempts)
        { _pendingPostLoadNameReapplyDelayTicks = DeferredNameReapplyRetryDelayTicks; return; }
    _pendingPostLoadNameReapply = false;   // give up for this load
}
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Systems/SegmentMetadataSystem.cs#L3376-L3419` (@559e72cdb3180e3e71869643367e094a22eed988)

The gate is a plain predicate - `NameSystem` present, `GameManager` present,
`gameMode == GameMode.Game`, and **not** `isGameLoading` - each returning a named
reason so a stuck reapply is diagnosable from the log:

```csharp
if (gameManager.gameMode != GameMode.Game) { reason = "NotInGameMode"; return false; }
if (gameManager.isGameLoading)             { reason = "GameLoading";    return false; }
reason = "Ready";
return true;
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Systems/SegmentMetadataSystem.cs#L3440-L3453` (@559e72cdb3180e3e71869643367e094a22eed988)

The whole `OnUpdate` is itself gated on this same predicate, so the live stability
loop from step 4 also stops touching ECS while a load is in flight
(`SegmentMetadataSystem.cs#L83-L94`).

## Pitfalls & gotchas

- **Reapplying mid-recompute makes it worse.** If you re-stamp the moment the readback
  fails, you can be writing into a half-built aggregate that vanilla is still
  mutating, producing thrash. The `StableChecksRequired = 3` window is not
  cosmetic: an edit is only "done" after it has *held* across multiple ticks, and a
  single regression resets the counter to zero
  (`SegmentMetadataSystem.cs#L399-L413`). Do not collapse it to a single check.

- **A one-way verify misses leaked-in members.** Confirming your value survived is
  half the job; a recompute can also *pull extra edges into* the aggregate you
  edited. The verify must assert both directions - selected edges are owned, remainder
  edges are not, and the owners' edge lists do not overlap
  (`SegmentMetadataSystem.cs#L446-L508`). Skipping the reverse check leaves silent
  contamination.

- **Unbounded reapply loops spin forever.** A value that genuinely cannot be made to
  stick (topology deleted, a conflicting mod, an engine change) will fail every check.
  Both loops are explicitly capped - `MaxReapplyAttempts = 3` for the live loop,
  `DeferredNameReapplyMaxAttempts = 10` for post-load - and log a give-up warning
  rather than retrying indefinitely
  (`SegmentMetadataSystem.cs#L415-L419`, `#L3409-L3418`). Always cap.

- **Writing during deserialisation corrupts or no-ops.** Reapplying while
  `isGameLoading` is true, or before `NameSystem` exists, is unsafe. Gate every write
  behind an explicit "world ready" predicate and *defer* until it passes
  (`SegmentMetadataSystem.cs#L3423-L3453`). The initial `DeferredNameReapplyDelayTicks`
  wait exists precisely so you are not first-in after load.

- **Ordering is load-bearing.** The heal only works because the metadata system is
  scheduled `UpdateAfter<...,AggregateSystem>` - it inspects what vanilla produced.
  Register in the wrong phase or before the producer and you verify stale state
  (`Mod.cs#L56-L57`).

- **The exact number of recompute retries the game forces in practice is not visible
  in source** - the constants bound the loop but how often `AggregateSystem` actually
  re-merges after an edit is runtime behaviour: `Needs Verification (in-game)`.

## Variations

- **Cheap re-set only.** If your overridden value has no "topology" dimension (you are
  just re-stamping a scalar the engine recomputed), you never need the re-partition
  branch - keep only the `SetAuthoritativeName`-style cheap path from step 6
  (`SegmentMetadataSystem.cs#L539-L543`) inside the verify loop.

- **Protect-before instead of heal-after.** Where you can mark intended state so the
  vanilla producer skips it, the `UpdateBefore<...,AggregateSystem>` protection system
  (step 1) reduces how often the heal-after loop has to fire at all. The two compose:
  protect going in, verify-and-reapply coming out.

- **One-shot deferred reapply without the live loop.** For values that are only
  clobbered by loads (not by continuous recompute), keep just step 7 - the gated,
  capped post-load reapply - and drop the per-tick stability checks entirely.

## See also
- Related recipes: [prefab-field override](prefab-field-override.md) (the static-data
  counterpart - use it when nothing recomputes the value),
  [aggregate topology mutation](aggregate-topology-mutation.md) (how the underlying
  split/partition of ECS aggregates works).
- Explanation: [system replacement & ordering](../../explanation/system-replacement.md)
  (scheduling around vanilla producers),
  [conditional execution](../../explanation/conditional-execution.md) (the
  "world ready" gating pattern).
- Case study demonstrating it:
  [advanced-road-naming](../../case-studies/advanced-road-naming.md).

## Sources
- Canonical mods (dossier + repo):
  - `advanced-road-naming` @559e72cdb3180e3e71869643367e094a22eed988 -
    `repo/Systems/SegmentMetadataSystem.cs`, `repo/Systems/RoadAggregateProtectionSystem.cs`,
    `repo/Mod.cs`
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding
