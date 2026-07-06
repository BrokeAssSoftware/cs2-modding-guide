---
FrontmatterVersion: 1
DocumentType: Guide
Title: CS2 Modding Technique Index
Summary: Coverage ledger of the modding technique families we understand, their canonical mods, applicability, and gaps ("wanted recipes"). Steers what to research next.
diataxis: reference
source_version: "n/a - no pinned source"
last_reverified: "2026-07-04"
Created: 2026-07-01
Updated: 2026-07-01
Owners:
  - codex
References:
  - Label: Agents (operating manual)
    Path: ./AGENTS.md
  - Label: Research dossiers
    Path: ../vice-and-order-research/mods/dossiers/
---

# CS2 Modding Technique Index (Coverage Ledger)

The families below were distilled from our source-verified mod dossiers. Each becomes a
`recipes/<slug>.md` how-to page as it is folded in. **Status** = handbook coverage of that
family: `source-verified` (recipe written + cited to a mod at a pinned commit), `queued`
(recipe planned), `wanted` (no mod teaches it yet). All 59 covered families below (A-BG) are
now `source-verified`; the uncovered "wanted" families are listed separately at the bottom.
Use this ledger to judge whether a candidate mod teaches something NEW before researching it.

## Covered families (from the 34-dossier corpus)

| # | Family | Recipe slug | Canonical mods (dossiers) | Applicability | Status |
|---|--------|-------------|---------------------------|---------------|--------|
| A | Prefab-field override (TryGet -> component -> AddComponentData) | `how-to/recipes/prefab-field-override.md` | outside-traffic-adjuster, magic-mail, smooth-left-hand-traffic, traffic-tool-essentials, better-bulldozer | core, economy, simulation | source-verified |
| B | ECS system replacement via ordering (disable vanilla + UpdateBefore/After) | `how-to/recipes/ecs-system-replacement-ordering.md` | traffic-tool-essentials, realistic-path-finding, advanced-road-naming | core, simulation | source-verified |
| C | Harmony postfix on economy/price getters | `how-to/recipes/harmony-price-getter-postfix.md` | market-based-economy, time2work-realistic-trips | economy | source-verified |
| D | Harmony redirector / reverse-patch bridge | `how-to/recipes/harmony-redirector-reverse-patch.md` | write-everywhere, anarchy | platform, ui | source-verified |
| E | ECS ISerializable save data / SaveVersion | `how-to/recipes/ecs-serializable-savedata.md` | advanced-road-naming, time2work-realistic-trips, traffic-tool-essentials, better-bulldozer | core | source-verified |
| F | External JSON side-car persistence | `how-to/recipes/json-sidecar-persistence.md` | specialized-industrial-zones, smooth-left-hand-traffic, road-speed-adjuster, time2work-realistic-trips, anarchy | core | source-verified |
| G | Reflection bridges between mods | `how-to/recipes/reflection-mod-bridges.md` | time2work-realistic-trips, realistic-path-finding, anarchy, elections-rt-module | platform, simulation | source-verified |
| H | COUI icon/host registration (AddHostLocation) | `how-to/recipes/coui-host-registration.md` | unified-icon-library, extra-lib, extra-assets-importer, write-everywhere | ui, media | source-verified |
| I | Render-pipeline / TextMeshPro overlay | `how-to/recipes/render-pipeline-overlay.md` | write-everywhere, advanced-road-naming, road-speed-adjuster | ui | source-verified |
| J | One-shot non-stacking prefab multiplier (PrefabBase baseline) | `how-to/recipes/one-shot-prefab-multiplier.md` | magic-mail, traffic-tool-essentials, better-bulldozer | economy, simulation | source-verified |
| K | DynamicBuffer<Resources> economy nudging | `how-to/recipes/dynamicbuffer-resources-nudge.md` | magic-mail, market-based-economy | economy | source-verified |
| L | Periodic GameSystemBase w/ UpdateInterval tuning | `how-to/recipes/periodic-updateinterval-system.md` | magic-mail, write-everywhere, anarchy | core, simulation | source-verified |
| M | Disable/replace vanilla system | `how-to/recipes/disable-replace-vanilla-system.md` | traffic-tool-essentials, realistic-path-finding | simulation | source-verified |
| N | SettingsUI slider/section/hide-by-condition | `how-to/recipes/settings-patterns.md` | time2work-realistic-trips, anarchy, road-speed-adjuster, magic-mail, better-bulldozer | core, ui | source-verified |
| O | Localization multi-locale registration | `how-to/recipes/localization-helper.md` | anarchy, i18n-everywhere, time2work-realistic-trips, magic-mail | media | source-verified |
| P | UISystemBase / React binding (ValueBinding/TriggerBinding) | `how-to/recipes/uisystembase-react-binding.md` | better-bulldozer, write-everywhere, traffic-tool-essentials, road-speed-adjuster | ui | source-verified |
| Q | NameSystem custom names | `how-to/recipes/namesystem-custom-names.md` | advanced-road-naming, custom-chirps | ui, media | source-verified |
| R | Network composition bitmasks | `how-to/recipes/network-composition-bitmasks.md` | anarchy | infrastructure | source-verified |
| S | Burst IJobChunk jobs (DOTS parallelism) | `how-to/recipes/burst-ijobchunk.md` | realistic-path-finding, time2work-realistic-trips, anarchy, traffic-tool-essentials, write-everywhere | core, simulation | source-verified |
| T | ECS query rewriting (Exclude/WithAll/WithNone) | `how-to/recipes/ecs-query-rewriting.md` | traffic-tool-essentials | core | source-verified |
| U | Harmony postfix on private job-mutation methods | `how-to/recipes/harmony-private-job-postfix.md` | time2work-realistic-trips | simulation | source-verified |
| V | Tool drag-select + Highlighted components | `how-to/recipes/tool-drag-select.md` | better-bulldozer, anarchy, advanced-road-naming, road-speed-adjuster | ui, infrastructure | source-verified |
| W | Custom ToolBaseSystem with N-waypoint path selection | `how-to/recipes/tool-nwaypoint-path.md` | advanced-road-naming | infrastructure | source-verified |
| X | Reflect into private fields via Harmony Traverse | `how-to/recipes/harmony-traverse-private-fields.md` | anarchy, time2work-realistic-trips, traffic-tool-essentials | platform | source-verified |
| Y | Event-driven mod-state write (ModificationEnd / ToolOutputBarrier) | `how-to/recipes/event-driven-modificationend.md` | better-bulldozer, anarchy | core | source-verified |
| Z | Preset/country profile lookup arrays | `how-to/recipes/preset-profile-arrays.md` | time2work-realistic-trips | economy, simulation | source-verified |
| AA | MainThreadDispatcher queue + coroutine re-marshalling | `how-to/recipes/mainthread-dispatcher-async.md` | extra-assets-importer, extra-lib | platform | source-verified |
| AB | Weather/climate system override | `how-to/recipes/weather-climate-override.md` | time2work-realistic-trips, time-weather-anarchy | simulation | source-verified |
| AC | Prefab cloning / new-prefab synthesis | `how-to/recipes/prefab-clone-synthesis.md` | specialized-industrial-zones, anarchy | content, core | source-verified |
| AD | Runtime entity/aggregate topology mutation | `how-to/recipes/aggregate-topology-mutation.md` | advanced-road-naming | infrastructure | source-verified |
| AE | Verify-and-reapply self-healing override | `how-to/recipes/verify-and-reapply-override.md` | advanced-road-naming | core, simulation | source-verified |
| AF | ECS structural cascade deletion | `how-to/recipes/ecs-cascade-deletion.md` | abandoned-building-remover | core, simulation | source-verified |
| AG | Reversible override (baseline capture/restore) | `how-to/recipes/reversible-override-baseline.md` | anarchy, time2work-realistic-trips, realistic-path-finding | core, simulation | source-verified |
| AH | Pathfind candidate-buffer rewrite (CompleteSetup prefix) | `how-to/recipes/pathfind-candidate-rewrite.md` | realistic-jobsearch | economy, simulation | source-verified |
| AI | Heuristic scoring & stochastic selection + component-strip veto | `how-to/recipes/heuristic-scoring-selection.md` | realistic-jobsearch, realistic-path-finding | simulation | source-verified |
| AJ | Telemetry-fed feedback loop (asymmetric EWMA) | `how-to/recipes/ewma-feedback-loop.md` | realistic-path-finding | simulation | source-verified |
| AK | Timed per-agent FSM persisted in ECS | `how-to/recipes/ecs-agent-fsm.md` | time2work-realistic-trips | simulation | source-verified |
| AL | Time/calendar tick-rate override | `how-to/recipes/time-tick-rate-override.md` | time2work-realistic-trips | simulation | source-verified |
| AM | Notification-icon via IconCommandBuffer | `how-to/recipes/icon-command-buffer.md` | magic-garbage-truck | simulation, ui | source-verified |
| AN | Durable vanilla-network-component write (survives uninstall) | `how-to/recipes/durable-network-component-write.md` | road-speed-adjuster | infrastructure, core | source-verified |
| AO | Harmony postfix on a vanilla System.OnUpdate | `how-to/recipes/harmony-system-onupdate-postfix.md` | market-based-economy, realistic-path-finding | economy, simulation | source-verified |
| AP | Cross-mod runtime service protocol | `how-to/recipes/cross-mod-service-protocol.md` | elections-rt-module, custom-chirps, anarchy | platform, simulation | source-verified |
| AQ | Chirper feed injection | `how-to/recipes/chirper-feed-injection.md` | custom-chirps | ui, media | source-verified |
| AR | Platform achievement control + windowed enforcement | `how-to/recipes/platform-achievement-control.md` | achievement-fixer | platform, core | source-verified |
| AS | Debug-gated CSV telemetry + reflection ECS-type introspection | `how-to/recipes/debug-telemetry-introspection.md` | realistic-jobsearch, vno-debug | operations | source-verified |
| AT | StackFrame-scoped / caller-sniffing Harmony postfix | `explanation/harmony-patching.md` (section) | extra-detailing-tools | platform | source-verified |
| AU | Augment vanilla UI via moduleRegistry.extend (toolbar/button/InfoSection/categories) | `how-to/recipes/vanilla-ui-augmentation.md` | advanced-simulation-speed, smooth-left-hand-traffic, extra-lib | ui | source-verified |
| AV | Simulation-speed control (SimulationSystem.selectedSpeed) | `how-to/recipes/simulation-speed-control.md` | advanced-simulation-speed | simulation, ui | source-verified |
| AW | Floating-panel framework (draggable/resizable/collapsible) | `how-to/recipes/floating-panel-framework.md` | extra-lib | ui | source-verified |
| AX | Read-only status/observability binding (zero steady-state) | `how-to/recipes/status-observability-binding.md` | magic-mail, magic-garbage-truck | ui, operations | source-verified |
| AY | IMGUI debug/analytics overlay + sample recorder | `how-to/recipes/imgui-debug-overlay.md` | market-based-economy | tooling | source-verified |
| AZ | Custom snap-mode framework + in-scene raycast engine | `how-to/recipes/custom-snap-raycast-framework.md` | extra-detailing-tools | tooling | source-verified |
| BA | Gate-only cross-entity coordination state machine | `how-to/recipes/gate-only-coordination.md` | traffic-tool-essentials | simulation, infrastructure | source-verified |
| BB | Custom polygon-zone tool + save-stable conflict resolution | `how-to/recipes/polygon-zone-tool.md` | traffic-tool-essentials | tooling, infrastructure | source-verified |
| BC | Custom AssetDatabase registration + component-importer registry | `how-to/recipes/custom-assetdatabase-registration.md` | extra-assets-importer | content | source-verified |
| BD | Mod-extensible reflective formula/expression surface | `how-to/recipes/mod-extensible-formula-surface.md` | write-everywhere | platform, ui | source-verified |
| BE | HDRP light-per-entity from ECS render data | `how-to/recipes/hdrp-light-from-ecs.md` | write-everywhere | ui | source-verified |
| BF | Enumerate installed mods / active playset | `how-to/recipes/modsdata-playset-autodiscovery.md` | i18n-everywhere | platform | source-verified |
| BG | Prefab-subnet traversal + host->upgrade propagation | `how-to/recipes/prefab-subnet-traversal.md` | smooth-left-hand-traffic | infrastructure | source-verified |

## Wanted recipes (thin / uncovered - steer future mod picks here)

**Newly covered by the 2026-07-05 deep re-mine** (moved out of "wanted"):
- Citizen AI behaviour FSM rewrite -> family **AK** (ecs-agent-fsm, time2work HospitalStay).
- Custom income / progressive tax hooks -> family **AO** (harmony-system-onupdate-postfix, MBE profit-tax + WorkProvider floor).
- Network topology mutation -> families **AD** (aggregate-topology-mutation) + **AC** (prefab-clone-synthesis).
- Asset bundle loading (external) -> family **BC** (custom-assetdatabase-registration).
- UI info-panel/tab extensions -> family **AU** (vanilla-ui-augmentation).
- Achievements/platform beyond flag-fixing -> family **AR** (platform-achievement-control).
- Building life-cycle / custom mixed-use zoning -> **partial** (AC prefab cloning + specialized-industrial-zones zoning; AF cascade deletion) - full age/upgrade/decay lifecycle still open.

Still genuinely uncovered - prioritise mods that would teach these, over parameter-tuner re-treads:
- Audio / SFX system integration.
- Custom standalone simulation systems (parallel economy / epidemic / crime sim) - **partial**: heuristic scoring (AI), EWMA feedback (AJ), and timed FSMs (AK) are the building blocks, but no single mod teaches a full from-scratch standalone sim.
- Economy resource production chains (new industries/resources) - **gated**: demand-modifier's live ECS resource control is storefront-only/unverified (blocked on a DLL decompile; see [research hygiene](explanation/research-hygiene.md)).
- Disaster / weather event spawning + economy chaining.

> Coverage/novelty note: the deep re-mine took the ledger from 28 to 59 source-verified families
> (A-BG). Future research should target the remaining "still uncovered" list above, weighted by
> technique novelty and how commonly it is needed, rather than best-month storefront rating.
