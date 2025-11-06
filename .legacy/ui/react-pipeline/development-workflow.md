# Development Workflow

1. Scaffold the UI project with `npx create-csii-ui-mod` inside the code mod folder.
2. Install dependencies (`npm install`) if the scaffold skipped them and ensure `mod.json.id` matches the C# assembly.
3. Wire MSBuild targets so `dotnet build` also runs `npm run build` and copies the UI bundle into the deploy directory.
4. Run `npm run dev` for hot reload, launch the game with `-uiDeveloperMode`, and use `http://localhost:9444` to inspect component state.
5. Use CSS modules and memoisation (`useMemo`, `useCallback`) to keep runtime updates efficient.
