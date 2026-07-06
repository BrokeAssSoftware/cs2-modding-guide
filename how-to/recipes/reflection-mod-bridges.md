---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: Reflection bridges between mods"
recipe: reflection-mod-bridges
technique_family: "G - Reflection bridges between mods"
diataxis: how-to
source_version: "1.6.0f1 (time2work-realistic-trips@d42921f; date-pinned, static source only)"
last_reverified: "2026-07-04"
status: source-verified
canonical_mods:
  - time2work-realistic-trips@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39
  - realistic-path-finding@50645fa6a078181365e36a42e2e27b96699bf02a
  - anarchy@a6311e898d20a775368668b234aaa32f06e3e1eb
  - elections-rt-module@36c25afcda67c80ce75ba68d423fe8238499435a
  - custom-chirps@f018ac382e93e0b56cd7b974d1ce0b155d23d2b4
  - better-bulldozer@4408466f226db811159d92859479ae1e1c28ba06
technique_applicability: [platform, simulation]
Created: 2026-07-04
Updated: 2026-07-05
Owners:
  - codex
---

# Reflection bridges between mods

> Detect whether another mod is loaded and call its API at runtime through
> reflection - no `<Reference>` in your `.csproj`, no hard assembly dependency, and a
> safe no-op when the other mod is absent.

## Problem
You want your mod to cooperate with an *optional* companion mod - post to CustomChirps'
chirper feed, read Time2Work's day-length factor, hook MoveIt's copy pipeline - but you
cannot take a compile-time reference on it. A hard `<Reference>` means your assembly
fails to load the moment the other mod is missing, on a different version, or renamed;
it also creates a load-order coupling the platform does not guarantee. You need
**detect-and-degrade**: use the other mod's feature when present, and silently do
nothing when it is not.

## Solution
Resolve the foreign type by name at runtime, cache the resolved `MethodInfo`/`FieldInfo`,
and invoke it via reflection - guarding every path so a missing type is a clean no-op.
There are two resolution styles in the wild: **(1)** `Type.GetType("Ns.Type, Asm")` with
a `?? FindType(...)` fallback that scans loaded assemblies (time2work, elections), and
**(2)** a raw `AppDomain.CurrentDomain.GetAssemblies()` loop that asks each assembly for
the type (realistic-path-finding, anarchy). Both mirror the foreign enum/signature
*locally by name* so no shared type is ever referenced. Resolve **once** (a `_resolved`
latch), cache the handles, and expose an `IsAvailable` gate that call sites check before
they do any work.

## Steps & Code

### 1. Mirror the foreign contract locally and declare cached handles

Never reference the other mod's types. Re-declare the enum you need (names must match the
real one, so `Enum.Parse` round-trips), and hold the resolved reflection handles in
`static` fields behind a `_resolved` latch:

```csharp
public static class CustomChirpsBridge
{
    private static bool _resolved;
    private static Type _apiType;           // CustomChirps.Systems.CustomChirpApiSystem
    private static Type _deptEnumType;      // CustomChirps.Systems.DepartmentAccount
    private static MethodInfo _postChirp;   // PostChirp(string, DepartmentAccount, Entity, string)

    public static bool IsAvailable
    {
        get { EnsureResolve(); return _apiType != null && _deptEnumType != null && _postChirp != null; }
    }
```
Source: `../../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Bridge/CustomChirpsBridge.cs#L40-L51` (@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39)

The mirrored `enum DepartmentAccountBridge { Electricity, FireRescue, Roads, ... }` lives
in the same file with the members in the same order/name as the real enum
(`CustomChirpsBridge.cs#L14-L34`).

### 2. Resolve the type once - FQN first, then scan (style 1)

`Type.GetType("Namespace.Type, AssemblyName")` succeeds only if that assembly is already
loaded *and* named exactly; the `?? FindType(...)` fallback covers the common case where
the assembly name differs from what you guessed. Do this behind the `_resolved` latch so
it runs at most once:

```csharp
private static void EnsureResolve()
{
    if (_resolved) return;
    _resolved = true;

    // Find API & enum types (by FQN first, then scan loaded assemblies)
    _apiType = Type.GetType("CustomChirps.Systems.CustomChirpApiSystem, CustomChirps") ?? FindType("CustomChirps.Systems.CustomChirpApiSystem");
    _deptEnumType = Type.GetType("CustomChirps.Systems.DepartmentAccount, CustomChirps") ?? FindType("CustomChirps.Systems.DepartmentAccount");

    if (_apiType != null)
        _postChirp = _apiType.GetMethod("PostChirp", BindingFlags.Public | BindingFlags.Static);
}
```
Source: `../../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Bridge/CustomChirpsBridge.cs#L106-L119` (@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39)

### 3. Implement `FindType` as a defensive assembly scan

The fallback walks every loaded assembly and asks it for the type by full name.
`throwOnError: false` plus a swallowing `try/catch` makes a broken assembly a skip, not a
crash:

```csharp
private static Type FindType(string fullName)
{
    foreach (var asm in AppDomain.CurrentDomain.GetAssemblies())
    {
        try
        {
            var t = asm.GetType(fullName, throwOnError: false);
            if (t != null) return t;
        }
        catch { /* ignore */ }
    }
    return null;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Bridge/CustomChirpsBridge.cs#L121-L133` (@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39)

### 4. Invoke through the cached `MethodInfo`, mapping the mirror enum by name

Convert your local mirror enum to the real one with `Enum.Parse(realEnumType, name)`, box
the arguments, and call the cached `MethodInfo`. A static method invokes with `null` as
the target:

```csharp
public static bool PostChirp(string text, DepartmentAccountBridge department, Entity entity, string customSenderName = null)
{
    EnsureResolve();
    var realDept = MapDepartment(department);          // Enum.Parse(_deptEnumType, department.ToString())
    var args = new object[] { text ?? string.Empty, realDept, entity, customSenderName };
    _postChirp.Invoke(null, args);
    return true;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Bridge/CustomChirpsBridge.cs#L62-L70` (@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39)

`MapDepartment` does `Enum.Parse(_deptEnumType, department.ToString(), ignoreCase: false)`
inside a `try` (`CustomChirpsBridge.cs#L75-L88`) - that name round-trip is exactly why the
mirror enum members must match the foreign enum's names.

### 5. Gate every call site on `IsAvailable`

The bridge is only half the pattern; callers must check the gate and bail early so the
absent-mod path costs nothing:

```csharp
private void TryPostDailyAnnouncements(int todaySimDay)
{
    if (!CustomChirpsBridge.IsAvailable)
        return;
    // ... build entities and post chirps only when the companion mod is present
}
```
Source: `../../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Systems/SpecialEventChirpSystem.cs#L146-L149` (@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39)

## Pitfalls & gotchas

- **Resolve once, cache the handles - never reflect per call.** The `_resolved` latch
  (step 2) guarantees the assembly scan and `GetMethod` run a single time; every
  subsequent `PostChirp` uses the cached `MethodInfo`. realistic-path-finding uses the
  same idea with a `_checked` latch for a field read (`Time2WorkInterop.cs#L15`,
  `#L48`). Reflecting on every invocation is the classic performance trap this pattern
  exists to avoid.

- **Overload ambiguity: pass the signature to `GetMethod`.** The two-arg
  `GetMethod(name, flags)` throws `AmbiguousMatchException` if the foreign API has
  overloads. elections resolves each overload precisely by passing the exact parameter
  types (and reuses the *resolved* enum type in that signature):

  ```csharp
  s_PostChirp = s_ApiType.GetMethod(
      "PostChirp",
      BindingFlags.Public | BindingFlags.Static,
      null,
      new[] { typeof(string), s_DepartmentEnumType, typeof(Entity), typeof(string) },
      null);
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/elections-rt-module/repo/Elections/Bridge/CustomChirpsBridge.cs#L249-L254` (@36c25afcda67c80ce75ba68d423fe8238499435a)

- **Degrade gracefully, and log the failure only once.** elections wraps the invoke in a
  `try/catch`, returns `false` on any exception, and guards the warning with a
  `s_LoggedFailure` flag so a broken companion API does not spam the log every tick:

  ```csharp
  if (s_PostChirp == null || s_DepartmentEnumType == null)
      return false;
  try
  {
      object realDepartment = Enum.Parse(s_DepartmentEnumType, department.ToString(), false);
      s_PostChirp.Invoke(null, new object[] { text ?? string.Empty, realDepartment, targetEntity, senderName });
      return true;
  }
  catch (Exception ex)
  {
      if (!s_LoggedFailure) { s_LoggedFailure = true; Mod.log.Warn($"CustomChirps invocation failed: {ex.Message}"); }
      return false;
  }
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/elections-rt-module/repo/Elections/Bridge/CustomChirpsBridge.cs#L54-L75` (@36c25afcda67c80ce75ba68d423fe8238499435a)

  Note the contrast with the time2work primary: its `PostChirp` invokes without a
  try/catch (`CustomChirpsBridge.cs#L62-L70`), so a signature drift there would throw at
  the call site. The elections style is the more defensive one.

- **Name coupling is the fragile point.** The whole bridge is stringly-typed:
  `"CustomChirps.Systems.CustomChirpApiSystem"`, the method name `"PostChirp"`, the enum
  member names. If the companion mod renames any of these, resolution returns `null` and
  your feature silently stops (that is the *intended* degrade, but there is no compile
  error to warn you). Re-verify the foreign names on each companion-mod update. Whether a
  given companion version still exposes these exact members is **Needs Verification
  (in-game)**.

- **Load order.** `Type.GetType`/`GetAssemblies` only see assemblies already loaded when
  `EnsureResolve` runs. Resolving lazily on first *use* (as all four mods do) rather than
  in `OnCreate` avoids racing the companion mod's load. Exactly when each mod's assembly
  is loaded relative to yours is **Needs Verification (in-game)**.

## Variations

- **Raw `AppDomain` scan without the `Type.GetType` shortcut (style 2).** anarchy skips
  the FQN attempt entirely and loops assemblies directly, then wraps the found `Type` in a
  `ComponentType` for use in an `EntityQuery` - detecting MoveIt's presence by whether its
  component type resolves:

  ```csharp
  Assembly[] assemblies = AppDomain.CurrentDomain.GetAssemblies();
  if (!m_FoundOriginalType)
  {
      foreach (Assembly assembly in assemblies)
      {
          Type type = assembly.GetType("MoveIt.Components.MIT_Original");
          if (type != null)
          {
              m_Log.Info($"Found {type.FullName} in {type.Assembly.FullName}. ");
              ComponentType originalType = ComponentType.ReadOnly(type);
              m_FoundOriginalType = true;
              // ... build a query that includes originalType
          }
      }
  }
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/Systems/MoveItIntegration/CopyAnarchyComponentsSystem.cs#L63-L73` (@a6311e898d20a775368668b234aaa32f06e3e1eb)

  anarchy uses the identical loop to detect the Platter mod's
  `Platter.Components.ParcelPlaceholderData`
  (`Anarchy/Systems/Common/AnarchyUISystem.cs#L332-L340`).

- **Read a field instead of calling a method, with a load-tolerant `GetTypes`.**
  realistic-path-finding reads Time2Work's static `timeReductionFactor` field. Its scan
  uses `SelectMany(a => a.GetTypes())` with a `ReflectionTypeLoadException` catch that
  keeps the non-null `e.Types` - a broken assembly does not abort the search - and it has
  a computed fallback when the preferred field is absent:

  ```csharp
  var type = AppDomain.CurrentDomain.GetAssemblies()
      .SelectMany(a => {
          try { return a.GetTypes(); }
          catch (ReflectionTypeLoadException e) { return e.Types.Where(t => t != null); }
      })
      .FirstOrDefault(t => t.FullName == "Time2Work.Time2WorkTimeSystem");

  if (type != null)
  {
      var f = type.GetField("timeReductionFactor", BindingFlags.Public | BindingFlags.Static);
      if (f != null && f.FieldType == typeof(float)) { /* use (float)f.GetValue(null) */ }
      // else fall back to deriving the factor from kTicksPerDay
  }
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Time2WorkInterop.cs#L19-L39` (@50645fa6a078181365e36a42e2e27b96699bf02a)

- **Overload cascade for progressive enhancement.** elections resolves many optional
  overloads (`PostChirpWith2Targets`, `...With3Targets`, `PostLargeChirpFromEntityWithPortraitImage`)
  and each richer method falls back to the simpler one when its handle is `null` - e.g.
  `if (s_PostChirpWith2Targets == null) return PostChirp(...)`
  (`Elections/Bridge/CustomChirpsBridge.cs#L78-L82`). This lets one bridge target several
  companion-API versions from a single build.

- **Reflection-*gated* optional interop - skip the whole feature branch when the type is
  absent.** better-bulldozer runs a one-shot post-load cleanup that only needs to touch
  network elements owned by the Traffic mod. It resolves `Traffic.Components.ModifiedConnections`
  with the same assembly-scan loop as anarchy above, but the *entire* work branch lives
  inside `if (type != null)` - when Traffic is not loaded no assembly returns the type, the
  loop finds nothing, and the enhancement silently no-ops. The resolved `Type` is wrapped
  in a `ComponentType` and used for `HasComponent` tests exactly like the anarchy MoveIt
  case, the difference being it runs once at load (gated behind `OnGameLoadingComplete` +
  a frame delay) rather than per-frame:

  ```csharp
  Assembly[] assemblies = AppDomain.CurrentDomain.GetAssemblies();
  foreach (Assembly assembly in assemblies)
  {
      Type type = assembly.GetType("Traffic.Components.ModifiedConnections");
      if (type != null)
      {
          m_Log.Info($"Found {type.FullName} in {type.Assembly.FullName}. ");
          ComponentType trafficModComponent = ComponentType.ReadOnly(type);
          // ... query permanently-removed subelements and only queue those
          //     whose edge endpoints carry the Traffic component for cleanup
      }
  }
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/better-bulldozer/repo/BetterBulldozer/Systems/RemoveRegeneratedSubelementPrefabsSystem.cs#L325-L355` (@4408466f226db811159d92859479ae1e1c28ba06)

## The provider side: designing a bridge others can reflect onto

Everything above is the *consumer* view - reflecting onto someone else's mod. If you
author the *host*, you make yourself reflectable by exposing a small, stable, `static`
surface whose names will not churn; it is the mirror image of the consumer pattern. Three
shapes appear in the wild.

### Provider A: a thread-safe static entry point backed by a queue

CustomChirps - the mod every `CustomChirpsBridge` in this recipe targets - exposes plain
`public static void PostChirp(...)` overloads that *any* thread may call. Each overload only
*enqueues* a request onto a `ConcurrentQueue`; the owning `GameSystemBase` drains that queue
on the main thread. That queue is what makes the API safe to invoke from a reflecting caller
you know nothing about:

```csharp
private static readonly ConcurrentQueue<PendingRequest> s_requests = new ConcurrentQueue<PendingRequest>();
private const int MaxRequestsPerFrame = 512;

// ======= Public API (thread-safe) =======
public static void PostChirp(string text, DepartmentAccount dept, Entity targetEntity, string customSenderName = null)
{
    EnqueueChirp(text, dept, targetEntity, customSenderName, Entity.Null, ChirpDisplayMode.Compact);
}
```
Source: `../../../vice-and-order-research/mods/dossiers/custom-chirps/repo/CustomChirps/Systems/CustomChirpApiSystem.cs#L68-L79` (@f018ac382e93e0b56cd7b974d1ce0b155d23d2b4)

The consumer end drains on the main thread with a *bounded* loop and swallows per-request
failures so one bad request cannot break the frame or leak an exception back across the
bridge:

```csharp
// Drain queued requests on the main thread
int processed = 0;
while (processed < MaxRequestsPerFrame && s_requests.TryDequeue(out var req))
{
    try
    {
        ProcessRequestOnMainThread(req);
    }
    catch (Exception ex)
    {
        _log.Error($"[CustomChirps] Failed to process chirp request: {ex}");
    }
    processed++;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/custom-chirps/repo/CustomChirps/Systems/CustomChirpApiSystem.cs#L253-L267` (@f018ac382e93e0b56cd7b974d1ce0b155d23d2b4)

Because a reflecting caller only ever sees these `static` methods, the enum parameter
(`DepartmentAccount`, `CustomChirpApiSystem.cs#L17-L37`) is the *one* type both sides must
agree on by name - which is exactly the enum the consumer mirrors and round-trips with
`Enum.Parse` in step 1 above.

### Provider B: a public static extension class

anarchy ships `Anarchy.Bridge.AnarchyBridge`, a `public static class` of `bool`-returning
helpers (`TryAddToolSystem`, `TryAddAnarchyComponent`, `TryAddTransformLockComponent`)
whose doc-comment states it is "for other mods to tie into Anarchy." Each helper resolves
Anarchy's own systems through `World.DefaultGameObjectInjectionWorld` *internally*, so the
caller passes only game types (`ToolBaseSystem`, `Entity`) and never touches an Anarchy
type - keeping the reflected surface free of foreign types the caller would have to mirror:

```csharp
public static bool TryAddToolSystem(ToolBaseSystem tool)
{
    AnarchyUISystem uiSystem = World.DefaultGameObjectInjectionWorld.GetOrCreateSystemManaged<AnarchyUISystem>();
    if (uiSystem is null ||
        tool is null ||
        tool.toolID is null)
    {
        return false;
    }

    return uiSystem.TryAddTool(tool.toolID);
}
```
Source: `../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/Bridge/AnarchyBridge.cs#L24-L35` (@a6311e898d20a775368668b234aaa32f06e3e1eb)

### Provider C: an inbound static push surface, versioned

Instead of exposing data to be *read*, a host can expose a `static` surface others push
*into*. Time2Work's `ElectionsBridge` is a `public static class` of setter/getter pairs
(`SetMayorResourceConsumptionMultiplier`, `SetElectionDaySundayOverride`, ...) over private
`static` backing fields, published with an `ApiVersion` constant so a reflecting caller can
check compatibility before pushing anything:

```csharp
public static class ElectionsBridge
{
    public const int ApiVersion = 3;

    private static int s_EffectId;
    private static float s_ResourceConsumptionMultiplier = 1f;

    public static int GetApiVersion()
    {
        return ApiVersion;
    }

    public static void SetMayorResourceConsumptionMultiplier(int effectId, float multiplier)
    {
        s_EffectId = effectId;
        s_ResourceConsumptionMultiplier = Clamp(multiplier, 0.75f, 1.25f);
    }
```
Source: `../../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Bridge/ElectionsBridge.cs#L5-L28` (@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39)

The `ApiVersion` constant is the provider-side answer to the "name coupling is fragile"
pitfall: it gives the reflecting side a single cheap value to gate on (via a reflected
`GetApiVersion()`) instead of probing every method. And note the setter *clamps* its input
to `0.75f..1.25f` (`ElectionsBridge.cs#L24-L27`) - a push surface must defend against
whatever an unknown caller sends, because there is no compiler between the two mods.

## See also
- Related recipes: [harmony-traverse-private-fields](harmony-traverse-private-fields.md)
  (reflection into private members of the *same* assembly you patch);
  [cross-mod-service-protocol](cross-mod-service-protocol.md) (designing the versioned
  static service surface a provider like `ElectionsBridge` exposes, from the protocol side).
- Reference: [dependency handling in UI](../ui/dependency-handling.md);
  [technique index](../../technique-index.md) (family G coverage ledger).
- Explanation: [dependency strategy](../../explanation/dependency-strategy.md) (the
  detect-and-degrade decision, of which this is the code-level realization).
- Case studies demonstrating it:
  [time2work-realistic-trips](../../case-studies/time2work-realistic-trips.md).

## Sources
- Canonical mods (dossier + repo):
  - `time2work-realistic-trips` @d42921ffb1f6bbbf2c2b086bb17727bf0f637c39 - `repo/NightShift/Bridge/CustomChirpsBridge.cs`, `repo/NightShift/Systems/SpecialEventChirpSystem.cs`
  - `elections-rt-module` @36c25afcda67c80ce75ba68d423fe8238499435a - `repo/Elections/Bridge/CustomChirpsBridge.cs`
  - `realistic-path-finding` @50645fa6a078181365e36a42e2e27b96699bf02a - `repo/RealisticPathFinding/Time2WorkInterop.cs`
  - `anarchy` @a6311e898d20a775368668b234aaa32f06e3e1eb - `repo/Anarchy/Systems/MoveItIntegration/CopyAnarchyComponentsSystem.cs`, `repo/Anarchy/Systems/Common/AnarchyUISystem.cs`, `repo/Anarchy/Bridge/AnarchyBridge.cs`
  - `time2work-realistic-trips` @d42921ffb1f6bbbf2c2b086bb17727bf0f637c39 - `repo/NightShift/Bridge/ElectionsBridge.cs` (provider-side push surface)
  - `custom-chirps` @f018ac382e93e0b56cd7b974d1ce0b155d23d2b4 - `repo/CustomChirps/Systems/CustomChirpApiSystem.cs`
  - `better-bulldozer` @4408466f226db811159d92859479ae1e1c28ba06 - `repo/BetterBulldozer/Systems/RemoveRegeneratedSubelementPrefabsSystem.cs`
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding
