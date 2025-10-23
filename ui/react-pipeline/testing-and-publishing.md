# Testing and Publishing

## Testing
- Run `npm run lint` and `npm run typecheck` locally or in CI.
- Build with `npm run build` and open `dist/index.html` for a quick smoke test.
- Launch the game without `npm run dev` to ensure the production bundle loads correctly.
- Test in multiple languages and without shared dependencies (UIL, ExtraLib) to confirm fallbacks.
- Profile with the Gameface inspector or Chromium dev tools to keep components light.

## Publishing
1. Run `npm run build` to produce the bundle.
2. Run `dotnet build -c Release` so MSBuild copies the bundle alongside the mod.
3. Publish via the in-game toolchain and verify the uploaded archive contains the UI `dist/` files.
4. Tag the release and note the bundle version in release notes for automation tracking.
