---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: Event-driven mod-state write (ModificationEnd / ToolOutputBarrier)"
recipe: event-driven-modificationend
technique_family: "Y - Event-driven mod-state write (ModificationEnd / ToolOutputBarrier)"
diataxis: how-to
source_version: "~1.5.x (better-bulldozer@4408466; date-pinned, static source only)"
last_reverified: "2026-07-04"
status: source-verified
canonical_mods:
  - better-bulldozer@4408466f226db811159d92859479ae1e1c28ba06
  - anarchy@a6311e898d20a775368668b234aaa32f06e3e1eb
technique_applicability: [core]
Created: 2026-07-04
Updated: 2026-07-04
Owners:
  - codex
---

# Event-driven mod-state write (ModificationEnd / ToolOutputBarrier)

> Run your ECS mutation in response to a tool/modification event by scheduling the
> system at the right `SystemUpdatePhase` and writing through the *matching* barrier's
> command buffer - so your changes land in the same frame the event happened, without
> racing the simulation.

## Problem
You want mod logic to fire *because something happened* - a tool finished placing
geometry, sub-lanes were regenerated, an error prefab needs re-enabling - and then
write ECS changes (add/remove/set components, delete entities) that stick. Two things
bite here. First, if you run your write in the wrong phase it either clobbers other
writers or lands a frame late with no error. Second, structural ECS changes cannot be
made inside a running job or mid-phase directly; they must be queued into a **barrier**
(a system that owns an `EntityCommandBuffer` and plays it back at a fixed point). Pick
the wrong barrier for your phase and the playback timing is wrong.

## Solution
Schedule your system at the `SystemUpdatePhase` that corresponds to the event
(`ModificationEnd` for post-modification cleanup, `ToolUpdate` for tool-output work),
then acquire the barrier whose playback point matches that phase -
`ModificationEndBarrier` for `ModificationEnd`, `ToolOutputBarrier` for tool phases.
Queue every structural change into that barrier's `EntityCommandBuffer`; the barrier
plays it back and disposes it for you. The phase and the barrier are a matched pair:
choose them together, never independently.

## Steps & Code

### 1. Schedule the reactive systems at `ModificationEnd`

Better Bulldozer's auto-remove systems (fences/hedges, branding objects, regenerated
sub-elements) run their cleanup *after* the modification phase produced the geometry,
so they are registered at `SystemUpdatePhase.ModificationEnd` in `OnLoad`:

```csharp
updateSystem.UpdateAt<AutomaticallyRemoveFencesAndHedges>(SystemUpdatePhase.ModificationEnd);
updateSystem.UpdateAt<AutomaticallyRemoveBrandingObjects>(SystemUpdatePhase.ModificationEnd);
updateSystem.UpdateAt<RemoveRegeneratedSubelementPrefabsSystem>(SystemUpdatePhase.ModificationEnd);
```
Source: `../../../vice-and-order-research/mods/dossiers/better-bulldozer/repo/BetterBulldozer/BetterBulldozerMod.cs#L119-L121` (@4408466f226db811159d92859479ae1e1c28ba06)

`UpdateAt<T>(phase)` places system `T` in that phase's ordered group; the phase is what
makes the system "event-driven" - it only does meaningful work when that phase runs
after a modification.

### 2. Acquire the barrier that matches the phase

Because the system runs in `ModificationEnd`, it must use `ModificationEndBarrier`.
Anarchy states this rule verbatim on the field it caches in `OnCreate`:

```csharp
private ModificationEndBarrier m_Barrier; // System runs on SystemUpdatePhase.ModificationEnd therefore use ModificationEndBarrier. Using a barrier in the wrong phase will produce an error.
```
Source: `../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/Systems/ErrorChecks/EnableToolErrorsSystem.cs#L23` (@a6311e898d20a775368668b234aaa32f06e3e1eb)

Grab the barrier instance once in `OnCreate` via `GetOrCreateSystemManaged`:

```csharp
m_Barrier = World.GetOrCreateSystemManaged<ModificationEndBarrier>(); // Get an System reference to the barrier with the right timing.
```
Source: `../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/Systems/ErrorChecks/EnableToolErrorsSystem.cs#L52` (@a6311e898d20a775368668b234aaa32f06e3e1eb)

### 3. Queue changes into the barrier's command buffer - do not play it back yourself

In `OnUpdate`, ask the barrier for a fresh `EntityCommandBuffer`, queue your structural
changes onto it, and stop. The barrier owns playback and disposal:

```csharp
EntityCommandBuffer buffer = m_Barrier.CreateCommandBuffer(); // Create the command buffer that we will schedule changes too.
// ... compute toolErrorData, then queue it ...
buffer.SetComponent(currentEntity, toolErrorData); // Queue ups all changes to be played back automatically with ModificationEndBarrier. When using a barrier you should not manually playback the ECB, nor  should you dispose of the ECB. All handled by the barrier.
```
Source: `../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/Systems/ErrorChecks/EnableToolErrorsSystem.cs#L76-L99` (@a6311e898d20a775368668b234aaa32f06e3e1eb)

### 4. If you queue from a job, register the producer handle

The same pattern works from a scheduled job: pass `m_Barrier.CreateCommandBuffer()`
into the job struct, then tell the barrier which job produced entries so it waits for
that job before replaying. Better Bulldozer's auto-remove system does exactly this:

```csharp
HandleDeleteInXFramesJob handleDeleteInXFramesJob = new HandleDeleteInXFramesJob()
{
    m_DeleteInXFramesLookup = SystemAPI.GetComponentLookup<DeleteInXFrames>(isReadOnly: true),
    m_SubLanes = fenceAndHedgeSublanes,
    buffer = m_Barrier.CreateCommandBuffer(),
};

JobHandle handleDeleteInXFramesJobHandle = handleDeleteInXFramesJob.Schedule(Dependency);
m_Barrier.AddJobHandleForProducer(handleDeleteInXFramesJobHandle);
```
Source: `../../../vice-and-order-research/mods/dossiers/better-bulldozer/repo/BetterBulldozer/Systems/AutomaticallyRemoveFencesAndHedgesSystem.cs#L180-L188` (@4408466f226db811159d92859479ae1e1c28ba06)

Skipping `AddJobHandleForProducer` when you queued from a job is a data race: the
barrier may replay the buffer before the job has finished writing it.

## Pitfalls & gotchas

- **Phase and barrier MUST match - this is the whole point of the recipe.** A system in
  `SystemUpdatePhase.ModificationEnd` uses `ModificationEndBarrier`; a system in a tool
  phase uses `ToolOutputBarrier`. Anarchy spells it out verbatim: *"System runs on
  SystemUpdatePhase.ModificationEnd therefore use ModificationEndBarrier. Using a
  barrier in the wrong phase will produce an error."*
  (`../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/Systems/ErrorChecks/EnableToolErrorsSystem.cs#L23`).
  Mismatch symptoms: writes clobbered by another writer, or applied a frame late - and
  the failure is silent in-code. Whether the engine itself surfaces the "error" the
  comment warns about at runtime is `Needs Verification (in-game)`.

- **Never manually play back or dispose the barrier's ECB.** The buffer belongs to the
  barrier. Calling `Playback()` or `Dispose()` on it yourself double-plays or corrupts
  state. Anarchy documents this inline: *"When using a barrier you should not manually
  playback the ECB, nor should you dispose of the ECB. All handled by the barrier."*
  (`../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/Systems/ErrorChecks/EnableToolErrorsSystem.cs#L99`).

- **Queuing from a job without `AddJobHandleForProducer` is a race.** The barrier needs
  the producing `JobHandle` so it defers replay until your writes complete (Step 4).

- **`ModificationEnd` vs `Modification1..5`.** `ModificationEnd` is the terminal
  modification slot - use it for cleanup that must see the *finished* modification
  output. Better Bulldozer deliberately spreads related systems across phases:
  auto-remove systems at `ModificationEnd`, but `HandleUpdateNextFrameSystem` at
  `Modification5` and a grass system before `Modification1`
  (`../../../vice-and-order-research/mods/dossiers/better-bulldozer/repo/BetterBulldozer/BetterBulldozerMod.cs#L122-L128`).
  Order within the modification band is a real design lever, not incidental.

- **Self-disable event-driven systems when idle.** Anarchy's `EnableToolErrorsSystem`
  sets `Enabled = false` in `OnCreate` and only turns itself on when there is work,
  re-disabling at the end of `OnUpdate` - so the reactive write fires on the event, not
  every frame
  (`../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/Systems/ErrorChecks/EnableToolErrorsSystem.cs#L52-L109`).

## Variations

- **Tool-output work -> `ToolOutputBarrier` at a tool phase.** When your reaction is to
  tool output rather than post-modification cleanup, schedule at `ToolUpdate` and use
  `ToolOutputBarrier`. Better Bulldozer's `HandleDeleteInXFramesSystem` (registered at
  `ToolUpdate`, `BetterBulldozerMod.cs#L117`) caches and uses it identically:

  ```csharp
  m_Barrier = World.GetOrCreateSystemManaged<ToolOutputBarrier>();
  // ...in OnUpdate:
  EntityCommandBuffer buffer = m_Barrier.CreateCommandBuffer();
  // ...schedule job that writes to buffer...
  m_Barrier.AddJobHandleForProducer(jobHandle);
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/better-bulldozer/repo/BetterBulldozer/Systems/HandleDeleteInXFramesSystem.cs#L40-L65` (@4408466f226db811159d92859479ae1e1c28ba06)

  Same shape as the `ModificationEndBarrier` path - only the barrier type and the phase
  change, and they change *together*.

- **Main-thread queue (no job).** If the reaction is cheap and touches few entities,
  skip the job entirely: iterate a `Temp`-allocated entity array on the main thread and
  call `buffer.SetComponent(...)` directly, as Anarchy's `EnableToolErrorsSystem` does
  in `OnUpdate` (`EnableToolErrorsSystem.cs#L74-L109`). No `AddJobHandleForProducer` is
  needed because nothing was scheduled.

## See also
- Explanation: [system scheduling](../../explanation/system-scheduling.md) (why phases
  and barriers pair up).
- Reference: [system update phases](../../reference/system-update-phases.md) (the full
  `SystemUpdatePhase` enum and each phase's ordering).
- Case studies demonstrating it: [better-bulldozer](../../case-studies/better-bulldozer.md),
  [anarchy](../../case-studies/anarchy.md).

## Sources
- Canonical mods (dossier + repo):
  - `better-bulldozer` @4408466f226db811159d92859479ae1e1c28ba06 - `repo/BetterBulldozer/BetterBulldozerMod.cs`, `repo/BetterBulldozer/Systems/AutomaticallyRemoveFencesAndHedgesSystem.cs`, `repo/BetterBulldozer/Systems/HandleDeleteInXFramesSystem.cs`
  - `anarchy` @a6311e898d20a775368668b234aaa32f06e3e1eb - `repo/Anarchy/Systems/ErrorChecks/EnableToolErrorsSystem.cs`
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding
