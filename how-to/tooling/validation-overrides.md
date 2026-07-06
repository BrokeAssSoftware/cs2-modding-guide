---
FrontmatterVersion: 1
DocumentType: Guide
Title: "How-to: Bypassing placement/tool validation errors"
diataxis: how-to
source_version: "~1.5.9 (anarchy@a6311e8; date-pinned, static source only)"
last_reverified: "2026-07-04"
canonical_mods:
  - anarchy@a6311e898d20a775368668b234aaa32f06e3e1eb
technique_applicability: [tooling]
status: source-verified
Created: 2026-07-03
Updated: 2026-07-03
Owners:
  - codex
---

# Bypassing placement/tool validation errors

> Let players place assets in normally-forbidden situations (overlaps, water, steep
> slopes, "already exists") by flipping the disable-flags on the game's own
> `ToolErrorData` prefab entities while your override is on - and cleanly restoring them
> when it is off.

## Problem
The tool system refuses many placements: overlapping props, building in water, roads
too close, a unique building that already exists. Each refusal is a **tool error**. You
want an "anarchy" mode that suppresses a chosen set of these errors so designers can
place freely - without permanently breaking the safety net for everyone, and without
patching the validation code path itself.

## Solution
CS2 models every possible tool error as a **prefab entity** carrying a `ToolErrorData`
component. That component has a `m_Error` (which error) and a `m_Flags` bitfield with
`ToolErrorFlags.DisableInGame` / `DisableInEditor` bits. If those bits are set, the game
does not raise the error. So the whole technique is: query all `ToolErrorData` prefabs,
and for each error the player opted to suppress, OR in the disable bits via a barrier
command buffer. A companion system clears the bits again (`&= ~`) when the override is
turned off, restoring vanilla behavior. No Harmony patch on the validation methods is
needed - you toggle data the vanilla checks already read.

## Steps & Code

### 1. Query the `ToolErrorData` prefab entities

In `OnCreate`, build a query over the error prefabs and grab a barrier for the phase the
system runs in (`ModificationBarrier5` here):

```csharp
m_Barrier = World.GetOrCreateSystemManaged<ModificationBarrier5>();
m_ToolErrorPrefabQuery = GetEntityQuery(new EntityQueryDesc
{
    All = new ComponentType[]
    {
        ComponentType.ReadOnly<ToolErrorData>(),
        ComponentType.ReadOnly<NotificationIconData>(),
    },
});
RequireForUpdate(m_ToolErrorPrefabQuery);
```
Source: `../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/Systems/ErrorChecks/DisableToolErrorsSystem.cs#L48-L60` (@a6311e898d20a775368668b234aaa32f06e3e1eb)

### 2. For each opted-in error, set the disable flags

Get the player's allow-list, iterate the error prefabs, and for any error on the list
whose disable bits are not already set, OR them in and queue the write on the command
buffer:

```csharp
EntityCommandBuffer buffer = m_Barrier.CreateCommandBuffer();
List<ErrorType> errorTypesToDisable = m_AnarchyUISystem.GetAllowableErrorTypes();

foreach (Entity currentEntity in toolErrorPrefabs)
{
    if (EntityManager.TryGetComponent(currentEntity, out ToolErrorData toolErrorData) &&
        errorTypesToDisable.Contains(toolErrorData.m_Error) &&
        ((toolErrorData.m_Flags & ToolErrorFlags.DisableInEditor) != ToolErrorFlags.DisableInEditor ||
         (toolErrorData.m_Flags & ToolErrorFlags.DisableInGame)   != ToolErrorFlags.DisableInGame))
    {
        toolErrorData.m_Flags |= ToolErrorFlags.DisableInGame;
        toolErrorData.m_Flags |= ToolErrorFlags.DisableInEditor;
        buffer.SetComponent(currentEntity, toolErrorData);
    }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/Systems/ErrorChecks/DisableToolErrorsSystem.cs#L73-L110` (@a6311e898d20a775368668b234aaa32f06e3e1eb)

At the end of the pass, enable the counterpart restore system so the change can be
reversed on the next toggle:

```csharp
m_EnableToolErrorsSystem.Enabled = true;
```
Source: `../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/Systems/ErrorChecks/DisableToolErrorsSystem.cs#L116` (@a6311e898d20a775368668b234aaa32f06e3e1eb)

### 3. Restore vanilla by clearing the flags

The reverse system runs on `ModificationEndBarrier`, clears the disable bits with
`&= ~`, and disables itself after one pass. Note the barrier is chosen to match the
system's update phase - a mismatched barrier throws:

```csharp
EntityCommandBuffer buffer = m_Barrier.CreateCommandBuffer();
// ... for each error prefab whose flag is set and is not on the do-not-re-enable list:
toolErrorData.m_Flags &= ~ToolErrorFlags.DisableInGame;
toolErrorData.m_Flags &= ~ToolErrorFlags.DisableInEditor;
buffer.SetComponent(currentEntity, toolErrorData);
// ...
Enabled = false;
```
Source: `../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/Systems/ErrorChecks/EnableToolErrorsSystem.cs#L76-L109` (@a6311e898d20a775368668b234aaa32f06e3e1eb)

### 4. (Related) widen what a tool can even target

Some "cannot place" situations are really "the tool cannot raycast that layer." Anarchy
also postfixes `NetToolSystem.InitializeRaycast` to add upgradeable layers in replace
mode - see [raycast filters](raycast-filters.md) for that pattern
(`NetToolSystem_InitializeRaycast.cs#L45-L47`).

## Pitfalls & gotchas

- **The type is `ToolErrorData` with `m_Flags` of type `ToolErrorFlags`, and the error
  enum is `ErrorType`** - not `ToolErrorType`/`ToolBase.Error`. Suppression is a bit-flip
  on prefab data, not removal of entries from an error collection. (Older community
  write-ups describe a Harmony-postfix-strips-error-list approach; the source-verified
  Anarchy path is the flag flip.)
- **Every override must be reversible.** Because you are mutating shared prefab data,
  the disable bits persist until something clears them. Always ship the paired
  restore system (`&= ~` the flags) and run it on toggle-off / tool-change / load, or
  the safety net stays disabled indefinitely.
- **Match the barrier to the system phase.** The disable system uses
  `ModificationBarrier5`; the enable system uses `ModificationEndBarrier` and documents
  in-source that "using a barrier in the wrong phase will produce an error." Do not copy
  a barrier type across systems that run in different phases.
- **Guard with `activeTool == null`.** The disable system returns early when
  `m_ToolSystem.activeTool.toolID == null` so it does not thrash flags when no tool is
  active (`DisableToolErrorsSystem.cs#L67-L70`).
- **Opt-in, per-error.** Suppress only errors on an explicit allow-list
  (`GetAllowableErrorTypes()`), never all of them. A blanket "disable all tool errors"
  removes legitimate guards (e.g. building overlap that would corrupt geometry). Keep a
  do-not-re-enable list too, mirroring the enable system's `m_DoNotReEnableForEditor`.
- Whether suppressing a specific error actually yields a stable placement (vs a
  visually-placed but simulation-broken entity) is `Needs Verification (in-game)` per
  error type - disabling the *error* does not guarantee the *placement* is sound.

## Variations

- **Per-tool auto-suspend.** Some workflows (line-tool painting, tree brushing) should
  not run with anarchy on. Detect the active tool via `ToolSystem.activeTool` and skip
  the disable pass (or re-enable errors) while an opt-out tool is selected. `Needs
  Verification (in-game)` for the exact opt-out list, which is UI/settings-driven.
- **Single special-case error.** Anarchy suppresses the "Already Exists" unique-building
  error on its own toggle (`AllowPlacingMultipleUniqueBuildings`) by looking up that one
  `NotificationIconPrefab` by `PrefabID` and setting its flags
  (`DisableToolErrorsSystem.cs#L75-L91`) - a template for suppressing one named error
  rather than a category.
- **Layer widening instead of error suppression.** If the block is "tool won't target
  this," extend the raycast (step 4 / [raycast filters](raycast-filters.md)) rather than
  disabling an error.

## See also
- Sibling tooling how-tos: [raycast filters](raycast-filters.md),
  [transform gizmos](transform-gizmos.md) (Anarchy's `PreventOverride` gives a related
  placement freedom), [sub-element removal](sub-element-removal.md).
- Reference: [technique index](../../technique-index.md) - family Y (event-driven
  ModificationEnd / ToolOutputBarrier) and family X (reflect into private fields via
  Harmony Traverse, used by Anarchy's raycast patch).
- Explanation: [multi-phase scheduling](../../explanation/multi-phase-scheduling.md)
  (why the barrier phase matters),
  [conditional execution](../../explanation/conditional-execution.md) (gating the
  override by active tool / game mode).

## Sources
- Canonical mods (dossier + repo):
  - `anarchy` @a6311e898d20a775368668b234aaa32f06e3e1eb -
    `repo/Anarchy/Systems/ErrorChecks/DisableToolErrorsSystem.cs`,
    `repo/Anarchy/Systems/ErrorChecks/EnableToolErrorsSystem.cs`,
    `repo/Anarchy/Patches/NetToolSystem_InitializeRaycast.cs`
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding
