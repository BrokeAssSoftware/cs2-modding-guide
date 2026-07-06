---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Keep a CS2 Mod Secure and Stable"
Summary: Runtime-hardening practices for a CS2 mod - validate input, degrade gracefully when optional dependencies are missing, strip debug tooling from Release, avoid per-frame log spam, and never throw from OnUpdate.
diataxis: how-to
source_version: "n/a - concept/process page, no pinned source"
last_reverified: "2026-07-04"
status: needs-verification
Created: 2026-07-02
Updated: 2026-07-02
Owners:
  - codex
References:
  - Label: Log and Debug (log levels, developer commands)
    Path: ./logging-and-debugging.md
  - Label: Release Checklist
    Path: ./release-checklist.md
  - Label: Dependency Strategy (optional vs hard deps)
    Path: ../../explanation/dependency-strategy.md
  - Label: Technique Index
    Path: ../../technique-index.md
---

# Keep a CS2 Mod Secure and Stable

This page covers the runtime practices that keep a mod from corrupting saves, crashing
the simulation, or degrading performance for the whole city. They are habits to apply as
you write features, and they double as review criteria for contributions.

Companion pages: [Log and Debug](./logging-and-debugging.md) for the logging mechanics
referenced here, and the [Release Checklist](./release-checklist.md) for the gate that
enforces several of these before you ship.

---

## 1. Never fetch or execute external code

Ship everything through PDX Mods. Do not download, unpack, or execute external binaries
or scripts at runtime, and do not reach out to the network to pull code. Everything a
player runs must come through the same signed, moderated distribution channel as the
mod itself. Anything else is an unaudited code path on the player's machine.

## 2. Validate every input at the boundary

Treat all input as untrusted until you have checked it: settings values, values from
other mods, and any file path you read or write.

- **Clamp numeric ranges.** Reject or clamp out-of-range numbers before applying them;
  never feed an unchecked value straight into the simulation.
- **Validate enums and file paths.** Reject unknown enum values and paths that escape
  your expected directory.
- **Surface a clear message.** When you reject a value, log a specific reason (and,
  where appropriate, tell the player) rather than failing silently or crashing.

Validation at the boundary means the rest of your code can assume clean inputs.

## 3. Degrade gracefully when a dependency is missing

An optional dependency may not be installed. Detect that during load and disable the
dependent feature instead of crashing.

- **Check, then decide.** Resolve each optional dependency in `OnLoad`; if it is absent,
  disable the feature that needs it and leave the rest of the mod running.
- **Log the degradation once.** Emit a single `Warn` at load time ("feature X disabled:
  dependency Y not found"), not a repeated message. See
  [Log and Debug](./logging-and-debugging.md).
- **Distinguish optional from hard dependencies.** If a dependency is genuinely required,
  it must be declared in `PublishConfiguration.xml` (see the
  [Release Checklist](./release-checklist.md)) rather than handled at runtime. For the
  distinction, see [Dependency Strategy](../../explanation/dependency-strategy.md).

## 4. Strip developer tooling from Release builds

Developer-only commands, debug panels, and verbose diagnostics must not exist in a
shipped build. Guard them with conditional compilation or a build-time flag:

```csharp
#if DEBUG
    RegisterDeveloperCommand("mymod.dump", DumpState);
#endif
```

The [Release Checklist](./release-checklist.md) has you smoke-test the *Release* artifact
specifically to confirm this tooling is compiled out.

## 5. Do not spam the log every frame

A `Log.Info`/`Log.Warn` call inside `OnUpdate` (or any per-tick simulation path) runs
tens of times per second. That floods `Player.log`, hides real messages, and measurably
hurts frame time.

- **Log transitions, not state.** Emit a line when something *changes*, not while it
  stays the same.
- **Gate verbose diagnostics.** Put detailed per-operation logging behind a settings
  toggle or `#if DEBUG` so it is off by default.

## 6. Never throw out of OnUpdate (or other per-frame paths)

An unhandled exception thrown from a per-frame or per-tick method fires every frame and
can spam errors, stall the simulation, or take down the mod. Systems that run on the
game loop must not let exceptions escape.

- **Contain failures.** Wrap risky per-frame work so a failure logs once and the system
  keeps running (or disables itself) instead of throwing on every subsequent frame.
- **Fail once, then stop.** If a per-frame operation cannot recover, disable that path
  and log a single error - do not re-throw the same exception 60 times a second.
- **Validate up front (see section 2)** so the per-frame path never receives the bad
  input that would make it throw.

## 7. Review contributions for unsafe IO and network access

Before merging third-party contributions, scan for the failure modes above - especially
new file/network IO, new external process calls, un-gated developer commands, and
per-frame logging or throwing. A contribution that reintroduces any of these is a
regression regardless of the feature it adds.

---

## Where to go next

- The logging mechanics behind sections 3, 5, and 6:
  [Log and Debug](./logging-and-debugging.md).
- The gate that enforces "Release strips debug" and "dependencies declared":
  [Release Checklist](./release-checklist.md).
- Optional vs. hard dependencies:
  [Dependency Strategy](../../explanation/dependency-strategy.md).
- Pick your next technique from the [Technique Index](../../technique-index.md).
