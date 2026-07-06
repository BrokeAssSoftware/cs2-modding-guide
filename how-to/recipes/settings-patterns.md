---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: Settings UI patterns"
recipe: settings-patterns
technique_family: "N - SettingsUI slider/section/hide-by-condition"
diataxis: how-to
source_version: "1.6.0f1 (magic-mail@6fb3d2b; date-pinned, static source only)"
last_reverified: "2026-07-04"
canonical_mods:
  - magic-mail@6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47
  - better-bulldozer@4408466f226db811159d92859479ae1e1c28ba06
  - anarchy@a6311e898d20a775368668b234aaa32f06e3e1eb
  - road-speed-adjuster@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed
  - demand-modifier@e93ec1c1acdb58764c7949c4fca1654d451b690b
  - magic-garbage-truck@1b6a478753e1ef4e43ac9b90d567f3d7183c7be2
  - achievement-fixer@4d3a1eccc64a7a09de65a23f6abbb3daa998fd0d
  - realistic-jobsearch@7a096b2ab974bb03cc4cf0937f250bf1d7671f31
  - realistic-path-finding@50645fa6a078181365e36a42e2e27b96699bf02a
technique_applicability: [core, ui]
status: source-verified
Created: 2026-07-03
Updated: 2026-07-05
Owners:
  - codex
---

# Settings UI patterns

> Build a mod Options page - sliders, checkboxes, dropdowns, action buttons,
> conditional visibility, presets, and disk persistence - by subclassing
> `ModSetting` and decorating properties with `SettingsUI*` attributes. No custom
> UI code, no React, no Harmony.

## Problem
Your mod needs user-facing configuration: a multiplier slider, an on/off toggle, a
"reset to defaults" button, a preset picker, and options that only appear when a
master toggle is on. You want it to show up in the game's own **Options** panel, to
persist across sessions, and to be localizable - without writing any UI markup. The
game gives you a declarative attribute API for exactly this; the trick is knowing
which attribute does what and how the pieces compose.

## Solution
Derive one class from `Game.Settings.ModSetting`. Each public property becomes a
control: a `float`/`int` with `[SettingsUISlider]` is a slider, a `bool` is a
checkbox, an `enum` is a dropdown, and a write-only `bool` with `[SettingsUIButton]`
is a clickable action. `[SettingsUISection(tab, group)]` places each control on a tab
and in a group; `[SettingsUIHideByCondition]` / `[SettingsUIDisableByCondition]` wire
one control's visibility to another property. Override `SetDefaults()` for the initial
values and add extra methods for named presets. `[FileLocation(...)]` on the class
picks the on-disk file; the game serializes the settings there and
`AssetDatabase.global.LoadSettings(...)` restores them at load. Register the whole
thing with `setting.RegisterInOptionsUI()`.

## Steps & Code

### 1. Subclass `ModSetting` and declare the file location + defaults

`[FileLocation]` names the settings file the game reads/writes (a JSON document under
the user's `ModsSettings` tree). Seed values in the constructor on first run and in
the mandatory `SetDefaults()` override:

```csharp
[FileLocation("ModsSettings/MagicMail/MagicMail")]
public sealed class Setting : ModSetting
{
    public Setting(IMod mod) : base(mod)
    {
        // First run: start from pure game defaults (vanilla).
        if (!NotFirstTime) { SetDefaults(); NotFirstTime = true; }
    }

    public override void SetDefaults() { SetToVanilla(); }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/magic-mail/repo/Settings/Settings.cs#L26-L104` (@6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47) - `[FileLocation]` L26, class decl L43, constructor L95-L104, `SetDefaults` L509-L512.

`[SettingsUIHidden]` on a helper flag (like `NotFirstTime`) keeps bookkeeping state in
the settings file without drawing a control for it
(`magic-mail/repo/Settings/Settings.cs#L85-L90`).

### 2. Add a slider with `[SettingsUISlider]` + `[SettingsUISection]`

`[SettingsUISlider]` takes `min`, `max`, `step`, `scalarMultiplier`, and `unit` as
**named** arguments. `[SettingsUISection(tab, group)]` puts the control on a tab and
group defined by string constants on the class:

```csharp
[SettingsUISection(kActionsTab, PostVanGroup)]
[SettingsUISlider(min = 100, max = 500, step = 10,
    scalarMultiplier = 1, unit = Unit.kPercentage)]
[SettingsUIHideByCondition(typeof(Setting), nameof(ChangeCapacity), true)]
public int PostVanMailLoadPercentage { get; set; }
```
Source: `../../../vice-and-order-research/mods/dossiers/magic-mail/repo/Settings/Settings.cs#L190-L201` (@6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47)

`Unit` selects the display format: `Unit.kPercentage` here; Anarchy uses
`Unit.kFloatTwoFractions` for a `0-1.75` clearance slider and `Unit.kInteger` for a
`1-600` frequency slider:

```csharp
[SettingsUISlider(min = 0f, max = 1.75f, step = 0.25f,
    scalarMultiplier = 1, unit = Unit.kFloatTwoFractions)]
public float MinimumClearanceBelowElevatedNetworks { get; set; }
```
Source: `../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/Settings/AnarchyModSettings.cs#L191-L193` (@a6311e898d20a775368668b234aaa32f06e3e1eb); integer slider L204-L207.

### 3. Show/hide (or enable/disable) controls conditionally

`[SettingsUIHideByCondition(type, memberName, ...)]` hides a control based on another
member. Magic Mail passes a **property name + expected value** (`true`) so its
per-van sliders vanish when the `ChangeCapacity` master toggle is off - see the
attribute on the slider in step 2. Anarchy instead points the attribute at a
**predicate method**, which is more flexible:

```csharp
[SettingsUISlider(min = 1, max = 600, step = 1, scalarMultiplier = 1, unit = Unit.kInteger)]
[SettingsUIHideByCondition(typeof(AnarchyModSettings), nameof(IsCullingNotBeingPrevented))]
public int PropRefreshFrequency { get; set; }
// ...
public bool IsCullingNotBeingPrevented() => !PreventAccidentalPropCulling;
```
Source: `../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/Settings/AnarchyModSettings.cs#L204-L207,#L388` (@a6311e898d20a775368668b234aaa32f06e3e1eb)

Use `[SettingsUIDisableByCondition]` instead when you want the control **greyed out but
still visible** - Better Bulldozer disables its "restore" button while the matching
auto-remove toggle is on (`better-bulldozer/repo/BetterBulldozer/Settings/BetterBulldozerModSettings.cs#L70-L71`).

### 4. Add an action button (`[SettingsUIButton]`) with a confirmation dialog

A **write-only** `bool` property with `[SettingsUIButton]` renders as a button; the
`set` accessor runs on click. Add `[SettingsUIConfirmation]` to gate a destructive
action behind a yes/no prompt:

```csharp
[SettingsUIButton]
[SettingsUIConfirmation]
[SettingsUISection(kMainTab, kActionsGroup)]
public bool ClearAllCustomSpeeds
{
    set { if (value) ClearAllCustomSpeedsAction(); }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/sETTING.cs#L43-L55` (@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed)

Button `set` accessors typically reach into ECS to flip a system on. Better Bulldozer's
buttons do `World.DefaultGameObjectInjectionWorld.GetOrCreateSystemManaged<T>().Enabled = true`
so the click merely *arms* a one-shot system
(`better-bulldozer/repo/BetterBulldozer/Settings/BetterBulldozerModSettings.cs#L51-L60`).

### 5. Provide presets via extra methods; persist with `ApplyAndSave()`

`SetDefaults()` is the required baseline. Add more methods for named presets and bind
each to its own button. Magic Mail ships a "vanilla" baseline and a "recommended"
tuning preset:

```csharp
public void SetToVanilla()   // 100% capacities, no magic
{
    ChangeCapacity = false;
    PostVanMailLoadPercentage = 100;
    PostVanFleetSizePercentage = 100;
    // ...
}

private void SetRecommended()   // magic top-ups on, 200% van load
{
    ChangeCapacity = true;
    PostVanMailLoadPercentage = 200;
    PSF_SortingSpeedPercentage = 200;
    // ...
}
```
Source: `../../../vice-and-order-research/mods/dossiers/magic-mail/repo/Settings/Settings.cs#L516-L567` (@6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47); the two reset buttons that call them at L139-L171.

A "reset" button calls the preset then persists immediately. `ApplyAndSave()` (an
inherited `ModSetting` member) writes the file so the change survives a restart:

```csharp
[SettingsUIButton]
[SettingsUIConfirmation]
public bool ResetModSettings
{
    set { SetDefaults(); ApplyAndSave(); }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/better-bulldozer/repo/BetterBulldozer/Settings/BetterBulldozerModSettings.cs#L110-L118` (@4408466f226db811159d92859479ae1e1c28ba06)

### 6. Override `Apply()` to push changes into the live simulation

`ModSetting.Apply()` runs whenever the user changes an option. Override it to react -
Magic Mail re-enables its capacity systems so slider changes take effect "instantly":

```csharp
public override void Apply()
{
    base.Apply();
    World? world = World.DefaultGameObjectInjectionWorld;
    if (world == null || !world.IsCreated) return;
    MailCapacitySystem? capacitySystem = world.GetExistingSystemManaged<MailCapacitySystem>();
    if (capacitySystem != null) capacitySystem.Enabled = true;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/magic-mail/repo/Settings/Settings.cs#L108-L133` (@6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47) (abridged; guard + system lookups shown)

### 7. Register + load in the mod entry point

In your `IMod.OnLoad`, construct the settings, register the UI, then load the persisted
file (falling back to a fresh instance for first run):

```csharp
Setting setting = new Setting(this);
setting.RegisterInOptionsUI();
AssetDatabase.global.LoadSettings(ModId, setting, new Setting(this));
```
Source: `../../../vice-and-order-research/mods/dossiers/magic-mail/repo/Mod.cs#L105-L109` (@6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47) (`LoadSettings` L105, `RegisterInOptionsUI` L108)

Labels for every control/tab/group come from a paired localization source - see the
sibling recipe [localization helper](localization-helper.md). Without it, the UI shows
raw locale-ID keys instead of readable text.

## Pitfalls & gotchas

- **A control with no localized label shows its raw key.** Every property, tab, and
  group ID must have a matching entry in an `IDictionarySource`. Register the settings
  object **before** the locale sources so labels resolve (Magic Mail comments this
  explicitly: "Settings must exist before locales so labels resolve correctly",
  `magic-mail/repo/Mod.cs#L83-L88`). See [localization helper](localization-helper.md).

- **Buttons are write-only bools; the setter fires on click, not the value.** Guard
  with `if (value)` / `if (!value) return;` - the accessor can be invoked with `false`.
  Road Speed Adjuster and Magic Mail both guard
  (`road-speed-adjuster/repo/sETTING.cs#L48-L52`; `magic-mail/repo/Settings/Settings.cs#L144-L152`).

- **`SetDefaults()` is mandatory and must set every field.** It is the initial state
  and the reset target. A field you forget to set there keeps its zero value, which for
  a percentage slider means a broken 0%. Better Bulldozer and Magic Mail set every
  option in `SetDefaults`/`SetToVanilla`.

- **Named vs positional slider args.** The canonical mods all use the **named** form
  (`min =`, `max =`, `step =`, `scalarMultiplier =`, `unit =`). A positional form
  exists in older code, but named is unambiguous and what current source uses.

- **`[SettingsUIHideByCondition]` re-uses the same value on hidden controls.** Hiding a
  slider does not neutralize its stored value; it just isn't drawn. If a hidden option
  still feeds the simulation, gate the *behavior* too, not only the UI.

- **Do not cache a setting in a `static readonly` field.** A `static readonly` is
  evaluated once at type initialization (first access to the type), so it snapshots the
  setting value at that moment and never sees a later change. Realistic Jobsearch's
  pathfinding patch captures its tuning weights this way -
  `static readonly float AlphaJobs = Mod.m_Setting.alpha_jobs;` and three more - which
  means editing those sliders does nothing until the assembly reloads (a game restart).
  Read `Mod.m_Setting.<prop>` at the point of use, or refresh a cache from `Apply()`
  instead.

  ```csharp
  [HarmonyPatch(typeof(PathfindSetupSystem), "CompleteSetup")]
  public static class Patch_CompleteSetup_FilterJobSeekerTargets
  {
      static readonly float AlphaJobs = Mod.m_Setting.alpha_jobs;   // frozen at type-init
      static readonly float BetaMinute = Mod.m_Setting.beta_minute; // never re-reads
      static readonly float WTotal = Mod.m_Setting.weight_total_jobs;
      static readonly float WFree = Mod.m_Setting.weight_free_jobs;
  }
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/realistic-jobsearch/repo/RealisticJobSearch/Patches/Patch_CompleteSetup_FilterJobSeekerTargets.cs#L15-L22` (@7a096b2ab974bb03cc4cf0937f250bf1d7671f31) - `[HarmonyPatch]` L15, captured fields L19-L22.

- **A `[SettingsUISlider]` min/max is a UI hint, not an enforced bound.** The slider only
  constrains what the *panel* can produce; the value can still be set out of range by a
  hand-edited settings file, and whatever your runtime does with it is what actually
  governs behavior. Realistic Path Finding caps `ped_crosswalk_factor` at `max = 5f` in
  the UI but its system clamps the same value to `0.1f..50f` - a manually edited file can
  push a 10x-wider crosswalk penalty than the slider ever offered. If a range matters for
  balance or safety, clamp to the *same* bounds at the point of use, not looser ones.

  ```csharp
  // UI slider:
  [SettingsUISlider(min = 0.1f, max = 5f, step = 0.05f, unit = Unit.kFloatTwoFractions)]
  public float ped_crosswalk_factor { get; set; }
  // Runtime, in the cost system - clamps far looser than the slider:
  float newCross = math.clamp(Mod.m_Setting?.ped_crosswalk_factor ?? 1f, 0.1f, 50f);
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Setting.cs#L243-L246` (slider L243, property L246, @50645fa6a078181365e36a42e2e27b96699bf02a); runtime clamp `../../../vice-and-order-research/mods/dossiers/realistic-path-finding/repo/RealisticPathFinding/Systems/PedestrianCrosswalkCostFactorSystem.cs#L55-L56`.

- **The exact confirmation-dialog wording and the precise JSON on-disk schema are
  `Needs Verification (in-game)`.** The confirmation prompt text is supplied via
  `GetOptionWarningLocaleID(...)` in a locale source
  (`road-speed-adjuster/repo/sETTING.cs#L176`), but the rendered dialog layout is not
  provable from source.

## Variations

- **Enum dropdown instead of a slider.** A public `enum` property renders as a
  dropdown; each enum value is localized via `GetEnumValueLocaleID`. Road Speed
  Adjuster exposes a `SpeedUnit { Auto, Metric, Imperial }` preference this way:

  ```csharp
  [SettingsUISection(kMainTab, kDisplayGroup)]
  public SpeedUnit SpeedUnitPreference { get; set; } = SpeedUnit.Auto;
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/road-speed-adjuster/repo/sETTING.cs#L31-L32` (@e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed); enum labels at `sETTING.cs#L169-L171`.

- **Enum backing values that ARE the stored quantity.** An enum dropdown does not have
  to be an opaque label set - you can pick the enum's integer backing values so the
  selection *is* the number you feed the simulation. Demand Modifier's `DemandLevel`
  maps its five choices straight onto the game's `0-255` demand scale
  (`Off = 0, Low = 64, Medium = 128, High = 192, Maximum = 255`), so the chosen literal
  is the value with no lookup table:

  ```csharp
  public enum DemandLevel
  {
      Off = 0, Low = 64, Medium = 128, High = 192, Maximum = 255
  }
  // ...
  [SettingsUIDropdown(typeof(DemandModifierSettings), nameof(GetDemandLevelOptions))]
  public DemandLevel ResidentialDemandLevel { get; set; }
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/demand-modifier/repo/DemandModifier/DemandModifierSettings.cs#L14-L21` (@e93ec1c1acdb58764c7949c4fca1654d451b690b); dropdown property L76-L77.

- **`[SettingsUIDropdown]` populated at runtime, not from a fixed enum.** The dropdown's
  `itemsGetter` method can build its `DropdownItem<T>[]` from live game data instead of a
  hard-coded enum. Achievement Fixer fills its picker by enumerating the platform's
  achievement list at open time, keying on `string` values so it survives roster changes:

  ```csharp
  [SettingsUIDropdown(typeof(Settings), nameof(GetAchievementChoices))]
  public string SelectedAchievement { get; set; } = "";

  public static DropdownItem<string>[] GetAchievementChoices()
  {
      PlatformManager pm = PlatformManager.instance;
      if (pm == null) return Array.Empty<DropdownItem<string>>();
      return pm.EnumerateAchievements()
          .Select(a => a.internalName ?? a.id.ToString())
          .OrderBy(id => AchievementDisplay.Get(id), StringComparer.CurrentCultureIgnoreCase)
          .Select(id => new DropdownItem<string> { value = id, displayName = AchievementDisplay.Get(id) })
          .ToArray();
  }
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/achievement-fixer/repo/Settings/Settings.cs#L133-L134` (dropdown, @4d3a1eccc64a7a09de65a23f6abbb3daa998fd0d); `GetAchievementChoices` L282-L302. The paired action buttons gate destruction with
  `[SettingsUIConfirmation]` - `ClearSelectedAchievement` (L184-L188) and
  `ResetAllAchievements` (L239-L243) both fire a Yes/No modal before touching platform
  state.

- **Mutually-exclusive "mode" toggles that wake systems via `[SettingsUISetter]`.** Two
  bool toggles can be made to auto-clear each other in their setters, giving radio-button
  behavior without a dropdown; a shared `[SettingsUISetter]` callback then arms the ECS
  systems so the switch takes effect immediately. Magic Garbage Truck's `TotalMagic` and
  `TrashBossEnabled` each zero the other, and `OnModeToggleChanged` flips the affected
  systems back on:

  ```csharp
  [SettingsUISetter(typeof(Setting), nameof(OnModeToggleChanged))]
  public bool TotalMagic
  {
      get => m_TotalMagic;
      set { if (m_TotalMagic == value) return;
            m_TotalMagic = value;
            if (m_TotalMagic) m_TrashBossEnabled = false;   // exclusive
            Apply(); }
  }
  // setter callback re-arms the systems:
  private void OnModeToggleChanged(bool _)
  {
      if (!TryGetWorld(out World world)) return;
      TotalMagicSystem tm = world.GetExistingSystemManaged<TotalMagicSystem>();
      if (tm != null) tm.Enabled = true;
      // ...same for GarbageTruckCapacitySystem, GarbageThresholdSystem, etc.
  }
  ```
  Source: `../../../vice-and-order-research/mods/dossiers/magic-garbage-truck/repo/Settings/Setting.cs#L117-L162` (exclusive toggles, @1b6a478753e1ef4e43ac9b90d567f3d7183c7be2); `OnModeToggleChanged` L514-L555. The paired `[SettingsUIHideByCondition(..., nameof(TrashBossEnabled), true)]` sliders (L172-L192) then appear only in the matching mode.

- **Read-only "About" text rows.** A `string` get-only property with
  `[SettingsUISection]` renders as a static text row - handy for version, credits, or a
  live status line. Magic Mail's Status tab builds localized summary strings this way
  (`magic-mail/repo/Settings/Settings.cs#L391-L433`); Road Speed Adjuster shows version
  and credits (`sETTING.cs#L61-L74`).

- **Grouped/ordered buttons.** `[SettingsUIButtonGroup(name)]` lays multiple buttons on
  one row; class-level `[SettingsUITabOrder]`, `[SettingsUIGroupOrder]`, and
  `[SettingsUIShowGroupName]` control tab/group ordering and headers
  (`magic-mail/repo/Settings/Settings.cs#L27-L42`; `road-speed-adjuster/repo/sETTING.cs#L12-L14`).

- **Setter callback on value change.** `[SettingsUISetter(type, method)]` runs a method
  when a toggle flips - Better Bulldozer uses it to enable/disable a system the instant
  a checkbox changes, without waiting for `Apply()`
  (`better-bulldozer/repo/BetterBulldozer/Settings/BetterBulldozerModSettings.cs#L45-L46`).

## See also
- Related recipes: [localization helper](localization-helper.md) (family O - the
  paired `IDictionarySource` that supplies every label these controls need).
- Reference: [technique index](../../technique-index.md) (family N coverage ledger).
- Explanation: [mod lifecycle](../../explanation/mod-lifecycle.md) (where `OnLoad`
  registration and `Apply()` sit in the load sequence).
- Case studies demonstrating it: [anarchy](../../case-studies/anarchy.md).

## Sources
- Canonical mods (dossier + repo):
  - `magic-mail` @6fb3d2b4dade5a8717aeee6c9e8456bd0f69cb47 - `repo/Settings/Settings.cs`, `repo/Mod.cs`
  - `better-bulldozer` @4408466f226db811159d92859479ae1e1c28ba06 - `repo/BetterBulldozer/Settings/BetterBulldozerModSettings.cs`
  - `anarchy` @a6311e898d20a775368668b234aaa32f06e3e1eb - `repo/Anarchy/Settings/AnarchyModSettings.cs`
  - `road-speed-adjuster` @e0c0c0b3c30fd43dd29bf3322614f8c0e95172ed - `repo/sETTING.cs`
  - `demand-modifier` @e93ec1c1acdb58764c7949c4fca1654d451b690b - `repo/DemandModifier/DemandModifierSettings.cs`
  - `magic-garbage-truck` @1b6a478753e1ef4e43ac9b90d567f3d7183c7be2 - `repo/Settings/Setting.cs`
  - `achievement-fixer` @4d3a1eccc64a7a09de65a23f6abbb3daa998fd0d - `repo/Settings/Settings.cs`
  - `realistic-jobsearch` @7a096b2ab974bb03cc4cf0937f250bf1d7671f31 - `repo/RealisticJobSearch/Patches/Patch_CompleteSetup_FilterJobSeekerTargets.cs`
  - `realistic-path-finding` @50645fa6a078181365e36a42e2e27b96699bf02a - `repo/RealisticPathFinding/Setting.cs`, `repo/RealisticPathFinding/Systems/PedestrianCrosswalkCostFactorSystem.cs`
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding
