---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Recipe: IMGUI debug / analytics overlay + sample recorder"
recipe: imgui-debug-overlay
technique_family: "AY - IMGUI debug / analytics overlay + sample recorder"
diataxis: how-to
source_version: "~1.4.x (market-based-economy@b83f196; date-pinned, static source only)"
last_reverified: "2026-07-05"
status: source-verified
canonical_mods:
  - market-based-economy@b83f196a36bc74388accebdeb7c81f0f35dbab37
technique_applicability: [tooling]
Created: 2026-07-05
Updated: 2026-07-05
Owners:
  - codex
---

# IMGUI debug / analytics overlay + sample recorder

> Draw a self-contained, hotkey-gated developer overlay - draggable window, tabs,
> and live line graphs - using Unity's immediate-mode `OnGUI`, fed by a bounded
> in-memory sample recorder, with zero dependency on the game's cohtml UI.

## Problem
You need to *see* what your simulation is doing over time - wages drifting, a
product price oscillating, a value diverging from its baseline - while the game
runs. The game's real UI (cohtml/React binding) is heavyweight to wire up, ships
to players, and is the wrong tool for a throwaway diagnostic. You want a
developer-only overlay you can toggle with a hotkey, that plots recent history as
a graph, and that never leaks into the shipped player experience or grows memory
without bound.

## Solution
Use Unity's legacy immediate-mode GUI (`OnGUI` / IMGUI) on a plain `MonoBehaviour`
hosted on a `DontDestroyOnLoad` GameObject. IMGUI runs entirely outside the game's
UI stack: you draw a `GUI.Window` every frame, put your controls in it with
`GUILayout`, and rasterize graphs by hand into a `Texture2D` you blit with
`GUI.DrawTexture`. Feed it from a separate singleton **recorder** that other
systems push samples into on a fixed interval, storing each series in a
**bounded ring** (drop-oldest past a cap) so a long session cannot exhaust memory.
Gate visibility behind a hotkey so the overlay is invisible - and nearly free -
until a developer asks for it. This is distinct from a world-space TextMeshPro
overlay (family I, [render-pipeline-overlay](render-pipeline-overlay.md)): IMGUI is
a flat screen-space developer tool, not diegetic 3D text.

## Steps & Code

### 1. Host the overlay on a hidden, persistent GameObject

The overlay is a `MonoBehaviour`; create one GameObject to carry it (and its hotkey
component) exactly once, mark it hidden + persistent, and add the components. Call
`Ensure()` from the mod's `OnLoad` and `Dispose()` from `OnDispose`.

```csharp
s_Root = new GameObject("MarketEconomyAnalyticsOverlay")
{
    hideFlags = HideFlags.HideAndDontSave
};
Object.DontDestroyOnLoad(s_Root);

if (!s_Root.TryGetComponent(out EconomyAnalyticsOverlay _))
    s_Root.AddComponent<EconomyAnalyticsOverlay>();
if (!s_Root.TryGetComponent(out EconomyAnalyticsHotkey _))
    s_Root.AddComponent<EconomyAnalyticsHotkey>();
```
Source: `../../../vice-and-order-research/mods/dossiers/market-based-economy/repo/Analytics/EconomyAnalyticsOverlayHost.cs#L19-L33` (@b83f196a36bc74388accebdeb7c81f0f35dbab37)

`HideFlags.HideAndDontSave` keeps the object out of the scene hierarchy and out of
saves; `DontDestroyOnLoad` keeps it alive across the main-menu/city load boundary.
The mod wires it up in `OnLoad`: `EconomyAnalyticsOverlayHost.Ensure();`
(`../../../vice-and-order-research/mods/dossiers/market-based-economy/repo/Mod.cs#L62`).

### 2. Draw the window in `OnGUI`, gated on a visibility flag

`OnGUI` fires every IMGUI event (multiple times per frame). Bail immediately when
hidden so the overlay costs almost nothing when off; otherwise hand a draw callback
to `GUI.Window`, which returns the (possibly dragged) window rect to store back.

```csharp
private void OnGUI()
{
    EnsureStyles();
    if (!m_ShowOverlay)
        return;

    m_WindowRect = GUI.Window(GetInstanceID(), m_WindowRect, DrawWindow, "Market Economy Analytics");
}
```
Source: `../../../vice-and-order-research/mods/dossiers/market-based-economy/repo/Analytics/EconomyAnalyticsOverlay.cs#L132-L142` (@b83f196a36bc74388accebdeb7c81f0f35dbab37)

### 3. Lay out tabs and make the window draggable

Inside the `DrawWindow` callback, use `GUILayout.Toolbar` for tab selection, switch
on the selected index to draw the active tab, and register a drag strip at the top
with `GUI.DragWindow` so the developer can reposition it.

```csharp
private void DrawWindow(int id)
{
    GUILayout.BeginVertical();
    DrawHeader();
    GUILayout.Space(6f);
    DrawTabSelector();                     // GUILayout.Toolbar(current, s_TabLabels)
    GUILayout.Space(8f);
    switch (Mathf.Clamp(m_SelectedTab, 0, s_TabLabels.Length - 1))
    {
        case 0: DrawLiveSection(); break;
        case 1: DrawWageTab();     break;
        case 2: DrawPriceTab();    break;
    }
    GUILayout.EndVertical();
    GUI.DragWindow(new Rect(0f, 0f, m_WindowRect.width, 24f));
}
```
Source: `../../../vice-and-order-research/mods/dossiers/market-based-economy/repo/Analytics/EconomyAnalyticsOverlay.cs#L155-L179` (@b83f196a36bc74388accebdeb7c81f0f35dbab37)

The tab labels are a static array `{ "Live", "Wages", "Prices" }`
(`EconomyAnalyticsOverlay.cs#L43`); `DrawTabSelector` clamps and stores
`GUILayout.Toolbar`'s return (`EconomyAnalyticsOverlay.cs#L206-L210`).

### 4. Record samples on a fixed interval into a bounded ring

The overlay never queries the simulation directly - it reads from a thread-safe
singleton recorder. Simulation systems push samples in; the recorder **throttles**
to one sample per interval (overwriting the last sample between intervals) and
trims the oldest entries past the cap.

```csharp
private const int kDefaultSampleCap = 2048;
private const float kWageSampleInterval = 0.1f;   // sample every 100ms
private const float kPriceSampleInterval = 0.1f;
```
Source: `../../../vice-and-order-research/mods/dossiers/market-based-economy/repo/Analytics/EconomyAnalyticsRecorder.cs#L13-L15` (@b83f196a36bc74388accebdeb7c81f0f35dbab37)

```csharp
public void RecordWageSample(int wage0, int wage1, int wage2, int wage3, int wage4)
{
    float timestamp = Time.realtimeSinceStartup;
    lock (m_Lock)
    {
        bool shouldAppend = m_WageSamples.Count == 0
            || timestamp - m_LastWageSampleTime >= kWageSampleInterval;
        if (shouldAppend)
        {
            m_WageSamples.Add(new WageSample { Time = timestamp, Level0 = wage0, /* ... */ Level4 = wage4 });
            TrimWageSamples_NoLock();
            m_LastWageSampleTime = timestamp;
        }
        else // within the interval: overwrite the last sample instead of appending
        {
            m_WageSamples[m_WageSamples.Count - 1] = new WageSample { Time = timestamp, /* ... */ };
        }
    }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/market-based-economy/repo/Analytics/EconomyAnalyticsRecorder.cs#L63-L98` (@b83f196a36bc74388accebdeb7c81f0f35dbab37)

The cap is enforced by dropping the oldest entries - a ring by trim, not a
fixed-size circular buffer:

```csharp
private void TrimWageSamples_NoLock()
{
    int excess = m_WageSamples.Count - m_MaxSamples;
    if (excess > 0)
        m_WageSamples.RemoveRange(0, excess);
}
```
Source: `../../../vice-and-order-research/mods/dossiers/market-based-economy/repo/Analytics/EconomyAnalyticsRecorder.cs#L221-L228` (@b83f196a36bc74388accebdeb7c81f0f35dbab37)

Prices are stored per-resource in a `Dictionary<Resource, List<PriceSample>>`, each
list trimmed to the same cap, so *every* tracked resource gets its own ring of
<=2048 samples (`EconomyAnalyticsRecorder.cs#L100-L150, #L230-L238`). Producers call
in from the simulation - e.g. `WageAdjustmentSystem.cs#L48` and
`MarketEconomyManager.cs#L334`.

### 5. Rasterize the graph by hand into a `Texture2D`

There is no chart widget in IMGUI, so plot the samples yourself. Allocate a
fixed-size `Texture2D` once in `Awake`, and each frame clear it, scan the visible
window for min/max, then walk the samples drawing Bresenham line segments between
consecutive points into a `Color32[]` pixel buffer.

```csharp
for (int i = 0; i < sampleCount; i++)
{
    var sample = m_WageSamples[startIndex + i];
    float value = GetWageValue(sample, selectedLevel);
    float normalized = useIndexFallback ? i / maxIndex
                                        : Mathf.Clamp01((sample.Time - firstTime) / timeRange);
    int x = Mathf.Clamp(Mathf.RoundToInt(normalized * (kGraphWidth - 1)), 0, kGraphWidth - 1);
    int y = ValueToPixelY(value, min, max);            // (1 - InverseLerp(min,max)) * (H-1)
    if (hasPrev) DrawLine(m_WagePixels, prevX, prevY, x, y, lineColor);
    else       { SetPixel(m_WagePixels, x, y, lineColor); hasPrev = true; }
    prevX = x; prevY = y;
}
ApplyTexture(m_WageTexture, m_WagePixels);             // SetPixels32 + Apply
```
Source: `../../../vice-and-order-research/mods/dossiers/market-based-economy/repo/Analytics/EconomyAnalyticsOverlay.cs#L569-L598` (@b83f196a36bc74388accebdeb7c81f0f35dbab37)

`ValueToPixelY` maps a value to a pixel row (Y inverted, since texture row 0 is the
top) via `Mathf.InverseLerp`
(`EconomyAnalyticsOverlay.cs#L820-L824`); `DrawLine` is a hand-rolled Bresenham
writing into the flat pixel array
(`EconomyAnalyticsOverlay.cs#L791-L818`); `ApplyTexture` uploads it with
`SetPixels32` + `Apply` (`EconomyAnalyticsOverlay.cs#L760-L764`). The texture and
pixel buffers are allocated once, up front:

```csharp
m_WageTexture = CreateTexture();       // Texture2D(320,160,RGBA32,false){ Point, Clamp }
m_PriceTexture = CreateTexture();
m_WagePixels = new Color32[kGraphWidth * kGraphHeight];
m_PricePixels = new Color32[kGraphWidth * kGraphHeight];
```
Source: `../../../vice-and-order-research/mods/dossiers/market-based-economy/repo/Analytics/EconomyAnalyticsOverlay.cs#L89-L92` (@b83f196a36bc74388accebdeb7c81f0f35dbab37)

Then blit it into the layout with `GUI.DrawTexture` on a reserved rect
(`EconomyAnalyticsOverlay.cs#L376-L379`).

### 6. Toggle visibility from a hotkey component

A sibling `MonoBehaviour` polls the configured binding each `Update` and flips the
overlay flag. The default chord is **Shift + G** (`RequireShift = true`, default key
`KeyCode.G`).

```csharp
private void Update()
{
    var binding = EconomyAnalyticsConfig.GetToggleBinding();
    var inputManager = InputManager.instance;
    if (binding != null && inputManager != null)
    {
        var action = inputManager.FindAction(binding);
        if (action != null && action.WasPerformedThisFrame())
            m_Overlay.ToggleVisibility();
    }
}
```
Source: `../../../vice-and-order-research/mods/dossiers/market-based-economy/repo/Analytics/EconomyAnalyticsHotkey.cs#L25-L43` (@b83f196a36bc74388accebdeb7c81f0f35dbab37)

`ToggleVisibility` just flips `m_ShowOverlay`
(`EconomyAnalyticsOverlay.cs#L109-L112`); the default hotkey constants are in
`EconomyAnalyticsConfig.cs#L12,L15`.

## Pitfalls & gotchas

- **This is IMGUI, not the game UI - and not family I.** `OnGUI` draws in flat
  screen space on top of everything; it is a developer tool, not the cohtml/React UI
  players see, and not a world-space TextMeshPro overlay (that is family I,
  [render-pipeline-overlay](render-pipeline-overlay.md)). Do not use this pattern for
  shipped player-facing UI.

- **`OnGUI` runs several times per frame.** IMGUI callbacks fire once per GUI event
  (layout + repaint + input), so anything expensive inside `OnGUI` is multiplied.
  Keep the early-out on `!m_ShowOverlay` first (`EconomyAnalyticsOverlay.cs#L136-L139`)
  and do heavy work (texture rebuild) only when actually visible.

- **Bound the sample store or leak memory over a long session.** The recorder caps
  each series at `kDefaultSampleCap = 2048` and drops oldest on overflow
  (`EconomyAnalyticsRecorder.cs#L13,L221-L228`). Prices are per-resource, so total
  retained samples scale with the number of tracked resources x 2048 - still bounded,
  but not a single fixed budget. Without the trim, an unbounded `List` would grow for
  the life of the process.

- **Cross-thread access needs a lock.** Simulation systems (potentially job/worker
  threads) push samples while the main-thread `OnGUI` reads them; the recorder guards
  every mutation and copy with `lock (m_Lock)` and hands the overlay a *copy*
  (`CopyWageSamples`) rather than the live list
  (`EconomyAnalyticsRecorder.cs#L152-L164`). Rendering off a shared live list would
  risk `InvalidOperationException` mid-enumeration.

- **Throttle vs. append.** Between intervals the recorder *overwrites* the last
  sample instead of appending (`EconomyAnalyticsRecorder.cs#L84-L96`), so the graph's
  last point tracks the newest value without inflating the ring at the caller's tick
  rate. If you append unconditionally, your 2048-sample window collapses to a tiny
  wall-clock span.

- **Allocate textures/pixel buffers once.** They are created in `Awake` and only
  re-filled (`SetPixels32`) each frame (`EconomyAnalyticsOverlay.cs#L89-L97`).
  Allocating a `Texture2D` or `Color32[]` inside `OnGUI` would churn GC every event.

- **Actual on-screen behaviour is `Needs Verification (in-game)`.** The draw code is
  source-verified, but whether Shift+G is unclaimed by the game, how the window
  renders at various resolutions/DPI, and input focus interactions with the game UI
  are runtime concerns not provable from source: `Needs Verification (in-game)`.

## Variations

- **Index-fallback X axis for near-simultaneous samples.** When the visible window's
  time span is degenerate (`timeRange <= 0.0015f`), the plotter spaces points evenly
  by index instead of by timestamp (`useIndexFallback`,
  `EconomyAnalyticsOverlay.cs#L560`) so a burst of same-tick samples still draws
  a readable line instead of collapsing onto one column.

- **Time-window zoom.** The overlay keeps a set of window durations
  (`{30s, 2m, 5m, 10m, All}`) and computes a `startIndex` into the ring for the
  selected span (`EconomyAnalyticsOverlay.cs#L41-L42`, `GetStartIndex`
  `#L714-L749`), so the same unbounded ring renders at multiple zoom levels without
  re-sampling.

- **Per-key series with filtering.** For many series (here, one price ring per
  `Resource`), store them in a dictionary and add a text filter + prev/next selector
  in the tab rather than plotting all at once
  (`DrawResourceSelectionUI`, `EconomyAnalyticsOverlay.cs#L436-L483`).

- **Config-driven cap.** Expose the ring cap as a setting - the recorder's
  `MaxSamples` setter clamps to a floor of 32 and re-trims immediately
  (`EconomyAnalyticsRecorder.cs#L38-L49`), so shrinking the budget frees memory at
  once instead of waiting for new samples to push old ones out.

## See also
- Related recipes: [render-pipeline-overlay](render-pipeline-overlay.md) (family I -
  the world-space TextMeshPro overlay this is explicitly *not*).
- Operations: [logging and debugging](../operations/logging-and-debugging.md),
  [memory and performance](../operations/memory-and-performance.md) (the bounded-ring
  memory discipline above).
- Case study demonstrating it:
  [market-based-economy](../../case-studies/market-based-economy.md) (its own
  "Diagnostics & Telemetry" surface).

## Sources
- Canonical mods (dossier + repo):
  - `market-based-economy` @b83f196a36bc74388accebdeb7c81f0f35dbab37 -
    `repo/Analytics/EconomyAnalyticsOverlay.cs`,
    `repo/Analytics/EconomyAnalyticsRecorder.cs`,
    `repo/Analytics/EconomyAnalyticsOverlayHost.cs`,
    `repo/Analytics/EconomyAnalyticsHotkey.cs`,
    `repo/Analytics/EconomyAnalyticsConfig.cs`, `repo/Mod.cs`
- Official/community references (link out, do not duplicate):
  https://docs.unity3d.com/Manual/GUIScriptingGuide.html (Unity IMGUI)
