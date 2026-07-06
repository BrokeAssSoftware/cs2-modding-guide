---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Reference: Unified Icon Library (UIL)"
Summary: Lookup reference for depending on Unified Icon Library (Paradox Mods id 74417) - the coui://uil/Style/Icon.svg URI scheme, the three shipped styles, CSS layer-colour tinting, and the AddHostLocation call any mod uses to register its own COUI icon host for custom art - each claim cross-checked against real mod source at a pinned commit.
diataxis: reference
source_version: "~1.5.x (unified-icon-library@b200d89; date-pinned, static source only)"
last_reverified: "2026-07-04"
canonical_mods:
  - unified-icon-library@b200d8901346457f97c032e86e1d8437a12c0cc8
  - extra-lib@4879487b7df62e83676030a87f92d4b102955eee
technique_applicability: [ui, content]
status: source-verified
Created: 2026-07-03
Updated: 2026-07-03
Owners:
  - codex
References:
  - Label: "Reference: ExtraLib (shared library)"
    Path: ./extralib.md
  - Label: "Explanation: Dependency Strategy (declare/validate/degrade)"
    Path: ../../explanation/dependency-strategy.md
  - Label: "Explanation: The Gameface UI Runtime (coui:// hosts)"
    Path: ../../explanation/gameface-runtime.md
  - Label: "Technique H - COUI icon/host registration"
    Path: ../../technique-index.md
  - Label: "Reference: External Resources (official wiki, repos)"
    Path: ../external-resources.md
  - Label: Official CS2 UI modding wiki (authoritative)
    Path: https://cs2.paradoxwikis.com/UI_Modding
---

# Reference: Unified Icon Library (UIL)

**Unified Icon Library** (UIL) is a standalone dependency mod that ships a curated catalogue of
SVG icons matching the Cities: Skylines II visual language and pre-registers them with the
Gameface UI runtime under a single COUI host named `uil`. Any dependent mod then references an
icon by URI (`coui://uil/Standard/ArrowLeft.svg`) instead of bundling its own sprites. This is
the canonical example of *shipping shared UI art as its own mod plus a host alias* so consumers
stay lean and glyph updates land in one place.

- **Paradox Mods id:** `74417` (`repo/Properties/PublishConfiguration.xml#L2`).
- **Source:** `github.com/algernon-A/UnifiedIconLibrary`.
- **Mechanism:** a single `UIManager.defaultUISystem.AddHostLocation("uil", ...)` call in
  `OnLoad` - no Harmony patching (`repo/Code/Mod.cs#L86`).

**Scope of verification.** Every claim below was confirmed in real mod source at a pinned
commit (citations inline). UIL is pinned at `unified-icon-library@b200d890` (tag v1.0.14) and
the peer host pattern at `extra-lib@4879487`. Behaviour that lives in the closed-source
`Colossal.UI` assembly (for example duplicate-host collision semantics) cannot be confirmed
from mod source and is labelled **Needs Verification**. For the authoritative COUI/Gameface
documentation, link out to the [CS2 UI modding wiki](https://cs2.paradoxwikis.com/UI_Modding)
rather than trusting a duplicate here.

Corpus and pinned commits used for verification:

- `unified-icon-library` @`b200d89` - `repo/Code/Mod.cs`, `repo/README.md`, `repo/Properties/PublishConfiguration.xml`, `repo/Icons/`
- `extra-lib` @`4879487` - `repo/MOD/Helpers/Icons.cs` (three-arg host overload with teardown)

## Declaring the dependency

UIL is a **hard dependency**: a consumer that references `coui://uil/...` URIs compiles fine
without UIL, but the icons render blank at runtime if it is not installed and loaded first.
Declare it so the in-game publisher offers to install it for players.

| Where | What to write | Verified basis |
| --- | --- | --- |
| `Properties/PublishConfiguration.xml` | A `<Dependency Id="74417" />` entry (numeric Paradox Mods id). | The reciprocal is verified in a real consumer: `extra-assets-importer/repo/Properties/PublishConfiguration.xml#L81` declares `<Dependency Id="75724" />` for its own library dependency (same numeric-id mechanism). |
| README / release notes | Mark UIL as **Required** so players understand the dependency chain. | Convention. |

Reference the library by **both name and id** (`Unified Icon Library`, id `74417`) in prose so
readers can find it unambiguously. A UI package's `mod.json` may also carry a string dependency
identifier for the library; the exact token and casing are **Needs Verification** - take them
from the current UIL release, do not hard-code a guessed value.

For the full declare / reference / validate / degrade strategy (and why each step matters), see
the concept page [Dependency Strategy](../../explanation/dependency-strategy.md).

### Validating at runtime (degrade gracefully)

Declaring the dependency does not guarantee UIL loaded before your code. To detect its absence
and fall back to text or neutral glyphs, use the runtime assembly-detection pattern documented
(with a source-cited example) in
[Dependency Strategy - validate at runtime](../../explanation/dependency-strategy.md): scan
`AppDomain.CurrentDomain.GetAssemblies()` for the library assembly, tolerate
`ReflectionTypeLoadException`, and record the result once. That page carries the canonical cited
implementation; this page does not duplicate it.

## Referencing library icons (coui:// URI scheme)

UIL pre-injects its `/Icons` directory into the UI runtime as the `uil` host, so every shipped
SVG is addressable by URI without any per-icon registration
(`repo/Code/Mod.cs#L86`, `repo/README.md#L14`):

```
coui://uil/<Style>/<IconName>.svg
```

Typically this URI forms the `src` attribute of a UI element, for example
`src="coui://uil/Standard/ArrowLeft.svg"` (`repo/README.md#L14`). The general form is
`coui://<host>/<relative-path>`, where `<host>` is whatever string was passed to
`AddHostLocation` and `<relative-path>` is resolved against the registered directory.

### Styles

`<Style>` maps directly to a folder under `/Icons/`. UIL ships three
(`repo/Icons/Standard/`, `repo/Icons/Dark/`, `repo/Icons/Colored/`; `repo/README.md#L20`):

| Style | Purpose |
| --- | --- |
| `Standard` | Matches the vanilla CS2 UI chrome. |
| `Dark` | Higher-contrast dark-theme variant. |
| `Colored` | Saturated multi-colour glyphs for toolbars. |

Confirm a specific icon name exists in the chosen style before hard-coding its URI; UIL's
catalogue grows over time (v1.0.14 ships 268 icons per style) and a misspelled name resolves to
nothing rather than erroring.

### Tinting the Colored set (CSS layer colours)

The `Colored` icons expose named layer-colour IDs you can override in CSS to recolour every
coloured icon globally, without editing the SVGs (`repo/README.md#L22-L33`). Verified IDs:

`black`, `white`, `grey`, `brown`, `violet`, `red`, `orange`, `yellow`, `green`, `blue`

Explicit hex colour codes also work against the same single palette (`repo/README.md#L35`).
Single-tone `Standard`/`Dark` icons carry no colour layers; recolour them with a CSS `filter`
or `fill` override instead. Re-validate contrast (target 4.5:1) after any custom tint.

## Registering your own COUI host (shipping custom icons)

When UIL lacks an icon you need, register your **own** COUI host for your mod's SVGs. Both
canonical libraries do this with the same runtime API - `UIManager.defaultUISystem` -
differing only in overload and teardown. Use a **unique host key** (not `uil`) to avoid
clashing with other mods.

### The two AddHostLocation overloads (verified)

| Overload | Signature (as called) | Watch / hot-reload | Teardown | Verified usage |
| --- | --- | --- | --- | --- |
| Two-arg | `AddHostLocation(hostName, path)` | No | None (host persists for process lifetime) | UIL: `UIManager.defaultUISystem.AddHostLocation("uil", AssemblyPath + "/Icons/")` (`unified-icon-library/repo/Code/Mod.cs#L86`) |
| Three-arg | `AddHostLocation(hostName, path, shouldWatch)` | `shouldWatch: true` re-detects files dropped into the folder at runtime | `RemoveHostLocation(uri)` / `RemoveHostLocation(uri, path)` | ExtraLib: `AddHostLocation(uri, path, shouldWatch)` inside `LoadIconsFolder` (`extra-lib/repo/MOD/Helpers/Icons.cs#L36`), torn down in `UnLoadIconsFolder` (`#L47`, `#L53`) |

Resolve your mod's on-disk directory first, then point the host at your icons subfolder:

- UIL caches the executable asset path via
  `AssetDatabase.global.GetAsset(SearchFilter<ExecutableAsset>.ByCondition(...))` and
  `Path.GetDirectoryName(modAsset.GetMeta().path)`
  (`unified-icon-library/repo/Code/Mod.cs#L38-L60`).
- ExtraLib resolves it via
  `GameManager.instance.modManager.TryGetExecutableAsset(this, out var asset)` then
  `new FileInfo(asset.path).Directory.FullName`
  (`extra-lib/repo/EL.cs#L49-L59`).

Reference your custom icons through your own host: `coui://<your-host>/<Style>/<Icon>.svg`.

### Choosing the overload

- **Static catalogue, simplest wiring:** the two-arg overload (UIL's approach). Note UIL's
  `OnDispose` clears its instance but never calls `RemoveHostLocation`, so the `uil` host - and
  by extension a two-arg host you register the same way - **leaks for the process lifetime**
  after a hot-disable (`unified-icon-library/repo/Code/Mod.cs#L92-L96`). Icons dropped into the
  folder after mount do **not** appear, because watching is off.
- **Hot-reloadable or removable:** the three-arg overload with `shouldWatch: true` and explicit
  teardown (ExtraLib's `Icons.LoadIconsFolder` / `UnLoadIconsFolder`). ExtraLib also dedups
  registrations in a `Dictionary<string, List<string>>` so the same path is not mounted twice
  and supports multiple paths per host (`extra-lib/repo/MOD/Helpers/Icons.cs#L23-L57`).

If several mods need the same custom glyph, promote it into a shared library (see
[ExtraLib](./extralib.md)) rather than duplicating the SVG per mod.

## Needs Verification

| Claim | Why unverified |
| --- | --- |
| Duplicate-host collision semantics (overwrite vs. throw vs. no-op when two mods register the same host name) | Lives in the closed-source `Colossal.UI` assembly; cannot be confirmed from mod source. A same-host survey found UIL is currently the only mod shipping the `uil` host and ExtraLib defaults to `extralib`, so collisions are unlikely but untested. |
| `coui://uil` URIs resolving *after* UIL is hot-disabled mid-session | UIL never removes its host, so the registration persists, but whether the URIs still resolve after disable is an in-game behaviour not confirmable from source. |
| The string dependency identifier for a UI package `mod.json` | Not present in the pinned corpus; take name/casing from the current UIL release. |

## See also

- [ExtraLib](./extralib.md) - a shared library that bundles an icon host (`coui://extralib`)
  alongside localisation, notification, and prefab helpers, and is the canonical three-arg
  `AddHostLocation` example.
- [Dependency Strategy](../../explanation/dependency-strategy.md) - declare / reference /
  validate / degrade, with the cited runtime assembly-detection guard.
- [The Gameface UI Runtime](../../explanation/gameface-runtime.md) - how `coui://` hosts and the
  Gameface layer fit together.
- [Technique Index](../../technique-index.md) - family H (COUI icon/host registration via
  `AddHostLocation`).
- [External Resources](../external-resources.md) - UIL repository, the official UI modding wiki,
  and related libraries.
- Official [CS2 UI modding wiki](https://cs2.paradoxwikis.com/UI_Modding) - authoritative
  COUI/Gameface reference; link out rather than duplicating.
