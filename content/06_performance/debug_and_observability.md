# Debug and Observability

## Quick capture (minimum viable)
```bash
mkdir -p logs
PYTHONUNBUFFERED=1 PYTHONFAULTHANDLER=1 python -m ui.app 2>&1 | tee logs/ui_run_$(date +%Y%m%d_%H%M%S).log
```
Also capture: browser DevTools Console (Preserve log on), WebSocket close code (DevTools Network → WS entry).

## Repro template (use for every report)
- OS + Python (e.g. "macOS 14.6, Python 3.12")
- Audio input: file name, size, duration, sample rate
- Tab + mode + FX state at time of failure
- Trigger action: drag, play, switch tab, load preset
- Timestamp of failure
- Attach log file + console output

---

## Known failure patterns

### NiceGUI slot stack empty
UI updates fired from a background task without a slot context.
Fix: update UI inside a known container (`with container:`) or use client-scoped calls.

### UI stalls
Heavy CPU work on the event loop — motion build, Plotly re-render, large WAV analysis.
Fix: move to background tasks, throttle UI updates, decimate data.

### Payload overload
Very large series or huge JSON updates passed to canvas or Plotly.
Fix: cap points, downsample to preview FPS, use preview-only data paths.

### Memory blow (large WAV + long session)
Upload reads entire file into RAM (`await e.file.read()`), `sf.read()` creates another
full buffer, optional resample adds copies. On large files (> ~200MB) this can cause
a server restart, which resets the browser to the start page.

Triage (no code change):
- Check terminal logs for crashes around the reload.
- Monitor memory during upload (Activity Monitor).
- Keep `audio_dtype` at float32.
- Lower `audio_target_sr` when resampling.
- Increase `hop_ms` to reduce feature frame count.

Action plan (future):
- Stream/segment decode instead of loading full WAV.
- Store audio on disk + mmap or windowed analysis.
- Add lightweight telemetry: memory + timing around upload/analyze.

### Trajectory modes not responding (xy/polar/sphere)
Reported behavior: first three trajectory modes appear identical after UI changes.
Hypotheses: `traj_plot_mode` changes but canvas does not refresh; render preview state
overrides output; motion arrays are too flat (dx/dy ≈ az/alt).
Steps: log mode changes + confirm `update_motion_canvas` fires; compare `dx/dy` vs
`az/alt` series; verify `traj_plot_mode` survives tab refresh/mount.

---

## Performance metrics (Settings → Performance)
- Rebuilds per second (motion build rate)
- UI payload sizes for motion dict and taps dict
- Whether last_motion/last_taps are retained in state
- Timer active status while Dynamics tab is inactive
- Resample duplication detection (same curve resampled twice)

Canvas updates are throttled; Plotly 3D is opt-in and off by default.

## Immune system features (planned)
- **Structured logs**: per-run log file with run ID + version hash.
- **UI action ring buffer**: last N actions (tab switches, preset loads) in memory.
- **Auto snapshot**: write current config + selected file on crash.
- **Safe mode**: one-click disable of render overlays and 3D preview.
- **Performance counters**: build time, payload size, 3D render metrics in Settings → Performance.

---

## Where to look in code
- `ui/tabs/dynamics_tab.py` — motion build, throttling, preview refresh
- `ui/canvas_viz.py` — 2D canvas, overlay rendering
- `engine/audio.py` — feature extraction (upstream of build, often the memory hotspot)
- `engine/routing.py` + `engine/motion.py` — heavy computation
- `ui/app.py` — background task management, MIDI/serial drain loops
