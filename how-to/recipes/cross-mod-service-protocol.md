---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: Cross-mod runtime service protocol"
recipe: cross-mod-service-protocol
technique_family: "AP - Cross-mod runtime service protocol (call/offer another mod)"
diataxis: how-to
source_version: "~1.6.0f1 (elections-rt-module@36c25af; date-pinned, static source only)"
last_reverified: "2026-07-05"
status: source-verified
canonical_mods:
  - elections-rt-module@36c25afcda67c80ce75ba68d423fe8238499435a
  - custom-chirps@f018ac382e93e0b56cd7b974d1ce0b155d23d2b4
  - anarchy@a6311e898d20a775368668b234aaa32f06e3e1eb
technique_applicability: [platform, simulation]
Created: 2026-07-05
Updated: 2026-07-05
Owners:
  - codex
---

# Cross-mod runtime service protocol

> Drive another mod's live behaviour by calling its runtime API (encode your
> semantics into the host's method + track the outcome yourself), fire a vanilla
> game event by hand, or OFFER such a surface to other mods with a thread-safe
> static entry point.

## Problem
You need two mods to cooperate at **runtime**, not just co-exist. As a **caller**
you want to make a host mod *do something* - spawn a trip, run a policy, react to a
state change - and remember what you asked for. As a **provider** you want to expose
a surface other mods can call safely, possibly from a job or a background thread.
Plain reflection that merely *resolves* another mod's types (family G) is not enough:
here you are invoking behaviour and coordinating over a shared protocol, and the peer
may be absent, a different version, or calling you from the wrong thread.

## Solution
Three composable sub-patterns, all verified below:

1. **Call a host method over an agreed protocol.** Treat the host's public method as a
   wire format: pack your mod-specific meaning into its parameters (here, integer
   *trip-type codes*), invoke it through a resolved delegate, and record the result on
   a **per-entity tracking component** you own.
2. **Fire a vanilla event by hand.** When the "host" is the base game, build the
   event's ECS archetype yourself (`Event` + the event component), instantiate an
   entity of that archetype, and set its payload - the game's systems pick it up.
3. **Offer a surface.** Expose a `public static` API. If callers may run off the main
   thread, make the entry point enqueue onto a `ConcurrentQueue` and drain it on the
   main thread (any-thread producer, main-thread consumer); otherwise a plain static
   bridge is enough.

Everything the caller does is guarded so a missing or throwing peer is a safe no-op.

## Steps & Code

### 1. (Caller) Encode your semantics into the host's method signature

Realistic Trips (assembly `Time2Work`) exposes one generic entry point,
`RequestSocialTrip(citizen, target, host, tripType, durationMinutes, priority)`.
Elections resolves it as a delegate and declares that exact shape:

```csharp
private delegate bool RequestSocialTripDelegate(Entity citizen, Entity targetBuilding,
    Entity hostCitizen, int tripType, float durationMinutes, int priority);
```
Source: `../../../vice-and-order-research/mods/dossiers/elections-rt-module/repo/Elections/Bridge/RealisticTripsBridge.cs#L15` (@36c25afcda67c80ce75ba68d423fe8238499435a)

The `int tripType` is the protocol. Elections gives each of its intents a distinct
code and wraps each in a named method - voting is `1001`:

```csharp
public static bool RequestVotingTrip(Entity citizen, Entity targetBuilding, float durationMinutes, int priority)
{
    EnsureResolve();
    if (s_RequestSocialTrip == null)
        return false;

    try
    {
        return s_RequestSocialTrip(citizen, targetBuilding, Entity.Null, 1001, durationMinutes, priority);
    }
    catch (Exception ex)
    {
        Mod.log.Warn($"RealisticTrips RequestSocialTrip failed: {ex.Message}");
        return false;
    }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/elections-rt-module/repo/Elections/Bridge/RealisticTripsBridge.cs#L154-L169` (@36c25afcda67c80ce75ba68d423fe8238499435a)

The victory-party trip uses code `1002`
(`RealisticTripsBridge.cs#L179`) and the bribe-meeting trip uses `1003` and additionally
passes a real `hostCitizen` instead of `Entity.Null`
(`RealisticTripsBridge.cs#L196`) - same method, three different meanings selected by the code.

### 2. (Caller) Negotiate capability and degrade gracefully

Delegate resolution returns `null` when the host method is absent, so a single
`IsAvailable` check tells you whether the whole surface is usable:

```csharp
public static bool IsAvailable
{
    get
    {
        EnsureResolve();
        return s_RequestSocialTrip != null;
    }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/elections-rt-module/repo/Elections/Bridge/RealisticTripsBridge.cs#L54-L61` (@36c25afcda67c80ce75ba68d423fe8238499435a)

Optional finer capabilities degrade to a safe default when the host does not expose
them. `CanRequestTrip` falls back to base availability if the host has no dedicated
eligibility method:

```csharp
public static bool CanRequestTrip(Entity citizen, Entity targetBuilding)
{
    EnsureResolve();
    if (s_CanRequestSocialTrip == null)
        return IsAvailable;   // optional capability missing -> fall back to base availability
    ...
}
```
Source: `../../../vice-and-order-research/mods/dossiers/elections-rt-module/repo/Elections/Bridge/RealisticTripsBridge.cs#L63-L67` (@36c25afcda67c80ce75ba68d423fe8238499435a)

### 3. (Caller) Track what you requested with a per-entity component

The host owns the trip; you own the record. After the peer *accepts* the request,
Elections stamps a tracking component on the citizen so its own systems can follow the
trip's lifecycle (and it survives saves - the component is `ISerializable`):

```csharp
if (!RealisticTripsBridge.CanRequestTrip(citizen, pollingPlace))
{
    rejectedByBridgeCount++;
    continue;
}

float visitDurationMinutes = random.NextFloat(kMinVotingVisitMinutes, kMaxVotingVisitMinutes);
if (!RealisticTripsBridge.RequestVotingTrip(citizen, pollingPlace, visitDurationMinutes, 100))
{
    rejectedByBridgeCount++;
    continue;
}

EntityManager.AddComponentData(citizen, new ElectionVoteTrip
{
    version = 1,
    electionDayKey = state.electionDayKey,
    pollingPlace = pollingPlace,
    voted = false,
    chosenCandidate = -1
});
```
Source: `../../../vice-and-order-research/mods/dossiers/elections-rt-module/repo/Elections/Systems/ElectionVotingSystem.cs#L299-L319` (@36c25afcda67c80ce75ba68d423fe8238499435a)

The component is a small POD struct with a versioned serializer, so the record persists
and can be migrated (`ElectionVoteTrip.cs#L6-L12`).

### 4. (Caller) Fire a vanilla event by hand

When the "host" is the base game, there is no method to call - you build the event's
archetype yourself. A vanilla event is an entity carrying `Game.Common.Event` plus the
specific event component. Elections builds the renter-update archetype once in
`OnCreate`:

```csharp
m_RentEventArchetype = EntityManager.CreateArchetype(
    ComponentType.ReadWrite<Game.Common.Event>(),
    ComponentType.ReadWrite<RentersUpdated>());
```
Source: `../../../vice-and-order-research/mods/dossiers/elections-rt-module/repo/Elections/Systems/MayorWorkplaceSystem.cs#L95-L97` (@36c25afcda67c80ce75ba68d423fe8238499435a)

Firing it is: instantiate an entity of that archetype and set the payload. Vanilla
systems that consume `RentersUpdated` then run as if the game raised it:

```csharp
Entity rentEvent = EntityManager.CreateEntity(m_RentEventArchetype);
EntityManager.SetComponentData(rentEvent, new RentersUpdated(property));
```
Source: `../../../vice-and-order-research/mods/dossiers/elections-rt-module/repo/Elections/Systems/MayorWorkplaceSystem.cs#L767-L768` (@36c25afcda67c80ce75ba68d423fe8238499435a)

### 5. (Provider) Offer a thread-safe surface: static enqueue, main-thread drain

Custom Chirps lets any mod post a chirp. Because callers may run off the main thread,
the public API only *enqueues* onto a `ConcurrentQueue`; nothing touches ECS until the
main thread drains it:

```csharp
private static readonly ConcurrentQueue<PendingRequest> s_requests = new ConcurrentQueue<PendingRequest>();
private const int MaxRequestsPerFrame = 512;

// Public API (thread-safe): producer may be any thread
public static void PostChirp(string text, DepartmentAccount dept, Entity targetEntity, string customSenderName = null)
{
    EnqueueChirp(text, dept, targetEntity, customSenderName, Entity.Null, ChirpDisplayMode.Compact);
}
```
Source: `../../../vice-and-order-research/mods/dossiers/custom-chirps/repo/CustomChirps/Systems/CustomChirpApiSystem.cs#L68-L79` (@f018ac382e93e0b56cd7b974d1ce0b155d23d2b4)

The system drains the queue on `OnUpdate`, bounded to `MaxRequestsPerFrame`, with
each item processed inside a `try/catch` so one bad request cannot stall the drain:

```csharp
protected override void OnUpdate()
{
    EnsureInit();
    if (!_didInit) return;

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
}
```
Source: `../../../vice-and-order-research/mods/dossiers/custom-chirps/repo/CustomChirps/Systems/CustomChirpApiSystem.cs#L248-L267` (@f018ac382e93e0b56cd7b974d1ce0b155d23d2b4)

### 6. (Provider) The simpler case: a plain public static bridge

If your API only runs on the main thread, skip the queue. Anarchy exposes a
`public static` class whose methods resolve the owning system on demand and return a
`bool` success flag:

```csharp
public static class AnarchyBridge
{
    public static bool TryAddToolSystem(ToolBaseSystem tool)
    {
        AnarchyUISystem uiSystem = World.DefaultGameObjectInjectionWorld.GetOrCreateSystemManaged<AnarchyUISystem>();
        if (uiSystem is null || tool is null || tool.toolID is null)
            return false;

        return uiSystem.TryAddTool(tool.toolID);
    }
    ...
}
```
Source: `../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/Bridge/AnarchyBridge.cs#L17-L35` (@a6311e898d20a775368668b234aaa32f06e3e1eb)

## Pitfalls & gotchas

- **The protocol is unchecked.** Integer codes (`1001/1002/1003`) and parameter order
  are a private contract between two mods with **no compile-time check**. If the host
  changes what a code means, or reorders parameters, the caller silently drives the
  wrong behaviour. That the host actually maps `1001` -> a voting trip is
  `Needs Verification (in-game)` from the caller's source alone.

- **A provider surface is only as safe as its author.** An offered `public static`
  method can ship a real bug. Anarchy's array overload iterates
  `for (int i = 0; i <= entities.Length; i++)` - the `<=` reads one past the end
  (`AnarchyBridge.cs#L93`). This is exactly why every caller in this recipe wraps peer
  calls in `try/catch` and treats a throw as failure; never assume the peer's API is
  correct.

- **Producer thread vs consumer thread.** ECS structural changes (create entity, add
  component) are **main-thread only**. A provider whose callers might run in a job or a
  background thread MUST marshal, as Custom Chirps does with the `ConcurrentQueue` +
  main-thread drain. Doing ECS work directly in a static method invoked from another
  thread is a crash waiting to happen.

- **Bounded drains create backpressure.** The drain caps at
  `MaxRequestsPerFrame = 512` (`CustomChirpApiSystem.cs#L69`). If producers post faster
  than 512/frame the queue grows unboundedly; there is no drop policy in the source.
  Budget your posting rate accordingly.

- **Hand-built events assume the vanilla shape.** Firing `Event` + `RentersUpdated`
  and calling `new RentersUpdated(property)` assumes that archetype and constructor
  still exist and that a vanilla system consumes them. That is patch-sensitive; whether
  the fired event actually triggers the game's renter recalculation is
  `Needs Verification (in-game)`.

- **Track on your side, not the host's.** The caller records intent on its own
  component (`ElectionVoteTrip`) rather than reading host-internal state. This keeps the
  coupling to the single method call and lets your record survive saves via
  `ISerializable`.

## Variations

- **Component-type handshake instead of a method call.** A provider can hand callers a
  `ComponentType` so they can query/filter without referencing the concrete type.
  Anarchy exposes `GetAnarchyComponentType()` returning
  `ComponentType.ReadWrite<PreventOverride>()`:

  ```csharp
  public static ComponentType GetAnarchyComponentType()
  {
      return ComponentType.ReadWrite<PreventOverride>();
  }
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/Bridge/AnarchyBridge.cs#L263-L266` (@a6311e898d20a775368668b234aaa32f06e3e1eb)

- **Reflection-resolved caller when you cannot reference the peer assembly.** The
  Elections bridge does not link `Time2Work` at build time; it resolves the type and
  binds the method to a delegate at runtime (`Type.GetType(...)` + `CreateDelegate`,
  `RealisticTripsBridge.cs#L434-L458`). Use this when the host is optional or its
  assembly is not a build reference; see [reflection mod bridges](reflection-mod-bridges.md).

- **Overloaded named entry points over one host method.** Rather than exposing `int`
  codes to your own call sites, wrap each intent in a named method
  (`RequestVotingTrip`/`RequestElectionVictoryPartyTrip`/`RequestBribeMeetingTrip`) that
  hides the code and default arguments - your systems call intent, the bridge speaks
  protocol.

## See also
- Related recipes: [reflection mod bridges](reflection-mod-bridges.md) (resolving the
  peer's types/methods at runtime - the family-G foundation this builds on),
  [chirper feed injection](chirper-feed-injection.md) (what the Custom Chirps provider
  surface ultimately does).
- Explanation: [dependency strategy](../../explanation/dependency-strategy.md) (hard vs
  soft dependencies; when to couple to a peer at all).
- Case studies demonstrating it: [elections-rt-module](../../case-studies/elections-rt-module.md).

## Sources
- Canonical mods (dossier + repo):
  - `elections-rt-module` @36c25afcda67c80ce75ba68d423fe8238499435a - `repo/Elections/Bridge/RealisticTripsBridge.cs`, `repo/Elections/Systems/ElectionVotingSystem.cs`, `repo/Elections/Systems/MayorWorkplaceSystem.cs`, `repo/Elections/Components/ElectionVoteTrip.cs`
  - `custom-chirps` @f018ac382e93e0b56cd7b974d1ce0b155d23d2b4 - `repo/CustomChirps/Systems/CustomChirpApiSystem.cs`
  - `anarchy` @a6311e898d20a775368668b234aaa32f06e3e1eb - `repo/Anarchy/Bridge/AnarchyBridge.cs`
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding
