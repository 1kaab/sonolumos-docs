# Settings Tab

## Purpose
Configure analysis, motion, appearance, network packets, export, presets, performance, and training.

## Subtabs
- Audio
- Motion
- Appearance
- Network
- Export
- Presets
- Performance
- Training
- Debug

## Config model
- `cfg_draft`: editable settings.
- `cfg`: committed settings.
- Apply writes `cfg_draft` to disk and runs `hydrate_from_cfg()` to push into runtime state and theme.

## Key behaviors
- Appearance: preset themes plus custom palette for Background, Background 2, Header, Header 2, Menu, Menu 2, Text, Glass tint, alpha sliders, and plot accents.
- Network: energy packets are created here; Apply recomputes band RMS features and updates routing/brown sources.
- Network: connection controls for OSC/UDP/TCP (protocol, mode, host/port, bind host/port, OSC prefix, payload profile/series). Apply starts/stops `engine/net_runtime.py`.
- Preferences → Training sample: `train_frames` controls trajectory resampling for training objectives.
- Preferences → Audio: library volume gate (silence dB, min active ratio, peak min) for the sample indexer.
- Training config: dataset root/filter, row filter (expression), objective selection, spectral filter/weight, structure objective params (bins, lag, lmax, entropy target/band, MI/excess weights), seed policy/sweep, passes, max iters, time budget, and trainable params.
- Training → DEAP: population, crossover, mutation rates, per-gene mutation rate, mutation sigma, tournament size, and batch size (see below).
- Performance: live indicators for preview/refresh rates, payload sizes, timer status, and resample duplication.
- Export: CSV + Canvas + SVG subtabs, duration/export FPS, CSV extra columns, and SVG path export.
- Export → SVG: "Generate SVG" writes `motion.svg` in the repo root. "Generate SVG + data" writes `motion.svg`, `motion_meta.csv`, and `mapping_table.csv` under `output/svg_exports/<name>`.

## Key files
- ui/tabs/settings_tab.py
- ui/config.py
- ui/state.py
- ui/theme.py
- presets.py

## Training → DEAP controls
- Population: number of individuals per generation.
- Crossover probability (cxpb): chance a pair will exchange genes.
- Mutation probability (mutpb): chance an individual is mutated at all.
- Mutation per gene (mut_indpb): chance each gene mutates when mutation happens.
- Sigma: mutation scale as a fraction of the parameter range (larger = wilder jumps).
- Tournament: selection pressure; higher favors top scores more aggressively.
- Batch size: number of audio rows scored per individual (averaged); higher = more stable, slower.

## Training progress note (DEAP)
- The progress target is `population * (passes + 1)` (or `max_iters`), which assumes every individual is re-evaluated every generation.
- In practice, DEAP reuses fitness for unchanged individuals, so fewer evaluations can occur.
- Result: status can be `complete` with `processed < total`; this is expected unless a stop/error is shown.
