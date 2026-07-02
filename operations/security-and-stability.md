# Security and Stability

- Never download or execute external binaries at runtime; all dependencies must ship through Paradox Mods.
- Validate user input (ranges, enums, file paths) before applying it. Reject invalid values and surface clear error messages.
- Handle missing dependencies gracefully: disable dependent features, log a single warning, and keep the mod running.
- Strip developer-only commands and debug panels from release builds with conditional compilation or build-time flags.
- Review third-party contributions for suspicious IO or network access before merging.
