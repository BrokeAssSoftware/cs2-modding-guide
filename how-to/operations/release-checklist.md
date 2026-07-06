---
FrontmatterVersion: 1
DocumentType: Guide
Title: "Release Checklist for a CS2 Mod"
Summary: A pre-publish gate for shipping a CS2 mod - build in Release, smoke-test against regression saves, verify declared dependencies, bump the version and changelog, publish, and tag.
diataxis: how-to
source_version: "n/a - concept/process page, no pinned source"
last_reverified: "2026-07-04"
status: needs-verification
Created: 2026-07-02
Updated: 2026-07-02
Owners:
  - codex
References:
  - Label: Build, Test, and Publish (publish walkthrough)
    Path: ../../tutorials/build-and-publish.md
  - Label: Log and Debug (regression saves, Player.log)
    Path: ./logging-and-debugging.md
  - Label: Security and Stability (strip debug in Release)
    Path: ./security-and-stability.md
  - Label: Dependency Strategy (declaring and testing deps)
    Path: ../../explanation/dependency-strategy.md
  - Label: Technique Index
    Path: ../../technique-index.md
---

# Release Checklist for a CS2 Mod

Run this checklist every time you ship a new version to **PDX Mods**. It is a gate, not
a suggestion: each step catches a class of "works on my machine" failure before players
hit it. For the full publish walkthrough (IDE commands, `PublishConfiguration.xml`),
see [Build, Test, and Publish](../../tutorials/build-and-publish.md).

Work top to bottom; do not skip a step because "nothing changed there."

---

## The checklist

1. **Build in Release.**
   ```powershell
   dotnet build -c Release
   ```
   Also run `npm run build` if your mod ships a UI bundle. Release strips debug symbols
   and (if you gated them correctly) your developer commands and debug panels - confirm
   they are gone. See [Security and Stability](./security-and-stability.md).

2. **Smoke-test the Release build.** Launch the game, load your
   [regression saves](./logging-and-debugging.md), and exercise every critical feature.
   Test the *Release* artifact, not the Debug build you have been developing against -
   conditional compilation means they are not the same binary.

3. **Review the log.** After the smoke test, read `Player.log` (and the dated
   `Log_<date>.txt`) at:
   ```
   %AppData%\LocalLow\Colossal Order\Cities Skylines II\Logs\
   ```
   There must be no new warnings or exceptions attributable to your mod. A dirty log is
   a failed release.

4. **Verify declared dependencies.** Open `PublishConfiguration.xml` and confirm every
   mod your build hard-requires (a shared icon library, a localization framework, and so
   on) is listed by **both name and mod ID**. A dependency added mid-development but
   missing here is the classic failure that loads fine for you and breaks for everyone
   else. See [Dependency Strategy](../../explanation/dependency-strategy.md).

5. **Update the changelog.** Write concise, player-facing release notes: what changed,
   what is fixed, what to expect. This is what players read to decide whether to update.

6. **Bump the version.** Increment the version and make it match the changelog exactly,
   so a player can map an installed version to a set of notes. Version and notes drift
   apart the moment you let them.

7. **Publish.** From your IDE, run the update-an-existing-mod publish command (for
   example `PublishNewVersion`) as described in
   [Build, Test, and Publish](../../tutorials/build-and-publish.md). Confirm the
   uploaded archive contains **both** the compiled C# and the UI assets - a publish that
   silently omits the UI bundle is a common regression.

8. **Tag the release.** Create a git tag for the shipped version and record the Paradox
   Mod ID next to it. This gives you a reproducible point to branch a hotfix from and a
   durable link between "the code" and "the listing."

---

## Notes

- **`Player.log` is a gate, not a formality.** Steps 2-3 exist to catch runtime failures
  that a clean compile hides. Reserve time for them.
- **Re-verify dependencies every release.** Dependency needs drift as the mod grows;
  re-confirm step 4 even on a "small" patch.
- **One version, everywhere.** The version in `PublishConfiguration.xml`, the changelog,
  and the git tag should all agree. Reconcile them before publishing, not after.

---

## Where to go next

- Full publish mechanics: [Build, Test, and Publish](../../tutorials/build-and-publish.md).
- Why dependencies must be declared and how to test their absence:
  [Dependency Strategy](../../explanation/dependency-strategy.md).
- Runtime hardening that keeps releases stable:
  [Security and Stability](./security-and-stability.md).
