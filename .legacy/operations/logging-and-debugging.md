# Logging and Debugging

## Logging
- Create a single logger per module:
  ```csharp
  internal static readonly ILog Log = LogManager
      .GetLogger("VNO.Vice")
      .SetShowsErrorsInUI(false);
  ```
- Log executable asset paths, toolchain version, and dependency status during `OnLoad`.
- Use structured messages (for example `Log.Info($"Heat changed | district={districtId} | value={heat}")`) so automation can parse logs.
- Gate verbose output behind settings toggles or `#if DEBUG` blocks and write large diagnostics to `ModsDataTemp/<Module>`.

## Debugging Toolkit
- Build with `dotnet build -c Debug` to retain symbols.
- Attach Rider or Visual Studio to `Cities2.exe`; rely on conditional breakpoints to avoid pausing every frame.
- Use `-developerMode` to access simulation speed controls, the object browser (`Home`), and console commands.
- Run `npm run dev` for UI hot reload and inspect React trees via `http://localhost:9444/`.
- Maintain regression saves (traffic stress, budget collapse, vice escalation) with notes describing expected behaviour.
- Expose developer-only commands such as `vno.heat.dump` for targeted diagnostics.
