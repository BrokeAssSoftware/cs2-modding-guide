---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Case study: Custom Chirps"
case_study: custom-chirps
mod: "Custom Chirps"
dossier: ../../vice-and-order-research/mods/dossiers/custom-chirps/
repo_commit: f018ac382e93e0b56cd7b974d1ce0b155d23d2b4
source_version: "~1.5.x (custom-chirps@f018ac3; date-pinned, static source only)"
last_reverified: "2026-07-05"
diataxis: explanation
techniques: [AQ, Q, G]
technique_applicability: [ui, media, platform]
status: source-verified
Created: 2026-07-05
Updated: 2026-07-05
Owners:
  - codex
---

# Custom Chirps - case study

> A mod that takes ownership of the vanilla Chirper feed - tagging every message
> it creates with `customchirps:` keys - and exposes a thread-safe static
> `PostChirp*` producer API that other mods (Elections, Time2Work) call purely by
> reflection. It is a compact tour of how UI-injection, cross-mod-provider, and
> custom-name techniques combine when one mod becomes an infrastructure surface
> for others.

All code claims below are cited to `repo/<path>#Lxx` at pinned commit
`f018ac382e93e0b56cd7b974d1ce0b155d23d2b4` ("Chirps with image can now support
three links"), surfaced through the dossier at
`../../vice-and-order-research/mods/dossiers/custom-chirps/`. Every type and
method named here was confirmed present at that commit.

## What it does / why it's instructive

Custom Chirps lets any code post a message into the in-game Chirper feed with
custom text, a chosen department sender icon, a custom sender label, up to three
clickable entity links, an optional "large" layout, and an optional portrait
image. The mod itself ships almost no gameplay - its value is the **producer
API** (`CustomChirps.Systems.CustomChirpApiSystem.PostChirp*`) that consumer mods
invoke, plus the machinery that makes an arbitrary runtime string render inside a
UI pipeline that was never designed to accept one.

It is instructive because it solves the same problem three different ways that
each map to a distinct technique family:

- To make text appear, it does not add a locale entry and reload the locale
  (which stutters). It **intercepts the localization dictionary lookup** for keys
  it owns (family AQ - Chirper/UI feed injection).
- To make the sender row show a custom name, it reuses the vanilla
  `NameSystem.Name.CustomName` binding contract (family Q).
- To let other mods drive it without a compile-time reference, it is the
  **provider side of a reflection bridge**: a static queue drained on the main
  thread (family G).

The whole design reduces to one predicate repeated everywhere: does this message
id `StartsWith("customchirps")`? That single decision drives publishing, panel
filtering, localization, and the JS layout switch.

## Architecture at a glance

### Systems and scheduling

`Mod.OnLoad` registers three systems and applies all Harmony patches in the
assembly (repo/CustomChirps/Mod.cs#L44-L55):

- `CustomChirpSpawnerSystem` - `UpdateAfter<CreateChirpSystem>` in
  `SystemUpdatePhase.GameSimulation`; it stamps `ModChirpText` (and marker-derived
  payload) onto freshly created chirps (repo/CustomChirps/Mod.cs#L46-L47).
- `CustomChirpApiSystem` - `UpdateAt` in `SystemUpdatePhase.GameSimulation`; it
  drains the producer queue (repo/CustomChirps/Mod.cs#L49).
- `CustomChirpImageSourceUISystem` - `UpdateAt` in `SystemUpdatePhase.UIUpdate`;
  it publishes the image-token table to the UI (repo/CustomChirps/Mod.cs#L50).

Harmony (`harmonyID = "CustomChirps"`) is load-bearing: `harmony.PatchAll` binds
the six `ChirperUISystem` / `LocalizationDictionary` patches described below
(repo/CustomChirps/Mod.cs#L53-L55).

### The publish path (marker -> entity binding)

The API never writes chirp text into the ECS world directly. `PostChirp*` enqueue
a `PendingRequest` onto a `ConcurrentQueue`; `OnUpdate` drains it on the main
thread and, per request, builds a `customchirps:`-prefixed key, registers the
text in a runtime dictionary, and enqueues a vanilla `ChirpCreationData` via
`CreateChirpSystem.GetQueue` / `AddQueueWriter`
(repo/CustomChirps/Systems/CustomChirpApiSystem.cs#L317-L363). The trick that
binds "which text belongs to which chirp" deterministically is a **throwaway
marker entity**: `RuntimeChirpTextBus.CreateMarker` mints a monotonic `ulong`
token, stamps a `CustomChirpMarker` on a temp entity, and passes that entity as
the chirp's link target; the payload is stashed by token
(repo/CustomChirps/Systems/RuntimeChirpTextBus.cs#L68-L86,
repo/CustomChirps/Systems/CustomChirpApiSystem.cs#L348-L356). When the chirp
materializes, `TryConsumeByMarker` walks its `ChirpEntity` link buffer, finds the
marker, pops the payload by token, replaces the marker link with the real
target(s), and destroys the temp entity
(repo/CustomChirps/Systems/RuntimeChirpTextBus.cs#L88-L128).

### Data flow / phases summary

- `GameSimulation`: `CustomChirpApiSystem` drains queued requests and injects
  `ChirpCreationData`; `CustomChirpSpawnerSystem` (after `CreateChirpSystem`)
  consumes markers on new chirps and stamps `ModChirpText`.
- UI bind time (Harmony on `ChirperUISystem`): `GetMessageID` returns the
  `customchirps:` key, `BindChirpSender` writes the custom sender row,
  `PublishAddedChirps` / `UpdateChirps` filter vanilla out when settings say so.
- `LocalizationDictionary.TryGetValue` (any thread the UI resolves on): the
  prefix-patch returns the runtime text for `customchirps:` keys.
- `UIUpdate`: `CustomChirpImageSourceUISystem` flushes the image-token table to JS
  when its version changes; `CustomChirps.mjs` resolves tokens to real sources.

## Techniques demonstrated

- [Chirper feed injection](../how-to/recipes/chirper-feed-injection.md) (family AQ)
  - the mod owns the whole feed pipeline through message-id tagging. Its
  `GetMessageID` prefix returns the runtime `customchirps:` key so every
  downstream decision is a `StartsWith` check
  (repo/CustomChirps/UI/ChirperMessagePatch.cs#L9-L69). Runtime text is resolved
  by a `LocalizationDictionary.TryGetValue` prefix gated on the `customchirps:`
  prefix, avoiding `ReloadActiveLocale`
  (repo/CustomChirps/UI/RuntimeChirpLocalizationPatch.cs#L7-L23) backed by a
  144-entry LRU window (repo/CustomChirps/Utils/RuntimeChirpLocalization.cs#L8-L45).
  Publish/panel visibility are two more prefixes on `PublishAddedChirps` and
  `UpdateChirps`, each gated on a settings toggle
  (repo/CustomChirps/UI/PublishAddedChirps_FilterPatch.cs#L15-L58,
  repo/CustomChirps/UI/UpdateChirps_FilterPatch.cs#L12-L59). See also
  [localization at runtime](../how-to/recipes/localization-helper.md) and
  [reflecting into private UI internals](../how-to/recipes/harmony-traverse-private-fields.md).
- [Custom sender-row names](../how-to/recipes/namesystem-custom-names.md) (family
  Q) - `BindChirpSender` is prefixed so that when a payload carries an override
  label the mod writes the `chirper.ChirpSender` row itself and hands
  `NameSystem.Name.CustomName(...)` to `BindChirpLink`, then returns `false` to
  skip vanilla (repo/CustomChirps/UI/ChirperSenderPatch.cs#L19-L74). The same
  `CustomName` contract is the recipe's canonical pattern for surfacing a
  mod-chosen string through a vanilla name binding.
- [Reflection mod bridges](../how-to/recipes/reflection-mod-bridges.md) (family G)
  - Custom Chirps is the **provider** end. The public surface is a set of static
  `PostChirp* / PostLargeChirp* / PostChirpFromEntity*` overloads that any
  assembly can call by reflection with no shared type
  (repo/CustomChirps/Systems/CustomChirpApiSystem.cs#L76-L208). They only
  `Enqueue` onto a `ConcurrentQueue<PendingRequest>`
  (repo/CustomChirps/Systems/CustomChirpApiSystem.cs#L68-L69,
  repo/CustomChirps/Systems/CustomChirpApiSystem.cs#L210-L233); the consumer
  (`OnUpdate`) drains at most `MaxRequestsPerFrame = 512` per frame on the main
  thread, wrapping each in try/catch so one bad request cannot stall the feed
  (repo/CustomChirps/Systems/CustomChirpApiSystem.cs#L69,
  repo/CustomChirps/Systems/CustomChirpApiSystem.cs#L248-L267). This is the
  producer/consumer half of the pattern that
  [cross-mod service protocols](../how-to/recipes/cross-mod-service-protocol.md)
  and [main-thread dispatch](../how-to/recipes/mainthread-dispatcher-async.md)
  describe from the consumer side.

Supporting technique also on display: an **async C#<->JS image pipeline**. The
API registers a portrait source string, gets back an `img_<guid>` token, and bakes
it into the key as `customchirps:portraitimg:<token>:`
(repo/CustomChirps/Systems/CustomChirpImageSourceRegistry.cs#L25-L42,
repo/CustomChirps/Systems/CustomChirpApiSystem.cs#L318-L329). A `RawValueBinding`
under group `customChirps` / prop `imageSources` re-publishes the token table only
when the registry version changes
(repo/CustomChirps/Systems/CustomChirpImageSourceUISystem.cs#L15-L32). The JS side
binds that value, and `resolveImageSource` returns `{ pending: true }` for a token
not yet in the table, re-resolving via a React effect once the binding flushes -
so a token can reach the UI before its source does without breaking the render
(repo/CustomChirps/CustomChirpsUI/CustomChirps.mjs#L10,
repo/CustomChirps/CustomChirpsUI/CustomChirps.mjs#L40-L59,
repo/CustomChirps/CustomChirpsUI/CustomChirps.mjs#L164-L218). The JS also switches
layout purely on the message-id prefix
(repo/CustomChirps/CustomChirpsUI/CustomChirps.mjs#L112-L118).

## Key decisions & tradeoffs

- **Own the whole publish/panel pipeline, reduce every decision to
  `StartsWith("customchirps")`.** Rather than track its own entities, the mod
  makes `GetMessageID` the single source of identity: publish filtering, panel
  filtering, localization, and JS layout all key off the same prefix
  (repo/CustomChirps/UI/ChirperMessagePatch.cs#L53,
  repo/CustomChirps/UI/UpdateChirps_FilterPatch.cs#L61-L66,
  repo/CustomChirps/UI/RuntimeChirpLocalizationPatch.cs#L13-L16). Cheap and
  uniform, but it means any vanilla message that ever started with that literal
  would be misclassified.
- **Runtime localization instead of locale reload.** Text is served from an
  in-process LRU dictionary via a `TryGetValue` prefix, explicitly to avoid
  `LocalizationManager.ReloadActiveLocale()` and its stutter
  (repo/CustomChirps/Systems/CustomChirpApiSystem.cs#L331-L334,
  repo/CustomChirps/UI/RuntimeChirpLocalizationPatch.cs#L7-L23). Tradeoff: only
  the last 144 keys survive (repo/CustomChirps/Utils/RuntimeChirpLocalization.cs#L8),
  so a very old chirp scrolled back to in the panel can lose its text.
- **Marker-entity binding over content matching.** A monotonic token on a
  throwaway entity binds chirp->payload deterministically, sidestepping the fuzzy
  target/sender/prefab scoring the legacy path still contains
  (repo/CustomChirps/Systems/RuntimeChirpTextBus.cs#L88-L128 vs
  repo/CustomChirps/Systems/RuntimeChirpTextBus.cs#L192-L235).
- **Bounded main-thread drain.** The producer API is lock-free
  (`ConcurrentQueue`) and the consumer caps work at 512/frame, trading worst-case
  latency for a guaranteed frame budget
  (repo/CustomChirps/Systems/CustomChirpApiSystem.cs#L68-L69,
  repo/CustomChirps/Systems/CustomChirpApiSystem.cs#L254-L267).
- **Reflect into private `ChirperUISystem` internals via `AccessTools`.** The
  filters read private fields `m_CreatedChirpQuery` / `m_ChirpQuery` and invoke
  private `GetSortedChirps` (repo/CustomChirps/UI/UpdateChirps_FilterPatch.cs#L17-L35,
  repo/CustomChirps/UI/PublishAddedChirps_FilterPatch.cs#L25-L32) - powerful, but
  a rename in a CS2 patch breaks the binding (see pitfalls).

## Pitfalls / upstream-watch

- **Asymmetric drift-guard between sibling filters.**
  `UpdateChirps_FilterPatch` caches its reflected `FieldInfo`/`MethodInfo` as
  statics and null-guards them, returning `true` (fall back to vanilla) if either
  resolves to null - it degrades gracefully when the field is renamed
  (repo/CustomChirps/UI/UpdateChirps_FilterPatch.cs#L17-L29). Its sibling
  `PublishAddedChirps_FilterPatch` resolves the field inline and casts the result
  with no null-check, so the same rename throws an NRE inside the prefix instead
  of degrading (repo/CustomChirps/UI/PublishAddedChirps_FilterPatch.cs#L25-L26).
  `PublishAddedChirps_ClearTags_Postfix` has the same unguarded inline pattern
  (repo/CustomChirps/UI/PublishAddedChirps_ClearCreated_Postfix.cs#L17-L18,
  repo/CustomChirps/UI/PublishAddedChirps_ClearCreated_Postfix.cs#L30-L31).
  Watch both `ChirperUISystem` field names on every CS2 update.
- **Latent dead vanilla-throttle path.** `CustomChirpSpawnerSystem` hardcodes
  `allowPct = 100`, making the two throttle branches below it unreachable dead
  code (repo/CustomChirps/Systems/CustomChirpSpawnerSystem.cs#L34,
  repo/CustomChirps/Systems/CustomChirpSpawnerSystem.cs#L71-L79). The
  `Mod.VanillaKeepPercent` field that presumably fed it exists but is never read
  (repo/CustomChirps/Mod.cs#L29). Treat percentage-based vanilla suppression as
  not implemented; suppression is all-or-nothing via the settings toggles.
- **Prefix collision surface.** All identity keys off the literal
  `customchirps` / `customchirps:`; a future vanilla or third-party key sharing
  that prefix would be silently captured or filtered
  (repo/CustomChirps/UI/PublishAddedChirps_FilterPatch.cs#L40,
  repo/CustomChirps/UI/RuntimeChirpLocalizationPatch.cs#L14).
- **Two consumers race for the same marker.** Both
  `ChirperMessagePatch.GetMessageID` and `CustomChirpSpawnerSystem` call
  `TryConsumeByMarker`, which `Remove`s the token; whichever runs first wins and
  the other falls through
  (repo/CustomChirps/UI/ChirperMessagePatch.cs#L22, 
  repo/CustomChirps/Systems/CustomChirpSpawnerSystem.cs#L47). It is safe (the loser
  no-ops) but the dual path is easy to misread as redundant.
- **Sender override needs a valid cim.** `PostChirpFromEntity*` require the sender
  entity to carry `Game.Citizens.Citizen`; otherwise the request is dropped with a
  warning (repo/CustomChirps/Systems/CustomChirpApiSystem.cs#L296-L302).

Needs Verification (in-game): the actual on-screen appearance of large/portrait
chirps, whether the 144-entry text window ever visibly drops old chirp text
during normal play, and the real-world call sites in Elections / Time2Work
(only the provider side is present in this repo).

## Source pointers

- Dossier: `../../vice-and-order-research/mods/dossiers/custom-chirps/`.
- Repo @ `f018ac382e93e0b56cd7b974d1ce0b155d23d2b4`, key files:
  - `repo/CustomChirps/Mod.cs` - system scheduling + Harmony `PatchAll`.
  - `repo/CustomChirps/Systems/CustomChirpApiSystem.cs` - static `PostChirp*`
    producer API, `ConcurrentQueue`, 512/frame drain, key building.
  - `repo/CustomChirps/Systems/RuntimeChirpTextBus.cs` - token/marker payload
    binding.
  - `repo/CustomChirps/UI/ChirperMessagePatch.cs`,
    `.../ChirperSenderPatch.cs` - `GetMessageID` + `BindChirpSender` prefixes.
  - `repo/CustomChirps/UI/RuntimeChirpLocalizationPatch.cs`,
    `.../PublishAddedChirps_FilterPatch.cs`, `.../UpdateChirps_FilterPatch.cs` -
    localization + publish/panel filters.
  - `repo/CustomChirps/Systems/CustomChirpImageSourceRegistry.cs`,
    `.../CustomChirpImageSourceUISystem.cs`,
    `repo/CustomChirps/CustomChirpsUI/CustomChirps.mjs` - async C#<->JS image
    pipeline.
