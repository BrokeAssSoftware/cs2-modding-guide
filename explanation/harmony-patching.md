---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Harmony Patching in CS2"
Summary: "What prefix/postfix/transpiler/reverse-patch are, how a CS2 mod targets a method (PatchAll vs manual AccessTools.Method), and the idioms and hazards - explained against real cited mod source."
diataxis: explanation
source_version: "~1.4.x (market-based-economy@b83f196; date-pinned, static source only)"
last_reverified: "2026-07-04"
status: source-verified
canonical_mods:
  - market-based-economy@b83f196a36bc74388accebdeb7c81f0f35dbab37
  - write-everywhere@13c70eb04e6bed152257c516a982455148f591a5
  - anarchy@a6311e898d20a775368668b234aaa32f06e3e1eb
  - time2work-realistic-trips@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39
  - extra-detailing-tools@41df3c2b8b444197f7c3e2de4496c01cfd8bfc23
  - realistic-jobsearch@7a096b2ab974bb03cc4cf0937f250bf1d7671f31
technique_applicability: [core, economy, simulation]
Created: 2026-07-04
Updated: 2026-07-05
Owners:
  - codex
References:
  - Label: System Replacement (Harmony vs ECS-replacement tradeoff)
    Path: ./system-replacement.md
  - Label: Recipe - Harmony price-getter postfix
    Path: ../how-to/recipes/harmony-price-getter-postfix.md
  - Label: Recipe - Harmony redirector reverse-patch
    Path: ../how-to/recipes/harmony-redirector-reverse-patch.md
  - Label: Recipe - Harmony private-job postfix
    Path: ../how-to/recipes/harmony-private-job-postfix.md
  - Label: Recipe - Harmony Traverse private fields
    Path: ../how-to/recipes/harmony-traverse-private-fields.md
  - Label: Reference - Economy game systems
    Path: ../reference/game-systems/economy.md
---

# Harmony Patching in CS2

Cities: Skylines II ships as compiled .NET assemblies (`Game.dll` and friends). When the
behaviour you want to change lives *inside a method* - a price getter returns the wrong
number, a pathfind branch ignores your data, a clock reads the wrong time - and there is
no prefab field and no whole `GameSystemBase` to swap out, you splice your code into that
method at runtime with **Harmony**. Harmony is the IL-level detour library that every
serious CS2 mod carries; the game's own modding stack loads it for you.

This is a **concept** page: what the four patch kinds are, how you *point* a patch at a
target, the two or three idioms that recur across real mods, and the hazards that bite.
The click-by-click mechanics live in the how-to recipes linked at the end. For the
larger "should I patch a method at all, or replace the whole system?" decision, see
[System Replacement](./system-replacement.md) - method patching is the precise, brittle
option; system replacement is the heavy, all-or-nothing one.

## The four kinds of patch

Harmony can attach four kinds of code to a target method. You reach for a different one
depending on *how much* of the original you want to keep.

- **Postfix** - runs *after* the original. The original's return value is handed to you as
  `ref __result`, so you read or rewrite the answer without touching the original's logic.
  This is the safest and most common patch: the original still runs, you only massage its
  output. Reach for it first.
- **Prefix** - runs *before* the original. Returning `void` (or `true`) lets the original
  run afterwards; returning `false` **skips the original entirely**, and you become
  responsible for setting `ref __result`. A prefix is how you conditionally replace or
  short-circuit a method.
- **Transpiler** - rewrites the target's IL instruction stream. The surgical, fragile
  option: use it only when you must change logic *inside* a method that neither a prefix
  nor postfix can reach. Nothing on this page's canonical mods needs one, which is itself
  the lesson: transpilers are a last resort.
- **Reverse patch** - the inverse direction. Instead of injecting into a game method, you
  copy a (usually private) game method's body *out* into a stub you own, so you can call
  the original logic directly and stably even after other patches pile onto it. It is a
  "give me a clean handle to the vanilla implementation" tool, not a "change vanilla" tool.

Harmony exposes all four through one call. Write-Everywhere's vendored redirector wrapper
shows the prefix/postfix/transpiler triple in a single `Harmony.Patch` call, and the
reverse patch as a separate `Harmony.ReversePatch`:

```csharp
public void AddRedirect(MethodInfo oldMethod, MethodInfo newMethodPre, MethodInfo newMethodPost = null, MethodInfo transpiler = null)
{
    m_detourList.Add(Harmony.Patch(oldMethod,
        newMethodPre  != null ? new HarmonyMethod(newMethodPre)  : null,
        newMethodPost != null ? new HarmonyMethod(newMethodPost) : null,
        transpiler    != null ? new HarmonyMethod(transpiler)    : null));
    m_patches.Add(oldMethod);
}

public void AddReversePatch(MethodInfo targetMethod, MethodInfo ownMethod)
{
    Harmony.ReversePatch(targetMethod, new HarmonyMethod(ownMethod));
}
```
[write-everywhere repo/BelzontWE/Commons/Utils/Redirector.cs#L75-L87](../../vice-and-order-research/mods/dossiers/write-everywhere/repo/BelzontWE/Commons/Utils/Redirector.cs), commit `3698b64`

That `Redirector.cs` lives in **CS2-BelzontCommons, a git submodule** vendored into
Write-Everywhere at `BelzontWE/Commons` (its own pinned commit `3698b64`, distinct from
the outer mod's `13c70eb`). Vendoring the patch plumbing as a shared submodule is a common
pattern across a modder's family of mods - the citation must be to the submodule's commit.

## How you point a patch at a target

A patch is useless until Harmony knows *which* method to detour. There are two ways to
bind that target, and a real mod often uses both.

### Attribute discovery: PatchAll

You decorate a class (or method) with `[HarmonyPatch(typeof(X), "MethodName")]` plus
`[HarmonyPrefix]`/`[HarmonyPostfix]`, then call `PatchAll` once and let Harmony scan the
assembly for those attributes and wire them up. Time2Work's clock patches are pure
attribute style - the target and patch kind are declared inline:

```csharp
[HarmonyPatch(typeof(TimeSystem), "OnUpdate")]
[HarmonyPostfix]
public static void TimeSystemPatches_OnUpdate_Postfix(TimeSystem __instance)
{
    Traverse.Create(__instance).Field("m_Time").SetValue(t2wTimeSystem.normalizedTime);
    Traverse.Create(__instance).Field("m_Date").SetValue(t2wTimeSystem.normalizedDate);
    Traverse.Create(__instance).Field("m_Year").SetValue(t2wTimeSystem.year);
}
```
[time2work-realistic-trips repo/NightShift/Patches/Time2WorkPatches.cs#L34-L41](../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Patches/Time2WorkPatches.cs), commit `d42921f`

Attribute discovery is compact and self-documenting, but the target lookup happens by
string at scan time, so a signature that Harmony can't resolve simply does not get patched.

### Manual binding: AccessTools.Method + HarmonyInstance.Patch

When you need to compute the target at runtime, disambiguate an overload, or wrap the
patch in a `try/catch` with a fallback log, you resolve the `MethodInfo` yourself with
`AccessTools.Method(...)` and call `HarmonyInstance.Patch(target, postfix: ...)`
directly. Market-Based-Economy's `HarmonyBridge` is a **hybrid**: it calls `PatchAll`
*and then* applies every real patch manually.

```csharp
HarmonyInstance.PatchAll(typeof(HarmonyBridge).Assembly);
ApplyMarketPricePostfix();
ApplyMarketPriceEntityManagerPostfix();
ApplyMarketPriceComponentPostfixes();
ApplyWorkforceMaintenancePostfix();
```
[market-based-economy repo/Harmony/HarmonyBridge.cs#L32-L36](../../vice-and-order-research/mods/dossiers/market-based-economy/repo/Harmony/HarmonyBridge.cs), commit `b83f196`

The subtle, important detail: **that `PatchAll` discovers zero patches here.** This
assembly has no `[HarmonyPatch]` attribute classes at all - every actual patch is one of
the four manual `Apply...` calls that follow. The `PatchAll` line is inert habit. When you
read a mod, do not assume `PatchAll` is where the patches are; follow the manual
`HarmonyInstance.Patch` calls.

Each manual apply resolves its target explicitly, guards against a null lookup, and patches:

```csharp
var target = AccessTools.Method(
    typeof(EconomyUtils),
    nameof(EconomyUtils.GetMarketPrice),
    new[] { typeof(Resource), typeof(ResourcePrefabs), typeof(ComponentLookup<ResourceData>).MakeByRefType() });

var postfix = AccessTools.Method(typeof(HarmonyBridge), nameof(MarketPricePostfix));
if (target == null || postfix == null) { Log.Warn("...target or postfix not found; skipping."); return; }
HarmonyInstance.Patch(target, postfix: new HarmonyMethod(postfix));
```
[market-based-economy repo/Harmony/HarmonyBridge.cs#L45-L58](../../vice-and-order-research/mods/dossiers/market-based-economy/repo/Harmony/HarmonyBridge.cs), commit `b83f196`

### Overloaded and by-ref targets

`EconomyUtils.GetMarketPrice` is **overloaded** - there is a `ComponentLookup<ResourceData>`
version (above) and an `EntityManager` version. A name alone is ambiguous, so you pass the
exact parameter-type array as the third `AccessTools.Method` argument to pick one overload:

```csharp
var target = AccessTools.Method(
    typeof(EconomyUtils),
    nameof(EconomyUtils.GetMarketPrice),
    new[] { typeof(Resource), typeof(ResourcePrefabs), typeof(EntityManager) });
```
[market-based-economy repo/Harmony/HarmonyBridge.cs#L72-L77](../../vice-and-order-research/mods/dossiers/market-based-economy/repo/Harmony/HarmonyBridge.cs), commit `b83f196`

Two more rules the same file demonstrates: for a `ref`/`out`/`in` parameter, append
`.MakeByRefType()` to that parameter's type (the first `GetMarketPrice` overload above
takes `ComponentLookup<ResourceData>` by ref, hence `typeof(...).MakeByRefType()`); and
you can target a system's own `protected` method by string name -
`AccessTools.Method(typeof(WorkProviderSystem), "OnUpdate")` binds a postfix onto a live
DOTS system's update
([HarmonyBridge.cs#L142](../../vice-and-order-research/mods/dossiers/market-based-economy/repo/Harmony/HarmonyBridge.cs), commit `b83f196`).

## The postfix idiom: read or rewrite `__result`

The economy postfixes are the archetype. The original `GetMarketPrice` runs, its answer
arrives as `ref float __result`, and the postfix overwrites it with an adjusted value.
Nothing about the original's internals is touched:

```csharp
private static void MarketPricePostfix(Resource r, ref float __result)
{
    __result = Economy.MarketEconomyManager.Instance.AdjustMarketPrice(r, __result);
}
```
[market-based-economy repo/Harmony/HarmonyBridge.cs#L161-L163](../../vice-and-order-research/mods/dossiers/market-based-economy/repo/Harmony/HarmonyBridge.cs), commit `b83f196`

The same shape scales the industrial and service getters, which additionally receive the
original's `ResourcePrefabs` and `ComponentLookup<ResourceData>` parameters by name so the
postfix can re-derive context
([HarmonyBridge.cs#L176, #L199](../../vice-and-order-research/mods/dossiers/market-based-economy/repo/Harmony/HarmonyBridge.cs), commit `b83f196`).
Full walk-through: [harmony-price-getter-postfix](../how-to/recipes/harmony-price-getter-postfix.md).

## Scoping a postfix to one caller: StackFrame sniffing

A `[HarmonyPatch]` on a shared helper fires for *every* caller in the game - usually far
more surface than you want. When only one call site should see your changed answer, a
postfix can inspect the managed call stack and narrow itself to that site. Extra Detailing
Tools patches `GameModeExtensions.IsEditor` - a global "are we in the editor?" predicate
the game consults in many places - but forces it to `true` only when the *immediate
caller* is `NetToolSystem.GetNetPrefab`:

```csharp
[HarmonyPatch(typeof(GameModeExtensions), "IsEditor")]
public class IsEditor
{
    public static void Postfix(ref bool __result)
    {
        MethodBase caller = new StackFrame(2, false).GetMethod();
        if (caller.DeclaringType == typeof(NetToolSystem) && caller.Name == "GetNetPrefab")
            __result = true;
    }
}
```
[extra-detailing-tools repo/MOD/Patches/GameModeExtensions.cs#L11-L26](../../vice-and-order-research/mods/dossiers/extra-detailing-tools/repo/MOD/Patches/GameModeExtensions.cs), commit `41df3c2`

`new StackFrame(2, false)` walks two frames up - frame 0 is the postfix itself, frame 1 is
the patched `IsEditor`, frame 2 is whoever called it - and the guard keys on that method's
declaring type and name. Every other caller of `IsEditor` gets the vanilla answer; only the
net-tool prefab lookup is tricked into editor mode. This is the scalpel version of "patch a
shared method": instead of changing the answer for the whole game, you change it for exactly
one call path. The cost is fragility on two axes at once - the patch breaks if the game
renames *either* `IsEditor` or `GetNetPrefab`, and stack-frame depth is an implementation
detail that inlining can shift (a Burst/JIT-inlined intermediate frame silently changes what
frame `2` points at; see Hazards). The two commented-out sibling clauses in the source
(`ObjectToolSystem.GetObjectPrefab`, `DefaultToolSystem.InitializeRaycast`) are a live
reminder that this coupling is brittle across game versions.

## The prefix idiom: return false + set `__result` to replace a branch

When a postfix is not enough because you need to *skip* the original for some inputs, a
prefix returning `false` short-circuits it - and then **you** must fill `ref __result`.
Time2Work's leisure-bias patch is the clean example. It targets a private method resolved
with a static `TargetMethod()` (the method-side equivalent of the attribute's type/name),
using an `in` parameter expressed as `.MakeByRefType()`:

```csharp
static MethodBase TargetMethod()
{
    // private JobHandle FindTargets(SetupTargetType targetType, in PathfindSetupSystem.SetupData setupData)
    return AccessTools.Method(
        typeof(PathfindSetupSystem),
        "FindTargets",
        new[] { typeof(SetupTargetType), typeof(PathfindSetupSystem.SetupData).MakeByRefType() });
}
```
[time2work-realistic-trips repo/NightShift/Patches/PathfindSetupSystem_LeisureEventBiasPatch.cs#L43-L54](../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Patches/PathfindSetupSystem_LeisureEventBiasPatch.cs), commit `d42921f`

The prefix then handles *only* the `Leisure` branch: for anything else it returns `true`
to let vanilla run, but for leisure it schedules its own Burst job, writes the resulting
`JobHandle` into `ref __result`, and returns `false` so vanilla never executes:

```csharp
static bool Prefix(PathfindSetupSystem __instance, SetupTargetType targetType,
    ref PathfindSetupSystem.SetupData setupData, ref JobHandle __result)
{
    if (targetType != SetupTargetType.Leisure)
        return true;                 // run vanilla for everything else
    // ... build and schedule the replacement job ...
    __result = handle;
    return false;                    // skip vanilla leisure branch
}
```
[time2work-realistic-trips repo/NightShift/Patches/PathfindSetupSystem_LeisureEventBiasPatch.cs#L56-L63, #L131-L132](../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Patches/PathfindSetupSystem_LeisureEventBiasPatch.cs), commit `d42921f`

This "selective `return false` + set `__result`" is the idiom for replacing one path
through a method while leaving the rest of vanilla intact - a middle ground between a
postfix (change the output) and full system replacement (own everything). Because the
method returns a `JobHandle`, the prefix must preserve the job dependency chain it
replaces. Full write-up: [harmony-private-job-postfix](../how-to/recipes/harmony-private-job-postfix.md).

### Unconditional replacement: a prefix that always returns false

The leisure patch above is *selective* - it returns `true` for most inputs. The other end
of the spectrum is a prefix that returns `false` on **every** call, turning Harmony into a
whole-getter replacement: the vanilla body never runs and your value is the only value.
Time2Work does this across roughly eleven time getters, each a short prefix that computes
`__result` from its own `Time2WorkTimeSystem` and returns `false`:

```csharp
[HarmonyPatch(typeof(TimeSystem), "get_normalizedDate")]
[HarmonyPrefix]
static bool TimeSystemPatches_normalizedDate(ref float __result)
{
    Time2WorkTimeSystem t2wTimeSystem = World.DefaultGameObjectInjectionWorld
        .GetOrCreateSystemManaged<Time2WorkTimeSystem>();
    __result = t2wTimeSystem.normalizedDate;
    return false; // Skip original getter
}
```
[time2work-realistic-trips repo/NightShift/Patches/Time2WorkPatches.cs#L83-L90](../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Patches/Time2WorkPatches.cs), commit `d42921f`

The same shape repeats for `GetYear` (both overloads, disambiguated by the `new Type[]{...}`
argument), `GetDay`, `GetCurrentDateTime`, `GetStartingDate`, `GetElapsedYears`,
`GetTimeOfYear`, `GetTimeOfDay`, and the `TimeUISystem` day/tick getters
([Time2WorkPatches.cs#L58-L169](../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Patches/Time2WorkPatches.cs), commit `d42921f`).
Note the getter target syntax: a C# property getter is patched by its compiler-generated
method name (`get_normalizedDate`), not the property name. Two structural points fall out of
doing this at scale. First, this is a transpiler-free way to *replace* a method - the return
value is the only thing you own, so it only works when you can recompute the whole answer
externally. Second, the tradeoff against a full [system replacement](./system-replacement.md)
is that the vanilla `TimeSystem` object still exists and still ticks (Time2Work also postfixes
its `OnUpdate`, above) - you are shadowing its outputs, not displacing the system.

## Reaching private state: Traverse

Patches routinely need to read or write fields the game keeps `private`. `Traverse` is
Harmony's reflection convenience for exactly that. Anarchy reads the engine's `m_World`
field off the injected `GameManager` to recover the ECS `World` inside a patch context:

```csharp
// Getting World instance
world = Traverse.Create(GameManager.instance).Field<World>("m_World").Value;
```
[anarchy repo/Anarchy/ExtendedRoadUpgrades/UpgradesManager.cs#L94](../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/ExtendedRoadUpgrades/UpgradesManager.cs), commit `a6311e8`

`.Field<T>(name).Value` **reads**; `.Field(name).SetValue(...)` **writes** - Time2Work's
`TimeSystem` postfix (above, `Time2WorkPatches.cs#L39-L41`) uses `SetValue` to push its own
clock values back into the vanilla system's private `m_Time`/`m_Date`/`m_Year` fields.
Full write-up: [harmony-traverse-private-fields](../how-to/recipes/harmony-traverse-private-fields.md).

### The compiled alternative: FieldRefAccess

`Traverse` resolves the field by reflection on *every* access - fine for a once-per-frame
clock write, costly on a hot path. When a patch touches a private field repeatedly (inside
a per-entity loop, say), `AccessTools.FieldRefAccess` compiles a typed delegate **once**
that hands back a `ref` to the field, so subsequent reads and writes are direct. Realistic
Job Search caches two such refs into `PathfindSetupSystem`'s private `m_SetupList` and
`m_SetupDependencies` as `static readonly` delegates:

```csharp
static readonly AccessTools.FieldRef<PathfindSetupSystem, NativeList<PathfindSetupSystem.SetupListItem>> _setupListRef =
    AccessTools.FieldRefAccess<PathfindSetupSystem, NativeList<PathfindSetupSystem.SetupListItem>>("m_SetupList");

static readonly AccessTools.FieldRef<PathfindSetupSystem, Unity.Jobs.JobHandle> _setupDepsRef =
    AccessTools.FieldRefAccess<PathfindSetupSystem, Unity.Jobs.JobHandle>("m_SetupDependencies");
```
[realistic-jobsearch repo/RealisticJobSearch/Patches/Patch_CompleteSetup_FilterJobSeekerTargets.cs#L26-L31](../../vice-and-order-research/mods/dossiers/realistic-jobsearch/repo/RealisticJobSearch/Patches/Patch_CompleteSetup_FilterJobSeekerTargets.cs), commit `7a096b2`

Invoking the delegate against an instance yields a real `ref`, so the prefix can complete a
job handle through it and read the native list in place without a second reflection hop:

```csharp
ref var setupDeps = ref _setupDepsRef(__instance);
setupDeps.Complete();
ref var setupList = ref _setupListRef(__instance);
```
[realistic-jobsearch repo/RealisticJobSearch/Patches/Patch_CompleteSetup_FilterJobSeekerTargets.cs#L52-L55](../../vice-and-order-research/mods/dossiers/realistic-jobsearch/repo/RealisticJobSearch/Patches/Patch_CompleteSetup_FilterJobSeekerTargets.cs), commit `7a096b2`

Rule of thumb: `Traverse` for occasional access and quick prototyping; `FieldRefAccess`
(built once, stored `static readonly`) when the same private field is hit on a hot path.
Both fail the same way - a missing field when the game renames the member - so the same
lookup-guarding discipline applies.

## Hazards

Method patching is precise but fragile. Budget for these.

- **Burst-compiled and inlined methods do not patch reliably.** A method the game compiles
  with Burst (native code) or the JIT inlines has no managed IL body for Harmony to detour;
  a patch may silently never fire. This is why Burst and Harmony are entangled: both Anarchy
  and Write-Everywhere gate their `[BurstCompile]` attributes behind a `#if BURST` symbol
  (`#define BURST` at the top of a file, then `#if BURST` around the attribute) so a build
  can *disable* Burst on jobs when patchability or debuggability matters - e.g.
  [anarchy .../SetRetainingWallSegmentElevationSystem.cs#L5, #L84](../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/Systems/NetworkAnarchy/SetRetainingWallSegmentElevationSystem.cs), commit `a6311e8`,
  and [write-everywhere .../WENodeExtraDataUpdater.cs#L94-L95](../../vice-and-order-research/mods/dossiers/write-everywhere/repo/BelzontWE/Systems/WENodeExtraDataUpdater.cs), commit `13c70eb`.
  If a patch on a hot simulation method mysteriously does nothing, suspect Burst/inlining first.
- **Two mods patching the same method collide.** Harmony will stack multiple prefixes and
  postfixes on one target, but a prefix that returns `false` starves every patch that
  ordered after it, and two postfixes both rewriting `__result` fight over the final value.
  Patching a widely-used getter like `EconomyUtils.GetMarketPrice` is inherently a shared
  surface - assume another economy mod may be on it. `Needs Verification (in-game)`: the
  exact resolved order and final value when two specific mods patch the same method depends
  on load order and is only observable at runtime.
- **Unpatch symmetry on dispose.** A patch installed in `OnCreate`/load must be removed on
  teardown, or a reload leaves a stale detour pointing at a dead delegate. The redirector
  wrapper pairs its `PatchAll` with an explicit `UnpatchAll` that unpatches every recorded
  method by the mod's own Harmony id and runs registered cleanup actions:
  [write-everywhere repo/BelzontWE/Commons/Utils/Redirector.cs#L90-L105](../../vice-and-order-research/mods/dossiers/write-everywhere/repo/BelzontWE/Commons/Utils/Redirector.cs), commit `3698b64`.
  Unpatch with *your* id so you remove only your patches, never another mod's.
- **A second Harmony instance leaks if you unpatch only one id.** "Unpatch with your id" cuts
  both ways: a mod that constructs *more than one* `Harmony` instance must dispose each one,
  or the others stay installed. Extra Detailing Tools builds its main instance in `OnLoad` and
  on teardown calls `harmony.UnpatchAll("ExtraDetailingTools.EDT")` - only its own id
  ([extra-detailing-tools repo/EDT.cs#L126-L130](../../vice-and-order-research/mods/dossiers/extra-detailing-tools/repo/EDT.cs), commit `41df3c2`).
  But its ExtraSnap helpers each construct a *separate* `Harmony` keyed on `GetType().FullName`
  ([extra-detailing-tools repo/MOD/ExtraSnap/ExtraSnap.cs#L95](../../vice-and-order-research/mods/dossiers/extra-detailing-tools/repo/MOD/ExtraSnap/ExtraSnap.cs), commit `41df3c2`)
  and live in a static registry. Their `Dispose()` (which *would* run
  `_harmony.UnpatchAll(_harmony.Id)`,
  [ExtraSnap.cs#L121-L123](../../vice-and-order-research/mods/dossiers/extra-detailing-tools/repo/MOD/ExtraSnap/ExtraSnap.cs), commit `41df3c2`)
  is never invoked from `EDT.OnDispose`, so those snap postfixes are never removed. The lesson
  generalizes: track *every* Harmony id your mod creates and unpatch all of them on teardown,
  not just the one you named first. `Needs Verification (in-game)`: whether the un-disposed
  snap patches actually cause a visible fault on reload depends on runtime lifecycle.
- **String targets fail silently.** `AccessTools.Method` returns `null` when a renamed or
  re-signatured method can't be found; without the null guard MBE writes
  (`if (target == null ...) return;`), your patch just never installs and nothing logs.
  A CS2 patch that renames the target turns a working mod into a no-op with no exception.

## See also

- Concept: [System Replacement](./system-replacement.md) - when to patch a method vs
  disable the whole vanilla system (Harmony vs ECS-replacement tradeoff).
- How-to: [Harmony price-getter postfix](../how-to/recipes/harmony-price-getter-postfix.md) -
  the `__result`-rewrite recipe (MBE economy getters).
- How-to: [Harmony redirector reverse-patch](../how-to/recipes/harmony-redirector-reverse-patch.md) -
  the reverse-patch / detour-list machinery.
- How-to: [Harmony private-job postfix](../how-to/recipes/harmony-private-job-postfix.md) -
  the `return false` + `__result` job-replacement prefix.
- How-to: [Harmony Traverse private fields](../how-to/recipes/harmony-traverse-private-fields.md) -
  reading and writing private state.
- Reference: [Economy game systems](../reference/game-systems/economy.md) - the systems the
  MBE price patches target.
