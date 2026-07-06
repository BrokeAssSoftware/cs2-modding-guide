---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Case study: Achievement Fixer"
case_study: achievement-fixer
mod: "Achievement Fixer (121256)"
dossier: ../../vice-and-order-research/mods/dossiers/achievement-fixer/
repo_commit: 4d3a1eccc64a7a09de65a23f6abbb3daa998fd0d
source_version: "n/a - no pinned source"
last_reverified: "2026-07-04"
diataxis: explanation
techniques: [A, N, O]
technique_applicability: [core, ui]
status: source-verified
Created: 2026-07-03
Updated: 2026-07-03
Owners:
  - codex
---

# Achievement Fixer - case study

> Achievement Fixer keeps Cities: Skylines II achievements enabled under mods without
> Harmony or reflection: one short-lived system re-asserts a single platform flag for
> 1,800 frames after each load, and one install-once localization override rewrites the
> "achievements disabled" banner per locale. It is the cleanest example of solving a
> real problem with the smallest possible patch surface.

All code claims below are cited to `repo/<path>#Lxx` at pinned commit
`4d3a1eccc64a7a09de65a23f6abbb3daa998fd0d` (branch `main`, modVersion 21 /
userModVersion 1.2.6, "Updated for 1.6 game patch"), surfaced through the dossier at
`../../vice-and-order-research/mods/dossiers/achievement-fixer/`. That dossier corrected
earlier passes that described a per-frame banner reapply and an
`onActiveDictionaryChanged` subscription: both existed in an older commit but were
removed in the v1.2.3+ "simplified internal localization" refactor and are **not**
present at this commit. None are reintroduced here.

## What it does / why it's instructive

Cities: Skylines II disables Steam/PDX achievements while code mods are active. Players
who run curated, non-cheating mod lists still want progression to count. Achievement
Fixer (author RiverMochi, Paradox ModId 121256) forces the game's internal achievement
eligibility flag back on and replaces the scary "achievements are disabled because of
mods" banner with a friendly "achievements enabled" message. No configuration is
required: install it, keep it enabled, and it works.

It is an instructive teaching example precisely because of how *little* it does:

1. **No Harmony, no reflection (beyond reading its own version string).** The mod never
   patches a vanilla method. It writes one public property (`PlatformManager.
   achievementsEnabled`) and registers localization sources through the sanctioned
   `LocalizationManager.AddSource` API. That is the entire mechanism. For a mod whose
   whole job is to survive every CS2 patch, choosing the lowest-drift surface available
   is the design.
2. **Windowed enforcement instead of a one-shot toggle.** Setting the flag once at load
   loses to late-loading mods and delayed platform callbacks. The mod instead keeps a
   system scheduled for a bounded window and re-asserts the flag every frame, then goes
   idle - a general pattern for "hold a value true through a noisy startup."
3. **Install-once, last-writer-wins localization override.** A single-key
   `IDictionarySource` replaces just the banner string, registered once per locale at
   load. The localization manager replays it on every dictionary rebuild (including
   language switches), so no event subscription is needed.

## Architecture at a glance

`Mod.OnLoad` is the whole setup, in order (`repo/Mod.cs#L43-L77`):

1. Construct `Settings` before locales so option labels resolve
   (`repo/Mod.cs#L53-L54`).
2. Register twelve UI locale sources (en-US, fr-FR, de-DE, es-ES, it-IT, ja-JP, ko-KR,
   vi-VN, pl-PL, pt-BR, zh-HANS, zh-HANT) via `AddLocaleSource`, then call
   `AddWarningOverrideSources()` to install the banner override once per locale
   (`repo/Mod.cs#L57-L69`).
3. Load persisted settings from `AssetDatabase.global.LoadSettings("AchievementFixer",
   ...)` and register the Options UI (`repo/Mod.cs#L72-L73`).
4. Schedule the enforcement system *after* the vanilla `AchievementTriggerSystem` in the
   main loop so the mod re-asserts immediately after the game's own achievement pass:
   `updateSystem.UpdateAfter<AchievementFixerSystem, AchievementTriggerSystem>(
   SystemUpdatePhase.MainLoop)` (`repo/Mod.cs#L76`).

There is exactly one gameplay system and two static helper classes; there is no
Harmony bootstrapping anywhere in `OnLoad`.

### The flag enforcement is frame-windowed, then idle

`AchievementFixerSystem` starts disabled in `OnCreate` so it costs nothing until a real
gameplay load (`repo/Systems/AchievementFixerSystem.cs#L20-L32`). `OnGameLoadingComplete`
skips menu/editor loads (`if (mode != GameMode.Game) { Enabled = false; return; }`),
then opens a fixed window `m_FramesLeft = kAssertFrames` (1,800 frames) and enforces
immediately (`repo/Systems/AchievementFixerSystem.cs#L34-L58`). `OnUpdate` disables
itself once the window closes and otherwise re-asserts every frame and decrements the
counter (`repo/Systems/AchievementFixerSystem.cs#L60-L84`):

```csharp
protected override void OnUpdate()
{
    if (m_FramesLeft <= 0) { Enabled = false; return; }   // window closed -> idle
    ForceEnableIfNeeded("OnUpdate");                       // keep flag true every frame
    m_FramesLeft--;
}
```

`ForceEnableIfNeeded` only writes when the game has actually flipped the flag false, and
logs that flip-and-fix in Release as player-visible proof
(`repo/Systems/AchievementFixerSystem.cs#L86-L107`):

```csharp
if (!pm.achievementsEnabled)
{
    Mod.s_Log.Info($"{source}: ATTN: detected game flipped achievementsEnabled == FALSE. Forcing TRUE now");
    pm.achievementsEnabled = true;
    return true;
}
```

`kAssertFrames = 1800` is ~30 s at 60 FPS or ~60 s at 30 FPS
(`repo/Systems/AchievementFixerSystem.cs#L15`) - long enough to outlast a heavy mod
load, short enough to stop touching the flag once the city is stable.

### The banner override is installed once per locale

`AddWarningOverrideSources` iterates `LocaleBannerText.LocaleIds` and calls
`EnsureWarningOverrideFor` for each (`repo/Mod.cs#L130-L136`). That helper builds a
one-key dictionary mapping `Menu.ACHIEVEMENTS_WARNING_MODS` to the localized banner and
registers it as a `LocaleOverrideSource` - a tiny `IDictionarySource` whose `ReadEntries`
returns just that dictionary (`repo/Mod.cs#L141-L167`,
`repo/Locale/AchievementLocaleHelpers.cs#L46-L64`). A code comment states the intent
directly: adding the sources once "lets the localization manager rebuild normally when
players switch languages, without this mod subscribing to onActiveDictionaryChanged"
(`repo/Mod.cs#L124-L128`). The enforcement system does **not** touch the banner - the two
concerns are fully separate.

### Every AddSource is guarded

Both the UI locales and the banner override route through `TryAddLocaleSource`, which
null-checks the `LocalizationManager` and wraps `AddSource` in try/catch so a fragile
third-party localization hook (for example an I18n Everywhere null-ref during
`AddSource`) logs a warning instead of becoming a global crash
(`repo/Mod.cs#L97-L122`). A locale is only recorded in `s_InstalledLocales` after a
successful add (`repo/Mod.cs#L149-L166`).

## Techniques demonstrated

- [Prefab-field override](../how-to/recipes/prefab-field-override.md) (family A) - the
  *field-override* family, in its platform-flag variant. The canonical family-A form
  resolves a prefab and commits a component via `PrefabSystem.AddComponentData`;
  Achievement Fixer instead overrides a value the game owns by writing the managed
  singleton property `PlatformManager.achievementsEnabled = true` in a windowed system
  (`repo/Systems/AchievementFixerSystem.cs#L97-L104`). Same intent - re-assert a
  game-owned value from an immutable target (here the literal `true`, so re-applying
  never drifts) - but no ECS prefab data and no Harmony. Treat it as the degenerate,
  no-ECS end of the field-override spectrum, not a canonical prefab example.
- [Settings patterns (section / dropdown / confirmation)](../how-to/recipes/settings-patterns.md)
  (family N) - the Options UI is a two-tab `ModSetting` with declared group order and
  shown-group names (`repo/Settings/Settings.cs#L13-L36`). The Advanced tab exposes a
  data-driven dropdown populated from `PlatformManager.EnumerateAchievements()` via
  `[SettingsUIDropdown(typeof(Settings), nameof(GetAchievementChoices))]`
  (`repo/Settings/Settings.cs#L132-L134`, `#L282-L303`) and three action buttons.
  Destructive actions are gated by `[SettingsUIConfirmation]` (Clear and Reset All);
  Unlock is not (`repo/Settings/Settings.cs#L184-L187`, `#L239-L242`). This shows the
  section/group/dropdown/confirmation subset of family N (it uses no slider or
  hide-by-condition - those live in other family-N mods).
- [Localization: multi-locale registration](../how-to/recipes/localization-helper.md)
  (family O) - twelve UI locale sources are registered at load
  (`repo/Mod.cs#L57-L68`), and a single game key is overridden per locale through a
  purpose-built `LocaleOverrideSource : IDictionarySource` so the mod replaces one string
  without forking a whole dictionary (`repo/Locale/AchievementLocaleHelpers.cs#L46-L64`).
  Friendly achievement titles are resolved from the game's own active dictionary key
  `Achievements.TITLE[<internalName>]`, falling back to the raw internal name
  (`repo/Locale/AchievementLocaleHelpers.cs#L15-L39`).

See the [technique index](../technique-index.md) for the full family ledger. Supporting
lifecycle and settings context lives in [mod lifecycle](../explanation/mod-lifecycle.md),
[system scheduling](../explanation/system-scheduling.md),
[conditional execution](../explanation/conditional-execution.md), and
[settings and data](../explanation/settings-and-data.md).

## Key decisions & tradeoffs

- **Property write vs. Harmony patch.** The mod could have patched whatever vanilla code
  flips the flag. Instead it writes the public `PlatformManager.achievementsEnabled`
  property directly (`repo/Systems/AchievementFixerSystem.cs#L101`). The cost is that it
  must *keep* writing (a single write can be overwritten later); the benefit is zero
  patch-drift risk, which is the whole point for a compatibility shim that must ship on
  the day of every game patch.
- **Windowed enforcement vs. a one-shot toggle.** Re-asserting every frame for 1,800
  frames absorbs delayed flips from heavy load orders and late-loading mods, then idles
  so there is no steady-state cost (`repo/Systems/AchievementFixerSystem.cs#L60-L84`).
  The tunable `kAssertFrames` is the single knob if future DLC extends load times
  (`repo/Systems/AchievementFixerSystem.cs#L15`).
- **Install-once localization vs. event-driven reapply.** Registering each source once
  and letting the localization manager replay it on rebuild is simpler and crash-safer
  than subscribing to `onActiveDictionaryChanged` and reinstalling on every change - the
  mod explicitly dropped that subscription (`repo/Mod.cs#L124-L136`). The tradeoff is
  last-writer-wins: another localization mod that registers the same key *after*
  Achievement Fixer can win until the next rebuild.
- **Guarded registration vs. trusting AddSource.** Wrapping every `AddSource` in
  try/catch trades a few lines of boilerplate for immunity to a fragile third-party
  localization hook turning into a global NRE (`repo/Mod.cs#L111-L121`).
- **Confirmation only where it is destructive.** Unlock fires immediately; Clear and
  Reset All require a modal (`repo/Settings/Settings.cs#L184-L187`, `#L239-L242`). The
  Advanced tab calls `PlatformManager` APIs directly - a QA control panel, not a
  gameplay feature.

## Pitfalls / upstream-watch

- **Shared flag, last-writer-wins.** Because the enforcement writes a global platform
  property, another mod that toggles `achievementsEnabled` can fight it; the listing
  advises removing redundant achievement mods
  (`repo/Systems/AchievementFixerSystem.cs#L97-L104`).
- **Banner override is install-once and last-writer-wins.** If another localization mod
  registers `Menu.ACHIEVEMENTS_WARNING_MODS` for the active locale *after* Achievement
  Fixer (for example I18n Everywhere loaded later), it can win until the dictionary
  rebuilds. `TryAddLocaleSource` guards against exceptions but does not enforce priority
  (`repo/Mod.cs#L141-L167`). `Needs Verification (in-game)`: whether the override still
  wins on CS2 1.6.* when other localization mods load afterward.
- **Assert-window coverage.** A load heavier or slower than ~1,800 frames could let a
  flip land after the window closes; the observable Release signal is the
  `ForceEnableIfNeeded` flip-and-fix log lines. Bumping `kAssertFrames` is the fix
  (`repo/Systems/AchievementFixerSystem.cs#L60-L107`). `Needs Verification (in-game)`:
  a real log capture plus wall-clock load time requires a running CS2 session.
- **Banner glyph integrity.** Non-English banner strings are stored as raw UTF-8 glyphs
  in `LocaleBannerText.s_Text` (the older `\u`-escape encoding was dropped in the
  simplified-localization refactor), so source files must stay UTF-8 to avoid mojibake;
  the en-US string is `"Achievements enabled by Achievement Fixer."`
  (`repo/Locale/AchievementLocaleHelpers.cs#L71-L85`). `Needs Verification (in-game)`:
  that JP/KR/ZH glyphs render correctly on CS2 1.6.*.
- **Advanced-tab governance.** Unlock/Clear/Reset call `PlatformManager` immediately
  (Clear/Reset behind a confirm modal); restrict the tab to QA and delete
  `ModsSettings/AchievementFixer/AchievementFixer` between sessions to clear a stale
  selection (`repo/Settings/Settings.cs#L129-L277`, `#L335-L338`).

## Source pointers

- Dossier: `../../vice-and-order-research/mods/dossiers/achievement-fixer/`
  (index / source / modding / guide + notes).
- Repo @ `4d3a1eccc64a7a09de65a23f6abbb3daa998fd0d` (branch `main`), key files:
  - `repo/Mod.cs` - `OnLoad`, twelve locale registrations, install-once banner override,
    guarded `TryAddLocaleSource`, `UpdateAfter` scheduling.
  - `repo/Systems/AchievementFixerSystem.cs` - frame-windowed flag enforcement.
  - `repo/Settings/Settings.cs` - two-tab Options UI, dropdown, confirmation-gated
    actions.
  - `repo/Locale/AchievementLocaleHelpers.cs` - `LocaleOverrideSource`,
    `LocaleBannerText`, `AchievementDisplay`.
- Storefront: mod 121256, version 21 / userModVersion 1.2.6; live Paradox API
  requiredVersion 1.6.*; PublishConfiguration `GameVersion 1.6.*`.
- No Harmony and no reflection beyond reading the assembly version string.
