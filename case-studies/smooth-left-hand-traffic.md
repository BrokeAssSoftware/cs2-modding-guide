---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Case study: Smooth Left-Hand Traffic"
case_study: smooth-left-hand-traffic
mod: "Smooth Left-Hand Traffic"
dossier: ../../vice-and-order-research/mods/dossiers/smooth-left-hand-traffic/
repo_commit: 37b2850b10ced1cd17457c25778409e57e00a03e
source_version: "~1.6.x (smooth-left-hand-traffic@37b2850; GameVersion 1.6.*/1.6.0f1, date-pinned, static source only)"
last_reverified: "2026-07-05"
diataxis: explanation
techniques: [A, F, AU, BG]
technique_applicability: [infrastructure, core, ui]
status: source-verified
Created: 2026-07-05
Updated: 2026-07-05
Owners:
  - codex
Summary: A small, no-Harmony mod that inverts building-internal road subnets for left-hand-traffic cities - a clean canonical example of prefab-field mutation via UpdatePrefab, subnet-graph traversal with host->upgrade recursion, opt-out-only JSON persistence, and vanilla MouseToolOptions augmentation, all delegating RHT safety to the game's own NetInvertMode semantics.
---

# Smooth Left-Hand Traffic - case study

> A deliberately small mod that does one thing well: it flips the internal
> driveways and lane arrows of service buildings so they read correctly in
> left-hand-traffic cities - and in doing so gives the cleanest canonical
> example of prefab-field mutation, subnet traversal, opt-out JSON persistence,
> and vanilla-UI augmentation combined without a single Harmony patch.

All code claims below are cited to `repo/<path>#Lxx` at pinned commit
`37b2850b10ced1cd17457c25778409e57e00a03e` (tag `v0.5.1`, "Release 0.5.1",
published modVersion 18), surfaced through the dossier at
`../../vice-and-order-research/mods/dossiers/smooth-left-hand-traffic/`. That
dossier corrected an earlier (v0.2.3-era) description - a single
`InvertPrefabLHT` system, a `CityConfigurationSystem.leftHandTraffic`
early-return gate, an `isAllInverted` one-shot guard, a "Developer Mode LHT
toggle" dependency, and no UI/persistence. None of that exists at this commit and
none of it is reintroduced here.

## What it does / why it's instructive

Vanilla CS2 lays out the internal roads of many service buildings (bus stations,
depots, cargo yards, parking structures) with right-hand-traffic geometry baked
into their prefab subnets. In a left-hand-traffic city those internal driveways
and lane arrows face the wrong way. Smooth Left-Hand Traffic (SLHT) walks every
eligible building/extension prefab and sets a single field -
`ObjectSubNets.m_InvertWhen` - to `NetInvertMode.LefthandTraffic`, then re-bakes
the prefab so already-placed instances pick up the corrected internal network.

It is instructive precisely because it is *small and correct*. It shows how much
you can accomplish by mutating **prefab data** rather than per-instance ECS
components or Harmony-patching the game: no simulation systems are replaced, no
save data is written into the ECS world, and RHT-vs-LHT correctness is never
computed by the mod at all - it is delegated to the game's own `NetInvertMode`
semantics (`LefthandTraffic` only actually inverts when the active city is LHT),
so the exact same prefab edit is safe to apply on every save regardless of type.
Around that one-field edit it layers four reusable techniques: a subnet-graph
eligibility scan with host->upgrade recursion, an opt-out-only JSON side-car,
and a button injected straight into the vanilla tool panel.

## Architecture at a glance

Two ECS systems, registered in `Mod.OnLoad` at two distinct phases - a clean
data/UI split (repo/SmoothLHT/Mod.cs#L26-L27):

```csharp
updateSystem.UpdateAt<InvertPrefabLHTSystem>(SystemUpdatePhase.PrefabUpdate);
updateSystem.UpdateAt<InvertPrefabUISystem>(SystemUpdatePhase.UIUpdate);
```

- `InvertPrefabLHTSystem` (`SystemUpdatePhase.PrefabUpdate`) owns all prefab
  mutation. Its `OnUpdate` is empty; the work runs on lifecycle hooks instead -
  both `OnWorldReady` and `OnGamePreload` call `InvertAllPrefabs`
  (repo/SmoothLHT/Systems/InvertPrefabLHTSystem.cs#L49-L65). Re-applying on every
  world-ready is deliberate: it re-honors the player's opt-outs each load without
  any per-instance saved state.
- `InvertPrefabUISystem` (`SystemUpdatePhase.UIUpdate`, a `UISystemBase`) owns
  the C#->React bindings and the per-prefab toggle
  (repo/SmoothLHT/Tools/InvertPrefabUISystem.cs#L19-L31).

`InvertAllPrefabs` queries every `PrefabData` entity carrying `BuildingData` or
`BuildingExtensionData`, loads the preference store, runs the scanner, caches the
results (`InvertibleAssets`, `buildingUpgrades`), then applies the preferred mode
to each scanned prefab (repo/SmoothLHT/Systems/InvertPrefabLHTSystem.cs#L121-L148).
Data flow: **query -> scan (eligibility + upgrade map) -> load preferences ->
apply (mutate + UpdatePrefab) -> save**.

## Techniques demonstrated

- [Prefab-field override](../how-to/recipes/prefab-field-override.md) (family A) -
  the whole mod pivots on one guarded field write. `UpdatePrefabInvertMode` only
  touches a prefab when the value actually differs, then re-bakes via
  `PrefabSystem.UpdatePrefab` (repo/SmoothLHT/Services/PrefabInvertService.cs#L68-L78):

  ```csharp
  private int UpdatePrefabInvertMode(PrefabBase prefab, NetInvertMode invertMode)
  {
      if (prefab.TryGet(out ObjectSubNets subNets) && subNets.m_InvertWhen != invertMode)
      {
          subNets.m_InvertWhen = invertMode;
          prefabSystem.UpdatePrefab(prefab);
          return 1;
      }

      return 0;
  }
  ```

  Note the variant of family A on display: SLHT mutates an **existing managed
  component** on the `PrefabBase` (`ObjectSubNets`, fetched with `TryGet`) and
  calls `UpdatePrefab` to re-bake it into the ECS world - it does **not**
  `AddComponentData` a new component onto an entity. `UpdatePrefab` is the
  re-bake path that propagates the changed prefab definition to placed instances;
  that is what makes an edit to a shared prefab retroactively affect every
  building already in the city. The `!= invertMode` guard keeps the pass
  idempotent so the every-load re-apply is cheap and side-effect-free when
  nothing changed.

- [Prefab-subnet traversal + host->upgrade propagation](../how-to/recipes/prefab-subnet-traversal.md)
  (family BG) - eligibility is decided by walking the subnet geometry graph, not
  by a name list. `InvertiblePrefabScanner` accepts a prefab only if it is a
  `BuildingPrefab`/`BuildingExtensionPrefab`, is not an ignored prefix, has
  `ObjectSubNets`, and carries a supported transport kind
  (repo/SmoothLHT/Services/InvertiblePrefabScanner.cs#L51-L87). The transport
  check descends `ObjectSubNets.m_SubNets -> NetGeometryPrefab.m_Sections ->
  m_Section.m_Pieces -> NetPieceLanes.m_Lanes`, inspecting each lane's `CarLane`
  (Car/Bicycle `RoadTypes`) or `TrackLane` (Tram `TrackTypes`) to build a
  `SupportedTransportKinds` flag set
  (repo/SmoothLHT/Services/InvertiblePrefabScanner.cs#L102-L179). Separately it
  maps upgrades to their host buildings via `ServiceUpgrade.m_Buildings`
  (repo/SmoothLHT/Services/InvertiblePrefabScanner.cs#L181-L210), and
  `PrefabInvertService.ApplyInvertModeRecursively` walks host->upgrades with a
  visited-name `HashSet` so a shared or cyclic upgrade graph is traversed exactly
  once (repo/SmoothLHT/Services/PrefabInvertService.cs#L40-L66).

- [Augment vanilla UI via moduleRegistry.extend](../how-to/recipes/vanilla-ui-augmentation.md)
  (family AU) - SLHT is the simplest canonical example of injecting a button into
  a stock panel. `index.tsx` resolves the real `Section` and `ToolButton`
  components (and the tool-button CSS class) through the official
  `ModuleRegistry.get`, then appends its own section by extending the vanilla
  module (repo/SmoothLHT/UI/SmoothLHT/src/index.tsx#L8-L34):

  ```tsx
  const register: ModRegistrar = (moduleRegistry) => {
      const toolOptionsComponents = resolveToolOptionsComponents(moduleRegistry);
      moduleRegistry.extend(
          MOUSE_TOOL_OPTIONS_MODULE,
          "MouseToolOptions",
          createInvertLHTTool(toolOptionsComponents)
      );
  };
  ```

  The extend callback clones the vanilla result and appends one `<Section>` +
  `<ToolButton>` only while the `IsShowing` binding is true
  (repo/SmoothLHT/UI/SmoothLHT/src/mods/InvertLHTTool.tsx#L52-L110). Reusing the
  game's own `Section`/`ToolButton`/CSS means the toggle inherits vanilla
  selected-state styling and ships no bespoke SCSS. The C# side backs it with
  three bindings (`ValueBinding<bool> IsShowing`, `ValueBinding<int> IsInverted`,
  `TriggerBinding<int> ToggleInverted`) driven by `ToolSystem.EventToolChanged` /
  `EventPrefabChanged`, showing only for `ObjectToolSystem`/`UpgradeToolSystem`
  when the selected prefab is invertible
  (repo/SmoothLHT/Tools/InvertPrefabUISystem.cs#L19-L68).

- [External JSON side-car persistence](../how-to/recipes/json-sidecar-persistence.md)
  (family F) - preferences live in a `Colossal.Json` file, not in the save.
  `InvertPreferenceStore` reads/writes `ModsData/SmoothLHT/non_inverted_assets.json`
  under `EnvPath.kUserDataPath` via `JSON.Load`/`JSON.Dump`
  (repo/SmoothLHT/Services/InvertPreferenceStore.cs#L12-L65). The stored model is
  an **opt-out set**: a `HashSet<string>` of prefab names the player wants left
  *un*-inverted. Absence means `NetInvertMode.LefthandTraffic`, presence means
  `NetInvertMode.Never` (repo/SmoothLHT/Services/InvertPreferenceStore.cs#L67-L72):

  ```csharp
  public NetInvertMode GetDesiredInvertMode(string prefabName)
  {
      return NonInvertedAssets.Contains(prefabName)
          ? NetInvertMode.Never
          : NetInvertMode.LefthandTraffic;
  }
  ```

  On a missing file or read/parse failure the store falls back to a seeded
  default set (three bus-station prefabs) and logs, so a corrupt side-car
  degrades gracefully instead of throwing
  (repo/SmoothLHT/Services/InvertPreferenceStore.cs#L25-L51;
  repo/SmoothLHT/Systems/InvertPrefabLHTSystem.cs#L23-L28).

## Key decisions & tradeoffs

- **Prefab-level, not per-instance.** SLHT never writes a component onto placed
  building entities; it edits the shared prefab and lets `UpdatePrefab` propagate
  to instances (repo/SmoothLHT/Services/PrefabInvertService.cs#L68-L78). The
  payoff is zero save-game footprint and automatic coverage of buildings placed
  later; the cost is that the edit is global to a prefab (last-writer-wins - see
  Pitfalls) and there is no per-building override.
- **Delegate RHT-safety to `NetInvertMode`, don't compute it.** The mod applies
  the same `m_InvertWhen = LefthandTraffic` on every save and relies on the
  engine to only actually flip geometry when the city is LHT. This replaced the
  old v0.2.3 `CityConfigurationSystem.leftHandTraffic` early-return gate; the live
  code queries no `CityConfigurationSystem` and has no `leftHandTraffic` gate at
  all - the desired mode is a flat `LefthandTraffic`-unless-opted-out lookup
  (repo/SmoothLHT/Services/InvertPreferenceStore.cs#L67-L72). Fewer moving parts,
  but it hinges on the engine keeping that enum contract.
- **Re-apply every load rather than persist a flag.** `OnWorldReady` and
  `OnGamePreload` both re-run the full scan+apply
  (repo/SmoothLHT/Systems/InvertPrefabLHTSystem.cs#L53-L65). This guarantees
  opt-outs are re-honored after any prefab reload and removes the fragile
  one-shot `isAllInverted` guard of older versions; the `!= invertMode` guard
  keeps the repeated pass cheap.
- **Opt-out model, not opt-in.** Persisting only the *exceptions* means new
  eligible prefabs (from DLC or other mods) are inverted by default without any
  migration step, and the on-disk file stays tiny
  (repo/SmoothLHT/Services/InvertPreferenceStore.cs#L74-L84).
- **No Harmony.** The UI is added by the official `moduleRegistry.extend`
  contract and the data change is a plain prefab mutation
  (repo/SmoothLHT/UI/SmoothLHT/src/index.tsx#L29-L33) - no patching of game
  methods anywhere, which keeps the mod resilient to `Game.dll` internal changes.
- **A narrow policy gate.** `InvertModePolicy.IsSupported` accepts only `Never`
  and `LefthandTraffic`, and it is enforced at both the system entry and the UI
  trigger, so a malformed value from either boundary is rejected rather than
  written (repo/SmoothLHT/Services/InvertModePolicy.cs#L7-L11;
  repo/SmoothLHT/Tools/InvertPrefabUISystem.cs#L41-L46).

## Pitfalls / upstream-watch

- **Last-writer-wins on `m_InvertWhen`.** There is no merge logic. Any other mod
  that rewrites `ObjectSubNets` or re-registers building prefabs during
  `PrefabUpdate` can clobber (or be clobbered by) this edit. Because SLHT
  re-applies on `OnWorldReady` (after preload) it is *likely* to win for most load
  orders, but that is load-order-dependent and not source-provable
  (repo/SmoothLHT/Services/PrefabInvertService.cs#L68-L78).
- **MouseToolOptions co-existence with other tool mods.** The button is appended
  into the vanilla `MouseToolOptions` module
  (repo/SmoothLHT/UI/SmoothLHT/src/index.tsx#L29-L33). A 2025 community report
  (Reddit thread `1os138y`) described Traffic Manager / Road Builder buttons going
  non-interactive alongside SLHT + a UK Pack; that report predates this UI design
  and the shared tool-options surface remains a plausible interaction point.
  Label: `Needs Verification (in-game)` -
  (repo dossier notes/community-intel-20251110.md).
- **Package hygiene (fixed, keep watching).** An earlier build accidentally
  bundled a game assembly (`Colossal.PSI.Common.dll`), inflating the package. At
  this pin the reference is correctly marked `<Private>false</Private>` so it is
  not copied into the mod output (repo/SmoothLHT/SmoothLHT.csproj#L16-L19). Any
  new game-assembly `<Reference>` added without `<Private>false</Private>` would
  reintroduce the same bloat.
- **Hard-coded skip lists.** Both the ignored-prefix list and the default
  non-inverted set are literal prefab-name arrays
  (repo/SmoothLHT/Systems/InvertPrefabLHTSystem.cs#L14-L28). If Colossal renames
  those prefabs (e.g. `BusStation01`) the skips silently stop matching; these
  names are an upstream-watch item on each CS2 content patch.

Needs Verification (requires the running game, cannot be confirmed from source):
whether the every-load re-apply reliably wins ordering against other
subnet-editing mods; and the current status of the Traffic Manager / Road Builder
MouseToolOptions interaction report.

## Source pointers

- Dossier:
  `../../vice-and-order-research/mods/dossiers/smooth-left-hand-traffic/`
  (`index.md`, `source.md`, `modding.md`, `guide.md`, `notes/`).
- Repo @ `37b2850b10ced1cd17457c25778409e57e00a03e` (tag `v0.5.1`), key files:
  - `repo/SmoothLHT/Mod.cs` - two-system registration at `PrefabUpdate` /
    `UIUpdate`.
  - `repo/SmoothLHT/Systems/InvertPrefabLHTSystem.cs` - lifecycle-driven
    scan+apply orchestration, seed lists.
  - `repo/SmoothLHT/Services/PrefabInvertService.cs` - guarded `UpdatePrefab`
    field write + host->upgrade recursion.
  - `repo/SmoothLHT/Services/InvertiblePrefabScanner.cs` - subnet-graph
    eligibility scan + upgrade mapping.
  - `repo/SmoothLHT/Services/InvertPreferenceStore.cs` - opt-out-only
    `Colossal.Json` side-car.
  - `repo/SmoothLHT/Services/InvertModePolicy.cs` - accepted-mode gate.
  - `repo/SmoothLHT/Tools/InvertPrefabUISystem.cs` - C#->React bindings + tool
    events.
  - `repo/SmoothLHT/UI/SmoothLHT/src/index.tsx`,
    `.../mods/InvertLHTTool.tsx` - `moduleRegistry.get`/`extend` augmentation of
    `MouseToolOptions`.
