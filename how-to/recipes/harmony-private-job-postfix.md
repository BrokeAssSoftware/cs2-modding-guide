---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: Harmony patch on a private job-setup method (ref __result)"
recipe: harmony-private-job-postfix
technique_family: "U - Harmony patch on a private job-setup method (ref __result)"
diataxis: how-to
source_version: "1.6.0f1 (time2work-realistic-trips@d42921f; date-pinned, static source only)"
last_reverified: "2026-07-04"
status: source-verified
canonical_mods:
  - time2work-realistic-trips@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39
technique_applicability: [simulation]
Created: 2026-07-04
Updated: 2026-07-04
Owners:
  - codex
---

# Harmony patch on a private job-setup method (ref __result)

> Intercept a *private* engine method that schedules and returns a `JobHandle`,
> and for one selected branch schedule your own Burst job instead - writing the
> replacement handle back through `ref JobHandle __result` and telling Harmony to
> skip the original.

## Problem
The behaviour you want to change lives inside a **private** vanilla method that
builds an `EntityQuery`, schedules a job, and returns that job's `JobHandle` - here
`PathfindSetupSystem.FindTargets(SetupTargetType, in SetupData)`, which sets up
pathfinding targets for citizens. You only care about **one** of its branches (the
Leisure branch) and want to bias it toward special-event venues; every other branch
must run exactly as vanilla. You cannot subclass it (private), you must not break the
`TempJob` queue lifetime the original manages (the caller expects the returned handle
to gate that queue's disposal), and an overload-by-name lookup will not find a private
method with a `by-ref` parameter unless you resolve it explicitly.

## Solution
Write a Harmony patch class with no `[HarmonyPatch(...)]` target attribute; instead
supply a static `TargetMethod()` that resolves the private method by reflection
(`AccessTools.Method` with an explicit parameter-type array, using `.MakeByRefType()`
for the `in`/`ref` parameter). Then - despite the "job-setup" framing this is a
**`Prefix`, not a postfix** - branch on the argument: `return true` to let vanilla run
for every branch you do not care about, and for the one branch you own, schedule your
own job, assign its handle to `ref JobHandle __result`, and `return false` so the
original body never executes. Returning your handle (not `default`) is what keeps the
`TempJob` queue lifetime correct, because the caller receives a real dependency to
wait on.

## Steps & Code

### 1. Make sure something calls `PatchAll`

The patch class is discovered by Harmony's assembly scan; nothing works without a
`PatchAll` at mod init. time2work does this in `Mod.OnCreate`:

```csharp
var harmony = new Harmony(harmonyID);
//Harmony.DEBUG = true;
harmony.PatchAll(typeof(Mod).Assembly);
```
Source: `../../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Mod.cs#L164-L166` (@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39)

### 2. Declare a target-less `[HarmonyPatch]` class

No method is named in the attribute - the class supplies its own target in step 3.
A `static` class is fine; Harmony only needs the patch methods.

```csharp
[HarmonyPatch]
public static class PathfindSetupSystem_LeisureEventBiasPatch
```
Source: `../../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Patches/PathfindSetupSystem_LeisureEventBiasPatch.cs#L32-L33` (@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39)

### 3. Resolve the private method with `TargetMethod()`

A `static MethodBase TargetMethod()` is Harmony's escape hatch for targets you cannot
name in an attribute. Pass the exact parameter types; the `in SetupData` parameter is
a by-ref parameter, so it MUST be resolved with `.MakeByRefType()` or the lookup
returns `null`:

```csharp
static MethodBase TargetMethod()
{
    // private JobHandle FindTargets(SetupTargetType targetType, in PathfindSetupSystem.SetupData setupData)
    return AccessTools.Method(
        typeof(PathfindSetupSystem),
        "FindTargets",
        new[]
        {
            typeof(SetupTargetType),
            typeof(PathfindSetupSystem.SetupData).MakeByRefType()
        });
}
```
Source: `../../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Patches/PathfindSetupSystem_LeisureEventBiasPatch.cs#L43-L54` (@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39)

### 4. Write a `Prefix` that takes `ref JobHandle __result`

The magic parameter `__result` binds to the original's return value (a `JobHandle`);
because you intend to overwrite it, it is passed `ref`. The `in SetupData` parameter is
received as `ref` ("treat 'in' as ref"), and `__instance` gives you the system so you
can read its handles/lookups:

```csharp
static bool Prefix(
    PathfindSetupSystem __instance,
    SetupTargetType targetType,
    ref PathfindSetupSystem.SetupData setupData, // treat "in" as ref
    ref JobHandle __result)
```
Source: `../../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Patches/PathfindSetupSystem_LeisureEventBiasPatch.cs#L56-L60` (@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39)

### 5. Let vanilla handle every branch you do not own

The first thing the prefix does is bail out for anything that is not the branch you
mean to replace. `return true` runs the original method untouched:

```csharp
if (targetType != SetupTargetType.Leisure)
    return true; // run vanilla for everything else
```
Source: `../../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Patches/PathfindSetupSystem_LeisureEventBiasPatch.cs#L62-L63` (@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39)

### 6. Reuse the system's real job dependency

To slot your job in where vanilla's would go, feed it the same upstream dependency the
system uses - `SystemBase.Dependency`. That property is `protected`, so it is read
through a cached reflection getter:

```csharp
private static readonly MethodInfo MI_DependencyGetter =
    AccessTools.PropertyGetter(typeof(SystemBase), "Dependency");
// ...inside Prefix:
var dependsOnObj = MI_DependencyGetter.Invoke(__instance, null);
JobHandle dependsOn = dependsOnObj is JobHandle h ? h : default;
```
Source: `../../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Patches/PathfindSetupSystem_LeisureEventBiasPatch.cs#L39-L41,#L66-L67` (@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39)

### 7. Schedule your job, write `__result`, and skip the original

Schedule the replacement job against `dependsOn`, register any readers so the engine
waits on your handle, assign that handle to `__result`, and `return false` so the
private body never runs for this branch:

```csharp
}.ScheduleParallel(leisureProviderQuery, dependsOn);

resourceSystem.AddPrefabsReader(handle);

__result = handle;
return false; // skip vanilla leisure branch
```
Source: `../../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Patches/PathfindSetupSystem_LeisureEventBiasPatch.cs#L127,#L129-L132` (@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39)

## Pitfalls & gotchas

- **It is a PREFIX, not a postfix - despite the family name.** The whole "replace one
  branch" idiom depends on `return false` (skip original) plus a written `__result`;
  a postfix cannot suppress the original body. The two load-bearing lines are
  `__result = handle; return false;`
  (`.../PathfindSetupSystem_LeisureEventBiasPatch.cs#L131-L132`). If you accidentally
  write a `Postfix`, the vanilla leisure job runs *and* yours does, double-scheduling.

- **Return your real handle, never `default`.** The caller uses the returned
  `JobHandle` to gate the `TempJob` queue's lifetime. The file's own header comment
  states it keeps "the pathfind setup TempJob queue lifetime correct (because we return
  the JobHandle that FindTargets returns)". Writing `__result = default` would tell the
  engine your work is already finished and let the queue be disposed out from under a
  still-running job.

- **`.MakeByRefType()` is mandatory for `in`/`ref`/`out` params.** `AccessTools.Method`
  matches the parameter-type array exactly. `FindTargets` takes `in SetupData`; passing
  `typeof(SetupData)` (without `.MakeByRefType()`) resolves to `null` and Harmony
  silently patches nothing (`.../PathfindSetupSystem_LeisureEventBiasPatch.cs#L52`).

- **`SystemBase.Dependency` is `protected` - read it by reflection, cache the getter.**
  You cannot touch `__instance.Dependency` from a patch class; the mod caches an
  `AccessTools.PropertyGetter(typeof(SystemBase), "Dependency")` once as a static field
  and invokes it (`.../PathfindSetupSystem_LeisureEventBiasPatch.cs#L39-L41,#L66-L67`).
  Resolving the getter per-call would be needless reflection cost on a hot path.

- **Patching a private method is version-fragile.** The name `"FindTargets"` and its
  exact signature are resolved by string + reflected types at runtime. A game patch
  that renames the method, changes the parameter list, or splits the branch logic makes
  `TargetMethod()` return `null` (no patch, no error at the call site). Re-verify the
  target each CS2 release; `Harmony.DEBUG = true` (present but commented in `Mod.cs#L165`)
  helps confirm the patch actually attached.

- **Whether the biased job produces the intended in-game routing** (citizens actually
  favouring event venues) is a runtime-behavioural claim not visible in the patch
  wiring: `Needs Verification (in-game)`.

## Variations

- **Attribute target instead of `TargetMethod()`.** When the method is *public* and has
  no by-ref parameters, `[HarmonyPatch(typeof(T), "Method")]` on the class is simpler
  and needs no reflection. Reach for `TargetMethod()` only when the target is private,
  overloaded ambiguously, or has `in`/`ref`/`out` parameters (as here).

- **Full replace vs. selective replace.** This recipe replaces exactly one branch and
  delegates the rest with `return true`. To replace the method entirely, drop the
  branch guard and always `return false` after writing `__result`. To only *observe*
  the returned handle (e.g. chain your own job after vanilla's), use a `Postfix(ref
  JobHandle __result)` that combines handles with `JobHandle.CombineDependencies`
  instead - that is a genuine postfix and does not suppress the original.

- **Unpatch on dispose.** The same mod tears its patches down in `OnDispose`, which
  matters for a private-method patch that could otherwise linger across mod reloads:

  ```csharp
  var harmony = new Harmony(harmonyID);
  harmony.UnpatchAll(harmonyID);
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Mod.cs#L182-L183` (@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39)

## See also
- Explanation: [Harmony patching](../../explanation/harmony-patching.md) (prefix vs
  postfix, `__result`/`__instance`/`TargetMethod`, patch lifecycle).
- Related recipe: [Harmony postfix on a price getter](harmony-price-getter-postfix.md)
  (the read-only postfix counterpart - reads/rewrites a return value without suppressing
  the original).
- Case study demonstrating it: [time2work-realistic-trips](../../case-studies/time2work-realistic-trips.md).

## Sources
- Canonical mods (dossier + repo):
  - `time2work-realistic-trips` @d42921ffb1f6bbbf2c2b086bb17727bf0f637c39 -
    `repo/NightShift/Patches/PathfindSetupSystem_LeisureEventBiasPatch.cs`,
    `repo/NightShift/Mod.cs`
- Official/community references (link out, do not duplicate):
  https://harmony.pardeike.net/ (Harmony docs: `Prefix`, `__result`, `TargetMethod`)
