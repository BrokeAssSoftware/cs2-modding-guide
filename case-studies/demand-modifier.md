---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Case study: Demand Modifier (research-hygiene cautionary)"
case_study: demand-modifier
mod: "Demand Modifier"
dossier: ../../vice-and-order-research/mods/dossiers/demand-modifier/
repo_commit: e93ec1c1acdb58764c7949c4fca1654d451b690b
source_version: "~1.5.x (v0.3.2 source; live ECS gated) (demand-modifier@e93ec1c; date-pinned, static source only)"
last_reverified: "2026-07-05"
diataxis: explanation
techniques: [N]
technique_applicability: [economy, core]
status: source-verified
Created: 2026-07-05
Updated: 2026-07-05
Owners:
  - codex
---

# Demand Modifier - case study (research-hygiene cautionary)

> A mod whose *public* source tree is a frozen, non-compiling v0.3.2 snapshot -
> declarative Options UI, an enum whose literals are demand bytes, verbose
> bilingual bootstrap logging, and a Harmony `PatchAll` that patches nothing -
> while the *storefront* advertises an unrelated ECS rewrite. It is the
> handbook's worked example of why you verify content, not the listing.

All code claims below are cited to `repo/<path>#Lxx` at pinned commit
`e93ec1c` (author ChengBoChuan, Paradox ModId 123170 "Demand Modifier (BETA)"),
surfaced through the dossier at
`../../vice-and-order-research/mods/dossiers/demand-modifier/`. Read this page
alongside its companion, [Research Hygiene](../explanation/research-hygiene.md),
which uses the same mod as its case-in-point.

> **Gating note (read first).** The current live listing is an unpublished ECS
> rewrite marketed under an "Infinite Resources" line (author-claimed v0.5.2,
> ~7 resource systems). None of that is present in this source tree, does not
> resolve to any file at this pin, and is **`Needs Verification (in-game)`** -
> it is blocked on decompiling the shipped DLL. Everything asserted as fact
> below is confirmed against the v0.3.2 Harmony/dropdown source only. The live
> resource-control mechanism is *not* described here as if verified.

## What it does / why it's instructive

Demand Modifier's stated purpose is a sandbox aid: pin residential, commercial,
and industrial demand to a chosen level, plus (advertised, not implemented here)
unlimited-service and unlimited-economy toggles. As a *feature* it is
unremarkable. As a *teaching artifact* it is valuable for the opposite reason to
most case studies: it is a compact catalogue of how a mod's public source can
diverge from both its own documentation and its storefront, and how to write an
honest page anyway.

Four things make it instructive, and only the first two are "how you'd want to
do it":

1. A three-tab Options panel built entirely from `SettingsUI*` attributes - zero
   UI code.
2. An enum whose literal values *are* the game's 0-255 demand byte, so the
   dropdown choice needs no lookup table.
3. A four-phase, try/catch-guarded, bilingual bootstrap log - defensive to a
   fault, and pointing at helper types that are **not in the tree**.
4. A Harmony `PatchAll` that succeeds while patching nothing, because every
   `[HarmonyPatch]` class the README describes has been excised.

Points 3 and 4 are why the public tree **does not compile**, and why the frozen
source cannot be the build the storefront ships. That gap is the lesson.

## Architecture at a glance

There is no ECS system in this tree. The entire mod is two source files - a
`ModSetting` subclass and an `IMod` entry point - plus a project file. Confirmed
by the pinned tree: the only `.cs` files at `e93ec1c` are
`repo/DemandModifier/DemandModifierMod.cs` and
`repo/DemandModifier/DemandModifierSettings.cs`.

### Load-time bootstrap (`IMod.OnLoad`)

`DemandModifierMod.OnLoad` runs four independently try/catch-guarded phases, each
narrating itself in mixed Traditional-Chinese/English with emoji markers
(repo/DemandModifier/DemandModifierMod.cs#L43-L147):

1. Advanced logger init - `Utils.Logger.Initialize(...)`
   (repo/DemandModifier/DemandModifierMod.cs#L51-L63).
2. Localization init - `LocalizationInitializer.Initialize()`
   (repo/DemandModifier/DemandModifierMod.cs#L65-L82).
3. Settings - construct, `RegisterInOptionsUI()`, then
   `AssetDatabase.global.LoadSettings(...)`
   (repo/DemandModifier/DemandModifierMod.cs#L84-L118).
4. Harmony - `new Harmony(id)` then `harmony.PatchAll(assembly)`
   (repo/DemandModifier/DemandModifierMod.cs#L120-L143).

`OnDispose` mirrors it: `UnpatchAll(id)` then `UnregisterInOptionsUI()`
(repo/DemandModifier/DemandModifierMod.cs#L152-L204). There is no
`UpdateSystem.UpdateAt`/`UpdateAfter` scheduling anywhere - the `updateSystem`
parameter is accepted and ignored. This mod registers no ECS phase work at all.

### The Options model (`ModSetting` subclass)

`DemandModifierSettings` declares the whole UI declaratively. Class-level
attributes lay out three tabs and five named groups
(repo/DemandModifier/DemandModifierSettings.cs#L28-L35):

```csharp
[FileLocation(nameof(DemandModifier))]
[SettingsUITabOrder("DemandControl", "ServiceControl", "EconomyControl")]
[SettingsUIGroupOrder(
    "ResidentialDemand", "CommercialDemand", "IndustrialDemand",
    "ServiceSettings", "EconomySettings")]
[SettingsUIShowGroupName(
    "ResidentialDemand", "CommercialDemand", "IndustrialDemand",
    "ServiceSettings", "EconomySettings")]
public class DemandModifierSettings : ModSetting
```
(repo/DemandModifier/DemandModifierSettings.cs#L28-L36)

Each property then names its `(tab, group)` and, for the three demand levels, a
dropdown source method (repo/DemandModifier/DemandModifierSettings.cs#L75-L91).
The service and economy tabs are plain `bool` toggles
(repo/DemandModifier/DemandModifierSettings.cs#L177-L240). No `RegisterInOptionsUI`
call needs to know any of this - the attributes drive the panel.

### The enum that carries the value

`DemandLevel` is the mechanism's core idea: the enum literal is the demand byte,
not an index into one (repo/DemandModifier/DemandModifierSettings.cs#L14-L21):

```csharp
public enum DemandLevel
{
    Off = 0,           // Off (use the game default)   [comment translated from zh]
    Low = 64,          // Low demand (25%)
    Medium = 128,      // Medium demand (50%)
    High = 192,        // High demand (75%)
    Maximum = 255      // Maximum demand (100%)
}
```
(repo/DemandModifier/DemandModifierSettings.cs#L14-L21; the source comments are in Chinese, translated to English here for ASCII cleanliness)

The dropdown is populated by `GetDemandLevelOptions`, which walks the five
literals in display order and pairs each with a localized name (via
`GetLocalizedEnumName`, which falls back to hard-coded English like
`"Maximum (100%)"` at repo/DemandModifier/DemandModifierSettings.cs#L154-L169) -
so the *chosen* `DemandLevel` value would be exactly the number a consumer writes
into the demand system
(repo/DemandModifier/DemandModifierSettings.cs#L97-L127).

> **`Needs Verification (in-game)`:** at this pin there is *no consumer*. The
> byte is never read by any patch or system in the tree (the intended reader,
> `DemandSystemPatch.cs`, is gone - see below). "The dropdown choice IS the
> demand the game receives" is the design *intent*, source-verified as an enum
> shape but **not** as a wired-up effect. Whether selecting `Maximum` changes
> in-game demand cannot be confirmed from this source.

## Techniques demonstrated

- [Settings UI patterns](../how-to/recipes/settings-patterns.md) (family N) -
  a full three-tab, five-group Options panel assembled from `SettingsUITabOrder`
  / `SettingsUIGroupOrder` / `SettingsUIShowGroupName` at the class and
  `SettingsUISection` / `SettingsUIDropdown` per property, with **no UI code**
  (repo/DemandModifier/DemandModifierSettings.cs#L28-L91). This is the one
  technique here worth copying verbatim.
- [Settings UI patterns](../how-to/recipes/settings-patterns.md) (family N),
  enum-as-value variant - `DemandLevel {Off=0..Maximum=255}` makes the dropdown
  selection double as the payload, so no value-mapping table is needed
  (repo/DemandModifier/DemandModifierSettings.cs#L14-L21,
  repo/DemandModifier/DemandModifierSettings.cs#L97-L127). See the
  Needs-Verification note - the payload is never consumed at this pin.
- [Localization helper](../how-to/recipes/localization-helper.md) - the dropdown
  reads `activeDictionary` by key
  (`Common.ENUM[DemandModifier.DemandModifier.DemandLevel.*]`) and degrades to
  hard-coded English when a key is missing
  (repo/DemandModifier/DemandModifierSettings.cs#L133-L169). Note the localizer
  *bootstrap* it depends on (`LocalizationInitializer`) is not in the tree.
- [Mod lifecycle](../explanation/mod-lifecycle.md) - a defensively phased
  `OnLoad`: four try/catch blocks so one failing subsystem cannot abort the rest,
  each logging entry/exit bilingually
  (repo/DemandModifier/DemandModifierMod.cs#L43-L147). Instructive as *shape*;
  see the anti-pattern below for why it still won't build.
- [Harmony patching](../explanation/harmony-patching.md), anti-pattern -
  `harmony.PatchAll(typeof(DemandModifierMod).Assembly)` runs and logs success
  even though the assembly contains **zero** `[HarmonyPatch]` classes
  (repo/DemandModifier/DemandModifierMod.cs#L131). `PatchAll` over a
  patch-free assembly is a silent no-op, so the mod's "all patches applied" success
  log line (a Chinese string in source) is true and meaningless.

## Key decisions & tradeoffs

- **Attribute-only UI is the right call.** For a settings-only mod, deriving one
  `ModSetting` and decorating properties is strictly better than hand-rolled
  React/COUI - it is localizable, persists via `[FileLocation]` +
  `AssetDatabase.global.LoadSettings`, and needs no UI-thread code
  (repo/DemandModifier/DemandModifierSettings.cs#L28-L36,
  repo/DemandModifier/DemandModifierMod.cs#L93-L98). Copy this part.
- **Enum-literals-as-value is elegant but fragile.** Encoding 0/64/128/192/255
  directly in the enum removes a mapping table, but it silently couples the UI
  enum to the game's demand scale; if Colossal ever changed the demand range, the
  literals - and their `(25%)`/`(50%)` labels - would be wrong with no compiler
  signal (repo/DemandModifier/DemandModifierSettings.cs#L14-L21).
- **Defensive logging taken past the point of value.** Four try/catch phases and
  dozens of log lines make failures legible, but they also mask a hard truth: the
  phases call into `Utils.Logger` and `DemandModifier.Localization`
  (repo/DemandModifier/DemandModifierMod.cs#L3, and the `Utils.Logger.*` calls at
  L49/L55/L69/L135) whose defining files are **absent from the tree**. The
  logging survives a runtime exception but not the *compiler* - the references are
  unresolved symbols.
- **`PatchAll` with no patches is the tell.** Harmony is a declared dependency
  (`Lib.Harmony 2.2.2`, repo/DemandModifier/DemandModifier.csproj#L16) and is
  wired through load and dispose, yet nothing in the assembly carries
  `[HarmonyPatch]`. The plumbing is real; the payload was removed. That is the
  signature of a tree that was gutted mid-refactor, not one that was finished.

## Pitfalls / upstream-watch (the research-hygiene core)

This is the reason the mod is in the handbook. Every item below is a *documented
divergence* between what a source of information claims and what the pinned tree
actually contains.

- **The public tree does not compile.** `OnLoad` references three types with no
  defining file at this pin: `Utils.Logger`
  (repo/DemandModifier/DemandModifierMod.cs#L49), `LocalizationInitializer`
  (repo/DemandModifier/DemandModifierMod.cs#L70), and the
  `DemandModifier.Localization` namespace (repo/DemandModifier/DemandModifierMod.cs#L3).
  The tree's only `.cs` files are `DemandModifierMod.cs` and
  `DemandModifierSettings.cs`. Therefore this source *cannot* be the shipping
  build - a critical thing to state before citing any behaviour from it.
- **The README documents a file that was deleted.** `DemandModifier/README.md`'s
  file guide describes `DemandSystemPatch.cs` as "containing three Harmony patch
  classes: `CommercialDemandSystemPatch`, `IndustrialDemandSystemPatch`,
  `ResidentialDemandSystemPatch`" (repo/DemandModifier/README.md). No such file
  or class exists at `e93ec1c` (a tree grep for `DemandSystemPatch` matches only
  docs, never a `.cs`). This is precisely the demand-writing mechanism a reader
  would want - and it is the excised payload that leaves `PatchAll` empty.
- **Version and game-version stamps disagree across four sources:**
  - source constant `MOD_VERSION = "0.3.2"`
    (repo/DemandModifier/DemandModifierMod.cs#L30);
  - `PublishConfiguration.xml`: `ModVersion 0.3.2`
    (repo/DemandModifier/Properties/PublishConfiguration.xml#L82) but
    `GameVersion 1.3.*`
    (repo/DemandModifier/Properties/PublishConfiguration.xml#L84), `DisplayName
    "Demand Modifier (BETA)"` and `ModId 123170`
    (repo/DemandModifier/Properties/PublishConfiguration.xml#L4-L6);
  - `DemandModifier/README.md` claims game compatibility **v1.2.\*** and target
    framework **.NET 4.8.1** (repo/DemandModifier/README.md);
  - the actual project targets **net472**
    (repo/DemandModifier/DemandModifier.csproj#L8);
  - the root README's install link points at a **different** Paradox ModId,
    123136, than the publish config's 123170 (repo/README.md).
  No single stamp can be trusted; the source constant + publish config
  (`0.3.2`, game `1.3.*`, id `123170`) are the closest to authoritative because
  they live next to the code, but even they disagree on game version with the
  README.
- **The storefront describes a different mod than the source.** The live listing
  advertises an ECS "Infinite Resources" rewrite with multiple resource systems;
  none of that exists at this pin. Writing that mechanism up from the listing
  would fabricate a source-verified page. It stays
  **`Needs Verification (in-game)`** pending a DLL decompile - the honest archive
  of a real gap, not an invented answer.

**Key lesson.** Three independent artifacts - the README, the publish config, and
the storefront - each assert something the pinned code contradicts. The only
reliable witness is the tree read at the pin. Verify *content*, at a commit;
treat every listing, tag, and README as a claim to be checked, never as evidence.

## Source pointers

- Companion explanation:
  [Research Hygiene](../explanation/research-hygiene.md) (uses this same mod as
  its worked example; `canonical_mods: demand-modifier@e93ec1c`).
- Dossier:
  `../../vice-and-order-research/mods/dossiers/demand-modifier/`
  (`index.md`, `source.md`, `modding.md`, `guide.md`, `notes/`).
- Repo @ `e93ec1c`, the entire relevant surface:
  - `repo/DemandModifier/DemandModifierSettings.cs` - `ModSetting` subclass,
    `DemandLevel` enum, dropdown source + localization fallback.
  - `repo/DemandModifier/DemandModifierMod.cs` - `IMod` entry point, four-phase
    bootstrap, empty `PatchAll`, references to absent helper types.
  - `repo/DemandModifier/DemandModifier.csproj` - `net472`, `Lib.Harmony 2.2.2`.
  - `repo/DemandModifier/Properties/PublishConfiguration.xml` - the disagreeing
    version/game/id stamps.
  - `repo/DemandModifier/README.md`, `repo/README.md` - documentation that names
    a deleted patch file and a mismatched ModId.
