# Namespace Map

## Summary
This is the current signal/tap landscape and a proposed namespace scheme that keeps feature, model, modifier, output, and analysis signals separate. The goal is clarity, easy routing, and stable documentation as the network grows.


## Current keys (actual)

### Feature
- Audio feature dict keys from `engine/audio.py`:
  - `Spectral Centroid`
  - `Spectral Flux`
  - `RMS`
  - `Zero Crossing`
  - `Custom`
  - `Band RMS` (if configured)
- Exposed for export/debug in motion:
  - `spectral_flux`
- Energy packets (band RMS) live in `audio_features.extra` and are surfaced as feature names in routing.


### Geometry
- Routing taps (debug) currently stored in `taps`:
  - `fov.g`, `az.g`, `alt.g`
- UI overlay curves:
  - `fov_geom_norm` (normalized for plot)


### Medium
- Routing taps (debug):
  - `fov.m`, `az.m`, `alt.m`
- UI overlay curves:
  - `fov_medium_norm`


### Temporal / Hysteresis
- Routing taps (debug):
  - `fov.h`, `az.h`, `alt.h`
- UI overlay curves:
  - `fov_hyst_norm`
- Temporal post-process (alias/smooth):
  - `fov_final_norm`


### Modifier (Brownian)
- Brownian driver + debug:
  - `brown_source`
  - `brown_energy01`
  - `brown_volatility`
  - `brown_step`
- Override path:
  - `dx`, `dy`
  - `az_raw`, `alt_raw` (when brownian is enabled)


### Output
- Core motion outputs:
  - `fov` (degrees)
  - `az`, `alt`
- Raw outputs (before wrap/clamp):
  - `az_raw`, `alt_raw`


### Analysis (Spectral tab, current state keys)
- `spectral_matrix`
- `spectral_freqs`
- `spectral_times`
- `spectral_selfsim`
- Derived in Spectral UI (not yet namespaced):
  - harmonic frames
  - harmonic field
  - expression signal


### UI / View
- Trajectory modes and overlay state:
  - `traj_plot_mode`
  - `traj_fov_source`
  - `macro_visible`, `macro_visibility`
- Theme and layout settings:
  - `ui.theme`, `ui.theme_custom`, etc.


### Network / Transport
- Runtime state:
  - `net_connection` (host/port/protocol/mode, enabled)
  - `net_payload` (profile + series selection)
- Config path:
  - `cfg.network.connection.*`
  - `cfg.network.payload.*`


## Intended namespace scheme (proposal)

### feature.*
Raw analysis channels, aligned to analysis timeline.

- `feature.spectral.centroid`
- `feature.spectral.flux`
- `feature.rms`
- `feature.band.rms.<packet_name>`
- `feature.custom`


### geometry.<param>.*
Geometry shaping stage for each routed param.

- `geometry.fov.in`
- `geometry.fov.out`
- `geometry.fov.debug.curve256`
- `geometry.az.in`
- `geometry.az.out`
- `geometry.alt.in`
- `geometry.alt.out`


### medium.<param>.*

- `medium.fov`
- `medium.az`
- `medium.alt`


### temporal.<param>.*
Hysteresis + temporal post-process.

- `temporal.fov` (hysteresis output)
- `temporal.fov.alias`
- `temporal.fov.smooth`


### modifier.brown.*

- `modifier.brown.source`
- `modifier.brown.energy`
- `modifier.brown.volatility`
- `modifier.brown.step`
- `modifier.brown.mode`
- `modifier.brown.mix`
- `modifier.brown.dx`
- `modifier.brown.dy`


### output.*
Final motion signals used by UI and export.

- `output.fov`
- `output.az`
- `output.alt`
- `output.az_raw`
- `output.alt_raw`


### analysis.spectral.*
Spectral matrix + harmonic representations.

- `analysis.spectral.matrix`
- `analysis.spectral.freqs`
- `analysis.spectral.times`
- `analysis.spectral.self_similarity`
- `analysis.spectral.harmonics`
- `analysis.spectral.harmonic_field`
- `analysis.spectral.signal.<expr>`


### ui.*
View-only signals and derived overlays.

- `ui.traj.mode`
- `ui.traj.fov_source`
- `ui.macro.visible`
- `ui.theme`


### net.*
Network transport + external exchange.

- `net.connection.protocol`
- `net.connection.mode`
- `net.connection.host`
- `net.connection.port`
- `net.connection.bind_host`
- `net.connection.bind_port`
- `net.connection.osc_prefix`
- `net.payload.profile`
- `net.payload.series`
- `net.status.connected`
- `net.status.last_error`

## Migration checklist (short)
- Freeze the list of active taps used by UI + export (so we don't break consumers).
- Add a thin mapping layer (old key -> new namespace) for one release.
- Update routing + overlay reads to prefer namespaced keys.
- Update CSV export to include both keys during transition.
- Remove legacy keys only after the UI + training are verified against the new map.

## Node/namespace migration plan (safe path)
1) **Stabilize contracts**: define the canonical namespace keys per stage (`feature.*`, `geometry.*`, `medium.*`, `temporal.*`, `modifier.*`, `output.*`).
2) **Wrap legacy logic as nodes**: implement `GeometryNode`, `MediumNode`, `HysteresisNode` that call existing functions but emit namespaced taps.
3) **Dual outputs** (one release): write both legacy keys (`fov_geom_norm`, `fov.h`) and namespaced taps to avoid breaking UI/export.
4) **Routing swap**: in `engine/routing.py`, prefer node `.apply()` path for G/M/H; keep fallback to legacy until parity is verified.
5) **UI + export switch**: update readers to prefer namespaced keys; keep fallback to legacy keys.
6) **Remove legacy keys**: only after training + UI regression is clean and docs updated.

## DomainSpec integration plan (make it canonical)
- **Goal**: enforce min/max/clamp/periodic rules for every signal we persist or export.
- **Namespace‑keyed specs**: add entries for `geometry.*`, `medium.*`, `temporal.*`, `modifier.*`, and `output.*`.
- **Apply points**:
  - After each node (optional clamp to keep values sane).
  - After FOV finalization (`output.fov` in degrees).
  - After modifier mix (`output.az/alt`, `output.az_raw/alt_raw`).
- **Compatibility**: during migration, map legacy keys to their namespaced equivalents for domain enforcement.

## Central functions (summary)
- `engine/motion.build_motion`: end‑to‑end pipeline; builds features, routing (GMH), domains, brownian, FOV mapping, gains, and output mix.
- `engine/routing.generate_motion_from_features`: per‑route G/M/H processing + fov blending + taps.
- `engine/controller.build_motion_from_state`: bridges runtime state → routing config → motion build.
- `training/runner._run_deap_job` + `_eval`: dataset loop, genome evaluation, scoring, and results logging.
- `ui/canvas_viz.mount_motion_canvas` / `mount_fov_canvas`: installs JS renderers and updates for the canvases.
- `ui/graph_controls._on_commit`: applies XY pad / curve edits to params and triggers callbacks.


## Notes
- Current routing taps use short keys like `fov.g` and `fov.h`. The intended scheme above is a forward-compatible map so we can migrate without breaking usage.
- Spectral analysis is computed in the UI tab today; namespacing it under `analysis.spectral.*` lets it integrate with routing and training later.


## Key files
- `engine/audio.py`
- `engine/routing.py`
- `engine/motion.py`
- `engine/namespace.py`
- `ui/tabs/dynamics_tab.py`
- `ui/tabs/spectral_tab.py`
