---
FrontmatterVersion: 1
DocumentType: Guide
Title: Performance and Terminology
Summary: Dry lookup for CS2 modding - the DOTS/ECS/Burst vocabulary used across this handbook, the simulation tick constant, and how to think about per-frame performance budgets (which are project-specific, not fixed by the game).
diataxis: reference
source_version: "1.6.0f1 (magic-mail@6fb3d2b; date-pinned, static source only)"
last_reverified: "2026-07-04"
canonical_mods:
  - magic-mail@6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47
status: source-verified
Created: 2026-07-03
Updated: 2026-07-03
Owners:
  - codex
References:
  - Label: System Scheduling (concept)
    Path: ../explanation/system-scheduling.md
  - Label: Conditional Execution and Diagnostics (concept)
    Path: ../explanation/conditional-execution.md
  - Label: Technique Index
    Path: ../technique-index.md
---

# Performance and Terminology

A dry lookup page: the vocabulary this handbook uses for CS2's DOTS/ECS runtime, the
one hard simulation constant, and how to reason about performance budgets. For the
authoritative engine docs, link out rather than copy - the Unity DOTS docs and the CS2
modding wiki are the sources of record (see [External references](#external-references)).

## Canonical terminology

Terms are grouped by layer. Where a term names a real type or constant used in a
source-verified mod, the concept page that demonstrates it is linked.

### Runtime model (DOTS / ECS)

| Term | Meaning |
| --- | --- |
| **ECS** (Entity Component System) | The data-oriented architecture CS2's simulation is built on: behaviour lives in *systems*, state lives in *components* attached to *entities*. |
| **DOTS** (Data-Oriented Technology Stack) | Unity's umbrella for the ECS runtime plus the Job System and Burst compiler that CS2 uses. |
| **Entity** | A lightweight id that components attach to (a citizen, a road segment, a building). |
| **Component** | A struct of plain data attached to an entity; the unit of simulation state. |
| **DynamicBuffer** | A variable-length component (e.g. a per-entity list such as `DynamicBuffer<Resources>`). |
| **World** | The container holding all entities, components, and systems. Mods reach it via `World.DefaultGameObjectInjectionWorld`. |
| **System** | A unit of behaviour that runs each time its phase fires. CS2 mod systems derive from `GameSystemBase` (a Colossal wrapper over Unity's `SystemBase`). |
| **EntityQuery** | A filter selecting the entities a system operates on (`WithAll` / `WithNone` / `WithAny`, or `Exclude<T>` narrowing). |
| **ComponentLookup** | A cached random-access handle for reading/writing a component type by entity; built once in `OnCreate`. |
| **SystemHandle** | A handle to another system, resolved once (e.g. via `GetOrCreateSystemManaged<T>()`) so you can order against or disable it. |

### Scheduling model

| Term | Meaning |
| --- | --- |
| **`UpdateSystem`** | The scheduler CS2 hands your mod in `OnLoad`; you register systems on it. |
| **`SystemUpdatePhase`** | The ordered set of per-frame slots a system can attach to (`Deserialize`, `ToolUpdate`, `Modification4B` / `ModificationEnd`, `GameSimulation`, `UIUpdate`, `UITooltip`, `Rendering`, and more). See [System Scheduling](../explanation/system-scheduling.md) for the phases exercised in this handbook's corpus; treat unseen phases as `Needs Verification`. |
| **`UpdateAt<T>(phase)`** | Run `T` during `phase`. |
| **`UpdateBefore<T, TOther>(phase)` / `UpdateAfter<T, TOther>(phase)`** | Order `T` before/after `TOther` *within the same phase*. Ordering does not cross phase boundaries. |
| **`GetUpdateInterval(phase)`** | Override on a system: ticks between runs (cadence). |
| **`GetUpdateOffset(phase)`** | Override on a system: phase offset used to stagger a system into a distinct slot. |
| **`RequireForUpdate<T>()` / `RequireForUpdate(query)`** | Gate: the system is skipped entirely unless the requirement is met. See [Conditional Execution](../explanation/conditional-execution.md). |

### Performance / compilation

| Term | Meaning |
| --- | --- |
| **Burst** | Unity's compiler that turns tight, unmanaged jobs into highly optimised native code. Burst-compiled code cannot touch managed objects. |
| **Job System** | Unity's worker-thread scheduler; DOTS jobs (e.g. `IJobChunk`) run work in parallel across chunks. |
| **`IJobChunk`** | A job that processes entities a *chunk* (block of same-layout entities) at a time - the common parallel pattern for heavy simulation work. |
| **Chunk** | A fixed-size block of memory holding entities with identical component layout; the granularity Burst jobs iterate. |
| **Managed vs unmanaged system** | Managed systems (`GetOrCreateSystemManaged<T>`) can hold references and be enabled/disabled at runtime; unmanaged systems are struct-based and Burst-friendly. |
| **`ProfilerMarker`** | A named scope that attributes frame time to a code path in the Unity profiler. `Needs Verification` - not demonstrated in the current corpus (see [Conditional Execution](../explanation/conditional-execution.md)). |

### The simulation tick constant

| Constant | Value | Meaning |
| --- | --- | --- |
| **Ticks per in-game day** | `262144` | CS2 quantizes one in-game day into 262144 simulation ticks. `262144 / N` = "run N times per day"; this is how a periodic system picks its `GetUpdateInterval`. Source-verified: Magic Mail computes its interval as `262144 / UpdatesPerDay` ([`magic-mail` `repo/Systems/MagicMailSystem.cs#L59-L67`](../../vice-and-order-research/mods/dossiers/magic-mail/repo/Systems/MagicMailSystem.cs), commit `6fb3d2b`). |

## Performance budgets

> **Performance targets are project-specific, not fixed by the game.** CS2 sets no
> published per-system frame budget for mods. The numbers below are *illustrative example
> targets* for reasoning about cost - they are **not** measured values and each carries
> `Needs Verification`. Establish your own budget against your own baseline hardware and
> representative saves before treating any figure as a threshold.

### How to think about a budget

A frame has a fixed time budget (e.g. ~16.7 ms at 60 FPS, shared across the whole game).
Your mod's systems compete for a slice of it. Set a target per category, measure against
it on a deterministic save, and profile before shipping.

| Category | Example target (illustrative, `Needs Verification`) | Notes |
| --- | --- | --- |
| Per-frame simulation system | under ~5 ms | Systems in `GameSimulation` that run every tick; the tightest constraint. |
| UI data-feed system | under ~2 ms | `UIUpdate`-phase systems feeding panels/overlays. |
| Background / periodic analytics | under ~1 ms | Amortised by running only every N ticks via `GetUpdateInterval`. |

These splits are a *starting shape*, not a rule. A mod that only runs a heavy system a few
times per in-game day can spend more per run; a system in `GameSimulation` every tick must
stay lean.

### Establishing your own targets

1. **Pick a baseline machine and settings.** Record CPU/GPU/RAM, resolution, and graphics
   preset. A budget is meaningless without the hardware it was measured on. (The original
   draft of this page hard-coded one team's reference machine; that has been removed because
   it does not generalise.)
2. **Keep deterministic benchmark saves.** Maintain a few representative saves (large city,
   stress scenario) and profile the same saves each release.
3. **Measure, do not guess.** Attribute cost with the profiler (and `ProfilerMarker` scopes
   once verified) rather than assuming; `Needs Verification` applies to any number you did
   not measure yourself.
4. **Re-verify per CS2 release.** Budgets and system costs shift between patches; this page
   is version-tagged `1.6.x` and should be re-checked each release.

## Reducing cost (cross-references)

The techniques that keep systems within budget are documented as concepts and recipes:

- **Cache in `OnCreate`, gate with `RequireForUpdate`, throttle with `GetUpdateInterval`** -
  see [System Scheduling](../explanation/system-scheduling.md) and
  [Conditional Execution](../explanation/conditional-execution.md).
- **Enable/run-once/disable instead of polling** - see
  [Conditional Execution](../explanation/conditional-execution.md) and the recipe
  [periodic-updateinterval-system](../how-to/recipes/periodic-updateinterval-system.md).
- **Burst `IJobChunk` parallelism** - technique family S in the
  [Technique Index](../technique-index.md).

## External references

Link out; do not duplicate these.

- Unity DOTS / Entities (systems, jobs, Burst): https://docs.unity3d.com/Packages/com.unity.entities@latest
- CS2 modding wiki (system update phases, runtime): https://cs2.paradoxwikis.com/Modding

## See also

- Concept: [System Scheduling](../explanation/system-scheduling.md)
- Concept: [Conditional Execution and Diagnostics](../explanation/conditional-execution.md)
- Concept: [System Replacement](../explanation/system-replacement.md)
- Reference: [Technique Index](../technique-index.md)
