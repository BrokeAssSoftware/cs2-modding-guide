---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: MainThreadDispatcher queue + async re-marshalling"
recipe: mainthread-dispatcher-async
technique_family: "AA - MainThreadDispatcher queue + async re-marshalling"
diataxis: how-to
source_version: "~1.5.10f1 (extra-assets-importer@ed23afc; date-pinned, static source only)"
last_reverified: "2026-07-04"
status: source-verified
canonical_mods:
  - extra-assets-importer@ed23afc5c76442ccc4d1a2cc0817c9bc6f42d256
  - extra-lib@4879487b7df62e83676030a87f92d4b102955eee
technique_applicability: [platform]
Created: 2026-07-04
Updated: 2026-07-04
Owners:
  - codex
---

# MainThreadDispatcher queue + async re-marshalling

> Do slow work (file I/O, parsing, image decode) on a background `Task` so the
> simulation never stalls, then hop the results back onto the main thread through
> `Colossal.Core.MainThreadDispatcher` before you touch any engine/ECS API.

## Problem
You have work that is genuinely slow - importing hundreds of custom asset files,
decoding textures, walking directories, deserialising JSON - and running it inline on
the main thread would freeze the game for seconds. But the moment you need to talk to
the engine (register a prefab, add an ECS component, mutate `PrefabSystem`) you are no
longer allowed to be on an arbitrary thread pool thread: those APIs are main-thread /
`World`-affine. So you need two things at once: get off the main thread to compute, and
get *back* onto it to commit. Doing engine calls from the background `Task` directly
will crash or corrupt state.

## Solution
Split the work in half. Push the CPU/I-O-bound part onto the thread pool with
`Task.Run(...)`, and when a piece of that work needs the engine, marshal that specific
call back onto the main thread with `MainThreadDispatcher.RunOnMainThread(...)`. The
dispatcher is a **shared Colossal SDK type** - `Colossal.Core.MainThreadDispatcher` -
that both mods reach through a `using`; **neither mod ships it**. Chain a
`Task.ContinueWith` on the caller side so a background fault is logged instead of
silently swallowed, and use the dispatcher's frame-wait helper when the engine needs a
frame or two to settle before you continue.

## Steps & Code

### 1. Alias the SDK dispatcher with a `using` (you do not ship it)

`MainThreadDispatcher` is not a type in your mod - it lives in `Colossal.Core`. Extra
Assets Importer pulls it in with an aliasing `using` so the call sites read cleanly:

```csharp
using System.Threading.Tasks;
using UnityEngine;
using MainThreadDispatcher = Colossal.Core.MainThreadDispatcher;
```
Source: `../../../vice-and-order-research/mods/dossiers/extra-assets-importer/repo/MOD/AssetImporter/AssetsImporterManager.cs#L22-L24` (@ed23afc5c76442ccc4d1a2cc0817c9bc6f42d256)

Extra Lib reaches the same type through the plain namespace `using Colossal.Core;`
instead of an alias - either style works, both resolve to the SDK type:

```csharp
using Colossal.Core;
```
Source: `../../../vice-and-order-research/mods/dossiers/extra-lib/repo/MOD/Systems/MainSystem.cs#L10` (@4879487b7df62e83676030a87f92d4b102955eee)

### 2. Push the slow work onto the thread pool with `Task.Run`

The entry point returns the `Task` (it does not `await` it) so the caller stays
responsive. Everything inside `LoadCustomAssetsAsync_Impl` runs off the main thread:

```csharp
public static Task LoadCustomAssetsAsync(ImporterSettings importerSettings)
{
    return Task.Run(() => LoadCustomAssetsAsync_Impl(importerSettings));
}
```
Source: `../../../vice-and-order-research/mods/dossiers/extra-assets-importer/repo/MOD/AssetImporter/AssetsImporterManager.cs#L299-L302` (@ed23afc5c76442ccc4d1a2cc0817c9bc6f42d256)

### 3. Marshal the engine call back onto the main thread

Inside the background work, when a step must touch the engine - here registering a
prefab via `PrefabSystem.AddPrefab` - it is wrapped in `RunOnMainThread(...)` so the
actual engine call executes on the main thread, not the `Task.Run` worker:

```csharp
UIObject assetPackUI = assetPackPrefab.AddComponent<UIObject>();
assetPackUI.m_Icon = Icons.GetIcon(assetPackPrefab);

MainThreadDispatcher.RunOnMainThread(() => EL.m_PrefabSystem.AddPrefab(assetPackPrefab));
MainThreadDispatcher.WaitXFrames(2).Wait();
```
Source: `../../../vice-and-order-research/mods/dossiers/extra-assets-importer/repo/MOD/AssetImporter/AssetsImporterManager.cs#L398-L402` (@ed23afc5c76442ccc4d1a2cc0817c9bc6f42d256)

`WaitXFrames(2).Wait()` blocks the background thread for two engine frames so the just-
registered prefab is fully live before the code continues. Because this runs on the
background thread (not the main thread), the blocking `.Wait()` does not freeze the
game. The exact scheduling semantics of `RunOnMainThread` / `WaitXFrames` (which frame
the callback fires on, ordering guarantees) are SDK-internal and not visible in either
mod's source - treat those as `Needs Verification (in-game)`.

### 4. Rejoin the background `Task` on the caller side and handle faults

A background `Task` that throws will otherwise fail silently. The caller chains
`ContinueWith`, inspects `IsFaulted`, logs the exception, and only then kicks off the
follow-up work:

```csharp
Task task = AssetsImporterManager.LoadCustomAssetsAsync(importerSettings);

task.ContinueWith(t =>
{
    if (t.IsFaulted)
    {
        EAI.Logger.Error($"Error while loading custom assets with new importers: {t.Exception}");
    }

    AssetsImporterManager.BuildAllAssetPacks();
});
```
Source: `../../../vice-and-order-research/mods/dossiers/extra-assets-importer/repo/MOD/AssetImporter/AssetsImporterManager.cs#L169-L179` (@ed23afc5c76442ccc4d1a2cc0817c9bc6f42d256)

## Pitfalls & gotchas

- **The dispatcher is not yours - do not try to ship or reimplement it.**
  `MainThreadDispatcher` is `Colossal.Core.MainThreadDispatcher`, a shared SDK type
  reached via a `using` (aliased in extra-assets-importer L24, plain namespace in
  extra-lib L10). It is not defined anywhere in either mod. Reference it; do not
  reinvent a hand-rolled main-thread queue.

- **Engine/ECS calls from a `Task.Run` worker are illegal.** The whole reason this
  recipe exists: `PrefabSystem.AddPrefab`, `EntityManager` mutations, and other
  `World`-affine APIs must run on the main thread. Everything the background `Task`
  touches on the engine must go through `RunOnMainThread` (see step 3). Calling them
  directly from the worker is the bug this pattern prevents.

- **A background `Task` swallows exceptions unless you observe it.** Without the
  `ContinueWith` / `IsFaulted` check in step 4, an import failure disappears with no
  log. Always rejoin the task and inspect it.

- **`.Wait()` is only safe because it is on the background thread.** In step 3
  `WaitXFrames(2).Wait()` blocks the caller's thread - which is the `Task.Run` worker,
  not the main thread - so it does not freeze the simulation. Blocking with `.Wait()`
  *on* the main thread would deadlock against the very frames it is waiting for. Keep
  the blocking waits inside the background half.

- **Guard before you dispatch.** Extra Assets Importer checks `TryGetPrefab` and
  returns early if the prefab already exists before doing the create-and-dispatch
  (`AssetsImporterManager.cs#L392-L393`), so a re-run does not double-register.

## Variations

- **Register a per-frame updater instead of a one-shot dispatch.** When you do not have
  a discrete result to hand back but instead need to poll until the engine reaches a
  ready state, Extra Lib registers a predicate with the dispatcher from `OnCreate` and
  lets the SDK call it each frame until it returns `true`:

  ```csharp
  protected override void OnCreate()
  {
      base.OnCreate();
      Enabled = false;
      EL.m_PrefabSystem = base.World.GetOrCreateSystemManaged<PrefabSystem>();
      EL.m_ToolbarUISystem = base.World.GetOrCreateSystemManaged<ToolbarUISystem>();
      EL.m_NotificationUISystem = base.World.GetOrCreateSystemManaged<NotificationUISystem>();
      EL.m_EntityManager = EntityManager;

      MainThreadDispatcher.RegisterUpdater(Initialize);
  }
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/extra-lib/repo/MOD/Systems/MainSystem.cs#L46-L59` (@4879487b7df62e83676030a87f92d4b102955eee)

  `Initialize` returns `false` while the game is still loading/booting and only returns
  `true` once `modManager.isInitialized` and the game state are ready
  (`MainSystem.cs#L89-L105`); whether returning `true` retires the updater is
  SDK-internal and `Needs Verification (in-game)`. This is the "wait on the main thread for a
  condition" counterpart to the "compute off-thread, commit on-thread" flow above.

- **Fan out multiple background tasks, then join.** Inside the background impl, Extra
  Assets Importer itself spawns a `Task.Run` per importer and waits on all of them
  before continuing (`AssetsImporterManager.cs#L315-L332`), i.e. nested `Task.Run` for
  parallel I/O under the single outer async entry point.

## See also
- Explanation: [mod lifecycle](../../explanation/mod-lifecycle.md) (where `OnCreate` /
  loading-complete phases sit, and why main-thread affinity matters).
- Reference: [shared libraries](../../reference/shared-libraries/README.md) and
  [ExtraLib](../../reference/shared-libraries/extralib.md) (the `Colossal.Core` /
  ExtraLib surface these mods build on).
- Related recipe: [unofficial asset import](../content/unofficial-asset-import.md)
  (the import workflow this dispatcher pattern powers).

## Sources
- Canonical mods (dossier + repo):
  - `extra-assets-importer` @ed23afc5c76442ccc4d1a2cc0817c9bc6f42d256 - `repo/MOD/AssetImporter/AssetsImporterManager.cs`
  - `extra-lib` @4879487b7df62e83676030a87f92d4b102955eee - `repo/MOD/Systems/MainSystem.cs`
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding
