# Conditional Execution and Diagnostics

- Register systems only when configuration enables the feature; otherwise return early in `OnUpdate`.
- Use lightweight components (`EnabledRefRW`, shared singletons) to flip behaviour without rebuilding worlds.
- Pair DOTS systems with Harmony patches for high-level API hooks and unpatch them in `OnDispose` to support hot reload.
- Wrap heavy code paths in `ProfilerMarker` scopes and expose developer commands behind `-developerMode` for diagnostics and dumps.
