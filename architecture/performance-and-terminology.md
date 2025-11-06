# Performance Targets and Terminology

Keep these shared references in mind when designing new features or reviewing backlog items.

## Reference Hardware and Budgets
- **Baseline hardware**: Intel i7-11700K (or Ryzen 7 5800X equivalent), NVIDIA RTX 3070, 32 GB RAM, 1440p, Medium graphics preset.
- **Frame budgets**: simulation systems under 5 ms per frame, UI systems under 2 ms, background analytics under 1 ms.
- **Benchmark saves**: maintain deterministic saves for high-crime, budget-collapse, and vice escalation scenarios; run profiling on these saves before shipping features.

## Instrumentation
- Wrap heavy jobs in `ProfilerMarker` scopes and capture traces in profiling builds.
- Expose developer commands that dump component state or analytics so QA can validate metrics quickly.

## Canonical Terminology
- **Identity triangle**: `(loyalty, fear, opportunity)` ranges `0.0 - 1.0`, default `0.5`, stored under `vno_economy.identity`.
- **Ideology vector**: `(reformist, developer, lawAndOrder, unionist, populist, viceAligned)` sums to `1.0`.
- **Influence metric**: `current`, `decayRate`, `maxCapacity`, floats `0.0 - 100.0`.
- **Heat index**: `0.0 - 100.0`, escalation thresholds at `25`, `50`, `75`.
- **Legitimacy / Trust**: `0.0 - 100.0`, stored in `vno_order.legitimacy` and `vno_governance.trust`.

## Draft Event and API Contracts
Promote entries into `docs/API_REFERENCE.md` once the implementation stabilises.

| Contract | Summary | Status |
| --- | --- | --- |
| `HeatChangedEvent` | `{ factionId: Guid, previous: float, current: float, delta: float, timestamp: long }` | Draft |
| `GangEvent` | `{ type: enum, territoryId: Guid, actors: Guid[], loyaltyDelta: float, liquidityDelta: float }` | Draft |
| `IFinanceService` | `GetBalances(factionId)`, `InitiateLaunder(job)`, `SetAutoPolicy(policyId, enabled)` | Draft |
| `IVoteService` | `ScheduleSession(config)`, `GetForecast(sessionId)`, `SubmitInfluence(action)` | Draft |
| `IHealthService` | `GetStress(districtId)`, `RegisterProgram(programConfig)`, `ReportOutcome(outcome)` | Draft |

Reference this file when backlog stories mention performance budgets or shared vocabulary.
