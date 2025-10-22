# Quality And Operations

Robust Vice & Order modules rely on disciplined debugging, memory hygiene, and operational safeguards.

## Logging

- Use the `Colossal.Logging` API; create per-module loggers and store them on static fields.
- Control verbosity through configuration flags (`LogLevel`) or `#if DEBUG` blocks.
- Log Harmony patch lists, folder locations, and detected companion mods during `OnLoad`.
- Redirect heavy diagnostic dumps to files under `ModsDataTemp/<Mod>` to preserve runtime performance.

## Debugging

- Attach Visual Studio or Rider to the `Cities2.exe` process; enable symbols by building in Debug configuration.
- Use developer mode (`--developerMode`) to toggle simulation speed, unlock milestones, and inspect object IDs.
- For UI issues, run `npm run dev` and inspect with the Gameface debugger at `http://localhost:9444/`.
- Maintain a library of regression saves targeting common scenarios (traffic spike, budget collapse, high crime) for quick reproduction.

## Memory & Performance

- Dispose `NativeArray`, `NativeList`, and `BlobAssetReference` instances in `OnDestroy` or when scope ends (see `how_to_avoid_memory_leaks.md`).
- Avoid capturing `Entity` references inside lambdas that outlive their chunk iteration.
- Batch component queries to reduce sync points; prefer `Entities.ForEach().ScheduleParallel()` when no structural changes occur.
- Profile with Unity Profiler or Visual Studio performance tools to catch allocations per frame.

## Security & Stability

- Never execute external binaries or download code at runtime; follow guidance in `mod_security.md`.
- Validate user input from settings before applying (ranges, enums) to avoid corrupting simulation data.
- Handle missing dependencies gracefully; log and disable features rather than throwing.
- Strip debug-only commands from release builds to reduce attack surface.

## Automation & Tooling

- Plan linters/checkers for YAML policy packs and module manifests (see TODO in `Agents.md`).
- Add CI hooks to run `dotnet build`, `npm run build`, and unit/integration tests once available.
- Script periodic wiki snapshot comparisons to surface API changes and feed this guide.
- Keep issue templates for module bugs (system name, logs, reproduction steps) to streamline support.
