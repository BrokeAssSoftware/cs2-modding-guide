# Memory and Performance

- Dispose `NativeArray`, `NativeList`, `BlobAssetReference`, and `NativeSlice` instances at the end of their lifetime.
- Avoid capturing `Entity` or `ComponentLookup` inside lambdas that run asynchronously; copy values locally before scheduling jobs.
- Batch structural changes by collecting entities in a `NativeList<Entity>` and processing them with a single `EntityCommandBuffer`.
- Annotate heavy code paths with `ProfilerMarker` scopes and monitor allocations in Unity Profiler.
- Use job-friendly patterns (`ScheduleParallel`) and mark inputs `WithReadOnly` / `WithDisposeOnCompletion` to keep Burst efficient.
