---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: Chirper feed injection (custom chirps + feed ownership)"
recipe: chirper-feed-injection
technique_family: "AQ - Chirper feed injection (custom chirps + feed ownership)"
diataxis: how-to
source_version: "~1.6.0f1 (custom-chirps@f018ac3; date-pinned, static source only)"
last_reverified: "2026-07-05"
status: source-verified
canonical_mods:
  - custom-chirps@f018ac382e93e0b56cd7b974d1ce0b155d23d2b4
technique_applicability: [ui, media]
Created: 2026-07-05
Updated: 2026-07-05
Owners:
  - codex
---

# Chirper feed injection (custom chirps + feed ownership)

> Publish your own messages into the in-game Chirper feed - with a custom sender,
> real entity links, runtime-localized text, and an optional portrait image - and
> own the publish/panel pipeline so every decision is a single `StartsWith` check.

## Problem
You want a mod to post messages to Chirper (the social-media ticker) that read like
first-class game chirps: a chosen department account, clickable `{LINK_n}` targets, a
large card with an image, and text you compose at runtime. Two things fight you.
First, chirp text is served from the localization dictionary keyed by a string ID, and
the vanilla path to add strings is `LocalizationManager.ReloadActiveLocale()`, which
stutters the whole UI. Second, once you enqueue a chirp you do not get the resulting
entity back - the game spawns it a frame or more later on a job - so you cannot bind
"this queued payload" to "that spawned chirp", and you cannot cleanly tell your chirps
apart from vanilla ones in the publish and panel code.

## Solution
Tag ownership into the **message ID**: prefix `ChirperUISystem.GetMessageID` so a chirp
you own returns a `customchirps:...` key, and now every publish/panel branch is one
`key.StartsWith("customchirps")` test. Bind payload to entity **deterministically**:
stamp a throwaway marker entity with a unique token, pass that marker in as the chirp's
`m_Target`, and when the game later spawns the chirp your `GetMessageID` prefix finds the
marker in the chirp's link buffer, pops the payload by token under a lock, and swaps the
marker link for the real target(s) - preserving `{LINK_1..3}` order. Serve the text with a
Harmony prefix on `LocalizationDictionary.TryGetValue` at `Priority.First` that answers
`customchirps:` keys from an in-memory dictionary, so no locale reload ever happens. Images
ride an async C#->JS bus: register the source as a `img_<guid>` token, publish the token
list as a UI binding, and let the JS resolve token to URL (showing a pending state until
the binding arrives).

## Steps & Code

### 1. Tag every owned chirp by prefixing `GetMessageID`

`ChirperUISystem.GetMessageID(Entity chirp)` is the single point the UI asks "what string
should this chirp show". Patch it. On a chirp you own, return a `customchirps:` key and
skip vanilla; otherwise return `true` to fall through untouched.

```csharp
[HarmonyPatch(typeof(Game.UI.InGame.ChirperUISystem), nameof(Game.UI.InGame.ChirperUISystem.GetMessageID))]
public static class ChirperMessagePatch
{
    public static bool Prefix(ref string __result, Entity chirp)
    {
        var world = World.DefaultGameObjectInjectionWorld;
        if (world == null || chirp == Entity.Null) return true;
        var em = world.EntityManager;
        if (!em.Exists(chirp)) return true;

        if (RuntimeChirpTextBus.TryConsumeByMarker(em, chirp, out var payload))
        {
            em.AddComponentData(chirp, new ModChirpText { Key = payload.Key });
            __result = payload.Key.ToString();
            return false; // skip vanilla GetMessageID
        }
        return true; // fall back to vanilla
    }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/custom-chirps/repo/CustomChirps/UI/ChirperMessagePatch.cs#L9-L68` (@f018ac382e93e0b56cd7b974d1ce0b155d23d2b4)

Because the key resolution runs inside `GetMessageID`, it also does the marker->real-target
swap and (in the full source) forces the sender and stamps `OverrideSender` at the same time
(`ChirperMessagePatch.cs#L27-L54`). A `Postfix` restores the key from a per-instance
attachment if a later pass blanks `__result` (`ChirperMessagePatch.cs#L72-L85`).

### 2. Compose the key + register text WITHOUT a locale reload

Build a collision-resistant `customchirps:...:<guid>` key, then register the text in a
plain in-memory dictionary. The prefix (`customchirps:`, `customchirps:large:`, or
`customchirps:portraitimg:<token>:`) is what steps 1, 5, and 6 branch on.

```csharp
var keyPrefix = hasPortraitImage
    ? $"customchirps:portraitimg:{imageKey}:"
    : req.DisplayMode == ChirpDisplayMode.Large
        ? "customchirps:large:"
        : "customchirps:";
var key = $"{keyPrefix}{Guid.NewGuid():N}";
var value = string.IsNullOrWhiteSpace(finalText) ? "(empty)" : finalText;

// The UI localization lookup patch resolves these keys directly,
// avoiding LocalizationManager.ReloadActiveLocale() and the stutter it can cause.
RuntimeChirpLocalization.Add(key, value);
```
Source: `../../../vice-and-order-research/mods/dossiers/custom-chirps/repo/CustomChirps/Systems/CustomChirpApiSystem.cs#L323-L333` (@f018ac382e93e0b56cd7b974d1ce0b155d23d2b4)

`RuntimeChirpLocalization` is a locked dictionary with a bounded LRU window (`WindowSize
= 144`) so the string table cannot grow without bound
(`../../../vice-and-order-research/mods/dossiers/custom-chirps/repo/CustomChirps/Utils/RuntimeChirpLocalization.cs#L8-L45`).

### 3. Create a marker, remember the payload by token, spawn with the marker as target

Do not try to catch the spawned entity. Instead create a temporary marker entity carrying a
unique `Token` plus the real final targets, file the payload under that token, and hand the
**marker itself** to the game as the chirp's `m_Target`.

```csharp
var marker = RuntimeChirpTextBus.CreateMarker(em, req.Target, req.Target2, req.Target3, out var token);
RuntimeChirpTextBus.AddPendingByToken(token, in payload);

var data = new ChirpCreationData
{
    m_TriggerPrefab = _chirpPrefabEntity,
    m_Sender = senderAccount,
    m_Target = marker
};
// enqueued onto CreateChirpSystem's queue via an IJob (see source)
```
Source: `../../../vice-and-order-research/mods/dossiers/custom-chirps/repo/CustomChirps/Systems/CustomChirpApiSystem.cs#L348-L363` (@f018ac382e93e0b56cd7b974d1ce0b155d23d2b4)

The marker component is tiny - a token and up to three final targets:

```csharp
public struct CustomChirpMarker : IComponentData
{
    public ulong Token;        // unique id to look up the payload
    public Entity FinalTarget; // the real building/entity we want linked in the chirp
    public Entity FinalTarget2;
    public Entity FinalTarget3;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/custom-chirps/repo/CustomChirps/Systems/RuntimeChirpTextBus.cs#L9-L15` (@f018ac382e93e0b56cd7b974d1ce0b155d23d2b4)

`CreateMarker` allocates the token under a lock, spawns the marker entity, and stamps it
(`RuntimeChirpTextBus.cs#L58-L81`).

### 4. Consume the marker under lock and swap it for the real link(s)

When the game spawns the chirp it copies `m_Target` into the chirp's `ChirpEntity` link
buffer. Step 1 calls this: scan the buffer for a marker, `Remove` the payload from the token
map (atomic claim under `_lock`), replace the marker link in place with `FinalTarget`
(preserving `{LINK_1}` order), append the extra targets, and destroy the marker entity.

```csharp
if (em.HasComponent<CustomChirps.Components.CustomChirpMarker>(cand))
{
    var marker = em.GetComponentData<CustomChirps.Components.CustomChirpMarker>(cand);
    bool found;
    lock (_lock) { found = _byToken.Remove(marker.Token, out payload); }
    if (!found) return false;

    // Replace marker with real target(s), preserving link order for {LINK_1}, {LINK_2}, ...
    if (marker.FinalTarget != Entity.Null)
        links[i] = new Game.Triggers.ChirpEntity(marker.FinalTarget);
    else { links.RemoveAt(i); i--; }
    if (marker.FinalTarget2 != Entity.Null) AddUniqueLink(links, marker.FinalTarget2);
    if (marker.FinalTarget3 != Entity.Null) AddUniqueLink(links, marker.FinalTarget3);

    em.DestroyEntity(cand);
    return true;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/custom-chirps/repo/CustomChirps/Systems/RuntimeChirpTextBus.cs#L101-L125` (@f018ac382e93e0b56cd7b974d1ce0b155d23d2b4)

Using `_byToken.Remove` (not a read) is what makes the binding one-shot and race-free: the
first consumer to claim the token wins, and a re-entrant `GetMessageID` on the same chirp
finds nothing and falls through to the already-stamped `ModChirpText`.

### 5. Serve the text with a `TryGetValue` prefix at `Priority.First`

This is the no-reload localization trick. Patch `LocalizationDictionary.TryGetValue`; for a
`customchirps:` key, answer from your dictionary and short-circuit vanilla. `Priority.First`
makes sure you win against other prefixes.

```csharp
[HarmonyPatch(typeof(LocalizationDictionary), nameof(LocalizationDictionary.TryGetValue))]
internal static class RuntimeChirpLocalizationPatch
{
    [HarmonyPriority(Priority.First)]
    public static bool Prefix(string entryID, ref string value, ref bool __result)
    {
        if (!string.IsNullOrEmpty(entryID) &&
            entryID.StartsWith("customchirps:", System.StringComparison.Ordinal) &&
            RuntimeChirpLocalization.TryGetValue(entryID, out var runtimeValue))
        {
            value = runtimeValue;
            __result = true;
            return false;
        }
        return true;
    }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/custom-chirps/repo/CustomChirps/UI/RuntimeChirpLocalizationPatch.cs#L7-L24` (@f018ac382e93e0b56cd7b974d1ce0b155d23d2b4)

### 6. Own the publish set with one `StartsWith` check

When you want to suppress vanilla chirps, prefix `ChirperUISystem.PublishAddedChirps`: read
the UI's own "new chirps" query, and for anything whose message ID does **not** start with
`customchirps`, strip it out of the created/updated set so vanilla never publishes it.

```csharp
var em = __instance.EntityManager;
var list = createdQuery.ToEntityArray(Allocator.Temp); // snapshot so we can mutate
foreach (var e in list)
{
    var key = __instance.GetMessageID(e);      // same id the UI will use
    if (key.StartsWith("customchirps")) continue;

    if (!em.HasComponent<Deleted>(e)) em.AddComponent<Deleted>(e);
    if (em.HasComponent<Created>(e))  em.RemoveComponent<Created>(e);
    if (em.HasComponent<Updated>(e))  em.RemoveComponent<Updated>(e);
}
```
Source: `../../../vice-and-order-research/mods/dossiers/custom-chirps/repo/CustomChirps/UI/PublishAddedChirps_FilterPatch.cs#L31-L50` (@f018ac382e93e0b56cd7b974d1ce0b155d23d2b4)

The sibling panel filter (`UpdateChirps_FilterPatch`) does the same `StartsWith` test to keep
only your chirps in the open Chirper panel (`UpdateChirps_FilterPatch.cs#L61-L66`).

### 7. (Optional) Async image pipeline: register a token, bind it, resolve in JS

Large chirps with a portrait image cannot ship the URL through ECS text. Register the source
string, get back a stable `img_<guid>` token (dedup by source, bump a version counter):

```csharp
string token = $"{TokenPrefix}{Guid.NewGuid():N}";   // TokenPrefix = "img_"
s_TokenBySource[source] = token;
s_SourceByToken[token]  = source;
s_Version++;
return token;
```
Source: `../../../vice-and-order-research/mods/dossiers/custom-chirps/repo/CustomChirps/Systems/CustomChirpImageSourceRegistry.cs#L25-L42` (@f018ac382e93e0b56cd7b974d1ce0b155d23d2b4)

A UI system republishes the token->source table only when the version changes, then the JS
resolves the token to a URL and shows a pending state until the binding arrives:

```javascript
if (!isRegisteredImageToken(trimmed)) {
    return { source: trimmed, pending: false };      // absolute url / data: / Media/
}
if (Array.isArray(entries)) {
    const match = entries.find((entry) => entry?.token === trimmed);
    const registeredSource = typeof match?.source === "string" ? match.source.trim() : "";
    if (registeredSource) return { source: registeredSource, pending: false };
}
return { source: null, pending: true };              // token not bound yet -> wait
```
Source: `../../../vice-and-order-research/mods/dossiers/custom-chirps/repo/CustomChirps/CustomChirpsUI/CustomChirps.mjs#L40-L58` (@f018ac382e93e0b56cd7b974d1ce0b155d23d2b4)

The C# side only calls `binding.Update()` when `CustomChirpImageSourceRegistry.Version`
changes, so the (potentially large) source table is not re-serialized every frame
(`../../../vice-and-order-research/mods/dossiers/custom-chirps/repo/CustomChirps/Systems/CustomChirpImageSourceUISystem.cs#L24-L31`).

## Pitfalls & gotchas

- **Asymmetric drift-guard between the two filter patches.** `UpdateChirps_FilterPatch`
  reflects its `m_ChirpQuery` field and `GetSortedChirps` method up front and null-guards them
  (`if (s_ChirpQueryField == null || s_GetSortedChirpsMethod == null) return true;`), so a game
  update that renames either member degrades to vanilla behaviour
  (`../../../vice-and-order-research/mods/dossiers/custom-chirps/repo/CustomChirps/UI/UpdateChirps_FilterPatch.cs#L17-L29`).
  Its sibling `PublishAddedChirps_FilterPatch` reflects `m_CreatedChirpQuery` inline with **no
  null-guard** and immediately calls `.GetValue(__instance)`
  (`../../../vice-and-order-research/mods/dossiers/custom-chirps/repo/CustomChirps/UI/PublishAddedChirps_FilterPatch.cs#L25-L26`)
  - if that field is renamed in a future patch, `AccessTools.Field` returns `null` and this
  prefix throws an NRE. If you copy this pattern, guard **every** reflected member the same way.

- **The token binding is one-shot by design.** `TryConsumeByMarker` uses
  `_byToken.Remove(...)`, so once claimed the payload is gone. This is what prevents a
  double-apply if `GetMessageID` runs twice on the same chirp, but it also means if you never
  route the marker into a chirp's link buffer the payload leaks until `Clear()`. The marker
  reaches the buffer only because it is passed as `ChirpCreationData.m_Target`
  (`CustomChirpApiSystem.cs#L355`); skip that and the token is never consumed.

- **`Priority.First` matters on the localization prefix.** The text resolver only wins the
  race against other `TryGetValue` prefixes because it declares `[HarmonyPriority(Priority.First)]`
  (`RuntimeChirpLocalizationPatch.cs#L10`). Without it, another mod's prefix could short-circuit
  the lookup before yours runs and your `customchirps:` text would fall through to a missing-key
  string.

- **Everything hinges on the `customchirps:` prefix contract.** Publish filtering, panel
  filtering, image-mode detection, and text resolution are all `StartsWith` checks on the key
  built in step 2. If the key format and the checks ever disagree (e.g. you add a new prefix
  variant but forget one branch), chirps silently route to the wrong path. Keep the prefix set in
  one place.

- **Bounded string window can evict old chirps' text.** `RuntimeChirpLocalization` keeps only
  the last 144 keys (`RuntimeChirpLocalization.cs#L8`). A chirp older than the window that is
  re-localized (e.g. reopening a long feed) can lose its text. Whether the game actually
  re-queries `TryGetValue` for old visible chirps is **Needs Verification (in-game)**.

- **Sender/account resolution is by prefab name.** Departments map to hard-coded prefab names
  like `"PoliceChirperAccount"` (`CustomChirpApiSystem.cs#L376-L397`); a renamed vanilla account
  prefab would break sender resolution silently, the same fragility as any named-prefab lookup.

## Variations

- **Panel-only ownership vs publish suppression.** The two filters are independent settings:
  `disable_vanilla_chirps` gates the publish-set strip (step 6), while
  `hide_vanilla_chirps_in_chirper_panel` gates the panel rebuild
  (`UpdateChirps_FilterPatch.cs#L25`). Ship one without the other if you only want to *add*
  chirps rather than take over the feed.

- **Legacy target/prefab matching instead of tokens.** Before the marker mechanism, the same bus
  matched a spawned chirp to a queued payload heuristically - by target entity, by prefab, or by a
  scored best-match over prefab + sender + target
  (`RuntimeChirpTextBus.cs#L148-L235`). These are still present as fallbacks but are
  non-deterministic (they can mis-pair when two payloads share a target); the token/marker path is
  the deterministic replacement. Prefer the marker.

- **Direct URL / `data:` / `Media/` image instead of a registered token.** The JS treats any
  source that is not an `img_<guid>` token as a literal and uses it directly, skipping the async
  binding round-trip (`CustomChirps.mjs#L28-L47`). Use a registered token only when the source is
  large or dynamic enough to warrant deduping and versioned publishing.

## See also
- Related recipes: [namesystem custom names](namesystem-custom-names.md) (the parallel
  runtime-string-injection surface for entity names), [cross-mod service protocol](cross-mod-service-protocol.md)
  (letting other mods enqueue chirps through your API), [localization helper](localization-helper.md)
  (the conventional locale path this recipe deliberately bypasses).
- Reference: [technique index](../../technique-index.md) (family AQ).
- Case study demonstrating it: [custom-chirps](../../case-studies/custom-chirps.md).

## Sources
- Canonical mods (dossier + repo):
  - `custom-chirps` @f018ac382e93e0b56cd7b974d1ce0b155d23d2b4 -
    `repo/CustomChirps/UI/ChirperMessagePatch.cs`,
    `repo/CustomChirps/UI/RuntimeChirpLocalizationPatch.cs`,
    `repo/CustomChirps/UI/PublishAddedChirps_FilterPatch.cs`,
    `repo/CustomChirps/UI/UpdateChirps_FilterPatch.cs`,
    `repo/CustomChirps/Systems/RuntimeChirpTextBus.cs`,
    `repo/CustomChirps/Systems/CustomChirpApiSystem.cs`,
    `repo/CustomChirps/Systems/CustomChirpImageSourceRegistry.cs`,
    `repo/CustomChirps/Systems/CustomChirpImageSourceUISystem.cs`,
    `repo/CustomChirps/Utils/RuntimeChirpLocalization.cs`,
    `repo/CustomChirps/CustomChirpsUI/CustomChirps.mjs`
- Official/community references (link out, do not duplicate): https://cs2.paradoxwikis.com/Modding
