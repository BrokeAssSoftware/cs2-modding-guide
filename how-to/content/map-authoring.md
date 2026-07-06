---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Author a CS2 Map: Terrain, Resources, Climate, Spawn Setup"
Summary: The map-authoring workflow for CS2 - lock terrain early, sculpt water, paint resources, tune climate, set start tile and connections, and run a pre-export validation checklist.
diataxis: how-to
source_version: "n/a - not source-verified (editor workflow, not run this pass)"
last_reverified: "2026-07-04"
status: needs-verification
Created: 2026-07-03
Updated: 2026-07-03
Owners:
  - codex
References:
  - Label: Editor Integration (asset side of the same editor)
    Path: ./editor-integration.md
  - Label: Support and Publishing (publish the finished map)
    Path: ./support-and-publishing.md
  - Label: Build, Test, and Publish (publish walkthrough)
    Path: ../../tutorials/build-and-publish.md
  - Label: Technique Index
    Path: ../../technique-index.md
---

# Author a CS2 Map: Terrain, Resources, Climate, Spawn Setup

This how-to covers building a playable Cities: Skylines II map: shaping terrain and water,
painting natural resources, tuning climate, and configuring the spawn setup (start tile,
connections, display names) - then a pre-export checklist that catches the failures that
only show up after a map ships.

It is goal-directed and assumes you are already in the map editor. Work the stages in
order; several steps are hard to change once later work depends on them.

> **Version note.** The editor's tool names, brush controls, climate-curve editors, and
> connection/spawn panels are version-specific. Editor-specific labels below are marked
> **Needs Verification** - confirm the exact wording and control locations against the
> current editor and the official modding wiki.

---

## Stage 1: Terrain (do this first)

Terrain decisions constrain everything downstream, so lock them early.

1. **Lock your elevation range early.** Establishing the min/max height (sea level to peak)
   up front matters because changing it later disrupts water tables and any spline meshes
   (roads, rail) already placed against the old heights.
2. **Sculpt river beds and water bodies before you place splines.** Carve the channels and
   shorelines first; retrofitting water under existing road/rail is painful and tends to
   leave floating or submerged segments.
3. **Block out the major landforms** (coastline, valleys, ridgelines) before fine
   detailing, so the playable buildable area is clear.

> **Needs Verification:** the exact elevation-range control, the water/river sculpting
> tools, and whether sea level is a fixed or author-set value in the current editor.

## Stage 2: Resource painting

1. **Paint natural resources** (for the resource types the game supports - for example
   fertile land, ore, oil, and similar) where they make geographic sense.
2. **Use broad, feathered brushes.** Hard-edged resource boundaries create sharp
   simulation transitions that read as artificial and can cause abrupt economic effects at
   the border. Soft, feathered edges blend better.
3. **Balance the map.** Spread resources so no single strategy is the only viable one, and
   leave buildable land that is not sitting on a resource.

> **Needs Verification:** the exact resource types available to paint, the brush/painting
> tool names, and any per-resource density controls in the current editor.

## Stage 3: Climate and weather

1. **Tune the climate curves** - temperature, precipitation, and any special effects
   (for example aurora) - to fit the scenario you are building (arid, temperate, arctic).
2. **Test with accelerated time** so you see a full seasonal cycle quickly and confirm the
   climate behaves as intended year-round rather than only at the season you authored in.

> **Needs Verification:** which climate parameters are curve-editable, the exact editor for
> them, and the list of special weather effects in the current version.

## Stage 4: Spawn setup

Configure how a new city starts on this map.

1. **Set the start tile** - the tile the player begins on - and confirm it has buildable
   land, road access, and enough resource/water variety for an early game.
2. **Place inbound/outbound connections** - road, rail, air, ship, and any external
   pipes/lines the map should offer - at the map edges so the city can trade and grow.
3. **Confirm water availability** (fresh water intake and a viable place for sewage
   outflow) near the start area.
4. **Set localized display names** for the map and any labeled features so they read
   correctly for players in every supported language.

> **Needs Verification:** the exact start-tile and connection-placement tools, the set of
> connection types, and how localized names are entered in the current editor.

---

## Pre-export checklist

Run this before you export/publish the map.

- [ ] Elevation range locked; no later terrain edits pending.
- [ ] Water bodies sculpted; no floating or submerged road/rail segments.
- [ ] Resources painted with feathered edges; map is strategically balanced.
- [ ] Climate curves tuned; tested across a full accelerated-time cycle.
- [ ] Start tile set with buildable land, road access, and water.
- [ ] Inbound/outbound connections placed for the transport modes the map offers.
- [ ] Fresh-water intake and sewage outflow viable near the start.
- [ ] Localized display names set for the map and labeled features.
- [ ] **Ran the simulation preview for several in-game days** (a few days minimum) and
      confirmed core services function, traffic flows, and the economy does not stall.

> **Needs Verification:** the exact simulation-preview control and a recommended preview
> duration. Several in-game days is a sensible minimum; confirm the tooling in the current
> editor.

---

## Where to go next

- **Author a custom asset to place on the map:**
  [PBR Asset Workflow](./pbr-asset-workflow.md) and
  [Editor Integration](./editor-integration.md).
- **Publish the finished map:** [Support and Publishing](./support-and-publishing.md)
  and [Build, Test, and Publish](../../tutorials/build-and-publish.md).
- **Pick a related technique:** [Technique Index](../../technique-index.md).
