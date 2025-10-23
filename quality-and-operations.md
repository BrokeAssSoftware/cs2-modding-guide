# Quality and Operations

Reliability matters when multiple Vice & Order modules share a save. This guide covers logging, debugging, performance hygiene, security, release management, and automation so contributors can diagnose issues quickly and ship stable builds.

## Logging Strategy
- Create a single logger per module:
  ```csharp
  internal static readonly ILog Log = LogManager
      .GetLogger("VNO.Vice")
      .SetShowsErrorsInUI(false);
  ```
- Log the executable asset path, toolchain version, and dependency status during `OnLoad`.
- Use structured messages (`Log.Info($"Heat changed | district={districtId} | value={heat}")`) so log parsing scripts can extract fields.
- Gate verbose output behind settings toggles or `#if DEBUG` blocks. Never leave high-volume logging enabled in release builds.
- Write large diagnostics (JSON dumps, telemetry exports) to `ModsDataTemp/<Module>` and timestamp them so they can be purged safely.

## Debugging Toolkit
- **Debug builds** - build with `dotnet build -c Debug` to keep symbols and optimiser-friendly code.
- **Attach IDEs** - Rider and Visual Studio can attach to `Cities2.exe`; use conditional breakpoints to avoid pausing every frame.
- **Developer mode** - use `-developerMode` to unlock simulation speed controls, the object browser (`Home`), and console commands.
- **UI debugging** - run `npm run dev` for hot reload and inspect React trees via `http://localhost:9444/`.
- **Regression saves** - maintain a curated set of saves (traffic stress, budget collapse, vice escalation) under `docs/regression/` with notes describing expected behaviour.
- **Diagnostic commands** - expose developer-only commands (for example `vno.heat.dump`) that output state snapshots without requiring external tools.

## Memory and Performance Hygiene
- Dispose `NativeArray`, `NativeList`, `BlobAssetReference`, and `NativeSlice` instances. Use `using` blocks or explicit `Dispose()` in `OnDestroy`.
- Avoid capturing `Entity` or `ComponentLookup` inside lambdas that run asynchronously; copy values locally before scheduling jobs.
- Batch structural changes. Collect entities in a `NativeList<Entity>` and process them in a single `EntityCommandBuffer` rather than calling `EntityManager` repeatedly.
- If a system allocates temporary memory, annotate the code with `ProfilerMarker` scopes and watch allocations in Unity Profiler.
- Use job-friendly patterns (`ScheduleParallel`) where possible, and prefer `WithReadOnly` / `WithDisposeOnCompletion` to keep the Burst compiler happy.

## Security and Stability Controls
- Never download or execute external binaries at runtime. All dependencies must ship through Paradox Mods.
- Validate configuration input (ranges, enums, file paths) before applying it. Reject invalid values and surface a clear message in the Options UI.
- Handle missing dependencies gracefully: disable dependent features, log a single warning, and keep the mod running.
- Strip developer-only commands and debug panels from release builds with conditional compilation or build-time flags.
- Review third-party contributions for suspicious IO or network calls before merging.

## Release Checklist
1. **Build** - run `dotnet build -c Release` and `npm run build` (if applicable).
2. **Smoke test** - launch the game, load regression saves, and exercise critical features.
3. **Logs** - inspect `%AppData%\LocalLow\Colossal Order\Cities Skylines II\Logs\Log_<date>.txt` for warnings or errors introduced by the change.
4. **Dependencies** - confirm `PublishConfiguration.xml` lists every required mod (ExtraLib, UIL, I18n Everywhere, etc.).
5. **Changelog** - update the module changelog and Paradox Mods release notes with concise, player-friendly summaries.
6. **Publish** - use the in-game publisher (`PublishNewVersion`) and verify the uploaded package contains both C# and UI bundles.
7. **Tag** - create a git tag and link the Paradox Mod ID for traceability.

## Automation and CI Ideas
- **Build validation** - run `dotnet build`, `npm run build`, and linting tasks on every pull request.
- **Manifest linting** - validate YAML/JSON policy packs and module manifests against a schema to catch missing fields.
- **Wiki sync** - script periodic fetches of official wiki pages (`docs/research/wiki/`) and diff changes; flag notable updates that may require guide revisions.
- **Dependency audit** - add a check that compares live files against `PublishConfiguration.xml` to ensure dependency lists stay in sync.
- **Telemetry smoke tests** - run scripted city simulations (headless or accelerated) and assert that logs do not exceed defined error thresholds.

## Incident Response Playbook
- Capture affected save files, logs, and the list of active mods from players reporting bugs.
- Attempt to reproduce using the closest regression save; document reproduction steps in an issue tracker.
- If the incident is regression-worthy, cut a hotfix branch, write automated tests to cover the scenario, and re-run the release checklist before publishing the fix.
- Share a short post-mortem in `docs/ops/reports/` when the incident surfaces a new process or automation gap.

Keeping these practices in place ensures Vice & Order modules remain stable, debuggable, and easy to support across multiple codebases and contributors.
