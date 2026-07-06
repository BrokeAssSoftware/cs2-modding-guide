---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: Gate-only cross-entity coordination state machine"
recipe: gate-only-coordination
technique_family: "BA - Gate-only cross-entity coordination state machine"
diataxis: how-to
source_version: "~1.5.10f1 (traffic-tool-essentials@1097359; date-pinned, static source only)"
last_reverified: "2026-07-05"
status: source-verified
canonical_mods:
  - traffic-tool-essentials@10973595ac9ed37f47ca24eb788d4421dc295fa0
technique_applicability: [simulation, infrastructure]
Created: 2026-07-05
Updated: 2026-07-05
Owners:
  - codex
---

# Gate-only cross-entity coordination state machine

> Coordinate many entities that a vanilla system still owns (traffic-light phases,
> spawn cycles, timers) by GATING their transitions - preventing the one phase you
> do not want yet - instead of force-writing their state, so your logic composes
> with the vanilla state machine instead of fighting it.

## Problem
You want a group of entities to move in lockstep, but each entity's real state is
driven by a vanilla system you do not own and cannot stop. In Traffic Tool
Essentials the concrete case is a "green wave": every intersection runs its own
internal phase cycle (P1 -> P2 -> P3 -> P4 -> P1...) inside the game's traffic-light
simulation, and you can influence *when* a phase is allowed but not *when* it is
reached. The naive fix - directly writing each entity's phase every frame to force
alignment - fights the vanilla state machine, produces visual snapping and lane-signal
desync, and is exactly what TTE removed for stability. The source states the
constraint plainly: "We can only control WHEN P1 is allowed, not WHEN it's reached."

```
/// - Each intersection runs its own internal cycle (P1->P2->P3->P4->P1...)
/// - We can only control WHEN P1 is allowed, not WHEN it's reached
/// - Pure AVOID can block but not synchronize
```
Source: `../../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Systems/TrafficLightSystems/Simulation/SyncGroupSystem.cs#L83-L85` (@10973595ac9ed37f47ca24eb788d4421dc295fa0)

## Solution
Model the group with a small explicit state machine, but let it act only through a
single **AVOID gate** - a field that tells the vanilla system "do not transition into
this phase yet". Pick one entity as the **reference** that holds the current phase
while every other entity "rushes" toward the sync phase; when all are ready, drop the
gate and everyone enters the sync phase together. After alignment, remove all
intervention and let vanilla run freely. Two invariants make this safe: (1) an AVOID
field *composes* with vanilla far better than a FORCE field, because it constrains the
existing machine rather than overwriting it; and (2) because gating can wedge an entity
that never leaves a phase, you must ship a **deadlock force-release timeout** that
drops the gate unconditionally after a bounded wait.

## Steps & Code

### 1. Define the coordination states explicitly (not booleans)

The group's progress is a 6-value enum, not a scatter of flags. The ordering encodes
the protocol: an initial BARRIER that blocks the sync phase for everyone until the
group is aligned, then per-entity roles (reference waits; others rush), a ready state,
and a terminal SYNCED state that means "no more interventions".

```csharp
public enum SyncState
{
    BARRIER = 0,              // Block P1 for everyone until all are out of P1
    UNSYNCED = 1,            // Not yet synchronized - needs initial sync
    WAITING_FOR_OTHERS = 2,  // Reference: hold current phase until others ready
    RUSHING_TO_P1 = 3,       // Non-reference: rush through phases to reach P1
    READY_FOR_SYNC = 4,      // At last phase before P1, waiting for GO
    SYNCED = 5               // Synchronized - running freely, no interventions
}
```
Source: `../../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Systems/TrafficLightSystems/Simulation/SyncGroupSystem.cs#L14-L33` (@10973595ac9ed37f47ca24eb788d4421dc295fa0)

### 2. Make the coordination field an AVOID, not a FORCE

This is the load-bearing design choice. The component carries two mutually exclusive
levers: `m_ManualSignalGroup` FORCES a phase; `m_AvoidSignalGroup` PREVENTS one. The
field's own doc comment spells out the difference - the AVOID field is what the sync
machine writes:

```csharp
/// Signal group to avoid transitioning TO. Used by Green Wave to prevent
/// fast-cycling intersections from drifting into sync phase too early.
/// Unlike m_ManualSignalGroup which FORCES a phase, this PREVENTS a phase.
/// Not serialized - runtime only.
public byte m_AvoidSignalGroup;
```
Source: `../../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Components/CustomTrafficLights.cs#L47-L53` (@10973595ac9ed37f47ca24eb788d4421dc295fa0)

### 3. Apply the gate: write only the AVOID phase, clear the FORCE lever

Each tick the machine computes an `avoidPhase` for the entity and applies it by
writing the single byte. Note it first zeroes any legacy `m_ManualSignalGroup` so the
FORCE and AVOID levers can never fight: coordination is expressed purely as a gate.

```csharp
// === APPLY AVOID ===
byte previousAvoidGroup = ctl.m_AvoidSignalGroup;

// Ensure manual is always 0 (clear any legacy state)
if (ctl.m_ManualSignalGroup != 0)
{
    ctl.SetManualSignalGroup(0);
}

// Apply avoid signal group
ctl.m_AvoidSignalGroup = avoidPhase;
```
Source: `../../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Systems/TrafficLightSystems/Simulation/SyncGroupSystem.cs#L2178-L2188` (@10973595ac9ed37f47ca24eb788d4421dc295fa0)

### 4. Always ship a deadlock force-release timeout

A gate can wedge an entity that is genuinely stuck on the avoided phase. Guard it: if
an entity has held one phase longer than `maxCycleDuration * SAFETY_TIMEOUT_MULTIPLIER`
(floored at `SAFETY_TIMEOUT_MIN_TICKS`), drop the gate (`avoidPhase = 0`), grant a
grace period, and reset the entity to BARRIER for a clean re-sync. Traffic flows
immediately; the coordination heals rather than hangs.

```csharp
if (avoidPhase != 0)
{
    uint safetyTimeout = maxCycleDuration * SAFETY_TIMEOUT_MULTIPLIER;
    if (safetyTimeout < SAFETY_TIMEOUT_MIN_TICKS)
        safetyTimeout = SAFETY_TIMEOUT_MIN_TICKS;

    if (ctl.m_Timer > safetyTimeout)
    {
        avoidPhase = 0; // release immediately - traffic flows NOW
        decisionReason = "SAFETY_TIMEOUT_RELEASE";
        m_ForceReleaseGraceUntil[entity] = group.m_GroupTimer + maxCycleDuration;
        m_SyncStates[entity] = SyncState.BARRIER;
    }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Systems/TrafficLightSystems/Simulation/SyncGroupSystem.cs#L2124-L2141` (@10973595ac9ed37f47ca24eb788d4421dc295fa0)

The timeout constants live next to the state machine so they are auditable in one
place: `SAFETY_TIMEOUT_MULTIPLIER = 3` and `SAFETY_TIMEOUT_MIN_TICKS = 90`.

```csharp
private const uint SAFETY_TIMEOUT_MULTIPLIER = 3;
private const uint SAFETY_TIMEOUT_MIN_TICKS = 90;
```
Source: `../../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Systems/TrafficLightSystems/Simulation/SyncGroupSystem.cs#L144-L145` (@10973595ac9ed37f47ca24eb788d4421dc295fa0)

### 5. Run the coordination write in the SAME system group as the consumer

The gate write is worthless if the vanilla job that reads it runs first and writes the
whole component struct back. TTE fixes this write-after-write hazard by making
`SyncGroupSystem.OnUpdate` **intentionally empty** and instead calling `UpdateSync()`
by hand from `PatchedTrafficLightSystem.OnUpdate`, immediately before the traffic-light
job is scheduled - so the gate lands before the consumer reads it.

```csharp
protected override void OnUpdate()
{
    // V422.30: OnUpdate is intentionally empty.
    // UpdateSync() is called from PatchedTrafficLightSystem.OnUpdate() to ensure
    // correct execution order: sync writes m_AvoidSignalGroup BEFORE the traffic
    // light job reads it. This prevents the write-after-write hazard that occurred
    // when UpdateSync ran from UISystem (different SystemGroup, wrong timing).
}
```
Source: `../../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Systems/TrafficLightSystems/Simulation/SyncGroupSystem.cs#L729-L736` (@10973595ac9ed37f47ca24eb788d4421dc295fa0)

The consumer owns the ordering explicitly: it caches the sync system in `OnCreate`
and calls `UpdateSync()` at the top of its own `OnUpdate`, wrapped so a sync error can
never block traffic-light processing.

```csharp
// V422.30: Run GW sync BEFORE scheduling the traffic light job.
try
{
    if (m_SyncGroupSystem != null)
    {
        m_SyncGroupSystem.UpdateSync();
    }
}
catch (System.Exception e)
{
    Mod.LogError($"[PatchedTrafficLightSystem] V422.30: UpdateSync error: {e.Message}");
}
```
Source: `../../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Systems/TrafficLightSystems/Simulation/PatchedTrafficLightSystem.cs#L884-L905` (@10973595ac9ed37f47ca24eb788d4421dc295fa0)

### 6. Advance group time in the consumer's tick units, not display frames

Because `UpdateSync()` is now driven from the traffic-light system, the group timer
advances in that system's TL-tick unit. TTE derives `SIM_FRAMES_PER_TL_TICK = 64`
(update interval 4 x 16 update-frame buckets) and accumulates simulation frames into
whole TL-ticks so timing is independent of the display frame rate:

```csharp
// SIM_FRAMES_PER_TL_TICK = GetUpdateInterval(4) x UpdateFrame-Buckets(16) = 64
private const uint SIM_FRAMES_PER_TL_TICK = 64;
```
Source: `../../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Systems/TrafficLightSystems/Simulation/SyncGroupSystem.cs#L149-L150` (@10973595ac9ed37f47ca24eb788d4421dc295fa0)

Real-time conversions are pinned to the sim rate via `UPDATES_PER_REAL_SECOND = 60f`
so timeout durations map to wall-clock seconds regardless of game speed
(`SyncGroupSystem.cs#L45`, `RealSecondsToFrames`/`FramesToRealSeconds` at
`SyncGroupSystem.cs#L61-L73`).

## Pitfalls & gotchas

- **FORCE composes badly; AVOID composes well.** Directly writing the vanilla state
  each frame ("copy the reference's phase onto everyone") looks simpler but fights the
  state machine: TTE's own comment notes the mid-cycle lockstep of that copy approach
  was "illusory" - only the component/panel data followed the reference, the real lane
  signals did not - and the direct-copy path was removed
  (`../../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Systems/TrafficLightSystems/Simulation/SyncGroupSystem.cs#L2145-L2176`).
  Prefer a gate you can only *deny* with; never a write you *impose*.

- **Cross-SystemGroup write-after-write silently clobbers your write.** When the
  coordination write ran from the UI SystemGroup (which executes *after*
  `SimulationSystemGroup`), the traffic-light job read the old value, then wrote the
  entire struct back - overwriting the freshly written gate. The fix is not a fence;
  it is running the write from the *same* system, right before the consumer job
  (Step 5). If your gate "doesn't take", check *which group* writes it relative to who
  reads it.

- **A gate with no escape hatch deadlocks.** Gating can wedge an entity forever on the
  avoided phase. The `SAFETY_TIMEOUT` force-release (Step 4) is mandatory, not
  optional. Note the design detail: detection uses the entity's per-phase `m_Timer`
  (which the vanilla machine zeroes on every phase change), so a healthy free-cycling
  entity never trips - only one genuinely stuck on one phase accumulates enough ticks
  (`SyncGroupSystem.cs#L2108-L2122`).

- **Stale lookups across system boundaries.** `UpdateSync()` deliberately calls
  `GetComponentLookup<...>(false)` each tick rather than `Update(this)`, because a
  cached handle refreshed via `Update(this)` can hand back stale chunk data after a
  job write-back when the dependency chain crosses SystemGroups - the source calls
  this a "known ECS Heisenbug"
  (`../../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Systems/TrafficLightSystems/Simulation/SyncGroupSystem.cs#L513-L522`).

- **Docstring drift.** The `UpdateSync()` summary still reads "called from
  `UISystem.OnUpdate()`" (`SyncGroupSystem.cs#L506`), but the authoritative wiring is
  the empty `OnUpdate` (Step 5) plus the explicit call from
  `PatchedTrafficLightSystem`. Trust the call site, not the comment.

- The exact in-game smoothness of the green wave (visible platooning, absence of
  snapping) is a runtime-observable property and is **Needs Verification (in-game)**;
  the source proves the mechanism and ordering, not the visual outcome.

## Variations

- **Barrier-first alignment vs. free re-sync.** The `BARRIER` state (value 0) blocks
  the sync phase for the *entire* group until everyone is out of it, giving a clean
  cold-start alignment; the force-release path (Step 4) also routes a wedged entity
  back to `BARRIER` for a bounded, self-healing re-sync rather than a hard reset.

- **Reference-holds vs. all-rush.** Only the reference entity sits in
  `WAITING_FOR_OTHERS` holding its current phase; every non-reference takes
  `RUSHING_TO_P1`. Swapping which entity is the reference reshapes the group's timing
  without changing the gate mechanism - the coordination policy lives in the state
  assignment, not in the AVOID write.

- **Gate a spawn/timer instead of a phase.** The technique generalizes to any vanilla
  cycle you can deny-but-not-drive: replace `m_AvoidSignalGroup` with your own "not
  yet" flag consumed by the owning system, keep the explicit state enum, and keep the
  timeout. The three invariants (deny not impose; write in the consumer's group;
  bounded force-release) carry over unchanged.

## See also
- Explanation: [system scheduling & update order](../../explanation/system-scheduling.md)
  (why cross-SystemGroup ordering causes the write-after-write hazard),
  [ECS fundamentals](../../explanation/ecs-fundamentals.md).
- Related recipes: [ECS query rewriting](ecs-query-rewriting.md).
- Case study demonstrating it: [traffic-tool-essentials](../../case-studies/traffic-tool-essentials.md).

## Sources
- Canonical mods (dossier + repo):
  - `traffic-tool-essentials` @10973595ac9ed37f47ca24eb788d4421dc295fa0 -
    `repo/TrafficToolEssentials/Systems/TrafficLightSystems/Simulation/SyncGroupSystem.cs`,
    `repo/TrafficToolEssentials/Systems/TrafficLightSystems/Simulation/PatchedTrafficLightSystem.cs`,
    `repo/TrafficToolEssentials/Components/CustomTrafficLights.cs`
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding
