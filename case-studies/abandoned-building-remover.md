---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Case study: Abandoned Building Remover"
case_study: abandoned-building-remover
mod: "Abandoned Building Remover (Deviance Fix)"
dossier: ../../vice-and-order-research/mods/dossiers/abandoned-building-remover-deviance-fix/
repo_commit: a515bfe588965cbc988cba6a974242f66e5386b7
source_version: "1.6.0f1 (abandoned-building-remover-deviance-fix@a515bfe; date-pinned, static source only)"
last_reverified: "2026-07-05"
diataxis: explanation
techniques: [AF]
technique_applicability: [core, simulation, operations]
status: source-verified
Created: 2026-07-05
Updated: 2026-07-05
Owners:
  - codex
Summary: A ~160-line structural-deletion mod that tags an abandoned building and every entry of its SubArea/SubNet/SubLane buffers with Deleted, then lets vanilla cleanup do the rest - a clean study in idempotent-tag cascade deletion, a compile-time Burst/managed dual path, and cheap-idle scheduling.
---

# Abandoned Building Remover - case study

> A whole shipped mod in one ~160-line system: it finds abandoned buildings,
> tags the building **and** its entire owned infrastructure sub-graph with
> `Deleted`, and hands off to vanilla economy/refund/notification flows - the
> minimal, correct shape of an ECS structural-cascade-deletion mod.

All code claims below are cited to `repo/<path>#Lxx` at pinned commit
`a515bfe588965cbc988cba6a974242f66e5386b7`, surfaced through the dossier at
`../../vice-and-order-research/mods/dossiers/abandoned-building-remover-deviance-fix/`.
This is the "Deviance Fix" recompile of the original Wayzware mod; the published
XML still carries the original mod identity (`ModId 75107`, `ModVersion 1.0.0`,
`GameVersion 1.1.*`, repo/Properties/PublishConfiguration.xml#L3-L28) even though
the source compiles and runs against a much later game - see the
recompile-provenance pitfall below.

## What it does / why it's instructive

The mod deletes abandoned buildings automatically. In vanilla CS2 an abandoned
building sits derelict until the player bulldozes it; this mod continuously sweeps
for the `Abandoned` state and demolishes those buildings a moment after the
simulation runs. That is the entire player-facing feature.

It is instructive precisely because it is *tiny and complete*. The whole mod is
two files - a `Mod` entry point (repo/Mod.cs#L1-L27) and one
`GameSystemBase` (repo/AbandonedBuildingRemoverSystem.cs#L1-L160) - yet it
demonstrates the full correct shape of a **structural-deletion** mod:

- It does not "destroy" anything itself. It only **adds a `Deleted` tag** to the
  building and each child infrastructure entity, then lets the game's own cleanup,
  refund, and notification pipeline finish the job.
- Tagging is **idempotent and order-independent**, which is what lets the mod ship
  two completely separate code paths (a Burst job and a managed loop) that scrub
  the child buffers in *different orders* with no behavioural difference.
- It stays cheap when idle via `RequireForUpdate` plus a throttled update
  interval.

It is a good teaching artifact because everything a beginner tends to get wrong
about deleting entities in DOTS - orphaning sub-lanes, double-destroying, fighting
the barrier, over-scheduling - is handled here in a way you can read in one sitting.

## Architecture at a glance

One system, one entry point. `Mod.OnLoad` registers the system into
`SystemUpdatePhase.GameSimulation` with `UpdateAfter`, so the sweep runs *after*
the simulation has updated building state each frame
(repo/Mod.cs#L20). A commented-out line right above it,
`//updateSystem.UpdateAfter<...>(SystemUpdatePhase.Deserialize)`, is left in as a
first-party record of the rejected alternative: running once at load rather than
continuously (repo/Mod.cs#L19-L20).

`AbandonedBuildingRemoverSystem` builds one query in `OnCreate`: **All**
`Abandoned` + `Building`, **None** `Deleted` + `Temp`
(repo/AbandonedBuildingRemoverSystem.cs#L28-L40). The `None` clause is what makes
the sweep self-terminating and safe - already-deleted buildings and preview/`Temp`
entities are excluded, so a building is never processed twice and edit-mode ghosts
are never demolished. `RequireForUpdate(_abandonedBuildingQuery)` then gates
`OnUpdate` so the system is completely idle when no building is abandoned
(repo/AbandonedBuildingRemoverSystem.cs#L49).

`GetUpdateInterval` returns `16` for `GameSimulation` (and `1` otherwise), so even
when abandoned buildings exist the sweep runs on a throttled cadence rather than
every simulation frame (repo/AbandonedBuildingRemoverSystem.cs#L107).

### The cascade

The core idea: an abandoned building *owns* infrastructure through three
`DynamicBuffer` links - `SubArea` (lots/areas), `SubNet` (owned network segments),
and `SubLane` (owned lanes). Deleting only the building would orphan those child
entities. So the system walks each buffer and tags every referenced child with
`Deleted`, then tags the building itself. In the managed path that reads
(repo/AbandonedBuildingRemoverSystem.cs#L73-L100):

```csharp
if (EntityManager.TryGetBuffer<SubArea>(entity, false, out var subareas))
    foreach (var subArea in subareas)
        EntityManager.AddComponent<Deleted>(subArea.m_Area);

if (EntityManager.TryGetBuffer<SubNet>(entity, false, out var subnets))
    foreach (var net in subnets)
        EntityManager.AddComponent<Deleted>(net.m_SubNet);

if (EntityManager.TryGetBuffer<SubLane>(entity, false, out var sublanes))
    foreach (var lane in sublanes)
        EntityManager.AddComponent<Deleted>(lane.m_SubLane);

EntityManager.AddComponent<Deleted>(entity);
```

Every operation is `AddComponent<Deleted>` - a pure tag add. Nothing is read back,
nothing is destroyed inline, and the building is tagged last. Because a `Deleted`
tag is a set-membership marker (adding it twice is a no-op) the exact order of the
three buffer scans is irrelevant. That is the property the mod leans on to justify
its dual implementation.

### The dual path

The same logic exists twice, selected at **compile time** by the `USE_BURST`
preprocessor symbol, which the project defines whenever `BurstCompile` is on
(repo/AbandonedBuildingRemover.csproj#L8, repo/AbandonedBuildingRemover.csproj#L24-L26):

- **Burst path** (`#if USE_BURST`): `OnUpdate` fills an
  `AbandonedBuildingRemoverJob` (a `[BurstCompile] struct : IJob`) with an
  `EndFrameBarrier` command buffer, an async chunk list, and three `BufferLookup`s,
  then `Schedule`s it single-threaded and registers the handle with the barrier
  (repo/AbandonedBuildingRemoverSystem.cs#L54-L65, repo/AbandonedBuildingRemoverSystem.cs#L110-L157).
- **Managed path** (`#if !USE_BURST`): the direct `EntityManager` loop shown above,
  run on the main thread with a `ToEntityArray(Allocator.Temp)`
  (repo/AbandonedBuildingRemoverSystem.cs#L67-L104).

`OnCreate` reports which path is compiled in, logging `"Using Burst"` (and caching
the `EndFrameBarrier`) or `"NOT using Burst"`
(repo/AbandonedBuildingRemoverSystem.cs#L42-L47) - a self-report so a log tells you
which build shipped.

Note the child-scan order actually **differs** between the two paths: the managed
loop scrubs SubArea -> SubNet -> SubLane
(repo/AbandonedBuildingRemoverSystem.cs#L73-L98) while the Burst job scrubs
SubArea -> SubLane -> SubNet
(repo/AbandonedBuildingRemoverSystem.cs#L129-L151). This is harmless exactly
because each step only *adds* a `Deleted` tag - a vivid demonstration of why
idempotent tagging makes order-independence a feature rather than a bug.

### Deferred vs immediate structural change

The two paths also differ in *when* the structural change lands:

- The Burst path defers everything through an `EndFrameBarrier`
  `EntityCommandBuffer` (`_endFrameBarrier.CreateCommandBuffer()`,
  repo/AbandonedBuildingRemoverSystem.cs#L57), so no structural change happens
  inside the job - it is all replayed at the end-of-frame barrier. It captures the
  chunks with `ToArchetypeChunkListAsync(World.UpdateAllocator.ToAllocator, out _)`
  and reads children through `BufferLookup<SubArea/SubNet/SubLane>`
  (repo/AbandonedBuildingRemoverSystem.cs#L58-L61), then
  `AddJobHandleForProducer(handle)` so the barrier waits on the job
  (repo/AbandonedBuildingRemoverSystem.cs#L63).
- The managed path mutates the `EntityManager` immediately inside `OnUpdate`.

Crucially the job is `IJob`, scheduled single-threaded - not `IJobChunk` /
`ScheduleParallel`. For a low-frequency sweep of a handful of abandoned buildings
that is the right call: the cost is dominated by the structural changes the barrier
replays, not by iterating the (usually tiny) result set, so parallelism would add
race-management complexity for no real throughput gain.

## Techniques demonstrated

- [ECS structural cascade deletion of an entity + its sub-graph](../how-to/recipes/ecs-cascade-deletion.md)
  (family AF) - the headline technique: an `EntityQuery` for
  `Abandoned`+`Building` (excluding `Deleted`/`Temp`) drives an
  `AddComponent<Deleted>` on the building **and** every `SubArea.m_Area`,
  `SubNet.m_SubNet`, and `SubLane.m_SubLane` child, leaving vanilla cleanup to
  finish (repo/AbandonedBuildingRemoverSystem.cs#L28-L40,
  repo/AbandonedBuildingRemoverSystem.cs#L73-L100,
  repo/AbandonedBuildingRemoverSystem.cs#L129-L153).
- [Burst IJobChunk jobs](../how-to/recipes/burst-ijobchunk.md) (family S) - the
  deferred branch: a `[BurstCompile] IJob` reads children via `BufferLookup` over a
  `ToArchetypeChunkListAsync` chunk list and writes only through an `EndFrameBarrier`
  `EntityCommandBuffer`, with the producer handle registered on the barrier. Note
  the concrete type here is single-threaded `IJob`, not `IJobChunk`/`ScheduleParallel`
  (repo/AbandonedBuildingRemoverSystem.cs#L54-L65,
  repo/AbandonedBuildingRemoverSystem.cs#L110-L157).
- [Periodic UpdateInterval system](../how-to/recipes/periodic-updateinterval-system.md)
  - cheap-idle scheduling: `RequireForUpdate` on the query plus
  `GetUpdateInterval => phase == GameSimulation ? 16 : 1` keeps the sweep dormant
  until there is work and throttled when there is
  (repo/AbandonedBuildingRemoverSystem.cs#L49, repo/AbandonedBuildingRemoverSystem.cs#L107).

Supporting technique on display (not a headline family): a **compile-time dual
path** - one `USE_BURST` MSBuild define gates two `#if` implementations of the same
logic, with a per-path `OnCreate` self-report log
(repo/AbandonedBuildingRemover.csproj#L8, repo/AbandonedBuildingRemover.csproj#L24-L26;
repo/AbandonedBuildingRemoverSystem.cs#L42-L47).

## Key decisions & tradeoffs

- **Tag `Deleted`, do not immediately destroy.** The mod never calls a destroy API;
  it only adds a `Deleted` tag and lets the game react
  (repo/AbandonedBuildingRemoverSystem.cs#L100). This is the load-bearing decision:
  it makes the mod compatible with every other system that watches `Deleted`
  (vanilla refund/economy/notification cleanup, and third-party bulldoze mods), and
  it makes each write idempotent so ordering and repetition are harmless.
- **Cascade the whole owned sub-graph, not just the building.** Explicitly tagging
  `SubArea` / `SubNet` / `SubLane` children prevents orphaned lots, network
  segments, and lanes surviving the building's removal
  (repo/AbandonedBuildingRemoverSystem.cs#L73-L98).
- **Single-threaded `IJob`, deferred via barrier.** Structural changes are batched
  onto an `EndFrameBarrier` command buffer rather than mutating live inside the job,
  and the job is scheduled single-threaded because the workload is tiny and
  structural-change-bound (repo/AbandonedBuildingRemoverSystem.cs#L57-L64).
- **Compile-time, not runtime, path selection.** Burst vs managed is chosen by an
  MSBuild define, so the shipped binary is a single code path with no runtime branch;
  the tradeoff is that "which path shipped" is only visible in the `OnCreate` log,
  not toggleable by the player (repo/AbandonedBuildingRemover.csproj#L24-L26,
  repo/AbandonedBuildingRemoverSystem.cs#L42-L47).
- **Continuous sweep over one-shot.** Scheduling `UpdateAfter(GameSimulation)`
  instead of the commented `Deserialize` alternative means the mod catches buildings
  that become abandoned *during* play, not only those present at load
  (repo/Mod.cs#L19-L20).

## Pitfalls / upstream-watch

- **Duplicate-`Deleted` when stacked with bulldoze mods.** Running alongside Better
  Bulldozer / Delete Everything can cause the same entity to be tagged `Deleted` by
  both mods. This is benign - the second `AddComponent<Deleted>` is a no-op - and is
  a direct consequence of the idempotent-tag design
  (repo/AbandonedBuildingRemoverSystem.cs#L100).
- **Mid-sweep churn / no re-check.** The result set is captured (chunk list or
  entity array) and then replayed; a building that is *repaired* between capture and
  ECB playback is still tagged `Deleted`. There is no re-validation between capture
  and write (repo/AbandonedBuildingRemoverSystem.cs#L58,
  repo/AbandonedBuildingRemoverSystem.cs#L68). `Needs Verification (in-game)`:
  whether the abandoned->repaired transition is fast enough in practice to observe
  this.
- **Recompile-provenance drift.** The published `PublishConfiguration.xml` still
  carries the *original* mod's identity - `ModId 75107`, `ModVersion 1.0.0`,
  `GameVersion 1.1.*` - even though this Deviance-Fix source is a later recompile
  targeting a newer game (repo/Properties/PublishConfiguration.xml#L3-L28). Treat the
  XML `GameVersion`/`ModId` as inherited metadata, not proof of the version this
  source actually runs against; verify the true target from the build environment,
  not the manifest.
- **`GameVersion 1.1.*` mismatch surfaces at load.** Because the manifest advertises
  `1.1.*` while the code compiles against current assemblies, the game's version-gate
  and any consumer reading that field will see a value that does not match the binary
  (repo/Properties/PublishConfiguration.xml#L28). `Needs Verification (in-game)`:
  whether the live loader warns on or ignores this manifest mismatch.
- **No save data, safe removal.** The mod adds no components to savegames and is
  designed to be removable at any time (per its own long description,
  repo/Properties/PublishConfiguration.xml#L10-L16) - a property that follows from
  tagging only vanilla `Deleted` and never introducing a custom component.

## Source pointers

- Dossier:
  `../../vice-and-order-research/mods/dossiers/abandoned-building-remover-deviance-fix/`
  (`index.md`, `source.md`, `modding.md`, `guide.md`, `notes/`).
- Repo @ `a515bfe588965cbc988cba6a974242f66e5386b7`, key files:
  - `repo/AbandonedBuildingRemoverSystem.cs` - the whole mod: query, dual path,
    cascade, `IJob`, `GetUpdateInterval`.
  - `repo/Mod.cs` - `OnLoad` scheduling (`GameSimulation` vs commented `Deserialize`).
  - `repo/AbandonedBuildingRemover.csproj` - `USE_BURST` define wiring.
  - `repo/Properties/PublishConfiguration.xml` - published identity / version metadata.
