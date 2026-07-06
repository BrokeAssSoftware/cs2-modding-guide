---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: Timed per-agent FSM persisted in ECS"
recipe: ecs-agent-fsm
technique_family: "AK - Timed per-agent FSM persisted in ECS"
diataxis: how-to
source_version: "~1.6.0f1 (time2work-realistic-trips@d42921f; date-pinned, static source only)"
last_reverified: "2026-07-05"
status: source-verified
canonical_mods:
  - time2work-realistic-trips@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39
technique_applicability: [simulation]
Created: 2026-07-05
Updated: 2026-07-05
Owners:
  - codex
---

# Timed per-agent FSM persisted in ECS

> Run a per-citizen state machine (enter a state, hold it for a timed interval, then
> transition out) whose in-flight state is stored in a serialized ECS component, so a
> save/load resumes the agent mid-state instead of losing or restarting it.

## Problem
You want an agent (a citizen, vehicle, animal) to enter a behavioural state, stay in
it for a duration, then move on - and you need three things vanilla per-frame systems
do not give you for free: (1) the "I am currently in state X, until frame N" fact has
to **survive save/load** (a player can save mid-state and reload hours later); (2) the
deadline must be robust to variable simulation speed and pause; (3) turning the
feature off must not leave agents **stuck** in a half-applied state. A stateless system
that recomputes everything every frame cannot remember "this specific cim is on hour 3
of a 6-hour hospital stay" across a reload.

## Solution
Model the in-flight state as its own **ECS tag/state component that implements
`ISerializable`**, so Colossal's save system writes and restores it with the entity.
Store the **deadline as an absolute simulation frame index** (`endFrame`) inside that
component, not a wall-clock time or a countdown you decrement. A periodic system
queries for the tag, and on each tick compares `SimulationSystem.frameIndex` against
`endFrame`: below it, re-assert the hold state; at or past it, remove the tag and route
the agent onward. Because the deadline is an absolute frame and the whole record is
persisted, a reload picks up exactly where it left off. Guard the loop so that when the
feature toggle is off (or the agent is no longer eligible) the system **strips its own
component**, which is what stops agents getting stranded.

time2work-realistic-trips implements this as a hospital-stay FSM: a cim who arrives at a
hospital is tagged, held `InHospital` on a 64-frame cadence until the stay's `endFrame`,
then sent `GoingHome`.

## Steps & Code

### 1. Define the in-flight state as a serialized ECS component

The state record is a plain `IComponentData` struct that also implements
`ISerializable`. The key fields are the deadline (`endFrame`) plus a `version` int for
forward-compatible save format changes:

```csharp
public struct HospitalStay : IComponentData, IQueryTypeParameter, ISerializable
{
    public int version;
    public uint startFrame;
    public uint endFrame;
    public float durationHours;

    public HospitalStay(uint startFrame, uint endFrame, float durationHours)
    {
        version = 1;
        this.startFrame = startFrame;
        this.endFrame = endFrame;
        this.durationHours = durationHours;
    }
```
Source: `../../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Components/HospitalStay.cs#L6-L19` (@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39)

Implementing `Serialize`/`Deserialize` is what makes the state survive save/load - the
save system walks these fields in order. Write `version` first so a later format can
branch on it while reading:

```csharp
public void Serialize<TWriter>(TWriter writer) where TWriter : IWriter
{
    writer.Write(version);
    writer.Write(startFrame);
    writer.Write(endFrame);
    writer.Write(durationHours);
}

public void Deserialize<TReader>(TReader reader) where TReader : IReader
{
    reader.Read(out version);
    reader.Read(out startFrame);
    reader.Read(out endFrame);
    reader.Read(out durationHours);
}
```
Source: `../../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Components/HospitalStay.cs#L21-L35` (@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39)

(See [Persist mod data across save/load](ecs-serializable-savedata.md) for the
`ISerializable` contract in full.)

### 2. Enter the state: stamp the tag with a frame-index deadline

Something has to put an agent into the FSM. Compute the deadline as
`frameIndex + durationFrames` - an **absolute** future frame - and add the component.
Note the deadline is derived from a sampled duration converted into frames, never a
wall-clock timestamp:

```csharp
uint durationFrames = (uint)math.max(1, (int)math.round(sampledHours / 24f * math.max(1, ticksPerDay)));
this.m_CommandBuffer.AddComponent<HospitalStay>(
    unfilteredChunkIndex,
    entity,
    new HospitalStay(frameIndex, frameIndex + durationFrames, sampledHours));
```
Source: `../../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Systems/Time2WorkCitizenTravelPurposeSystem.cs#L1019-L1023` (@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39)

