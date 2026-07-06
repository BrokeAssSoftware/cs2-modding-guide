---
FrontmatterVersion: 1
DocumentType: Guide
Title: Dependency Strategy
Summary: How a CS2 mod handles a dependency on a shared library or another mod - declare it in publish metadata, reference it without bundling, validate it is actually loaded at runtime, and degrade gracefully when it is absent - explained against a real runtime assembly-detection pattern.
diataxis: explanation
source_version: "1.6.0f1 (realistic-path-finding@50645fa; date-pinned, static source only)"
last_reverified: "2026-07-04"
canonical_mods:
  - realistic-path-finding@50645fa6a078181365e36a42e2e27b96699bf02a
  - road-speed-adjuster@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed
  - market-based-economy@b83f196a36bc74388accebdeb7c81f0f35dbab37
  - realistic-jobsearch@7a096b2ab974bb03cc4cf0937f250bf1d7671f31
  - magic-mail@6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47
status: source-verified
Created: 2026-07-02
Updated: 2026-07-05
Owners:
  - codex
References:
  - Label: Module Layout (companion concept)
    Path: ./module-layout.md
  - Label: Lifecycle and Initialization (companion concept)
    Path: ./mod-lifecycle.md
  - Label: Technique Index
    Path: ../technique-index.md
---

# Dependency Strategy

Many CS2 mods lean on shared libraries (icon libraries, localisation helpers, UI toolkits)
or soft-integrate with *other mods* (reading another mod's data if it happens to be
installed). A dependency introduces a failure mode that a self-contained mod does not have:
the dependency might be missing, disabled, or loaded *after* you. Handling that well is what
separates a mod that shows a clean warning from one that throws a `TypeLoadException` on a
player's machine.

This page explains the *strategy* for depending on something you do not control: the four
things you must do (declare, reference, validate, degrade) and *why* each is necessary. It
is a concept page, not a step-by-step recipe.

## The dependency lifecycle

A dependency lives in four places, and skipping any one of them produces a specific class of
bug:

1. **Declare** it in publishing metadata (`PublishConfiguration.xml`, and the UI package's
   `mod.json` if the mod ships a Gameface UI). This is what makes the in-game publisher offer
   to install prerequisites for the player. *Skip this and players load your mod without its
   dependency and hit runtime errors.*
2. **Reference** the assembly for compilation *without bundling it* (`<Private>false</Private>`,
   see [Module Layout](./module-layout.md)). *Skip the non-private flag and you ship a stale
   copy that fights the real one.*
3. **Validate at runtime** that the dependency is actually loaded, and record the result.
   Metadata declares an *intention*; it does not guarantee the assembly is present and loaded
   when your `OnLoad` runs. *Skip this and a missing dependency surfaces as a crash instead of
   a warning.*
4. **Degrade gracefully** when it is absent - fall back to reduced behaviour and log exactly
   one warning. *Skip this and every feature that touches the dependency throws, per frame.*

Declaring is covered by publishing docs; referencing is [Module Layout](./module-layout.md).
The interesting - and most-skipped - half is runtime validation and graceful degradation.

## Why metadata is not enough: runtime validation

Load order across mods is not something you control. A dependency declared in metadata can
still be absent at the moment your code runs (the player disabled it, it failed to load, or
you are soft-depending on a mod you did not require). So the robust pattern is a **defensive
assembly check**: scan the loaded assemblies for the dependency by name, and branch on the
result instead of assuming it is there.

Realistic Path Finding does exactly this to soft-integrate with the Time2Work mod. It scans
`AppDomain.CurrentDomain.GetAssemblies()`, tolerates `ReflectionTypeLoadException`, looks for
a specific type by full name, and - crucially - falls back to a safe default and caches the
result so the scan runs once
([`realistic-path-finding` `repo/RealisticPathFinding/Time2WorkInterop.cs#L11-L52`](../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Time2WorkInterop.cs), commit `50645fa`):

```csharp
static bool _checked;
static float _factor = 1f;

public static float GetFactor()
{
    if (_checked) return _factor;   // cache: scan the assembly list once
    _factor = 1f;                   // safe default if the dependency is absent
    try
    {
        var type = AppDomain.CurrentDomain.GetAssemblies()
            .SelectMany(a => {
                try { return a.GetTypes(); }
                catch (ReflectionTypeLoadException e) { return e.Types.Where(t => t != null); }
            })
            .FirstOrDefault(t => t.FullName == "Time2Work.Time2WorkTimeSystem");

        if (type != null)
        {
            // read a public static field off the detected type via reflection...
        }
    }
    catch { /* ignore - keep the safe default */ }
    _checked = true;
    return _factor;
}
```

Three properties make this correct, and they generalise to any dependency check:

- **Match by name via reflection**, so you have no *hard* compile-time reference to a mod
  that may not be installed. (`AppDomain.CurrentDomain.GetAssemblies()` also lets you match
  the *assembly* name directly when you only need presence, not a type.)
- **Tolerate partial-load failures.** `a.GetTypes()` can throw `ReflectionTypeLoadException`;
  catching it and keeping the loadable types stops one broken assembly from crashing your
  probe.
- **Default safe and cache.** If detection fails, keep working with a neutral value, and do
  not re-scan every call.

### Generalising to a reusable status object

When a mod depends on several shared libraries it is convenient to collect the checks into a
single status object built once in `OnLoad`, then read the booleans everywhere else. The
following is a **generic illustrative pattern** (not a specific mod's code) that applies the
same `GetAssemblies()` idea RPF uses, generalised for a hypothetical `MyMod.Core`:

```csharp
public sealed class Mod : IMod
{
    private static readonly ILog Log = LogManager
        .GetLogger("MyMod.Core.Mod")
        .SetShowsErrorsInUI(false);

    private readonly DependencyStatus _dependencies = new();

    public void OnLoad(UpdateSystem updateSystem)
    {
        _dependencies.Refresh();

        if (!_dependencies.HasIconLibrary)
            Log.Warn("Icon library missing. Falling back to text labels.");

        // Gate feature registration on dependency availability.
        if (_dependencies.HasIconLibrary)
            IconBootstrap.Register();
        else
            FallbackBootstrap.Register();
    }
}

internal sealed class DependencyStatus
{
    public bool HasIconLibrary { get; private set; }
    public bool HasLocalizationHelper { get; private set; }

    public void Refresh()
    {
        HasIconLibrary = IsAssemblyLoaded("SomeIconLibrary");
        HasLocalizationHelper = IsAssemblyLoaded("SomeLocalizationHelper");
    }

    private static bool IsAssemblyLoaded(string assemblyName) =>
        AppDomain.CurrentDomain
            .GetAssemblies()
            .Any(a => a.GetName().Name.Equals(assemblyName, StringComparison.OrdinalIgnoreCase));
}
```

The `IsAssemblyLoaded` helper is the same `GetAssemblies()` scan RPF uses, reduced to a
presence test. Promote it into a shared utility assembly (for example `MyMod.Core`) when more
than one module needs the same guard, to avoid the checks drifting apart.

## Degrading gracefully

Once you know a dependency is missing, the whole point is to keep the mod *usable*:

- **Expose booleans on settings** (`HasIconLibrary`, `HasLocalizationHelper`) so UI and system
  code can branch cleanly instead of re-probing.
- **Provide UI fallbacks** - replace icon URIs with plain text labels or neutral assets when
  the icon library is unavailable.
- **Skip system registration** for systems that require a missing dependency, rather than
  scheduling them and letting them throw inside `OnUpdate` every tick (see
  [Lifecycle and Initialization](./mod-lifecycle.md) for where scheduling sits in `OnLoad`).
- **Log exactly one warning per missing dependency during `OnLoad`** - never per frame.

## Soft inter-mod integration

The RPF example above is not really about a *required* library; it is about *soft*
integration - "if Time2Work is present, cooperate with it; otherwise behave normally." That
is a distinct and valuable posture: wrap the other mod's API behind a typed adapter (a
"bridge") that checks for the assembly, resolves members via reflection, caches handles, and
guards every entry point so the absence of the other mod is a no-op rather than an error. The
reflection-and-cache shape shown above is the reusable core of such a bridge.

## Interop & compatibility hazards

Assembly-presence checks (above) handle *code* coupling. A second, quieter class of
dependency does not live in an assembly at all: two mods that write the *same shared data* -
a prefab field, a live-graph edge, an ECS component buffer, or the savegame itself. Nothing
about detection helps here; the hazard is *ordering* and *persistence*, and it survives long
after your `OnLoad` has run.

### Multi-writer clobber of shared data (last-write-wins)

When two mods mutate the same game data, whichever runs last wins, and neither knows the other
touched it. This is a data-level conflict, not an assembly-load one, so the reflection guards
above do not catch it.

- **Live-graph and prefab writes are not multi-writer safe.** Realistic Path Finding restores
  pedestrian costs from *its own cached baseline*, so a later reapply overwrites any change
  another mod made to the same prefab cost or live-graph edge after RPF captured it. The mod
  documents this explicitly and ships `disable_ped_cost` as an escape hatch that lets users
  restore RPF's changes - but the docs are careful to note it "does not make the shared data
  fully multi-writer safe"
  ([`realistic-path-finding` `repo/docs/pedestrian-cost-systems.md#L47-L56`](../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/docs/pedestrian-cost-systems.md), commit `50645fa`).
- **Direct net-lane writes collide with every other speed writer.** Road Speed Adjuster writes
  `m_DefaultSpeedLimit` and `m_SpeedLimit` straight onto the live `CarLane`/`TrackLane`
  components of each sub-lane
  ([`road-speed-adjuster` `repo/Systems/RoadSpeedApplySystem.cs#L109-L136`](../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/Systems/RoadSpeedApplySystem.cs), commit `e0c0c0b`).
  Any other mod that sets lane speed limits fights it frame-to-frame; there is no arbitration,
  only whichever system's `OnUpdate` ran most recently.
- **Shared ECS components are a read-after-write hazard.** Market Based Economy rewrites
  `WorkProvider.m_MaxWorkers` on company entities from a Harmony postfix on
  `WorkProviderSystem.OnUpdate`
  ([`market-based-economy` `repo/Economy/WorkforceUtilizationManager.cs#L43-L95`](../../vice-and-order-research/mods/dossiers/market-based-economy/repo/Economy/WorkforceUtilizationManager.cs) and
  [`repo/Harmony/HarmonyBridge.cs#L142-L143`](../../vice-and-order-research/mods/dossiers/market-based-economy/repo/Harmony/HarmonyBridge.cs), commit `b83f196`).
  Realistic JobSearch *reads* the same field to size its job-seeker targeting
  ([`realistic-jobsearch` `repo/RealisticJobSearch/Systems/GravityPreFilterSystem.cs#L199-L200`](../../vice-and-order-research/mods/dossiers/realistic-jobsearch/repo/RealisticJobSearch/Systems/GravityPreFilterSystem.cs)),
  so if MBE's postfix runs first, JobSearch silently consumes MBE's mutated worker counts
  instead of the vanilla value. Two mods can each be "correct" in isolation and still produce
  a wrong result together purely from update order.

The takeaway: an assembly-detection bridge tells you the *other mod is loaded*; it does not
tell you *who wrote a shared field last*. Where you write shared prefab/net/component data,
assume you are one of several writers, capture your own baseline, and offer a restore path.

### Uninstall and save-persistence asymmetry

Detection is a load-time concern; some writes outlive the mod that made them. If a mod mutates
data that the base game serialises into the savegame, uninstalling the mod does *not* undo the
change - the data is now part of the save.

- **Durable net writes survive removal.** Because Road Speed Adjuster's edits land on the
  game's own `CarLane`/`TrackLane` components (persisted with the network), simply removing the
  mod leaves every customised road stuck at its overridden speed. The mod exposes a settings
  action that queues `RequestClearAllCustomSpeeds`, which walks the tagged entities and restores
  each lane to its captured original before you uninstall
  ([`road-speed-adjuster` `repo/Systems/ClearCustomSpeedsSystem.cs#L41-L110`](../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/Systems/ClearCustomSpeedsSystem.cs), commit `e0c0c0b`).
  The correct removal order is *Clear All first, then uninstall* - not the reverse.
- **Top-ups bake into savegame resources.** Magic Mail adds directly into the `Resources`
  dynamic buffer on postal facilities (`AddResourceAmount(resources, Resource.OutgoingMail, ...)`),
  which is core game state serialised with the city
  ([`magic-mail` `repo/Systems/MagicMailSystem.cs#L299-L301`](../../vice-and-order-research/mods/dossiers/magic-mail/repo/Systems/MagicMailSystem.cs), commit `6fb3d2b`).
  The mail those writes injected persists in the save's resource buffers after the mod is gone.

If your mod mutates persisted state, treat "how does a player cleanly remove this?" as a
first-class design question: provide a restore/clear action and document the removal order.

### Storefront requirements as a no-code load guarantee

Not every dependency is enforced in code. The Paradox storefront lets a listing declare
*requirements* the launcher enforces before the mod can load, which can substitute for a
runtime check when the dependency is a hard prerequisite:

- **DLC-type requirements** gate on owned content - e.g. Bridge Expansion Pack: B&P lists
  "Bridges & Ports DLC" (plus a `CS2 1.5.*` version target) as required
  ([`bridge-expansion-pack-b-and-p` `index.md` requirements block](../../vice-and-order-research/mods/dossiers/bridge-expansion-pack-b-and-p/index.md)).
- **Version-pinned MOD requirements** gate on another mod at a specific version - FraggnAUT's
  render-prefab pack declares `mod_dependency` entries on the French Pack (modId 91930) and UK
  Pack (modId 92859), each pinned to a version, so subscribers must own those packs before the
  shared prefabs initialise
  ([`fraggnauts-render-prefab-pack-fr-and-uk-rp` `index.md` requirements block](../../vice-and-order-research/mods/dossiers/fraggnauts-render-prefab-pack-fr-and-uk-rp/index.md)).

Used well, this is a *declarative* load-order guarantee: the launcher refuses to load you until
the prerequisite is present, so `OnLoad` can assume it. It does not replace runtime validation
for *soft/optional* integrations (the launcher enforces nothing you did not declare), and the
exact enforcement behaviour is metadata-only in these dossiers - `Needs Verification (in-game)`
that the launcher blocks load rather than merely warning. Treat these storefront requirements
as documentation of intent that pairs with, not substitutes for, the defensive checks above;
see [Research Hygiene](./research-hygiene.md) for why storefront-declared behaviour is verified
in-game before it is trusted.

## Common pitfalls

- **Trusting metadata as a guarantee.** A declared dependency can still be absent at runtime;
  always validate.
- **A hard reference to an optional mod.** Referencing another mod's assembly at compile time
  makes *your* mod fail to load when it is absent. Use reflection for optional integrations.
- **Re-probing every frame.** Assembly scans are not free; cache the result.
- **Bundling the dependency.** Shipping a copy of a shared DLL (`<Private>true</Private>` or a
  committed binary) causes version conflicts. Reference, do not bundle.
- **Warning spam.** One warning per missing dependency, in `OnLoad` - not inside `OnUpdate`.
- **Assuming detection covers data conflicts.** An assembly-presence check proves another mod
  is loaded; it says nothing about who wrote a shared prefab/net/component field last. Where
  you mutate shared data, capture your own baseline and expect other writers.
- **Leaving persisted writes stranded on uninstall.** If you mutate savegame-serialised state
  (net components, resource buffers), removing the mod does not undo it. Ship a restore/clear
  action and document the removal order.

## See also

- [Module Layout](./module-layout.md) - referencing assemblies with `<Private>false</Private>`
  and the identifiers other mods match you by.
- [Lifecycle and Initialization](./mod-lifecycle.md) - where dependency detection and
  conditional system scheduling sit inside `OnLoad`.
- [Research Hygiene](./research-hygiene.md) - why storefront-declared requirements and other
  metadata claims are verified in-game before the handbook trusts them.
- [Technique Index](../technique-index.md) - coverage ledger for these techniques.
