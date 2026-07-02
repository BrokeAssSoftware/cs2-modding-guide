---
FrontmatterVersion: 1
DocumentType: Guide
Title: CS2 Save/Load Serialization - Modding Hook Map
Summary: How Cities Skylines II save/load and serialization work (OnGameLoaded, update phases, deterministic seeds) and how a mod persists a versioned parallel data payload - written for the vno-core persistence seam and vno-economy's finance ledger. Anchored to the live Systems wiki snapshot.
Created: 2026-06-20
Updated: 2026-06-20
Owners:
  - codex
Tags:
  - architecture
  - serialization
  - save-load
  - vno-core
References:
  - Label: Systems (wiki snapshot, refreshed 2026-06-20)
    Path: ../../vice-and-order-research/wiki/systems.md
  - Label: Systems and Components Catalog (wiki snapshot, refreshed 2026-06-20)
    Path: ../../vice-and-order-research/wiki/systems_and_components_catalog.md
  - Label: vno-core Module Guide
    Path: ../../vice-and-order/vno-core/README.md
---

# CS2 Save/Load Serialization - Modding Hook Map

## Purpose and scope

How CS2 save/load and serialization work, and how a mod persists a **versioned parallel data payload** that survives game patches. Written for the [vno-core](../../vice-and-order/vno-core/README.md) persistence seam (a near-term core TODO) and vno-economy's dirty/clean/escrow finance ledger, which must serialize deterministically and reload safely. API names are from the [Systems wiki snapshot](../../vice-and-order-research/wiki/systems.md) (refreshed 2026-06-20).

## The real serialization API

- **`OnGameLoaded(Context serializationContext)`** - override on a system to set up anything needed within a loaded save game. This is the load-time entry point for restoring a mod's persisted state.
- **Update phases** (`SystemUpdatePhase`):
  - `LoadSimulation` - runs specifically while a save game is loading; **executes 8 times in a row**.
  - `GameSimulation` - the normal loaded-game simulation phase.
  - `Serialize` - runs specifically while a save game or map is being saved.
- **Determinism:** `Game.Common.PseudoRandomSeed` - per-entity seeds stored in save games so reloading a save and running the simulation is reproducible (used e.g. by lights). Any vno-* randomness that must survive save/reload should follow this stored-seed pattern rather than ad-hoc RNG.

## Persisting a parallel ledger (vno-economy / vno-core)

vno-economy stores runtime data under `ModsData/ViceAndOrder/Finance` (ledger snapshots, analytics caches) and config under `ModsSettings/ViceAndOrder/Finance`. For state that must live **inside the save game** (so a city's illicit balances travel with the save), use the serialization phases above and:

- **Version the payload.** Write a schema/format version into the serialized blob so a future patch can migrate older saves. The vno-core save-payload-with-versioning task should own this contract.
- **Serialize in the `Serialize` phase, restore in `OnGameLoaded`.** Keep the ledger's serialized representation narrow and explicit (don't serialize derived/cacheable analytics - rebuild those after load).
- **Use `PseudoRandomSeed`-style stored seeds** for any deterministic finance/identity events so loaded saves replay identically.

## Save-compatibility context (recent patches)

- 1.5.8f1 - fixed a crash when loading save games (with mods) and long startup load times.
- 1.5.9f1 - custom water/terrain no longer break autosaving in the Editor.
- 1.5.10f1 - fixed old-save-game and custom-map saving issues.

These indicate save/load has been actively stabilized; a versioned, defensively-restored payload (tolerant of missing/extra fields) is the safe posture, especially across the Iceflake-era patch cadence.

## To confirm before coding

- The exact serialization read/write helper API surface (the `Context`/reader-writer methods used inside `OnGameLoaded` and the `Serialize` phase) - derive from a current decompile or a catalog refresh; the wiki snapshot documents the entry points and phases but not the full reader/writer API.

## Verification

- Re-verify `OnGameLoaded` signature, the `SystemUpdatePhase` values, and `PseudoRandomSeed` against [systems.md](../../vice-and-order-research/wiki/systems.md) before coding.
- Test save -> patch -> load with a versioned payload stub early in vno-core, given the run of save-compat fixes through 1.5.8-1.5.10.
