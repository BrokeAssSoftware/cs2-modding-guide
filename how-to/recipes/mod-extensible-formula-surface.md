---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: Mod-extensible reflective formula/expression surface"
recipe: mod-extensible-formula-surface
technique_family: "BD - Mod-extensible reflective formula/expression surface"
diataxis: how-to
source_version: "~2.0 / 1.6.0f1-era (write-everywhere@13c70eb; date-pinned, static source only)"
last_reverified: "2026-07-05"
status: source-verified
canonical_mods:
  - write-everywhere@13c70eb04e6bed152257c516a982455148f591a5
technique_applicability: [platform, ui]
Created: 2026-07-05
Updated: 2026-07-05
Owners:
  - codex
---

# Mod-extensible reflective formula/expression surface

> Publish a host-owned extension point - a pair of attributes plus a one-time
> `AppDomain` assembly scan - so any *other* loaded mod can drop new functions into
> your expression surface without your mod ever referencing theirs.

## Problem
You are the host of a scripting / formula / expression feature (Write Everywhere lets
users type formulas that resolve live game data onto placed text). You want the set of
callable functions to be **open**: your own builtins, plus whatever a third-party
add-on mod ships, all appearing in the same picker. You cannot take a hard reference on
mods that do not exist yet, and you do not want contributors editing your assembly.
You need a way to discover *unknown* contributors at runtime and admit their functions.

## Solution
Own the **contract**, not the contributors. Define two public attributes in your
assembly - one that tags a class as a function category, one that tags a method as a
callable formula. At runtime, enumerate `AppDomain.CurrentDomain.GetAssemblies()`,
pull every public static method, keep the ones whose signature fits your calling
convention, and classify each by attribute into a category/source. Any mod that
references your assembly and applies your attributes (or just follows a namespace
convention) is picked up automatically. Do the scan **once** and cache it - the
assembly set does not change mid-session - and expose a cache-reset for reloads.

This is genuinely different from a reflection bridge (family G): a bridge resolves a
**known** peer mod by name and calls into it. Here the host advertises a **surface**
and discovers **unknown** publishers by attribute. The host never names them.

## Steps & Code

### 1. Define the extension attributes (the public contract)

Two attributes are the entire published API surface. `[WEBuiltinFunction]` marks a
class as a category bucket; `[WEFormula]` marks a method as a callable formula and
carries its return type + tooltip. Both are `public` and `sealed` - a contributor
in another assembly references your DLL and applies them.

```csharp
[AttributeUsage(AttributeTargets.Class, Inherited = false)]
public sealed class WEBuiltinFunctionAttribute : Attribute
{
    public string Category { get; }
    public WEBuiltinFunctionAttribute(string category) { Category = category; }
}

[AttributeUsage(AttributeTargets.Method, Inherited = false)]
public sealed class WEFormulaAttribute : Attribute
{
    public Type ReturnType { get; }
    public string Description { get; }
    public WEFormulaAttribute(Type returnType, string description = null)
    { ReturnType = returnType; Description = description; }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/write-everywhere/repo/BelzontWE/BuiltinFn/WEBuiltinAttributes.cs#L5-L25` (@13c70eb04e6bed152257c516a982455148f591a5)

### 2. A contributor publishes a function into the surface

This is what a publisher writes - the host ships these builtins the same way a
third-party add-on would. The class is `[WEBuiltinFunction("Calendar")]`; each callable
carries `[WEFormula(...)]` and follows the host's `(Entity, Dictionary<string,string>)`
calling convention:

```csharp
[WEBuiltinFunction("Calendar")]
public static class WECalendarFn
{
    [WEFormula(typeof(string))]
    public static string GetTimeStringWeLocale(Entity reference, Dictionary<string, string> vars)
    {
        var time = GetNormalizedTime_binding() * 24;
        // ... formats normalized time into a locale-aware clock string ...
    }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/write-everywhere/repo/BelzontWE/BuiltinFn/WECalendarFn.cs#L10-L45` (@13c70eb04e6bed152257c516a982455148f591a5)

### 3. Discover contributors with a one-time AppDomain scan (then cache)

The heart of the technique. Enumerate every loaded assembly, flatten to public static
methods, keep those matching the calling convention, and **cache the list** in a static
field. `??=` means the expensive scan runs exactly once per session; every later query
just filters the cached list by compatibility:

```csharp
public static IEnumerable<MethodInfo> FilterAvailableMethodsForFormulae(
    Type currentComponentType, string className = null, string method = null)
{
    CACHED_AVAILABLE_STATIC_METHODS ??= AppDomain.CurrentDomain.GetAssemblies()
                                         .SelectMany(assembly => SafeGetTypes(assembly))
                                         .SelectMany(x => SafeGetMethods(x, BindingFlags.Static | BindingFlags.Public))
                                         .Where(m => IsValidFormulaMethod(m))
                                         .ToList();

    return CACHED_AVAILABLE_STATIC_METHODS.Where(m => CheckMethodIsCompatible(currentComponentType, className, method, m));
}
```
Source: `../../../vice-and-order-research/mods/dossiers/write-everywhere/repo/BelzontWE/Utils/WEFormulaeHelper.cs#L418-L427` (@13c70eb04e6bed152257c516a982455148f591a5)

