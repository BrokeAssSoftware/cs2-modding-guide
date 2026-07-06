---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: Harmony postfix on economy / price getters"
recipe: harmony-price-getter-postfix
technique_family: "C - Harmony postfix on economy / price getters"
diataxis: how-to
source_version: "~1.4.x (market-based-economy@b83f196; date-pinned, static source only)"
last_reverified: "2026-07-04"
status: source-verified
canonical_mods:
  - market-based-economy@b83f196a36bc74388accebdeb7c81f0f35dbab37
  - time2work-realistic-trips@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39
technique_applicability: [economy]
Created: 2026-07-04
Updated: 2026-07-04
Owners:
  - codex
---

# Harmony postfix on economy / price getters

> Scale or replace the value a vanilla price helper returns (e.g. an `EconomyUtils`
> price getter) by attaching a Harmony postfix that rewrites `ref __result` - applied
> **manually** with `HarmonyInstance.Patch(...)`, not by attribute discovery.

## Problem
You want to change the number a vanilla static getter returns - a market price, an
industrial or service price - without forking the game's simulation. The value is
computed inside a method you do not own (`Game.Economy.EconomyUtils.GetMarketPrice`
and friends), it is read from many call sites, and there is no prefab field or
setting that lets you influence it. A Harmony **postfix** lets the original run and
then hands you its return value in `ref __result` to scale, clamp, or replace.

The complication: the getter you want is **overloaded** (several `GetMarketPrice`
signatures exist), so attribute-based `[HarmonyPatch(typeof(...), "GetMarketPrice")]`
is ambiguous, and Harmony cannot pick a target. You need to disambiguate by exact
parameter types.

## Solution
Resolve each overload explicitly with `AccessTools.Method(type, name, new Type[]{...})`,
resolve your postfix the same way, and apply the pair **manually** with
`HarmonyInstance.Patch(target, postfix: new HarmonyMethod(postfix))`. The postfix
signature takes `ref __result` (the return value) plus any original parameters you
name by their exact parameter names; Harmony injects them positionally-by-name. Inside
the postfix you overwrite `__result` with your adjusted value. This is the pattern
Market Based Economy uses for all four of its price postfixes - and, crucially, it
applies them by hand precisely **because** `PatchAll` cannot discover them (there are
no `[HarmonyPatch]` attribute classes in the mod).

## Steps & Code

### 1. Create one Harmony instance and drive both discovery paths

`ApplyAll` runs `PatchAll` (attribute discovery) **and then** four hand-applied
postfixes. Keep the two paths in mind - in this mod, `PatchAll` finds nothing (see
Pitfalls); the real work is the manual calls:

```csharp
private static readonly HarmonyLib.Harmony HarmonyInstance = new HarmonyLib.Harmony(Mod.HarmonyId);

public static void ApplyAll(string harmonyId)
{
    if (_patchesApplied) return;

    HarmonyInstance.PatchAll(typeof(HarmonyBridge).Assembly);  // discovers ZERO attribute patches here
    ApplyMarketPricePostfix();
    ApplyMarketPriceEntityManagerPostfix();
    ApplyMarketPriceComponentPostfixes();
    ApplyWorkforceMaintenancePostfix();

    _patchesApplied = true;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/market-based-economy/repo/Harmony/HarmonyBridge.cs#L16-L39` (@b83f196a36bc74388accebdeb7c81f0f35dbab37)

### 2. Resolve the exact overload with `AccessTools.Method(..., new Type[]{...})`

`GetMarketPrice` is overloaded. Select the one you want by passing its full parameter
type array - note `ComponentLookup<ResourceData>.MakeByRefType()` for the `ref`
parameter. Resolve the postfix method the same way, and null-check both before patching:

```csharp
var target = AccessTools.Method(
    typeof(EconomyUtils),
    nameof(EconomyUtils.GetMarketPrice),
    new[] { typeof(Resource), typeof(ResourcePrefabs), typeof(ComponentLookup<ResourceData>).MakeByRefType() });

var postfix = AccessTools.Method(typeof(HarmonyBridge), nameof(MarketPricePostfix));

if (target == null || postfix == null)
{
    Log.Warn("Market price patch target or postfix not found; skipping.");
    return;
}

HarmonyInstance.Patch(target, postfix: new HarmonyMethod(postfix));
```
Source: `../../../vice-and-order-research/mods/dossiers/market-based-economy/repo/Harmony/HarmonyBridge.cs#L45-L60` (@b83f196a36bc74388accebdeb7c81f0f35dbab37)

The sibling overload that takes an `EntityManager` instead of the `ref ComponentLookup`
is resolved by a *different* `Type[]` (`{ Resource, ResourcePrefabs, EntityManager }`)
and gets its own postfix - the `Type[]` is the only thing that disambiguates them
(`Harmony/HarmonyBridge.cs#L72-L85`).

### 3. Write the postfix: `ref __result` is the value to rewrite

The postfix names the return value as `ref float __result`; any original parameter you
want (here `Resource r`) is named to match. Overwrite `__result` with your adjusted
number - that becomes the getter's effective return:

```csharp
private static void MarketPricePostfix(Resource r, ref float __result)
{
    __result = Economy.MarketEconomyManager.Instance.AdjustMarketPrice(r, __result);
    if (!_marketPricePostfixLogged)
    {
        _marketPricePostfixLogged = true;
        DiagnosticsLogger.Log("Harmony", $"MarketPricePostfix invoked for {r}.");
    }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/market-based-economy/repo/Harmony/HarmonyBridge.cs#L161-L169` (@b83f196a36bc74388accebdeb7c81f0f35dbab37)

### 4. Pull extra ECS state into the postfix via named original parameters

A postfix can take the original method's other parameters by name to do a
component-driven adjustment. The industrial/service price postfixes accept
`ResourcePrefabs prefabs` and `ComponentLookup<ResourceData> resourceDatas`, guard
against a non-positive baseline, look up the resource entity, and rewrite `__result`
from the component's price band:

```csharp
private static void IndustrialPricePostfix(Resource r, ResourcePrefabs prefabs,
    ComponentLookup<ResourceData> resourceDatas, ref float __result)
{
    if (__result <= 0f) return;

    Entity entity = prefabs[r];
    if (!resourceDatas.HasComponent(entity)) return;

    var data = resourceDatas[entity];
    __result = Economy.MarketEconomyManager.Instance.AdjustPriceComponent(
        r, data.m_Price.x, data.m_Price.y,
        Economy.MarketEconomyManager.PriceComponent.Industrial, skipLogging: false);
}
```
Source: `../../../vice-and-order-research/mods/dossiers/market-based-economy/repo/Harmony/HarmonyBridge.cs#L176-L197` (@b83f196a36bc74388accebdeb7c81f0f35dbab37)

Both `GetIndustrialPrice` and `GetServicePrice` are resolved with the same
`ComponentLookup<ResourceData>.MakeByRefType()` `Type[]` and each gets its own postfix,
applied by hand in `ApplyMarketPriceComponentPostfixes`
(`Harmony/HarmonyBridge.cs#L99-L130`).

## Pitfalls & gotchas

- **`PatchAll` discovers nothing here - the manual `Patch()` calls are the whole point.**
  Market Based Economy has **no** `[HarmonyPatch]` attribute classes at all (grep the
  repo at the pin: `HarmonyPatch` yields zero hits). So `PatchAll(...Assembly)` in step 1
  finds and applies **zero** patches; every price postfix lives only because
  `ApplyMarketPricePostfix` / `...EntityManagerPostfix` / `...ComponentPostfixes` call
  `HarmonyInstance.Patch(target, postfix: new HarmonyMethod(...))` explicitly
  (`../../../vice-and-order-research/mods/dossiers/market-based-economy/repo/Harmony/HarmonyBridge.cs#L32-L35`).
  If you copy this shape, do not assume `PatchAll` is doing anything - it is the manual
  calls that patch the getters. This is the inverse of the attribute-driven style (see
  Variations).

- **Overloaded getters need the exact `Type[]`, including `MakeByRefType()`.**
  Passing just the method name to `AccessTools.Method` on an overloaded target returns
  an ambiguous/null match. You must pass the full parameter type array, and a `ref`
  parameter must be spelled `typeof(T).MakeByRefType()` (e.g.
  `ComponentLookup<ResourceData>.MakeByRefType()`), or the lookup silently returns
  `null` and the `if (target == null)` guard skips your patch with only a warning
  (`Harmony/HarmonyBridge.cs#L45-L56`).

- **A failed target resolution is a silent no-op, not a crash.** Each apply method
  wraps its work in `try/catch` and, on a null target/postfix, logs `Warn` and returns
  (`Harmony/HarmonyBridge.cs#L52-L56`, `#L112-L115`). If a future game patch renames or
  re-signs `GetMarketPrice`, your postfix just stops applying - the economy silently
  reverts to vanilla with no hard error. Watch the log for the "target or postfix not
  found" warnings.

- **Dormant patch class in the tree.** `Harmony/ResourceBuyerPatches.cs` exists but its
  `Apply` short-circuits - it finds the nested `BuyJob` type and then logs
  "BuyJob instrumentation disabled; no ResourceBuyer patches applied." and returns
  without patching anything, and `ApplyAll` never calls it
  (`../../../vice-and-order-research/mods/dossiers/market-based-economy/repo/Harmony/ResourceBuyerPatches.cs#L17-L29`).
  Do not mistake a file named `*Patches.cs` for an active patch; read whether it is
  wired into the apply path.

- **Postfix ordering / stacking across mods** (two mods postfixing the same getter,
  and whether your `AdjustMarketPrice` compounds another mod's rewrite) is not visible
  in this source: `Needs Verification (in-game)`.

## Variations

- **Attribute-discovered patches with `[HarmonyPatch]` + `PatchAll` (the opposite of
  this recipe).** Time2Work Realistic Trips declares an attribute class and lets
  `PatchAll` find it - no manual `Patch()` calls. A postfix is marked with
  `[HarmonyPostfix]` and the target with `[HarmonyPatch(typeof(...), "method")]`:

  ```csharp
  [HarmonyPatch]
  public class Time2WorkPatches
  {
      [HarmonyPatch(typeof(TimeSystem), "OnUpdate")]
      [HarmonyPostfix]
      public static void TimeSystemPatches_OnUpdate_Postfix(TimeSystem __instance)
      {
          Time2WorkTimeSystem t2wTimeSystem = World.DefaultGameObjectInjectionWorld
              .GetOrCreateSystemManaged<Time2WorkTimeSystem>();
          Traverse.Create(__instance).Field("m_Time").SetValue(t2wTimeSystem.normalizedTime);
          // ...
      }
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Patches/Time2WorkPatches.cs#L30-L42` (@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39)

- **Disambiguating overloads inside the attribute itself.** The same `Type[]` trick from
  step 2 also works as an attribute argument. Time2Work patches two `GetYear` overloads
  by putting the parameter-type array in the `[HarmonyPatch]` attribute (here as a
  prefix that overwrites `ref __result` and returns `false` to skip the original):

  ```csharp
  [HarmonyPatch(typeof(TimeSystem), "GetYear", new Type[] { typeof(TimeSettingsData), typeof(TimeData) })]
  [HarmonyPrefix]
  static bool TimeSystemPatches_GetYear(TimeSettingsData settings, TimeData data, ref int __result, TimeSystem __instance)
  {
      Time2WorkTimeSystem t2wTimeSystem = World.DefaultGameObjectInjectionWorld
          .GetOrCreateSystemManaged<Time2WorkTimeSystem>();
      __result = t2wTimeSystem.GetYear(settings, data);
      return false;
  }
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Patches/Time2WorkPatches.cs#L59-L68` (@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39)

  Choose the manual path (this recipe) when you want fine control over *which* patches
  apply and when; choose attribute discovery when you have many patches and want
  `PatchAll` to wire them for you. Both use the identical `Type[]` overload
  disambiguation.

- **Postfix vs. replacing prefix.** A postfix (`ref __result` after the original runs)
  scales/clamps the real computed value - use it when the vanilla number is a good
  baseline. A prefix that sets `__result` and `return false` (Time2Work above) *replaces*
  the computation entirely - use it when you have your own value and want to skip the
  original's cost.

## See also
- Concept: [Harmony patching](../../explanation/harmony-patching.md) (prefix/postfix/
  transpiler, discovery vs. manual apply).
- Reference: [Economy game systems](../../reference/game-systems/economy.md)
  (`EconomyUtils` price getters and resource price data).
- Related recipe: [Traverse private fields](harmony-traverse-private-fields.md)
  (reading/writing private state from inside a patch, as the Time2Work postfix does with
  `Traverse.Create(...).Field(...)`).
- Case study: [Time2Work Realistic Trips](../../case-studies/time2work-realistic-trips.md).

## Sources
- Canonical mods (dossier + repo):
  - `market-based-economy` @b83f196a36bc74388accebdeb7c81f0f35dbab37 - `repo/Harmony/HarmonyBridge.cs`, `repo/Harmony/ResourceBuyerPatches.cs`
  - `time2work-realistic-trips` @d42921ffb1f6bbbf2c2b086bb17727bf0f637c39 - `repo/NightShift/Patches/Time2WorkPatches.cs`
- Official/community references (link out, do not duplicate):
  https://harmony.pardeike.net/ , https://cs2.paradoxwikis.com/Modding
