# Backend Separation Roadmap (Engine + Training)

## Goal
Keep UI thin and replaceable (NiceGUI → Qt or other), while backend stays stable and reusable.

## Target structure
- **Engine backend**: audio analysis, routing, motion, brownian, temporal.
- **Training backend**: orchestration, datasets, objectives, DEAP/optimization.
- **UI client**: renders views and calls backend APIs; no heavy logic.

## Current coupling (what makes UI sticky)
- UI owns the shared state dictionary (`ui/state.py`).
- Training orchestration lives in `ui/tabs/training_tab.py`.
- UI triggers engine work directly and mixes state + orchestration.

## Current status (in-progress decoupling)
- Training orchestration lives in `sonolumos_v3/training/runner.py` (no NiceGUI imports).
- `build_motion_from_state` requires an explicit state mapping (no UI fallback).
- Training uses **snapshots** of motion/audio config at run start, so UI tweaks during training do not alter results.
- UI can still **preview** a genome, but only via explicit user action (no automatic state mutation).

## Desired boundaries
### Backend model layer
- Central model/state object with typed fields and validation.
- UI reads/writes through an API (no direct dict mutation).

### Training backend
- Dataset loading, filtering, spectral cache, and DEAP in `sonolumos_v3/training/`.
- No UI or NiceGUI imports.
- Emits events or callbacks (progress, status, best score).

### Engine backend
- Stays as-is but is called through a service API.
- Training and UI both call the engine via the same interface.

## Incremental steps (low-risk)
1) **Extract training runner**:
   - Move `_run_training_job`, DEAP logic, and helpers into `sonolumos_v3/training/runner.py`.
   - UI calls `training.run(config, callbacks)` instead of owning the logic.
2) **Introduce a model adapter**:
   - Create `sonolumos_v3/model/state_store.py` with a typed state object and minimal getters/setters.
   - UI uses the adapter, engine reads from it.
3) **Add event hooks**:
   - Training emits progress via callbacks or a simple event bus.
   - UI listens and updates visuals; no training logic in UI.
4) **Gradually replace global state**:
   - Keep `ui/state.py` as a shim that forwards to the model layer.
   - Remove direct NiceGUI usage from backend modules.

## Operational detail (how it should work)
- Training snapshots `MotionState` + `AudioConfig` at run start; training never reads mutable UI state mid-run.
- Progress, current score, seeds, and results are written to `training_*` keys only.
- Preview is explicit: user clicks **preview genome** to apply params temporarily, with **restore** to roll back.
- Results are portable: training outputs include the config + genome used for each run.

## Why this matters
- UI swaps become feasible without rewriting training/engine.
- Long sessions are more stable (backend is not UI-bound).
- Easier to run headless training and reuse results in multiple frontends.

## Constraints to keep in mind
- Avoid UI-specific code in backend modules.
- Prefer small, explicit APIs over global dict mutation.
- Keep one source of truth for state (avoid dual state systems).
- Don’t introduce new training logic that mutates runtime UI params directly.

## Future options
- Qt UI for heavier graphics and local performance.
- Web UI for remote access and rapid iteration.
- Headless training service with result export.

## Guardrails for future development
- Backend modules must not import UI modules.
- Training must operate on snapshots and emit progress, not mutate UI state.
- UI changes must remain replaceable (NiceGUI should be a client, not a dependency).

## UI-agnostic engine checklist (what is still missing)
- **No UI state dependency**: engine should accept explicit config + inputs, never read `ui/state.py`.
- **Stable API surface**: `build_motion(config, inputs) -> motion`, `analyze_audio(audio, config) -> features`.
- **Backend-owned validation**: schema + validation live in backend, not UI.
- **Explicit side effects**: caching, I/O, and progress reporting behind backend services/callbacks.
- **Determinism controls**: seeds and sampling policies are explicit inputs, not implicit globals.

## Genome scope, seeds, and sub-genomes
- **Seed as a first-class gene**:
  - Can be fixed, swept, random, or derived from audio hashes/metadata.
  - Treat as a controlled knob for reproducibility and exploration.
- **Phase sub-genomes (G/M/H)**:
  - Separate genomes per phase, plus a small **blend/weights genome**.
  - Benefits: smaller search spaces, phase-specific objectives, clearer attribution.
  - Risks: phases drift apart or “fight” without coordination.
- **Network evolution options**:
  - **Cooperative co-evolution**: evolve G/M/H separately, evaluate in combinations.
  - **Alternating optimization**: freeze two phases while training the third.
  - **Meta-genome**: small controller genome that weights phase outputs based on features.
