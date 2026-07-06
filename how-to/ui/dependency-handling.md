---
FrontmatterVersion: 1
DocumentType: Guide
Title: "How-to: Detect a missing UI dependency and degrade gracefully"
diataxis: how-to
source_version: "1.6.0f1 (realistic-path-finding@50645fa; date-pinned, static source only)"
last_reverified: "2026-07-04"
canonical_mods:
  - realistic-path-finding@50645fa6a078181365e36a42e2e27b96699bf02a
technique_applicability: [core, ui]
status: source-verified
Created: 2026-07-03
Updated: 2026-07-03
Owners:
  - codex
---

# Detect a missing UI dependency and degrade gracefully

> When your mod's UI leans on a shared library - an icon pack like Unified Icon
> Library (UIL) or a localization helper like I18n Everywhere - check at runtime
> whether it is actually loaded, set a feature flag, and fall back to plain text or
> neutral assets instead of throwing.

## Problem
Your mod's Options page or tool UI uses icons from a shared icon library, or resolves
strings through a localization helper mod. Those are separate downloads. If a player
runs without them - disabled, failed to load, or you only *soft*-depend on them - any
code that references a missing icon URI or helper type will error, potentially every
frame. You want the mod to notice the absence once, warn once, and keep working with a
reduced UI.

## Solution
Do not trust publish-time metadata to guarantee a dependency is present when your code
runs. **Validate at runtime**: scan the loaded assemblies for the dependency by name,
record a boolean feature flag during `OnLoad`, and branch every UI code path on that
flag - icons when present, text labels when not. Log exactly one warning per missing
dependency. The source-verified core is a defensive assembly scan
(`AppDomain.CurrentDomain.GetAssemblies()`) that tolerates partial-load failures and
defaults to a safe value; the same idiom that detects an optional *simulation* mod
detects an optional *UI* library. The concept and its rationale live in
[Dependency Strategy](../../explanation/dependency-strategy.md); this page is the
task.

## Steps & Code

### 1. Probe for the dependency by name at runtime

Scan `AppDomain.CurrentDomain.GetAssemblies()` for the dependency's type (or just its
assembly name), tolerate `ReflectionTypeLoadException` from any half-loaded assembly,
and **default to a safe value** if it is absent. Realistic Path Finding uses exactly
this shape to soft-detect the Time2Work mod - the identical pattern works for a UI
library:

```csharp
public static float GetFactor()
{
    if (_checked) return _factor;   // scan once, then cache
    _factor = 1f;                   // safe default when the dependency is absent
    try
    {
        var type = AppDomain.CurrentDomain.GetAssemblies()
            .SelectMany(a => {
                try { return a.GetTypes(); }
                catch (ReflectionTypeLoadException e) { return e.Types.Where(t => t != null); }
            })
            .FirstOrDefault(t => t.FullName == "Time2Work.Time2WorkTimeSystem");

        if (type != null) { /* read a member off the detected type via reflection */ }
    }
    catch { /* ignore - keep the safe default */ }
    _checked = true;
    return _factor;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Time2WorkInterop.cs#L13-L52` (@50645fa6a078181365e36a42e2e27b96699bf02a) - scan L19-L24, safe default L16, cache L15/L48.

For a **presence-only** check (you only need "is it loaded?", not a type member),
reduce the scan to the assembly name - the generalized helper in
[Dependency Strategy](../../explanation/dependency-strategy.md#generalising-to-a-reusable-status-object):

```csharp
private static bool IsAssemblyLoaded(string assemblyName) =>
    AppDomain.CurrentDomain.GetAssemblies()
        .Any(a => a.GetName().Name.Equals(assemblyName, StringComparison.OrdinalIgnoreCase));
```
This is a generic illustrative helper (not a specific mod's line), applying the same
`GetAssemblies()` idea RPF uses. Point it at the icon library's or localization
helper's assembly name.

### 2. Alternative: enumerate the mod manager

If you want to detect a *mod* (not just an assembly), the game's mod manager can be
iterated and matched by asset name. RPF does this to log Time2Work's presence and set
a flag:

```csharp
bool realisticTripsMod = false;
foreach (var modInfo in GameManager.instance.modManager)
{
    if (modInfo.asset.name.Equals("Time2Work"))
    {
        Mod.log.Info($"Loaded ... with time factor: {Time2WorkInterop.GetFactor()}");
        realisticTripsMod = true;
    }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Mod.cs#L43-L51` (@50645fa6a078181365e36a42e2e27b96699bf02a).

### 3. Set a feature flag once in `OnLoad` and warn once

Call the probe during `OnLoad`, store the result, and log a single warning if the
dependency is missing. Do the probe here - not per frame:

```csharp
// Generic illustrative pattern (see Dependency Strategy for the full version).
_hasIconLibrary = IsAssemblyLoaded("SomeIconLibrary");
_hasLocalizationHelper = IsAssemblyLoaded("SomeLocalizationHelper");

if (!_hasIconLibrary)
    Log.Warn("Icon library missing. Falling back to text labels.");
```
Pattern source: [Dependency Strategy](../../explanation/dependency-strategy.md)
("Generalising to a reusable status object" and "Degrading gracefully"), grounded in
the RPF scan above.

### 4. Branch the UI on the flag - icons vs text fallback

Every UI path that touches the dependency reads the flag rather than re-probing. When
the icon library is absent, substitute a plain text label or a neutral built-in asset
so the control still renders and stays usable. RPF's consumer shows the degradation
shape: with the dependency absent the factor is the neutral `1f`, and the system
simply **skips its extra work**:

```csharp
float factor = Time2WorkInterop.GetFactor(); // returns 1f if the dependency is absent
if (factor == 1f) return;                     // nothing to adjust -> no-op, no error
```
Source: `../../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Systems/ScaleWaitingTimeSystem.cs#L19-L22` (@50645fa6a078181365e36a42e2e27b96699bf02a).

The UI equivalent: if `!_hasIconLibrary`, emit a text label instead of an icon URI; if
`!_hasLocalizationHelper`, fall back to your own bundled `en-US` strings (or raw keys)
instead of the helper's expanded catalogue. You can also surface a short notice in the
Options page (a read-only text row - see
[Build a mod Options-menu UI](options-ui.md)) telling the player which optional
dependency is missing and where to get it.

## Pitfalls & gotchas

- **Metadata is not a guarantee.** A dependency declared in `PublishConfiguration.xml`
  can still be absent at runtime (disabled, failed load, or a soft dependency you never
  required). Always validate in code.

- **No hard compile-time reference to an optional dependency.** A hard reference makes
  *your* mod fail to load when the optional one is absent. Match by name via reflection
  for optional integrations.

- **Tolerate partial-load failures.** `assembly.GetTypes()` can throw
  `ReflectionTypeLoadException`; catch it and keep the loadable types
  (`Time2WorkInterop.cs#L21-L22`), so one broken assembly does not crash your probe.

- **Probe once, cache, warn once.** Assembly scans are not free. Scan in `OnLoad`,
  cache the boolean, and log at most one warning per missing dependency - never inside
  `OnUpdate` (`Time2WorkInterop.cs#L15`/`L48` cache the result).

- **Do not bundle the dependency.** Shipping a copy of a shared icon/localization DLL
  causes version conflicts. Reference (or reflect), do not bundle - see
  [Dependency Strategy](../../explanation/dependency-strategy.md#common-pitfalls).

- **The UI-side fallback rendering is not source-provable here.** The RPF citations
  prove the *detection + degrade* idiom in C#. The specific icon-URI-to-text swap in a
  Gameface/React control, and how a missing icon actually renders, are
  general guidance / `Needs Verification (in-game)` - they depend on your UI layer,
  which is not exercised in the cited sources.

## Variations

- **Reusable status object.** When several UI libraries are optional, collect the
  checks into one `DependencyStatus` built in `OnLoad` and read its booleans
  everywhere - full pattern in
  [Dependency Strategy](../../explanation/dependency-strategy.md#generalising-to-a-reusable-status-object).
- **Skip system/UI registration entirely.** For a feature that is meaningless without
  its dependency, do not register its systems at all rather than scheduling them to
  no-op every tick.
- **Typed bridge / adapter.** For richer soft integration (call the other mod's API
  when present), wrap it behind an adapter that reflects members and caches handles, so
  absence is a no-op - the "Soft inter-mod integration" section of the explanation.

## See also
- Explanation (the "why"): [Dependency Strategy](../../explanation/dependency-strategy.md)
  - declare / reference / validate / degrade, and the reusable status helper.
- How-to: [Build a mod Options-menu UI](options-ui.md) - where to surface a
  "dependency missing" notice; [Declare key bindings](key-bindings.md).
- Explanation: [module layout](../../explanation/module-layout.md) - referencing a
  shared assembly with `<Private>false</Private>` instead of bundling it;
  [Gameface runtime](../../explanation/gameface-runtime.md) (forward reference) for
  the UI layer the fallbacks target.
- Index: [technique index](../../technique-index.md).

## Sources
- Canonical mods (dossier + repo):
  - `realistic-path-finding` @50645fa6a078181365e36a42e2e27b96699bf02a - `repo/RealisticPathFinding/Time2WorkInterop.cs`, `repo/RealisticPathFinding/Mod.cs`, `repo/RealisticPathFinding/Systems/ScaleWaitingTimeSystem.cs`
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding
