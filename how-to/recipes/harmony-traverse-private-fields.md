---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: Reflect into private fields via Harmony Traverse"
recipe: harmony-traverse-private-fields
technique_family: "X - Reflect into private fields via Harmony Traverse"
diataxis: how-to
source_version: "~1.5.9 (anarchy@a6311e8; date-pinned, static source only)"
last_reverified: "2026-07-04"
status: source-verified
canonical_mods:
  - anarchy@a6311e898d20a775368668b234aaa32f06e3e1eb
  - time2work-realistic-trips@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39
  - traffic-tool-essentials@10973595ac9ed37f47ca24eb788d4421dc295fa0
  - realistic-jobsearch@7a096b2ab974bb03cc4cf0937f250bf1d7671f31
  - specialized-industrial-zones@8d9153c3fda72e3b79a133bee6c83f9ba7df8b5a
technique_applicability: [platform]
Created: 2026-07-04
Updated: 2026-07-05
Owners:
  - codex
---

# Reflect into private fields via Harmony Traverse

> Read or write a `private`/`internal` field on a vanilla manager or system object -
> with no source access to that field - using HarmonyLib's `Traverse` reflection
> wrapper.

## Problem
You need a value that the game keeps behind a private field and never exposes through
a public API: the live `World` off `GameManager.instance`, the `m_Prefabs` list inside
`PrefabSystem`, or the `m_Time`/`m_Date`/`m_Year` fields the vanilla `TimeSystem`
owns. You can `[HarmonyPatch]` the method that touches them, but the field itself has
no getter/setter. You want to reach it by **name** without hand-writing `FieldInfo`
plumbing and `BindingFlags`.

## Solution
Use `HarmonyLib.Traverse`. `Traverse.Create(obj)` wraps any instance; `.Field<T>(name)`
selects a private field by its literal string name; `.Value` is a get/set property.
Reading is `Traverse.Create(obj).Field<T>("m_Name").Value`; writing is
`Traverse.Create(obj).Field("m_Name").SetValue(newValue)`. It resolves the field by
reflection each call, so it is the shortest path to a private member when no public API
exists - at the cost of a string name that the compiler cannot check. Reach for it
only when there is genuinely no public accessor, and null-guard every read because a
renamed or missing field yields `null`, not an exception, on the get path.

## Steps & Code

### 1. Read a private field on a vanilla singleton (`GameManager.m_World`)

Anarchy's Extended Road Upgrades needs the exact `World` the game is running, which
lives in a private `m_World` field on `GameManager.instance`. `Traverse.Create(...)`
wraps the singleton, `.Field<World>("m_World")` names the field, `.Value` reads it:

```csharp
// Getting World instance
world = Traverse.Create(GameManager.instance).Field<World>("m_World").Value;
if (world == null)
{
    AnarchyMod.Instance.Log.Error($"{logHeader} Failed retrieving World instance, exiting.");
    return;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/ExtendedRoadUpgrades/UpgradesManager.cs#L93-L99` (@a6311e898d20a775368668b234aaa32f06e3e1eb)

Note the immediate `null` guard: on the read path a wrong or renamed field name comes
back as `null`, never an exception (see Pitfalls).

### 2. Read a private collection off a resolved system (`PrefabSystem.m_Prefabs`)

The same shape reaches into a system instance. After resolving `prefabSystem` from the
`World`, Anarchy pulls its internal prefab list - a `private List<PrefabBase>
m_Prefabs` - the same way, with the generic `T` set to the field's real type:

```csharp
// Getting Prefabs list from PrefabSystem
var prefabs = Traverse.Create(prefabSystem).Field<List<PrefabBase>>("m_Prefabs").Value;
if (prefabs == null || !prefabs.Any())
{
    AnarchyMod.Instance.Log.Error($"{logHeader} Failed retrieving Prefabs list, exiting.");
    return;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/ExtendedRoadUpgrades/UpgradesManager.cs#L113-L119` (@a6311e898d20a775368668b234aaa32f06e3e1eb)

Anarchy repeats this exact `Traverse.Create(prefabSystem).Field<List<PrefabBase>>("m_Prefabs").Value`
call in a second code path (`UpgradesManager.cs#L262`), confirming it is the mod's
standard idiom for the private prefab list.

### 3. Write a private field from inside a Harmony patch (`TimeSystem.m_Time`)

Reading uses `Field<T>(name).Value`; **writing** uses the non-generic
`.Field(name).SetValue(...)`. Time2Work postfix-patches `TimeSystem.OnUpdate` and
overwrites the game's private clock fields with its own accelerated values. `__instance`
is the patched `TimeSystem` supplied by Harmony:

```csharp
[HarmonyPatch(typeof(TimeSystem), "OnUpdate")]
[HarmonyPostfix]
public static void TimeSystemPatches_OnUpdate_Postfix(TimeSystem __instance)
{
    Time2WorkTimeSystem t2wTimeSystem = World.DefaultGameObjectInjectionWorld.GetOrCreateSystemManaged<Time2WorkTimeSystem>();
    Traverse.Create(__instance).Field("m_Time").SetValue(t2wTimeSystem.normalizedTime);
    Traverse.Create(__instance).Field("m_Date").SetValue(t2wTimeSystem.normalizedDate);
    Traverse.Create(__instance).Field("m_Year").SetValue(t2wTimeSystem.year);
}
```
Source: `../../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Patches/Time2WorkPatches.cs#L34-L42` (@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39)

`SetValue` writes silently: unlike the read path, there is no `null` to check, so a
mistyped name simply fails to land the value. Combining the Harmony patch (to run at
the right moment, with the right `__instance`) with `Traverse.SetValue` (to reach the
private field) is the canonical write pattern.

## Pitfalls & gotchas

- **Name-string coupling breaks silently on rename.** `Field<T>("m_World")`,
  `Field<List<PrefabBase>>("m_Prefabs")`, `Field("m_Time")` are all literal strings the
  C# compiler never checks. If a game patch renames or removes the field, `Traverse`
  does not throw at the call site - the read path returns `null` and the write path
  quietly no-ops. This is why every Anarchy read is followed by an explicit `if
  (x == null)` bail-out
  (`../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/ExtendedRoadUpgrades/UpgradesManager.cs#L95`,
  `#L115`). Use this technique only when there is genuinely no public API; when there
  is one, prefer it.

- **Get-path failure is `null`, not an exception.** A wrong field name (or wrong
  generic `T`) on the read path surfaces as a `null`/default `Value`, so you MUST
  null-guard - a missing guard turns a rename into a `NullReferenceException` deep in
  your own code, far from the real cause. The write path (`SetValue`) gives you even
  less: no return value and nothing to inspect.

- **Per-call reflection cost.** Each `Traverse.Create(...).Field(...).Value` resolves
  the field by reflection on every invocation. Anarchy's calls run once during install,
  so cost is irrelevant; Time2Work's run inside a `TimeSystem.OnUpdate` postfix - a hot
  path. If you must reflect every frame, cache a `FieldInfo` once instead (see
  Variations). Whether the per-frame `Traverse` calls cause a measurable frame cost in
  a large city is `Needs Verification (in-game)`.

- **Generic `T` must match the field's declared type for reads.**
  `Field<World>("m_World")` and `Field<List<PrefabBase>>("m_Prefabs")` name both the
  field and its type. A mismatched `T` is another silently-wrong path. The write form
  `Field(name).SetValue(v)` skips the generic but then relies on `v` being
  assignment-compatible - also unchecked at compile time.

- **You are reaching past the public contract.** Private fields are private for a
  reason; the game team can change them in any patch with no deprecation. Treat every
  `Traverse` field name as a re-verification point on each CS2 release.

## Variations

- **Cache a `FieldInfo` for hot-path reflection (no HarmonyLib).** When you reflect
  every frame, resolve the field once with raw `System.Reflection` and reuse the handle.
  Traffic Tool Essentials reads `ToolRaycastSystem`'s private `m_RaycastSystem` and its
  nested private `m_Result` this way, caching the `FieldInfo`s to "minimize per-frame
  reflection":

  ```csharp
  var raycastSystemField = toolRaycastType.GetField("m_RaycastSystem",
      BindingFlags.NonPublic | BindingFlags.Instance);
  if (raycastSystemField == null) { /* log + bail */ return; }
  m_RaycastSystemRef = raycastSystemField.GetValue(m_ToolRaycastSystem);
  m_CachedResultField = m_RaycastSystemRef.GetType().GetField("m_Result",
      BindingFlags.NonPublic | BindingFlags.Instance);
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/traffic-tool-essentials/repo/TrafficToolEssentials/Systems/MobilityHub/MobilityHubSelectionSystem.cs#L192-L209` (@10973595ac9ed37f47ca24eb788d4421dc295fa0)

  Same private-field-by-name idea, but you write the `BindingFlags` yourself and hold a
  reusable `FieldInfo m_CachedResultField`
  (`MobilityHubSelectionSystem.cs#L51`) instead of paying reflection cost per call.
  `Traverse` is the concise choice for one-shot access; cached `FieldInfo` is the
  faster choice for per-frame access.

- **Compiled `FieldRef` for `ref` access on the hot path (`AccessTools.FieldRefAccess`).**
  HarmonyLib's `AccessTools.FieldRefAccess<TObject,TField>(name)` compiles a delegate
  once and returns a `ref` to the private field itself - no boxing, no per-call
  reflection, and the same handle reads *and* writes. Realistic Job Search caches two
  such refs at type-init to reach `PathfindSetupSystem`'s private `m_SetupList` and
  `m_SetupDependencies`:

  ```csharp
  static readonly AccessTools.FieldRef<PathfindSetupSystem, NativeList<PathfindSetupSystem.SetupListItem>> _setupListRef =
      AccessTools.FieldRefAccess<PathfindSetupSystem, NativeList<PathfindSetupSystem.SetupListItem>>("m_SetupList");

  static readonly AccessTools.FieldRef<PathfindSetupSystem, Unity.Jobs.JobHandle> _setupDepsRef =
      AccessTools.FieldRefAccess<PathfindSetupSystem, Unity.Jobs.JobHandle>("m_SetupDependencies");
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/realistic-jobsearch/repo/RealisticJobSearch/Patches/Patch_CompleteSetup_FilterJobSeekerTargets.cs#L27-L31` (@7a096b2ab974bb03cc4cf0937f250bf1d7671f31)

  Inside the patch it dereferences them as genuine `ref` locals -
  `ref var setupDeps = ref _setupDepsRef(__instance);` then `setupDeps.Complete();`
  (`Patch_CompleteSetup_FilterJobSeekerTargets.cs#L52`, `#L55`) - which is exactly why a
  `ref`-returning `FieldRef` (not a copy) matters for a mutable `NativeList`/`JobHandle`.
  This completes a three-rung ladder for private-field access: `Traverse` (resolves by
  reflection every call - the one-shot choice), cached `FieldInfo` (resolved once, but
  each `GetValue`/`SetValue` boxes and copies), and `FieldRefAccess` (compiled once,
  direct `ref`, no boxing - the choice for a per-frame or per-setup hot path). Whether
  the gap is measurable in a real city is `Needs Verification (in-game)`.

- **Mutate a private `NativeArray` in place from a postfix (no field write-back).** When
  the private member is a `NativeArray<T>` - a handle over native memory - you cache a
  `FieldInfo`, `GetValue` the handle once, and write *through* it; there is no need to
  `SetValue` the field back, because the elements live outside the managed object.
  Time2Work's night-discount postfix on `CityServiceBudgetSystem.OnUpdate` grabs the
  system's private `m_Expenses`/`m_ExpensesTemp` arrays and scales one slot:

  ```csharp
  private static readonly FieldInfo s_ExpensesField =
      AccessTools.Field(typeof(CityServiceBudgetSystem), "m_Expenses");
  // ...
  var expenses = (NativeArray<int>)s_ExpensesField.GetValue(__instance);
  int idx = (int)ExpenseSource.ServiceUpkeep;
  if (idx >= 0 && idx < expenses.Length)
      expenses[idx] = (int)math.round(expenses[idx] * factor);
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Patches/Time2WorkPatches.cs#L229-L265` (@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39)

  Contrast Step 3: `TimeSystem.m_Time` is a value field, so its write needs
  `Traverse...SetValue`; a `NativeArray` field is a struct handle, so `GetValue` once
  then index-assignment lands the change. Note `AccessTools.Field(typeof(T), name)` is
  HarmonyLib's one-liner for the raw `GetField(name, BindingFlags.NonPublic | Instance)`
  you write by hand in the cached-`FieldInfo` variation above.

- **Fail loudly instead of silently: `typeof(T).GetField(...) ?? throw`.** The get-path
  "returns `null`, never throws" behavior (see Pitfalls) can be turned into an explicit,
  diagnosable failure at the read site. Specialized Industrial Zones reads the very same
  `PrefabSystem.m_Prefabs` list that Anarchy reaches via `Traverse` (Step 2), but with
  raw `typeof().GetField` and a `?? throw` that names the likely cause:

  ```csharp
  _allPrefabs = typeof(PrefabSystem)
      .GetField("m_Prefabs", System.Reflection.BindingFlags.NonPublic | System.Reflection.BindingFlags.Instance)
      ?.GetValue(_prefabSystem) as List<PrefabBase>
      ?? throw new Exception("Could not access m_Prefabs field in PrefabSystem, likely broken by game update");
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/specialized-industrial-zones/repo/src/SpecializedIndustryZones/SpecializedZoningSystem.cs#L48-L51` (@8d9153c3fda72e3b79a133bee6c83f9ba7df8b5a)

  The `?.` short-circuits a missing field to `null`, the `as` short-circuits a
  type-mismatch to `null`, and the `?? throw` converts either into a message aimed
  straight at the next maintainer - the actionable form of the Pitfalls null-guard,
  chosen here because the whole system cannot function without that list.

- **Prefer a public API first.** Traffic Tool Essentials tries a direct public method
  (`GetRaycastResult`) and only falls back to reflection when it is absent
  (`MobilityHubSelectionSystem.cs#L155-L172`). Do the same: reflect only after
  confirming there is no supported accessor.

## See also
- Explanation: [Harmony patching](../../explanation/harmony-patching.md) (how `[HarmonyPatch]`,
  postfixes, and `__instance` work - the vehicle Time2Work uses to reach `m_Time`).
- Related recipes: [reflection mod bridges](reflection-mod-bridges.md) (reaching into
  *another mod's* internals by name, the cross-mod cousin of this technique).
- Case studies demonstrating it: [anarchy](../../case-studies/anarchy.md) (the
  `m_World`/`m_Prefabs` read exemplar),
  [time2work-realistic-trips](../../case-studies/time2work-realistic-trips.md) (the
  `SetValue` write exemplar).

## Sources
- Canonical mods (dossier + repo):
  - `anarchy` @a6311e898d20a775368668b234aaa32f06e3e1eb - `repo/Anarchy/ExtendedRoadUpgrades/UpgradesManager.cs`
  - `time2work-realistic-trips` @d42921ffb1f6bbbf2c2b086bb17727bf0f637c39 - `repo/NightShift/Patches/Time2WorkPatches.cs` (both the `m_Time` `SetValue` write and the `CityServiceBudgetSystem` `NativeArray` reflection)
  - `traffic-tool-essentials` @10973595ac9ed37f47ca24eb788d4421dc295fa0 - `repo/TrafficToolEssentials/Systems/MobilityHub/MobilityHubSelectionSystem.cs`
  - `realistic-jobsearch` @7a096b2ab974bb03cc4cf0937f250bf1d7671f31 - `repo/RealisticJobSearch/Patches/Patch_CompleteSetup_FilterJobSeekerTargets.cs`
  - `specialized-industrial-zones` @8d9153c3fda72e3b79a133bee6c83f9ba7df8b5a - `repo/src/SpecializedIndustryZones/SpecializedZoningSystem.cs`
- Official/community references (link out, do not duplicate):
  https://harmony.pardeike.net/articles/utilities.html (HarmonyLib `Traverse`)
