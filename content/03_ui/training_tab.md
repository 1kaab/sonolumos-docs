# Training Tab

## Purpose
Run training/optimization over a dataset using the current runtime configuration and scoring objective.

## Lifecycle
- Start pulls config from Settings -> Training.
- Stop sets a stop flag and ends the run cleanly.
- Runs are stored under `output/training_runs/<run_id>/`.

## Status panel
- Status + progress.
- Current/best score and seed.
- Current/best sample metadata (species/path).
- Progress uses a target evaluation count; with DEAP it is an upper bound, so `complete` with `processed < total` is normal.

## Interpreting results: volatility and span
- Volatility (score change rate): how much the score jumps between consecutive evaluations.
  - Low volatility = stable, repeatable scoring (good for refinement).
  - High volatility = sensitive/unstable scoring (often too many params or wide bounds).
- Span (score range): max score minus min score over a run or window.
  - Small span (e.g., 0.70–0.71) = tight convergence or under-exploration.
  - Large span (e.g., 0.40–0.80) = broad exploration or unstable objective.
- Practical read: tighten bounds and reduce trainable params when volatility is high; widen bounds or increase mutation when span is too tight.
- Use them as a search temperature:
  - High volatility + wide span → unstable: narrow bounds, reduce params, or increase batch size/train_frames.
  - Low volatility + tiny span → stuck: widen bounds, increase mutation sigma, or add params.
  - Plateau + low volatility → stop and lock params; move to the next genome family.
  - Compare runs by best score + lower volatility (robust winners).
- Logged in `summary.json` as `score_metrics` (count/min/max/span/mean/std/volatility).

## Results view
- Saved runs selector (run directory).
- Optional upload of results (CSV/JSON).
- Score timeline canvas (from results.csv).
- Top results thumbnails (downsampled path preview).
- Configurable best/mid/worst result lists (text summaries).
- Selected entry details.

## Row filter (expression)
- Python-like expression evaluated per row.
- Common fields: `species`, `label`, `path`, `source` (alias of `source_file`), `has_audio`.
- Helpers: `contains`, `startswith`, `endswith`, `regex`, `len`, `lower`, `upper`, `exists`, `abs`, `min`, `max`, `round`.

## Spectral filters / weights
- Spectral metrics (self-similarity + harmonicity) are planned for dataset filtering/weighting before motion training.
- These are audio-derived and do not change during training; they bias which samples are selected or weighted.
- See [[03_ui/spectral_tab]] for the metric mapping.

## Seed strategy (current + future)
- Current: seed only drives brownian noise; policies are fixed/random/sweep (not audio-derived).
- Seed as optimizable parameter: treat seed as a search variable (optimize seed with fixed params) or as part of the genome.
- Audio-derived seed: hash a stable audio fingerprint (e.g., centroid/flux/RMS stats) to get a deterministic per-sample seed.
- Variants: per-sample seed (identity), per-segment seed (time windows), seed offset per pass for controlled exploration.
- Reproducibility goal: seed all RNGs and log seed policy + derived seed in run metadata.

## Genome configuration (proposed: `deap.genomes`)
Expose genomes explicitly for transparency and future network visualization. Each genome owns its params, bounds, and seed policy.

### UI location
- Settings → Training → **DEAP** → **Genomes**
- Each genome can be enabled, assigned params, and given its own seed policy + aggregation.

### Conceptual schema (draft)
```yaml
deap:
  genomes:
    geometry:
      enabled: true
      params: [cap_alpha, cap_leak, ind_beta, ind_gamma, smoothing]
      bounds:
        cap_alpha: [0.0, 1.0]
        cap_leak: [0.0, 1.0]
      seed_policy:
        mode: fixed | sweep | random | audio
        fixed: 1
        sweep: {start: 0, end: 9, step: 1}
        audio_source: hash | feature
        jitter: 0.0
      aggregation: mean | median | worst | p90
    medium:
      enabled: true
      params: [viscosity, elasticity, field_strength, turbulence]
      seed_policy: {mode: fixed, fixed: 1}
      aggregation: mean
    hysteresis:
      enabled: true
      params: [memory, threshold, decay, loop_width, gain]
      seed_policy: {mode: fixed, fixed: 1}
      aggregation: mean
    final:
      enabled: true
      blend_g: 0.34
      blend_m: 0.33
      blend_h: 0.33
      # final is derived from G/M/H only (no independent params)
```

### UI exposure (minimal → advanced)
- Minimal: genome enabled, param list, seed policy (mode + fixed/sweep), aggregation.
- Advanced: per‑param bounds, audio‑derived seed source, jitter, phase weights, post‑processing knobs.
- Later (Network tab): render each genome as a node with its params and seed policy as sub‑nodes.

## Entropy visualization (ideas)
- Temporal entropy trace: compute entropy over sliding windows of the trajectory, then plot a line over time.
- Delay-embedding view (current replacement for entropy radius) to inspect temporal structure.
- Heatmap vs trace: heatmap reveals spatial occupancy, anisotropy, boundary clipping, and dead zones; trace only shows scalar change.
- Lightweight option: low-res occupancy grid updated every N frames, used both for entropy and a coarse visual.

## Structure objective (entropy + complexity)
- Uses a 1D series (`fov_final_norm`, 0..1) instead of XY.
- Metrics: mutual information `I(S_t; S_{t+tau})`, conditional entropy `H(S_{t+tau} | S_t)` as an entropy-rate proxy, and block entropies `H(L)` to estimate excess entropy.
- Score = `(w_excess * excess_norm + w_mi * mi_norm) * h_score`, where `h_score` rewards entropy-rate near `h_target` within `h_band`.
- Parameters live in Settings → Training → Structure objective: `bins`, `lag_s`, `lmax`, `h_target`, `h_band`, `w_excess`, `w_mi`.
- Practical bounds: keep `train_frames >= lag_s * fps + lmax`; use fewer bins for short series to avoid sparse counts.
- For alignment with the preview, keep Trajectory → FOV source set to `final` (the series used for scoring).

## Current implementation notes (as coded)
- Objectives implemented: `spiral`, `entropy`, `fractal`, `structure`. Other objective values (e.g. `self_enclosing`, `harmonic`) fall back to `spiral`.
- Structure objective implements MI + excess entropy heuristics; statistical complexity and effective complexity are not implemented.
- Entropy uses a 2D occupancy grid (24x24) over normalized XY; fractal blends entropy with box-count dimension (scales 4/8/16, weight 0.5).
- Scoring uses the current `traj_plot_mode` to resolve XY (prefers `dx/dy`, otherwise `az/alt`); optional resample to `train_frames`. `structure` bypasses XY and uses `fov_final_norm`.
- Spectral filter uses only `harmonic_coverage` from cache: rows below min are skipped, and score is scaled by `(1 + weight * coverage)` then clipped to 0..1.
- DEAP genome = `seed` + `train_params`; bounds come from UI slider min/max when available, otherwise defaults.
- Trajectory genome (brownian params) is defined in Settings but not used by the runner yet; intended for a later tuning phase.
- Genome seed start/end fields in Settings → Training → DEAP → Genomes are UI-only right now; the runner ignores them.

## Settings.training: choosing values (phase 1)
- Fix dataset root + filter first, then lock `train_frames` to a stable value (512-1024 is a good starting window).
- For structure: pick `lag_s` to match the time scale you want to detect (0.25-1.0s), keep `lmax` small (4-6) unless series are long.
- Use fewer `bins` for shorter series to avoid sparse counts; increase bins only after results stabilize.
- Start `h_target` near mid-range (0.4-0.6) and widen `h_band` if training collapses to noise or to a flat line.
- Bias `w_excess > w_mi` when you want long-range organization; increase `w_mi` to reward short-range predictability.
- Keep spectral filter off at first; re-enable it once motion scoring is stable and you want dataset biasing.

## Fundamental strategy for training parameters (expandable)
<details>
<summary>Stabilize the baseline</summary>
Lock dataset filter, objective, and seed policy first. Change one major knob at a time so you can attribute score changes.
</details>
<details>
<summary>Start with the smallest trainable set</summary>
Train only the minimum parameters that directly affect the objective (e.g., `cap_alpha`, `cap_leak`, `smoothing` for structure).
</details>
<details>
<summary>Use bounds to control exploration</summary>
Training uses UI min/max as bounds. Tight bounds for refinement, wider bounds for discovery.
</details>
<details>
<summary>Separate exploration from refinement</summary>
Early runs: wider bounds + higher mutation sigma. Later runs: narrower bounds + lower sigma.
</details>
<details>
<summary>Prefer sweep seeds for comparability</summary>
Sweep gives stable comparisons between runs; random is for exploration only.
</details>
<details>
<summary>Apply best, then narrow</summary>
Apply the best genome to draft, reduce trainable params, and re-run to lock gains before expanding again.
</details>
<details>
<summary>Align scoring with visualization</summary>
Make sure the series used for scoring is the one you are visually judging in the UI.
</details>

## Mutation (how it works right now)
- Mutation is a controlled random perturbation of the genome used to explore new parameter combinations.
- Each individual mutates with probability `mutpb`. When it mutates, each gene mutates with probability `mut_indpb`.
- Float genes use Gaussian noise scaled by `sigma * (max - min)` for that parameter, then clamped to bounds.
- Integer genes (seed) use Gaussian noise scaled by the integer span, then clamped to bounds.
- Higher `sigma` = larger jumps; lower `sigma` = fine-grained tuning.
- If `mutpb`/`mut_indpb` are too low, the population can stagnate; too high and the search becomes noisy.

## Training roadmap (map going forward)
- Objectives: spiral (current), entropy (current), fractal (entropy + box-count), structure (MI + excess), future self-similarity and harmonic objectives.
- Filters/weights: spectral metrics used to select or bias dataset rows before motion scoring.
- Model library: named parameter sets + seeds + routing metadata, stored as reusable “modes.”
- Live control: audio-derived seeds, serial feedback, and real-time evaluation loops.
- DEAP integration: genome = selected params + seed policy, with reproducible runs and dataset-aware validation.

## Deferred decisions
- Species -> parameter-range ground truth is not defined yet. Decide later whether ranges are manual, learned clusters, or a hybrid, and how they feed validation or selection.

## Future map: visual-first training + flexible methods
Goal: let training produce *visual previews* that can be human-ranked before full audio renders, while staying open to new objectives and search strategies.

### Visual-first pipeline (human-in-the-loop)
- Use short, multi-window evaluation for scoring (fast).
- Generate lightweight visual previews only for finalists (top/mid/worst).
- Use the same params + seeds for a final full-length render (truth), and export to Krita for unique artwork.
- Keep preview styles swappable (trajectory line, density map, macro overlay) so new methods can plug in without reworking the pipeline.

### DEAP methods to support (combinable)
- Multi-objective (NSGA-II): balance contrast vs coherence, motion entropy vs stability.
- Novelty search (or novelty-weighted fitness): reward unusual motion patterns to explore the space.
- Quality-diversity (MAP-Elites): build a grid of distinct results, not just one winner.
- Co-evolution: separate populations for source selection vs motion params.

### Flexibility rules
- Keep objectives modular: add new metrics without changing dataset loading or motion rendering.
- Keep evaluation windows configurable: fast proxy first, full render last.
- Keep outputs versioned: store genome + routing + seed policy so new methods can be replayed later.

## Two-phase pipeline: discovery → creation
Phase 1 (Sonolumos discovery / training):
- Build a lightweight lookup over raw WAVs (paths + features + optional time windows).
- Train on short windows to score motion quickly.
- Persist winners as *references* (no audio duplication).

Phase 2 (creation in Reaper + Krita):
- Reaper loads the winning segments and assembles the session.
- Sonolumos exports the final motion path + params for Krita rendering.
- Krita renders the final artwork from the exact params used in training.

## Drive ingestion: segmentation policy (locked v1)
Segmenting long/unsorted drives for training on a dedicated machine.
- segment_len_s: 22
- min_file_len_s: 22
- segments_per_8min: 1 (≈ 1 per 480s block)
- max_segments_per_file: 8
- grid_stride_s: 360 (6 min)
- jitter_s: ±12
- grid_end_margin_s: segment_len_s
- cache_location: local SSD
- deterministic offsets: seed = global_seed + file_hash + pass_id
- per-pass sampling: stable shuffle of grid offsets, pick N, then jitter and clamp
- coverage: new pass_id explores new grid points; overlap allowed unless min distance enforced

### Handoff metadata (minimum viable)
- `source_file` (absolute or dataset-relative path)
- `start_s`, `duration_s`
- `seed`, `seed_policy`
- `genome` (params + bounds)
- `routing` snapshot (fov mode, brown source, macro config)
- `motion_export` (csv path or reference to a saved trajectory)

## Outputs
- `summary.json`: run config, best score, top results.
- `results.csv`: per-row score with seed and metadata.

## Key files
- ui/tabs/training_tab.py
- engine/audio.py
- engine/controller.py
- ui/state.py
