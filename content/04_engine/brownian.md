# Brownian Pan

## Summary
Brownian pan generates an audio-driven trajectory and mixes it with az/alt. It uses an energy driver derived from a selected feature (or packet), applies shaping/smoothing, and then walks either via the structured OU model or a legacy scribble mode. The driver is computed early, but the mix happens after FOV mode gains are applied.

## Flow
```mermaid
graph TD
  Feature[feature or packet] --> Energy[normalize 0..1]
  Energy --> Shape[shape + floor + smooth]
  Shape --> Walk[OU walk or legacy walk]
  Walk --> Mix[mix with az/alt (post FOV gains)]
  Mix --> Motion[az/alt + az_raw/alt_raw]
```

## Parameters
- `shape` (0..1): compress/expand energy.
- `floor` (0..1): minimum volatility.
- `inertia`: heading persistence (OU).
- `turn_noise`: direction randomness (OU).
- `turn_rate_deg`: turning intensity (OU) / step scale (legacy).
- `step_scale`: overall movement scale.
- `smooth_s`: smoothing in seconds (energy driver).
- `mode`: `ou` or `legacy`.
- `mix`: 0..1 (0 = no brown, 1 = full override).

## Notes (current behavior)
- Driver energy is `remap_minmax(feature)` → optional gaussian smoothing (`smooth_s` in seconds).
- Volatility uses `shape` + `floor`, then feeds OU or legacy walk.
- Mixed output writes `dx/dy`, then `az_raw/alt_raw`, then wraps/clamps to `az/alt`.

## Legacy vs OU (difference)
- **Legacy**: two independent random walks for az/alt. Volatility scales step size, but direction is memoryless, so it can "scribble" over long runs.
- **OU (heading)**: a single heading evolves with inertia + noise, producing longer arcs and more organic trajectories. Volatility scales turn jitter and step size, so motion has continuity and structure.

## Outputs
- `brown_dx`, `brown_dy` (raw brownian path)
- `dx`, `dy` (mixed output)
- `az_raw`, `alt_raw`
- `brown_source`, `brown_energy01`, `brown_volatility`, `brown_step`
- `brown_mix`, `brown_mode`

## Key files
- engine/brown_pan.py
- engine/motion.py
- engine/routing.py (BrownPanConfig)
