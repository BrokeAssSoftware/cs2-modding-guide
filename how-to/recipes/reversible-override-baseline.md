---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: Reversible override: capture baseline, restore on toggle-off/OnDestroy"
recipe: reversible-override-baseline
technique_family: "AG - Reversible override: capture baseline, restore on toggle-off/OnDestroy"
diataxis: how-to
source_version: "~1.5.9 (anarchy@a6311e8; date-pinned, static source only)"
last_reverified: "2026-07-05"
status: source-verified
canonical_mods:
  - anarchy@a6311e898d20a775368668b234aaa32f06e3e1eb
  - time2work-realistic-trips@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39
  - realistic-path-finding@50645fa6a078181365e36a42e2e27b96699bf02a
technique_applicability: [core, simulation]
Created: 2026-07-05
Updated: 2026-07-05
Owners:
  - codex
---

# Reversible override: capture baseline, restore on toggle-off/OnDestroy

> Mutate vanilla/shared prefab or composition data in a way you can cleanly UNDO -
> capture the original before you write, then replay it when the feature toggles off
> or the system is destroyed.

## Problem
You are about to overwrite shared data the game authored - a prefab component, a net
composition height range, a pathfind cost. Unlike a permanent balance tweak, this
change must be **reversible**: when the user disables the feature, switches tools, or
unloads the mod, the vanilla value has to come back exactly. If you just write the new
value you have destroyed the original and have nothing to restore from; if you restore
the wrong number you leave the save permanently corrupted. This recipe is about the
**reversal lifecycle**, not the write itself.

## Solution
Separate the *mutation* from the *baseline*. Before the first write, snapshot the
vanilla value somewhere durable, then always derive your new value from that snapshot
(never from the live field, which may already be edited). To revert, replay the
snapshot. There are three proven ways to hold the baseline, each with a different
undo trigger:

- **Record component** - stash the original in a sidecar `IComponentData` on the same
  entity; a second system queries for that record and writes it back (anarchy).
- **Cache + `OnDestroy`** - hold originals in a managed dictionary, restore them all in
  `OnDestroy` when the mod unloads (time2work).
- **Value-band sentinel (no backup store)** - encode "already applied" in the value's
  own magnitude, so you can boost-once / restore-once idempotently with no separate
  baseline at all (time2work), optionally hardened with an epsilon write-suppressor so
  redundant writes never touch the graph (realistic-path-finding).

Distinct from family A (edit-a-field) and family J (one-shot multiply): here the write
is the easy half - the point is that the edit is designed to be *taken back*.

## Steps & Code

### 1. Flavour A - capture the baseline into a sidecar record component

Define a tiny `IComponentData` that just holds the original numbers:

```csharp
public struct HeightRangeRecord : IComponentData, IQueryTypeParameter
{
    public float min;
    public float max;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/Components/HeightRangeRecord.cs#L12-L23` (@a6311e898d20a775368668b234aaa32f06e3e1eb)

Before mutating, guard on the record's ABSENCE so you snapshot exactly once, then write
the record and only afterwards overwrite the live field:

```csharp
if (!EntityManager.HasComponent<HeightRangeRecord>(currentEntity))
{
    HeightRangeRecord heightRangeRecord = new ()
    {
        min = netCompositionData.m_HeightRange.min,
        max = netCompositionData.m_HeightRange.max,
    };
    buffer.AddComponent<HeightRangeRecord>(currentEntity);
    buffer.SetComponent(currentEntity, heightRangeRecord);
}
// ... later the live m_HeightRange is clamped/zeroed and written back:
netCompositionData.m_HeightRange.min = Mathf.Clamp(-1f * ... , min, max);
netCompositionData.m_HeightRange.max = Mathf.Clamp(0, min, max);
buffer.SetComponent(currentEntity, netCompositionData);
```
Source: `../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/Systems/ClearanceViolation/ModifyNetCompositionDataSystem.cs#L115-L151` (@a6311e898d20a775368668b234aaa32f06e3e1eb)

### 2. Flavour A - replay the record to restore, then self-disable

A separate system queries entities that carry the record, copies it back onto the live
component, and disables itself so the restore runs once per trigger. It starts disabled
(`Enabled = false` in `OnCreate`) and is switched on by the mutating system when the
tool is deselected or anarchy turns off:

```csharp
if (EntityManager.TryGetComponent(currentEntity, out NetCompositionData netCompositionData) &&
    EntityManager.TryGetComponent(currentEntity, out HeightRangeRecord heightRangeRecord))
{
    netCompositionData.m_HeightRange.min = heightRangeRecord.min;
    netCompositionData.m_HeightRange.max = heightRangeRecord.max;
    buffer.SetComponent(currentEntity, netCompositionData);
}
// ...
entities.Dispose();
Enabled = false;
```
Source: `../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/Systems/ClearanceViolation/ResetNetCompositionDataSystem.cs#L70-L92` (@a6311e898d20a775368668b234aaa32f06e3e1eb)

The mutating system flips it on from its "not applicable" branch (tool changed, anarchy
off, or a moveable bridge is active), then clears its own latch:

```csharp
if (m_EnsureReset)
{
    m_ResetNetCompositionDataSystem.Enabled = true;
    m_EnsureReset = false;
}
return;
```
Source: `../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/Systems/ClearanceViolation/ModifyNetCompositionDataSystem.cs#L97-L103` (@a6311e898d20a775368668b234aaa32f06e3e1eb)

### 3. Flavour B - cache originals, restore in `OnDestroy`

When the baseline lives on shared prefab data rather than a per-entity record, hold it
in a managed dictionary keyed by prefab entity. Capture on first touch, always rewrite
from the cached base (not the live value), then self-disable:

```csharp
private readonly Dictionary<Entity, Bounds1> _baseOccurrenceProbabilities = new();
// ...
var data = EntityManager.GetComponentData<HealthEventData>(prefab);
if (!_baseOccurrenceProbabilities.TryGetValue(prefab, out var baseProbability))
{
    baseProbability = data.m_OccurenceProbability;   // snapshot once
    _baseOccurrenceProbabilities[prefab] = baseProbability;
}
data.m_OccurenceProbability = new Bounds1(
    baseProbability.min / slowTimeFactor,            // derive from base, never live
    baseProbability.max / slowTimeFactor);
EntityManager.SetComponentData(prefab, data);
```
Source: `../../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Systems/HealthEventProbabilityScalerSystem.cs#L51-L69` (@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39)

Because prefab data is shared and survives the system, the undo has to run when the mod
unloads. Restore every cached baseline in `OnDestroy`:

```csharp
protected override void OnDestroy()
{
    if (_query != null && !_query.IsEmptyIgnoreFilter)
    {
        using var prefabs = _query.ToEntityArray(Allocator.Temp);
        foreach (var prefab in prefabs)
        {
            if (!_baseOccurrenceProbabilities.TryGetValue(prefab, out var baseProbability))
                continue;
            var data = EntityManager.GetComponentData<HealthEventData>(prefab);
            data.m_OccurenceProbability = baseProbability;   // put vanilla back
            EntityManager.SetComponentData(prefab, data);
        }
    }
    _baseOccurrenceProbabilities.Clear();
    base.OnDestroy();
}
```
Source: `../../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Systems/HealthEventProbabilityScalerSystem.cs#L72-L92` (@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39)

### 4. Flavour C - value-band sentinel: reversible with NO backup store

If your multiplier is large and one-directional you can skip the baseline entirely and
read "applied vs not" from the value's magnitude. Here efficiency is boosted x1000 only
while it is still in the normal band (`<= 200`), and restored by dividing only when it
looks boosted (`> 200`) - so both directions are idempotent with no stored original:

