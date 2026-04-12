# Krita Paint-Op (C++)

## Objective
Build a native Krita paint-op plugin that renders Slum exports through Krita's
brush engine (not QPainter). This is the bridge between precomposed motion data
and Krita's visual engine, with a stable data contract and no UI dependency.

## Target platforms
- Linux + macOS are both supported targets.
- Build artifacts are tied to a specific Krita version/ABI; we must target and
  document versions explicitly.
- macOS typically requires a Krita dev SDK or a full Krita source build
  environment. If the SDK is unknown, plan on building Krita from source.

## Data contract (input folder)
- `motion_meta.csv`: per-frame x/y + drivers.
- `mapping_table.csv`: mapping rules for brush properties.
- `render_config.json` (optional): preferences such as step size, layer prefix,
  output mode.

Derived drivers (from x/y):
- `heading_deg`
- `curvature`
- `speed` (optional)

Default mapping rows to start:
- size <- fov_final_norm
- opacity <- brown_volatility
- color_ramp <- spectral_*_norm (driver from render settings)
- sat_mult <- spectral_flux_norm (optional)
- val_mult <- spectral_centroid_norm (optional)
- rotation <- heading_deg
- scatter <- curvature
- layer_g/m/h <- fov_geom_norm / fov_medium_norm / fov_hyst_norm

## Rendering plan (current)
- Offline, time-driven, per-point raster render using the Molit paint-op (custom marker paintop; not preset fidelity).
- Time is the canonical clock; `frame` is a hint. Resample to internal `dt` + arc-length spacing, then map to output FPS.
- Mapping table drives per-point params (size/opacity/hue/rotation/scatter). Global controls are optional and default to no-op.
- Ordering matters for animation; stills can use order-neutral rendering when needed.
- Optional vector reference path export for inspection only.
- Molit preset is the baseline reference (not trained yet). Training support should be possible later by exposing a limited trainable subset.

## Rendering plan (next: native Krita brush engine)
Decision: move to Krita's actual brush system for maximum flexibility and
future-proof rendering. Keep the Slum data contract stable and swap renderers
underneath it.

Goals:
- Preserve the existing CSV+mapping flow so results remain cohesive across
  renderers.
- Use Krita paint-op/preset features (tip shape, texture, spacing, scatter,
  anisotropy, rotation, opacity, HSV, grain) instead of marker circles.
- Enable richer effects (spray, filament/threads, fractal textures, waves,
  color meshes) without changing upstream data semantics.

Target effects -> brush engine capabilities:
- Spray: scatter + dab count + spacing jitter (drive from volatility/flux).
- Threads/filaments: anisotropic tip + rotation aligned to heading + spacing
  modulation.
- Fractal patterns: textured tip masks + multi-scale dab sizes.
- Waves/color meshes: hue/val modulation + multi-layer offsets + phase-shifted
  normal.

Implementation strategy:
- Keep Molit marker renderer as a baseline/fallback.
- Add a brush-engine renderer path that uses `KisPainter` + preset brush to
  render a stroke from the same per-dab parameters.
- Extend the mapping table with new properties (tip scale, texture strength,
  emission count/spread, anisotropy) while keeping backwards compatibility.

## Krita engine synchronization steps (offline first, streaming later)
1) Lock the data contract:
   - Canonical timebase: `frame` + `fps` (t = frame / fps).
   - Required channels: `x`, `y`, plus drivers for brush params.
   - CSV + streaming share identical field names and ranges.
2) Brush-engine capability map (Molit preset):
   - Enumerate which paintop settings can change per dab / per frame.
   - Split into sensor-driven vs parameter-driven sets.
3) Animation frame control in Krita:
   - Ensure doc is animated; target layer exists; timeline row available.
   - Guarantee a keyframe per frame (insert blank if missing).
4) Stroke construction strategy:
   - Convert frame points to stroke segments + dabs.
   - Pick spacing rules (distance-based vs time-based) and stick to them.
   - Decide per-dab vs per-frame param updates.
5) Mapping layer (patch bay):
   - Apply Slum drivers via curves/gates/blend to brush params.
   - Keep mapping logic identical for offline and streaming modes.
6) Offline render pipeline:
   - Batch frames, suppress extra UI updates, render to a dedicated layer.
7) Performance + stability:
   - Cache derived values, avoid redundant recompute.
   - Add lightweight debug summaries (points per frame, bounds, keyframe state).
8) Streaming mode:
   - Live input via OSC/UDP/HTTP; add a frame-latch so each frame renders once.
   - Optional live overrides on top of CSV playback.
9) Validation:
   - Golden CSV -> deterministic output comparison.
   - Check frame counts, keyframes, and per-frame visible deltas.

## UI stability: large WAV uploads + Motion tab resets
Observed issue: loading large WAVs (e.g., ~500MB) can cause the UI to jump
back to the start page or exit the Motion tab. Recently, this is happening
more frequently, even on smaller files that used to be stable.

Likely root cause: backend process restart due to memory spikes or unhandled
exceptions during upload/analysis. When the server restarts, the browser
reconnects and resets to the default page.

Primary risk path (upload pipeline):
- Upload reads entire file into RAM (`raw = await e.file.read()`).
- `sf.read(io.BytesIO(raw))` creates another full buffer.
- Optional resample uses `linspace` + `interp` (full-sized copies).
- Signal is stored in state + a separate analysis copy.
- Feature extraction runs on the full signal (`analyze_audio`).

Secondary contributors:
- Waveform plotting downsamples the full signal every refresh.
- Motion refresh cycles can re-trigger heavy work after large loads.
- Less headroom now due to extra UI state (training history, render preview).

Quick triage (no code):
- Check terminal logs for crashes/exceptions around the reload.
- Monitor memory during upload (Activity Monitor).
- Keep `audio_dtype` at `float32`.
- Lower `audio_target_sr` when resampling.
- Increase `hop_ms` to reduce feature frames.

Action plan (later):
- Stream/segment decode instead of loading entire WAV into RAM.
- Store audio on disk + mmap or windowed analysis.
- Add lightweight telemetry: memory + timing around upload/analyze.

## Known issue: Trajectory mode toggles not responding (xy/polar/sphere)
Reported behavior: the first three trajectory modes appear unresponsive or
visually identical after recent UI changes (pads removal).

Hypotheses to investigate:
- `traj_plot_mode` changes but canvas does not refresh (state reset or
  refresh throttling).
- render preview state (`traj_render_preview`) overrides visual output.
- motion arrays are too flat (dx/dy and az/alt nearly identical).

Next steps:
- Log mode changes + confirm `update_motion_canvas` is called on click.
- Compare `dx/dy` vs `az/alt` series for the current motion.
- Verify `traj_plot_mode` survives tab refresh/mount.

## Harmonic proportion rules for richer brush animation
Goal: introduce harmonic, "sacred" proportions as *structuring rules* rather
than hard overrides. The motion data still drives the visual; we simply
quantize or modulate spacing, rotation, and emission with stable ratios.

### Ratio palettes (the harmonic vocabulary)
Pick a small, stable palette and keep it consistent across renders:
- Musical: 1/1, 2/1, 3/2, 4/3, 5/4, 5/3, 8/5
- Geometric: phi=1.618, sqrt2=1.414, sqrt3=1.732, pi/2=1.571
- Golden angle: 137.507 degrees (for even distribution)

Rule: map normalized driver values onto the ratio palette by index or nearest
distance. This yields structured variation without flattening the motion.

### (1) Ratio-quantized spacing
Base spacing comes from size or distance, then snap to a ratio:
- `base = size * spacing_scale`
- `ratio = nearest(ratio_palette, driver_norm)`
- `spacing = base * ratio`

This keeps spacing coherent while still reflecting the underlying driver.
It is the safest harmonic rule to introduce first because it does not change
color or stroke topology.

Where to apply:
- CSV path building: decimate points by spacing rule before stroke.
- Brush engine: per-dab spacing (preferred for long-term).

### (2) Golden-angle rotation / scatter
Use the golden angle to avoid clustering when distributing dabs:
- `rotation = heading_deg + (index * 137.507) * rotation_strength`
- Optional: add a small driver-based jitter to vary the rotation speed.

This yields an even, spiral-like distribution without random clumps. It is
especially effective when the brush tip is anisotropic or textured.

Where to apply:
- In the paintop: per-dab rotation or scatter offset.
- In Python: inject per-segment rotation values in the mapping (approx).

### (3) Euclidean gating (harmonic rhythm)
Gate which dabs emit based on a Euclidean rhythm:
- For N dabs, allow K emits (K from driver or ratio index).
- Distribute K across N evenly.

This creates rhythmic breathing and "pulse" without needing a separate audio
envelope. It also reduces overdraw in dense regions.

Where to apply:
- Pre-stroke: remove or skip dabs based on gate rule.
- Paintop: per-dab gate test (best long-term).

### How these map into Slum/Krita
We keep the same CSV drivers and add **policy** in render_config or paintop:
- `spacing_policy`: `linear | ratio_quantized`
- `ratio_palette`: `musical | geometric | custom`
- `rotation_policy`: `heading | golden_angle`
- `gate_policy`: `none | euclidean`
- `gate_density`: 0..1 or integer K

This keeps the data contract stable and makes the harmonic layer optional.

### Interaction with mapping table
The mapping table still maps drivers to brush params, but the harmonic layer
can quantize or gate *after* mapping:
- First: driver -> normalized value.
- Then: apply mapping table (curve/gamma/range).
- Finally: apply harmonic policy (spacing snap / golden angle / gate).

This preserves the fidelity of the driver signal while adding structure.

### Implementation sequence (transition to brush engine)
1) Ratio-quantized spacing in the Python path builder (fast, visible change).
2) Golden-angle rotation in Molit or a brush-engine path (anisotropic tips).
3) Euclidean gating to control density and rhythm (reduces overdraw).

These three are the first-class rules to carry into the native brush-engine
renderer so the visual language stays consistent across renderers.

### render_config.json example (optional)
Place next to `motion_meta.csv`:
```
{
  "harmonics": {
    "enabled": true,
    "spacing": {
      "enabled": true,
      "base_px": 2.0,
      "min_px": 0.5,
      "max_px": 12.0,
      "palette": "musical",
      "driver": "speed",
      "mode": "index"
    },
    "rotation": {
      "enabled": true,
      "radius_px": 2.0,
      "driver": "curvature",
      "scope": "frame"
    },
    "gating": {
      "enabled": true,
      "pattern_len": 8,
      "density": 0.6,
      "scope": "frame",
      "driver": "speed"
    }
  }
}
```

## Krita API investigation (current)
Python plugin path (motion_csv_trace):
- Uses `QPainter`/`QPen` on a `QImage` for per-frame raster in `_render_per_frame`; this bypasses brush engine dynamics. Float args must use `QPointF` overloads (see `development/krita-dev/krita/plugins/python/motion_csv_trace/motion_csv_trace.py`).
- Builds an SVG path in a vector layer and triggers the `stroke_shapes` action to stroke with the current preset (brush engine path) in `development/krita-dev/krita/plugins/python/motion_csv_trace/motion_csv_trace.py`.
- Brush animation path auto-sets `MOLIT_MOTION_META` + `MOLIT_MAPPING` from the selected CSV (mapping inferred from `mapping_table.csv` next to it) and attempts to auto-activate a Molit preset (name contains `molit`); it forces a Qt event flush so shape selection is live before `stroke_shapes`.
- Raster animation output is restricted to RGBA/U8 documents by the script guard.

C++ brush engine path:
- `KisPainter` is the preset-faithful entry point: `setPaintOpPreset()` + `paintLine()`/`paintAt()` in `development/krita-dev/krita/libs/image/kis_painter.h`.
- Preset loading examples: `development/krita-dev/krita/benchmarks/kis_stroke_benchmark.cpp` and `development/krita-dev/krita/sdk/tests/stroke_testing_utils.cpp`.
- Paintop plugins (custom engines) live under `development/krita-dev/krita/plugins/paintops`.

Animation (timeline) API:
- Raster animation uses `KisRasterKeyframeChannel` + `addKeyframe(frame)` in `development/krita-dev/krita/plugins/impex/csv/csv_loader.cpp` and `development/krita-dev/krita/libs/image/kis_raster_keyframe_channel.h`.
- Clip/playback range lives on `KisImage` (`development/krita-dev/krita/libs/image/kis_image.h`), exposed in Python as `setFullClipRangeStartTime`, `setFullClipRangeEndTime`, `setPlayBackRange`.

Molit paintop (custom brush):
- Implementation: `KisMolitOp` in `development/krita-dev/krita/plugins/paintops/molit/kis_molitop.cpp`, draws circular dabs via `KisMarkerPainter`.
- Options/config: diameter, spacing, auto-spacing, motion/meta paths in `development/krita-dev/krita/plugins/paintops/molit/KisMolitOpOptionData.h`.
- Motion/meta inputs are loaded via env/config: `MOLIT_MOTION_META`, `MOLIT_MAPPING`, `MOLIT_SIZE_COLUMN`, `MOLIT_OPACITY_COLUMN`, `MOLIT_LOOP`, `MOLIT_RAW`, `MOLIT_SIZE_SCALE`, `MOLIT_OPACITY_SCALE`, `MOLIT_AUTO_RELOAD`, `MOLIT_RELOAD_MS`, `MOLIT_REPO_ROOT`.
- Mapping supports range format headers (`property,driver,min,max,curve,gamma,gate_min,gate_max`) or scale/offset; derived columns `heading_deg` + `curvature` are computed when `x/y` are present.
- Motion sample index advances per dab; it is not frame/time-driven unless we modify it.

## Mapping table deep dive (Molit)
The Molit mapping table is a CSV that turns motion drivers into per-dab brush
parameters. It is read by `KisMolitOp` and applied on every dab in
`development/krita-dev/krita/plugins/paintops/molit/kis_molitop.cpp`.

### Format: range mapping (what we use now)
Header (range format):
`property,driver,min,max,curve,gamma,gate_min,gate_max,notes`

How a row is evaluated:
1) Read driver value from `motion_meta.csv`.
2) Normalize into `t` using gates:
   - If `gate_max != gate_min`: `t = (value - gate_min) / (gate_max - gate_min)`
   - Else: `t = value`
3) Clamp:
   - If `MOLIT_RAW=0` (default): clamp `t` to `[0, 1]`
   - If `MOLIT_RAW=1`: clamp only to `t >= 0`
4) Apply gamma: `t = t ^ gamma`
5) Apply curve: `linear`, `smoothstep`, `ease_in`, `ease_out` (others ignored)
6) Map to the property range:
   - `mapped = min + t * (max - min)`
7) Property-specific conversion:
   - `hue`: if `mapped` is in degrees (> 1.0), convert to 0..1 by `/ 360`
   - `rotation`: if `mapped` is in degrees (> 2*pi), convert to radians

Notes:
- For `size`, range format sets `sizeIsAbsolute = true` so the mapped value is
  interpreted as a direct diameter (still multiplied by the brush size option
  and LOD scale).
- If the mapping file is missing, Molit falls back to default mappings using
  `MOLIT_SIZE_COLUMN`/`MOLIT_OPACITY_COLUMN` and their scales.

### Format: scale/offset mapping (legacy)
Header (scale/offset format):
`property,source,scale,offset,clamp_min,clamp_max,default`

This is a simple affine mapping:
`mapped = value * scale + offset`, then clamped. We are not using this format
in `mapping_table.csv`.

### What each property means
All of these are applied per dab:
- `size`: dab diameter. With range format this is absolute (px at scale 1),
  then multiplied by the brush size option and LOD scale. Example in current
  mapping: `fov_final_norm` mapped to `2..80`.
- `opacity`: multiplies the current brush opacity. Final opacity is
  `brushOpacity * motionOpacity`. Current mapping uses `brown_volatility`
  mapped to `0.1..1.0`, so it can be subtle if the driver range is tight.
- `color_ramp` (derived ramp): takes a 0..1 driver and maps it through Turbo
  colormap in `turboColor()`; this replaces the brush base hue but keeps the
  brush alpha. In our data this is usually `spectral_combo_norm`.
- `hue`: sets the HSV hue directly (0..1). If both `color_ramp` and `hue` are
  present, `color_ramp` wins.
- `sat_mult`: multiplies HSV saturation of the current color, clamped to 0..1.
  Example `0.6..1.2` boosts or attenuates saturation.
- `val_mult`: multiplies HSV value (brightness), clamped to 0..1. Example
  `0.7..1.15` dims or brightens.
- `rotation`: adds to the current dab rotation (pressure/tilt rotation + this
  rotation). If you pass degrees, it is converted to radians.
- `scatter`: randomly offsets the dab within `scatter * radius`. The random
  seed is stable per sample index, so it is deterministic for a given CSV.

### Derived drivers in motion_meta.csv
These are computed from `x/y` by Molit before mapping:
- `heading_deg`: direction of travel, 0..360 degrees; if no movement, the
  previous heading is reused.
- `curvature`: absolute heading change between samples (in degrees).

### Current kritaone mapping (what it does)
`sonolumos_v3/output/svg_exports/kritaone/mapping_table.csv`:
- `size <- fov_final_norm` (2..80): absolute diameter range.
- `opacity <- brown_volatility` (0.1..1.0): per-dab opacity.
- `color_ramp <- spectral_combo_norm` (0..1): Turbo color ramp.
- `sat_mult <- spectral_flux_norm` (0.6..1.2): saturation multiplier.
- `val_mult <- spectral_centroid_norm` (0.7..1.15): value multiplier.
- `rotation <- heading_deg` (0..360): dab rotation.
- `scatter <- curvature` (0..1): scatter radius in dab units.

If dynamics feel flat, tighten `gate_min/gate_max` or widen the `min/max` range
to amplify driver contrast (example: gate 0.6..0.8 to exaggerate variation).

## Data contract notes
- Keep per-property mapping in `mapping_table.csv`.
- Keep render policy (fps, dt, spacing, ordering, seeds, presets) in `render_config.json` to keep training clean.
- Color ramp driver is selected in SLUM render settings and written into the mapping table.
- SLUM edits mapping; Krita is the renderer.

## Timing strategy (animation vs streaming)
- Offline animation: treat `frame` as canonical if export FPS is fixed in SLUM; it maps 1:1 to Krita keyframes and avoids float drift.
- Time metadata: keep `t` in CSV as `frame / fps`; use it for retiming, resampling, or validation.
- If time is the source: derive frame index as `round(t * fps)`; this is how the current QPainter path resolves frames in `development/krita-dev/krita/plugins/python/motion_csv_trace/motion_csv_trace.py`.
- Streaming: treat `t` as the canonical clock and derive `frame = round((t - t0) * fps)` only when writing to a timeline. For live rendering, skip discrete frames and paint directly.
- Streaming FPS: use `preview_fps` or add a `stream_fps` setting; bucket samples into `[n/fps, (n+1)/fps)` windows and resample if it differs from export FPS.

```mermaid
flowchart TD
  A[CSV rows: frame,t,x,y] --> B{Render mode}
  B -->|Offline animation| C[Use frame as canonical index]
  C --> D[Write keyframes per frame]
  B -->|Streaming| E[Use t as canonical clock]
  E --> F[Bucket samples into frame windows]
  F --> D
  E --> G[Optional live paint (no timeline)]
```

## Terminology
- Point: one CSV sample (x, y, frame, t, plus drivers).
- Dab: a single brush stamp placed by the paint engine at spacing intervals along a stroke. A segment between two points can generate multiple dabs depending on spacing.

## MIDI controller integration (MPC Live 2 / macOS Sonoma)
Goal: allow hardware controls to drive Slum params live, without disrupting the
offline CSV workflow.

Assumptions:
- MPC Live 2 in Controller Mode presents as a class-compliant USB MIDI device.
- We only need MIDI In (CC + Note) for now.

Pipeline:
1) Device discovery: list inputs via `mido`/`python-rtmidi`, expose a selector
   in Settings -> Signal Flow -> MIDI.
2) Background listener thread: open the MIDI input and parse CC/Note messages,
   push updates into a thread-safe queue.
3) UI-safe apply: a main-thread timer drains the queue, writes into `state`,
   and calls `bump_motion()` + `refresh_plot()` with throttling.
4) Mapping: per-control mapping table (cc -> param, scale, offset, smoothing,
   clamp, latch/toggle).
5) Persistence: store the mapping in config and reload on boot.

Implementation notes:
- Handle disconnects and auto-reconnect (MPC power cycle or cable replug).
- For pads/notes, support both momentary triggers and latching toggles.
- Add per-control smoothing (EMA) to avoid jitter on continuous knobs.

## Live audio input (streaming)
Goal: process audio in real time and update motion drivers with low latency,
without blocking UI or losing stability.

Pipeline:
1) Capture: use `sounddevice` (or PyAudio) input stream, float32, block size
   tied to `hop_ms`.
2) Ring buffer: append incoming frames into a deque/ring; avoid full-file
   buffering.
3) Analysis worker: read windowed chunks, run STFT/features, write driver
   updates into shared state.
4) Throttle: update UI/motion at `preview_fps`; decouple from render FPS.
5) Modes: monitor (no disk), record (save WAV), and hybrid (live + cache).

Notes:
- Keep feature extraction on a background thread or process to avoid UI stalls.
- Live input uses time as the canonical clock; frame indices are only derived
  when writing to a timeline or export.
- Expose a "live latency" setting: hop size, window size, smoothing.

## Current slum output: kritaone
- Files in `sonolumos_v3/output/svg_exports/kritaone`: `motion_meta.csv`, `mapping_table.csv`, `mapping_table_layers.csv`, `motion.svg`.
- `motion_meta.csv` header: `frame,t,x,y,fov_final_norm,fov_geom_norm,fov_medium_norm,fov_hyst_norm,fov,brown_volatility,brown_step,spectral_flux,spectral_centroid,spectral_centroid_norm,spectral_flux_norm,spectral_combo_norm`.
- `mapping_table.csv` maps size/opacity/color_ramp/sat_mult/val_mult/rotation/scatter.
- `mapping_table_layers.csv` adds `layer_g`, `layer_m`, `layer_h` weights.
- `motion.svg` is a single polyline path of the motion trajectory for visual reference or vector stroke.

## Render config schema (draft)
Minimal JSON shape for render policy (separate from mapping):

```json
{
  "time": {
    "time_mode": "clock",
    "fps": 24,
    "dt": 0.0083,
    "frame_source": "time",
    "resample_arc_px": 1.5
  },
  "order": {
    "animation": "time",
    "still": "neutral"
  },
  "render": {
    "mode": "both",
    "growth": "progressive",
    "vector_reference": true,
    "seed": 0
  },
  "brush": {
    "preset": "molit_reference"
  },
  "units": {
    "coords": "px"
  }
}
```

Field notes:
- `time.time_mode`: `clock` uses CSV `time`; `frame` uses CSV `frame` if present.
- `time.fps`: output FPS for animation rendering.
- `time.dt`: internal resample step in seconds.
- `time.frame_source`: `time` (derive from `time` + fps) or `frame` (trust CSV frame).
- `time.resample_arc_px`: max arc-length gap between points; smaller = more detail.
- `order.animation`: `time` for faithful motion; `order.still`: `neutral` for order-independent stills.
- `render.mode`: `still` | `animation` | `both`.
- `render.growth`: `progressive` (draw-on) or `per_frame` (chunked).
- `render.vector_reference`: export a vector reference path for inspection only.
- `render.seed`: deterministic scatter/brush randomness.
- `brush.preset`: base Molit preset name (baseline reference).

## Scope and phases
Phase 0: Lock the data contract
- Finalize the CSV columns and mapping table semantics.
- Define default driver set and naming.

Phase 1: Plugin skeleton
- CMake project, metadata, and a minimal paint-op stub that loads in Krita.
- Document build steps for Linux/macOS.

Phase 2: CSV reader + mapping
- Parse `motion_meta.csv` + `mapping_table.csv`.
- Render a still image into a single layer using Krita brush engine.

Phase 3: Multi-layer output
- Support G/M/H layers with per-layer weights.
- Most importantly: gmh weighted layer as new dimension.
- the chosen mode 

Phase 4: Animation mode
- Render frames to a timeline or image sequence.

Phase 5: Performance tuning
- Decimation, spacing control, and max dab rate.
- Cache derived drivers and clamp numeric ranges.

Phase 6: Live signals (future)
- Ring buffer input.
- Transport layer (OSC/UDP/TCP) and throttling.
- Live mode remains additive; CSV workflow stays primary.

## Repository layout
Recommended: keep the plugin inside this repo for now, for tight coupling to the
export contract. Suggested location:
`plugins/krita_paintop/`

Option later: split to a separate repo once the contract stabilizes and the
plugin becomes independently reusable.

## Open decisions
- Target Krita versions for Linux/macOS.
- Build toolchain (Krita SDK vs full source build).
- Animation output format (timeline vs image sequence).

## Reusable terminal commands
Configure (pick a build dir and adjust as needed):
- `cmake -S development/krita-dev/krita -B development/krita-dev/krita/_build -G Ninja -DCMAKE_BUILD_TYPE=RelWithDebInfo` — configure the build.
- `cmake -S development/krita-dev/krita -B development/krita-dev/krita/_build -G Ninja -DKRITA_BUILD_INTEGRATION=OFF` — macOS without Finder integration.

Build + install:
- `ninja -C development/krita-dev/krita/_build` — compile.
- `ninja -C development/krita-dev/krita/_build install` — install into `development/krita-dev/krita/_install`.
- `ninja -C development/krita-dev/krita/_build clean` — clean build outputs.

Run + log:
- `development/krita-dev/run-krita-dev.sh` — run the dev app with the repo env.
- `QT_LOGGING_RULES="krita.lib.resources.debug=true;krita.plugins.debug=true;krita.action.debug=true" development/krita-dev/run-krita-dev.sh 2>&1 | tee /tmp/krita-dev-run.log` — run with logs saved.
- `tail -n 200 /tmp/krita-dev-run.log` — quick log tail.
- `grep -n "krita.action" /tmp/krita-dev-run.log` — search logs (use `grep` if `rg` is missing).

## Build tools installed for this build
- `cmake` — configures the build and generates Ninja files from `CMakeLists.txt`.
- `ninja` — the fast build runner CMake targets (compiles and installs).
- `clang`/`clang++` — C/C++ compiler (from Xcode or Command Line Tools on macOS).
- `python3` — required for Krita’s Python plugin system and SIP tooling.
- `sip-build` — generates Python bindings metadata used by Krita’s Python plugin system.
- `PyQt5` — Python Qt bindings required by Krita’s Python plugins.
- `ccache` — speeds up rebuilds by caching compiled objects.
- `git` — source control for fetching and updating the Krita source tree.

## Dev build status (macOS)
Progress:
- Homebrew toolchain installed (cmake/ninja/python/ccache).
- Krita source checked out at `release/5.2.13`.
- KDE CI deps fetched into `development/krita-dev/krita/_install` (master deps).

Blocked:
- Build stopped at `krita/integration` because `xcodebuild` requires full Xcode.

Next steps (macOS):
1) Install Xcode (App Store).
2) `sudo xcode-select -s /Applications/Xcode.app/Contents/Developer`
3) `sudo xcodebuild -license accept`
4) Re-run the build command.

Alternate path (no Xcode):
- Disable Finder integration targets (QuickLook/Spotlight) by configuring
  `-DKRITA_BUILD_INTEGRATION=OFF`. This skips the `xcodebuild` step and keeps
  the core Krita app intact (no Finder previews/thumbnails).
