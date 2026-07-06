---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: ECS query rewriting (Exclude / WithAll / WithNone)"
recipe: ecs-query-rewriting
technique_family: "T - ECS query rewriting (Exclude / WithAll / WithNone)"
diataxis: how-to
source_version: "~1.5.10f1 (traffic-tool-essentials@1097359; date-pinned, static source only)"
last_reverified: "2026-07-04"
status: source-verified
canonical_mods:
  - traffic-tool-essentials@10973595ac9ed37f47ca24eb788d4421dc295fa0
technique_applicability: [core]
Created: 2026-07-04
Updated: 2026-07-04
Owners:
  - codex
---

# ECS query rewriting (Exclude / WithAll / WithNone)

> Make a vanilla system stop touching the entities you own by rewriting its private
> `EntityQuery` to add a `WithNone<YourMarker>()` filter - no Harmony, no method patch.

## Problem
You want a vanilla ECS system to keep running normally but ignore the specific
entities your mod has taken over. In Traffic Tool Essentials this is the traffic-light
problem: the game's `TrafficLightInitializationSystem` and `TrafficLightSystem` should
keep driving every stock intersection, but must NOT overwrite the intersections the
mod is controlling with its own custom phases. Disabling the vanilla systems outright
(see [disable-replace-vanilla-system](disable-replace-vanilla-system.md)) is too
blunt - it stops them for the whole map. You need to carve out only your entities.

## Solution
Tag every entity you own with a **marker component**, then rewrite the vanilla
system's `EntityQuery` so that marker becomes a `WithNone` (Exclude) filter. The
vanilla query now matches every intersection *except* yours. The rewrite is done by
reflection: read the system's private query field by name, rebuild an equivalent
query that preserves all the original constraints, append your `none` filter, and set
the field back. You also schedule your replacement systems `UpdateBefore` the vanilla
ones so your marker is in place before the vanilla query runs. This is the surgical,
narrowing alternative to disabling a vanilla system - and Traffic Tool Essentials does
the whole thing with **zero Harmony** (a `git grep` for `HarmonyPatch` at the pin
returns nothing).

## Steps & Code

### 1. Define a marker component for the entities you own

An `IComponentData` you add to the entities you take over is enough - what matters for
this technique is that it is a `ComponentType` you can put in a `WithNone` filter.

```csharp
public struct CustomTrafficLights : IComponentData, IQueryTypeParameter, ISerializable
```
Source: `../../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Components/CustomTrafficLights.cs#L6` (@10973595ac9ed37f47ca24eb788d4421dc295fa0)

### 2. Resolve the vanilla systems you want to narrow

Grab the managed vanilla system instances up front - you need the object whose private
query field you will rewrite.

```csharp
m_TrafficLightInitializationSystem =
    m_World.GetOrCreateSystemManaged<Game.Net.TrafficLightInitializationSystem>();
m_TrafficLightSystem =
    m_World.GetOrCreateSystemManaged<Game.Simulation.TrafficLightSystem>();
```
Source: `../../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Mod.cs#L122-L123` (@10973595ac9ed37f47ca24eb788d4421dc295fa0)

### 3. Build the `none` list and rewrite each system's query

Create a `NativeList<ComponentType>` holding your marker as `ReadOnly`, then hand the
system, the **private field name** of its query, and that list to the rewrite util.
Dispose the list after.

```csharp
var noneList = new NativeList<ComponentType>(1, Allocator.Temp);
noneList.Add(ComponentType.ReadOnly<Components.CustomTrafficLights>());

Utils.EntityQueryUtils.UpdateEntityQuery(m_TrafficLightInitializationSystem, "m_TrafficLightsQuery", noneList);
Utils.EntityQueryUtils.UpdateEntityQuery(m_TrafficLightSystem, "m_TrafficLightQuery", noneList);

// V205: Dispose noneList after use
noneList.Dispose();
```
Source: `../../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Mod.cs#L209-L216` (@10973595ac9ed37f47ca24eb788d4421dc295fa0)

Note the two field names differ: `"m_TrafficLightsQuery"` on the initialization
system, `"m_TrafficLightQuery"` (no `s`) on the simulation system. These are the
vanilla systems' actual private field names, reached by string - see the gotcha below.

### 4. Read the private query field by name (reflection)

`UpdateEntityQuery` is three moves: reflect the current `EntityQuery` out of the
private field, rebuild an equivalent query with your `none` filter appended, then
reflect the new query back into the same field.

```csharp
public static void UpdateEntityQuery(SystemBase systemBase, string fieldName, NativeList<ComponentType> none)
{
    EntityQuery query = GetEntityQuery(systemBase, fieldName);
    EntityQuery newQuery = GetEntityQueryBuilder(query, none).Build(systemBase);
    SetEntityQuery(systemBase, fieldName, newQuery);
}
```
Source: `../../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Utils/EntityQueryUtils.cs#L21-L26` (@10973595ac9ed37f47ca24eb788d4421dc295fa0)

The get/set are plain `FieldInfo` reflection with `NonPublic | Instance` binding:

```csharp
public static EntityQuery GetEntityQuery(object obj, string fieldName)
{
    FieldInfo fieldInfo = obj.GetType().GetField(fieldName, BindingFlags.NonPublic | BindingFlags.Instance);
    return (EntityQuery)fieldInfo.GetValue(obj);
}
```
Source: `../../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Utils/EntityQueryUtils.cs#L9-L13` (@10973595ac9ed37f47ca24eb788d4421dc295fa0)

### 5. Preserve the original constraints when rebuilding (do not clobber)

Do NOT rebuild the query from scratch with only your `none`. The util reads every
`EntityQueryDesc` off the old query and re-applies all six constraint buckets
(`Any / None / All / Disabled / Absent / Present`) before adding your new filters, so
the vanilla query keeps matching exactly what it did - minus your entities. It also
handles multi-desc queries via `AddAdditionalQuery()`.

```csharp
var descArray = oldQuery.GetEntityQueryDescs();
for (int i = 0; i < descArray.Length; i++)
{
    EntityQueryDesc desc = descArray[i];
    var oldNone = CreateNativeList(desc.None, Allocator.Temp);
    var oldAll  = CreateNativeList(desc.All,  Allocator.Temp);
    // ... oldAny/oldDisabled/oldAbsent/oldPresent likewise ...
    builder.WithNone(ref oldNone);     // carry the vanilla constraints over
    builder.WithAll(ref oldAll);
    builder.WithNone(ref none);        // then append YOUR exclude filter
    // ... dispose old* lists ...
    if (i < descArray.Length - 1) { builder.AddAdditionalQuery(); }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Utils/EntityQueryUtils.cs#L49-L83` (@10973595ac9ed37f47ca24eb788d4421dc295fa0)

### 6. Schedule your replacement systems BEFORE the vanilla ones

The rewrite only helps if your marker is already on the entity when the vanilla query
runs. Order your patched systems `UpdateBefore` the vanilla systems in the matching
phase so ownership is settled first.

```csharp
updateSystem.UpdateBefore<...Initialisation.PatchedTrafficLightInitializationSystem,
    Game.Net.TrafficLightInitializationSystem>(SystemUpdatePhase.Modification4B);
updateSystem.UpdateBefore<...Simulation.PatchedTrafficLightSystem,
    Game.Simulation.TrafficLightSystem>(SystemUpdatePhase.GameSimulation);
```
Source: `../../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Mod.cs#L218-L219` (@10973595ac9ed37f47ca24eb788d4421dc295fa0)

## Pitfalls & gotchas

- **Reflection-by-name is brittle across game versions.** The whole technique hinges
  on the vanilla query being stored in a private field with the exact name you pass as
  a string (`"m_TrafficLightsQuery"`, `"m_TrafficLightQuery"`). `GetField(...)` returns
  `null` if the game renames, removes, or re-types that field in a patch, and the very
  next `fieldInfo.GetValue(obj)` throws `NullReferenceException` - there is no null
  guard in the canonical `GetEntityQuery`
  (`EntityQueryUtils.cs#L11-L12`). Re-verify these field names every CS2 release, and
  consider guarding the reflection yourself.

- **The two field names are not the same.** The initialization system's field is
  plural (`m_TrafficLightsQuery`); the simulation system's is singular
  (`m_TrafficLightQuery`). Copy-pasting one name to both silently narrows the wrong
  system (or nothing). Confirm each vanilla system's actual field name at the pin
  before wiring it (`Mod.cs#L212-L213`).

- **Rebuild-preserving, not rebuild-from-scratch.** If you skip step 5 and build a
  query with only your `none`, you drop every original constraint and the vanilla
  system starts matching far too many (or too few) entities. The util deliberately
  reconstructs all six buckets from `GetEntityQueryDescs()` first
  (`EntityQueryUtils.cs#L49-L70`); do the same.

- **Marker must be applied before the vanilla query runs (ordering, step 6).** The
  exclude filter only removes entities that already carry the marker at query time. If
  your system that adds `CustomTrafficLights` runs *after* the vanilla system in the
  same frame, the vanilla system still processes the entity that frame. The system
  ordering via `UpdateBefore` (`Mod.cs#L218-L219`) is what closes that gap.

- **`Allocator.Temp` lists must be disposed.** Every `NativeList` here (`noneList`,
  the `empty` helper, and all `old*` lists) is `Allocator.Temp` and explicitly
  disposed after use (`Mod.cs#L216`, `EntityQueryUtils.cs#L33`, `#L72-L78`). Leaking
  them triggers the Collections safety/leak warnings.

- **Runtime correctness (in-game).** That the rewritten query actually causes the
  vanilla traffic-light systems to skip marked intersections while still driving all
  others - and that no visual flicker/desync occurs on the frame ownership transfers -
  is a runtime-behavioural claim not visible in source: `Needs Verification (in-game)`.

## Variations

- **Include instead of exclude (`WithAll`).** The same util's full overload takes an
  `all` list alongside `none` (`EntityQueryUtils.cs#L37-L46, #L67`). Pass a component
  in `all` to *narrow* a vanilla query to only entities carrying your tag, rather than
  excluding them - useful when you want a vanilla system to act on just your subset.

- **Any / Disabled / Absent / Present buckets.** The full `GetEntityQueryBuilder`
  overload exposes all six ECS constraint buckets
  (`EntityQueryUtils.cs#L37-L46`); the single-arg `none` convenience
  (`EntityQueryUtils.cs#L28-L35`) just passes empty lists for the other five. Use the
  full overload when you need a compound rewrite.

- **Narrow vs. disable.** When you want the vanilla system gone entirely rather than
  filtered, disable/replace it wholesale instead of rewriting its query - see
  [disable-replace-vanilla-system](disable-replace-vanilla-system.md). Query rewriting
  is the surgical middle ground: vanilla keeps running for everything you do not own.

## See also
- Related recipes: [disable-replace-vanilla-system](disable-replace-vanilla-system.md)
  (the blunt alternative this narrows).
- Explanation: [system-replacement](../../explanation/system-replacement.md),
  [ecs-fundamentals](../../explanation/ecs-fundamentals.md) (why marker components and
  `EntityQuery` filters work).
- Case studies demonstrating it:
  [time2work-realistic-trips](../../case-studies/time2work-realistic-trips.md).

## Sources
- Canonical mods (dossier + repo):
  - `traffic-tool-essentials` @10973595ac9ed37f47ca24eb788d4421dc295fa0 - `repo/TrafficToolEssentials/Mod.cs`, `repo/TrafficToolEssentials/Utils/EntityQueryUtils.cs`, `repo/TrafficToolEssentials/Components/CustomTrafficLights.cs`
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding
