---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: NameSystem custom names"
recipe: namesystem-custom-names
technique_family: "Q - NameSystem custom names"
diataxis: how-to
source_version: "~1.5.10f1 (advanced-road-naming@559e72c; date-pinned, static source only)"
last_reverified: "2026-07-04"
status: source-verified
canonical_mods:
  - advanced-road-naming@559e72cdb3180e3e71869643367e094a22eed988
  - custom-chirps@f018ac382e93e0b56cd7b974d1ce0b155d23d2b4
  - elections-rt-module@36c25afcda67c80ce75ba68d423fe8238499435a
technique_applicability: [ui, media]
Created: 2026-07-04
Updated: 2026-07-05
Owners:
  - codex
---

# NameSystem custom names

> Set (or clear) the display name shown for an entity by writing through the
> vanilla `NameSystem` - pure ECS, no Harmony, no reflection.

## Problem
You want the game's UI - the road label, the info panel title, a chirp sender row -
to show *your* text for an entity instead of the vanilla-generated name. The visible
name is owned by the game's `NameSystem`, not by a field you can poke; overwriting a
`PrefabRef` or a component won't change what the label renders. You need the one API
the game itself uses to attach a user/custom name to an entity.

## Solution
Resolve the managed `NameSystem` once in `OnCreate` via
`World.GetOrCreateSystemManaged<NameSystem>()`, then call
`_nameSystem.SetCustomName(entity, name)` to attach a custom name, or
`SetCustomName(entity, null)` to clear it back to the vanilla default. Read the
current name back with `TryGetCustomName` / `GetRenderedLabelName`. The write is
persistent (it adds a `CustomName` to the entity), so **sanitize the string first**
(a blank name must fall back, never write empty) and **make sure your system runs at
a defined phase relative to the systems that recompute names** - Advanced Road Naming
schedules itself around the vanilla `AggregateSystem` for exactly this reason.

## Steps & Code

### 1. Resolve `NameSystem` in `OnCreate`

Grab the managed system once and cache it; every name read/write goes through it.

```csharp
private NameSystem _nameSystem;
// ...
protected override void OnCreate()
{
    base.OnCreate();
    _nameSystem = World.GetOrCreateSystemManaged<NameSystem>();
    // ... other services / queries
}
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Systems/SegmentMetadataSystem.cs#L37,L70` (@559e72cdb3180e3e71869643367e094a22eed988)

### 2. Sanitize, then write the custom name

Never hand `SetCustomName` a null/blank string as the *intended* name - derive a safe
fallback first, then write. This is the core call: `SetCustomName(entity, safeName)`.

```csharp
private void SetSegmentDisplayName(Entity segment, string displayName)
{
    if (segment == Entity.Null || !_validation.IsValidRoadSegment(segment)) return;

    var safeDisplayName = string.IsNullOrWhiteSpace(displayName)
        ? $"Road Segment {segment.Index}" : displayName.Trim();

    // CS2 integration point: writes a custom name on the individual edge entity only.
    _nameSystem.SetCustomName(segment, safeDisplayName);
    EnsureRefreshTag<CustomName>(segment);
    EnsureRefreshTag<BatchesUpdated>(segment);   // refresh label/render batches
}
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Systems/SegmentMetadataSystem.cs#L3807-L3823` (@559e72cdb3180e3e71869643367e094a22eed988)

`EnsureRefreshTag<T>` just adds the tag component if it is missing, nudging the game to
re-render the label:

```csharp
private void EnsureRefreshTag<T>(Entity segment) where T : struct, IComponentData
{
    if (!EntityManager.HasComponent<T>(segment))
        EntityManager.AddComponent<T>(segment);
}
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Systems/SegmentMetadataSystem.cs#L3848-L3852` (@559e72cdb3180e3e71869643367e094a22eed988)

### 3. Read the current name back (echo check)

After a write, verify the game accepted it. `TryGetCustomName` returns the stored
custom string; `GetRenderedLabelName` returns whatever label is actually shown
(custom or vanilla-generated).

```csharp
private string GetCurrentAuthoritativeName(Entity nameEntity, string fallback)
{
    if (nameEntity != Entity.Null && EntityManager.Exists(nameEntity) && _nameSystem != null)
    {
        if (_nameSystem.TryGetCustomName(nameEntity, out var customName)
            && !string.IsNullOrWhiteSpace(customName))
            return customName.Trim();

        var renderedName = _nameSystem.GetRenderedLabelName(nameEntity);
        if (!string.IsNullOrWhiteSpace(renderedName)) return renderedName.Trim();
    }
    return string.IsNullOrWhiteSpace(fallback) ? $"Road Segment {nameEntity.Index}" : fallback.Trim();
}
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Systems/SegmentMetadataSystem.cs#L848-L861` (@559e72cdb3180e3e71869643367e094a22eed988)

### 4. Clear a custom name (reset to vanilla)

Passing `null` removes the custom name so the entity reverts to its vanilla-generated
label. ARN uses this to strip stray custom names off child edges of an aggregate:

```csharp
private void ClearChildEdgeCustomName(Entity nameEntity, Entity edge)
{
    if (edge == Entity.Null || edge == nameEntity || !EntityManager.Exists(edge)) return;

    if (_nameSystem.TryGetCustomName(edge, out _))
        _nameSystem.SetCustomName(edge, null);      // null -> back to vanilla name

    if (EntityManager.HasComponent<CustomName>(edge))
        EnsureRefreshTag<BatchesUpdated>(edge);
}
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Systems/SegmentMetadataSystem.cs#L1685-L1695` (@559e72cdb3180e3e71869643367e094a22eed988)

### 5. Schedule your system relative to the name-recompute system

Custom names on road edges interact with vanilla `AggregateSystem` (which recomputes
aggregate/street names). ARN registers its writer to run **after** `AggregateSystem`
in `ModificationEnd`, and its protection system **before** it - so its writes are not
clobbered by the vanilla recompute in the same frame:

```csharp
updateSystem.UpdateAt<SegmentMetadataSystem>(SystemUpdatePhase.Deserialize);
updateSystem.UpdateAt<SegmentMetadataSystem>(SystemUpdatePhase.Serialize);
updateSystem.UpdateAfter<SegmentMetadataSystem, AggregateSystem>(SystemUpdatePhase.ModificationEnd);
updateSystem.UpdateBefore<RoadAggregateProtectionSystem, AggregateSystem>(SystemUpdatePhase.ModificationEnd);
```
Source: `../../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Mod.cs#L54-L57` (@559e72cdb3180e3e71869643367e094a22eed988)

## Pitfalls & gotchas

- **Never write a blank name.** `SetCustomName(entity, "")`/whitespace would attach an
  empty label. Both ARN write paths coerce a blank to a fallback (`"Road Segment
  {index}"`) before writing and only pass a genuine `null` when the *intent* is to
  clear (SegmentMetadataSystem.cs#L1657, #L3815, #L1691). Decide up front: real name,
  or clear - do not let empty strings leak through.

- **`null` clears; the empty string does not.** `SetCustomName(entity, null)` is the
  reset-to-vanilla path (SegmentMetadataSystem.cs#L1691). Use `null` to clear, a
  sanitized non-empty string to set.

- **The write does not repaint by itself.** ARN adds refresh tags (`CustomName`,
  `BatchesUpdated`, and for aggregate owners `Updated`) after the write so the label
  re-renders (SegmentMetadataSystem.cs#L1659-L1661). Note ARN deliberately avoids
  adding `Updated` to individual `Aggregated` road edges because `AggregateSystem`
  treats `Updated` as a topology/name-group recompute signal (SegmentMetadataSystem.cs#L3821);
  it uses `BatchesUpdated` for a visual-only refresh instead. Which tags a given
  entity kind needs to repaint is `Needs Verification (in-game)`.

- **Timing vs the name-recompute system.** On networks, a plain `SetCustomName` can be
  overwritten in the same frame by `AggregateSystem`. ARN's answer is scheduling
  (step 5) plus a post-write echo check via `TryGetCustomName`/`GetRenderedLabelName`
  that logs a warning when the game did not echo the expected name
  (SegmentMetadataSystem.cs#L3825-L3828). If your label mysteriously reverts, this is
  the first thing to check.

- **Own the right entity.** ARN writes to the *individual edge* entity, not
  `Aggregated.m_Aggregate`, precisely because writing the aggregate would rename every
  edge in the street (SegmentMetadataSystem.cs#L3817-L3819). Pick the entity whose
  label you actually want to change.

- **Reflecting into `NameSystem` is fragile - keep it a *fallback*, not the primary
  path.** The read-side reflection chain below (Elections) only touches the private
  `GetCitizenName` / nested `Name` struct *after* the public `TryGetCustomName` misses,
  and every reflected step is wrapped in its own `try/catch` so a renamed member at a
  future game version degrades to `GetRenderedLabelName` instead of throwing
  (ElectionNameUtility.cs#L21-49). Prefer the public API (steps 2-4); reflect only when
  you need vanilla-generated detail the public surface does not expose.

- **DRIFT TRAP (advanced-road-naming).** The working tree is ahead of the pin. All line
  numbers above are read at pinned commit `559e72c` via `git show 559e72c:<path>`, not
  the checked-out `f31d043` "1.7 Update" tree. Re-verify against the pin, not HEAD.

## Variations

- **Build a name *value* for a UI binding instead of persisting it.** When you only
  need a display name inside a UI binding (not a stored `CustomName` on an entity),
  construct a `NameSystem.Name` with the `CustomName` factory and hand it to the
  binder. Custom Chirps does this to override a chirp's *sender* row - a different
  entity kind (`Game.Triggers.Chirp`) and a transient, per-render name that is never
  written back to the entity:

  ```csharp
  __instance.BindChirpLink(
      binder,
      chirp.m_Sender,
      NameSystem.Name.CustomName(payload.OverrideSenderName.ToString())
  );
  // ... binder.TypeEnd();
  return false; // we handled it
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/custom-chirps/repo/CustomChirps/UI/ChirperSenderPatch.cs#L37-L44` (@f018ac382e93e0b56cd7b974d1ce0b155d23d2b4)

  Note the trade-off: this is a Harmony prefix on `ChirperUISystem.BindChirpSender`
  and the name lives only for that binding pass - nothing persists on the entity, so
  there is no `AggregateSystem`-style clobber to schedule around. Use this when the
  name is presentation-only; use `SetCustomName` (steps 2-4) when the name must stick
  to the entity and survive saves.

- **Read a name back with a 3-tier reflection fallback (custom -> reflected -> label).**
  When you want the *rendered* name of an entity the public API cannot fully resolve
  (Elections needs a citizen's full first+last name for ballots), layer three reads and
  never throw. Try the public `TryGetCustomName` first; if it misses, invoke the private
  `NameSystem.GetCitizenName(entity)` via reflection and render its nested `Name` struct;
  if that fails too, fall back to `GetRenderedLabelName`:

  ```csharp
  public static string GetCitizenFullName(NameSystem nameSystem, EntityManager em,
                                          Entity entity, string fallback)
  {
      // 1. Public custom name, if one was explicitly set.
      try { if (nameSystem.TryGetCustomName(entity, out var custom)) return Sanitize(custom, fallback); }
      catch { }

      // 2. Private NameSystem.GetCitizenName(entity), rendered from its nested Name struct.
      try {
          EnsureReflection(nameSystem);
          object name = s_GetCitizenName?.Invoke(nameSystem, new object[] { entity });
          string fullName = RenderName(name);
          if (!string.IsNullOrWhiteSpace(fullName)) return Sanitize(fullName, fallback);
      } catch { }

      // 3. Last resort: whatever label the UI would show.
      try { return Sanitize(nameSystem.GetRenderedLabelName(entity), fallback); }
      catch { return fallback; }
  }
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/elections-rt-module/repo/Elections/Systems/ElectionNameUtility.cs#L16-L50` (@36c25afcda67c80ce75ba68d423fe8238499435a)

  The reflection is resolved once and cached: `EnsureReflection` grabs the non-public
  `GetCitizenName` method plus the nested `Name` type and its private `m_NameType` /
  `m_NameID` / `m_NameArgs` fields, and `RenderName` interprets the struct (type `0` =
  literal id, `1` = localized id, `2` = format string with `FIRST_NAME`/`LAST_NAME`
  args) - this is the READ complement to the WRITE path in steps 2-4.
  Source: `../../../vice-and-order-research/mods/dossiers/elections-rt-module/repo/Elections/Systems/ElectionNameUtility.cs#L52-L115` (@36c25afcda67c80ce75ba68d423fe8238499435a)

- **Normalize and format the string *before* you hand it to `SetCustomName`.** ARN never
  writes raw user input. Route codes are validated against a Unicode-aware regex (letters,
  numbers, hyphens; first char alphanumeric; max 16 chars), trimmed, and rejected with a
  human-readable error before they can reach the entity:

  ```csharp
  private static readonly Regex RouteCodePattern = new Regex(
      @"^[\p{L}\p{N}][\p{L}\p{N}\-]{0,15}$",
      RegexOptions.Compiled | RegexOptions.CultureInvariant);

  public bool TryNormalize(string value, out string routeCode, out string error)
  {
      routeCode = string.IsNullOrWhiteSpace(value) ? string.Empty : value.Trim();
      if (string.IsNullOrWhiteSpace(routeCode)) { error = "Enter a route code such as A1, M2, HWY7..."; return false; }
      if (!RouteCodePattern.IsMatch(routeCode)) { error = "Route codes may use letters, numbers and hyphens..."; return false; }
      error = null; return true;
  }
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Services/RouteCodeService.cs#L11-L36` (@559e72cdb3180e3e71869643367e094a22eed988)

  The final label is then *composed* from parts rather than typed by hand:
  `SegmentDisplayNameResolver.Resolve` picks a base name (custom road name -> snapshot ->
  a `"Unnamed Road Segment"` safe default), de-dups and optionally sorts the route codes,
  then joins them either before or after the base name per a `RouteNumberPlacement`
  setting. Build the string this way, then feed the result to `SetCustomName`:

  ```csharp
  var routeNumberDisplay = string.Join(settings.RouteNumberSeparator, routeNumbers);
  return metadata.RouteNumberPlacement == RouteNumberPlacement.BeforeBaseName
      ? routeNumberDisplay + settings.BaseRouteSeparator + baseName
      : baseName + settings.BaseRouteSeparator + routeNumberDisplay;
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/advanced-road-naming/repo/Services/SegmentDisplayNameResolver.cs#L9-L31` (@559e72cdb3180e3e71869643367e094a22eed988)

## See also
- Related recipes: [render-pipeline overlay](render-pipeline-overlay.md) (another
  UI/media technique from the same road-naming lineage).
- Reference: [ECS fundamentals](../../explanation/ecs-fundamentals.md) (managed systems,
  `GetOrCreateSystemManaged`, update phases, tag components).
- Case study demonstrating it: [advanced-road-naming](../../case-studies/advanced-road-naming.md).

## Sources
- Canonical mods (dossier + repo):
  - `advanced-road-naming` @559e72cdb3180e3e71869643367e094a22eed988 - `repo/Systems/SegmentMetadataSystem.cs`, `repo/Mod.cs`, `repo/Services/RouteCodeService.cs`, `repo/Services/SegmentDisplayNameResolver.cs`
  - `custom-chirps` @f018ac382e93e0b56cd7b974d1ce0b155d23d2b4 - `repo/CustomChirps/UI/ChirperSenderPatch.cs`
  - `elections-rt-module` @36c25afcda67c80ce75ba68d423fe8238499435a - `repo/Elections/Systems/ElectionNameUtility.cs`
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding
