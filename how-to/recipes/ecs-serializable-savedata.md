---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: ECS ISerializable save data / SaveVersion"
recipe: ecs-serializable-savedata
technique_family: "E - ECS ISerializable save data / SaveVersion"
diataxis: how-to
source_version: "~1.5.10f1 (advanced-road-naming@559e72c; date-pinned, static source only)"
last_reverified: "2026-07-04"
canonical_mods:
  - advanced-road-naming@559e72cdb3180e3e71869643367e094a22eed988
  - time2work-realistic-trips@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39
  - traffic-tool-essentials@10973595ac9ed37f47ca24eb788d4421dc295fa0
  - better-bulldozer@4408466f226db811159d92859479ae1e1c28ba06
  - elections-rt-module@36c25afcda67c80ce75ba68d423fe8238499435a
  - extra-detailing-tools@41df3c2b8b444197f7c3e2de4496c01cfd8bfc23
technique_applicability: [core]
status: source-verified
Created: 2026-07-01
Updated: 2026-07-05
Owners:
  - codex
---

# ECS ISerializable save data / SaveVersion

> Persist your mod's state inside the player's savegame - not a sidecar file - by
> implementing Colossal's `ISerializable` / `IDefaultSerializable` on an ECS
> component or system and writing an explicit version int first so future builds
> can read old saves.

## Problem
Your mod holds state that must survive save/reload and travel *with the save file*:
per-entity metadata (a named road segment), a per-citizen daily plan, custom traffic
phase state, or a record of removed sub-elements. External JSON (see
[json-sidecar-persistence](json-sidecar-persistence.md)) decouples that state from
the save and breaks the moment the player renames, copies, or shares a save. When the
data is intrinsically tied to entities in *this* city, you want it embedded in the
save stream itself, and you want a version stamp so a schema change next month does
not corrupt every existing save.

## Solution
Colossal's serialization layer (`Colossal.Serialization.Entities`) lets you opt an ECS
type into the save stream by implementing `ISerializable` (write/read your own bytes)
or `IDefaultSerializable` (adds `SetDefaults` for a clean new-game state). You get two
generic methods - `Serialize<TWriter>(TWriter)` and `Deserialize<TReader>(TReader)` -
and you call `writer.Write(...)` / `reader.Read(out ...)` in the **exact same field
order**. The non-negotiable discipline: **write a version number as the very first
value**, and on read, branch on it (`if (version >= N)`) so newer fields are only read
from saves that actually contain them. Two idioms coexist in the wild: a *system-level*
`SaveVersion` const guarding a repository of records, and a *component-level* `version`
field carried per entity. Empty tag components use `IEmptySerializable` - no
payload, they persist by mere presence.

## Steps & Code

### 1. Opt a component into the save stream

Add `ISerializable` to an `IComponentData` struct. Carry an explicit `version` field so
the struct is self-describing across builds:

```csharp
public struct CitizenSchedule : IComponentData, IQueryTypeParameter, ISerializable
{
    public int version;
    public int day;
    public bool dayoff;
    // ... work-hour fields ...
    public int work_type;
```
Source: `../../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Components/CitizenSchedule.cs#L12` (@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39)

### 2. Provide a default factory for new entities

New-game / newly-spawned entities need a known-good starting value. time2work stamps
the current schema version into the default so every schedule is born tagged:

```csharp
public static CitizenSchedule CreateDefault()
{
    return new CitizenSchedule
    {
        version = 2,
        day = -1000,
        // ...
        work_type = 0
    };
}
```
Source: `../../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Components/CitizenSchedule.cs#L26` (@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39)

### 3. Write version first, then fields, in a fixed order

`Serialize` writes the version, then each field once. `Deserialize` reads them back in
the identical order. Wrap the read in a try/catch so a truncated/legacy blob falls back
to a safe default instead of throwing during load:

```csharp
public void Serialize<TWriter>(TWriter writer) where TWriter : IWriter
{
    writer.Write(version);
    writer.Write(day);
    // ... same order as fields ...
    writer.Write(work_type);
}

public void Deserialize<TReader>(TReader reader) where TReader : IReader
{
    try
    {
        reader.Read(out version);
        reader.Read(out day);
        // ... same order ...
        reader.Read(out work_type);
    }
    catch
    {
        // fallback to legacy-compatible deserialization
        version = 2;
        work_type = 0; // default fallback
    }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Components/CitizenSchedule.cs#L43-L78` (@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39)

### 4. For a system-owned record store, use `IDefaultSerializable` + a `SaveVersion` const

When the state is a *collection* (a repository of segment records, a route database)
owned by a system rather than a single component, implement `IDefaultSerializable` on
the `GameSystemBase` and keep the schema number as a const:

```csharp
public sealed partial class SegmentMetadataSystem : GameSystemBase, IDefaultSerializable
{
    private const int SaveVersion = 6;
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Systems/SegmentMetadataSystem.cs#L19-L21` (@559e72c)

The system's `Serialize` writes `SaveVersion`, then a record count, then each record's
fields. Note per-field additions are appended at the end so older layouts stay readable:

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
        // ...
        writer.Write((int)metadata.RouteNumberPlacement);
    }
    SerializeRoutes(writer);
}
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Systems/SegmentMetadataSystem.cs#L3257-L3277` (@559e72c)

### 5. Guard the read: reject unknown versions, gate newer fields

`Deserialize` reads the version, rejects anything out of range (a *newer* save opened by
an *older* build is ignored, not corrupted), and gates fields added in later schemas
behind `if (version >= N)`:

```csharp
reader.Read(out version);
if (version <= 0 || version > SaveVersion)
{
    Mod.log.Warn(() => $"Unsupported Road Naming: save data version {version}; metadata ignored.");
    return;
}
// ... read count + per-record fields ...
reader.Read(out flags);
if (version >= 6)
    reader.Read(out placementValue);   // field added in SaveVersion 6
// ...
if (version >= 2)
    DeserializeRoutes(reader, version);  // route DB added in SaveVersion 2
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Systems/SegmentMetadataSystem.cs#L3281-L3340` (@559e72c)

### 6. Implement `SetDefaults` for a clean new-game state

`IDefaultSerializable` requires `SetDefaults(Context)` - called for a brand-new game (no
save blob), so it must reset every runtime buffer to empty:

```csharp
public void SetDefaults(Context context)
{
    _repository.Clear();
    _routeDatabase.Clear();
    _aggregateStabilityChecks.Clear();
    // ... clear all deferred/pending state ...
}
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Systems/SegmentMetadataSystem.cs#L3343-L3355` (@559e72c)

### 7. Persist marker-only state with `IEmptySerializable`

A tag component that carries no payload but must survive save/reload implements
`IEmptySerializable` - it persists by presence alone, no `Serialize`/`Deserialize`
body needed:

```csharp
public struct AdvancedRoadNamingManagedAggregate
    : IComponentData, IQueryTypeParameter, IEmptySerializable
{
}
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Components/AdvancedRoadNamingManagedAggregate.cs#L6` (@559e72c)
(paired member tag: `.../Components/AdvancedRoadNamingAggregateMember.cs#L6`)

## Pitfalls & gotchas

- **SaveVersion migration / back-compat is manual and asymmetric.** The write side always
  emits the *current* schema; the read side must handle *every* older schema it might
  meet. advanced-road-naming appends new fields and gates them with `if (version >= N)`
  (`SegmentMetadataSystem.cs#L3326`, `#L3335`) so old saves skip fields they never wrote.
  If you insert a field in the *middle* of the order instead of appending, every prior
  save mis-aligns and reads garbage. Always append, always version-gate.
- **A newer save in an older build must fail safe, not crash.** advanced-road-naming
  rejects `version > SaveVersion` and returns, dropping the metadata rather than
  mis-parsing it (`SegmentMetadataSystem.cs#L3290-L3294`). Decide this policy explicitly;
  the default (blind read) corrupts state.
- **Wrap risky reads in try/catch for legacy blobs.** time2work's `Deserialize` catches and
  falls back to `version = 2` defaults when an old/truncated blob is encountered
  (`CitizenSchedule.cs#L59-L77`). Without it, one malformed record aborts the whole load.
- **Field write order == field read order, exactly.** Both sides are hand-written call
  sequences (`writer.Write` / `reader.Read`); there is no schema reflection. A single
  reordered or forgotten field silently desynchronizes every subsequent field.
- **A version slot you never branch on buys you nothing.** better-bulldozer's `OwnerRecord`
  writes `1` then reads `out int version` but never uses it
  (`OwnerRecord.cs#L30-L43`) - it reserves the slot for future migrations but currently
  offers no back-compat behavior. Reserving is fine; just do not mistake it for a working
  migration path.
- **Transient vs persisted components.** Not everything should be saved. Runtime-only
  fields (frame countdowns, tool selection, highlight sets) belong on plain
  `IComponentData` with no `ISerializable` so they are recomputed each session; only
  durable player intent should enter the save stream. better-bulldozer keeps
  `DeleteInXFrames` as a plain `IComponentData` frame countdown while `OwnerRecord` /
  `PermanentlyRemovedSubElementPrefab` are `ISerializable`
  (`OwnerRecord.cs#L13`; contrasted in dossier `source.md#L77`).
- **Orphaned persisted state needs a cleanup pass at load.** Saved entity references can
  dangle if the referenced entity no longer exists. better-bulldozer runs
  `CleanUpOwnerRecordsSystem` in the `Deserialize` update phase to prune stale buffers
  before they are used (dossier `source.md#L76`, `modding.md#L95`). advanced-road-naming
  calls `CleanupOrphanedMetadata()` at the top of `Serialize`
  (`SegmentMetadataSystem.cs#L3259`).
- **Clamp deserialized values you later use as indices or bounds - a corrupt save must not
  crash the load.** A version stamp only protects field *layout*; it says nothing about whether
  a value is *sane*. traffic-tool-essentials' ring-buffer telemetry re-validates every
  index/counter after reading: `if (WriteIndex >= HISTORY_SIZE) WriteIndex = 0;`,
  `if (Count > HISTORY_SIZE) Count = HISTORY_SIZE;`, and a NaN/out-of-range guard on the
  normalized timestamp - so a truncated or garbage blob degrades to an empty history rather
  than indexing out of bounds later.
  Source: `../../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Components/VehicleStatsHistoryComponent.cs#L183-L186` (@10973595ac9ed37f47ca24eb788d4421dc295fa0)
- **Contrast with external-JSON persistence.** ISerializable data lives *inside* the save
  and moves with it (copy/share/rename all just work), but it can only reference this
  city's entities and is invisible to external tools. A JSON sidecar is portable across
  saves and human-editable but silently desyncs from the save it describes. Choose per
  data kind; see [json-sidecar-persistence](json-sidecar-persistence.md).
- `Needs Verification (in-game)`: full save/load round-trip of the SaveVersion-6 route DB
  across schema upgrades (write on new build, read on old and vice-versa) is a runtime
  claim; the dossier flags it as not verifiable from source alone
  (advanced-road-naming `index.md`, "Genuinely remaining").

## Variations

- **Sentinel-marker + schema version (traffic-tool-essentials).** Instead of always leading
  with a version int, `CustomTrafficLights` writes `uint.MaxValue` as a sentinel, then the
  schema version. On read, if the first uint is `uint.MaxValue` it reads the real version;
  otherwise it treats the blob as schema 1 (a pre-versioning legacy layout). Each later
  schema's fields are gated `if (m_SchemaVersion >= N)`, and `Deserialize` sets all field
  defaults *before* reading so absent fields keep sane values. This lets a component that
  predates its own versioning scheme still load.
  Source: `../../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Components/CustomTrafficLights.cs#L116-L196` (@10973595ac9ed37f47ca24eb788d4421dc295fa0)
  (schema field decl `#L29`; struct decl `#L6`)
- **Fixed-size ring-buffer telemetry inside an `ISerializable` component (traffic-tool-essentials).**
  When a component holds rolling time-series (24h of vehicle counts, 48 samples) you can keep the whole
  history *in the component* using `FixedList128Bytes<ushort>` rings rather than a side buffer, so it
  travels in the save with no extra archetype. `AddSamples` clamps each value to `ushort` range and wraps
  via `WriteIndex = (WriteIndex + 1) % HISTORY_SIZE`; `Serialize` writes the schema ushort first, then the
  timing state, then each ring length-prefixed and per-element; `Deserialize` mirrors that and caps the
  replayed element count at `HISTORY_SIZE` (`for (int i = 0; i < len && i < HISTORY_SIZE; i++)`) before the
  corruption-clamp pass above. A companion empty buffer (`VehicleStatsHistoryMarker`) is added to the entity
  purely to stabilize the archetype for the serializer.
  Source: `../../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Components/VehicleStatsHistoryComponent.cs#L58-L92` (@10973595ac9ed37f47ca24eb788d4421dc295fa0)
  (rings decl `#L27-L31`; serialize/deserialize `#L147-L208`; schema const `#L20`)
- **Multi-component schema versioning across one mod (traffic-tool-essentials).** A single mod can carry
  several independently-versioned components. Alongside `CustomTrafficLights` (schema 9), `DepotZoneData`
  holds its own `m_SchemaVersion` and writes `(ushort)3`, gating later fields `if (m_SchemaVersion >= 2)`
  and `if (m_SchemaVersion >= 3)`. Each component owns its schema counter; do not share one version int
  across unrelated components or a bump to one silently invalidates the others.
  Source: `../../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Components/DepotZoneData.cs#L220-L266` (@10973595ac9ed37f47ca24eb788d4421dc295fa0)
  (schema field decl `#L25`; struct decl `#L23`)
- **Split write-version from read-layout-version + a legacy-alias table (elections-rt-module).** When the
  layout churns faster than you want to expose a "current" marker, decouple the two: `ElectionState` writes
  a stable `CurrentVersion = 24`, but reads route through `CurrentSerializedLayoutVersion = 32`. `Serialize`
  emits `CurrentVersion`; `Deserialize` reads the raw `serializedVersion` and maps it with
  `GetSerializedLayoutVersion(...)` - which aliases published markers to real layouts (`1 -> 16`,
  `23 -> 31`, `24 -> 32`) and clamps anything higher - then gates every field on the resulting
  `layoutVersion`. This also lets you *collapse* removed fields: at `layoutVersion >= 21` two legacy
  per-candidate percents are read and folded into one `cashAssistanceTurnoutBonusPercent` (max of the two)
  so old saves migrate forward without a schema break.
  Source: `../../../vice-and-order-research/mods/dossiers/elections-rt-module/repo/Elections/Components/ElectionState.cs#L2873-L2887` (@36c25afcda67c80ce75ba68d423fe8238499435a)
  (const decls `#L20-L21`; write `#L1666`; read map `#L1994-L1996`; legacy-alias collapse `#L2460-L2469`)
- **Sparse storage: the `ISerializable` component exists only when non-default (extra-detailing-tools).**
  Rather than tag every object with a scale component and persist mostly-identity data, `TransformObject`
  (a one-field `float3 m_Scale` with a trivial `Serialize`/`Deserialize`) is *added* to an entity only when
  the player sets a non-unit scale and *removed* the moment scale returns to `(1,1,1)`:
  `SetScale` calls `EntityManager.RemoveComponent<TransformObject>(...)` on reset and
  `EntityManager.AddComponent<TransformObject>(...)` when first scaled; `GetScale` returns `float3(1,1,1)`
  when the component is absent. Presence *is* the "non-default" flag, so untouched objects add zero bytes
  to the save and the deserializer never sees identity rows.
  Source: `../../../vice-and-order-research/mods/dossiers/extra-detailing-tools/repo/MOD/Systems/UI/TransformSection.cs#L357-L378` (@41df3c2b8b444197f7c3e2de4496c01cfd8bfc23)
  (component + Serialize/Deserialize `MOD/Prefabs/TransformObject.cs#L8-L32`; absent-defaults-to-unit `#L349`)
- **Minimal per-component version int (better-bulldozer).** For tiny records, write a bare
  version int and one payload field; keep the migration hook available even if unused.
  Source: `../../../vice-and-order-research/mods/dossiers/better-bulldozer/repo/BetterBulldozer/Components/OwnerRecord.cs#L13-L44` (@4408466f226db811159d92859479ae1e1c28ba06)
- **System-level `IDefaultSerializable` vs component-level `ISerializable`.** Use the system
  variant when one system owns a *collection* of records (advanced-road-naming); use the
  component variant when the state is naturally per-entity (time2work, traffic-tool,
  better-bulldozer). `IDefaultSerializable` adds `SetDefaults`; plain `ISerializable` does
  not (you seed defaults via a factory like `CreateDefault()` instead).
- **External JSON instead.** When state must be portable across saves or edited outside the
  game, skip the save stream entirely - see [json-sidecar-persistence](json-sidecar-persistence.md).

## See also
- Related recipes: [json-sidecar-persistence](json-sidecar-persistence.md) (the external-file counterpart)
- Coverage ledger: [technique-index](../../technique-index.md) (family E)
- Case studies demonstrating it: [advanced-road-naming](../../case-studies/advanced-road-naming.md), [time2work-realistic-trips](../../case-studies/time2work-realistic-trips.md)

## Sources
- Canonical mods (dossier + repo):
  - `advanced-road-naming` @559e72c - `Systems/SegmentMetadataSystem.cs` (IDefaultSerializable, SaveVersion 6), `Components/AdvancedRoadNamingManagedAggregate.cs`, `Components/AdvancedRoadNamingAggregateMember.cs` (IEmptySerializable tags)
  - `time2work-realistic-trips` @d42921ffb1f6bbbf2c2b086bb17727bf0f637c39 - `NightShift/Components/CitizenSchedule.cs` (ISerializable component, in-struct version + CreateDefault + try/catch fallback)
  - `traffic-tool-essentials` @10973595ac9ed37f47ca24eb788d4421dc295fa0 - `TrafficToolEssentials/Components/CustomTrafficLights.cs` (sentinel-marker + m_SchemaVersion 9); `.../VehicleStatsHistoryComponent.cs` (FixedList128 ring-buffer telemetry + corruption-clamp on read); `.../DepotZoneData.cs` (per-component m_SchemaVersion 3)
  - `better-bulldozer` @4408466f226db811159d92859479ae1e1c28ba06 - `BetterBulldozer/Components/OwnerRecord.cs` (minimal version-int record)
  - `elections-rt-module` @36c25afcda67c80ce75ba68d423fe8238499435a - `Elections/Components/ElectionState.cs` (dual CurrentVersion 24 written vs CurrentSerializedLayoutVersion 32 read, GetSerializedLayoutVersion alias table + legacy field collapse)
  - `extra-detailing-tools` @41df3c2b8b444197f7c3e2de4496c01cfd8bfc23 - `MOD/Prefabs/TransformObject.cs` + `MOD/Systems/UI/TransformSection.cs` (sparse ISerializable: component added only when scale != unit, RemoveComponent on reset)
- Official/community references (link out, do not duplicate): Colossal `IWriter`/`IReader`/`ISerializable` live in `Colossal.Serialization.Entities`; see the CS2 modding wiki - https://cs2.paradoxwikis.com/Modding_Wiki
