# Release Checklist

1. **Build** - run `dotnet build -c Release` and `npm run build` if a UI bundle exists.
2. **Smoke test** - launch the game, load regression saves, and exercise critical features.
3. **Review logs** - inspect `%AppData%\LocalLow\Colossal Order\Cities Skylines II\Logs\Log_<date>.txt` for new warnings or errors.
4. **Verify dependencies** - ensure `PublishConfiguration.xml` lists every required mod (ExtraLib, UIL, I18n Everywhere, etc.).
5. **Update changelog** - publish concise, player-focused release notes.
6. **Publish** - use the in-game publisher (`PublishNewVersion`) and confirm the uploaded archive contains both C# and UI assets.
7. **Tag** - create a git tag and record the Paradox Mod ID for traceability.
