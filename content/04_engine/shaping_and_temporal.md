# Shaping, Temporal, and Domains

## Purpose
Three small utility modules that handle signal normalization, time-domain operations, and value
constraint. They are foundational helpers used throughout the engine but have no single home in
the higher-level pipeline docs.

---

## `engine/shaping.py` — signal normalization helpers

### `remap_minmax(curve, new_min, new_max)`
Linearly rescales a curve from its observed [min, max] to [new_min, new_max].
Returns the midpoint if the input is flat. Used at multiple points in routing to put raw feature
curves into a common range before processing.

### `normalize_01(x)` → `(y, lo, hi)`
Returns `(normalized_curve, original_min, original_max)`. Preserving the originals allows
inverse mapping later if needed. Returns zeros if input is flat.

### `normalize_and_smooth(x, sigma)`
Convenience: normalize to 0–1, then optionally apply gaussian smoothing with `sigma` in frames.
Used in geometry and brownian stages where a soft pre-normalized input is needed.

### Commented experiments (not active)
`time_bend_curve` and `snake_bend_curve` were two approaches to time-varying envelope shaping:
- `time_bend_curve`: linearly morphs the curve's min/max envelope from a start range to an end
  range over the duration. Creates a "fade in" or "drift out" effect.
- `snake_bend_curve`: adds a sinusoidal offset to the normalized curve, creating a breathing or
  oscillating modulation over time.
Neither is wired into the current pipeline but the intent is to use them as post-routing
modifiers on specific parameters.

---

## `engine/temporal.py` — time, resampling, and aliasing

### Constants
- `ENGINE_FPS_DEFAULT = 120` — default internal simulation rate.
- `REF_FPS = 30.0` — the reference rate against which slider values are "voiced". When engine
  FPS differs from REF_FPS, dynamics parameters (spring, damping, etc.) are rescaled so the
  knob feel stays consistent regardless of the actual FPS setting.

### `coerce_fps(fps)` → int
Single normalization point for FPS values: handles None, non-finite, and zero; returns at least
1. Always use this instead of `int(fps)` directly.

### `dt_from_fps(fps)` → float
Returns `1/fps`. The canonical timestep for dynamics integration — use instead of `1/fps`
inline so that zero-FPS edge cases are handled.

### `generate_time(duration_s, fps)` → ndarray
The canonical motion timeline. Always use this to build `t` arrays; it uses `coerce_fps` and
`np.round` to avoid off-by-one frame counts.

### `resample_curve(x, duration_s, src_fps, dst_fps)` → ndarray
Linear interpolation from one FPS timeline to another. Used in:
- Preview downsampling (engine FPS → preview FPS).
- Export downsampling (engine FPS → export FPS).
- Feature alignment (audio hop FPS → engine FPS via `np.interp` in `motion.py`).

### `apply_alias(signal, fps, alias_fps)` → ndarray
Sample-and-hold downsampling: picks one value per block of `fps/alias_fps` frames and holds it.
Creates a stepped, quantized appearance. Used on `fov_final_norm` when `alias_fps` is set in the
routing config — gives a "frame-rate locked" or "strobed" feel to the FOV curve.

### `gaussian_smooth(curve, sigma_frames)` → ndarray
Thin wrapper around `scipy.ndimage.gaussian_filter1d` with sigma expressed in frames (not
seconds). Used in geometry pre-smoothing and in the brownian energy driver smoothing step.

### Velocity layer (less-used)
`VelocityTemporalConfig` and `apply_temporal_velocity_layer` implement an optional
velocity-domain gain/smooth on motion curves: computes the derivative of a curve, scales it,
and reintegrates. This creates an "acceleration amplifier" effect. Currently not wired into
any UI controls but available for future use.

---

## `engine/domains.py` — value constraints

### `DomainSpec(min, max, periodic, clamp)`
A frozen dataclass describing the valid range and boundary behavior for a named signal:
- `periodic=True` → wrap into `[min, max)`. Use for azimuth (0–360°) and phase signals.
- `clamp=True` → hard-clip to `[min, max]`. Use for altitude (−90–90°) and normalized curves.

### `apply_domains(motion, domains)` → dict
Takes a motion dict and a `{key: DomainSpec}` mapping. Applies wrap or clamp only to keys that
appear in both. Everything else passes through unchanged.

### `domains_from_cfg(cfg["domains"])` → dict
Parses the `domains` section of settings.json into `DomainSpec` objects. Allows domain rules to
be set in config without code changes.

### Current wiring
`build_motion()` in `motion.py` calls `apply_domains` after GMH routing if a `domains` dict is
provided in the `RoutingConfig`. In practice, azimuth wrapping and altitude clamping are handled
explicitly in `geometry.py` (`wrap_azimuth`, `clamp_altitude`) rather than via `DomainSpec`.
The `DomainSpec` path is available but not the primary mechanism yet.

### Planned role (from namespace_map.md)
Once namespaced keys are in use, `DomainSpec` entries will enforce contracts on every named
signal — not just the final `az`/`alt` outputs — so intermediate geometry and medium curves
stay within expected ranges.

---

## Key files
- `engine/shaping.py`
- `engine/temporal.py`
- `engine/domains.py`
- `engine/motion.py` (uses all three)
- `engine/routing.py` (uses shaping + temporal)
- `engine/geometry.py` (`wrap_azimuth`, `clamp_altitude`)
