---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: Custom polygon-zone tool + save-stable conflict resolution"
recipe: polygon-zone-tool
technique_family: "BB - Custom polygon-zone tool + save-stable conflict resolution"
diataxis: how-to
source_version: "~1.5.x (traffic-tool-essentials@1097359; date-pinned, static source only)"
last_reverified: "2026-07-05"
status: source-verified
canonical_mods:
  - traffic-tool-essentials@10973595ac9ed37f47ca24eb788d4421dc295fa0
technique_applicability: [tooling, infrastructure]
Created: 2026-07-05
Updated: 2026-07-05
Owners:
  - codex
---

# Custom polygon-zone tool + save-stable conflict resolution

> Let a player draw an arbitrary closed polygon on the map, reject self-crossing or
> overlapping shapes at creation time, and resolve any residual overlap
> deterministically so the same depot always lands in the same zone across save/load.

## Problem
You are building a tool where the player defines a **region** by clicking vertices -
a service-catchment area, a jurisdiction, a "these depots belong to this zone" fence -
not a rectangular drag-select and not a linear waypoint path. Two hard parts follow.
First, the geometry: you need point-in-polygon and segment-intersection tests to know
what a shape encloses and to reject a shape that crosses itself or an existing zone.
Second, and easy to miss: when two zones legitimately or accidentally overlap and a
depot sits inside both, *which zone wins?* If you answer that by iteration order over
ECS chunks, the answer changes after a save/load and the depot silently migrates.

## Solution
Store each zone as an entity carrying a `DepotZonePoint` dynamic buffer (the ordered
polygon vertices) plus a `DepotZoneData` component holding a stable integer
`m_ZoneId`. At **creation** run three cheap CPU geometry checks against every existing
zone (vertex-in-me, my-vertex-in-you, edge-crosses-edge) and reject on any hit, so
crossing and overlapping shapes never get committed. For the residual case where
overlaps still exist, when you build the depot->zone assignment cache **sort the zones
by `m_ZoneId` ascending first** and let the lowest id claim each contested depot. That
makes "which zone wins" a pure function of a persisted integer, not of chunk
iteration order - which is *not* stable across save/load.

## Steps & Code

### 1. Point-in-polygon: even-odd ray cast in the X/Z plane

The workhorse. A horizontal ray-cast (Jordan curve / even-odd rule) over the polygon
edges, using world X and Z (the map ground plane), ignoring Y:

```csharp
public static bool IsPointInPolygon(float3 point, List<float3> polygon)
{
    if (polygon == null || polygon.Count < 3) return false;

    bool inside = false;
    int j = polygon.Count - 1;

    for (int i = 0; i < polygon.Count; i++)
    {
        float xi = polygon[i].x, zi = polygon[i].z;
        float xj = polygon[j].x, zj = polygon[j].z;

        if (((zi > point.z) != (zj > point.z)) &&
            (point.x < (xj - xi) * (point.z - zi) / (zj - zi) + xi))
        {
            inside = !inside;
        }
        j = i;
    }
    return inside;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Systems/DepotZone/DepotZoneSystem.cs#L1399-L1421` (@10973595ac9ed37f47ca24eb788d4421dc295fa0)

### 2. Segment intersection: Cramer's rule on an OPEN interval

To detect self-crossing and cross-shaped overlaps you need a proper segment-segment
test. This one parameterizes both segments and solves for `t`, `u` via the
determinant, treating parallel (`cross ~ 0`) as no-hit and - critically - requiring
both parameters strictly inside `(eps, 1-eps)` so that mere endpoint *touches* (shared
polygon vertices) do not count as intersections:

```csharp
float cross = d1x * d2z - d1z * d2x;
if (math.abs(cross) < 1e-6f)
    return false;                       // parallel -> no proper crossing

float t = (dx * d2z - dz * d2x) / cross;
float u = (dx * d1z - dz * d1x) / cross;

// Open interval excludes endpoint touches (adjacent edges share a vertex)
const float eps = 1e-6f;
return t > eps && t < (1.0f - eps) && u > eps && u < (1.0f - eps);
```
Source: `../../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Systems/DepotZone/DepotZoneSystem.cs#L1439-L1457` (@10973595ac9ed37f47ca24eb788d4421dc295fa0)

The open interval is the whole trick: adjacent polygon edges legitimately share an
endpoint, and a closed-interval test would flag every polygon as "self-intersecting."

### 3. Polygon-vs-polygon edge test (brute O(n*m))

Layer the segment test into an all-edges-against-all-edges scan. The source is candid
that this is O(n*m) and fine only because zones are capped small (max 20 points, a
handful of zones):

