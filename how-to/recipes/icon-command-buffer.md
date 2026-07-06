---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: Notification-icon management via IconCommandSystem / IconCommandBuffer"
recipe: icon-command-buffer
technique_family: "AM - Notification-icon management via IconCommandSystem / IconCommandBuffer"
diataxis: how-to
source_version: "~1.6.0f1 (magic-garbage-truck@1b6a478; date-pinned, static source only)"
last_reverified: "2026-07-05"
status: source-verified
canonical_mods:
  - magic-garbage-truck@1b6a478753e1ef4e43ac9b90d567f3d7183c7be2
technique_applicability: [simulation, ui]
Created: 2026-07-05
Updated: 2026-07-05
Owners:
  - codex
---

# Notification-icon management via IconCommandSystem / IconCommandBuffer

> Add or remove the little building notification icons (the "garbage piling up"
> exclamation, etc.) from inside a Burst `IJobChunk` by queueing commands on an
> `IconCommandBuffer` and completing the producer -> writer -> dependency handshake.

## Problem
You are mutating an entity's simulation state from a scheduled job - say you zero a
building's garbage, or fix its power, or clear a hazard - and the little floating
warning icon over the building needs to appear or disappear to match. You cannot
structurally add or delete the icon entity yourself from a parallel Burst job, and you
cannot touch managed UI from Burst. CS2 exposes a dedicated barrier system,
`IconCommandSystem`, that plays exactly the role an `EntityCommandBuffer` barrier plays
for structural changes - but for notification icons. This recipe shows the full,
easy-to-get-wrong handshake to drive it from a job.

## Solution
Grab the managed `IconCommandSystem` once in `OnCreate`. Each update, ask it for a fresh
`IconCommandBuffer` (`CreateCommandBuffer()`), pass that buffer by value into your
`[BurstCompile] IJobChunk`, and call `m_IconCommandBuffer.Remove(building, iconPrefab)`
(or the `Add` counterpart) per entity. The buffer is a deferred command queue, so the
crux is the three-line handshake **after** you schedule: tell the icon system your job
will write to its buffer with `AddCommandBufferWriter(jobHandle)`, then set
`Dependency = jobHandle` so downstream systems wait on you. Skip either half and you get
a race or a dropped command. Gate the whole thing behind a cheap early-return so an
inactive feature costs nothing.

## Steps & Code

### 1. Resolve `IconCommandSystem` in `OnCreate`

Get the managed barrier system once and cache it. This is the icon-world analogue of
grabbing an `EndSimulationEntityCommandBufferSystem`.

```csharp
private IconCommandSystem m_IconCommandSystem = null!;

protected override void OnCreate()
{
    base.OnCreate();
    m_IconCommandSystem = World.GetOrCreateSystemManaged<IconCommandSystem>();
    // ... build queries (GarbageProducer, GarbageParameterData) ...
}
```
Source: `../../../vice-and-order-research/mods/dossiers/magic-garbage-truck/repo/Systems/TotalMagicSystem.cs#L37,L46-L50` (@1b6a478753e1ef4e43ac9b90d567f3d7183c7be2)

The `IconCommandBuffer` / `IconCommandSystem` types live in the `Game.Notifications`
namespace (`using Game.Notifications;`), and the icon target here is a `GarbageProducer`
from `Game.Buildings`.
Source: `../../../vice-and-order-research/mods/dossiers/magic-garbage-truck/repo/Systems/TotalMagicSystem.cs#L20-L22` (@1b6a478753e1ef4e43ac9b90d567f3d7183c7be2)

### 2. Cheap early-return before you allocate a buffer

Do not create a command buffer for a feature that is switched off - bail first.

```csharp
protected override void OnUpdate()
{
    if (!Mod.TryGetSetting(out Setting setting))
    {
        return;
    }

    if (!setting.TotalMagic || setting.TrashBossEnabled)
    {
        return;
    }
    // ... only now create the buffer ...
}
```
Source: `../../../vice-and-order-research/mods/dossiers/magic-garbage-truck/repo/Systems/TotalMagicSystem.cs#L79-L89` (@1b6a478753e1ef4e43ac9b90d567f3d7183c7be2)

### 3. Create the buffer and hand it to a Burst `IJobChunk`

`CreateCommandBuffer()` returns a fresh `IconCommandBuffer` value. Pass it into the job
by value alongside the parameter data your icon prefab lookup needs, and schedule.

```csharp
IconCommandBuffer iconBuffer = m_IconCommandSystem.CreateCommandBuffer();

GarbageParameterData garbageParams =
    m_GarbageParamsQuery.GetSingleton<GarbageParameterData>();

JobHandle cleanHandle = new MagicJob
{
    m_EntityType = GetEntityTypeHandle(),
    m_GarbageProducerType = GetComponentTypeHandle<GarbageProducer>(false),
    m_IconCommandBuffer = iconBuffer,
    m_GarbageParameters = garbageParams
}.ScheduleParallel(m_GarbageProducerQuery, Dependency);
```
Source: `../../../vice-and-order-research/mods/dossiers/magic-garbage-truck/repo/Systems/TotalMagicSystem.cs#L91-L102` (@1b6a478753e1ef4e43ac9b90d567f3d7183c7be2)

### 4. The producer -> writer -> dependency handshake (the crux)

This is the part that is easy to get wrong. After scheduling, register your job as a
**writer** to the icon system's buffer, then publish your handle as the system
`Dependency`. Both lines are required: `AddCommandBufferWriter` makes the icon system
wait for your job before it plays back the buffer; `Dependency = cleanHandle` makes
every other consumer of your data wait for you.

```csharp
m_IconCommandSystem.AddCommandBufferWriter(cleanHandle);
Dependency = cleanHandle;
```
Source: `../../../vice-and-order-research/mods/dossiers/magic-garbage-truck/repo/Systems/TotalMagicSystem.cs#L104-L105` (@1b6a478753e1ef4e43ac9b90d567f3d7183c7be2)

### 5. Queue the icon command inside the job

The buffer is a plain field on the `[BurstCompile] IJobChunk` - no `[NativeContainer]`
attributes, no `EntityCommandBuffer.ParallelWriter` index. Call `Remove` (or `Add`) with
the target entity and the icon **prefab** entity. Here the prefab comes from
`GarbageParameterData.m_GarbageNotificationPrefab`.

```csharp
[BurstCompile]
private struct MagicJob : IJobChunk
{
    [ReadOnly] public EntityTypeHandle m_EntityType;
    public ComponentTypeHandle<GarbageProducer> m_GarbageProducerType;
    public IconCommandBuffer m_IconCommandBuffer;
    [ReadOnly] public GarbageParameterData m_GarbageParameters;
    // ... Execute removes m_GarbageNotificationPrefab from each building ...
}
```
Source: `../../../vice-and-order-research/mods/dossiers/magic-garbage-truck/repo/Systems/TotalMagicSystem.cs#L124-L132` (@1b6a478753e1ef4e43ac9b90d567f3d7183c7be2)

### 6. Idempotent removal - remove the icon even when already clean

The job removes the icon in **both** branches. The fast path handles an entity that is
already clean (zero garbage, no warning flag, no pending collection) but might still be
carrying a stale icon from a previous frame - so it removes the icon and `continue`s
without touching the component. This makes the sweep safe to run every interval.

```csharp
// Fast path: already clean (still remove icon idempotently).
if (producer.m_Garbage == 0 &&
    (producer.m_Flags & GarbageProducerFlags.GarbagePilingUpWarning) == 0 &&
    producer.m_CollectionRequest == Entity.Null)
{
    m_IconCommandBuffer.Remove(building, m_GarbageParameters.m_GarbageNotificationPrefab);
    continue;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/magic-garbage-truck/repo/Systems/TotalMagicSystem.cs#L144-L151` (@1b6a478753e1ef4e43ac9b90d567f3d7183c7be2)

### 7. Clear the source flag, then remove the icon

For the "dirty" branch, zero the simulation state and clear the
`GarbagePilingUpWarning` flag on the component **before** queueing the icon removal, so
the state that spawns the icon and the icon itself stay consistent. Write the mutated
component back, then remove.

```csharp
producer.m_Garbage = 0;
producer.m_CollectionRequest = Entity.Null;
producer.m_DispatchIndex = 0;
producer.m_Flags &= ~GarbageProducerFlags.GarbagePilingUpWarning;

producers[i] = producer;

m_IconCommandBuffer.Remove(building, m_GarbageParameters.m_GarbageNotificationPrefab);
```
Source: `../../../vice-and-order-research/mods/dossiers/magic-garbage-truck/repo/Systems/TotalMagicSystem.cs#L153-L160` (@1b6a478753e1ef4e43ac9b90d567f3d7183c7be2)

## Pitfalls & gotchas

- **Forgetting `AddCommandBufferWriter` is a silent race.** `IconCommandSystem` is a
  barrier: it plays the buffer back on its own schedule. If you set `Dependency` but
  never call `AddCommandBufferWriter(cleanHandle)`, the icon system does not know it must
  wait for your job, and it can read/playback the buffer while your job is still writing
  to it. Both `AddCommandBufferWriter(cleanHandle)` **and** `Dependency = cleanHandle`
  are required, in that order
  (`../../../vice-and-order-research/mods/dossiers/magic-garbage-truck/repo/Systems/TotalMagicSystem.cs#L104-L105`).

- **The second argument is a PREFAB entity, not the building.** `Remove(building,
  iconPrefab)` takes the target entity first and the **notification prefab** second. This
  mod reads the prefab from the `GarbageParameterData` singleton
  (`m_GarbageNotificationPrefab`), fetched via `GetSingleton<GarbageParameterData>()`
  and required with `RequireForUpdate`, so passing the wrong entity (e.g. the building
  twice) removes nothing
  (`../../../vice-and-order-research/mods/dossiers/magic-garbage-truck/repo/Systems/TotalMagicSystem.cs#L69,L93-L94,L149`).

- **Remove the icon idempotently, or ghosts persist.** Zeroing `m_Garbage` and clearing
  `GarbagePilingUpWarning` does not itself delete an already-spawned icon; the icon is a
  separate entity managed by the notification system. You must issue the `Remove`
  command. This mod removes on the already-clean fast path too, so a stale icon left by a
  prior frame is still cleaned up
  (`../../../vice-and-order-research/mods/dossiers/magic-garbage-truck/repo/Systems/TotalMagicSystem.cs#L144-L151`).

- **Do not allocate the buffer before the early-return.** `CreateCommandBuffer()` is
  work; call it only after the feature-active gate so a disabled feature costs nothing
  (`../../../vice-and-order-research/mods/dossiers/magic-garbage-truck/repo/Systems/TotalMagicSystem.cs#L79-L91`).

- **The `Add` counterpart is not exercised here.** This mod only ever calls `Remove`.
  The exact `Add` overload signature and the visual result of an add-then-barrier-flush
  are **Needs Verification (in-game)** from this source - do not assume symmetry blindly.

- **When the icon system actually flushes.** The precise phase at which
  `IconCommandSystem` plays back the buffer (and thus when the icon visibly
  appears/disappears relative to your write) is not observable from this mod's source and
  is **Needs Verification (in-game)**.

## Variations

- **Add instead of remove.** To raise an icon rather than clear one, call the buffer's
  `Add` counterpart in the job body; the surrounding handshake (steps 1, 3, 4) is
  identical. Confirm the exact overload against `Game.Notifications.IconCommandBuffer`
  for your game version - `Needs Verification (in-game)` from this source.

- **Different notification target.** Any component that owns a notification icon works
  the same way: swap `GarbageProducer` for your target component and source the icon
  prefab from that system's parameter singleton, keeping the producer -> writer ->
  dependency handshake unchanged.

- **Single-threaded job.** The same `IconCommandBuffer` field works in a non-parallel
  `IJob`/`IJobChunk` - unlike an `EntityCommandBuffer` there is no `.AsParallelWriter()`
  ceremony to add or drop; you pass the buffer value directly either way
  (`../../../vice-and-order-research/mods/dossiers/magic-garbage-truck/repo/Systems/TotalMagicSystem.cs#L130`).

## See also
- Related recipes: [Burst IJobChunk](burst-ijobchunk.md) (the job shape this buffer rides
  inside), [periodic UpdateInterval system](periodic-updateinterval-system.md) (running
  the sweep on a cadence, as this mod does via `GetUpdateInterval`).
- Reference: [ECS components catalog](../../reference/ecs-components-catalog.md) -
  `IconCommandSystem` / `IconCommandBuffer` (`Game.Notifications`), `GarbageProducer` /
  `GarbageProducerFlags` (`Game.Buildings`), `GarbageParameterData` (`Game.Prefabs`).
- Case studies demonstrating it: [magic-garbage-truck](../../case-studies/magic-garbage-truck.md).

## Sources
- Canonical mods (dossier + repo):
  - `magic-garbage-truck` @1b6a478753e1ef4e43ac9b90d567f3d7183c7be2 - `repo/Systems/TotalMagicSystem.cs`
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding
