---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: Network composition bitmasks"
recipe: network-composition-bitmasks
technique_family: "R - Network composition bitmasks"
diataxis: how-to
source_version: "~1.5.9 (anarchy@a6311e8; date-pinned, static source only)"
last_reverified: "2026-07-04"
status: source-verified
canonical_mods:
  - anarchy@a6311e898d20a775368668b234aaa32f06e3e1eb
technique_applicability: [infrastructure]
Created: 2026-07-04
Updated: 2026-07-04
Owners:
  - codex
---

# Network composition bitmasks

> Drive road/segment placement - forced ground/elevated/tunnel, medians, walls, bike
> lanes - by testing and combining bit flags, then translating your mod's own toggle
> enum into the vanilla `Game.Net.CompositionFlags` the game actually bakes onto an
> `Upgraded` edge.

## Problem
You want a tool that forces or adds a network *composition* option - keep a road on the
ground, force it into a tunnel, add a wide median, put a retaining wall on the left
side - the way Anarchy's Network Anarchy tool does. Two things make this fiddly. First,
the state is a **bitmask**: several options are live at once and you must test/set
individual bits without disturbing the others. Second, your UI toggle state is *not*
the value the game stores. The game bakes `Game.Net.CompositionFlags` (`m_General`,
`m_Left`, `m_Right`) onto the edge's `Upgraded` component; your tool holds a separate
mod-owned enum. You need both layers and a clean translation between them.

## Solution
Keep two enum layers straight and translate deliberately between them:

1. **Mod-owned UI enums** - `NetworkAnarchyUISystem.Composition`,
   `NetworkAnarchyUISystem.SideUpgrades` - are plain `enum`s with power-of-two values
   used as bitmasks (`&` to test, `|` to combine, `&= ~` to clear). These hold what the
   user toggled.
2. **Vanilla `Game.Net.CompositionFlags`** - `CompositionFlags.General` and
   `CompositionFlags.Side` - are what the game reads off the `Upgraded` component.

You test the mod enum with the standard `(mask & Flag) == Flag` idiom to decide
behaviour, and you translate mod-enum -> vanilla-flags through a `Dictionary` lookup
before writing `Upgraded`. Some mod options (`Ground`, `ConstantSlope`) never become a
`CompositionFlags` at all - they are realised by editing course elevation instead.

## Steps & Code

### 1. Define the UI toggle enum as power-of-two bit values

Anarchy's composition options are a plain `enum` (note: **no `[Flags]` attribute**) with
`1,2,4,8,...` values, so they combine as a bitmask:

```csharp
public enum Composition
{
    None,
    Ground = 1,
    Elevated = 2,
    Tunnel = 4,
    ConstantSlope = 8,
    WideMedian = 16,
    Trees = 32,
    GrassStrip = 64,
    Lighting = 128,
    ExpandedElevationRange = 256,
}
```
Source: `../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/Systems/NetworkAnarchy/NetworkAnarchyUISystem.cs#L101-L152` (@a6311e898d20a775368668b234aaa32f06e3e1eb)

`SideUpgrades` (`Quay=1`, `RetainingWall=2`, `Trees=4`, `GrassStrip=8`, `WideSidewalk=16`,
`SoundBarrier=32`, `BikeLane=64`, `BikeRestriction=128`) is the same shape for
per-side options
(`NetworkAnarchyUISystem.cs#L50-L96`).

### 2. Expose the live mask, pre-filtered by what the current net allows

The public getter ANDs the raw toggle value with a separate "shown/allowed" mask, so
options the active prefab cannot support are stripped before any consumer reads them:

```csharp
public Composition NetworkComposition
{
    get { return m_Composition.Value & m_ShowComposition.Value; }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/Systems/NetworkAnarchy/NetworkAnarchyUISystem.cs#L197-L199` (@a6311e898d20a775368668b234aaa32f06e3e1eb)

### 3. Test bits with `(mask & Flag) == Flag`

The consumer system reads that mask and early-outs unless at least one elevation-
affecting bit is set. This is the canonical bit-test idiom - `& Flag` then compare to
`Flag`, never to non-zero:

```csharp
if ((m_UISystem.NetworkComposition & NetworkAnarchyUISystem.Composition.ConstantSlope) != NetworkAnarchyUISystem.Composition.ConstantSlope
    && (m_UISystem.NetworkComposition & NetworkAnarchyUISystem.Composition.Ground) != NetworkAnarchyUISystem.Composition.Ground
    && (m_UISystem.NetworkComposition & NetworkAnarchyUISystem.Composition.Tunnel) != NetworkAnarchyUISystem.Composition.Tunnel
    && (m_UISystem.NetworkComposition & NetworkAnarchyUISystem.Composition.Elevated) != NetworkAnarchyUISystem.Composition.Elevated)
{
    return;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/Systems/NetworkAnarchy/NetworkDefinitionSystem.cs#L69-L75` (@a6311e898d20a775368668b234aaa32f06e3e1eb)

### 4. Realise elevation-type bits by editing the course, not by setting a flag

`Ground`, `Tunnel`, and `Elevated` are branch selectors that rewrite `NetCourse`
elevation - they never become a `CompositionFlags`. Each branch is a single bit test:

```csharp
if ((m_UISystem.NetworkComposition & NetworkAnarchyUISystem.Composition.Ground) == NetworkAnarchyUISystem.Composition.Ground)
{
    netCourse.m_StartPosition.m_Elevation = new float2(ForceGroundElevation, ForceGroundElevation);
    netCourse.m_EndPosition.m_Elevation   = new float2(ForceGroundElevation, ForceGroundElevation);
    netCourse.m_Elevation                 = new float2(ForceGroundElevation, ForceGroundElevation);
}
else if ((m_UISystem.NetworkComposition & NetworkAnarchyUISystem.Composition.Tunnel) == NetworkAnarchyUISystem.Composition.Tunnel)
{
    netCourse.m_Elevation.x = Mathf.Min(netCourse.m_Elevation.x, TunnelThreshold);
    netCourse.m_Elevation.y = Mathf.Min(netCourse.m_Elevation.y, TunnelThreshold);
    // ... start/end positions clamped to TunnelThreshold too
}
else if ((m_UISystem.NetworkComposition & NetworkAnarchyUISystem.Composition.Elevated) == NetworkAnarchyUISystem.Composition.Elevated)
{
    netCourse.m_Elevation.x = Mathf.Max(netCourse.m_Elevation.x, ElevatedThreshold);
    // ... clamped up to ElevatedThreshold
}
```
Source: `../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/Systems/NetworkAnarchy/NetworkDefinitionSystem.cs#L269-L292` (@a6311e898d20a775368668b234aaa32f06e3e1eb)

### 5. Map mod-enum combinations to vanilla `CompositionFlags` via a dictionary

This is the crux: a `Dictionary` from the mod's `Composition` (and separately
`SideUpgrades`) to vanilla `CompositionFlags.General` (and `.Side`). Composite keys are
enumerated explicitly because a plain `Dictionary` keys on the exact combined value:

```csharp
private readonly Dictionary<NetworkAnarchyUISystem.Composition, CompositionFlags.General> GeneralCompositionLookup = new()
{
    { NetworkAnarchyUISystem.Composition.None, 0 },
    { NetworkAnarchyUISystem.Composition.Elevated, CompositionFlags.General.Elevated },
    { NetworkAnarchyUISystem.Composition.Tunnel, CompositionFlags.General.Tunnel },
    { NetworkAnarchyUISystem.Composition.WideMedian, CompositionFlags.General.WideMedian },
    { NetworkAnarchyUISystem.Composition.GrassStrip, CompositionFlags.General.PrimaryMiddleBeautification },
    { NetworkAnarchyUISystem.Composition.Trees, CompositionFlags.General.SecondaryMiddleBeautification },
    { NetworkAnarchyUISystem.Composition.Trees | NetworkAnarchyUISystem.Composition.GrassStrip,
      CompositionFlags.General.SecondaryMiddleBeautification | CompositionFlags.General.PrimaryMiddleBeautification },
    { NetworkAnarchyUISystem.Composition.Trees | NetworkAnarchyUISystem.Composition.WideMedian,
      CompositionFlags.General.SecondaryMiddleBeautification | CompositionFlags.General.WideMedian },
};
```
Source: `../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/Systems/NetworkAnarchy/TempNetworkSystem.cs#L76-L88` (@a6311e898d20a775368668b234aaa32f06e3e1eb)

The side-upgrade table (`SideUpgradeLookup`) is the same idea for `CompositionFlags.Side`,
e.g. `Quay -> Side.Raised`, `RetainingWall -> Side.Lowered`, `BikeRestriction -> Side.ForbidSecondary`,
and every legal *combination* is spelled out as its own key
(`TempNetworkSystem.cs#L46-L74`).

### 6. Strip non-mappable bits before the lookup, then read it

The general-flags translator clears the two elevation-only bits (`ConstantSlope`,
`Ground` - handled in step 4) so the remaining value is a valid dictionary key:

```csharp
private CompositionFlags.General GetCompositionGeneralFlags(NetworkAnarchyUISystem.Composition composition)
{
    composition &= ~NetworkAnarchyUISystem.Composition.ConstantSlope;
    composition &= ~NetworkAnarchyUISystem.Composition.Ground;

    if (GeneralCompositionLookup.ContainsKey(composition))
    {
        return GeneralCompositionLookup[composition];
    }

    return 0;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/Systems/NetworkAnarchy/TempNetworkSystem.cs#L613-L624` (@a6311e898d20a775368668b234aaa32f06e3e1eb)

### 7. Compose the flag struct and write it onto `Upgraded`

Assemble `CompositionFlags` from the three translated pieces (general via the helper,
each side via the side table with `|=`), then commit it as the edge's `Upgraded`
component:

```csharp
compositionFlags.m_General = GetCompositionGeneralFlags(effectiveComposition);   // #L466
compositionFlags.m_Left  |= SideUpgradeLookup[effectiveLeftUpgrades];            // #L515
// ... m_Right the same way

Game.Net.Upgraded upgrades = new Game.Net.Upgraded()
{
    m_Flags = compositionFlags,
};
// ... AddComponent<Upgraded> if missing
EntityManager.SetComponentData(entity, upgrades);
```
Source: `../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/Systems/NetworkAnarchy/TempNetworkSystem.cs#L466-L604` (@a6311e898d20a775368668b234aaa32f06e3e1eb)

## Pitfalls & gotchas

- **Two enum layers, not one.** The mod's `NetworkAnarchyUISystem.Composition` /
  `SideUpgrades` are private UI state; the game only understands
  `Game.Net.CompositionFlags.General` / `.Side`. Never write a mod-enum value into
  `Upgraded.m_Flags` - it will not match any vanilla flag. Always translate through the
  lookup tables (`TempNetworkSystem.cs#L46-L88`).

- **No `[Flags]` attribute - it is a bitmask only by convention.** These enums use
  power-of-two values and are manipulated with `& | ~`, but they carry no `[Flags]`
  attribute (`NetworkAnarchyUISystem.cs#L101-L152`). That is legal and works, but tools
  that pretty-print via `[Flags]` (e.g. `ToString`) will not decompose combined values,
  and nothing stops you assigning a non-power-of-two literal. Treat the bitmask
  discipline as your responsibility.

- **Dictionary keys on the EXACT combined value.** `GeneralCompositionLookup` /
  `SideUpgradeLookup` are ordinary `Dictionary`s keyed on the whole combined enum value,
  so a combination that is not spelled out as its own entry simply misses the lookup.
  The general translator returns `0` on a miss (`TempNetworkSystem.cs#L623`), and the
  side lookups are guarded by `ContainsKey` (`TempNetworkSystem.cs#L504`) - so an
  un-enumerated combination silently produces *no* upgrade rather than an error. That is
  exactly why every legal pairing (e.g. `BikeRestriction | Quay`,
  `Trees | GrassStrip`) is written out by hand in those tables.

- **`Ground` and `ConstantSlope` are not composition flags.** They never map to a
  `CompositionFlags`; `GetCompositionGeneralFlags` deletes them before the lookup
  (`TempNetworkSystem.cs#L615-L616`). `Ground` forces course elevation to zero and
  `ConstantSlope` drives slope interpolation - both live in
  `NetworkDefinitionSystem`, not in `Upgraded`. Mixing these two "layers" up (expecting
  `Ground` to appear as a baked flag) is the classic mistake.

- **Test bits against the flag, not against zero.** The source always writes
  `(mask & Flag) == Flag` (`NetworkDefinitionSystem.cs#L69-L72`), which is correct even
  when `Flag` is itself multi-bit. `(mask & Flag) != 0` would spuriously succeed on a
  partial overlap; do not shortcut it.

- **`NetworkComposition` is pre-masked.** Reading `m_Composition.Value` directly is not
  the same as reading `NetworkComposition` - the getter ANDs with `m_ShowComposition`
  (`NetworkAnarchyUISystem.cs#L199`) so unsupported options are dropped for the active
  net. Consume the property, not the raw binding, or you will act on a toggle the
  current prefab cannot honour.

- **The precise in-game placement outcome of any given flag combination** (how the game
  re-composes the edge geometry once `Upgraded` is written) is `Needs Verification
  (in-game)` - the mapping is source-visible, the resulting mesh/behaviour is not.

## Variations

- **Clearing a bit / mutual exclusion.** To make options exclusive, clear the
  conflicting bits with `&= ~`. Anarchy does this in the UI layer - selecting a ground/
  elevated/tunnel option clears the other two before applying the new one
  (`NetworkAnarchyUISystem.cs#L472-L474`), and elevated placement clears any left/right
  Raised/Lowered side flags on the baked struct so a road is not both elevated and
  walled (`TempNetworkSystem.cs#L588-L591`).

- **Side flags vs general flags.** `CompositionFlags` has three fields: `m_General`
  (whole-segment: Elevated/Tunnel/WideMedian/median beautification) and `m_Left`/
  `m_Right` (per-side: Raised/Lowered walls, sidewalks, sound barrier, secondary lane,
  `ForbidSecondary`). Use `GeneralCompositionLookup` for the first and
  `SideUpgradeLookup` for the last two; they are separate tables with separate key
  enums (`TempNetworkSystem.cs#L46-L88`).

- **Reading existing flags back into UI state.** To pre-populate your tool from an edge
  the user is replacing, do the inverse test: read `Upgraded.m_Flags` and OR the
  matching mod-enum bit into your effective state, e.g. `m_Left & Side.Raised` ->
  `SideUpgrades.Quay`, `m_General & General.Elevated` -> `Composition.Elevated`
  (`TempNetworkSystem.cs#L277-L305`).

## See also
- Reference: [ECS fundamentals](../../explanation/ecs-fundamentals.md) (components,
  queries, and command buffers behind the `Upgraded` write).
- Related recipes: [tool drag-select](tool-drag-select.md) (selecting the segments a
  composition change is applied to).
- Case studies demonstrating it: [anarchy](../../case-studies/anarchy.md).

## Sources
- Canonical mods (dossier + repo):
  - `anarchy` @a6311e898d20a775368668b234aaa32f06e3e1eb -
    `repo/Anarchy/Systems/NetworkAnarchy/NetworkAnarchyUISystem.cs`,
    `repo/Anarchy/Systems/NetworkAnarchy/NetworkDefinitionSystem.cs`,
    `repo/Anarchy/Systems/NetworkAnarchy/TempNetworkSystem.cs`
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding
