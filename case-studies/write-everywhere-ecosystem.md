---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Case study: Write Everywhere (ecosystem + reverse-patch bridges)"
case_study: write-everywhere-ecosystem
mod: "Write Everywhere (92908)"
dossier: ../../vice-and-order-research/mods/dossiers/write-everywhere/
repo_commit: 13c70eb04e6bed152257c516a982455148f591a5
source_version: "n/a - no pinned source"
last_reverified: "2026-07-04"
diataxis: explanation
techniques: [D, H, P]
technique_applicability: [ui, content, tooling]
status: source-verified
Created: 2026-07-03
Updated: 2026-07-03
Owners:
  - codex
---

# Write Everywhere - case study

> Write Everywhere is a *platform*, not just a mod: it ships a core signage engine and
> lets other mods add fonts, image atlases, meshes, and layouts through seven static
> `[Obsolete(..., true)]` "bridge" classes that consumers reach only by Harmony
> reverse-patch - a versioned API with zero compile-time coupling to the WE assembly.

## What it does / why it's instructive

Write Everywhere (klyte45, Paradox Mods id `92908`) lets players stamp custom text,
images/decals, and - since the 2.0 rewrite - real spawned props onto virtually any
pickable entity in the city, driven by a template/layout system with a formula language.
Its more interesting audience is *other modders*: WE is designed so that content packs
(a font pack, a sign-atlas pack, a mesh pack) ship as separate mods and register their
assets into WE at runtime without ever referencing `BelzontWE.dll`.

It is instructive for three reasons that generalize well beyond signage:

1. **A host mod with a deliberately un-referenceable API.** WE's whole extension surface
   is seven static bridge classes, each annotated `[Obsolete(..., error: true)]` so a
   direct call *fails to compile*. Consumers must reverse-patch. This is the cleanest
   worked example of "own a versioned static API and force loose coupling" (technique
   family **D**).
2. **Two ways to feed the Gameface UI runtime.** WE both *consumes* a shared COUI host
   (`coui://uil/...` from Unified Icon Library, its one hard dependency) and *serves its
   own* dynamic host (`coui://we.k45/...`) by Harmony-patching the game's UI resource
   handler rather than calling the static `AddHostLocation`. That contrast is a complete
   tour of technique family **H**.
3. **A full C#/React binding stack.** The C# systems expose RPC/value bindings that a
   TypeScript `moduleRegistry` frontend hooks into vanilla UI modules - technique family
   **P** at production scale.

## Architecture at a glance

The entry point `WriteEverywhereCS2Mod` inherits Klyte's `BasicIMod` (itself built on
`Game.Modding.IMod`) rather than deriving from `IMod` directly
(`repo/BelzontWE/WriteEverywhereCS2Mod.cs#L18`). `DoOnCreateWorld` schedules every
subsystem into an explicit `SystemUpdatePhase`, and the file embeds an ASCII dependency
graph documenting the order (`repo/BelzontWE/WriteEverywhereCS2Mod.cs#L22-L83`):

```
updateSystem.UpdateAt<WEWorldPickerController>(SystemUpdatePhase.ModificationEnd);
updateSystem.UpdateAt<WEUISystem>(SystemUpdatePhase.UIUpdate);
updateSystem.UpdateAt<FontServer>(SystemUpdatePhase.Rendering);
updateSystem.UpdateAt<WEAtlasesLibrary>(SystemUpdatePhase.Rendering);
updateSystem.UpdateAt<WECustomMeshLibrary>(SystemUpdatePhase.Rendering);
updateSystem.UpdateAt<WEPreCullingSystem>(SystemUpdatePhase.PreCulling);
```
(`repo/BelzontWE/WriteEverywhereCS2Mod.cs#L69-L83`)

The asset libraries (`FontServer`, `WEAtlasesLibrary`, `WECustomMeshLibrary`) run in the
`Rendering` phase and back the whole content pipeline: each is a file-system-rooted
singleton reading from a fixed subfolder under `BasicIMod.ModSettingsRootFolder`
(`[FileLocation("K45_WE_settings")]`, `repo/BelzontWE/WEModData.cs#L17`) -
`imageAtlases/` (`repo/BelzontWE/Font/Sprites/WEAtlasesLibrary.cs#L35`), `fonts/`
(`repo/BelzontWE/Font/FontServer.cs#L30`), and `objMeshes/`
(`repo/BelzontWE/Mesh/WECustomMeshLibrary.cs#L25`). WE 2.0 also caches BC7 + virtual-
texture atlas data under `.cache/vt*` (`repo/BelzontWE/Font/Sprites/WEAtlasesLibrary.cs#L37-L39`).

`DoOnLoad` registers the main panel with the vanilla `GamePanelUISystem` and force-loads
the `k45-we-vuio` UI module asset (`repo/BelzontWE/WriteEverywhereCS2Mod.cs#L98-L106`).
Load completion is *not* a fixed frame count - `IsInitializationComplete` gates on the
vanilla `TextureStreamingSystem`'s virtual-texture material counts, reflecting the 2.0
dependence on the game VT pipeline (`repo/BelzontWE/WriteEverywhereCS2Mod.cs#L112-L116`).

### The reverse-patch bridge API (7 bridges)

Everything a content pack can register lives in `repo/BelzontWE/Bridge/` as exactly seven
static classes: `FontManagementBridge`, `ImageManagementBridge`, `MeshManagementBridge`,
`TemplatesManagementBridge`, `ModuleOptionsBridge`, `LocalizationBridge`, and
`RoadFnBridge`. Every one carries a compile-time block:

```csharp
[Obsolete("Don't reference methods on this class directly. Always use reverse patch to " +
          "access them, and don't use this mod DLL as hard dependency of your own mod.", true)]
public static class ImageManagementBridge
{
    public static void RegisterImageAtlas(Assembly mainAssembly, string atlasName,
        string[] imagePaths, Action<string> onCompleteLoading = null)
    { /* -> CoroutineWithData -> WEAtlasesLibrary.LoadImagesToAtlas(...) */ }
}
```
(`repo/BelzontWE/Bridge/ImageManagementBridge.cs#L18-L28`, target at
`repo/BelzontWE/Font/Sprites/WEAtlasesLibrary.cs#L318`)

Because the `error: true` flag makes any direct reference a build error, consumers cannot
hard-link the WE DLL. Instead they declare a stub with the same signature and bind WE's
real method onto it with `Harmony.ReversePatch`. That machinery is not in the WE assembly
either - it lives in the **`CS2-BelzontCommons` submodule**, vendored at
`BelzontWE/Commons` (gitlink `3698b64795eec8fd7effe75cf08cb4d1cbe3c674` recorded at this
WE commit):

```csharp
Harmony.ReversePatch(srcMethod, new HarmonyMethod(method));
```
(`repo/BelzontWE/Commons/Utils/BridgeUtils.cs#L166`, submodule `CS2-BelzontCommons` @3698b64)

The same Commons `Redirector` base owns WE's Harmony instance and exposes both forward
patches and reverse patches (`repo/BelzontWE/Commons/Utils/Redirector.cs#L48`,
`#L83-L86`). Registration attributes content per-mod by taking the caller's `Assembly`,
and atlas loads are asynchronous coroutines reporting through a localized notification
helper (`repo/BelzontWE/Bridge/ImageManagementBridge.cs#L26-L28`).

### Feeding the Gameface UI runtime two ways (family H)

WE **consumes** the shared COUI host from Unified Icon Library - `coui://uil/...` - which
is its sole hard dependency, declared numerically in the csproj
(`repo/BelzontWE/BelzontWE.csproj#L36-L39`); its React frontend references those URIs
directly (for example `coui://uil/Standard/Plus.svg`). See
[Unified Icon Library reference](../reference/shared-libraries/unified-icon-library.md)
for the static `AddHostLocation` mechanism UIL uses.

WE **serves** its own host differently. Rather than a static `AddHostLocation`, it
Harmony-patches the game's resource handlers so `coui://we.k45/_fonts/...`,
`coui://we.k45/_css/...`, and `coui://we.k45/_textureAtlas/...` are generated on demand
from the runtime font/atlas data (`repo/BelzontWE/Overrides/GameUIResourceHandlerOverrides.cs#L16-L23`,
serving branches at `#L30-L104`):

```csharp
AddRedirect(typeof(GameUIResourceHandler).GetMethod("OnResourceRequest", ...),
    GetType().GetMethod(nameof(BeforeOnResourceRequest), ...));
```
(`repo/BelzontWE/Overrides/GameUIResourceHandlerOverrides.cs#L20`)

This is the *dynamic/virtual-host* variation of family H: a host whose contents change at
runtime (every user-added font/atlas) cannot be a static folder mount, so WE intercepts
the resolve call instead. WE also uses a second Commons `Redirector`,
`PrefabSystemOverrides`, to patch `PrefabSystem.UpdatePrefabs` and keep WE's default
layouts in sync when the game reloads prefabs
(`repo/BelzontWE/Overrides/PrefabSystemOverrides.cs#L12-L21`).

### The TypeScript moduleRegistry frontend (family P)

The `k45-we-vuio` frontend is a React/VuIO module. Its entry point receives the CS2
`moduleRegistry` and splices WE components into vanilla UI modules
(`repo/_Frontends/UI/k45-we-vuio/src/index.tsx#L9-L15`):

```tsx
const register: ModRegistrar = (moduleRegistry) => {
    moduleRegistry.extend("game-ui/game/components/tool-options/mouse-tool-options/mouse-tool-options.tsx",
        'MouseToolOptions', WriteEverywhereToolOptions);
    moduleRegistry.extend("game-ui/game/components/game-panel-renderer.tsx",
        'gamePanelComponents', RegisterWePanel);
    moduleRegistry.append('GameTopLeft', WEButton);
};
```

The `ModuleRegistry` type (`extend` / `append`) is declared in the module's own
`cs2/modding` typings (`repo/_Frontends/UI/k45-we-vuio/types/modding.d.ts#L7-L21`), and
each spliced component is a `ModuleRegistryExtend`
(`repo/_Frontends/UI/k45-we-vuio/src/toolOptions/WriteEverywhereToolOptions.tsx#L87-L91`).
On the C# side, `WEModulesSystem` binds RPCs under the `modules.` prefix and fires a
`modules.reloadOptions!` event back to the UI when a module registers its options pane
(`repo/BelzontWE/Controllers/WEModulesSystem.cs#L17`, `#L20`, `#L345`). Rendering itself
hooks the HDRP render loop directly: `WERendererSystem` subscribes to
`RenderPipelineManager.beginContextRendering` and unhooks on disable
(`repo/BelzontWE/Systems/WERendererSystem.cs#L43`, `#L51`).

## Techniques demonstrated

- **Harmony redirector / reverse-patch bridge (family D)** - the seven `Bridge/*` classes
  are the canonical example: a static API the host owns, blocked from direct reference,
  consumed via `Harmony.ReversePatch` from the Commons submodule
  (`repo/BelzontWE/Bridge/ImageManagementBridge.cs#L18-L28`,
  `repo/BelzontWE/Commons/Utils/BridgeUtils.cs#L166`). Worked build-your-own-module steps
  live in [how-to: Build a Write Everywhere module](../how-to/content/write-everywhere-modules.md);
  the family is catalogued in the [Technique Index](../technique-index.md) (D).
- **COUI icon/host registration (family H)** - consuming `coui://uil` (UIL hard
  dependency, `repo/BelzontWE/BelzontWE.csproj#L36-L39`) and serving a dynamic
  `coui://we.k45` host by patching `GameUIResourceHandler.OnResourceRequest`
  (`repo/BelzontWE/Overrides/GameUIResourceHandlerOverrides.cs#L20-L23`). Compare the
  static-mount path in the [Unified Icon Library reference](../reference/shared-libraries/unified-icon-library.md)
  and the runtime background in [The Gameface UI Runtime](../explanation/gameface-runtime.md).
- **UISystemBase / React binding (family P)** - a TypeScript `moduleRegistry` frontend
  (`repo/_Frontends/UI/k45-we-vuio/src/index.tsx#L9-L15`) over C#-side RPC/value bindings
  (`repo/BelzontWE/Controllers/WEModulesSystem.cs#L17-L20`). See
  [C#/React Communication](../explanation/ui-cs-communication.md) for the binding model
  and the [Technique Index](../technique-index.md) (P).

Supporting (not the primary lesson): multi-phase ECS scheduling
(`repo/BelzontWE/WriteEverywhereCS2Mod.cs#L69-L83`; see
[Multi-Phase Scheduling](../explanation/multi-phase-scheduling.md)) and file-system-backed
asset libraries with BC7/VT caching.

## Key decisions & tradeoffs

- **Un-referenceable API vs. a normal public DLL.** Marking every bridge
  `[Obsolete(..., true)]` forces consumers into reverse-patching, which guarantees a
  module keeps loading (and simply no-ops) when WE is absent - no hard dependency, no
  load-order crash. The cost is real friction: authors must copy signatures exactly and
  cannot get compile-time checking against them, so a signature change surfaces only at
  runtime (`repo/BelzontWE/Bridge/ImageManagementBridge.cs#L18`).
- **Vendoring the reverse-patch machinery in a submodule.** The Commons `Redirector` /
  `BridgeUtils` live in `CS2-BelzontCommons`, shared across klyte45's mods, so the
  patch/reverse-patch plumbing is written once (`repo/BelzontWE/Commons/Utils/Redirector.cs#L48-L86`).
  The tradeoff is a build that depends on submodule state pinned per commit
  (`3698b64` here).
- **Dynamic COUI host vs. static mount.** Because users add fonts/atlases at runtime, WE
  cannot pre-register a fixed folder; patching the resource handler lets it answer any
  `coui://we.k45/...` URL from live data, at the price of owning a Harmony patch on a
  closed-source game class (`repo/BelzontWE/Overrides/GameUIResourceHandlerOverrides.cs#L20-L23`).
- **One hard dependency, many optional modules.** WE depends only on Unified Icon Library
  (`74417`); fonts/atlases/meshes/layouts are all optional add-on mods
  (`repo/BelzontWE/BelzontWE.csproj#L36-L39`). This keeps the base install lean and pushes
  content growth to the community.
- **ECS-scheduled *and* Harmony-based.** WE is not "Harmony-free": it declares
  `Lib.Harmony 2.2.2` (`repo/BelzontWE/BelzontWE.csproj#L191`) and uses it for both the
  reverse-patch bridges and the two resource/prefab redirects, on top of explicit
  `SystemUpdatePhase` scheduling.

## Pitfalls / upstream-watch

- **Bridge signatures are the compatibility contract.** They are copied by hand into every
  consumer and bound at runtime, so a signature change in a WE major version silently
  breaks modules until they re-copy it. Re-verify the seven bridges on each WE major bump
  (checked here at `v2.0.0r12`) (`repo/BelzontWE/Bridge/`).
- **Patches target closed-source game internals.** `GameUIResourceHandler.OnResourceRequest`
  and `PrefabSystem.UpdatePrefabs` are Colossal methods; a rename or signature change on a
  game patch breaks WE's UI hosting or prefab-sync
  (`repo/BelzontWE/Overrides/GameUIResourceHandlerOverrides.cs#L20-L23`,
  `repo/BelzontWE/Overrides/PrefabSystemOverrides.cs#L20`). These are patch-sensitive.
- **The frontend build is not part of this snapshot's verified surface.** The
  `k45-we-vuio` npm/webpack scripts and the `CS2-WEModuleTemplate` folder layout were not
  re-derived at this commit and are `Needs Verification`; the C#-side binding names and the
  `moduleRegistry` calls above *are* source-confirmed.
- **Atlas loads are asynchronous.** `RegisterImageAtlas` returns before the atlas exists
  and completes via callback, so a module cannot assume its icons are addressable
  synchronously after the call (`repo/BelzontWE/Bridge/ImageManagementBridge.cs#L26-L28`).
- **GitHub issue #1 (pre-2.0) reported a module crash via a deprecated
  `UISystem.AddHostLocation` signature.** Whether it still applies after the 2.0 UI
  rewrite is `Needs Verification` (the API surface changed), per the dossier.

## Source pointers

- Dossier: `../../vice-and-order-research/mods/dossiers/write-everywhere/`
  (index / source / modding + notes)
- Repo @ `13c70eb04e6bed152257c516a982455148f591a5` (branch master, tag `v2.0.0r12`);
  Commons submodule `CS2-BelzontCommons` @`3698b64795eec8fd7effe75cf08cb4d1cbe3c674`.
- Storefront: Paradox Mods id `92908`, userModVersion `2.0.0.12` (modVersion 68),
  requiredVersion `1.6*`; sole hard dependency Unified Icon Library (`74417`).
- Related handbook pages: [how-to: Write Everywhere modules](../how-to/content/write-everywhere-modules.md),
  [Unified Icon Library reference](../reference/shared-libraries/unified-icon-library.md),
  [C#/React Communication](../explanation/ui-cs-communication.md),
  [The Gameface UI Runtime](../explanation/gameface-runtime.md),
  [Technique Index](../technique-index.md) (families D, H, P).