The scan touches *foreign* assemblies, so both the type-load and method-load steps are
wrapped to swallow a bad DLL rather than abort the whole surface - `SafeGetTypes`
returns the loadable subset of a `ReflectionTypeLoadException`
(`.../BelzontWE/Utils/WEFormulaeHelper.cs#L377-L397`).

### 4. Enforce the calling convention as the admission filter

An admitted method must fit the host's signature or it cannot be invoked. The filter is
the real contract enforcement - attributes are metadata, but *this* is what keeps an
incompatible method out of the surface: exactly one or two parameters (the optional
second being the `vars` dictionary), first parameter not by-ref-like, non-generic,
returns non-void:

```csharp
private static bool IsValidFormulaMethod(MethodInfo m)
{
    var p = m.GetParameters();
    return p != null
        && (p.Length == 1 || (p.Length == 2 && p[1].ParameterType == typeof(Dictionary<string, string>)))
        && !IsByRefLikeSafe(p[0].ParameterType)
        && !m.IsGenericMethod
        && m.ReturnType != typeof(void);
}
```
Source: `../../../vice-and-order-research/mods/dossiers/write-everywhere/repo/BelzontWE/Utils/WEFormulaeHelper.cs#L401-L416` (@13c70eb04e6bed152257c516a982455148f591a5)

### 5. Classify each admitted method into a category + source

Discovery yields raw `MethodInfo`; `WEStaticMethodDesc.From` turns each into a
descriptor and decides *where it came from*. Attribute-tagged methods become
`Formulae` with the declaring class's `Category`; a namespace convention
(`.Formulas` / `.Formulaes`) is a second on-ramp; everything else is bucketed by
assembly origin (Game / Unity / Mod / ...) via `GetSource`:

```csharp
public static WEStaticMethodDesc From(MethodInfo mi)
{
    if (mi.TryGetAttribute<WEFormulaAttribute>(out var attr)
        && mi.DeclaringType.TryGetAttribute<WEBuiltinFunctionAttribute>(out var attrBtn))
    {
        source = WEMemberSource.Formulae;
        weTooltip = attr.Description;
        weCategory = attrBtn.Category;
        dllName = mi.DeclaringType.Assembly.GetName().Name;
    }
    else if (mi.DeclaringType.Namespace.EndsWith(".Formulas") || mi.DeclaringType.Namespace.EndsWith(".Formulaes"))
    { source = WEMemberSource.Formulae; dllName = mi.DeclaringType.Assembly.GetName().Name; weCategory = mi.DeclaringType.Name; }
    else
    { source = WEMemberSourceExtensions.GetSource(mi.DeclaringType.Assembly, out modUrl, out modName, out dllName); }
    // ... builds the descriptor (className, methodName, returnType, supportsMathOp) ...
}
```
Source: `../../../vice-and-order-research/mods/dossiers/write-everywhere/repo/BelzontWE/IO/WEStaticMethodDesc.cs#L22-L46` (@13c70eb04e6bed152257c516a982455148f591a5)

`GetSource` is where a foreign method's origin mod is resolved - it reads the display
name and Workshop URL out of the asset database so the picker can attribute the
function to its publisher (`.../BelzontWE/Enum/WEMemberSource.cs#L19-L53`).

### 6. Register the surface for the UI (group by source -> mod -> category)

The controller turns the classified descriptors into a nested lookup the picker renders:
source bucket -> owning DLL -> category -> methods. Contributor functions naturally sort
into their own `Formulae`/mod group beside the host's builtins:

```csharp
return type == null ? null : FilterAvailableMethodsForFormulae(type)
    .Select(x => WEStaticMethodDesc.From(x))
    .OrderBy(x => x.source)
    .GroupBy(x => x.source)
    .ToDictionary(
        srcGrouping => (int)srcGrouping.Key, srcGrouping => srcGrouping
        .OrderBy(x => x.dllName)
        .GroupBy(x => x.dllName)
        .ToDictionary( /* ...then group by weCategory / className... */ ));
```
Source: `../../../vice-and-order-research/mods/dossiers/write-everywhere/repo/BelzontWE/Utils/WEFormulaeControllerHelper.cs#L26-L44` (@13c70eb04e6bed152257c516a982455148f591a5)

### 7. Expose a cache reset for reloads

The scan is memoized in a static field, so a hot-reload or mod set change would serve a
stale surface. Ship an explicit reset that nulls the cache; the next query re-scans:

```csharp
private static List<MethodInfo> CACHED_AVAILABLE_STATIC_METHODS;

internal static void ResetMethodCache()
{
    CACHED_AVAILABLE_STATIC_METHODS = null;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/write-everywhere/repo/BelzontWE/Utils/WEFormulaeHelper.cs#L370-L375` (@13c70eb04e6bed152257c516a982455148f591a5)

## Pitfalls & gotchas

- **The scan touches assemblies you do not own - one bad DLL must not sink the
  surface.** `AppDomain.CurrentDomain.GetAssemblies()` includes half-loaded or
  version-mismatched foreign assemblies. Calling `GetTypes()` on one can throw
  `ReflectionTypeLoadException`. Write Everywhere wraps both type and method
  enumeration so a failing assembly contributes its loadable subset (or nothing)
  instead of aborting discovery
  (`.../BelzontWE/Utils/WEFormulaeHelper.cs#L377-L397`). Skip this and one broken
  add-on disables *everyone's* functions.

- **Cache staleness after reload.** The `??=` memoization is a per-session assumption.
  If a mod is enabled/disabled or hot-reloaded without a full restart, the cached list
  is wrong until `ResetMethodCache()` runs
  (`.../BelzontWE/Utils/WEFormulaeHelper.cs#L372-L375`). Wire the reset into whatever
  reload path your host has. *Whether the game reloads assemblies without a restart at
  all is* `Needs Verification (in-game)`.

- **Attributes are advisory; the signature filter is the real gate.** A contributor can
  tag a method `[WEFormula]` all day - if its signature does not match
  `IsValidFormulaMethod` (wrong arity, by-ref-like first arg, generic, void return) it
  is silently dropped with no error
  (`.../BelzontWE/Utils/WEFormulaeHelper.cs#L401-L416`). Document the calling convention
  loudly for publishers; the surface will not tell them why their function vanished.

- **Two on-ramps, so untagged methods can leak in.** Because `From` also admits any
  public static method in a `*.Formulas` / `*.Formulaes` namespace
  (`.../BelzontWE/IO/WEStaticMethodDesc.cs#L37-L42`), a contributor need not use the
  attribute at all. Convenient, but it means the attribute is not a hard boundary - a
  method that merely lands in that namespace and fits the signature is published too.

- **Origin resolution depends on the asset database.** `GetSource` attributes a foreign
  function to its mod by querying the asset DB for the assembly's display name / URL,
  falling back to `"??????"` when unresolved
  (`.../BelzontWE/Enum/WEMemberSource.cs#L44-L53`). Discovery still succeeds; only the
  *labelling* degrades. The precise runtime population of that DB is
  `Needs Verification (in-game)`.

- **Trust boundary.** Every admitted method is foreign public static code your host will
  invoke. This surface has no sandbox - admission is purely signature + attribute. Treat
  contributor code as running with your mod's privileges. Any stronger isolation is
  `Needs Verification (in-game)`.

## Variations

- **Attribute-only, no namespace fallback.** Drop the `.Formulas` namespace branch in
  step 5 and require `[WEFormula]` + `[WEBuiltinFunction]` for admission. Stricter, no
  accidental leakage, at the cost of contributor convenience.

- **Category from convention instead of an attribute argument.** The namespace on-ramp
  already derives the category from the declaring type name
  (`weCategory = mi.DeclaringType.Name`,
  `.../BelzontWE/IO/WEStaticMethodDesc.cs#L37-L42`) - a zero-attribute publishing path
  when you would rather infer buckets than have publishers name them.

- **Eager registry instead of a lazy scan.** Instead of memoizing on first query, run
  the `AppDomain` scan once at host init and hand out an immutable snapshot. Same
  discovery, different lifetime - simpler if your host has a clean "all mods loaded"
  moment and never reloads.

- **Compatibility narrowing per call.** Discovery is global, but each query further
  filters by the target's component type / class / method name via
  `CheckMethodIsCompatible` (`.../BelzontWE/Utils/WEFormulaeHelper.cs#L429-L434`) so the
  same cached surface serves many contexts. Reuse this shape when one surface must be
  filtered differently per call site.

## See also
- Related recipes: [reflection mod bridges](reflection-mod-bridges.md) (family G -
  resolving a *known* peer, the deliberate contrast to this "discover unknowns"
  technique); [cross-mod service protocol](cross-mod-service-protocol.md).
- Explanation: [dependency strategy](../../explanation/dependency-strategy.md) (host owns
  the contract, not the contributors).
- Case study demonstrating it:
  [write-everywhere ecosystem](../../case-studies/write-everywhere-ecosystem.md).

## Sources
- Canonical mod (dossier + repo):
  - `write-everywhere` @13c70eb04e6bed152257c516a982455148f591a5 -
    `repo/BelzontWE/BuiltinFn/WEBuiltinAttributes.cs`,
    `repo/BelzontWE/BuiltinFn/WECalendarFn.cs`,
    `repo/BelzontWE/Utils/WEFormulaeHelper.cs`,
    `repo/BelzontWE/IO/WEStaticMethodDesc.cs`,
    `repo/BelzontWE/Enum/WEMemberSource.cs`,
    `repo/BelzontWE/Utils/WEFormulaeControllerHelper.cs`
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding
