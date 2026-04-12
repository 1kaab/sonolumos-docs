# Audio Streaming + Motion Pipeline

## Goal
Support **live audio input** that drives the motion engine in real time, while keeping UI responsiveness and enabling multiple output targets (digital + mechanical).

## Live streaming pipeline (proposed)
1) **Audio input**
   - Live capture (studio session, mic, line‑in).
   - Chunked into a ring buffer (raw PCM).
2) **Analysis**
   - Windowed feature extraction (RMS, centroid, flux, etc).
   - Runs on a worker thread using hop size and FFT size.
3) **Routing**
   - Audio features → routing → motion drivers (fov/az/alt).
4) **Motion build**
   - Geometry + Medium + Temporal + Brownian → trajectory.
   - Output is a motion series (x/y + optional cartesian z).
5) **Outputs**
   - **Digital APIs**: Stellarium / Krita / TouchDesigner.
   - **Mechanical**: serial‑driven motors / LEDs.
6) **UI layer**
   - Slum UI for live tweaks (and optional TD UI layer).

## Motion pipeline (deep assessment)
**Audio → Features**
- `engine/audio.py` extracts spectral / temporal features from windowed audio.
- Optional band RMS (energy packets) adds feature channels.

**Features → Routing**
- Routing selects feature → driver mapping (fov/az/alt).
- Brownian source can override/modify directional outputs.

**Motion Build**
- `engine/controller.py` orchestrates:
  - Geometry layer (cap/ind/mix)
  - Medium layer (viscoelastic/diffusion)
  - Temporal layer (hysteresis)
  - Brownian pan (OU / legacy)
- Output: motion series used by render/exports.

**Render/Outputs**
- UI canvas (2D) is lightweight.
- 3D Plotly view is heavy and must be opt‑in.
- CSV/SVG export consumes the same motion series.

## Ring buffer and timing
**Ring buffer should hold raw PCM only.**
- No curves or remaps on the raw ring buffer.
- Apply transformations downstream (feature or mapping layer).

**Timing budgets**
- Capture hop (e.g., 10 ms) sets feature update rate.
- FFT size determines frequency resolution (larger = slower).
- UI preview FPS should be lower than engine rate to avoid UI overload.

## Curve256: where it belongs
**Not in the ring buffer.**  
Curve256 (or any LUT) should be applied at the **feature → param mapping stage**, not to raw audio:
- Ring buffer must remain linear PCM for correct analysis.
- Curve256 is useful for **perceptual shaping** (log/exp) or artistic response curves.
- Best spot: driver normalization or patch‑bay curve stage.

## Interface config + loopback
For live capture we should expose:
- `input_device`
- `sample_rate`
- `buffer_size`
- `channels`
- `loopback_mode` (off / virtual / system)

Loopback is only needed when the **audio source** is a DAW or system output.  
If audio comes from a mic or line‑in, use the interface directly (no loopback).

## Capturing Slum output (motion)
Slum’s **output is motion**, so we should capture/route **motion streams**, not audio.

Preferred motion output options:
- **OSC/UDP** for low‑latency live control (TD, Unreal, custom rigs).
- **HTTP/WS** for structured event streams or dashboard control.
- **CSV/JSON** for offline playback and reproducible renders.

Architecture detail:
- Motion build should publish a **stream of frames** (or batched chunks).
- Output adapters subscribe to the same motion stream.
- No audio loopback is needed for motion output.

## Output integration map (from live audio)
```
Audio In → Slum Engine Motion → Outputs
  ├─ Digital APIs → (Stellarium / Krita / TD)
  └─ Mechanical → (Serial → motors / LEDs)
```

**Digital APIs**
- OSC / HTTP / CSV for TD or other compositors.
- Krita: use motion series to drive brush engine or render.
- Stellarium: script/OSC for camera/scene changes.

**Mechanical**
- Serial output mirrors selected drivers at throttled rate.
- Use dedicated microcontroller for physical output stability.

## Live session behavior
- Slum UI remains the **primary live control surface**.
- TD UI can sit above for scene‑level composition.
- Motion engine stays deterministic; outputs subscribe to the same motion stream.

## Notes for implementation
- Keep audio capture + feature extraction **off the UI thread**.
- Use a **feature queue** to decouple capture from motion build.
- Expose streaming parameters (buffer size, hop, FFT, throttle) in preferences.

## UI styling centralization (phased)
We will centralize styling in **layers**, starting with the most repeated elements:
1) **Buttons** (primary/secondary/danger styles)
2) **Network graph nodes** (role + stage colors)
3) **Core widgets** (knobs, pills, sliders)

This avoids a massive refactor and keeps UI iteration fast while steadily moving toward a single source of truth.
