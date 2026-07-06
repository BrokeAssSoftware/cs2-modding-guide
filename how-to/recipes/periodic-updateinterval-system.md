---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: Periodic GameSystemBase with UpdateInterval tuning"
recipe: periodic-updateinterval-system
technique_family: "L - Periodic GameSystemBase with UpdateInterval tuning"
diataxis: how-to
source_version: "1.6.0f1 (magic-mail@6fb3d2b; date-pinned, static source only)"
last_reverified: "2026-07-04"
status: source-verified
canonical_mods:
  - magic-mail@6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47
  - write-everywhere@13c70eb04e6bed152257c516a982455148f591a5
  - anarchy@a6311e898d20a775368668b234aaa32f06e3e1eb
  - magic-garbage-truck@1b6a478753e1ef4e43ac9b90d567f3d7183c7be2
  - abandoned-building-remover-deviance-fix@a515bfe588965cbc988cba6a974242f66e5386b7
technique_applicability: [core, simulation]
Created: 2026-07-04
Updated: 2026-07-05
Owners:
  - codex
---

# Periodic GameSystemBase with UpdateInterval tuning

> Make a `GameSystemBase` fire on a slow, deliberate cadence (e.g. ~32 times per
> in-game day) instead of every simulation frame, by overriding
> `GetUpdateInterval` - and optionally `GetUpdateOffset` to stagger it.

## Problem
Your system does periodic bookkeeping - scanning post facilities, topping up mail,
disposing stale entities - work that is meaningful once every in-game while, not 60+
times a second. If it runs in its update phase every frame it burns CPU and can race
the vanilla system it shadows. You want a controlled cadence measured in in-game
time, not frames, and you want to slot the system safely relative to the vanilla one.

## Solution
A system registered into a `SystemUpdatePhase` runs at that phase's default rate. To
throttle it, override `GetUpdateInterval(SystemUpdatePhase phase)` and return the
number of **simulation frame-ticks between updates**. The simulation clock advances
`262144` ticks per in-game day, so `262144 / UpdatesPerDay` gives a per-day cadence:
pick `UpdatesPerDay = 32` and the system fires every `8192` ticks (~32x/day). If you
also override `GetUpdateOffset`, you shift the phase so your system lands in a safe
slot **before** the vanilla system it mirrors, instead of colliding with it. Return
`1` from `GetUpdateInterval` to run effectively every update (the throttle is opt-in,
so most systems that do not override it just run at their phase's native rate).

## Steps & Code

### 1. Extend `GameSystemBase` and pick a per-day cadence

Declare a constant for readability, then derive the interval from the day-tick
constant. Magic Mail's flagship system chooses 32 updates per in-game day:

```csharp
public partial class MagicMailSystem : GameSystemBase
{
    /// <summary>
    /// Controls how often the system updates for each phase.</summary>
    private const int UpdatesPerDay = 32;   // ~ once per 45 in-game minutes.
```
Source: `../../../vice-and-order-research/mods/dossiers/magic-mail/repo/Systems/MagicMailSystem.cs#L33-L57` (@6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47)

### 2. Override `GetUpdateInterval` to return ticks-between-updates

`262144` is the number of simulation ticks in one in-game day; dividing by
`UpdatesPerDay` yields the frame-tick interval. Note the DEBUG branch deliberately
returns a *different* value to line the system up with the vanilla facility system:

```csharp
public override int GetUpdateInterval(SystemUpdatePhase phase)
{
#if DEBUG
    // Matching the vanilla PostFacilityAISystem interval for easier debugging.
    return 256;
#else
    return 262144 / UpdatesPerDay;
#endif
}
```
Source: `../../../vice-and-order-research/mods/dossiers/magic-mail/repo/Systems/MagicMailSystem.cs#L59-L67` (@6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47)

In Release this returns `262144 / 32 = 8192` ticks between updates; in DEBUG it
returns `256`. Both are constants independent of `phase` here, but the signature
lets you vary the interval per phase if a system runs in more than one.

### 3. Optionally override `GetUpdateOffset` to stagger the system

The offset shifts *when within the interval window* the system fires, so it does not
land on the same tick as the vanilla system it shadows:

```csharp
/// <summary>
/// Controls when the system runs relative to other systems.</summary>
public override int GetUpdateOffset(SystemUpdatePhase phase)
{
    // Keeps this system in a safe slot before the vanilla system.
    return 48;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/magic-mail/repo/Systems/MagicMailSystem.cs#L69-L75` (@6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47)

### 4. Return `1` when you want "every update" (throttle is opt-in)

The same override, returning `1`, means "run every time this phase ticks." Magic
Mail's capacity system uses this together with self-`Enabled` gating so it runs once
right after a load, then disables itself:

```csharp
/// <summary>
/// Runs immediately once enabled, then disables itself.
/// </summary>
public override int GetUpdateInterval(SystemUpdatePhase phase)
{
    return 1;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/magic-mail/repo/Systems/MailCapacitySystem.cs#L79-L85` (@6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47)

### 5. Second real example: interval from a named constant

Write Everywhere's disposal system throttles cleanup work the same way, sourcing the
interval from a shared constant rather than a per-day computation:

```csharp
public override int GetUpdateInterval(SystemUpdatePhase phase)
{
    return WEConstants.DISPOSAL_FRAME_INTERVAL;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/write-everywhere/repo/BelzontWE/Templates/WETemplateDisposalSystem.cs#L76-L79` (@13c70eb04e6bed152257c516a982455148f591a5)

The constant is `256` - one dispose pass every 256 frame-ticks:

```csharp
// Frame interval configuration
public const int RENDERER_FRAME_CHECK_MASK = 0x1f;
public const int DISPOSAL_FRAME_INTERVAL = 256;
```
Source: `../../../vice-and-order-research/mods/dossiers/write-everywhere/repo/BelzontWE/Utils/WEConstants.cs#L24-L26` (@13c70eb04e6bed152257c516a982455148f591a5)

### 6. Per-day cadence driving a Burst sweep (service purge)

Magic Garbage Truck uses the same `TicksPerDay / UpdatesPerDay` form, but names the
day-tick constant explicitly instead of inlining `262144`, and each throttled tick
schedules a Burst job that zeroes state across every matching entity:

```csharp
public static readonly int UpdatesPerDay = 64; // raise (128, 256...) if warn garbage icons appear
private const int TicksPerDay = 262144;
// ...
public override int GetUpdateInterval(SystemUpdatePhase phase)
{
    return TicksPerDay / UpdatesPerDay;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/magic-garbage-truck/repo/Systems/TotalMagicSystem.cs#L34-L44` (@1b6a478753e1ef4e43ac9b90d567f3d7183c7be2)

`262144 / 64 = 4096` ticks between sweeps (~22.5 in-game minutes). The work each
sweep does is a `[BurstCompile] IJobChunk` that clears the garbage backlog on every
`GarbageProducer` in the query:

```csharp
[BurstCompile]
private struct MagicJob : IJobChunk
{
    // ...
    producer.m_Garbage = 0;
    producer.m_CollectionRequest = Entity.Null;
    producer.m_DispatchIndex = 0;
    producer.m_Flags &= ~GarbageProducerFlags.GarbagePilingUpWarning;
    producers[i] = producer;
```
Source: `../../../vice-and-order-research/mods/dossiers/magic-garbage-truck/repo/Systems/TotalMagicSystem.cs#L124-L158` (@1b6a478753e1ef4e43ac9b90d567f3d7183c7be2)

This is the throttle earning its keep: a full-population component sweep is exactly
the kind of work you do 64x/day, not 60x/second.

### 7. Cheap idle baseline: `RequireForUpdate` + a small interval, no settings

At the opposite end from Magic Garbage Truck's tuned `UpdatesPerDay`, Abandoned
Building Remover ships the minimum viable throttle - a fixed small interval and a
query gate, with no settings surface at all (no user-facing cadence option in the
system). `RequireForUpdate` skips the system entirely while no matching entities
exist, so the interval only costs anything when there is work to do:

```csharp
RequireForUpdate(_abandonedBuildingQuery);
// ...
public override int GetUpdateInterval(SystemUpdatePhase phase) => phase == SystemUpdatePhase.GameSimulation ? 16 : 1;
```
Source: `../../../vice-and-order-research/mods/dossiers/abandoned-building-remover-deviance-fix/repo/AbandonedBuildingRemoverSystem.cs#L49` and `#L107` (@a515bfe588965cbc988cba6a974242f66e5386b7)

Interval `16` in `GameSimulation` (and `1` elsewhere) is a hard-coded literal rather
than a derived per-day figure - fine for a maintenance sweep where an exact in-game
cadence does not matter and the query gate already caps the cost.

## Pitfalls & gotchas

- **Most systems do NOT override the interval - the throttle is opt-in.** Anarchy is
  the clean contrast: it registers a dozen `GameSystemBase` subclasses into explicit
  phases (`updateSystem.UpdateAt<...>`, `UpdateBefore<...>`, `UpdateAfter<...>`) but
  overrides `GetUpdateInterval` in **none** of them - a repo-wide search for the
  method at the pin returns zero hits. Those systems run at their phase's native rate.
  Registration:
  `../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/AnarchyMod.cs#L131-L151`;
  a representative system with no interval override:
  `../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/Systems/OverridePrevention/PreventOverrideSystem.cs#L19`
  (@a6311e898d20a775368668b234aaa32f06e3e1eb). Reach for this recipe only when
  per-frame is genuinely wasteful; do not throttle tool/UI-reactive systems that must
  respond immediately.

- **The `262144` magic number is ticks-per-in-game-day.** The interval is in
  simulation frame-ticks, not seconds and not real frames. `262144 / UpdatesPerDay`
  is the whole point of the constant: it converts a human-legible "N times per day"
  into the tick count the scheduler expects
  (`MagicMailSystem.cs#L57-L65`). Hard-coding a raw tick count instead loses that
  intent and is easy to get wrong by an order of magnitude.

- **DEBUG vs Release divergence is real and intentional here.** Magic Mail returns
  `256` under `#if DEBUG` but `262144 / 32 = 8192` in Release - a **32x** difference
  in cadence between a debug build and a shipped build
  (`MagicMailSystem.cs#L59-L67`). That is deliberate (the debug value matches the
  vanilla `PostFacilityAISystem` interval for side-by-side debugging), but it means
  behaviour you observe in a debug build does NOT reflect shipping cadence. Always
  confirm timing in a Release build.

- **Interval is not the same as offset.** `GetUpdateInterval` sets *how often*;
  `GetUpdateOffset` sets *where in the window*. Magic Mail returns `48` from the
  offset to sit "in a safe slot before the vanilla system"
  (`MagicMailSystem.cs#L69-L75`). Omitting the offset does not change the frequency,
  only the phase alignment; whether a given offset actually avoids a race with a
  specific vanilla system is timing-dependent - **Needs Verification (in-game)**.

- **`return 1` is not "disabled" - it is "every update."** Interval `1` means the
  system runs on every tick of its phase. Magic Mail's capacity system pairs it with
  `Enabled = false` self-gating so the practical effect is one run after load
  (`MailCapacitySystem.cs#L82-L84`); the throttle itself imposes no delay.

## Variations

- **Interval from a shared constant (Write Everywhere).** Instead of computing a
  per-day cadence inline, return a named constant so the value is centralised and
  testable (Write Everywhere even asserts `DISPOSAL_FRAME_INTERVAL == 256` in a unit
  test at `BelzontWE.Tests/Utils/WEConstantsTests.cs#L75-L76`). Good when several
  systems should share one cadence.

- **Run-once-after-load (Magic Mail capacity).** Combine `GetUpdateInterval => 1`
  with `Enabled = false` in `OnCreate` and `Enabled = true` in
  `OnGameLoadingComplete` so the system fires immediately post-load, does its work,
  and disables itself
  (`MailCapacitySystem.cs#L61-L84`). This is throttling by lifecycle rather than by
  frequency.

- **Per-day cadence (Magic Mail flagship).** The `262144 / UpdatesPerDay` form when
  you want the cadence expressed in in-game time; tune `UpdatesPerDay` (32 here) to
  trade responsiveness against cost.

- **No override at all (Anarchy).** When a system must react to tool/UI state every
  frame, skip the override entirely and rely on phase registration
  (`AnarchyMod.cs#L131-L151`).

- **Named day-tick constant + Burst sweep (Magic Garbage Truck).** Same per-day form
  as Magic Mail, but `TicksPerDay = 262144` is a `private const` rather than an inline
  literal, and each throttled tick schedules a `[BurstCompile] IJobChunk` that zeroes
  `GarbageProducer` fields across the whole query
  (`TotalMagicSystem.cs#L34-L44`, `#L124-L158`). Use this shape when the periodic work
  is a full-population component mutation that benefits from Burst + parallel chunks.

- **Minimum-viable throttle, no settings surface (Abandoned Building Remover).** A
  fixed `GetUpdateInterval => 16` (in `GameSimulation`) paired with
  `RequireForUpdate` on the target query, no per-day math and no user-exposed cadence
  option (`AbandonedBuildingRemoverSystem.cs#L49`, `#L107`). The query gate means the
  interval only costs anything when abandoned buildings actually exist - a good
  default for cheap idle maintenance sweeps.

## See also
- Explanation: [system scheduling](../../explanation/system-scheduling.md) (the
  interval/offset tuning concept and the phase model), [ECS
  fundamentals](../../explanation/ecs-fundamentals.md).
- Reference: [system update phases](../../reference/system-update-phases.md)
  (`SystemUpdatePhase` values and their native rates), [technique
  index](../../technique-index.md).
- Case study: [magic-mail](../../case-studies/magic-mail.md) (the flagship periodic
  system in context).

## Sources
- Canonical mods (dossier + repo):
  - `magic-mail` @6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47 - `repo/Systems/MagicMailSystem.cs`, `repo/Systems/MailCapacitySystem.cs`
  - `write-everywhere` @13c70eb04e6bed152257c516a982455148f591a5 - `repo/BelzontWE/Templates/WETemplateDisposalSystem.cs`, `repo/BelzontWE/Utils/WEConstants.cs`
  - `anarchy` @a6311e898d20a775368668b234aaa32f06e3e1eb - `repo/Anarchy/AnarchyMod.cs`, `repo/Anarchy/Systems/OverridePrevention/PreventOverrideSystem.cs`
  - `magic-garbage-truck` @1b6a478753e1ef4e43ac9b90d567f3d7183c7be2 - `repo/Systems/TotalMagicSystem.cs`
  - `abandoned-building-remover-deviance-fix` @a515bfe588965cbc988cba6a974242f66e5386b7 - `repo/AbandonedBuildingRemoverSystem.cs`
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding
