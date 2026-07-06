---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: Harmony postfix on a vanilla System.OnUpdate to inject ECS writes"
recipe: harmony-system-onupdate-postfix
technique_family: "AO - Harmony postfix on a vanilla System.OnUpdate to inject ECS writes"
diataxis: how-to
source_version: "~1.4.x (market-based-economy@b83f196; date-pinned, static source only)"
last_reverified: "2026-07-05"
status: source-verified
canonical_mods:
  - market-based-economy@b83f196a36bc74388accebdeb7c81f0f35dbab37
  - realistic-path-finding@50645fa6a078181365e36a42e2e27b96699bf02a
technique_applicability: [economy, simulation]
Created: 2026-07-05
Updated: 2026-07-05
Owners:
  - codex
---

# Harmony postfix on a vanilla System.OnUpdate to inject ECS writes

> Run your own ECS writes at a known point in the frame - right after a vanilla
> `GameSystemBase` has finished its `OnUpdate` - by Harmony-postfixing that
> `OnUpdate` and committing your changes through an `EndFrameBarrier` command buffer.

## Problem
You want to enforce your own rule on entity component data that a vanilla system
owns and rewrites every tick - for example raising each company's
`WorkProvider.m_MaxWorkers` to a prefab-derived floor. A prefab-field override
(family A) will not help: the value is per-instance runtime state, not authoring
data, and the vanilla system overwrites it on its own schedule. You need to land
*after* the vanilla system has done its pass, read the entities it just touched,
and write your correction - without racing its in-flight jobs.

## Solution
Postfix the whole `OnUpdate` of the target system. A postfix on `OnUpdate` runs on
the main thread once the system's update returns, giving you a stable point to open
an `EntityQuery`, read the current component values, compute your correction, and
commit it. The one hard rule: **do not mutate entities directly inside the postfix**
- acquire the existing `EndFrameBarrier` and record your writes into *its* command
buffer, so they play back at the frame barrier alongside the game's own deferred
structural changes instead of fighting the system's still-running jobs. Resolve the
target `OnUpdate` by reflection (`AccessTools.Method(type, "OnUpdate")`) because it
is a non-public override.

## Steps & Code

### 1. Resolve the vanilla `OnUpdate` and attach the postfix

`OnUpdate` is a protected override, so look it up by name via `AccessTools` rather
than a `MethodInfo` you cannot reference. Guard both the target and your postfix so
a signature change is a logged no-op, not a crash:

```csharp
var target = AccessTools.Method(typeof(WorkProviderSystem), "OnUpdate");
var postfix = AccessTools.Method(typeof(HarmonyBridge), nameof(WorkProviderOnUpdatePostfix));

if (target == null || postfix == null)
{
    Log.Warn("Workforce maintenance patch target or postfix not found; skipping.");
    return;
}

HarmonyInstance.Patch(target, postfix: new HarmonyMethod(postfix));
```
Source: `../../../vice-and-order-research/mods/dossiers/market-based-economy/repo/Harmony/HarmonyBridge.cs#L142-L151` (@b83f196a36bc74388accebdeb7c81f0f35dbab37)

This runs from `ApplyAll` alongside the attribute-driven `PatchAll` pass; the manual
`Patch` call is needed here precisely because the target is resolved by string name
(`HarmonyBridge.cs#L32-L36`).

### 2. Keep the postfix itself trivial - hand off to a manager

The postfix receives the patched system instance as `__instance`. Do no work in the
postfix body beyond forwarding to a helper that holds the logic; the `?.` makes a
not-yet-initialised manager a safe skip:

```csharp
private static void WorkProviderOnUpdatePostfix(WorkProviderSystem __instance)
{
    Economy.WorkforceUtilizationManager.Instance?.ApplyPostUpdate(__instance);
}
```
Source: `../../../vice-and-order-research/mods/dossiers/market-based-economy/repo/Harmony/HarmonyBridge.cs#L222-L225` (@b83f196a36bc74388accebdeb7c81f0f35dbab37)

### 3. Query the entities the system just touched

Inside the helper, use the patched system's `EntityManager` and lookups so you share
its dependency tracking. Bail early if the system is disabled or the query is empty
- the postfix fires every tick and you do not want per-frame allocation churn on an
idle query:

```csharp
var workProviderQuery = entityManager.CreateEntityQuery(new EntityQueryDesc
{
    All = new[]
    {
        ComponentType.ReadWrite<WorkProvider>(),
        ComponentType.ReadOnly<PrefabRef>()
    },
    Any = new[] { ComponentType.ReadOnly<CompanyData>() },
    None = Array.Empty<ComponentType>()
});
using var entities = workProviderQuery.ToEntityArray(Allocator.TempJob);
using var providers = workProviderQuery.ToComponentDataArray<WorkProvider>(Allocator.TempJob);
```
Source: `../../../vice-and-order-research/mods/dossiers/market-based-economy/repo/Economy/WorkforceUtilizationManager.cs#L43-L65` (@b83f196a36bc74388accebdeb7c81f0f35dbab37)

### 4. Acquire the EndFrameBarrier ECB - do NOT write entities directly

This is the crux. Instead of `entityManager.SetComponentData(...)` mid-postfix, grab
the game's existing `EndFrameBarrier` and record into its command buffer so the write
plays back at the frame boundary:

```csharp
var commandBuffer = World.DefaultGameObjectInjectionWorld
    .GetExistingSystemManaged<EndFrameBarrier>().CreateCommandBuffer();

for (int i = 0; i < entities.Length; i++)
{
    var entity = entities[i];
    var provider = providers[i];
    // ... compute correction, mutate the local 'provider' copy ...
    commandBuffer.SetComponent(entity, provider);
}
```
Source: `../../../vice-and-order-research/mods/dossiers/market-based-economy/repo/Economy/WorkforceUtilizationManager.cs#L72-L95` (@b83f196a36bc74388accebdeb7c81f0f35dbab37)

`GetExistingSystemManaged` (not `GetOrCreate`) is correct: the barrier already exists
in the default world, and recording into it means your `SetComponent` is applied at
the same barrier the simulation uses for its own deferred structural changes.

### 5. Compute the correction on the local copy, then let the ECB commit it

The actual rule lives outside the loop. Here MBE raises `m_MaxWorkers` to a floor
only when it is below that floor - a one-way clamp on the local struct copy, which
step 4 then commits:

```csharp
private void ApplyUtilization(Entity workplaceEntity, DynamicBuffer<Employee> employees,
    ref WorkProvider workProvider, int minimumCompanyWorkers)
{
    if (minimumCompanyWorkers > 0 && workProvider.m_MaxWorkers < minimumCompanyWorkers)
    {
        workProvider.m_MaxWorkers = minimumCompanyWorkers;
    }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/market-based-economy/repo/Economy/WorkforceUtilizationManager.cs#L100-L109` (@b83f196a36bc74388accebdeb7c81f0f35dbab37)

The floor itself is **prefab-derived**, not a constant: MBE reads the building's
`BuildingData`, `BuildingPropertyData`, and the company's `IndustrialProcessData` off
the prefab, computes fitting workers, and returns `math.max(1, fittingWorkers / 4)`
(`WorkforceUtilizationManager.cs#L111-L159`). So the postfix pushes a
prefab-shaped minimum back onto runtime component data the vanilla system owns.

## Pitfalls & gotchas

- **Never mutate entities directly inside the postfix.** The vanilla `OnUpdate` may
  still have scheduled jobs or deferred structural changes in flight; writing with
  `EntityManager.SetComponentData` from the postfix can race them or force a sync
  point. Record into the `EndFrameBarrier` command buffer instead
  (`WorkforceUtilizationManager.cs#L72,#L95`) so writes play back at the frame
  boundary. This is the single reason this recipe is distinct from a plain price
  getter postfix.

- **The postfix runs every tick - guard for cost.** An `OnUpdate` postfix fires on
  the target system's full cadence. MBE returns early when the system is disabled
  (`WorkforceUtilizationManager.cs#L38-L41`) or the entity array is empty
  (`WorkforceUtilizationManager.cs#L67-L70`) before allocating the command buffer at
  `WorkforceUtilizationManager.cs#L72`. Do the cheap checks first.

- **`OnUpdate` is non-public - resolve it by name.** You cannot take a `MethodInfo`
  for a protected override directly; `AccessTools.Method(typeof(System), "OnUpdate")`
  (`HarmonyBridge.cs#L142`) is the reliable resolve. Because it is a string lookup, a
  vanilla rename/signature change returns `null` - MBE logs and skips rather than
  crashing (`HarmonyBridge.cs#L145-L148`).

- **Prefab-side cost vs path-side cost (why this technique is a last resort).** MBE's
  own design rule: prefer mutating **prefab data** so the game's Burst systems pick
  the change up on their next rebuild, and reserve Harmony for the per-query getters
  and per-update injection points you cannot reach that way. Postfixing a
  `System.OnUpdate` to write ECS data is the heavier tool; use it only when the value
  is runtime state the vanilla system rewrites every tick (like `m_MaxWorkers`), not
  authoring data you could have edited on the prefab.

- **Clamp one-way to stay idempotent.** MBE only raises `m_MaxWorkers` when it is
  below the floor (`WorkforceUtilizationManager.cs#L102-L107`); it never scales the
  live value, so re-running the postfix every tick converges instead of drifting.

- The exact ordering guarantee between this `EndFrameBarrier` playback and other
  mods' postfixes on the same system is **Needs Verification (in-game)** - not proven
  from this source.

## Variations

- **Postfix a per-query getter instead of a whole `OnUpdate` (lighter, no ECB).**
  When the value you want to bias is read through a static utility getter rather than
  written by a system, postfix the getter and adjust `__result` in place - no command
  buffer, no query. Realistic Path Finding postfixes `PathUtils.GetCarDriveSpecification`
  (both overloads, disambiguated with an explicit `Type[]`) and `GetTaxiDriveSpecification`,
  gating the added cost on a lane flag:

  ```csharp
  public static void AddPenalty(ref PathSpecification specification, in NetCarLane carLane)
  {
      float sec = Seconds;
      if (sec <= 0f) return;
      if ((carLane.m_Flags & Game.Net.CarLaneFlags.PublicOnly) != 0)
      {
          PathUtils.TryAddCosts(ref specification.m_Costs,
              new PathfindCosts { m_Value = new float4(sec, 0f, 0f, 0f) });
      }
  }
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Patches/BusLanePatches.cs#L22-L32` (@50645fa6a078181365e36a42e2e27b96699bf02a)

  Overloaded targets must be pinned by argument types - RPF passes the full
  `new[] { typeof(NetCurve), typeof(NetCarLane), ... }` array to the `[HarmonyPatch]`
  attribute so Harmony binds the right overload
  (`BusLanePatches.cs#L42-L54`, `#L64-L77`, `#L86-L97`). This is the "per-query
  getter" side of MBE's rule: cheap, allocation-free, and it never touches the ECS
  structural graph.

- **Attribute patch vs manual `Patch` call.** RPF uses `[HarmonyPatch]` classes swept
  up by `PatchAll`; MBE resolves `OnUpdate` by string and calls `HarmonyInstance.Patch`
  directly (`HarmonyBridge.cs#L142-L151`). Reach for the manual call when the target
  is non-public or resolved dynamically.

## See also
- Related recipes: [Harmony price-getter postfix](harmony-price-getter-postfix.md)
  (family C - the per-query getter variant above, in full).
- Explanation: [Harmony patching](../../explanation/harmony-patching.md) (when to
  patch vs mutate prefab data).
- Reference: [Economy systems](../../reference/game-systems/economy.md)
  (`WorkProviderSystem`, `WorkProvider`).
- Case study demonstrating it: [market-based-economy](../../case-studies/market-based-economy.md).

## Sources
- Canonical mods (dossier + repo):
  - `market-based-economy` @b83f196a36bc74388accebdeb7c81f0f35dbab37 - `repo/Harmony/HarmonyBridge.cs`, `repo/Economy/WorkforceUtilizationManager.cs`
  - `realistic-path-finding` @50645fa6a078181365e36a42e2e27b96699bf02a - `repo/RealisticPathFinding/Patches/BusLanePatches.cs`
- Official/community references (link out, do not duplicate):
  https://harmony.pardeike.net/ , https://cs2.paradoxwikis.com/Modding
