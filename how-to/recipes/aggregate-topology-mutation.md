---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: Runtime entity/aggregate topology mutation"
recipe: aggregate-topology-mutation
technique_family: "AD - Runtime entity/aggregate topology mutation"
diataxis: how-to
source_version: "~1.5.10f1 (advanced-road-naming@559e72c; date-pinned, static source only)"
last_reverified: "2026-07-05"
status: source-verified
canonical_mods:
  - advanced-road-naming@559e72cdb3180e3e71869643367e094a22eed988
technique_applicability: [infrastructure]
Created: 2026-07-05
Updated: 2026-07-05
Owners:
  - codex
---

# Runtime entity/aggregate topology mutation

> Take ownership of a subset of a live ECS aggregate (a road street, a route) by
> cloning the aggregate **entity** - not a prefab - and re-homing the child edges'
> membership onto your clone, so you can name/own that subset without disturbing the
> remainder.

## Problem
A vanilla "aggregate" is a runtime ECS entity that groups many child edges under one
owner - the thing that carries a street's single visible name over all its segments.
You want to affect **only some** of those edges (rename a stretch, tag a section) but
the aggregate owns all of them as one unit. You cannot reach this by editing a prefab
(there is no prefab template for a specific runtime street) and you must not rename the
whole street. You need to split the live aggregate: carve the selected edges into a new
owner you control, leave the rest on the original, and do it without fighting the
vanilla `AggregateSystem` that constantly rebuilds these groups.

## Solution
Duplicate the aggregate **entity** with `EntityManager.Instantiate` (this copies its
component makeup and label plumbing), clear the clone's `AggregateElement` buffer, then
re-home the selected edges by rewriting each edge's `Aggregated.m_Aggregate` to point at
your clone and adding it to the clone's buffer. Tag the clone with your own marker
component so you can find and reap your clones later. The subtle part is the refresh
signalling: you mark child edges `BatchesUpdated` (render-batch refresh) **without**
`Updated`, so labels/render refresh but you do **not** feed the vanilla
`AggregateSystem` an `Updated` edge that would trigger it to re-merge your split away.
Finally, sweep empty clones every tick so no orphan owners accumulate.

## Steps & Code

### 1. Declare a marker component and a query for your managed clones

Add a tag `IComponentData` so every aggregate you mint is identifiable and reapable.
Advanced Road Naming defines `AdvancedRoadNamingManagedAggregate` and builds a query
for `(marker + AggregateElement)` in `OnCreate`:

```csharp
public struct AdvancedRoadNamingManagedAggregate : IComponentData, IQueryTypeParameter, IEmptySerializable
{
}
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Components/AdvancedRoadNamingManagedAggregate.cs#L6` (@559e72cdb3180e3e71869643367e094a22eed988)

```csharp
_managedAggregateQuery = GetEntityQuery(
    ComponentType.ReadOnly<AdvancedRoadNamingManagedAggregate>(),
    ComponentType.ReadOnly<AggregateElement>());
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Systems/SegmentMetadataSystem.cs#L72` (@559e72cdb3180e3e71869643367e094a22eed988)

### 2. Clone the aggregate ENTITY (not a prefab) and empty its member buffer

`EntityManager.Instantiate(sourceAggregate)` duplicates the runtime aggregate entity
with all its label-bearing components. Clear the copied `AggregateElement` buffer so the
clone starts empty, stamp your marker, and tag it for refresh:

```csharp
var clone = EntityManager.Instantiate(sourceAggregate);
if (clone == Entity.Null || !EntityManager.Exists(clone) || !EntityManager.HasBuffer<AggregateElement>(clone))
{
    Mod.log.Warn(() => $"Road Naming: aggregate clone failed. ...");
    return Entity.Null;
}

EntityManager.GetBuffer<AggregateElement>(clone).Clear();
MarkManagedAggregate(clone);            // AddComponent<AdvancedRoadNamingManagedAggregate>
InvalidateAggregateLabelState(clone);
EnsureRefreshTag<Updated>(clone);       // clone is a NEW owner -> full rebuild is wanted here
EnsureRefreshTag<BatchesUpdated>(clone);
return clone;
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Systems/SegmentMetadataSystem.cs#L600-L613` (@559e72cdb3180e3e71869643367e094a22eed988)

The marker is applied idempotently (only if absent):

```csharp
private void MarkManagedAggregate(Entity aggregate)
{
    if (aggregate != Entity.Null && EntityManager.Exists(aggregate)
        && !EntityManager.HasComponent<AdvancedRoadNamingManagedAggregate>(aggregate))
        EntityManager.AddComponent<AdvancedRoadNamingManagedAggregate>(aggregate);
}
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Systems/SegmentMetadataSystem.cs#L780-L784` (@559e72cdb3180e3e71869643367e094a22eed988)

### 3. Re-home the selected edges onto the new owner

Membership is two-sided: the aggregate lists edges in its `AggregateElement` buffer, and
each edge points back through `Aggregated.m_Aggregate`. Rewrite **both** - clear the
clone's buffer, add the selected edges, and set each edge's back-pointer to the clone:

```csharp
var buffer = EntityManager.GetBuffer<AggregateElement>(aggregate);
buffer.Clear();
for (var i = 0; i < edges.Count; i++)
{
    var edge = edges[i];
    if (!_validation.IsValidRoadSegment(edge)) continue;

    buffer.Add(new AggregateElement { m_Edge = edge });
    if (EntityManager.HasComponent<Aggregated>(edge))
        EntityManager.SetComponentData(edge, new Aggregated { m_Aggregate = aggregate });
    else
        EntityManager.AddComponentData(edge, new Aggregated { m_Aggregate = aggregate });

    EnsureRefreshTag<BatchesUpdated>(edge);   // NOTE: no Updated on the edge - see step 4
}
InvalidateAggregateLabelState(aggregate);
EnsureRefreshTag<Updated>(aggregate);
EnsureRefreshTag<BatchesUpdated>(aggregate);
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Systems/SegmentMetadataSystem.cs#L757-L778` (@559e72cdb3180e3e71869643367e094a22eed988)

### 4. THE key idiom: tag child edges `BatchesUpdated` WITHOUT `Updated`

This is what makes topology mutation survive. `BatchesUpdated` forces the render/label
batches to refresh; `Updated` on an **edge** is the signal the vanilla `AggregateSystem`
consumes to re-derive aggregate membership. If you set `Updated` on the re-homed edges,
vanilla will look at the edge, decide it belongs to the original street, and **undo your
split** on the next pass. So refresh the child edges with `BatchesUpdated` only. The mod
even documents this in a comment where it re-labels child edges under an owner:

```csharp
ClearChildEdgeCustomName(nameEntity, edge);
// BatchesUpdated refreshes label/render batches without feeding AggregateSystem an Updated edge.
EnsureRefreshTag<BatchesUpdated>(edge);
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Systems/SegmentMetadataSystem.cs#L1671-L1673` (@559e72cdb3180e3e71869643367e094a22eed988)

`EnsureRefreshTag<T>` is just an idempotent add-if-absent - so re-running the mutation
never stacks duplicate tags:

```csharp
private void EnsureRefreshTag<T>(Entity segment) where T : struct, IComponentData
{
    if (!EntityManager.HasComponent<T>(segment))
        EntityManager.AddComponent<T>(segment);
}
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Systems/SegmentMetadataSystem.cs#L3848-L3852` (@559e72cdb3180e3e71869643367e094a22eed988)

### 5. Orchestrate the split: remainder stays on source, selected go to clones

`TryPartitionAggregateForSelectedEdges` ties it together. It bails unless the selection
is a **strict subset** (both `selectedEdges` and `remainderEdges` non-empty), splits each
side into connected components, keeps the first remainder component on the original
`sourceAggregate` and clones for the rest, then clones fresh owners for every selected
component:

```csharp
if (selectedEdges.Count == 0 || remainderEdges.Count == 0)
    return false;   // nothing to split - all-or-none selection is not this technique

// remainder: first component reuses sourceAggregate, extra components get clones
var remainderAggregate = sourceAssigned ? CreateAggregateClone(sourceAggregate, ...) : sourceAggregate;
AssignAggregateEdges(remainderAggregate, remainderComponents[i], operation, "Remainder");

// selected: every connected component becomes a fresh owner you name
var selectedAggregate = CreateAggregateClone(sourceAggregate, operation, "Selected");
AssignAggregateEdges(selectedAggregate, selectedComponents[i], operation, "Selected");
SetAuthoritativeName(selectedAggregate, selectedFinalName, ...);
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Systems/SegmentMetadataSystem.cs#L251-L288` (@559e72cdb3180e3e71869643367e094a22eed988)

Splitting per connected component matters: after re-homing, a selected set that is not
physically contiguous must become **separate** owners, or one owner would span a gap.

### 6. Reap empty clones every tick

Re-homing can leave an aggregate with zero edges (e.g. all its edges got pulled onto a
clone). Sweep them: query your marked aggregates, destroy any whose `AggregateElement`
buffer is empty. Advanced Road Naming runs this from `OnUpdate` every frame it is safe:

```csharp
protected override void OnUpdate()
{
    if (!IsSafeForLiveAggregateMaintenance()) return;
    ProcessDeferredPostLoadNameReapply();
    UpdateAggregateStabilityChecks();
    ValidatePendingProtectedModAggregates();
    CleanupEmptyManagedAggregates();
}
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Systems/SegmentMetadataSystem.cs#L81-L90` (@559e72cdb3180e3e71869643367e094a22eed988)

```csharp
var aggregates = _managedAggregateQuery.ToEntityArray(Allocator.Temp);
for (var i = 0; i < aggregates.Length; i++)
{
    var aggregate = aggregates[i];
    if (aggregate == Entity.Null || !EntityManager.Exists(aggregate)
        || !EntityManager.HasBuffer<AggregateElement>(aggregate)) continue;
    if (EntityManager.GetBuffer<AggregateElement>(aggregate, true).Length != 0) continue;
    EntityManager.DestroyEntity(aggregate);   // only destroys OUR marked, empty owners
}
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Systems/SegmentMetadataSystem.cs#L1122-L1135` (@559e72cdb3180e3e71869643367e094a22eed988)

Because the query is scoped to your marker component, this cleanup can only ever destroy
aggregates you minted - never a vanilla street.

## Pitfalls & gotchas

- **`Updated` on a re-homed edge lets vanilla re-merge your split.** This is the whole
  reason the technique exists. The vanilla `AggregateSystem` treats an edge's `Updated`
  tag as "recompute my aggregate membership," which will pull the edge back onto the
  street it geometrically belongs to and erase your clone's ownership. Refresh child
  edges with `BatchesUpdated` **only**
  (`Systems/SegmentMetadataSystem.cs#L1671-L1673`). Note the asymmetry: the mod *does*
  set `Updated` on the clone **aggregate** itself (`#L610`, `#L776`) - a new owner needs
  a full rebuild - but never on the edges.

- **Both sides of membership must agree.** An edge belongs to an aggregate via two links:
  the aggregate's `AggregateElement` buffer entry **and** the edge's
  `Aggregated.m_Aggregate` back-pointer. Rewrite both (`AssignAggregateEdges`,
  `#L757-L769`). Update only one and the topology is inconsistent - vanilla and your code
  will disagree about who owns the edge.

- **All-or-nothing selection is not a split.** If the selection covers every edge of the
  aggregate (or none), `TryPartitionAggregateForSelectedEdges` returns `false` and does
  nothing (`#L251-L252`). A whole-street rename is a different, simpler path (rename the
  existing owner in place); this technique is specifically for **partial** ownership.

- **Non-contiguous selections need one owner per connected component.** The mod runs
  `BuildConnectedEdgeComponents` on each side and clones a separate owner per component
  (`#L254-L288`). Assigning a spatially broken set to a single aggregate would make one
  owner span disconnected road, which vanilla labelling does not expect.

- **Orphan owners accumulate without a sweep.** Re-homing routinely empties an aggregate.
  Without the per-tick `CleanupEmptyManagedAggregates` sweep (`#L1117`, called at
  `#L89`), empty marked owners pile up. Keep the cleanup query scoped to your marker so
  it can never destroy a vanilla aggregate.

- **Vanilla's re-merge timing / persistence across save-load is behavioural.** Exactly
  *when* `AggregateSystem` would re-merge, and whether a cloned-and-re-homed aggregate
  survives a save/reload round-trip, are not provable from this source alone. The mod
  carries a `RegisterAggregateStabilityCheck` / `UpdateAggregateStabilityChecks` guard
  that reapplies the split if it detects drift (`#L290-L291`), implying the split can be
  disturbed - but the trigger is `Needs Verification (in-game)`.

## Variations

- **Recombine instead of split.** The inverse operation - merge several compatible
  aggregates into one - reuses the same primitives in reverse: pick a `targetAggregate`,
  `AssignAggregateEdges` all the group's edges onto it, name it, then
  `ClearMergedAggregateOwner` (empty the buffer + re-mark + refresh) on the donors so the
  tick sweep reaps them:

  ```csharp
  var targetAggregate = owners[0];
  AssignAggregateEdges(targetAggregate, group.Segments, operation, "Combined");
  SetAuthoritativeName(targetAggregate, finalName, ...);
  for (var i = 1; i < owners.Count; i++)
      ClearMergedAggregateOwner(owners[i], operation);   // empties donor -> reaped next tick
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Systems/SegmentMetadataSystem.cs#L1075-L1080` (@559e72cdb3180e3e71869643367e094a22eed988)

- **Verify ownership after mutating.** Rather than trust the write, re-read each edge's
  authoritative owner and confirm it lands on the expected clone
  (`VerifyPartitionOwnership`, `#L299-L330`); return `false` and warn if any edge's owner
  or name is wrong. Pair this with the verify-and-reapply pattern for drift resilience.

## See also
- Related recipes: [prefab-clone-synthesis](prefab-clone-synthesis.md) (family AC -
  cloning **prefab templates**, the distinct sibling technique: AC clones a shared
  authoring template, AD clones a live runtime aggregate entity and re-homes membership);
  [verify-and-reapply-override](verify-and-reapply-override.md) (guarding a mutation
  against vanilla drift).
- Reference: [ECS fundamentals](../../explanation/ecs-fundamentals.md) (entities,
  components, buffers, and how systems consume tags like `Updated`).
- Case studies demonstrating it:
  [advanced-road-naming](../../case-studies/advanced-road-naming.md).

## Sources
- Canonical mods (dossier + repo):
  - `advanced-road-naming` @559e72cdb3180e3e71869643367e094a22eed988 -
    `repo/Systems/SegmentMetadataSystem.cs`, `repo/Components/AdvancedRoadNamingManagedAggregate.cs`
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding
