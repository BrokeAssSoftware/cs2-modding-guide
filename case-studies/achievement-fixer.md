# Achievement Fixer

- Adds a short-lived `GameSystemBase` that runs for roughly 300 frames after load to restore `achievementsEnabled`.
- Hooks localisation changes to reapply option overrides and custom banner strings.
- Keeps settings and static state lightweight so the system idles when not needed.
- **Takeaway:** implement diagnostic or safeguard systems with tight execution windows and full localisation coverage.
