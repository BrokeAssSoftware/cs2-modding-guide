---
FrontmatterVersion: 1
DocumentType: Guide
Title: CS2 Modding Recipes (How-To Index)
Summary: Index of source-verified, technique-by-technique how-to recipes distilled from real Cities Skylines II mods.
diataxis: how-to
source_version: "n/a - no pinned source"
last_reverified: "2026-07-04"
Created: 2026-07-01
Updated: 2026-07-01
Owners:
  - codex
References:
  - Label: Technique Index (coverage ledger)
    Path: ../../technique-index.md
  - Label: Recipe template
    Path: ../../.templates/recipe.md
  - Label: Agents (operating manual)
    Path: ../../AGENTS.md
---

# Recipes (How-To)

Each recipe is a task-focused, source-verified how-to for one CS2 modding **technique family**:
Problem -> Solution -> cited code (`repo/...#L` from a canonical mod at a pinned commit) ->
Pitfalls -> Variations -> See-also. Modes and coverage status live in the
[Technique Index](../../technique-index.md). New recipes use [`.templates/recipe.md`](../../.templates/recipe.md)
and follow the rules in [AGENTS.md](../../AGENTS.md).

## Available recipes

All 59 technique families (A-BG) are now source-verified. Family letters map to the coverage ledger in [`technique-index.md`](../../technique-index.md). Families A-AB are the original set; AC-BG landed in the 2026-07-05 deep re-mine (grouped at the end).

**Core ECS & systems**
- [Prefab-field override](prefab-field-override.md) - family A (TryGet -> component -> AddComponentData).
- [ECS system replacement via ordering](ecs-system-replacement-ordering.md) - family B (disable vanilla + UpdateBefore/After, no Harmony).
- [Disable/replace a vanilla system](disable-replace-vanilla-system.md) - family M (hard-disable + take over the phase).
- [ECS query rewriting](ecs-query-rewriting.md) - family T (narrow a vanilla query with a marker, no Harmony).
- [Burst IJobChunk / IJobEntity jobs](burst-ijobchunk.md) - family S (DOTS parallelism, ScheduleParallel).
- [Periodic GameSystemBase w/ UpdateInterval](periodic-updateinterval-system.md) - family L (slow-cadence systems).
- [Event-driven mod-state write](event-driven-modificationend.md) - family Y (ModificationEnd / ToolOutputBarrier pairing).
- [ECS ISerializable save data / SaveVersion](ecs-serializable-savedata.md) - family E (persist custom data across save/load).
- [External JSON side-car persistence](json-sidecar-persistence.md) - family F (ModsData JSON alongside the save).

**Prefab & economy data**
- [One-shot non-stacking prefab multiplier](one-shot-prefab-multiplier.md) - family J (PrefabBase baseline, apply once).
- [DynamicBuffer&lt;Resources&gt; economy nudging](dynamicbuffer-resources-nudge.md) - family K (adjust stored resources via EconomyUtils).

**Harmony**
- [Harmony postfix on economy/price getters](harmony-price-getter-postfix.md) - family C (manual Patch, ref __result).
- [Harmony redirector / reverse-patch bridge](harmony-redirector-reverse-patch.md) - family D (BelzontCommons submodule).
- [Harmony prefix on a private job-setup method](harmony-private-job-postfix.md) - family U (ref JobHandle __result, return false).
- [Reflect into private fields via Traverse](harmony-traverse-private-fields.md) - family X (Field&lt;T&gt;(name).Value read/write).
- [Weather/climate system override](weather-climate-override.md) - family AB (Harmony prefix on ClimateSystem.SampleClimate).

**UI & tooling**
- [SettingsUI patterns](settings-patterns.md) - family N (sliders, sections, hide-by-condition, buttons, presets, persistence).
- [Preset/country profile lookup arrays](preset-profile-arrays.md) - family Z (index-keyed backing arrays fanned into settings).
- [UISystemBase / React binding](uisystembase-react-binding.md) - family P (ValueBinding/TriggerBinding, the C# side).
- [COUI icon/host registration](coui-host-registration.md) - family H (AddHostLocation, coui:// hosts).
- [Render-pipeline overlay](render-pipeline-overlay.md) - family I (beginContextRendering + Graphics.DrawMesh).
- [Tool drag-select + Highlighted](tool-drag-select.md) - family V (ToolBaseSystem multi-select).
- [Custom ToolBaseSystem N-waypoint path](tool-nwaypoint-path.md) - family W (ordered waypoint selection).
- [NameSystem custom names](namesystem-custom-names.md) - family Q (SetCustomName via NameSystem).
- [Network composition bitmasks](network-composition-bitmasks.md) - family R (CompositionFlags upgrades).
- [Localization helper](localization-helper.md) - family O (IDictionarySource + multi-locale AddSource).

**Platform & lifecycle**
- [Reflection bridges between mods](reflection-mod-bridges.md) - family G (Type.GetType / assembly-scan, detect-and-degrade).
- [MainThreadDispatcher async](mainthread-dispatcher-async.md) - family AA (Task.Run + marshal back to the main thread).

**Skeletons**
- [System registration](system-registration.md) - schedule systems with UpdateAt/UpdateBefore/UpdateAfter across phases.
- [System template](system-template.md) - a GameSystemBase/SystemBase skeleton with cached queries and optional Burst jobs.

### Expansion families (AC-BG, 2026-07-05 deep re-mine)

**Simulation & ECS**
- [Prefab cloning / new-prefab synthesis](prefab-clone-synthesis.md) - AC (PrefabBase.Clone + AddPrefab + ObsoleteIdentifiers).
- [Runtime aggregate topology mutation](aggregate-topology-mutation.md) - AD (clone an aggregate entity, re-home edges).
- [Verify-and-reapply self-healing override](verify-and-reapply-override.md) - AE (durably override a recomputed ECS value).
- [ECS structural cascade deletion](ecs-cascade-deletion.md) - AF (AddComponent&lt;Deleted&gt; on entity + SubArea/SubNet/SubLane).
- [Reversible override (baseline capture/restore)](reversible-override-baseline.md) - AG (restore on toggle-off/OnDestroy).
- [Pathfind candidate-buffer rewrite](pathfind-candidate-rewrite.md) - AH (CompleteSetup prefix + FieldRefAccess).
- [Heuristic scoring & stochastic selection](heuristic-scoring-selection.md) - AI (gravity/softmax + component-strip veto).
- [Telemetry-fed feedback loop (asymmetric EWMA)](ewma-feedback-loop.md) - AJ.
- [Timed per-agent FSM persisted in ECS](ecs-agent-fsm.md) - AK.
- [Time/calendar tick-rate override](time-tick-rate-override.md) - AL (visual time dilation).
- [Notification-icon via IconCommandBuffer](icon-command-buffer.md) - AM.
- [Durable vanilla-network-component write](durable-network-component-write.md) - AN (survives uninstall).

**Harmony, interop & platform**
- [Harmony postfix on a vanilla System.OnUpdate](harmony-system-onupdate-postfix.md) - AO.
- [Cross-mod runtime service protocol](cross-mod-service-protocol.md) - AP (call/offer another mod's API).
- [Chirper feed injection](chirper-feed-injection.md) - AQ.
- [Platform achievement control + windowed enforcement](platform-achievement-control.md) - AR.
- [Debug-gated CSV telemetry + ECS-type introspection](debug-telemetry-introspection.md) - AS.
- StackFrame-scoped / caller-sniffing Harmony postfix - AT (a section in [harmony patching](../../explanation/harmony-patching.md)).

**UI & tooling**
- [Augment vanilla UI via moduleRegistry.extend](vanilla-ui-augmentation.md) - AU (toolbar/button/InfoSection/categories).
- [Simulation-speed control](simulation-speed-control.md) - AV (SimulationSystem.selectedSpeed).
- [Floating-panel framework](floating-panel-framework.md) - AW (draggable/resizable/collapsible).
- [Read-only status/observability binding](status-observability-binding.md) - AX (zero steady-state cost).
- [IMGUI debug/analytics overlay](imgui-debug-overlay.md) - AY.
- [Custom snap-mode framework + raycast engine](custom-snap-raycast-framework.md) - AZ.
- [Gate-only cross-entity coordination](gate-only-coordination.md) - BA.
- [Custom polygon-zone tool + save-stable conflict resolution](polygon-zone-tool.md) - BB.

**Content & platform**
- [Custom AssetDatabase registration + component-importer registry](custom-assetdatabase-registration.md) - BC.
- [Mod-extensible reflective formula surface](mod-extensible-formula-surface.md) - BD.
- [HDRP light-per-entity from ECS](hdrp-light-from-ecs.md) - BE.
- [Enumerate installed mods / active playset](modsdata-playset-autodiscovery.md) - BF.
- [Prefab-subnet traversal + host-&gt;upgrade propagation](prefab-subnet-traversal.md) - BG.

<!-- Keep this list in sync with technique-index.md. All A-BG families are source-verified as of 2026-07-05. -->

## Wanted recipes

See the [Technique Index "wanted" list](../../technique-index.md#wanted-recipes-thin--uncovered---steer-future-mod-picks-here) for uncovered families to prioritise (weighted by technique novelty and how commonly it is needed).
