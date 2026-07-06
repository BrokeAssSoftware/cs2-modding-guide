---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Case study: Market Based Economy"
case_study: market-based-economy
mod: "Market Based Economy"
dossier: ../../vice-and-order-research/mods/dossiers/market-based-economy/
repo_commit: b83f196a36bc74388accebdeb7c81f0f35dbab37
source_version: "~1.4.x (market-based-economy@b83f196; date-pinned, static source only)"
last_reverified: "2026-07-05"
diataxis: explanation
techniques: [C, AO, AY]
technique_applicability: [economy, simulation]
status: source-verified
Created: 2026-07-05
Updated: 2026-07-05
Owners:
  - codex
---

# Market Based Economy - case study

> An elastic-pricing economy layer that never subclasses or replaces a vanilla
> system: it wraps four `EconomyUtils` price getters with hand-applied Harmony
> postfixes, drives them from a standalone singleton price manager backed by
> `NativeHashMap` supply/demand state, and re-orders three of its own ECS systems
> to run *just before* the vanilla wage, export, and tax systems that consume
> their output.

All code claims below are cited to `repo/<path>#Lxx` at pinned commit
`b83f196a36bc74388accebdeb7c81f0f35dbab37` (annotated tag `0.3.1`, modVersion 7,
requiredVersion `1.4.*`), surfaced through the dossier at
`../../vice-and-order-research/mods/dossiers/market-based-economy/`. This is the
**oldest pin in the corpus** (~1.4.x era) against a current live game of
1.6.0f1, so its economy touch-points carry the most drift risk - see
Pitfalls / upstream-watch.

## What it does / why it's instructive

Market Based Economy (MBE, author "andrew", Harmony id
`com.andrew.marketbasedeconomy`, repo/Mod.cs#L22) turns CS2's largely static
resource prices into a supply-and-demand-elastic market. Every price the game
asks for - the pooled market price plus the industrial and service components -
is re-derived from a demand/supply ratio, clamped to a configurable band around
a vanilla (or "real-world baseline") reference, and blended back toward that
reference by an external-trade weight. On top of pricing it adds a workforce
floor, a labor-driven wage rewrite, an experimental profit-based company tax, and
an in-game IMGUI analytics overlay.

It is instructive as the **"wrap, don't replace"** counterpoint to a
system-replacement mod. MBE owns zero vanilla systems: it never sets
`Enabled = false` on a Colossal system and ships no decompiled clones. Instead it
teaches four combinable techniques that let a mod steer a subsystem it does not
own: (1) precise Harmony postfixes bound by overload signature, (2) a plain-C#
singleton manager holding all mutable state outside ECS, (3) scheduling its own
systems into the vanilla update order with `UpdateBefore`, and (4) a
self-hosted diagnostics UI. The dossier confirms the exact footprint: **5 manual
postfixes, `PatchAll` matching zero attribute classes**, 8 mod ECS systems, and 3
manager singletons.

## Architecture at a glance

### Load sequence (`Mod.OnLoad`)

`Mod.OnLoad` wires three of the mod's systems into `SystemUpdatePhase.GameSimulation`
via `UpdateBefore`, each anchored to the vanilla system that will consume its
writes (repo/Mod.cs#L54-L56):

```csharp
updateSystem.UpdateBefore<WageAdjustmentSystem, PayWageSystem>(SystemUpdatePhase.GameSimulation);
updateSystem.UpdateBefore<MarketProductSystem, ResourceExporterSystem>(SystemUpdatePhase.GameSimulation);
updateSystem.UpdateBefore<CompanyProfitAdjustmentSystem, TaxSystem>(SystemUpdatePhase.GameSimulation);
```

It then initializes the real-world baseline feature and, last of all, applies
the Harmony patches (repo/Mod.cs#L58-L63). There is **no `OnDispose` unpatch**;
the mod deliberately leaves itself patched for the session
(repo/Mod.cs#L66-L79).

### The Harmony surface (family C + AO)

Every patch is applied **imperatively**, not through attributes.
`HarmonyBridge.ApplyAll` calls `HarmonyInstance.PatchAll(assembly)` - but there
is not a single `[HarmonyPatch]` class in the assembly, so that call binds
nothing; the real work is the four `ApplyMarket...` / `ApplyWorkforce...` calls
that follow (repo/Harmony/HarmonyBridge.cs#L25-L39):

```csharp
HarmonyInstance.PatchAll(typeof(HarmonyBridge).Assembly);   // matches ZERO classes
ApplyMarketPricePostfix();
ApplyMarketPriceEntityManagerPostfix();
ApplyMarketPriceComponentPostfixes();
ApplyWorkforceMaintenancePostfix();
```

Each target is resolved by `AccessTools.Method` with an explicit parameter-type
array, which is how the mod picks one specific overload out of a family. The
pooled-price patch targets the `ComponentLookup<ResourceData>`-by-ref overload of
`EconomyUtils.GetMarketPrice` (repo/Harmony/HarmonyBridge.cs#L45-L58); a second
patch targets the `EntityManager` overload of the same method
(repo/Harmony/HarmonyBridge.cs#L72-L85); and a third pair patches
`GetIndustrialPrice` and `GetServicePrice`
(repo/Harmony/HarmonyBridge.cs#L99-L129). Four price getters, four postfixes,
each a one-line reroute into the manager (repo/Harmony/HarmonyBridge.cs#L161-L220).

The fifth patch is a different family entirely: a postfix on the vanilla
`WorkProviderSystem.OnUpdate`, resolved by name (repo/Harmony/HarmonyBridge.cs#L138-L151),
whose body simply forwards the patched system instance to the workforce manager
(repo/Harmony/HarmonyBridge.cs#L222-L225).

`ResourceBuyerPatches` and `ResourceExporterPatches` exist but are **inert
scaffolding** - each locates a nested vanilla job type then logs
"instrumentation disabled; no ... patches applied" and returns without patching
(repo/Harmony/ResourceBuyerPatches.cs#L17-L34; repo/Harmony/ResourceExporterPatches.cs#L17-L34).

### The elastic price manager (`MarketEconomyManager`)

All mutable pricing state lives in a `sealed` lazily-initialized singleton
outside ECS (repo/Economy/MarketEconomyManager.cs#L17-L19), holding its
supply/demand and price-state maps as `NativeHashMap`s
(repo/Economy/MarketEconomyManager.cs#L26-L28) allocated `Allocator.Persistent`
and sized to `EconomyUtils.ResourceCount`
(repo/Economy/MarketEconomyManager.cs#L111-L112). Tunables are plain properties
with defaults: `MaximumPriceMultiplier = 0.2f` (the deviation clamp),
`Sensitivity = 0.65f`, `ExternalPriceInfluence = 0.6f`, and
`PriceAnchoringStrength = 0.1f`
(repo/Economy/MarketEconomyManager.cs#L120-L155).

`ComputeElasticPrice` is the core (repo/Economy/MarketEconomyManager.cs#L276-L335):
it forms a demand/supply `ratio` (repo/Economy/MarketEconomyManager.cs#L290),
raises the baseline by `ratio` to a sensitivity-mapped exponent
(repo/Economy/MarketEconomyManager.cs#L294), pulls that raw price partway back to
the baseline by the anchoring strength
(repo/Economy/MarketEconomyManager.cs#L297), then clamps into a logistic band
whose width is set by `MaximumPriceMultiplier` and finally lerps toward the
baseline again by the external-trade weight before a hard clamp
(repo/Economy/MarketEconomyManager.cs#L328-L330). Supply/demand is discovered by
`TryGetSupplyDemand`, which reads live demand/supply systems or an aggregated
snapshot and normalizes toward storage-driven surplus/deficit ratios
(repo/Economy/MarketEconomyManager.cs#L844-L906).

> Naming note: the case-study task calls the external anchor
> `ExternalMarketWeight`. At this commit that is the **settings-UI label** - the
> `Setting.ExternalMarketWeight` property (default 0.6, clamped 0-1) wires
> directly to `MarketEconomyManager.ExternalPriceInfluence`
> (repo/Setting.cs#L73-L76). The band clamp is exposed as
> `Setting.MaximumPriceMultiplier` -> `MarketEconomyManager.MaximumPriceMultiplier`
> (repo/Setting.cs#L82-L85). The two anchors that pull elastic prices back toward
> the reference are `PriceAnchoringStrength` (toward baseline) and
> `ExternalPriceInfluence` (toward vanilla/external), applied in that order.

### The workforce floor and re-ordered systems (family AO + scheduling)

The `WorkProviderSystem.OnUpdate` postfix forwards into
`WorkforceUtilizationManager.ApplyPostUpdate`
(repo/Harmony/HarmonyBridge.cs#L222-L225), a singleton
(repo/Economy/WorkforceUtilizationManager.cs#L19-L33) that re-queries every work
provider *after* the vanilla system has written it, computes a per-building
minimum staffing target, and **raises `m_MaxWorkers` up to that floor** when the
vanilla value is lower (repo/Economy/WorkforceUtilizationManager.cs#L100-L108).
The floor is a quarter of the building's fitting-worker capacity
(repo/Economy/WorkforceUtilizationManager.cs#L153-L159). Writes are staged
through the `EndFrameBarrier` command buffer rather than mutating in place, to
respect ECS structural-change safety
(repo/Economy/WorkforceUtilizationManager.cs#L72-L95).

Two more systems ride the vanilla update order instead of a postfix.
`WageAdjustmentSystem` (an ordinary `SystemBase`) rewrites the singleton
`EconomyParameterData` wage fields from the labor market **before** vanilla
`PayWageSystem` reads them (repo/Economy/WageAdjustmentSystem.cs#L26-L58), which
is exactly why its `UpdateBefore<WageAdjustmentSystem, PayWageSystem>` ordering
matters. `CompanyProfitAdjustmentSystem` recomputes each company's untaxed income
and average tax rate before vanilla `TaxSystem`
(repo/Economy/CompanyProfitAdjustmentSystem.cs#L198-L262); it is gated off by
default (`FeatureEnabled = false`, repo/Economy/CompanyProfitAdjustmentSystem.cs#L23,
L85-L88) and runs a `WithoutBurst().Run()` main-thread `ForEach`
(repo/Economy/CompanyProfitAdjustmentSystem.cs#L135-L273).

### The analytics overlay + recorder (family AY)

`EconomyAnalyticsRecorder` is a lock-guarded singleton that samples wages and
per-resource prices on a 100 ms interval (`kWageSampleInterval` /
`kPriceSampleInterval = 0.1f`) into managed lists capped at
`kDefaultSampleCap = 2048`, trimmed front-to-back like a ring
(repo/Analytics/EconomyAnalyticsRecorder.cs#L13-L15,
repo/Analytics/EconomyAnalyticsRecorder.cs#L221-L235). `ComputeElasticPrice`
feeds it every final price (repo/Economy/MarketEconomyManager.cs#L334) and
`WageAdjustmentSystem` feeds it one wage sample per update
(repo/Economy/WageAdjustmentSystem.cs#L46-L55).

The UI is a self-hosted `MonoBehaviour` overlay
(repo/Analytics/EconomyAnalyticsOverlay.cs#L12) drawn in `OnGUI` through
`GUI.Window` (repo/Analytics/EconomyAnalyticsOverlay.cs#L132-L141) with three
tabs - `Live`, `Wages`, `Prices`
(repo/Analytics/EconomyAnalyticsOverlay.cs#L43). A hotkey companion polls the
game `InputManager` and toggles visibility
(repo/Analytics/EconomyAnalyticsHotkey.cs#L25-L42); the binding is the settings
action `ToggleAnalyticsOverlay` bound to **Shift+G**
(repo/Setting.cs#L133). Both components are attached to a single
`DontDestroyOnLoad` GameObject created once by `EconomyAnalyticsOverlayHost.Ensure`
(repo/Analytics/EconomyAnalyticsOverlayHost.cs#L12-L34), which `Mod.OnLoad`
calls before applying Harmony (repo/Mod.cs#L62).

## Techniques demonstrated

- [Harmony postfix on economy/price getters](../how-to/recipes/harmony-price-getter-postfix.md)
  (family C) - four `EconomyUtils` price getters (`GetMarketPrice` x2 overloads,
  `GetIndustrialPrice`, `GetServicePrice`) are wrapped with postfixes that
  overwrite `ref float __result` with a manager-computed elastic price, each
  bound by explicit-signature `AccessTools.Method` overload resolution
  (repo/Harmony/HarmonyBridge.cs#L41-L129, repo/Harmony/HarmonyBridge.cs#L161-L220).
- [Harmony postfix on a vanilla System.OnUpdate](../how-to/recipes/harmony-system-onupdate-postfix.md)
  (family AO) - a postfix on `WorkProviderSystem.OnUpdate` re-reads and raises
  `m_MaxWorkers` to a computed floor via the `EndFrameBarrier` command buffer
  (repo/Harmony/HarmonyBridge.cs#L138-L151, repo/Harmony/HarmonyBridge.cs#L222-L225;
  repo/Economy/WorkforceUtilizationManager.cs#L72-L108).
- [IMGUI debug/analytics overlay + sample recorder](../how-to/recipes/imgui-debug-overlay.md)
  (family AY) - a `DontDestroyOnLoad` `OnGUI` overlay with Live/Wages/Prices tabs
  driven by a 100 ms-throttled, 2048-cap ring recorder, toggled by a Shift+G
  settings binding (repo/Analytics/EconomyAnalyticsOverlay.cs#L132-L141;
  repo/Analytics/EconomyAnalyticsRecorder.cs#L13-L15,
  repo/Analytics/EconomyAnalyticsRecorder.cs#L221-L235; repo/Setting.cs#L133).

Supporting technique on display (the pattern behind the frontmatter's fourth
family tag): **`UpdateBefore` interleaving into the vanilla update order.** The
mod schedules `WageAdjustmentSystem`, `MarketProductSystem`, and
`CompanyProfitAdjustmentSystem` to run immediately before the vanilla
`PayWageSystem`, `ResourceExporterSystem`, and `TaxSystem` respectively so their
writes land before consumption (repo/Mod.cs#L54-L56). See the
[ECS system replacement & ordering](../how-to/recipes/ecs-system-replacement-ordering.md)
recipe for the general ordering technique.

> Needs Verification: the frontmatter tags this mod with technique family **E**
> ("ECS ISerializable save data / SaveVersion"). At this pin the mod ships **no
> serialization surface** - there is no `Colossal.Serialization`, `ISerializable`,
> `IJsonWritable`, or `GetVersion()` anywhere in the assembly (verified by
> repo-wide search at `b83f196`). Its state is deliberately transient: the price
> and supply/demand `NativeHashMap`s are `Allocator.Persistent` but rebuilt each
> session and cleared by `ResetCaches`, not written to the save
> (repo/Economy/MarketEconomyManager.cs#L111-L112). Treat family E as **not
> demonstrated by this mod**; settings persist via the standard
> `AssetDatabase.global.LoadSettings` options store (repo/Mod.cs#L46), not ECS
> save data.

## Key decisions & tradeoffs

- **Wrap, don't replace.** MBE never disables a vanilla system. It intercepts at
  the two seams a code mod can reach cleanly: the pure `EconomyUtils` price
  functions (postfix the return) and `WorkProviderSystem.OnUpdate` (postfix and
  re-read). Everything else is its own system scheduled around the vanilla ones.
  This keeps the per-patch diff surface tiny compared to a decompiled clone, at
  the cost of being at the mercy of those method signatures
  (repo/Harmony/HarmonyBridge.cs#L25-L39).
- **Imperative `Patch`, not `PatchAll`.** Because the price getters are
  overloaded, the mod resolves each target by an explicit parameter-type array
  and calls `HarmonyInstance.Patch` directly; the `PatchAll` call is vestigial
  and binds nothing (repo/Harmony/HarmonyBridge.cs#L32,
  repo/Harmony/HarmonyBridge.cs#L45-L58). The reusable lesson: to hit one
  specific overload, prefer imperative `AccessTools.Method(type, name, types[])`
  over attribute discovery.
- **State outside ECS.** All pricing/workforce state lives in plain-C# singletons
  (`MarketEconomyManager`, `WorkforceUtilizationManager`,
  `EconomyAnalyticsRecorder`) reachable from both the Harmony postfixes and the
  mod's systems, avoiding the need to thread components through queries
  (repo/Economy/MarketEconomyManager.cs#L17-L19;
  repo/Economy/WorkforceUtilizationManager.cs#L19-L21).
- **Order by consumer.** Rather than an update group, each mod system declares
  `UpdateBefore<Self, VanillaConsumer>` so its writes are guaranteed fresh when
  the vanilla system reads them next tick (repo/Mod.cs#L54-L56).
- **Ship the experimental feature dark.** The profit-based company tax is fully
  implemented but `FeatureEnabled` defaults to `false` and its `OnUpdate`
  early-returns until toggled on - a deliberate opt-in for an unmeasured
  main-thread `WithoutBurst` pass over every company
  (repo/Economy/CompanyProfitAdjustmentSystem.cs#L23, L85-L88, L135-L273).

## Pitfalls / upstream-watch

- **`EconomyParameterData` is drift-prone.** `WageAdjustmentSystem` reads and
  rewrites the vanilla `EconomyParameterData` wage fields
  (repo/Economy/WageAdjustmentSystem.cs#L41-L55); Colossal has reworked economy
  parameters across patches, so this component's shape is the single most
  fragile dependency in the mod. At the ~1.4.x pin this is the highest-risk
  surface against live 1.6.0f1.
- **Overload-signature fragility.** All four price postfixes bind by explicit
  parameter-type arrays against `EconomyUtils` overloads
  (repo/Harmony/HarmonyBridge.cs#L45-L58, repo/Harmony/HarmonyBridge.cs#L99-L129);
  if Colossal changes any signature the patch logs "target ... not found;
  skipping" and silently stops adjusting that price
  (repo/Harmony/HarmonyBridge.cs#L52-L55).
- **WorkProvider / Employee shared-buffer load order (soft conflict).** The
  workforce floor re-queries every `WorkProvider` and its `Employee` buffer after
  the vanilla system runs (repo/Economy/WorkforceUtilizationManager.cs#L43-L93);
  another mod that also writes `m_MaxWorkers` or the employee buffer (e.g. a job
  search / hiring mod) can be overwritten depending on which postfix runs last -
  a load-order-sensitive soft conflict, not a hard crash.
- **Experimental profit-tax cost is unmeasured.** When enabled,
  `CompanyProfitAdjustmentSystem` runs a `WithoutBurst().Run()` main-thread
  `ForEach` over the full company query every update
  (repo/Economy/CompanyProfitAdjustmentSystem.cs#L135-L273); its performance on
  large cities is not characterizable from source.
- **Oldest pin in the corpus.** This dossier is pinned at ~1.4.x
  (requiredVersion `1.4.*`), the oldest studied. Everything above should be
  re-verified against the current 1.6.0f1 `Game.dll` before relying on it.

Needs Verification (requires the running game, cannot be confirmed from source):
the profit-tax system's per-frame cost and city-scale impact; whether the
elastic band and anchoring defaults produce a stable market rather than
oscillation; and the real severity of the WorkProvider/Employee co-write
conflict with a live job-search mod.

## Source pointers

- Dossier: `../../vice-and-order-research/mods/dossiers/market-based-economy/`
  (`index.md`, `source.md`, `modding.md`, `guide.md`, `notes/`).
- Repo @ `b83f196a36bc74388accebdeb7c81f0f35dbab37` (tag `0.3.1`), key files:
  - `repo/Mod.cs` - `UpdateBefore` scheduling + Harmony apply, no unpatch.
  - `repo/Harmony/HarmonyBridge.cs` - 5 manual postfixes; `PatchAll` binds nothing.
  - `repo/Harmony/ResourceBuyerPatches.cs`, `.../ResourceExporterPatches.cs` -
    inert scaffolding.
  - `repo/Economy/MarketEconomyManager.cs` - elastic price singleton + NativeHashMap state.
  - `repo/Economy/WorkforceUtilizationManager.cs` - workforce floor via EndFrameBarrier.
  - `repo/Economy/WageAdjustmentSystem.cs`, `.../CompanyProfitAdjustmentSystem.cs` -
    re-ordered systems before `PayWageSystem` / `TaxSystem`.
  - `repo/Analytics/EconomyAnalyticsRecorder.cs`, `.../EconomyAnalyticsOverlay.cs`,
    `.../EconomyAnalyticsOverlayHost.cs`, `.../EconomyAnalyticsHotkey.cs` - IMGUI overlay + recorder.
  - `repo/Setting.cs` - settings-UI wiring (`ExternalMarketWeight` ->
    `ExternalPriceInfluence`) and the Shift+G overlay binding.