```csharp
private static bool DoPolygonEdgesIntersect(List<float3> polygonA, List<float3> polygonB)
{
    int countA = polygonA.Count;
    int countB = polygonB.Count;

    for (int i = 0; i < countA; i++)
    {
        float3 a1 = polygonA[i];
        float3 a2 = polygonA[(i + 1) % countA];
        for (int j = 0; j < countB; j++)
        {
            float3 b1 = polygonB[j];
            float3 b2 = polygonB[(j + 1) % countB];
            if (DoLineSegmentsIntersect(a1, a2, b1, b2))
                return true;
        }
    }
    return false;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Systems/DepotZone/DepotZoneSystem.cs#L1465-L1487` (@10973595ac9ed37f47ca24eb788d4421dc295fa0)

### 4. Reject overlap at creation with THREE checks

Point-in-polygon alone misses cross/star overlaps where no vertex of either shape
lies inside the other. The overlap probe therefore runs three checks per existing
zone and returns the offending `m_ZoneId` on the first hit:

```csharp
// Check 1: any vertex of the NEW zone inside an EXISTING zone?
foreach (var newPoint in newZonePoints)
    if (IsPointInPolygon(newPoint, existingPolygon))
    { overlappingZoneId = existingZoneData.m_ZoneId; return true; }

// Check 2: any vertex of the EXISTING zone inside the NEW zone?
foreach (var existingPoint in existingPolygon)
    if (IsPointInPolygon(existingPoint, newZonePoints))
    { overlappingZoneId = existingZoneData.m_ZoneId; return true; }

// Check 3: any EDGE crosses any EDGE? (catches cross/star overlap, no vertex inside)
if (DoPolygonEdgesIntersect(newZonePoints, existingPolygon))
{ overlappingZoneId = existingZoneData.m_ZoneId; return true; }
```
Source: `../../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Systems/DepotZone/DepotZoneSystem.cs#L1533-L1560` (@10973595ac9ed37f47ca24eb788d4421dc295fa0)

`CreateZone` calls this before it commits anything, and bails out (returning
`Entity.Null`) on rejection so the crossing/overlapping shape is never persisted:

```csharp
// V401.41: Check for overlap with existing zones
if (DoesZoneOverlapWithExisting(points, out int overlappingZoneId))
{
    Mod.LogWarn($"[DepotZoneSystem] V401.41: CreateZone REJECTED - overlaps with zone {overlappingZoneId}");
    return Entity.Null;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Systems/DepotZone/DepotZoneSystem.cs#L525-L531` (@10973595ac9ed37f47ca24eb788d4421dc295fa0)

### 5. Resolve residual overlap by LOWEST m_ZoneId, not chunk order

Even with creation-time rejection, edited or migrated zones can still overlap, so the
depot->zone cache must break ties deterministically. Before assigning depots, sort the
zone entities by their persisted `m_ZoneId` ascending. The mod's own comment states
the reason: first-in-chunk-order used to win, but chunk order is not stable after
save/load, so a depot could change zones:

```csharp
// V426.97 (G4-1): ... Bei ueberlappenden Zonen gewinnt ein doppelt enthaltenes Depot
// bisher die ERSTE Zone in Chunk-Reihenfolge - diese ist aber nach Save/Load NICHT
// stabil ... Wir sortieren die Zonen hier nach m_ZoneId aufsteigend (stabiler int,
// nicht Entity) -> die KLEINSTE/aelteste Zone-ID gewinnt reproduzierbar.
var sortedZones = new List<Entity>(zones.Length);
for (int z = 0; z < zones.Length; z++) sortedZones.Add(zones[z]);
sortedZones.Sort((a, b) =>
{
    int idA = EntityManager.HasComponent<DepotZoneData>(a) ? EntityManager.GetComponentData<DepotZoneData>(a).m_ZoneId : int.MaxValue;
    int idB = EntityManager.HasComponent<DepotZoneData>(b) ? EntityManager.GetComponentData<DepotZoneData>(b).m_ZoneId : int.MaxValue;
    return idA.CompareTo(idB);
});
```
Source: `../../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Systems/DepotZone/DepotZoneSystem.cs#L416-L429` (@10973595ac9ed37f47ca24eb788d4421dc295fa0)

Then iterate the **sorted** zones and let the first (lowest-id) zone that contains a
depot claim it exclusively via an `assignedDepots` set; later zones skip an
already-claimed depot with a warning:

```csharp
if (IsPointInPolygon(depotPositions[d], polygon))
{
    if (assignedDepots.Contains(depots[d]))   // already taken by a lower zone id
    {
        Mod.LogWarn($"[DepotZoneSystem] V401.26: Depot {depots[d].Index} already assigned to another zone, skipping for zone {zoneData.m_ZoneId} (overlapping zones?)");
        overlapCount++;
        continue;
    }
    assignedDepots.Add(depots[d]);            // first (lowest-id) zone wins
    depotList.Add(depots[d]);
}
```
Source: `../../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Systems/DepotZone/DepotZoneSystem.cs#L461-L478` (@10973595ac9ed37f47ca24eb788d4421dc295fa0)

Because `m_ZoneId` is a monotonically assigned integer persisted in `DepotZoneData`
(`m_NextZoneId` is restored as `max(existing id) + 1` on load), the tie-break key
survives serialization and the winner is reproducible.
Source: `../../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Systems/DepotZone/DepotZoneSystem.cs#L1660-L1670` (@10973595ac9ed37f47ca24eb788d4421dc295fa0)

## Pitfalls & gotchas

- **Iteration-order tie-breaks are non-deterministic across save/load.** This is the
  headline. `EntityQuery` returns entities in ECS chunk order, and that order is *not*
  preserved across serialization. Any "first zone found wins" logic keyed on iteration
  order will re-assign contested depots after a load. Sort by a **persisted stable
  key** (here the integer `m_ZoneId`) before resolving. The mod's `V426.97` comment
  documents this exact bug and its fix
  (`DepotZoneSystem.cs#L416-L429`).

- **Point-in-polygon alone does not catch cross/star overlaps.** Two squares arranged
  as a plus sign overlap in the middle while no vertex of either lies inside the
  other. You need the edge-intersection check (Check 3) in addition to the two
  vertex-containment checks, or crossing shapes slip through
  (`DepotZoneSystem.cs#L1553-L1560`).

- **Use an OPEN interval in the segment test.** Adjacent polygon edges share a vertex;
  a closed-interval `[0,1]` test reports every polygon as self-intersecting at its own
  corners. The `(eps, 1-eps)` interval excludes endpoint touches
  (`DepotZoneSystem.cs#L1454-L1457`). Consequence: two zones that only *touch* at a
  single shared point are treated as non-overlapping.

- **Parallel/collinear segments return "no intersection."** The `math.abs(cross) <
  1e-6f` early-out means overlapping collinear edges are not detected as crossings
  (`DepotZoneSystem.cs#L1441-L1443`). For catchment fencing this is acceptable; if you
  need collinear-overlap detection you must add it yourself.

- **Everything is 2D in the X/Z plane; Y is ignored.** `IsPointInPolygon` reads only
  `.x` and `.z` (`DepotZoneSystem.cs#L1408-L1411`). On sloped or multi-level terrain
  the test is a top-down projection - fine for ground zones, wrong if you need true 3D
  containment.

- **Self-intersection is rejected on *edit*, not just create.** Vertex move/insert use
  a localized check around the moved vertex (`WouldCauseSelfIntersectionForEdit`, which
  tests only the two edges hanging off the moved point,
  `DepotZoneSystem.cs#L1257-L1285`); delete uses a full O(n^2) scan
  (`WouldPolygonSelfIntersect`, `DepotZoneSystem.cs#L1294-L1318`). If you add editing,
  wire the equivalent guard or edits can produce a shape creation would have rejected.

- **Small-N assumption is load-bearing.** The O(n*m) polygon test and O(n^2)
  self-intersection scan are only cheap because zones cap at 20 vertices
  (`CreateZone` truncates, `DepotZoneSystem.cs#L519-L523`). Raising the cap or adding
  many zones degrades quadratically; you would need a spatial index.

- Whether the resolved assignment is stable across save/load *in the running game* is
  consistent with the code but is `Needs Verification (in-game)`.

## Variations

- **Rectangular region instead of freeform polygon.** If your tool only needs an
  axis-aligned box, a two-corner drag with an AABB test is far cheaper than
  point-in-polygon; see [tool drag-select](tool-drag-select.md).

- **Linear path instead of enclosed area.** When the player is defining a route or a
  chain of stops rather than a filled region, use an ordered waypoint list, not a
  closed polygon; see [N-waypoint path tool](tool-nwaypoint-path.md).

- **Different stable tie-break keys.** Any persisted, comparable scalar works in place
  of `m_ZoneId` - a creation timestamp, an explicit priority field, or a name string.
  The invariant is *persisted + deterministic*, never `Entity.Index` (recycled) or
  iteration order (unstable across load).

- **Warn-and-skip vs. clip.** This mod resolves overlap by giving the whole contested
  depot to the lowest-id zone and warning. An alternative is to geometrically clip the
  overlapping area out of the newer zone; that is more work and not what the source
  does.

## See also
- Related recipes: [tool drag-select](tool-drag-select.md) (rectangular selection),
  [N-waypoint path tool](tool-nwaypoint-path.md) (linear path input).
- Reference: [raycast filters](../tooling/raycast-filters.md) (turning cursor hits into
  world positions for vertices), [technique index](../../technique-index.md).
- Case studies demonstrating it: [traffic-tool-essentials](../../case-studies/traffic-tool-essentials.md).

## Sources
- Canonical mods (dossier + repo):
  - `traffic-tool-essentials` @10973595ac9ed37f47ca24eb788d4421dc295fa0 - `repo/TrafficToolEssentials/Systems/DepotZone/DepotZoneSystem.cs`
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding
