---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: Prefab-field override"
recipe: prefab-field-override
technique_family: "A - Prefab-field override"
diataxis: how-to
source_version: "~1.5.2f1 (outside-traffic-adjuster@42afd29; date-pinned, static source only)"
last_reverified: "2026-07-04"
canonical_mods:
  - outside-traffic-adjuster@42afd29638267c9dc116b040515ceaf41eb9a9e1
  - magic-mail@6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47
  - smooth-left-hand-traffic@37b2850b10ced1cd17457c25778409e57e00a03e
technique_applicability: [core, economy]
status: source-verified
Created: 2026-07-01
Updated: 2026-07-01
Owners:
  - codex
---

# Prefab-field override

> Change a game balance value (spawn rate, capacity, behavior flag) by resolving a
> prefab, editing one field on its component, and writing the component back through
> `PrefabSystem` - no Harmony, no reflection.

## Problem
You want to tune a numeric or enum field that lives on a **prefab** (the shared
template behind every instance of a building, vehicle, or marker) - for example the
spawn rate of outside connections, the mail capacity of post vans, or the "invert
when" flag on road sub-nets. These values are authored into prefab component data at
load time; you cannot reach them from settings and you do not want to patch vanilla
methods. This recipe is the cleanest, lowest-risk way to change them.

## Solution
Resolve the prefab through the managed `PrefabSystem` API, read its ECS component
data, mutate the one field you care about, and write the component back. The whole
technique is three calls: `TryGetPrefab` (find the template) -> `TryGetComponentData`
(read the current struct) -> `AddComponentData` (commit the mutated struct). Because
`AddComponentData` on an existing component overwrites it, the edit takes effect for
**every instance** of that prefab. Guard both `TryGet` calls with `&&` so a rename or
missing prefab is a safe no-op, and derive your new value from an **immutable
baseline** (a hard-coded vanilla constant or the prefab's authoring `PrefabBase`
value) so re-applying the config never stacks.

## Steps & Code

### 1. Resolve `PrefabSystem` in `OnCreate`

```csharp
// Grab the managed PrefabSystem once; you will look prefabs up through it.
m_PrefabSystem = World.DefaultGameObjectInjectionWorld
    .GetOrCreateSystemManaged<PrefabSystem>();
```
Source: `../../../vice-and-order-research/mods/dossiers/outside-traffic-adjuster/repo/SpawnRateEditorSystem.cs#L20` (@42afd29638267c9dc116b040515ceaf41eb9a9e1)

### 2. Look the prefab up by `PrefabID` (type name + display name)

A `PrefabID` is `(prefabType, prefabName)`. Chain `TryGetPrefab` and (if you need the
entity) `TryGetEntity` so a miss falls through safely:

```csharp
private bool TryGetPrefab(string prefabType, string prefabName,
    out PrefabBase prefabBase, out Entity entity)
{
    prefabBase = null;
    entity = default;
    PrefabID prefabID = new PrefabID(prefabType, prefabName);
    return m_PrefabSystem.TryGetPrefab(prefabID, out prefabBase)
        && m_PrefabSystem.TryGetEntity(prefabBase, out entity);
}
```
Source: `../../../vice-and-order-research/mods/dossiers/outside-traffic-adjuster/repo/SpawnRateEditorSystem.cs#L100-L106` (@42afd29638267c9dc116b040515ceaf41eb9a9e1)

### 3. Read the component, mutate one field, write it back

This is the canonical `TryGet -> component -> AddComponentData` sequence. Note the
new value is `0.3f * roadMultiplier` - a **fixed vanilla baseline** (0.3) times the
user's multiplier, not the live field, so it is idempotent:

```csharp
if (TryGetPrefab(nameof(MarkerObjectPrefab), "Road Outside Connection - Oneway",
        out PrefabBase prefabBase, out Entity entity)
    && m_PrefabSystem.TryGetComponentData<TrafficSpawnerData>(prefabBase, out TrafficSpawnerData comp))
{
    comp.m_SpawnRate = 0.3f * roadMultiplier;   // baseline * multiplier, never live * multiplier
    m_PrefabSystem.AddComponentData(prefabBase, comp);
}
```
Source: `../../../vice-and-order-research/mods/dossiers/outside-traffic-adjuster/repo/SpawnRateEditorSystem.cs#L110-L114` (@42afd29638267c9dc116b040515ceaf41eb9a9e1)

The same mod repeats this block for six prefabs across four traffic modes with the
per-mode vanilla baselines Road `0.3`, Train `0.003`, Ship `0.002`, Plane `0.005`
(`SpawnRateEditorSystem.cs#L108-L152`).

### 4. Apply on load and on settings-applied; neutralize outside the game

Do the work only at well-defined moments, not every frame. Outside Traffic Adjuster
subscribes to `onSettingsApplied` in `OnCreate` and self-disables (`Enabled = false`),
so the system idles until settings change or a city loads:

```csharp
Mod.INSTANCE.m_Setting.onSettingsApplied += settings =>
{
    if (settings.GetType() == typeof(Setting))
        this.UpdateAndApplyMultipliers((Setting)settings);
};
this.Enabled = false;
```
Source: `../../../vice-and-order-research/mods/dossiers/outside-traffic-adjuster/repo/SpawnRateEditorSystem.cs#L22-L29` (@42afd29638267c9dc116b040515ceaf41eb9a9e1)

```csharp
protected override void OnGameLoadingComplete(Purpose purpose, GameMode mode)
{
    base.OnGameLoadingComplete(purpose, mode);
    if (mode == GameMode.Game) { InitializeMultipliers(Mod.INSTANCE.m_Setting); ApplyAllSpawnRates(); }
    else                       { ResetMultipliers();                            ApplyAllSpawnRates(); }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/outside-traffic-adjuster/repo/SpawnRateEditorSystem.cs#L37-L50` (@42afd29638267c9dc116b040515ceaf41eb9a9e1)

Applying `else -> reset to x1.0` matters: it stops edited prefab data from leaking
into the editor or main menu, where `GameMode.Game` is false.

## Pitfalls & gotchas

- **Silent prefab-miss (no log, no effect).** When `TryGetPrefab` fails - because a
  patch/DLC renamed the prefab or it is not loaded yet - the guarded `if` simply
  falls through. There is no error, no warning, and no fallback in the canonical
  source; the tuning just silently stops working. Outside Traffic Adjuster looks
  prefabs up by literal display name (`"Road Outside Connection - Oneway"`, etc.),
  which is exactly the fragile point
  (`../../../vice-and-order-research/mods/dossiers/outside-traffic-adjuster/repo/SpawnRateEditorSystem.cs#L110`).
  If you need to know when a lookup misses, add your own log on the `else` branch.

- **Stacking if you scale the live value.** `AddComponentData` overwrites the
  component, so if you read the current field and multiply *that*, every re-apply
  compounds (x1.5 -> x2.25 -> ...). The canonical mods avoid this two ways: Outside
  Traffic Adjuster multiplies a **hard-coded vanilla constant** (`0.3f * mult`), and
  Magic Mail reads an **immutable authoring baseline off `PrefabBase`** before
  scaling. See the one-shot non-stacking variant for the baseline-from-`PrefabBase`
  pattern in full: [one-shot prefab multiplier](one-shot-prefab-multiplier.md).

- **Hard-coded baselines can drift.** The `0.3/0.003/0.002/0.005` literals are safe
  from stacking but go stale if a future patch changes the vanilla authoring value;
  the mod would then scale from a wrong base. Reading the authoring baseline off the
  prefab (Magic Mail) is drift-resistant; hard-coded literals are simpler but need
  re-verification each game version.

- **This edits the PREFAB, so it affects ALL instances.** You are mutating the shared
  template, not a placed building or vehicle. Every current and future instance of
  that prefab picks up the change. That is the point for balance tuning, but it means
  you cannot target one placed instance with this technique.

- **Apply at the right time.** Prefab component data is not ready before prefabs load.
  Do the work in `OnGameLoadingComplete` and/or an `onSettingsApplied` handler, not in
  `OnCreate` or an unconditional `OnUpdate`. Neutralize (reset to baseline) when
  `mode != GameMode.Game` so menu/editor prefabs are not left edited.

- Anything above not shown in the cited source is marked `Needs Verification
  (in-game)`. In particular: the exact in-game behavior when two mods override the
  same prefab field (last-writer-wins is expected but not proven from this source) is
  **Needs Verification (in-game)**.

## Variations

- **Read baseline from `PrefabBase` authoring instead of a literal (non-stacking, no
  drift).** Magic Mail queries post-van / post-facility prefab entities, pulls the
  vanilla baseline off the authoring component via `PrefabBase.TryGet`, then scales:

  ```csharp
  if (!m_PrefabSystem.TryGetPrefab(prefabEntity, out PrefabBase prefabBase)) return false;
  if (!prefabBase.TryGet(out Game.Prefabs.PostVan postVan))                  return false;
  baseMailCapacity = postVan.m_MailCapacity;   // immutable vanilla baseline
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/magic-mail/repo/Systems/MailCapacitySystem.cs#L212-L228` (@6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47)

  It then writes the scaled value onto the ECS component via `RefRW` inside a query
  (`vanData.m_MailCapacity = newMailCapacity`,
  `MailCapacitySystem.cs#L146-L150`) rather than `AddComponentData`, and floors
  positive results at 1 so a facility never drops to broken-zero
  (`ScalePercentMin1`, `MailCapacitySystem.cs#L255-L263`). Full write-up:
  [one-shot prefab multiplier](one-shot-prefab-multiplier.md).

- **Query-driven bulk override instead of named lookup.** When you want *all* prefabs
  carrying a component (not a fixed list of names), build an `EntityQuery` over
  `PrefabData` + the target component and iterate with `SystemAPI.Query<RefRW<...>>()`
  (Magic Mail, `MailCapacitySystem.cs#L47-L56, #L134-L137`). This sidesteps the
  name-coupling fragility but touches every matching prefab.

- **Override an enum/flag field, not a number.** Smooth Left-Hand Traffic flips the
  `ObjectSubNets.m_InvertWhen` enum on road prefabs. It uses the same read-mutate
  shape but commits with `PrefabSystem.UpdatePrefab(prefab)` (managed-component path)
  instead of `AddComponentData`:

  ```csharp
  if (prefab.TryGet(out ObjectSubNets subNets) && subNets.m_InvertWhen != invertMode)
  {
      subNets.m_InvertWhen = invertMode;
      prefabSystem.UpdatePrefab(prefab);
      return 1;
  }
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/smooth-left-hand-traffic/repo/SmoothLHT/Services/PrefabInvertService.cs#L68-L78` (@37b2850b10ced1cd17457c25778409e57e00a03e)

  Note the `!= invertMode` guard: it skips prefabs already at the target value, so the
  pass is idempotent and only counts real changes.

## See also
- Related recipes: [one-shot prefab multiplier](one-shot-prefab-multiplier.md) (the
  non-stacking, baseline-from-`PrefabBase` variant, family J).
- Reference: [technique index](../../technique-index.md) (family A coverage ledger).
- Case studies demonstrating it: [outside-traffic-adjuster](../../case-studies/outside-traffic-adjuster.md),
  [magic-mail](../../case-studies/magic-mail.md).

## Sources
- Canonical mods (dossier + repo):
  - `outside-traffic-adjuster` @42afd29638267c9dc116b040515ceaf41eb9a9e1 - `repo/SpawnRateEditorSystem.cs`
  - `magic-mail` @6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47 - `repo/Systems/MailCapacitySystem.cs`
  - `smooth-left-hand-traffic` @37b2850b10ced1cd17457c25778409e57e00a03e - `repo/SmoothLHT/Services/PrefabInvertService.cs`
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding
