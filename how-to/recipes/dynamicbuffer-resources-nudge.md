---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: DynamicBuffer<Resources> economy nudging"
recipe: dynamicbuffer-resources-nudge
technique_family: "K - DynamicBuffer<Resources> economy nudging"
diataxis: how-to
source_version: "1.6.0f1 (magic-mail@6fb3d2b; date-pinned, static source only)"
last_reverified: "2026-07-04"
status: source-verified
canonical_mods:
  - magic-mail@6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47
  - market-based-economy@b83f196a36bc74388accebdeb7c81f0f35dbab37
technique_applicability: [economy]
Created: 2026-07-04
Updated: 2026-07-04
Owners:
  - codex
---

# DynamicBuffer<Resources> economy nudging

> Read and adjust the stored resources on a game entity by opening its
> `Game.Economy.Resources` dynamic buffer, finding the resource slot you care about,
> and writing a new amount back - to top up, drain, or clamp a building's or company's
> holdings.

## Problem
Almost everything the CS2 economy stores on an entity - a building's mail, a company's
product stock, a household's money - lives in a single `DynamicBuffer<Resources>`
attached to that entity. Each buffer element is a `(Resource kind, int amount)` pair.
You want to nudge one of those amounts: pull a post office's local mail up when it runs
low, clamp an overflowing facility back down, or credit a company money for a virtual
sale. Prefab-field overrides (family A) change the *template*; this technique changes
the *live per-entity balance* every tick.

## Solution
Get the entity's buffer with `EntityManager.GetBuffer<Resources>(entity)` (or receive
it as a `DynamicBuffer<Game.Economy.Resources>` parameter inside an `Entities.ForEach`),
then read/write through it. A `Resources` buffer is an unordered list, so "get amount
of resource X" means **scanning the buffer for the element whose `m_Resource == X`** and
returning its `m_Amount`; "add" means finding that element, adjusting `m_Amount`, and
assigning the struct back (`buffer[i] = value`), or appending a new element if the
resource is not present yet. The vanilla helper `Game.Economy.EconomyUtils` does exactly
this scan for you (`GetResources` / `AddResources`); some mods call it, others hand-roll
the identical loop. Both canonical mods below mutate the buffer on the **main thread**
(`.WithoutBurst()` / a plain `foreach`) because the value derivation uses managed state.

## Steps & Code

### 1. Require the buffer as ReadWrite in your query

Only entities that actually own a `Resources` buffer should reach your loop, and you must
declare write access. Magic Mail's post-facility query asks for `ReadWrite<Resources>`
alongside the building tag:

```csharp
m_PostFacilitiesQuery = GetEntityQuery(new EntityQueryDesc
{
    All = new[]
    {
        ComponentType.ReadOnly<PrefabRef>(),
        ComponentType.ReadOnly<Game.Buildings.PostFacility>(),
        ComponentType.ReadWrite<Resources>(),
    },
    None = new[]
    {
        ComponentType.ReadOnly<Destroyed>(),
        ComponentType.ReadOnly<Deleted>(),
        ComponentType.ReadOnly<Temp>(),
    },
});
```
Source: `../../../vice-and-order-research/mods/dossiers/magic-mail/repo/Systems/MagicMailSystem.cs#L84-L98` (@6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47)

### 2. Open the buffer for each entity (guard that it exists)

Iterate the matched entities on the main thread and pull the buffer. Check
`HasBuffer<Resources>` first so a mismatched entity is a safe `continue`, not a throw:

```csharp
foreach (var postEntity in postEntities)
{
    // ... resolve PrefabRef / PostFacilityData for capacity ...
    if (!entityManager.HasBuffer<Resources>(postEntity))
    {
        Mod.s_Log.Warn($"Post facility {postEntity} has no Resources buffer.");
        continue;
    }

    DynamicBuffer<Resources> resources = entityManager.GetBuffer<Resources>(postEntity);
```
Source: `../../../vice-and-order-research/mods/dossiers/magic-mail/repo/Systems/MagicMailSystem.cs#L136-L161` (@6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47)

### 3. Read the current amount, decide, write the delta

Reading and writing are both "find the slot for this `Resource`". Magic Mail reads the
current counts, checks them against a capacity threshold, and adds a top-up amount:

```csharp
var localMailCount = GetResourceAmount(resources, Resource.LocalMail);
// ... under threshold? ...
if (settings.PO_GetLocalMail &&
    mailCapacity > 0 &&
    localMailCount * 100 / mailCapacity <= settings.PO_GettingThresholdPercentage)
{
    var addAmount = mailCapacity * settings.PO_GettingPercentage / 100;
    AddResourceAmount(resources, Resource.LocalMail, addAmount);   // write the delta
}
```
Source: `../../../vice-and-order-research/mods/dossiers/magic-mail/repo/Systems/MagicMailSystem.cs#L238-L251` (@6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47)

Note the write is a **delta** (`addAmount`), not an absolute set. To clamp to an absolute
target, compute `target - current` and add that (Magic Mail's overflow path does exactly
this: `AddResourceAmount(resources, Resource.LocalMail, targetLocal - localMailCount)`,
`MagicMailSystem.cs#L300-L302`).

### 4. Understand what the read/write helper actually does

The "amount of resource X" read is a linear scan matching `m_Resource`; the write is the
same scan mutating `m_Amount` and assigning the element back, or appending if absent.
Magic Mail hand-rolls both (its comment calls them a "local replacement for
`EconomyUtils.*`"):

```csharp
private static int GetResourceAmount(DynamicBuffer<Resources> resources, Resource resource)
{
    for (var i = 0; i < resources.Length; i++)
    {
        var value = resources[i];
        if (value.m_Resource == resource)
        {
            return value.m_Amount;
        }
    }
    return 0;   // resource not present == zero held
}
```
Source: `../../../vice-and-order-research/mods/dossiers/magic-mail/repo/Systems/MagicMailSystem.cs#L428-L440` (@6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47)

```csharp
private static int AddResourceAmount(DynamicBuffer<Resources> resources, Resource resource, int amount)
{
    for (var i = 0; i < resources.Length; i++)
    {
        var value = resources[i];
        if (value.m_Resource == resource)
        {
            var newAmount = (long)value.m_Amount + amount;   // widen to avoid int overflow
            // ... clamp to int range ...
            value.m_Amount = (int)newAmount;
            resources[i] = value;          // struct copy: MUST assign back
            return value.m_Amount;
        }
    }
    resources.Add(new Resources { m_Resource = resource, m_Amount = amount });  // append if absent
    return amount;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/magic-mail/repo/Systems/MagicMailSystem.cs#L442-L472` (@6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47)

The `resources[i] = value` line is the crux: buffer elements are structs, so mutating the
local copy does nothing until you assign it back into the buffer.

## Pitfalls & gotchas

- **Buffer elements are value types - assign back or lose the write.** `var value =
  resources[i]` copies the struct. Changing `value.m_Amount` and forgetting
  `resources[i] = value` silently discards the edit (`MagicMailSystem.cs#L446-L460`).

- **A resource not in the buffer reads as zero, not an error.** The scan returns `0` when
  no element matches (`MagicMailSystem.cs#L439`), and the write path *appends* a brand-new
  element for that resource (`MagicMailSystem.cs#L465-L469`). So "add 50 mail to a
  facility that has none" creates the slot. Don't assume the resource already has a slot.

- **Amounts are `int`; unbounded adds can overflow.** Magic Mail widens to `long` before
  adding and clamps back into `int` range (`MagicMailSystem.cs#L448-L461`). If you skip
  that and repeatedly add large deltas you can wrap a stored amount negative.

- **These are main-thread edits.** Both canonical mods mutate the buffer without Burst -
  Magic Mail in a plain `foreach` over a `ToEntityArray(Allocator.Temp)`, Market-Based
  Economy with `.WithoutBurst().Run()` (see Variations). Value derivation touches managed
  singletons (`MarketEconomyManager.Instance`, mod settings), which Burst cannot use. If
  you write from a Bursted `IJobChunk` instead, your derivation code must be Burst-safe.

- **Does the game's ledger stay consistent if you edit the buffer directly?** The buffer
  is the storage of record, but whether other systems' cached totals, statistics, or
  trade flows reconcile with an out-of-band nudge is not observable from this source. The
  vanilla `EconomyUtils.AddResources` performs the same buffer scan (Market-Based Economy
  relies on it), but that it also updates any *derived* accounting is `Needs Verification
  (in-game)`. Treat large direct nudges as behaviourally unverified until tested live.

## Variations

- **Use vanilla `EconomyUtils` and an `Entities.ForEach` instead of a hand-rolled scan.**
  Market-Based Economy receives the buffer as a lambda parameter and calls the stock
  helpers `EconomyUtils.GetResources` / `EconomyUtils.AddResources` - identical scan
  semantics, but you write no loop yourself:

  ```csharp
  Entities
      .WithName("MarketProductSale")
      .WithStoreEntityQueryInField(ref m_CompanyQuery)
      .WithoutBurst() // managed helpers + diagnostics
      .ForEach((Entity entity,
                DynamicBuffer<Game.Economy.Resources> resources,
                in PrefabRef prefabRef,
                in UpdateFrame updateFrame) =>
      {
          // ... derive producedThisTick ...
          EconomyUtils.AddResources(outputResource, producedThisTick, resources);   // credit stock
          int available = EconomyUtils.GetResources(outputResource, resources);     // read stock
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/market-based-economy/repo/Economy/MarketProductSystem.cs#L70-L131` (@b83f196a36bc74388accebdeb7c81f0f35dbab37)

  It then settles a virtual sale by draining product and crediting money in the same
  buffer - note the **negative** amount to remove and `Resource.Money` as just another
  resource kind:

  ```csharp
  EconomyUtils.AddResources(outputResource, -saleAmount, resources);   // remove sold goods
  EconomyUtils.AddResources(Resource.Money, revenue, resources);       // credit the company
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/market-based-economy/repo/Economy/MarketProductSystem.cs#L159-L160` (@b83f196a36bc74388accebdeb7c81f0f35dbab37)

  The lambda runs on the main thread because it is closed with `.Run()`, not
  `.Schedule()` (`MarketProductSystem.cs#L204`), and its query declares
  `ReadWrite<Game.Economy.Resources>` (`MarketProductSystem.cs#L40-L49`).

- **Hand-roll the loop (Magic Mail) vs. call `EconomyUtils` (Market-Based Economy).** They
  are behaviourally the same buffer scan. Hand-rolling lets you add guards (Magic Mail's
  `long` overflow clamp) or avoid a dependency; `EconomyUtils` is less code and stays in
  step with any vanilla changes to the scan. Prefer `EconomyUtils` unless you need custom
  clamping.

- **Clamp to an absolute target instead of blind add.** Compute `target - current` and add
  that delta, so re-running the pass is idempotent rather than stacking - Magic Mail's
  overflow cleanup distributes a capacity-proportional target across mail types this way
  (`MagicMailSystem.cs#L298-L302`).

## See also
- Explanation: [ECS fundamentals](../../explanation/ecs-fundamentals.md) (what a
  `DynamicBuffer` is and why elements are value-type structs).
- Reference: [economy game systems](../../reference/game-systems/economy.md)
  (`Resources`, `Resource`, `EconomyUtils`).
- Related recipes: [Harmony price-getter postfix](harmony-price-getter-postfix.md)
  (adjust prices instead of stored amounts).
- Case study: [magic-mail](../../case-studies/magic-mail.md).

## Sources
- Canonical mods (dossier + repo):
  - `magic-mail` @6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47 - `repo/Systems/MagicMailSystem.cs`
  - `market-based-economy` @b83f196a36bc74388accebdeb7c81f0f35dbab37 - `repo/Economy/MarketProductSystem.cs`
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding
