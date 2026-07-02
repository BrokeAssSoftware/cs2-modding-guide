# Sub-Element Removal Workflows

Reference mod: `BetterBulldozer/Tools/SubElementBulldozerTool.cs`, `BetterBulldozer/Components/PermanentlyRemovedSubElementPrefab.cs`.

Use this pattern when you need to delete props, trees, decals, sub-buildings, or net segments that belong to a parent asset without destroying the entire structure.

## 1. Mirror The Bulldozer Prefab

- Keep `BulldozeToolSystem.prefab` as the canonical selection.
- Override `GetPrefab()` and `TrySetPrefab()` in your tool to delegate to the vanilla bulldozer so the UI stays synchronised even while your custom logic runs.

## 2. Collect Sub-Objects

- Grab the parent entity's `DynamicBuffer<Game.Objects.SubObject>` to enumerate child entities.
- Build cached `EntityQuery` instances for particular prefab families (trees, street lights, branding props, net lanes, etc.). This makes it cheap to find matching prefabs during bulk operations.

## 3. Highlight Targets

- Create a `NativeList<Entity>` to track the entities you plan to modify.
- In an `EntityCommandBuffer`, add `Highlighted` and `BatchesUpdated` components to each target so the player sees what will be removed.
- Optionally store the prefab entity in a second list to support later resets or reporting.

## 4. Schedule Safe Deletions

- Never remove components directly during the raycast pass. Instead, enqueue the work on `ToolOutputBarrier`.
- A simple approach is to add a lightweight component (for example `DeleteInXFrames`) and let a follow-up system perform the actual `EntityManager.DestroyEntity()` call once the tool pipeline finishes the frame.

## 5. Track Persistent Changes

- Record stripped prefabs in a component such as `PermanentlyRemovedSubElementPrefab` so you know which pieces were removed.
- Provide a reset action that iterates the records and re-spawns the original prefabs to restore factory state (useful for "undo all" workflows).

## 6. Expose Safety Toggles

- Not every sub-element is safe to delete (e.g., service upgrades or network connectors). Add settings and UI toggles that block risky categories.
- Validate the toggle state before scheduling the command and show warnings in the tool UI when dangerous categories are disabled.



