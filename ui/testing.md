# UI Testing Checklist

1. Build in Debug configuration and verify the Options menu renders correctly in at least two languages.
2. Change each setting, restart the game, and confirm persistence.
3. Rebind every key and ensure conflicts produce visible warnings.
4. Launch without shared dependencies (ExtraLib, I18n Everywhere, UIL) and confirm graceful fallbacks.
5. Enable `-developerMode` to verify debug-only sections remain hidden in release builds.