```csharp
private const int EfficiencyBoostFactor = 1000;
// ...
if (activePrefabs.Contains(prefab))
{
    if (lp.m_Efficiency <= 200)          // still vanilla -> boost once
    {
        lp.m_Efficiency *= EfficiencyBoostFactor;
        em.SetComponentData(prefab, lp);
    }
}
else
{
    if (lp.m_Efficiency > 200)           // looks boosted -> restore once
    {
        lp.m_Efficiency /= EfficiencyBoostFactor;
        em.SetComponentData(prefab, lp);
    }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Systems/SpecialEventLeisureEfficiencySystem.cs#L88-L105` (@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39)

### 5. Flavour C, hardened - snapshot component + epsilon write-suppression

Realistic Path Finding combines the record-component idea with an epsilon guard so it
never issues a redundant graph write. The baseline is a per-prefab snapshot component
captured before any multiplier is applied; every recompute derives from that snapshot:

```csharp
public struct CarTurnCostOrig : IComponentData
{
    public PathfindCosts Turning;
    public PathfindCosts UnsafeTurning;
    public PathfindCosts CurveAngle;
    public PathfindCosts UTurn;
    public PathfindCosts UnsafeUTurn;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Systems/CarPrefabTurnCostFactorSystem.cs#L13-L20` (@50645fa6a078181365e36a42e2e27b96699bf02a)

Capture-or-reuse the snapshot, recompute from it, then compare the target to the live
value with an epsilon `AlmostEqual` and `continue` if nothing meaningfully changed -
this is what suppresses redundant writes to the Burst-compiled path graph:

```csharp
if (EntityManager.HasComponent<CarTurnCostOrig>(e))
{
    orig = EntityManager.GetComponentData<CarTurnCostOrig>(e);
    sawCachedOrig = true;
}
else
{
    if (baseline) continue;                 // at x1.0 with no snapshot -> nothing to do
    orig = new CarTurnCostOrig { Turning = data.m_TurningCost, /* ... */ };
    EntityManager.AddComponentData(e, orig);
}
var turning = orig.Turning; turning.m_Value *= turnMultiplier;   // derive from snapshot
// ...
if (PathfindCostUtils.AlmostEqual(data.m_TurningCost, turning) &&
    /* ...all five costs already at target... */)
    continue;                               // epsilon write-suppression: skip the write
data.m_TurningCost = turning; /* ... */
EntityManager.SetComponentData(e, data);
```
Source: `../../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Systems/CarPrefabTurnCostFactorSystem.cs#L96-L136` (@50645fa6a078181365e36a42e2e27b96699bf02a)

The epsilon helper itself is a shared float / `PathfindCosts` compare at `1e-4`:

```csharp
internal const float DefaultEpsilon = 1e-4f;
internal static bool AlmostEqual(float a, float b, float epsilon = DefaultEpsilon)
    => math.abs(a - b) < epsilon;
```
Source: `../../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Utils/PathfindCostUtils.cs#L8-L13` (@50645fa6a078181365e36a42e2e27b96699bf02a)

Here the undo trigger is "set the multiplier back to x1.0": at baseline the recompute
`orig * 1.0` writes the snapshot value straight back, and the system only early-exits
when it is at baseline AND no snapshot exists (`baseline && !sawCachedOrig`) - so a
return-to-1.0 still restores prefabs that were previously edited.
Source: `../../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Systems/CarPrefabTurnCostFactorSystem.cs#L145-L149` (@50645fa6a078181365e36a42e2e27b96699bf02a)

## Pitfalls & gotchas

- **Always derive from the snapshot, never the live field.** Every flavour rewrites
  from the captured baseline (`baseProbability / factor`, `orig.Turning * mult`),
  because reading the already-edited live value and re-applying the factor compounds on
  each run. This is the whole reason to keep a baseline.

- **Capture BEFORE the first mutation, exactly once.** The snapshot is gated on the
  record/cache being absent (`!HasComponent<HeightRangeRecord>`,
  `!TryGetValue(...)`, `!HasComponent<CarTurnCostOrig>`). If you capture after you have
  written, you snapshot your own edit and can never get vanilla back.

- **Choose the undo trigger that matches where the data lives.** Per-entity composition
  data uses a record component replayed by a sibling system on tool-deselect (anarchy);
  shared prefab data that outlives the system must be restored in `OnDestroy`
  (time2work `HealthEventProbabilityScalerSystem`). RPF's cost systems restore on
  return-to-x1.0 rather than in `OnDestroy` - confirm which lifecycle events your
  feature actually fires and cover all of them.

- **Value-band sentinel needs a safe gap.** The `<=200 / >200` band only works because
  boosted values (`* 1000`) land far outside the vanilla range, so the sentinel is
  unambiguous. If a vanilla value could legitimately exceed the threshold, or another
  mod writes into the band, the "is it applied?" test misclassifies and you double-boost
  or fail to restore. It also cannot recover an exact original that was not a clean
  multiple - it only divides back.

- **Epsilon write-suppression avoids redundant graph writes.** RPF's `AlmostEqual(1e-4)`
  guard means a re-run that produces the same numbers issues no `SetComponentData`,
  which matters because these costs feed Burst-compiled lane/path rebuild jobs. Without
  it, every settings-applied pass would dirty every prefab even when nothing changed.

- **`OnDestroy` restore depends on the query still being valid.** time2work guards with
  `_query != null && !_query.IsEmptyIgnoreFilter` before iterating in `OnDestroy`. On
  teardown the world may be tearing down too; guard your restore or it can throw.

- Whether a given restore path actually fires on every real unload/toggle path in the
  live game (mod hot-unload, save-and-quit, tool switching edge cases) is
  `Needs Verification (in-game)` - the source shows the handlers, not that every exit
  route reaches them.

## Variations

- **Record component vs managed cache.** A sidecar `IComponentData` (anarchy) travels
  with the entity, survives system restarts, and is queryable - but it serializes into
  the save unless excluded. A managed `Dictionary` (time2work) leaves the save clean but
  is lost on domain reload, so it can only restore what it captured this session.

- **No-backup sentinel for large one-way factors.** When the multiplier is big and
  reversible by exact division, skip the baseline entirely (Flavour C, step 4). Cheapest
  option; only safe when the boosted band cannot collide with legitimate values.

- **Snapshot component + epsilon guard for graph-fed data (Flavour C, step 5).** Use the
  RPF shape when your edit feeds Burst jobs and redundant writes are expensive: cache the
  original in a component AND suppress no-op writes with an epsilon compare.

- **For a permanent, non-reversible balance edit** use the plain field override instead -
  see [prefab field override](prefab-field-override.md) - or the one-shot multiply from an
  immutable authoring baseline in [one-shot prefab multiplier](one-shot-prefab-multiplier.md).

## See also
- Related recipes: [prefab field override](prefab-field-override.md) (the non-reversible
  edit-a-field base technique), [one-shot prefab multiplier](one-shot-prefab-multiplier.md)
  (non-stacking multiply from an authoring baseline).
- Explanation: [conditional execution](../../explanation/conditional-execution.md)
  (self-disabling systems and event-driven `Enabled` toggling),
  [serialization](../../explanation/serialization.md) (which of your baseline stores
  survive a save/reload).

## Sources
- Canonical mods (dossier + repo):
  - `anarchy` @a6311e898d20a775368668b234aaa32f06e3e1eb - `repo/Anarchy/Components/HeightRangeRecord.cs`, `repo/Anarchy/Systems/ClearanceViolation/ModifyNetCompositionDataSystem.cs`, `repo/Anarchy/Systems/ClearanceViolation/ResetNetCompositionDataSystem.cs`
  - `time2work-realistic-trips` @d42921ffb1f6bbbf2c2b086bb17727bf0f637c39 - `repo/NightShift/Systems/HealthEventProbabilityScalerSystem.cs`, `repo/NightShift/Systems/SpecialEventLeisureEfficiencySystem.cs`
  - `realistic-path-finding` @50645fa6a078181365e36a42e2e27b96699bf02a - `repo/RealisticPathFinding/Systems/CarPrefabTurnCostFactorSystem.cs`, `repo/RealisticPathFinding/Utils/PathfindCostUtils.cs`
- Official/community references (link out, do not duplicate): https://cs2.paradoxwikis.com/Modding
