# KLC Training Model (Crystal / Language / Cell)

## Purpose
Define the training model that optimizes GMH parameters and yields motion trajectories for output (API/serial/export).

## Concept mapping
- **KLC** is the trainer: Crystal / Language / Cell (optimizing layer).
- **GMH** is the model being optimized: Geometry / Medium / Hysteresis.
- **Trajectory** is the measurable output of GMH that KLC scores.
- **Output pipe** consumes the resulting motion (API, serial, CSV/SVG, etc).

## Scoring signals
- **Brown**: fractal/organic brownian shape score.
- **Entropy**: high entropic structure score (order without pure noise).
- **Error input (later phase)**: external feedback signal (human selection or biofeedback) that modulates scoring.

## Objective modes (current guidance)
- Keep brown and entropy objectives **separate** at first (like phase-separated GMH populations).
- Move to a **weighted composite** only after each objective is stable and interpretable.
- Composite goals should be explicit in config (weights + normalization), not implicit.

## Phase 1 training assumptions
- Start with **capacitor vs inductor routes** to maximize separation and observe stability.
- Use **structure** as the primary objective (GMH / FOV series).
- Keep **harmonic** as a filter/weight (not a full objective yet).
- Defer **brownian objective** until the cell-like metric is defined.
- Defer **mix_weight** blending until stable patterns are observed.
- No external error feedback in phase 1 (pure objective-driven training).

## Phase 2+ training assumptions
- Introduce **mixed** geometry (cap + ind) after phase 1 stabilizes.
- Introduce **human selection** as an error term.
- Add **live feedback error** (serial/biofeedback) after human selection stabilizes.
- Move to a **composite objective** once structure + brownian + harmonic are stable.

## CLI intent
- Provide a UI-agnostic training CLI that runs the same backend model.
- CLI should accept dataset root/filter, objective, seeds, and output paths.
- CLI should emit results compatible with the UI (results.csv / summary.json).

## CLI interface (draft)
Goal: a thin wrapper over the training backend (no UI dependencies).
Run via `python -m sonolumos_v3.training.cli ...` or `python scripts/klc.py ...`.

### Commands
- `klc inspect`: list dataset files, row counts, sample rows.
- `klc spectral-cache`: planned (compute harmonic_coverage cache for filtering/weighting).
- `klc train`: run training with a selected objective and genome params.

### Usage
```bash
python -m sonolumos_v3.training.cli --help
python -m sonolumos_v3.training.cli inspect --dataset-root ./datasets/ocean
python -m sonolumos_v3.training.cli train --dataset-root ./datasets/ocean \
  --dataset-filter whales --objective structure --geom-mode capacitor \
  --train-params cap_alpha,cap_leak,smoothing --passes 2 --max-iters 200
```

### Minimal flags (train)
- `--dataset-root PATH`
- `--dataset-filter STR`
- `--row-filter EXPR`
- `--objective spiral|entropy|fractal|structure`
- `--geom-mode capacitor|inductor|mixed|waveform`
- `--train-params cap_alpha,cap_leak,...`
- `--seed-policy fixed|random|sweep`
- `--seed INT` / `--seed-start INT --seed-end INT --seed-step INT`
- `--train-frames INT`
- `--passes INT`
- `--max-iters INT`
- `--time-budget-s FLOAT`
- `--backend deap|grid`

### Config file
- `--config PATH` (JSON/YAML) overrides flags; CLI flags override config.
- Store the resolved config in `summary.json` for reproducibility.

### Example
```bash
klc train \
  --dataset-root ./datasets/ocean \
  --dataset-filter whales \
  --objective structure \
  --geom-mode capacitor \
  --seed-policy sweep --seed-start 0 --seed-end 9 --seed-step 1 \
  --train-frames 512 --passes 2 --max-iters 200 \
  --train-params cap_alpha,cap_leak,smoothing
```

## Roadmap (current intent)
1) Objective + training network defined
2) First population (cap vs ind routes)
3) Error: human factor picking
4) Mixing
5) Error: human factor picking
6) Feedback error (serial/biofeedback)
7) Define results as boundaries/ranges
8) API/machine output
9) New sample input → mapping

## Key files
- training/runner.py
- engine/motion.py
- engine/routing.py
- ui/tabs/training_tab.py (UI only; backend should remain UI-agnostic)
