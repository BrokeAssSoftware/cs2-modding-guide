---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: Preset / country profile lookup arrays"
recipe: preset-profile-arrays
technique_family: "Z - Preset / country profile lookup arrays"
diataxis: how-to
source_version: "1.6.0f1 (time2work-realistic-trips@d42921f; date-pinned, static source only)"
last_reverified: "2026-07-04"
status: source-verified
canonical_mods:
  - time2work-realistic-trips@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39
technique_applicability: [economy, simulation]
Created: 2026-07-04
Updated: 2026-07-04
Owners:
  - codex
---

# Preset / country profile lookup arrays

> Ship dozens of tunable settings as a handful of named presets ("Balanced",
> "Germany", "Japan", ...) by storing every value as one column of a parallel
> index-keyed array, then fanning the selected column into live properties in a
> single `SetParameters(index)` call.

## Problem
Your mod exposes a large field of numeric/enum settings (shift shares, work hours,
school times, leisure frequencies) and you want the user to pick a **named profile**
that sets all of them at once - a "country" or difficulty preset - instead of hand-
tuning ~60 sliders. You need (a) a compact place to author each preset's full column
of values, (b) a way to map the dropdown choice to a column, and (c) one operation
that pushes every value of that column into the live settings the simulation reads.

## Solution
Model each setting as a **backing array** named `<prop>_` whose entries are the value
of that setting for preset 0, preset 1, ... All arrays share the same length and the
same column meaning: **column N is preset N**. A single method `SetParameters(int
index)` assigns `prop = prop_[index]` for every setting, so selecting a preset is one
`index`. The dropdown is an `enum`-typed setting; a lookup dictionary translates the
chosen enum member to the array column, and a Button calls `SetParameters` with it.
The whole preset system is one file and needs no per-preset classes.

## Steps & Code

### 1. Declare one backing array per setting, one column per preset

Each `<prop>_` array is the authored data for that setting across all presets. Every
array is the same length (here 20 columns) and column order is identical, so column
`index` always means the same preset:

```csharp
Dictionary<int, int> countryIndexLookup = new Dictionary<int, int>();
// Preset order follows SettingsEnum numeric order; keep every array length in sync.
int[] evening_share_ = new int[] { 10, 17, 15, 13, 13, 5, 19, 15, 22, 14, 14, 19, 31, 13, 16, 32, 18, 15, 17, 8 };
int[] night_share_   = new int[] { 8, 8, 7, 5, 7, 2, 12, 5, 10, 6, 8, 8, 8, 7, 11, 12, 10, 7, 9, 4 };
int[] delay_factor_  = new int[] { 2, 4, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2 };
```
Source: `../../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Setting.cs#L75-L79` (@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39)

This mod declares ~80 such parallel arrays (`int[]`, `float[]`, `bool[]`) covering
everything from `holidays_per_year_` to `travel_sunday_`
(`NightShift/Setting.cs#L77-L162`). The source's own comment ("keep every array
length in sync") is the load-bearing invariant - see Pitfalls.

### 2. Fan the chosen column into live properties in `SetParameters(index)`

One method reads column `index` out of every array and writes it to the public
property the simulation actually consumes. This is the entire preset-application
mechanism:

```csharp
public void SetParameters(int index)
{
    evening_share = evening_share_[index];
    night_share = night_share_[index];
    delay_factor = delay_factor_[index];
    lunch_break_percentage = lunch_break_percentage_[index];
    holidays_per_year = holidays_per_year_[index];
    // ... ~80 more assignments, one per backing array ...
```
Source: `../../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Setting.cs#L183-L189` (@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39)

`SetDefaults()` just calls `SetParameters(0)` so preset column 0 ("Balanced") is the
factory default (`NightShift/Setting.cs#L179-L182`).

### 3. Store enum-valued settings as ints in the array, cast back on fan-out

Arrays are homogeneous (`int[]`/`float[]`/`bool[]`), so an `enum`-typed setting is
authored as its underlying `int` in the array and cast back to the enum type at the
assignment site:

```csharp
school_start_time = (timeEnum)school_start_time_[index];
high_school_start_time = (timeEnum)high_school_start_time_[index];
work_start_time = (timeEnum)work_start_time_[index];
dt_simulation = (DTSimulationEnum)dt_simulation_[index];
school_vacation_month1 = (months)school_vacation_month1_[index];
```
Source: `../../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Setting.cs#L194-L245` (@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39)

The backing arrays for these are plain `int[]` holding the enum's ordinal, e.g.
`int[] school_start_time_ = new int[] { 2, 2, 1, 4, ... };`
(`NightShift/Setting.cs#L86`). This is the sharpest gotcha in the pattern - see
Pitfalls.

### 4. Map the dropdown enum member to a column with a lookup dictionary

The preset dropdown is an `enum`-typed setting. Because the enum values are **not**
contiguous (`Balanced = 0, Performance = 1, Argentina = 8, Australia = 10, ...`), the
enum value is not the array index. The constructor builds a dictionary from each enum
member to its **ordinal position** (0, 1, 2, ...), which is the real column:

```csharp
int i = 0;
foreach (var value in Enum.GetValues(typeof(SettingsEnum)))
{
    SettingsEnum e = (SettingsEnum)value;
    countryIndexLookup.Add((int)e, i);   // enum value -> column index
    i++;
}
```
Source: `../../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Setting.cs#L168-L174` (@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39)

`Enum.GetValues` returns members in ascending numeric order, so column `i` lines up
with the i-th column authored in every backing array. The `SettingsEnum` definition
(20 members, sparse values) is at `NightShift/Setting.cs#L928-L949`.

### 5. Apply the selected preset from the dropdown

The `enum` setting renders as a dropdown; a Button reads the current choice, resolves
it through the dictionary, and applies that column:

```csharp
[SettingsUISection(SettingsSection, SettingsGroup)]
public SettingsEnum settings_choice { get; set; } = SettingsEnum.Balanced;

[SettingsUIButton]
[SettingsUISection(SettingsSection, SettingsGroup)]
public bool Button
{
    set
    {
        countryIndexLookup.TryGetValue((int)settings_choice, out int selectedSetting);
        SetParameters(selectedSetting);
    }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/time2work-realistic-trips/repo/NightShift/Setting.cs#L306-L319` (@d42921ffb1f6bbbf2c2b086bb17727bf0f637c39)

`TryGetValue` leaves `selectedSetting` at `0` (Balanced) on a miss, so an unknown
enum falls back to the first column rather than throwing.

## Pitfalls & gotchas

- **Lockstep by index is the whole fragility.** Every array is positional: column N
  must mean preset N in *all* ~80 arrays. Insert, delete, or reorder one array's
  entries without touching the others and presets silently blend values from two
  different profiles - no compiler error, no exception, just wrong numbers. The source
  flags this itself with the comment "keep every array length in sync"
  (`NightShift/Setting.cs#L76`). There is no length assertion in the cited code; the
  invariant is maintained by hand.

- **Enum values stored as ints, cast back unchecked.** Enum settings live in `int[]`
  as ordinals and are cast with `(timeEnum)`, `(DTSimulationEnum)`, `(months)`
  (`NightShift/Setting.cs#L194-L245`). A C# enum cast does **not** validate range, so
  an out-of-range int becomes an undefined enum value that later code may mishandle.
  If you renumber or reorder an enum (e.g. `timeEnum`, `NightShift/Setting.cs#L953`)
  after authoring the array literals, every stored ordinal silently shifts meaning.

- **Dropdown value != array index.** `SettingsEnum` uses sparse real-world codes
  (`Argentina = 8`, `USA = 188`), so you cannot index arrays with `(int)settings_choice`
  directly - that would be an out-of-bounds crash. The `countryIndexLookup` indirection
  (`NightShift/Setting.cs#L168-L174`, `#L315`) exists solely to translate enum member
  -> dense column. Skipping it is the obvious-but-wrong shortcut.

- **`OnUpdate`-free, but not automatic.** In the cited source a preset is applied only
  when `SetParameters` is called - from `SetDefaults` (`#L179-L182`) and the Button
  setter (`#L316`). Whether the running game re-reads these properties immediately or
  only after `Apply()`/reload is a runtime behaviour not visible in this file:
  `Needs Verification (in-game)`.

- **Some fields are hard-coded, not preset-driven.** `SetParameters` also sets
  constants that ignore `index` (e.g. `resourceConsumption = 20;`,
  `hospital_stay_duration_enabled = false;`, `NightShift/Setting.cs#L268-L279`). Do
  not assume every property fanned in the method has a backing array; mixing per-preset
  and global defaults in the same method is intentional here but easy to misread.

## Variations

- **Skip the enum-code indirection.** If your preset dropdown enum is 0..N-1
  contiguous, you can index the arrays with `(int)choice` directly and drop
  `countryIndexLookup`. This mod needs the dictionary only because it reused sparse
  country codes as enum values (`NightShift/Setting.cs#L928-L949`).

- **Struct-of-arrays vs array-of-structs.** The cited mod uses struct-of-arrays (one
  array per field). The alternative is one `Preset[]` where each element is a struct
  holding all fields for that preset - fewer lockstep arrays to keep aligned, at the
  cost of more ceremony per field. Choose array-of-structs when field count is modest;
  the cited mod's ~80 fields make dense column-authored arrays easier to eyeball.

- **Apply on load instead of a button.** Instead of a manual Button, resolve and call
  `SetParameters` from your settings `Apply()`/load hook so the preset re-applies
  automatically. The cited mod's `Apply()` re-enables its systems
  (`NightShift/Setting.cs#L282-L303`) but leaves preset selection on the explicit
  Button; wiring it into a load hook is a common deviation.

## See also
- Related recipes: [settings patterns](settings-patterns.md) (family N - how the
  `ModSetting` properties, sections, and dropdowns these presets drive are declared).
- Reference: [settings and data](../../explanation/settings-and-data.md) (how mod
  settings are persisted and read).
- Case studies demonstrating it:
  [time2work-realistic-trips](../../case-studies/time2work-realistic-trips.md).

## Sources
- Canonical mods (dossier + repo):
  - `time2work-realistic-trips` @d42921ffb1f6bbbf2c2b086bb17727bf0f637c39 - `repo/NightShift/Setting.cs`
- Official/community references (link out, do not duplicate):
  https://cs2.paradoxwikis.com/Modding
