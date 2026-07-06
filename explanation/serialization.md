---
FrontmatterVersion: 1
DocumentType: Guide
Title: Save/Load Serialization
Summary: How CS2 save/load works for mods - the Serialize/Deserialize phases, Colossal's IReader/IWriter, and persisting a versioned parallel data payload inside the save file so it survives game patches - explained against real ISerializable / IDefaultSerializable mod source.
diataxis: explanation
source_version: "~1.5.10f1 (advanced-road-naming@559e72c; date-pinned, static source only)"
last_reverified: "2026-07-04"
canonical_mods:
  - advanced-road-naming@559e72cdb3180e3e71869643367e094a22eed988
  - time2work-realistic-trips@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39
status: source-verified
Created: 2026-07-02
Updated: 2026-07-02
Owners:
  - codex
References:
  - Label: Recipe - ECS ISerializable save data / SaveVersion
    Path: ../how-to/recipes/ecs-serializable-savedata.md
  - Label: System Scheduling (companion concept)
    Path: ./system-scheduling.md
  - Label: Multi-Phase Scheduling (companion concept)
    Path: ./multi-phase-scheduling.md
  - Label: Technique Index
    Path: ../technique-index.md
---

# Save/Load Serialization

A mod that holds state - a named road segment, a per-citizen daily plan, a custom parallel
data payload of any kind - eventually has to answer one question: *where does that state
live when the player saves and reloads?* There are two answers. You can write a sidecar file
next to the save, or you can embed the state **inside the save stream itself** so it travels
with the save file. This page is about the second option: how CS2's serialization layer lets
a mod persist a **versioned parallel data payload** that reloads safely across game patches.

It is a concept page - the mechanism and the *why* behind versioning - not a copy-paste
recipe. For the step-by-step, see
[recipes/ecs-serializable-savedata.md](../how-to/recipes/ecs-serializable-savedata.md).

## The serialization mechanism

CS2's save system runs your serialization code during two dedicated scheduler phases (see
[System Scheduling](./system-scheduling.md) for the phase model):

- **`SystemUpdatePhase.Serialize`** - runs while a save is being written. Your `Serialize`
  method streams your payload out.
- **`SystemUpdatePhase.Deserialize`** - runs while a save is being loaded. Your `Deserialize`
  method reads the payload back before the simulation ticks.

The plumbing is in `Colossal.Serialization.Entities`. You opt a type into the save stream by
implementing one of two interfaces:

- **`ISerializable`** on an ECS **component** (`IComponentData`) - the game serialises the
  component along with its entity. This is per-entity data.
- **`IDefaultSerializable`** on a **system** (a `GameSystemBase`) - the system writes and
  reads one blob for the whole mod, giving you a single place to persist a parallel data
  structure that is not naturally one-component-per-entity.

Both give you a `Serialize<TWriter>(TWriter writer)` / `Deserialize<TReader>(TReader reader)`
pair, where `writer` is an `IWriter` and `reader` is an `IReader`. You call
`writer.Write(value)` in order, and `reader.Read(out value)` in the **same order** to read it
back. The stream is positional; there are no field names, which is exactly why versioning
matters (below).

## Two grounded examples

### Component-level: a per-entity payload

Time2Work's `CitizenSchedule` is an `IComponentData` that also implements `ISerializable`. Its
first serialized field is a `version` int, and its `Deserialize` reads fields back in write
order inside a `try/catch` that falls back to legacy-compatible defaults on failure
([`time2work-realistic-trips` `repo/NightShift/Components/CitizenSchedule.cs#L12-L78`](../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Components/CitizenSchedule.cs), commit `d42921f`):

```csharp
public struct CitizenSchedule : IComponentData, IQueryTypeParameter, ISerializable
{
    public int version;
    public int day;
    public bool dayoff;
    // ...more fields...

    public void Serialize<TWriter>(TWriter writer) where TWriter : IWriter
    {
        writer.Write(version);
        writer.Write(day);
        writer.Write(dayoff);
        // ...write the rest in this order...
    }

    public void Deserialize<TReader>(TReader reader) where TReader : IReader
    {
        try
        {
            reader.Read(out version);
            reader.Read(out day);
            reader.Read(out dayoff);
            // ...read the rest in the SAME order...
        }
        catch
        {
            version = 2;     // fallback to legacy-compatible defaults
            work_type = 0;
        }
    }
}
```

### System-level: a versioned parallel payload

Advanced Road Naming persists a whole parallel data structure (per-segment route metadata
plus a saved-route database) from a single system, `SegmentMetadataSystem`, which implements
`IDefaultSerializable` and keeps a `SaveVersion` constant
([`advanced-road-naming` `repo/Systems/SegmentMetadataSystem.cs#L19-L21`](../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Systems/SegmentMetadataSystem.cs), commit `559e72c`):

```csharp
public sealed partial class SegmentMetadataSystem : GameSystemBase, IDefaultSerializable
{
    private const int SaveVersion = 6;
```

`Serialize` writes the version *first*, then a count, then each record's fields in a fixed
order ([`#L3255-L3275`](../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Systems/SegmentMetadataSystem.cs)):

```csharp
public void Serialize<TWriter>(TWriter writer) where TWriter : IWriter
{
    CleanupOrphanedMetadata();
    writer.Write(SaveVersion);
    writer.Write(_repository.Count);
    foreach (var metadata in _repository.All)
    {
        writer.Write(metadata.SegmentEntity);
        writer.Write(metadata.BaseNameSnapshot ?? string.Empty);
        // ...more fields, in a fixed order...
        writer.Write((int)metadata.RouteNumberPlacement);
    }
    SerializeRoutes(writer);
}
```

`Deserialize` reads the version first and **guards on it** before reading anything else,
bailing out cleanly on an unsupported version, then makes newer fields conditional on the
version they were introduced in
([`#L3281-L3340`](../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Systems/SegmentMetadataSystem.cs)):

```csharp
public void Deserialize<TReader>(TReader reader) where TReader : IReader
{
    _repository.Clear();
    var version = 0;
    reader.Read(out version);
    if (version <= 0 || version > SaveVersion)
    {
        Mod.log.Warn(() => $"Unsupported save data version {version}; metadata ignored.");
        return;                       // refuse data you cannot understand
    }
    // ...read count and each record...
    if (version >= 6)
        reader.Read(out placementValue);   // field added in v6: read only if present
    // ...
    if (version >= 2)
        DeserializeRoutes(reader, version);
    QueuePostLoadNameReapply("Deserialize");
}
```

It also implements `SetDefaults(Context context)` to reset runtime state to a clean baseline
when the serializer wants a fresh setup - for example a brand-new save with no stored payload
([`#L3343`](../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Systems/SegmentMetadataSystem.cs)).

## Why the version int is the whole point

The save stream is positional and unnamed, so the only way a *future* build can read an
*older* save is if the reader knows which layout it is looking at. Writing an explicit
version int first, and gating each later field on `if (version >= N)`, is what makes schema
evolution safe:

- **Old save, new mod.** The version is lower than current; you read only the fields that
  existed then, and supply defaults for the rest (Advanced Road Naming's `version >= 6` and
  `version >= 2` gates).
- **Corrupt or future data.** A version outside the supported range is refused rather than
  read as garbage (`version <= 0 || version > SaveVersion`). Time2Work's `try/catch` fallback
  is the component-level equivalent of the same defensiveness.
- **Never serialise what you can rebuild.** Keep the payload narrow: persist the *authoritative*
  state and rebuild derived caches and analytics after load, rather than bloating the save
  with recomputable data.

Because save/load has been actively stabilised across recent CS2 patches, a versioned,
defensively-restored payload - tolerant of missing or extra fields - is the safe posture,
especially across a fast patch cadence.

## Where restore work happens

Reading the payload back is only step one; you often need to *reapply* it once the world is
fully built. Advanced Road Naming does not write names straight from `Deserialize` - it
queues a deferred post-load reapply (`QueuePostLoadNameReapply("Deserialize")`) that its
`OnUpdate` drains once the world is safe to touch
([`#L81-L90`, `#L3338`](../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Systems/SegmentMetadataSystem.cs)).
This matters because the same system is scheduled into *three* phases - `Deserialize`,
`Serialize`, and `ModificationEnd`
([`advanced-road-naming` `repo/Mod.cs#L54-L56`](../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Mod.cs), commit `559e72c`) -
so its serialization hooks and its live-maintenance `OnUpdate` are different jobs on the same
object. That multi-phase arrangement is its own concept: see
[Multi-Phase Scheduling](./multi-phase-scheduling.md).

## Common pitfalls

- **Mismatched read/write order.** The stream is positional; reading fields in a different
  order than you wrote them corrupts everything after the first mismatch.
- **No version int.** Without a leading version stamp you can never change the schema without
  breaking every existing save.
- **Reading an unsupported version as data.** Guard the version and refuse (or fall back)
  rather than interpreting unknown bytes.
- **Serialising derived data.** Persist authoritative state only; rebuild caches after load.
- **Doing world edits directly in `Deserialize`.** The world may not be ready; defer reapply
  work to a later phase.

## See also

- Recipe: [ECS ISerializable save data / SaveVersion](../how-to/recipes/ecs-serializable-savedata.md) -
  the step-by-step implementation.
- [System Scheduling](./system-scheduling.md) - the `Serialize` / `Deserialize` phases in the
  update loop.
- [Multi-Phase Scheduling](./multi-phase-scheduling.md) - registering one system across
  Deserialize / Serialize / gameplay phases, as `SegmentMetadataSystem` does.
- [Technique Index](../technique-index.md) - coverage ledger for these techniques.
