---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: Custom ToolBaseSystem with N-waypoint path selection"
recipe: tool-nwaypoint-path
technique_family: "W - Custom ToolBaseSystem with N-waypoint path selection"
diataxis: how-to
source_version: "~1.5.10f1 (advanced-road-naming@559e72c; date-pinned, static source only)"
last_reverified: "2026-07-04"
status: source-verified
canonical_mods:
  - advanced-road-naming@559e72cdb3180e3e71869643367e094a22eed988
technique_applicability: [infrastructure]
Created: 2026-07-04
Updated: 2026-07-04
Owners:
  - codex
---

# Custom ToolBaseSystem with N-waypoint path selection

> Build an in-world tool that collects an ordered list of waypoints along the road
> network - one committed per left-click - and auto-paths the connected segments
> between each adjacent pair, so the user paints an arbitrarily long route rather than
> picking two endpoints.

## Problem
You need the player to select a **path**, not a single segment and not a rectangle:
"start here, go through here and here, end there". A drag-select (family V) gives you
a hovered set; a two-click endpoint tool gives you exactly A-to-B. Neither models a
multi-leg route where the user steers the path through named intermediate points and
the tool fills in the segments in between. You want each click to *commit* the hovered
segment as the next waypoint, accumulate them in order, and recompute the connected
segment list every time.

## Solution
Subclass `ToolBaseSystem`, restrict the raycast to the road network, and keep the
route state in a plain helper object (here `RouteSelectionController`) that owns an
ordered `List<RoadRouteWaypoint>`. Each frame, raycast the hovered segment/node into a
candidate waypoint. On a left-click, hand that hovered waypoint to
`TryAddWaypoint(...)`: the controller appends it and rebuilds the full segment list by
auto-pathing each consecutive waypoint pair. There is **no two-click endpoint mode and
no click that "finalizes"** - the route grows one waypoint per click, right-click pops
the last one, and a separate `Apply()` (driven from UI) consumes the accumulated
segments. This is a superset of [drag-select](tool-drag-select.md): the same raycast
plumbing, but the tool remembers an ordered history instead of a momentary hover set.

## Steps & Code

### 1. Create the tool, wire the selection controller, require the net layer

`OnCreate` builds the controller (passing in the validation + pathing services it will
use to reject off-road hits and to path between waypoints) and sets `requireNet` so the
base tool only activates over roads:

```csharp
protected override void OnCreate()
{
    base.OnCreate();
    _metadataSystem = World.GetOrCreateSystemManaged<SegmentMetadataSystem>();
    _selectionController = new RouteSelectionController(_metadataSystem.Validation, _metadataSystem.Pathing);
    _mode = RoadRouteToolMode.AssignMajorRouteNumber;
    // ...
    requireNet = Game.Net.Layer.Road;
    Mod.log.Info("RoadRouteToolSystem created");
}
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Systems/RoadRouteToolSystem.cs#L104-L114` (@559e72cdb3180e3e71869643367e094a22eed988)

### 2. Constrain the raycast to road segments in `InitializeRaycast`

Override `InitializeRaycast` to point `m_ToolRaycastSystem` at the net only. This is
what makes the hover land on road segments and nodes rather than terrain or buildings:

```csharp
public override void InitializeRaycast()
{
    base.InitializeRaycast();
    m_ToolRaycastSystem.collisionMask = _undergroundMode
        ? CollisionMask.Underground
        : CollisionMask.OnGround | CollisionMask.Overground;
    m_ToolRaycastSystem.typeMask = TypeMask.Net;
    m_ToolRaycastSystem.netLayerMask = Game.Net.Layer.Road;
    m_ToolRaycastSystem.raycastFlags = RaycastFlags.Markers | RaycastFlags.ElevateOffset | RaycastFlags.SubElements | RaycastFlags.OutsideConnections;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Systems/RoadRouteToolSystem.cs#L148-L156` (@559e72cdb3180e3e71869643367e094a22eed988)

See [raycast filters](../tooling/raycast-filters.md) for the full `typeMask` /
`netLayerMask` / `RaycastFlags` menu.

### 3. Turn the raycast hit into a candidate waypoint

Each frame's hover resolves the raycast hit into a `RoadRouteWaypoint`. Note the hit
can land on a **segment** or a **node**; a node is resolved to its nearest connected
edge, so both produce a segment-anchored waypoint:

```csharp
private bool TryGetSnappedWaypoint(out RoadRouteWaypoint waypoint)
{
    waypoint = default;
    Entity entity;
    RaycastHit hit;
    if (!GetRaycastResult(out entity, out hit))
        return false;

    if (_metadataSystem.Validation.IsValidRoadSegment(entity))
        return TryCreateWaypointOnSegment(entity, hit.m_Position, out waypoint);

    if (EntityManager.HasComponent<Node>(entity))
        return TryCreateWaypointFromNode(entity, hit.m_Position, out waypoint);

    return false;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Systems/RoadRouteToolSystem.cs#L879-L893` (@559e72cdb3180e3e71869643367e094a22eed988)

A `RoadRouteWaypoint` is just `(Entity Segment, float3 Position, float CurvePosition)`
- the segment it belongs to plus where along the curve the click landed
(`RoadRouteWaypoint.cs#L9-L20`).

### 4. Commit the hovered waypoint on each left-click

In `OnUpdate`, a left-press first tries to grab an *existing* waypoint for editing; if
the hover is not on an existing waypoint, it commits a **new** one. This is the core of
the N-waypoint model - every click that is not an edit appends a waypoint:

```csharp
if (leftClickPressed)
{
    Mod.log.Info(() => $"Road Naming: click received. Hovered={HoveredSegment.Index}, Waypoints={WaypointCount}");
    if (_selectionController.TryBeginEditFromHover())
    {
        _statusMessage = _selectionController.BuildRouteInstruction();
        // ... entered move/insert edit of an existing waypoint
    }
    else
    {
        TryAddHoveredWaypoint();   // <-- commit the hovered segment as the next waypoint
    }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Systems/RoadRouteToolSystem.cs#L818-L830` (@559e72cdb3180e3e71869643367e094a22eed988)

`TryAddHoveredWaypoint` reads the currently committed hover and forwards it to the
controller's `TryAddWaypoint`, logging the from/to waypoint counts:

```csharp
public bool TryAddHoveredWaypoint()
{
    var hoveredWaypoint = _selectionController.HoveredWaypoint;
    if (!hoveredWaypoint.HasValue) { /* warn, ignore click */ return false; }

    var previousWaypointCount = _selectionController.WaypointCount;
    var previousSegmentCount = _selectionController.SelectedSegments.Count;
    var added = _selectionController.TryAddWaypoint(hoveredWaypoint.Value);
    _statusMessage = added ? _selectionController.BuildRouteInstruction() : _selectionController.Warning;
    // ... log AppendedSegments = SelectedSegments.Count - previousSegmentCount
    return added;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Systems/RoadRouteToolSystem.cs#L270-L295` (@559e72cdb3180e3e71869643367e094a22eed988)

### 5. Append the waypoint and auto-path the new leg

`TryAddWaypoint` is where "N waypoints" becomes "a connected path". The first waypoint
just seeds the list; every subsequent one is added as a candidate and the whole route
is re-committed (which re-paths each adjacent pair):

```csharp
public bool TryAddWaypoint(RoadRouteWaypoint waypoint)
{
    Warning = null;
    if (!_validation.IsValidRoadSegment(waypoint.Segment)) { Warning = "..."; return false; }

    if (_waypoints.Count == 0)
    {
        _waypoints.Add(waypoint);
        _selectedSegments.Clear();
        AppendSegmentIfMissing(_selectedSegments, waypoint.Segment);
        return true;
    }

    var previousWaypoint = _waypoints[_waypoints.Count - 1];
    if (previousWaypoint.Segment == waypoint.Segment && math.distance(previousWaypoint.Position, waypoint.Position) < 1f)
    { Warning = "That waypoint is already the current route end."; return false; }

    var candidateWaypoints = new List<RoadRouteWaypoint>(_waypoints) { waypoint };
    return TryCommitCandidateWaypoints(candidateWaypoints, out _);
}
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Services/RouteSelectionController.cs#L167-L194` (@559e72cdb3180e3e71869643367e094a22eed988)

The commit walks consecutive waypoints and asks the pathing service for the segments
between each pair; if any leg has no connected path, the whole add is rejected:

```csharp
AppendSegmentIfMissing(segments, waypoints[0].Segment);
for (var i = 1; i < waypoints.Count; i++)
{
    if (!_validation.IsValidRoadSegment(waypoints[i].Segment)) { warning = "..."; return false; }

    var path = _pathing.FindPath(waypoints[i - 1].Segment, waypoints[i].Segment, AutoPathMaxDepth);
    if (path.Count == 0) { warning = "No connected road path found between adjacent waypoints."; return false; }

    AppendPath(segments, path);
}
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Services/RouteSelectionController.cs#L451-L468` (@559e72cdb3180e3e71869643367e094a22eed988)

### 6. Right-click undoes the last waypoint (there is no "finalize" click)

Right-click removes the hovered or last waypoint and rebuilds the segment list; there
is no click that ends the route. The accumulated `SelectedSegments` are consumed later
by a separate `Apply()` (invoked from the tool's UI):

```csharp
public void RemoveLastWaypoint()
{
    Warning = null;
    if (_waypoints.Count == 0) { Warning = "No waypoints to undo."; return; }

    _waypoints.RemoveAt(_waypoints.Count - 1);
    RebuildSelectedSegmentsFromWaypoints();
    RebuildPreviewState();
}
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Services/RouteSelectionController.cs#L202-L214` (@559e72cdb3180e3e71869643367e094a22eed988)

## Pitfalls & gotchas

- **This is N-waypoint, NOT two-click endpoints.** The most common wrong mental model
  is "click start, click end". Here *every* left-click commits one more waypoint
  (`TryAddHoveredWaypoint` -> `TryAddWaypoint`,
  `RoadRouteToolSystem.cs#L828`, `RouteSelectionController.cs#L167`), and the segment
  list is recomputed by pathing each adjacent pair. If you copy a two-click tool you
  will lose the intermediate steering the whole technique exists to provide.

- **No click "finalizes" the route.** Neither left- nor right-click ends the selection.
  Left-click adds, right-click undoes the last waypoint
  (`RouteSelectionController.cs#L202-L214`); consuming the route is a separate `Apply()`
  call wired to the tool UI, not an in-world click. Do not wait for a "done" click that
  does not exist - drive completion from your own UI button.

- **A leg with no connected path rejects the whole add.** `TryAddWaypoint` calls
  `_pathing.FindPath(prev, next, AutoPathMaxDepth)` for the new leg and, if it returns
  zero segments, discards the candidate and sets a warning
  (`RouteSelectionController.cs#L460-L465`). Placing a waypoint on a disconnected road
  (or beyond `AutoPathMaxDepth` hops) silently fails to extend the route - surface the
  `Warning` to the user or the click looks broken.

- **Duplicate-end clicks are ignored.** Clicking the same segment near the current end
  (< 1f from the last waypoint position) is rejected with "already the current route
  end" (`RouteSelectionController.cs#L187-L191`), so rapid double-clicks do not
  double-add.

- **Node hits resolve to an edge, not the node.** A raycast can hit a `Node`; the tool
  converts it to the nearest connected valid road segment before making a waypoint
  (`RoadRouteToolSystem.cs#L890`, `TryCreateWaypointFromNode`). If you skip that branch
  your tool will feel dead whenever the cursor snaps to an intersection.

- **DRIFT TRAP - read at the pin, not the working tree.** This mod's working HEAD is
  ahead of the pinned commit. All line numbers above are resolved at
  `559e72cdb3180e3e71869643367e094a22eed988`; read via
  `git show 559e72c:Systems/RoadRouteToolSystem.cs`, never the checked-out file.

- **Runtime pathing quality is `Needs Verification (in-game)`.** Whether
  `_pathing.FindPath`'s chosen route matches player intent across complex junctions,
  and the exact feel of the per-click accumulation, are behavioural and not provable
  from source alone.

## Variations

- **Preview the pending leg before committing.** The controller maintains a separate
  `_previewSegments` / `_previewWaypoints` set that paths from the last committed
  waypoint to the *current hover* every frame (`RouteSelectionController.cs#L352-L403`,
  `RebuildPreviewState`). Render that to show a live "where the next click will route"
  overlay without mutating the committed route.

- **Insert / move existing waypoints, not just append.** Left-pressing on an existing
  waypoint or on the route line enters an edit mode (`TryBeginEditFromHover`,
  `RoadRouteToolSystem.cs#L821`) that inserts or drag-moves a mid-route waypoint and
  re-paths only the affected legs - useful once routes get long.

- **Degenerate to two-click A-to-B.** If you only ever want endpoints, cap the list at
  two waypoints and treat the second `TryAddWaypoint` as the terminal - you keep the
  same raycast + `FindPath` plumbing with a fixed count.

## See also
- Related recipes: [tool drag-select](tool-drag-select.md) (family V - the momentary
  hover-set selection this technique extends into an ordered history).
- Reference: [raycast filters](../tooling/raycast-filters.md) (the `typeMask` /
  `netLayerMask` / `RaycastFlags` used in step 2);
  [technique index](../../technique-index.md).
- Case study demonstrating it:
  [advanced-road-naming](../../case-studies/advanced-road-naming.md).

## Sources
- Canonical mods (dossier + repo):
  - `advanced-road-naming` @559e72cdb3180e3e71869643367e094a22eed988 -
    `repo/Systems/RoadRouteToolSystem.cs`, `repo/Services/RouteSelectionController.cs`,
    `repo/Domain/RoadRouteWaypoint.cs`
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding
