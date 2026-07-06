---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Log and Debug a CS2 Mod"
Summary: Set up a per-mod logger, choose log levels, wire developer commands and hot reload, and capture regression saves plus Player.log so failures are reproducible and diagnosable.
diataxis: how-to
source_version: "n/a - process page; illustrative mechanisms pinned per canonical_mods"
last_reverified: "2026-07-05"
canonical_mods:
  - vno-debug@8d1cbb848850d152430c40aa374d970dc7618bae
  - demand-modifier@e93ec1c1acdb58764c7949c4fca1654d451b690b
  - anarchy@a6311e898d20a775368668b234aaa32f06e3e1eb
  - write-everywhere@13c70eb04e6bed152257c516a982455148f591a5
status: source-verified
Created: 2026-07-02
Updated: 2026-07-05
Owners:
  - codex
References:
  - Label: Build, Test, and Publish (daily loop)
    Path: ../../tutorials/build-and-publish.md
  - Label: Technique Index
    Path: ../../technique-index.md
  - Label: Security and Stability (avoid log spam, don't throw in OnUpdate)
    Path: ./security-and-stability.md
  - Label: Release Checklist
    Path: ./release-checklist.md
---

# Log and Debug a CS2 Mod

This page shows you how to make a mod observable: a single named logger, sensible
log levels, developer commands and hot reload for live inspection, and a repeatable
way to capture the artifacts (regression saves + `Player.log`) that turn a vague bug
report into a reproducible failure.

It assumes you already have a building mod and the daily loop from
[Build, Test, and Publish](../../tutorials/build-and-publish.md). Throughout, the
placeholder mod ID is `MyMod.Core`.

---

## 1. Create one named logger per mod

CS2 mods log through the game's `Colossal.Logging` framework, not `Console` or
`System.Diagnostics`. Create a single static logger, name it after your mod, and reuse
it everywhere. A per-mod name makes your lines greppable in the shared game log.

```csharp
using Colossal.Logging;   // ILog, LogManager

public static readonly ILog s_Log =
    LogManager.GetLogger(ModId).SetShowsErrorsInUI(false);
```

Source: `magic-mail` `repo/Mod.cs#L44-45` @ `6fb3d2b` (River-Mochi/MagicMail). The mod
passes its own `ModId` string as the logger name; use your mod's ID (for example
`"MyMod.Core"`).

`SetShowsErrorsInUI(false)` suppresses the game's on-screen error popup for lines this
logger emits. Set it to `false` for routine/expected conditions so you do not spam
players with red banners; leave it `true` (or omit the call) only when a logged error
genuinely warrants interrupting the user.

**Guidelines**

- **One logger, reused.** Do not call `GetLogger` per class or per frame - create the
  static field once and reference it. Multiple loggers with different names fragment
  the log and make filtering harder.
- **Name it after the mod, not the class.** A stable prefix (`MyMod.Core`) lets you and
  players filter the shared log to just your mod's lines.
- **Log a startup banner in `OnLoad`.** Record mod version, the executing assembly
  path, the CS2/SDK version you built against, and the resolved status of each optional
  dependency. This one block answers most "which build / what environment" questions
  before you ever open a debugger.
- **Wrap each bootstrap phase in its own `try/catch`.** Guard logging setup,
  localization, settings registration, and Harmony patching as separate tagged phases so
  a failure in one logs its message and stack trace but the remaining phases still run,
  rather than aborting `OnLoad` wholesale. Each phase logs an enter line and a
  pass/fail outcome marker so the last successful line pinpoints where load stopped.
  Source: `../../../vice-and-order-research/mods/dossiers/demand-modifier/repo/DemandModifier/DemandModifierMod.cs#L51-L143` (@e93ec1c1acdb58764c7949c4fca1654d451b690b).

---

## 2. Choose the right log level

Use levels deliberately so the default log stays readable and verbose diagnostics are
opt-in:

- **Info** - lifecycle milestones and one-time facts: load banner, feature enabled or
  skipped, dependency resolved or missing. Emitted once, not per frame.
- **Warn** - a recoverable degraded state: an optional dependency is absent so a feature
  is disabled, a config value was clamped to a valid range. Log it **once**, then
  continue.
- **Error** - a real failure you caught and handled. Include enough context to
  reproduce. If it should surface to the player, allow it through the UI; otherwise keep
  it log-only via `SetShowsErrorsInUI(false)`.
- **Debug / Trace (verbose)** - detailed per-operation diagnostics. Gate these behind a
  settings toggle or `#if DEBUG` so they never run in a shipped Release build.

**Set the log's effective level by build tier.** Colossal's `ILog` exposes an
`effectivenessLevel` you can pin at compile time so the verbose tier is a build you opt
into, not a runtime cost you ship. Define a `VERBOSE` symbol for the noisiest build and
fall through `DEBUG` to `Info` for Release:

```csharp
// #define VERBOSE   // top of file; uncomment for a verbose build
Log = LogManager.GetLogger("Mods_Yenyang_Anarchy").SetShowsErrorsInUI(false);
#if VERBOSE
    Log.effectivenessLevel = Level.Verbose;
#elif DEBUG
    Log.effectivenessLevel = Level.Debug;
#else
    Log.effectivenessLevel = Level.Info;
#endif
```

Source: `../../../vice-and-order-research/mods/dossiers/anarchy/repo/Anarchy/AnarchyMod.cs#L5`, `#L83-L91` (@a6311e898d20a775368668b234aaa32f06e3e1eb).

**Write structured, parseable messages.** Use a consistent `key=value` shape so you (or
tooling) can filter and diff logs:

```csharp
s_Log.Info($"feature=districtStats districts={districtCount} enabled={enabled}");
```

Keep large diagnostic dumps out of the main log. Write them to a temp data folder under
your mod (for example `ModsDataTemp/MyMod`) and log only the path.

### Optional: one JSON object per line, with a run id

If you post-process logs with tooling, go a step past `key=value` and emit **one JSON
object per line** behind a fixed prefix. A tiny hand-rolled stringifier (no serializer
dependency) writes a stable envelope - timestamp, run id, category, name, payload, and a
context object - and wraps the whole thing in `try/catch` so a bad payload degrades to a
`Warn`, never an exception on the log path:

```csharp
var sb = new StringBuilder(256);
sb.Append("[VNO-DEBUG] {");
AppendProp(sb, "ts", DateTime.UtcNow.ToString("o", CultureInfo.InvariantCulture));
sb.Append(','); AppendProp(sb, "run", Mod.RunId());
sb.Append(','); AppendProp(sb, "cat", cat);
sb.Append(','); AppendProp(sb, "name", name);
sb.Append(",\"payload\":"); sb.Append(ObjectToJson(payload));
sb.Append(",\"ctx\":");     sb.Append(ObjectToJson(ctx));
sb.Append('}');
Mod.Log.Info(sb.ToString());
```

Source: `../../../vice-and-order-research/mods/dossiers/vno-debug/repo/Utilities/LogJson.cs#L12-L33` (@8d1cbb848850d152430c40aa374d970dc7618bae).

Mint the **run id once per session** (a UTC timestamp plus a short random suffix, cached
in a static) so every line from one game session shares a key you can group and diff on.
Source: `../../../vice-and-order-research/mods/dossiers/vno-debug/repo/Mod.cs#L49-L57` (@8d1cbb848850d152430c40aa374d970dc7618bae).

To make "don't spam the log" enforceable instead of aspirational, put a **token-bucket
line budget** in front of your emitter: refill up to `MaxLinesPerSecond` each second and
spend one token per line, so a runaway loop self-limits rather than flooding `Player.log`:

```csharp
// in OnUpdate, gated on settings.Enabled
s_LineBudget = Math.Min(settings.MaxLinesPerSecond,
                        s_LineBudget + settings.MaxLinesPerSecond);
```

Source: `../../../vice-and-order-research/mods/dossiers/vno-debug/repo/VnoDebugSystem.cs#L38-L42` (@8d1cbb848850d152430c40aa374d970dc7618bae).

> **Do not spam the log every frame.** A per-frame `Info`/`Warn` call in `OnUpdate`
> floods `Player.log` and tanks frame time. Log state *transitions*, not state. See
> [Security and Stability](./security-and-stability.md) for the per-frame rules.

---

## 3. Debug with a debugger attached

1. **Build in Debug** to keep symbols:
   ```powershell
   dotnet build -c Debug
   ```
2. **Attach to `Cities2.exe`** from Rider or Visual Studio (or use the VS Code
   rebuild-and-attach setup in
   [Build, Test, and Publish](../../tutorials/build-and-publish.md)).
3. **Prefer conditional breakpoints.** Anything in the simulation or an `OnUpdate` path
   runs every frame; an unconditional breakpoint there pauses constantly. Condition on
   the entity, district, or value you actually care about.

---

## 4. Use developer mode, commands, and hot reload

Launch the game with `-developerMode` (see the getting-started/toolchain notes) to
unlock live inspection:

- **Simulation controls** - pause and step time to isolate a per-tick bug.
- **Object browser** - open with the `Home` key to inspect entities and components at
  runtime.
- **Console commands** - expose developer-only commands for targeted diagnostics, for
  example a `mymod.<command>` that dumps your subsystem's current state on demand.
  Namespace them under your mod ID so they are easy to find and unlikely to collide.
- **UI hot reload** - if your mod has UI, run `npm run dev` and (with
  `-uiDeveloperMode`) the game loads UI assets live from the dev server. Inspect the
  React tree in a browser at the dev server URL.

### On-demand snapshot and one-frame render dumps

Rather than logging continuously, expose an on-demand *dump* that a command sets a flag
for and a system drains on its next update. Two patterns are worth copying:

- **Gated reflection snapshot.** A debug system enumerates loaded assemblies via
  reflection to dump the live ECS surface - every `ComponentSystemBase`/`SystemBase`
  subtype, and every `IComponentData`/`IBufferElementData`/`ISharedComponentData` type
  with its public fields - grouped by namespace. The whole walk is gated behind a
  settings toggle, triggered by `RequestDump()` (a command sets a static flag that
  `OnUpdate` consumes once), and capped by `MaxItemsPerDump`/`MaxFieldDepth` plus include
  /exclude namespace filters so a dump can never run in a hot path or blow up the log.
  Source: `../../../vice-and-order-research/mods/dossiers/vno-debug/repo/VnoDebugSystem.cs#L62`, `#L142-L226` (@8d1cbb848850d152430c40aa374d970dc7618bae).
- **One-frame render dump.** For a UI or render-desync repro, wire a dev command to a
  `static bool dumpNextFrame`. The render system checks the flag inside its per-item
  loop, emits detailed lines (entities, matrices, effective values, buffer/mesh
  validity) for exactly that pass, then clears the flag at the end - so you capture one
  frame's worth of render state without leaving per-frame logging on.
  Source: `../../../vice-and-order-research/mods/dossiers/write-everywhere/repo/BelzontWE/Controllers/WEWorldPickerController.cs#L130-L133`, `../../../vice-and-order-research/mods/dossiers/write-everywhere/repo/BelzontWE/Systems/WERendererSystem.cs#L26`, `#L188`, `#L197` (@13c70eb04e6bed152257c516a982455148f591a5).

> **Ship-safe rule:** developer commands and debug panels must not exist in a Release
> build. Compile them out with `#if DEBUG` or a build-time flag - see
> [Security and Stability](./security-and-stability.md).

---

## 5. Capture reproducible failures

A bug you cannot reproduce cannot be fixed. Capture two artifacts for every non-trivial
issue.

### Regression saves

Keep a small library of saves that reliably trigger the conditions your mod touches -
for example a heavy-traffic city, a budget-collapse city, or whatever stress your
feature responds to. For each save, write a short note describing the *expected*
behavior so anyone can tell pass from fail. Reload these before every release (see the
[Release Checklist](./release-checklist.md)).

> **Tip:** keep one sandbox save with everything unlocked purely for fast, consistent
> testing.

### Player.log

The game writes a per-session log you should read after every test run and attach to
every bug report:

```
%AppData%\LocalLow\Colossal Order\Cities Skylines II\Logs\Player.log
```

Dated per-run logs also appear alongside it as `Log_<date>.txt`. Scan for new warnings
or exceptions after each run and fix them before moving on - a clean log is part of the
release gate.

When reporting or filing a bug, pair the **regression save** with the **`Player.log`**
from the run that failed. Together they let anyone reproduce the exact state.

### UI.log (Cohtml) - for UI-desync repros

If your mod has a Cohtml/React UI and the bug is a state desync between the C# side and
the panel, `Player.log` alone often will not show it. The game's UI host writes its own
log alongside `Player.log`; capture it too, and pair it with a **one-frame dump** (the
`dumpNextFrame` pattern above) so the render state and the UI state are timestamped from
the same frame. The exact UI log filename and whether it needs `-uiDeveloperMode` to be
populated is `Needs Verification (in-game)`.

---

## Where to go next

- Fold these habits into your ship gate: [Release Checklist](./release-checklist.md).
- Harden the runtime behavior behind your logs:
  [Security and Stability](./security-and-stability.md).
- Pick your next technique from the [Technique Index](../../technique-index.md).
