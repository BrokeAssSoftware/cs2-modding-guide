# Transform Gizmos And Snap Controls

Reference mod: ExtraDetailingTools.

Use these techniques to let players fine-tune placement transforms and snap behaviour for props, decals, and custom assets.

## 1. Listen For Selection Changes

- Hook into the tool UI to detect when an object becomes selected.
- When a supported asset is active, show a transform panel that exposes position, rotation, and scale fields.
- Synchronise fields with ValueBinding<float> so the Gameface UI reflects live changes.

## 2. Apply Precise Adjustments

- Write changes back through the ToolOutputBarrier to avoid race conditions. Update either the Transform component (for static objects) or the tool command struct (for in-flight placements).
- Allow configurable increments; store the step size in settings and apply it when reacting to slider or button events.
- Provide a dedicated delete button so the player can remove the selected asset without switching tools.

## 3. Snap To Surfaces

- Expose a toggle that maps to ObjectToolSystem.snapToSurface (or the equivalent boolean on the active tool).
- Persist the toggle in settings so the user's preference survives reloads.
- When snapping decals to walls, alter the raycast flags used to position the item so the result hugs the target normal rather than the ground plane.

## 4. Publish Curated Asset Menus

- Register additional tabs with ToolRegistry.AppendTab (for example "Surfaces", "Decals", "Net Lanes").
- Populate each tab with prefab references; when the player picks one, forward the prefab to the appropriate tool via TrySetPrefab().
- Combine with bundled asset packs so creators can rapidly place high-quality detailing content.
