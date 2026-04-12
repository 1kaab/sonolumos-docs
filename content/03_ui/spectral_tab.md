# Spectral Tab

## Purpose
Provide a lightweight workspace for spectral pattern exploration and expression-based scoring.

## UI pieces
- Spectral matrix canvas with expression trace overlay.
- Spectral expression input with explicit apply.
- Harmonic overlay toggle and reference F0 input.

## Notes
- The tab is intentionally low-cost: no continuous redraws, compute happens only when analysis is added.
- Expressions will be used to define spectral filters or objectives.
- Harmonic frames, self-similarity, harmonic field, and 3D map are computed for spectral analysis and dataset filtering, not used as training objectives (for now).

## Expression signals (UI only)
- Signals available in expressions: `energy`, `flux`, `centroid`, `harmonic_energy`, `harmonic_coverage`, `inharmonic_energy`.
- Allowed functions: `abs`, `clip`, `max`, `min`, `log`, `log1p`, `sqrt`.
- Script prefixes: `trace`, `line`, `bars`, `x`, `y`, `z`, `pattern` (multiple expressions separated by `;` or newlines).

## Current metrics (as coded)
- `compute_spectral_metrics` currently returns **only** `harmonic_coverage`.
- Spectral matrix uses rFFT magnitude → log10 → normalize 0..1, downsampled to max 256 frames and 128 bins.
- Harmonic rows use up to 16 harmonics from `f0`; coverage is mean harmonic energy / total energy across frames.
- Self-similarity in the UI is cosine similarity between time frames (column-normalized dot product), then min-max normalized.
- Metrics cache lives at `output/training_runs/_spectral_cache.json` keyed by dataset root/filter + hop_ms + n_fft + window + f0.

## Training usage (filters / weights)
Spectral metrics are used to filter or weight dataset rows before motion training. They are not objectives yet.

## Training integration policy (deferred)
- Decide later whether spectral metrics are always-on filters, optional weights, or full objectives.
- Revisit after new motion objectives (mutual information / complexity) are defined so spectral can align with them.

Planned self-similarity metrics (not implemented in `compute_spectral_metrics` yet):
- Diagonal strength (mean near diagonal): weight (higher is better).
- Diagonal contrast (near vs off diagonal): filter (min threshold).
- Periodicity peak (off-diagonal bands): weight (if we want repeated motifs).
- Self-sim entropy (matrix entropy): filter (exclude extremes), optional weight toward mid-range.

Planned harmonic metrics (not implemented in `compute_spectral_metrics` yet):
- Harmonic energy ratio: weight (higher is better).
- Harmonic coverage: filter (min coverage).
- Harmonic stability (variance of harmonic energy): weight (lower variance is better).
- Inharmonic energy: filter (max cap) or negative weight.

Related:
- Motion entropy is a training objective (separate from spectral metrics). The trajectory view uses delay-embedding instead of entropy-radius.

## Key files
- ui/tabs/spectral_tab.py
