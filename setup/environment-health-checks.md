# Environment Health Checks

Run these checks after setting up the toolchain and periodically during development to catch environment drift early.

- Confirm `ModsSettings/<ModuleId>` and `ModsData/<ModuleId>` folders exist after the first run; missing folders usually mean the mod failed to load.
- Install the Gameface developer certificate (`npm run install-cert` from the UI scaffold) once per machine to avoid HTTPS warnings when opening `http://localhost:9444/`.
- Maintain at least one sandbox save with all unlocks for regression testing.
- Document any additional prerequisites (database dumps, telemetry proxies) in the module's `Agents.md`.
- Review `%AppData%\LocalLow\Colossal Order\Cities Skylines II\Logs\Log_<date>.txt` during each test session and address new warnings promptly.