This entry point also early-returns when the feature toggle is off, so no new agents
enter the FSM while it is disabled
(`.../Time2WorkCitizenTravelPurposeSystem.cs#L998`).

### 3. Drive the FSM from a periodic system that queries the tag

The FSM lives in its own `GameSystemBase`. Set a coarse cadence with
`GetUpdateInterval` (here every 64 simulation frames - see
[Periodic system on an update interval](periodic-updateinterval-system.md)) so the loop
is cheap:

```csharp
public override int GetUpdateInterval(SystemUpdatePhase phase)
{
    return 64;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Systems/HospitalStaySystem.cs#L18-L21` (@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39)

In `OnCreate`, cache `SimulationSystem` (the source of `frameIndex`) and build a query
for exactly the agents carrying the state tag, excluding `Deleted`/`Temp`. Gating the
system on this query with `RequireForUpdate` means it does nothing when no agent is
mid-state:

```csharp
m_SimulationSystem = World.GetOrCreateSystemManaged<SimulationSystem>();

m_HospitalStayQuery = GetEntityQuery(new EntityQueryDesc
{
    All = new[]
    {
        ComponentType.ReadWrite<HospitalStay>(),
        ComponentType.ReadOnly<Citizen>(),
        ComponentType.ReadOnly<CurrentBuilding>()
    },
    None = new[]
    {
        ComponentType.Exclude<Deleted>(),
        ComponentType.Exclude<Temp>()
    }
});
RequireForUpdate(m_HospitalStayQuery);
```
Source: `../../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Systems/HospitalStaySystem.cs#L26-L43` (@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39)

### 4. Self-clean on toggle-off inside the loop

The `OnUpdate` loop reads the toggle every tick. If the feature is disabled, it
**removes its own tag component** from each agent instead of processing it - the agent
falls out of the FSM cleanly rather than freezing in the state:

```csharp
for (int i = 0; i < entities.Length; i++)
{
    Entity citizen = entities[i];
    if (setting == null || !setting.hospital_stay_duration_enabled)
    {
        EntityManager.RemoveComponent<HospitalStay>(citizen);
        continue;
    }

    ProcessHospitalStay(citizen, EntityManager.GetComponentData<HospitalStay>(citizen));
}
```
Source: `../../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Systems/HospitalStaySystem.cs#L52-L62` (@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39)

### 5. The timed transition: hold until `frameIndex >= endFrame`, then route out

This is the heart of the FSM. Compare the live simulation frame to the persisted
deadline. While the deadline is in the future, re-assert the state and return; once it
is reached, remove the tag and transition the agent to its next behaviour:

```csharp
bool completed = m_SimulationSystem.frameIndex >= stay.endFrame;
if (!completed)
{
    EnsureInHospital(citizen, hospital);
    return;
}

EntityManager.RemoveComponent<HospitalStay>(citizen);
// ... (skip if still sick / dead) ...
SendHomeIfPossible(citizen, hospital);
```
Source: `../../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Systems/HospitalStaySystem.cs#L86-L113` (@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39)

`EnsureInHospital` is the "hold" action - it is idempotent, only writing the agent's
`TravelPurpose`/`Target` when they are not already correct, so re-running it every 64
frames is harmless:

```csharp
if (EntityManager.HasComponent<TravelPurpose>(citizen))
{
    TravelPurpose travelPurpose = EntityManager.GetComponentData<TravelPurpose>(citizen);
    if (travelPurpose.m_Purpose != Purpose.InHospital)
    {
        travelPurpose.m_Purpose = Purpose.InHospital;
        EntityManager.SetComponentData(citizen, travelPurpose);
    }
}
else
{
    EntityManager.AddComponentData(citizen, new TravelPurpose { m_Purpose = Purpose.InHospital });
}
```
Source: `../../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Systems/HospitalStaySystem.cs#L118-L133` (@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39)

The exit action (`SendHomeIfPossible`) queues a `GoingHome` trip only after validating
the agent still has a household, a home building, and a `TripNeeded` buffer - otherwise
it returns without stranding the agent
(`.../HospitalStaySystem.cs#L153-L174`).

### 6. Also self-clean when the agent becomes ineligible

Beyond the toggle, `ProcessHospitalStay` strips the tag whenever the agent no longer
belongs in the state - it left the building, the building is not a valid hospital, or
the cim died - so the FSM never holds an agent in an impossible state:

```csharp
if (!EntityManager.HasComponent<CurrentBuilding>(citizen))
{
    EntityManager.RemoveComponent<HospitalStay>(citizen);
    return;
}

Entity hospital = EntityManager.GetComponentData<CurrentBuilding>(citizen).m_CurrentBuilding;
if (!IsValidHospital(hospital) || IsDead(citizen))
{
    EntityManager.RemoveComponent<HospitalStay>(citizen);
    return;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Systems/HospitalStaySystem.cs#L73-L84` (@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39)

## Pitfalls & gotchas

- **State that is not `ISerializable` is lost on reload.** The entire point is that a
  save mid-state resumes correctly. If your state component is a plain `IComponentData`
  without `Serialize`/`Deserialize`, the save system will not persist your fields and a
  reload drops (or re-adds with a wrong deadline) the FSM state. `HospitalStay`
  implements `ISerializable` precisely so the in-flight stay survives
  (`.../HospitalStay.cs#L6`, `#L21-L35`).

- **Use an absolute frame deadline, not a countdown or wall-clock time.** Storing
  `endFrame = frameIndex + durationFrames` and testing `frameIndex >= endFrame` is
  robust across save/load, pause, and speed changes because it is anchored to the
  simulation clock (`.../HospitalStaySystem.cs#L86`). A decremented counter would need
  ticking every frame (defeating the coarse 64-frame cadence) and a wall-clock time
  would drift with game speed. Note `frameIndex`/`endFrame` are `uint` - a stored
  `endFrame` is only meaningful within the same save's frame timeline.

- **Self-clean on BOTH toggle-off and ineligibility, or agents get stuck.** Because the
  hold action forces `TravelPurpose = InHospital` every tick, an agent whose tag is
  never removed would be pinned in the hospital forever. The system removes its own
  component when the feature is disabled (`.../HospitalStaySystem.cs#L57`) and when the
  agent leaves / the building is invalid / the cim dies (`#L73-L84`). Omitting either
  branch is how you strand agents.

- **Make the hold action idempotent.** The loop re-runs every 64 frames, so
  `EnsureInHospital` checks the current value before writing and only mutates on a real
  change (`.../HospitalStaySystem.cs#L118-L143`). A hold action that unconditionally
  re-adds components or re-issues orders each tick will thrash the simulation.

- **Entry point must also respect the toggle.** Disabling the feature stops *new*
  agents entering (`AddHospitalStay` early-returns at
  `.../Time2WorkCitizenTravelPurposeSystem.cs#L998`) while the FSM system drains the
  ones already mid-state. Guarding only one side leaves a leak.

- **Exact stay lengths / distributions are behavioural.** The sampled durations,
  inpatient probabilities, and how `ticksPerDay` maps hours to frames determine how the
  FSM *feels* in play; those tuning outcomes are `Needs Verification (in-game)`. What is
  proven from source is the FSM mechanism itself.

## Variations

- **Multi-state FSM (not just hold/exit).** Add an enum or additional fields to the
  serialized component and branch in the processing method to model several sequential
  states (e.g. `Arriving -> Treatment -> Recovery -> Discharge`), each with its own
  `endFrame`. The `version` field on `HospitalStay` (`.../HospitalStay.cs#L8`) exists so
  you can extend the serialized layout later without breaking old saves.

- **Contrast: stateless per-frame system.** If the behaviour has no memory that must
  survive reload - e.g. "every frame, cims currently at a hospital get purpose X" - you
  do not need a persisted tag at all; query the existing vanilla components and act.
  Reach for this recipe only when the *in-flight, per-agent progress* must persist. The
  serialized tag is the cost you pay for resumability.

- **Different cadence / phase.** The 64-frame interval is a cost/latency trade
  (`.../HospitalStaySystem.cs#L18-L21`); a shorter interval reacts faster to the
  deadline at higher cost. See [Periodic system on an update
  interval](periodic-updateinterval-system.md) for choosing the value.

## See also
- Related recipes: [Persist mod data across save/load](ecs-serializable-savedata.md)
  (the `ISerializable` component contract this relies on),
  [Periodic system on an update interval](periodic-updateinterval-system.md) (the
  `GetUpdateInterval` cadence driving the FSM).
- Reference: [ECS fundamentals](../../explanation/ecs-fundamentals.md) (components,
  queries, systems), [technique index](../../technique-index.md).
- Case study demonstrating it:
  [time2work-realistic-trips](../../case-studies/time2work-realistic-trips.md).

## Sources
- Canonical mods (dossier + repo):
  - `time2work-realistic-trips` @d42921ffb1f6bbbf2c2b086bb17727bf0f637c39 -
    `repo/NightShift/Components/HospitalStay.cs`,
    `repo/NightShift/Systems/HospitalStaySystem.cs`,
    `repo/NightShift/Systems/Time2WorkCitizenTravelPurposeSystem.cs`
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding
