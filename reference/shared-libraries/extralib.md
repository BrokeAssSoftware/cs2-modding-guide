---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Reference: ExtraLib (shared library)"
Summary: Lookup reference for depending on ExtraLib (Paradox Mods id 75724) - the shared library the Extra suite builds on. Covers how to declare and validate the dependency and the public helpers it exposes (coui://extralib icon host, notification UI, localization loader, entity-edit queue, ExtraPanels UI framework) - each helper cross-checked against real mod source at a pinned commit, including a real consumer (ExtraAssetsImporter).
diataxis: reference
source_version: "~1.5.10f1 (extra-lib@4879487; date-pinned, static source only)"
last_reverified: "2026-07-04"
canonical_mods:
  - extra-lib@4879487b7df62e83676030a87f92d4b102955eee
  - extra-assets-importer@ed23afc5c76442ccc4d1a2cc0817c9bc6f42d256
technique_applicability: [ui, content, tooling]
status: source-verified
Created: 2026-07-03
Updated: 2026-07-03
Owners:
  - codex
References:
  - Label: "Reference: Unified Icon Library (icon host peer)"
    Path: ./unified-icon-library.md
  - Label: "Explanation: Dependency Strategy (declare/validate/degrade)"
    Path: ../../explanation/dependency-strategy.md
  - Label: "How-to: I18n Everywhere localization backbone"
    Path: ../../how-to/localization/i18n-integration.md
  - Label: "Recipe: Localization helper (IDictionarySource path)"
    Path: ../../how-to/recipes/localization-helper.md
  - Label: "Technique H - COUI icon/host registration"
    Path: ../../technique-index.md
  - Label: Official CS2 modding wiki (authoritative)
    Path: https://cs2.paradoxwikis.com/Modding
---

# Reference: ExtraLib (shared library)

**ExtraLib** is a standalone dependency mod that bundles the shared services a suite of
sibling mods reuse instead of re-implementing: a COUI icon host, an embedded-localization
loader, a notification-UI helper, a centralized entity-edit queue with progress notifications,
prefab/asset toolbar scaffolding, and a draggable ExtraPanels UI framework. It is the canonical
example of *shipping a shared dependency* - a library mod that consumers declare as a hard
requirement and call into, rather than duplicating its code.

- **Paradox Mods id:** `75724` (`repo/Properties/PublishConfiguration.xml#L3`).
- **Source:** `github.com/AlphaGaming7780/ExtraLib`.
- **Bootstrap:** `EL.OnLoad` resolves the mod's asset path, mounts the `coui://extralib` icon
  host, loads embedded localization, registers ExtraLib's ECS/UI systems, and applies Harmony
  patches, so a consumer can call into shared services without repeating that setup
  (`repo/EL.cs#L45-L82`).

**Scope of verification.** Every helper below was confirmed in real mod source at a pinned
commit (citations inline). ExtraLib is pinned at `extra-lib@4879487` (v1.4.8); a real consumer,
`extra-assets-importer@ed23afc` (v1.7.4), is cited to show the dependency and helper calls from
the outside. Method names, signatures, and defaults were read at those commits. For the
authoritative game-side APIs these helpers wrap (`NotificationUISystem`, `IDictionarySource`,
`PrefabSystem`), link out to the [CS2 modding wiki](https://cs2.paradoxwikis.com/Modding)
rather than duplicating it here.

Corpus and pinned commits used for verification:

- `extra-lib` @`4879487` - `repo/EL.cs`, `repo/MOD/Helpers/Icons.cs`, `repo/MOD/Helpers/ExtraLocalization.cs`, `repo/MOD/ClassExtension/NotificationUI.cs`, `repo/MOD/Systems/UI/ExtraPanels/ExtraPanelsUISystem.cs`, `repo/Properties/PublishConfiguration.xml`
- `extra-assets-importer` @`ed23afc` - `repo/Properties/PublishConfiguration.xml`, `repo/EAI.cs`, `repo/MOD/AssetImporter/ImporterBase.cs` (a real consumer)

## Declaring the dependency

ExtraLib is a **hard dependency**: every mod in the suite requires it to be installed and
loaded, and calls into its static API fail if it is absent. Declare it by numeric mod id so the
in-game publisher offers to install it for players.

| Where | What to write | Verified basis |
| --- | --- | --- |
| `Properties/PublishConfiguration.xml` | A `<Dependency Id="75724" />` entry. | A real consumer declares exactly this: `extra-assets-importer/repo/Properties/PublishConfiguration.xml#L81` -> `<Dependency Id="75724" />`. |
| README / release notes | List ExtraLib as **Required**. | Convention. |

Reference the library by **both name and id** (`ExtraLib`, id `75724`). A UI package's
`mod.json` may also carry a string dependency identifier; the exact token/casing is
**Needs Verification** - take it from the current ExtraLib release. For the general strategy of
referencing a shared assembly for compilation *without bundling* it (`<Private>false</Private>`),
declaring it, validating it, and degrading gracefully, see
[Dependency Strategy](../../explanation/dependency-strategy.md).

### Validating and deferring (the verified consumer pattern)

The library self-guards its own boot: `EL.OnLoad` calls
`GameManager.instance.modManager.TryGetExecutableAsset(this, out var asset)` and logs a fatal
error and returns if it cannot resolve its own asset (`repo/EL.cs#L49-L53`). A **consumer** does
not scan for ExtraLib manually in the corpus; instead it declares the dependency and **defers**
its own setup until ExtraLib's systems have initialized, by registering an init callback:

```csharp
// ExtraAssetsImporter.OnLoad - hand setup to ExtraLib's lifecycle instead of racing it
EL.AddOnInitialize(Initialize);
```

(`extra-assets-importer/repo/EAI.cs#L160`). This is the source-verified way to avoid load-order
races. If you additionally want to detect ExtraLib's *absence* and degrade to text fallbacks,
use the runtime assembly-detection pattern documented in
[Dependency Strategy - validate at runtime](../../explanation/dependency-strategy.md) (that page
carries the cited implementation; this one does not duplicate it).

## Public helpers

The API is exposed as static entry points on `EL` plus a handful of helper classes. Names,
signatures, and defaults below are read at `extra-lib@4879487`.

### Icon host (coui://extralib)

ExtraLib mounts its `Icons` directory as a COUI host so consumers reference shared SVGs by URI,
exactly like [Unified Icon Library](./unified-icon-library.md) but under the `extralib` host.

| Symbol | Value / signature | Purpose | Verified usage |
| --- | --- | --- | --- |
| `Icons.IconsResourceKey` | `"extralib"` | Host name for every COUI location ExtraLib mounts. | `repo/MOD/Helpers/Icons.cs#L11` |
| `Icons.COUIBaseLocation` | `"coui://extralib"` | URI prefix; build icon URIs as `$"{Icons.COUIBaseLocation}/Icons/<Type>/<name>.svg"`. | `repo/MOD/Helpers/Icons.cs#L12` |
| `Icons.LoadIconsFolder(uri, path, shouldWatch = false)` | Registers a host via `UIManager.defaultUISystem.AddHostLocation(uri, path, shouldWatch)`, deduping in a `Dictionary<string,List<string>>`. | Mount a folder of SVGs under a host. | `repo/MOD/Helpers/Icons.cs#L23-L37` |
| `Icons.UnLoadIconsFolder(uri, path = null)` | `RemoveHostLocation(uri)` (all) or `RemoveHostLocation(uri, path)` (one path). | Tear a host down (hot-reload / disable). | `repo/MOD/Helpers/Icons.cs#L39-L57` |

Consumer example - ExtraAssetsImporter builds a notification thumbnail URI off the shared host:

```csharp
thumbnail: $"{Icons.COUIBaseLocation}/Icons/NotificationInfo/{ImporterId}.svg",
```

(`extra-assets-importer/repo/MOD/AssetImporter/ImporterBase.cs#L97`).

Note: `EL.OnLoad` mounts the host with the default `shouldWatch: false`
(`repo/EL.cs#L58`), and `GetIcon` resolves by on-disk file existence, falling back to
`Icons/Misc/placeholder.svg` when a `{Type}/{name}.svg` is missing - a misnamed icon silently
shows the placeholder rather than erroring (`repo/MOD/Helpers/Icons.cs#L66-L88`).

### Notification UI

An extension method wraps the game's `NotificationUISystem` so mods add or update a progress
notification in one call, and update it by re-submitting the same id.

| Symbol | Signature | Verified usage |
| --- | --- | --- |
| `NotificationUISystem.AddOrUpdateNotification` (extension) | `(this NotificationUISystem, ref NotificationInfo)` - forwards to the game's multi-arg overload. | `repo/MOD/ClassExtension/NotificationUI.cs#L8-L18` |
| `EL.m_NotificationUISystem` | Cached `NotificationUISystem` reference for consumers. | `repo/EL.cs#L43` |

Consumer example - ExtraAssetsImporter posts an import-progress notification:
`EL.m_NotificationUISystem.AddOrUpdateNotification(...)`
(`extra-assets-importer/repo/MOD/AssetImporter/ImporterBase.cs#L93`).

### Localization loader

A reusable embedded-JSON localization loader: it reads `embedded/Localization/{locale}.json`
from the calling assembly and registers a `MemorySource` per locale, falling back to `en-US`.

| Symbol | Signature | Verified usage |
| --- | --- | --- |
| `ExtraLocalization.LoadLocalization` | `(ILog log, Assembly assembly, bool singleFile = false, string namespaceName = null, string defaultLocalID = "en-US")` | `repo/MOD/Helpers/ExtraLocalization.cs#L27,L35` |

**Namespace trap:** `namespaceName` defaults to `assembly.GetName().Name` and the loader
resolves resources at `{namespaceName}.embedded.Localization.{locale}.json`
(`repo/MOD/Helpers/ExtraLocalization.cs#L42`). If a consumer's assembly name differs from its
root namespace, every locale fails silently - pass an explicit `namespaceName`. Consumer
example: `ExtraLocalization.LoadLocalization(Logger, Assembly.GetExecutingAssembly(), false)`
(`extra-assets-importer/repo/EAI.cs#L95`; ExtraLib itself does the same at `repo/EL.cs#L61`).

This is ExtraLib's *own* embedded-dictionary path. For localizing through the separate
**I18n Everywhere** dependency (community JSON bundles, runtime key interception), see
[Use I18n Everywhere as a Localization Backbone](../../how-to/localization/i18n-integration.md);
for the vanilla `IDictionarySource` route, see
[Localization helper](../../how-to/recipes/localization-helper.md).

### Lifecycle hooks and entity-edit queue

Static registration points on `EL` let consumers schedule work against ExtraLib's `MainSystem`
lifecycle, and queue prefab/entity mutations that run inside a notification coroutine (so heavy
edits do not freeze the UI on load).

| Symbol | Purpose | Verified usage |
| --- | --- | --- |
| `EL.AddOnInitialize(handler)` | Runs during `MainSystem.Initialize` once the mod manager is up and the game is at the main menu - seed menus / schedule jobs. | `repo/EL.cs#L111-L114` |
| `EL.AddOnMainMenu(handler)` | Runs whenever the game loads the main menu - rebuild toolbar categories / reset global state. | `repo/EL.cs#L106-L109` |
| `EL.AddOnEditEnities(handler, EntityQueryDesc)` / `EL.AddOnEditEnities(EntityRequester)` | Queues a delegate + query that ExtraLib materializes and invokes inside a notification coroutine, one requester per frame, with try/catch logging. | `repo/EL.cs#L96-L104` |

The queue's catch only logs (it does not rethrow), so guard your handler against despawned
entities yourself; build minimal queries to avoid iterating every entity.

### ExtraPanels UI framework

A framework for draggable/resizable/collapsible panels registered through ExtraLib's UI system.

| Symbol | Signature | Verified usage |
| --- | --- | --- |
| `ExtraPanelsUISystem.AddExtraPanel<T>()` | `where T : ExtraPanelBase` - creates the system via `World.GetOrCreateSystemManaged<T>()` and registers it. | `repo/MOD/Systems/UI/ExtraPanels/ExtraPanelsUISystem.cs#L28-L33` |
| `ExtraPanelsUISystem.AddExtraPanel(ExtraPanelBase)` | Registers an already-created panel instance. | `repo/MOD/Systems/UI/ExtraPanels/ExtraPanelsUISystem.cs#L35` |

Panels subclass `ExtraPanelBase`, and their id is `GetType().FullName` - two panels of the same
type cannot coexist, and unknown-id triggers warn-and-return. A collapsed panel skips
`OnProcess`; call `RequestUpdate()` after mutating state or the binding never refreshes
(`repo/MOD/Systems/UI/ExtraPanels/ExtraPanelBase.cs#L10,L41-L50,L84-L87`).

## Needs Verification

| Claim | Why unverified |
| --- | --- |
| The string dependency identifier for a UI package `mod.json` | Not present in the pinned corpus; take name/casing from the current ExtraLib release. |
| SemVer / binary-compatibility guarantees across ExtraLib minor versions | The repo has no `LICENSE` file and the `<Version>` element is intentionally commented out (`repo/ExtraLib.csproj#L7`); pin a specific build and re-run smoke tests when upgrading rather than assuming ABI stability. |
| `ExtraLibMonoScript` availability | `EL.extraLibMonoScript` is no longer instantiated by the library (the field is null unless a consumer creates the MonoBehaviour itself); do not rely on it. |

## See also

- [Unified Icon Library](./unified-icon-library.md) - the peer icon-host library, and the
  two-arg `AddHostLocation` example.
- [Dependency Strategy](../../explanation/dependency-strategy.md) - declare / reference /
  validate / degrade, with the cited runtime assembly-detection guard.
- [Use I18n Everywhere as a Localization Backbone](../../how-to/localization/i18n-integration.md)
  and [Localization helper](../../how-to/recipes/localization-helper.md) - localization routes
  beyond ExtraLib's own embedded loader.
- [Technique Index](../../technique-index.md) - family H (COUI icon/host registration); ExtraLib
  and ExtraAssetsImporter also appear under family AA (MainThreadDispatcher / coroutine
  re-marshalling).
- Official [CS2 modding wiki](https://cs2.paradoxwikis.com/Modding) - authoritative game APIs
  these helpers wrap; link out rather than duplicating.
