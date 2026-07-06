---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: Harmony redirector / reverse-patch bridge"
recipe: harmony-redirector-reverse-patch
technique_family: "D - Harmony redirector / reverse-patch bridge"
diataxis: how-to
source_version: "~1.5.x (write-everywhere@13c70eb; date-pinned, static source only)"
last_reverified: "2026-07-04"
status: source-verified
canonical_mods:
  - write-everywhere@13c70eb04e6bed152257c516a982455148f591a5
  - anarchy@a6311e898d20a775368668b234aaa32f06e3e1eb
technique_applicability: [platform, ui]
Created: 2026-07-04
Updated: 2026-07-04
Owners:
  - codex
---

# Harmony redirector / reverse-patch bridge

> Call a method you can't reference at compile time - a private game method or another
> mod's API - by stubbing a local method with a matching signature and having Harmony
> *reverse-patch* the original's IL into your stub, so calling your stub runs the
> original. A thin `Redirector` component wraps registration so patches apply and
> unwind as one unit.

## Problem
Two situations force you off the normal prefix/postfix path:

1. You need to **invoke** a method, not intercept it - e.g. a private/internal engine
   method, or a helper on a class you don't hold a hard reference to.
2. You need to **call another mod** without a compile-time assembly reference, so your
   mod still loads when that mod is absent (soft dependency / optional bridge).

A normal `Harmony.Patch` prefix/postfix lets you run code *around* a method but does not
hand you a callable copy of it, and a hard `using OtherMod;` makes your DLL fail to load
if the other mod isn't installed. Reverse patching solves both.

## Solution
Harmony's `ReversePatch` copies the **original method's IL into a method you own**
(a "stub"). After the patch, your stub *is* the original - call it directly and you run
the original's body, resolved at runtime with no static reference. Write Everywhere wraps
this in two reusable pieces from its vendored commons library: `Redirector.AddReversePatch`
(single method) and `BridgeUtils.ApplyPatches` (bulk-bind a whole local stub class onto
another assembly's class of the same name). The `Redirector` MonoBehaviour also records
every patch so the mod can `UnpatchAll` cleanly on disable.

> **Where the machinery lives (read this first).** The reverse-patch code is **not** in
> the main Write Everywhere tree. It sits in the vendored **CS2-BelzontCommons**
> submodule (`BelzontWE/Commons`), pinned at commit `3698b64`. Every `Redirector.cs` /
> `BridgeUtils.cs` citation below is at that *submodule* commit, not the WE mod pin. If
> you `git show` the WE mod pin for these paths you get nothing - you must descend into
> the submodule.

## Steps & Code

### 1. Understand the wrapper: reverse patch vs normal redirect

`Redirector` is a `MonoBehaviour` that exposes both patch shapes side by side. `AddRedirect`
is the ordinary prefix/postfix (intercept), shown here only for contrast:

```csharp
public void AddRedirect(MethodInfo oldMethod, MethodInfo newMethodPre, MethodInfo newMethodPost = null, MethodInfo transpiler = null)
{
    if (BasicIMod.TraceMode) LogUtils.DoTraceLog($"Adding patch! {oldMethod.DeclaringType} {oldMethod}");
    m_detourList.Add(Harmony.Patch(oldMethod, newMethodPre != null ? new HarmonyMethod(newMethodPre) : null, newMethodPost != null ? new HarmonyMethod(newMethodPost) : null, transpiler != null ? new HarmonyMethod(transpiler) : null));
    m_patches.Add(oldMethod);
}
```
Source: `../../../vice-and-order-research/mods/dossiers/write-everywhere/repo/BelzontWE/Commons/Utils/Redirector.cs#L75-L81` (@3698b64795eec8fd7effe75cf08cb4d1cbe3c674)

### 2. Register the reverse patch (stub <- original)

`AddReversePatch` takes the **target** (the original you want to call) and **your own**
method (the stub). After this runs, `ownMethod`'s body has been replaced with a copy of
`targetMethod`'s IL:

```csharp
public void AddReversePatch(MethodInfo targetMethod, MethodInfo ownMethod)
{
    if (BasicIMod.TraceMode) LogUtils.DoTraceLog($"Reverse patch! {targetMethod.DeclaringType} {targetMethod}");
    Harmony.ReversePatch(targetMethod, new HarmonyMethod(ownMethod));
}
```
Source: `../../../vice-and-order-research/mods/dossiers/write-everywhere/repo/BelzontWE/Commons/Utils/Redirector.cs#L83-L87` (@3698b64795eec8fd7effe75cf08cb4d1cbe3c674)

Your stub's declared body is a throwaway (it never executes after patching); its
**signature must match the target exactly** - that is how Harmony maps IL and how you get
a type-safe call site.

### 3. Drive it: discover redirectors and fire them per-world

`Redirector`s are found by interface and driven at defined moments. `IRedirectable`
implementors get `DoPatches(world)` called once per world creation; `IRedirectableWorldless`
patch immediately at `PatchAll` (both discovered by reflection in `PatchAll`):

```csharp
public interface IRedirectable
{
    void DoPatches(World world);
}
```
Source: `../../../vice-and-order-research/mods/dossiers/write-everywhere/repo/BelzontWE/Commons/Utils/Redirector.cs#L24-L27` (@3698b64795eec8fd7effe75cf08cb4d1cbe3c674)

```csharp
public static void OnWorldCreated(World world)
{
    foreach (var redirector in worldDependantRedirectors)
    {
        redirector.DoPatches(world);
    }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/write-everywhere/repo/BelzontWE/Commons/Utils/Redirector.cs#L156-L162` (@3698b64795eec8fd7effe75cf08cb4d1cbe3c674)

You call `AddReversePatch` (step 2) from inside your redirector's `DoPatches`, so a
world-scoped target is bound at the right time.

### 4. Bulk-bind a whole bridge class with `BridgeUtils.ApplyPatches`

For a cross-mod bridge you rarely reverse-patch one method at a time. `ApplyPatches`
takes the **other mod's assembly name** plus a list of `(localStubClass, remoteClassName)`
mappings, finds the remote class in the loaded assembly, and reverse-patches **every
public static method of your stub class onto the same-named/same-signature remote method**:

```csharp
foreach (var (type, sourceClassName) in bridgeMappings)
{
    var targetType = exportedTypes.First(x => x.Name == sourceClassName);
    if (targetType is null) { /* warn + return false */ }
    foreach (var method in type.GetMethods(BindingFlags.Public | BindingFlags.Static))
    {
        var srcMethod = targetType.GetMethod(method.Name, RedirectorUtils.allFlags, null,
            [.. method.GetParameters().Select(x => x.ParameterType)], null);
        if (srcMethod != null)
            Harmony.ReversePatch(srcMethod, new HarmonyMethod(method));
        else { /* warn + return false */ }
    }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/write-everywhere/repo/BelzontWE/Commons/Utils/BridgeUtils.cs#L152-L172` (@3698b64795eec8fd7effe75cf08cb4d1cbe3c674)

The `required` flag decides whether a missing target assembly is a hard error or a silent
skip, so an optional dependency degrades gracefully:

```csharp
public static bool ApplyPatches(string assemblyName, List<(Type, string)> bridgeMappings, bool required = false)
{
    var weAsset = AssetDatabase.global.GetAsset(SearchFilter<ExecutableAsset>.ByCondition(asset => asset.isLoaded && asset.name.Equals(assemblyName)));
    if (required && weAsset?.assembly is null)
    {
        LogUtils.DoErrorLog($"The module {typeof(BridgeUtils).Assembly.GetName().Name} requires {assemblyName}.dll mod to work!");
        return false;
    }
    // ... exportedTypes + bridgeMappings loop (step 4 above) ...
}
```
Source: `../../../vice-and-order-research/mods/dossiers/write-everywhere/repo/BelzontWE/Commons/Utils/BridgeUtils.cs#L140-L151` (@3698b64795eec8fd7effe75cf08cb4d1cbe3c674)

### 5. The other side: publish a bridge target other mods can reverse-patch

The consuming mod's stub class must mirror a real, stable class in the target mod. Anarchy
ships exactly such a surface - a `public static` class of plain, reference-free methods
whose signatures a consumer copies verbatim into its stub:

```csharp
/// <summary>A bridge class for other mods to tie into Anarchy.</summary>
public static class AnarchyBridge
{
    public static bool TryAddAnarchyComponent(Entity instanceEntity)
    {
        SelectedInfoPanelTogglesSystem uiSystem = World.DefaultGameObjectInjectionWorld.GetOrCreateSystemManaged<SelectedInfoPanelTogglesSystem>();
        if (uiSystem.CheckOverridable(instanceEntity))
        {
            uiSystem.EntityManager.AddComponent<PreventOverride>(instanceEntity);
            return true;
        }
        return false;
    }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/Bridge/AnarchyBridge.cs#L17-L52` (@a6311e898d20a775368668b234aaa32f06e3e1eb)

A consumer defines `public static class AnarchyBridge { public static bool TryAddAnarchyComponent(Entity e) => default; }` in its own namespace, then
`BridgeUtils.ApplyPatches("Anarchy", new List<(Type,string)>{ (typeof(MyAnarchyBridge), "AnarchyBridge") })`.
After that, calling the local stub runs Anarchy's real method - no assembly reference.

## Pitfalls & gotchas

- **Cite the submodule, not the mod.** The whole technique's code (`Redirector.cs`,
  `BridgeUtils.cs`) lives in the vendored `BelzontWE/Commons` submodule at commit
  `3698b64`, distinct from the Write Everywhere mod pin. Reading the WE mod pin for these
  paths yields nothing. Always `git show 3698b64:Utils/...`.

- **Reverse patch gives you a *copy*, not a hook.** After `ReversePatch`, your stub no
  longer runs its own body - it runs the target's IL. If you wanted to *observe or modify*
  the call, you want `AddRedirect` (prefix/postfix), not a reverse patch. Reverse patch is
  strictly for *invoking* an otherwise-unreachable method.

- **Signature drift = silent bind failure (or worse).** `ApplyPatches` matches by method
  name **and** exact parameter types
  (`targetType.GetMethod(method.Name, ..., paramTypes, ...)`). If the target mod changes a
  parameter, `srcMethod` comes back null and the whole `ApplyPatches` call bails with
  `return false` - your bridge is half-applied at best.
  Source: `../../../vice-and-order-research/mods/dossiers/write-everywhere/repo/BelzontWE/Commons/Utils/BridgeUtils.cs#L163-L171` (@3698b64795eec8fd7effe75cf08cb4d1cbe3c674)

- **The "method not found" log path throws.** In the else branch the code logs
  `srcMethod.Name` - but `srcMethod` is `null` there, so the diagnostic itself NREs before
  the intended warning is seen. Don't rely on that message to tell you which method
  drifted; verify signatures against the target's source yourself.
  Source: `../../../vice-and-order-research/mods/dossiers/write-everywhere/repo/BelzontWE/Commons/Utils/BridgeUtils.cs#L168-L172` (@3698b64795eec8fd7effe75cf08cb4d1cbe3c674)

- **Only public static methods are bridged.** `ApplyPatches` iterates
  `type.GetMethods(BindingFlags.Public | BindingFlags.Static)` on your stub class. Instance
  methods, non-public methods, and properties are ignored - a bridge surface must be a flat
  `public static` API on both sides.

- **Unwind is manual and global.** `Redirector` tracks patches in static lists and
  `UnpatchAll` removes them by Harmony id, but reverse patches copy IL into your assembly;
  there is nothing to "unpatch" from your stub. Design bridges so a stale copy after the
  target mod hot-unloads is tolerable. Exact hot-reload behaviour is
  **Needs Verification (in-game)**.

- **Timing.** World-dependent targets must be reverse-patched from `DoPatches(world)`
  (driven by `OnWorldCreated`), not at static init, or the target type/world isn't ready.

## Variations

- **Single method, no bridge class.** Skip `ApplyPatches` and call
  `Redirector.AddReversePatch(targetMI, stubMI)` directly when you only need one private
  engine method (resolve `targetMI` via reflection with `RedirectorUtils.allFlags`).

- **Prefix/postfix instead (intercept, don't call).** If your goal is to change what a
  method does rather than call it, use `AddRedirect` - see the Overrides in Write
  Everywhere (`GameUIResourceHandlerOverrides`, `PrefabSystemOverrides`) which register
  vanilla methods with pre/post handlers via the same `Redirector` base.

- **Optional vs required dependency.** Pass `required: true` to `ApplyPatches` to hard-fail
  (and log) when the target mod is absent; leave it `false` for a soft bridge that no-ops
  when the other mod isn't installed.

## See also
- Explanation: [Harmony patching](../../explanation/harmony-patching.md) (prefix / postfix
  / transpiler / reverse-patch concepts and when each applies).
- Case study: [Write Everywhere ecosystem](../../case-studies/write-everywhere-ecosystem.md)
  (how the commons library and bridges fit the wider mod).
- Reference: [Shared libraries](../../reference/shared-libraries/README.md)
  (BelzontCommons and other vendored helper libs).
- Related recipe: [Reflection mod bridges](reflection-mod-bridges.md) (the reflection-only
  alternative when you can't or don't want to reverse-patch).

## Sources
- Canonical mods (dossier + repo):
  - `write-everywhere` @13c70eb04e6bed152257c516a982455148f591a5 - machinery in vendored
    submodule `BelzontWE/Commons` @3698b64795eec8fd7effe75cf08cb4d1cbe3c674
    (`Utils/Redirector.cs`, `Utils/BridgeUtils.cs`)
  - `anarchy` @a6311e898d20a775368668b234aaa32f06e3e1eb -
    `Anarchy/Bridge/AnarchyBridge.cs` (bridge-target example)
- Official/community references (link out, do not duplicate):
  https://harmony.pardeike.net/articles/patching-reverse.html
