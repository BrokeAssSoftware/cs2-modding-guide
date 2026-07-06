---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Respond to a Mod-Breaking Incident"
Summary: A repeatable process for when a CS2 mod breaks after a game patch or a report - collect the log and save, reproduce, isolate the cause, ship a hotfix through the release gate, and communicate with players.
diataxis: how-to
source_version: "n/a - concept/process page, no pinned source"
last_reverified: "2026-07-04"
status: needs-verification
Created: 2026-07-03
Updated: 2026-07-03
Owners:
  - codex
References:
  - Label: Log and Debug (Player.log, regression saves)
    Path: ./logging-and-debugging.md
  - Label: Testing and Recovery (reproduce, isolate, safe teardown)
    Path: ./testing-and-recovery.md
  - Label: Release Checklist (the gate a hotfix must pass)
    Path: ./release-checklist.md
  - Label: Build, Test, and Publish (branch and publish the hotfix)
    Path: ../../tutorials/build-and-publish.md
  - Label: Technique Index
    Path: ../../technique-index.md
---

# Respond to a Mod-Breaking Incident

Cities: Skylines II ships patches on its own cadence, and a game update can break a mod
that worked yesterday - a vanilla system your mod patched changed shape, a component
moved, a save format shifted. Player reports also arrive without a patch. This page is
the process to run when that happens: contain the damage, find the cause, ship a fix, and
tell players. Work it in order; the early steps make the later ones possible.

This is a companion to [Testing and Recovery](./testing-and-recovery.md) (how to make a
mod survive a missing dependency in the first place) and the
[Release Checklist](./release-checklist.md) (the gate every fix, including a hotfix, must
pass).

---

## 1. Collect the artifacts

You cannot fix what you cannot see. For every report, gather the three artifacts that
together reconstruct the failure:

- **`Player.log`** from the failed session, at
  `%AppData%\LocalLow\Colossal Order\Cities Skylines II\Logs\Player.log` (plus the dated
  `Log_<date>.txt`). This is where exceptions and your mod's own log lines land - see
  [Log and Debug](./logging-and-debugging.md).
- **The save file** that exhibits the problem, if the player can share it.
- **The environment**: the game version, your mod's version, and the **full list of other
  active mods**. A break that only reproduces with a specific mod combination is a
  compatibility bug, not a bug in your code alone - the mod list is what tells them apart.

Ask for all three up front. A report of "it crashes" with none of them cannot be actioned;
a good bug-report template asks for the log, the save, and the mod list by name.

## 2. Reproduce

An intermittent, unreproduced failure cannot be verified as fixed. Turn the report into a
repeatable failure:

1. Load the reported save (or the closest [regression save](./logging-and-debugging.md))
   on the reported game version.
2. Follow the reported steps and confirm you see the same log signature or crash.
3. If it only reproduces with other mods present, rebuild that mod set - the interaction
   is the bug.

Capture the reproduction as a note next to the save so anyone can trigger it. If you
cannot reproduce, you are not ready to fix; keep narrowing the environment until you can.

## 3. Isolate the cause

Once it reproduces reliably, find the smallest thing that makes it go away:

- **Bisect against the patch.** If the break coincided with a game update, the vanilla
  surface your mod touches (a patched system, a component layout, a save version) is the
  first suspect. Compare what changed.
- **Toggle features.** Disable your mod's features one at a time (or via settings) to find
  which subsystem triggers it. Toggling the whole mod off should make the symptom vanish -
  if it does not, the mod may not be the cause.
- **Read the stack, not the symptom.** The exception in `Player.log` usually names the
  method and system; trace from there rather than from the player's description of what
  they saw.
- **Confirm it is yours.** A dirty log with an exception from another mod (or vanilla) is
  not your incident. Isolate before you commit to a fix.

See [Testing and Recovery](./testing-and-recovery.md) for the toggle-off / dependency-
absent checks that make this isolation fast.

## 4. Hotfix

Fix the isolated cause on a branch, then run the full ship gate - a hotfix is still a
release:

1. **Branch from the shipped tag.** Create a hotfix branch from the exact
   [git tag of the released version](./release-checklist.md) so you fix *what players are
   running*, not your in-progress `main`.
2. **Write the narrowest fix** that addresses the isolated cause. Resist bundling unrelated
   changes into a hotfix - they widen the blast radius.
3. **Add coverage.** Where you can, add a regression save (or a test) that reproduces the
   incident, so a future change cannot silently reintroduce it.
4. **Re-run the [Release Checklist](./release-checklist.md) in full.** Build in Release,
   smoke-test against the reproducing save, confirm a clean `Player.log`, re-verify
   declared dependencies, bump the version and changelog, publish, and tag. Do not skip
   steps because "it is only a hotfix" - the checklist exists precisely for the rushed
   fix. Publish mechanics are in [Build, Test, and Publish](../../tutorials/build-and-publish.md).

## 5. Communicate

Players who hit the bug need to know it is being handled, and players who have not yet
updated need to know whether to hold:

- **Acknowledge early.** A short "known issue, investigating, expected fix window" on your
  mod's page or channel prevents duplicate reports and buys you time to fix it right.
- **Write player-facing release notes.** When the hotfix ships, the changelog should name
  what broke and what is fixed in plain terms - this is what a player reads to decide
  whether to update (see the [Release Checklist](./release-checklist.md) changelog step).
- **Close the loop.** Reply to the reports that the fix has shipped and in which version,
  so the reporters can confirm.

## 6. Learn from it

An incident that reveals a process gap should change the process, not just the code:

- **Write a short post-mortem** - what broke, why the existing gate did not catch it, and
  the concrete change that would. A couple of paragraphs in your repo (for example under a
  `docs/incidents/` folder) is enough; the point is a durable record, not ceremony.
- **Add the missing guard.** If a game patch broke a vanilla surface you depend on, add a
  regression save or a version check that would have caught it. If a mod combination broke,
  add it to your test matrix.
- **Fold it back into the gate.** The best outcome of an incident is a new line on the
  [Release Checklist](./release-checklist.md) or a new regression save, so the next release
  cannot repeat it.

---

## Notes

- **Re-verify against every game patch.** `reference/game-systems/*` and any vanilla system
  you patch are patch-sensitive. Treat a new CS2 release as a prompt to smoke-test your mod
  before players do - proactive re-verification turns an incident into a routine update.
- **A missing artifact stalls the whole process.** Steps 2-4 all depend on step 1. Invest
  in a bug-report template that asks for the log, save, and mod list so you are not chasing
  them mid-incident.

## Where to go next

- Make future incidents rarer: [Testing and Recovery](./testing-and-recovery.md) and
  [Security and Stability](./security-and-stability.md).
- The gate every hotfix passes: [Release Checklist](./release-checklist.md).
- Publish the hotfix: [Build, Test, and Publish](../../tutorials/build-and-publish.md).
- Pick your next technique: [Technique Index](../../technique-index.md).
