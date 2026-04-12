# Routing Tab

## Purpose
Expose the routing graph, MIDI input mapping, and direct drag connections between features/generators/packets and outputs.

## UI pieces
- Node graph (features, packets, generators, outputs).
- MIDI device + mapping panel.

## Data handling
- Packet nodes are sourced from Settings -> Network (`state["energy_packets"]`).
- Dragging a node updates `state["routing"]` or `state["brown_source"]`.

## Current limitation: route persistence
- The Signal Flow graph currently edits live runtime state only.
- Dragging a route updates `state["routing"]` in `ui/tabs/routing_tab.py`.
- `Apply settings` persists `cfg_draft` to `cfg`, then `hydrate_from_cfg()` rebuilds `state["routing"]` from committed config.
- Because the graph does not write back into `cfg_draft["routing"]["routes"]`, graph edits are overwritten on apply/reload.
- Result: route changes affect the current session, but do not persist.

## Live 8-knob suggestion (Ableton .adg)
- `1. Drive` -> `gain`: global motion intensity, closest to effect amount/drive.
- `2. Gate` -> `threshold`: sets how much motion passes, like a gate threshold.
- `3. Hold` -> `memory`: keeps motion lingering, similar to feedback/hold.
- `4. Release` -> `decay`: controls how fast motion falls away.
- `5. Smooth` -> `smoothing`: removes jitter, like slew/lag.
- `6. Size` -> `fov_scale`: changes overall visual span/opening.
- `7. Motion` -> `brown_step_scale`: changes travel distance, like modulation depth.
- `8. Chaos` -> `brown_turn_noise`: changes randomness/complexity.

## Why this bank fits
- It mirrors the main engine groupings: hysteresis, final/FOV, and trajectory.
- It gives eight controls that are high-level and legible in performance.
- It avoids more technical coefficients (`cap_alpha`, `ind_beta`, `field_strength`, `loop_width`, `brown_inertia`) that are harder to read live.

## Key files
- ui/tabs/routing_tab.py
