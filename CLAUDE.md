# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Single-file Monte Carlo equity calculator for PLO (Pot Limit Omaha) poker variants (PLO4/PLO5/PLO6) with DBBP (Double Board Betting Poker) support. The current version is `calculator_v1_1_2_b_1_app_store_cc.html`. The previous version `calculator_v1_1_1_b_3_app_store_cc.html` is retained for reference.

## Running the App

No build system. Open the HTML file directly in any modern browser (Chrome, Firefox, Safari, iOS Safari). No dependencies, no npm, no compilation.

## Architecture

The file has three major embedded sections:

1. **CSS** (~lines 6–986): Dark theme with green/cyan gradients. Key classes: `.seat`, `.board`, `.cardchip`, `#resultsTable`.

2. **HTML** (~lines 1000–1373): Static structure — header, seat inputs (9), two boards (A and B, 5 cards each), results table, and modal overlays for Help/Buckets.

3. **JavaScript** (~lines 1375+): All logic, split into two logical contexts:
   - **Main thread**: DOM events, input validation, card parsing, UI state, IndexedDB scenarios, Web Worker lifecycle
   - **Web Worker**: Embedded as a string blob (built via `buildWorker()`), runs Monte Carlo simulation off the main thread

### Key Globals

- `CURRENT_PLAYERS` — number of active players
- `CARDS_PER_HAND` — 4, 5, or 6 based on PLO variant
- `RANGE_MODE` — boolean for omniscient vs. range-based simulation
- `simWorkers` — array of active Web Worker instances (one per CPU core)

### Simulation Flow

1. User fills seat/board inputs (exact cards or wildcards like `Ax`, `Xs`, `Xx`, or NOT constraints like `!AA`)
2. `runSim()` validates inputs, then spawns `Math.min(navigator.hardwareConcurrency || 4, 8)` workers
3. `buildWorker()` compiles the worker blob once; subsequent workers reuse the same `workerURL`
4. Each worker receives a slice of the total trial count and a unique RNG seed (`baseSeed + wi * 0x9E3779B9`)
5. Workers run Monte Carlo independently: deal deck → fill wildcards → evaluate best Omaha hand per player → accumulate equity stats
6. `progress` and `partial` messages from all workers are aggregated in the main thread via `mergeStats()` and displayed live
7. When all workers send `done`, `mergeLatest()` produces the final combined stats and `autosaveNow()` is called once
8. Cancel sends `{type:'cancel'}` to every worker; a 250 ms timeout force-terminates via `killWorker()`

### Hand Evaluation

Uses `bestOmaha5(hand, board)` in the Worker — evaluates all combinations of exactly 2 hole cards + 3 board cards (PLO rule) to find the best 5-card hand rank. Tracks hand categories: High Card, Pair, Two Pair, Trips/Set, Straight, Flush, Full House, Quads, Straight Flush.

### Data Persistence

Scenarios saved/loaded via IndexedDB (`saveScenario`, `loadScenario`, `deleteScenario`). Eight built-in template scenarios loaded via `applyTemplate()`.

### Card Input Formats

- Exact: `AS KD QC JH` or compact `ASKSQCJH`
- Wildcard suit: `Ax` (any ace), `Kx` (any king)
- Wildcard rank: `Xs` (any spade)
- Wildcard both: `Xx`
- NOT constraint: `!AA` (no paired aces), `!SS` (no double-suited to spades)

### iOS/Safari Notes

Special BFCache workaround code around lines 1699–1704 to prevent stale form state on back-navigation.

## Version Naming Convention

File versioning is encoded in the filename: `calculator_v{major}_{minor}_{patch}_{build}_{distribution}.html`. The `app_store_cc` suffix indicates App Store / Claude Code distribution variant.
