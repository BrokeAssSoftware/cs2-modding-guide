# Communication with C#

- Share simulation data via DTO buffers or message channels exposed on the C# side.
- For simple configuration, read `ModsSettings` through `cs2/modding.ModSettings` so React components start with correct defaults.
- When UI actions need to trigger simulation work, expose a service method (for example `ViceCommandBus.Enqueue`) and call it through a thin API wrapper.
- Keep payloads small—Gameface marshals data via JSON.
