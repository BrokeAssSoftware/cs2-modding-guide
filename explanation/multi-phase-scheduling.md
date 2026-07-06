---
FrontmatterVersion: 1
DocumentType: Guide
Title: Multi-Phase Scheduling
Summary: Why and how a single CS2 system is registered into more than one SystemUpdatePhase - so one object handles load, gameplay, and UI contexts - and how it branches on which phase is running, explained against real mod source.
diataxis: explanation
source_version: "~1.5.10f1 (advanced-road-naming@559e72c; date-pinned, static source only)"
last_reverified: "2026-07-04"
canonical_mods:
  - advanced-road-naming@559e72cdb3180e3e71869643367e094a22eed988
  - road-speed-adjuster@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed
status: source-verified
Created: 2026-07-02
Updated: 2026-07-02
Owners:
  - codex
References:
  - Label: System Scheduling (companion concept)
    Path: ./system-scheduling.md
  - Label: Save/Load Serialization (companion concept)
    Path: ./serialization.md
  - Label: Recipe - ECS ISerializable save data / SaveVersion
    Path: ../how-to/recipes/ecs-serializable-savedata.md
  - Label: Technique Index
    Path: ../technique-index.md
---

# Multi-Phase Scheduling

[System Scheduling](./system-scheduling.md) explains how you slot a system into *one*
`SystemUpdatePhase`. This page is about the case where a single system needs to run in
*several* phases - because the state it owns has a lifecycle that spans load, gameplay, and
sometimes UI or rendering. Rather than split that state across three systems that have to
coordinate, you register **the same system** into each phase it needs and let it branch on
the context it finds itself in.

It is a concept page - the "why one object across phases" - not a step-by-step recipe.

## Why register one system into multiple phases

The scheduler lets you call `UpdateAt<T>(phase)` for the *same* `T` more than once. Each call
adds another slot where the game will tick that system. That is useful when one coherent piece
of state must participate in several parts of the frame/save lifecycle:

- restore itself during **load** (`Deserialize`),
- write itself during **save** (`Serialize`),
- do live maintenance during **gameplay** (a `Modification*` or `GameSimulation` phase),
- and possibly feed **UI** (`UIUpdate`) or **draw** (`Rendering`).

Splitting those into separate systems means they must share the payload through singletons or
lookups and stay in sync. Keeping them in one system means the payload is a plain field and
the phases are just entry points into the same object.

## A grounded example: one system across three phases

Advanced Road Naming's `SegmentMetadataSystem` owns the mod's entire persisted data structure
(per-segment route metadata and a saved-route database). It is registered into **three**
distinct phases from `OnLoad`
([`advanced-road-naming` `repo/Mod.cs#L54-L56`](../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Mod.cs), commit `559e72c`):

```csharp
updateSystem.UpdateAt<SegmentMetadataSystem>(SystemUpdatePhase.Deserialize);
updateSystem.UpdateAt<SegmentMetadataSystem>(SystemUpdatePhase.Serialize);
updateSystem.UpdateAfter<SegmentMetadataSystem, AggregateSystem>(SystemUpdatePhase.ModificationEnd);
```

Each registration drives a *different responsibility* on the one object:

- In **`Deserialize`**, the game invokes the system's `Deserialize<TReader>` hook (it
  implements `IDefaultSerializable`) to read the persisted payload back
  ([`repo/Systems/SegmentMetadataSystem.cs#L3281`](../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Systems/SegmentMetadataSystem.cs)).
- In **`Serialize`**, the game invokes `Serialize<TWriter>` to write the payload out, version
  int first ([`#L3255`](../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Systems/SegmentMetadataSystem.cs)).
  The serialization half is its own concept - see [Save/Load Serialization](./serialization.md).
- In **`ModificationEnd`**, ordered *after* the vanilla `AggregateSystem`, the system's normal
  `OnUpdate` runs live maintenance - draining a deferred post-load name reapply, validating
  aggregate stability, cleaning up orphaned metadata
  ([`#L81-L90`](../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Systems/SegmentMetadataSystem.cs)):

```csharp
protected override void OnUpdate()
{
    if (!IsSafeForLiveAggregateMaintenance())
        return;

    ProcessDeferredPostLoadNameReapply();
    UpdateAggregateStabilityChecks();
    ValidatePendingProtectedModAggregates();
    CleanupEmptyManagedAggregates();
}
```

Note the ordering constraint: the `ModificationEnd` registration uses `UpdateAfter<...,
AggregateSystem>` so the mod's maintenance runs *after* the vanilla system that could
otherwise clobber its data. Multi-phase registration and inter-system ordering are
complementary - the phase says *when in the frame*, the `UpdateBefore`/`UpdateAfter` says
*relative to which neighbour* (see [System Scheduling](./system-scheduling.md)).

## Branching on context inside OnUpdate

When a system's *own* `OnUpdate` runs in more than one phase (as opposed to the game invoking
distinct serialization hooks), it has to decide what work is valid *now*. The two levers are:

- **Gate early and return.** `SegmentMetadataSystem.OnUpdate` opens with
  `if (!IsSafeForLiveAggregateMaintenance()) return;` - it refuses to do live edits when the
  world is not in a safe state (for example mid-load), which is the same defensiveness you
  want when a system is reachable from a phase where its normal work is unsafe.
- **Defer cross-phase handoff.** Rather than doing world edits directly during `Deserialize`,
  the system *queues* a reapply (`QueuePostLoadNameReapply("Deserialize")` at
  [`#L3338`](../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Systems/SegmentMetadataSystem.cs))
  and the `ModificationEnd` `OnUpdate` drains it once the world is ready. The load phase reads
  data; the gameplay phase applies it. That hand-off *is* the multi-phase design.

A system that runs its `OnUpdate` in several phases and needs to know which one it is in can
branch on a context singleton the game sets during load - the conceptual shape is:

```csharp
protected override void OnUpdate()
{
    if (LoadInProgress())      // e.g. a deserialize-in-progress marker
    {
        RebuildCaches();       // load-time work only
        return;
    }
    TickLiveSimulation();      // normal gameplay work
}
```

`Needs Verification`: the exact singleton/marker used to detect "deserialize in progress" is
not confirmed against a pinned dossier here; the source-verified pattern is the *structure* -
gate early, do phase-appropriate work, defer cross-phase effects - as shown in
`SegmentMetadataSystem` above.

## When NOT to multi-phase a system

Multi-phase registration is for *one payload with a multi-phase lifecycle*. If your phases do
genuinely independent jobs, prefer separate systems - one per phase - which is the more common
shape. Road Speed Adjuster, for instance, spreads six *distinct* systems across five phases
(save-data in `Deserialize`, tool in `ToolUpdate`, apply in `ModificationEnd`, UI in
`UIUpdate`, render in `Rendering`)
([`road-speed-adjuster` `repo/Mod.cs#L36-L51`](../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/Mod.cs), commit `e0c0c0b`)
rather than one multi-phase system, because those responsibilities do not share a single piece
of state. Reach for multi-phase registration only when a single object genuinely owns state
that lives across phases.

## Common pitfalls

- **Doing unsafe work in the wrong phase.** A system reachable from `Deserialize` and a
  gameplay phase must gate world edits so they do not run mid-load; defer them.
- **Forgetting ordering.** Being in the right phase is not enough - order against the vanilla
  or mod system that reads/writes the same data (`UpdateAfter`/`UpdateBefore`).
- **Multi-phasing unrelated work.** If the phases do not share state, use separate systems;
  one overloaded system is harder to reason about.

## See also

- [System Scheduling](./system-scheduling.md) - the single-phase model, phases, and ordering
  verbs this page builds on.
- [Save/Load Serialization](./serialization.md) - the `Serialize`/`Deserialize` hooks that a
  multi-phase persistence system implements.
- Recipe: [ECS ISerializable save data / SaveVersion](../how-to/recipes/ecs-serializable-savedata.md).
- [Technique Index](../technique-index.md) - coverage ledger for these techniques.
