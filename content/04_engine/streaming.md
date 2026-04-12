# Streaming Pipeline

## Parallel path, not a replacement

The batch pipeline processes a complete WAV file in one shot: full feature arrays
go in, full motion arrays come out. That workflow stays intact.

Streaming adds a parallel path for live audio input. One chunk arrives at a time,
one feature scalar is extracted per chunk, one output value leaves per chunk.
The same GMH stages run — but with no lookahead, producing output on every frame
continuously.

---

## The GMH stages as simulations

Each stage is a discretized differential equation advancing through time as a
for-loop. The loop state (`out[i-1]`, `vel`, `mem`) is the physical system mid-run.

- **Geometry / Capacitor**: stores charge. Output depends on the previous output.
- **Medium / Viscoelastic**: has position and velocity. Each frame moves the mass.
- **Hysteresis**: accumulates internal force (`mem`). Output depends on the history
  of how the input has been moving.

In batch mode the simulation runs start to finish on a complete signal.
In streaming mode it runs one tick at a time, pausing between audio chunks and
resuming when the next arrives. The math is identical — only where the state
lives between ticks changes.

---

## State dataclasses

Each stage exposes a state dataclass and a `step()` function:

```
GeometryState   → step_geometry(x, state, params)         → engine/geometry.py
MediumState     → step_medium(x, state, params, dt, dt_ref) → engine/medium.py
HysteresisState → step_hysteresis(x, state, params, dt)   → engine/hysteresis.py
```

State is separated from params deliberately. Params change when the user moves
a slider. State is the running physics — it must survive parameter changes.
`update_config()` in StreamingRouter swaps params without touching state.
Explicit `reset()` is required to restart the simulation.

---

## Normalization: the streaming problem

In batch mode, `normalize_feature()` in geometry sees the full signal and
normalizes globally (subtract mean, divide by max abs). In streaming, no
lookahead exists.

Three strategies with different tradeoffs:

| Mode | What it does | Academic use | Artistic use |
|---|---|---|---|
| **None** | raw feature value goes directly into geometry | transparent, source-dependent | requires pre-calibrated source |
| **Running min/max** | track min/max over a sliding window, normalize to [0, 1] | range drifts, window is a free parameter | responsive, source-independent |
| **Fixed range** | user sets min/max per feature manually | reproducible, comparable across runs | requires knowing the source |

**None is not "no normalization" — it is a normalization strategy.**
It says: the source is the reference. The GMH parameters are tuned against
whatever the source produces. This is correct when you have a stable,
calibrated source. It breaks when the source changes unexpectedly.

**Running min/max does not solve the normalization problem — it trades it.**
Instead of "unexpected absolute shift", you get "continuous slow drift" as the
window advances. Whether this is better depends on what the motion should mean.

### What the GMH stages actually expect

- **Geometry** normalizes its input internally (batch). The step function skips
  this — streaming input should be pre-normalized or use a running normalizer.
  `cap_alpha`, `cap_leak`, `ind_beta`, `ind_gamma` were tuned against `[-1, 1]`.
- **Medium** does not normalize. It tracks the input as a moving equilibrium.
  The spring pulls toward `x_in[i]`. Medium is range-agnostic by design —
  the physics are self-consistent at any scale.
- **Hysteresis** responds to `delta = x[i] - prev_out`. Purely differential,
  range-independent except for `threshold`, which has an implicit scale assumption.

**The design consequence**: normalization should be an explicit stage in the
streaming signal chain with its own parameters, not a hidden preprocessing step.
A `RunningNormalizer` layer is planned — see the streaming settings spec in
`03_ui/settings_tab.md` (planned).

---

## StreamingRouter

`engine/stream/router.py`

Holds one `RouteState` per route. `RouteState` bundles `GeometryState`,
`MediumState`, `HysteresisState` for one parameter (fov / az / alt / ...).

```python
router = StreamingRouter(config, fps=30.0)
out, taps = router.step({"rms": 0.42, "centroid": 0.61})
# out  -> {"fov": 0.38, "az": 142.1, "alt": -12.4}
# taps -> {"fov.feature": 0.42, "fov.g": ..., "fov.m": ..., "fov.h": ...}
```

`taps` mirrors the batch taps dict but holds scalars instead of arrays.
This is the feed for the scope panel (Settings → Streaming → Monitor).

### FOV blend in streaming

The batch pipeline normalizes G/M/H to `[0, 1]` before blending. The
streaming router blends raw values. Without a running normalizer the three
stages may have different scales, shifting the blend balance at extreme
parameter settings. A `RunningNormalizer` layer will fix this.

### Key methods

| Method | Behavior |
|---|---|
| `step(features)` | process one frame, advance all physics states |
| `update_config(config)` | swap params, preserve state |
| `reset(route_name=None)` | reinitialize state for one route or all |
| `set_fps(fps)` | update frame rate, no state reset |

---

## Remaining layers

```
[mic / line-in]
    → sounddevice input callback
    → feature extraction per chunk (engine/audio.py, per-frame variant)
    → RunningNormalizer (planned)
    → StreamingRouter.step()
    → out dict (fov, az, alt scalars)
    → net_runtime / serial / OSC output
```

Audio input and RunningNormalizer are not yet implemented. The streaming path
runs independently of the batch pipeline and the NiceGUI event loop. For
tight-timing outputs (serial motors, camera), the intended architecture is a
separate process feeding via net_runtime.

---

## Key files

- `engine/stream/router.py` — StreamingRouter, RouteState
- `engine/geometry.py` — GeometryState, step_geometry
- `engine/medium.py` — MediumState, step_medium
- `engine/hysteresis.py` — HysteresisState, step_hysteresis
- `engine/routing.py` — RoutingConfig (shared with batch pipeline)
