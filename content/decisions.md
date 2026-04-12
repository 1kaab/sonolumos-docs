# Architecture Decisions

Template:
- Date:
- Decision:
- Context:
- Consequences:

---

- Date: 2026-01-23
- Decision: Document the system in Obsidian with per-module notes and mermaid diagrams.
- Context: Need a durable, navigable internal reference before major changes.
- Consequences: Docs need updates when modules change.

---

- Date: ~2025-Q4
- Decision: Switch visualization from Plotly to HTML5 Canvas for the primary motion and
  trajectory views.
- Context: Plotly re-renders full figures on every update, causing UI stalls especially
  during live parameter edits. The 2D trajectory and FOV overlay don't need Plotly's
  interactivity — they need fast, low-cost updates.
- Consequences: Canvas renders are lighter and always-on; Plotly kept for FOV stage
  overlays (G/M/H layers) where the chart interactivity is useful. 3D views remain
  Plotly but are opt-in and off by default.

---

- Date: ~2025-Q4
- Decision: Use Ornstein-Uhlenbeck (OU) process for Brownian pan instead of two
  independent 1D random walks (legacy mode).
- Context: The legacy walk produces "scribble" trajectories — az and alt evolve
  independently with no directional memory, making the path look random rather than
  organic. The OU heading model evolves a single heading angle with inertia, producing
  longer arcs and coherent motion paths that reflect audio energy as volatility.
- Consequences: OU is now the default mode. Legacy walk is preserved under `mode="legacy"`
  for compatibility. OU has two additional parameters: `inertia` (heading persistence)
  and `turn_noise` (diffusion strength).

---

- Date: ~2025-Q4
- Decision: Introduce FOV mode gains (molecular / deep_space / constellations / sky)
  to modulate pan and FOV amplitude based on the current field-of-view bucket.
- Context: At very narrow FOVs (< 2.6°, molecular scale), even small az/alt excursions
  feel violent. At wide FOVs (sky, > 160°), more movement is needed to feel responsive.
  A flat amplitude mapping does not translate well across the full FOV range.
- Consequences: `pan_gain` and `fov_gain` are computed per-frame from the final FOV curve.
  This makes az/alt behavior FOV-dependent. The thresholds (2.6°, 6.18°, 160°) and gain
  values are currently hardcoded in `motion.py` and should eventually be configurable.

---

- Date: ~2025-Q4
- Decision: Keep the `structure` objective as the primary training target; treat
  `entropy`, `fractal`, and other objectives as experimental.
- Context: Structure (mutual information + excess entropy on `fov_final_norm`) produces
  the most stable, repeatable training results. Entropy (2D occupancy grid) and fractal
  (entropy + box-count) produce useful but harder-to-tune results. Composite objectives
  were deferred until single objectives are stable and interpretable.
- Consequences: Training documentation and phase guidance all assume `structure` as the
  baseline. Other objectives remain in the codebase but are marked experimental.

---

- Date: ~2025-Q4
- Decision: Move training orchestration out of the UI tab and into `training/runner.py`
  with no NiceGUI imports.
- Context: Training runs were tied to the UI lifecycle. If the browser disconnected or
  the tab was closed, a run could be interrupted or leave the state inconsistent.
  Decoupling training from the UI allows headless runs, reproducible results, and a
  path to a CLI interface.
- Consequences: `training/runner.py` operates on snapshots of state taken at run start.
  UI tweaks during training do not alter running results. The UI tab becomes a thin
  progress monitor. A CLI (klc) is planned but not yet implemented.

---

- Date: ~2025-Q4
- Decision: Represent GMH phase roles semantically (ice / water / vapor) in the UI,
  separate from the underlying math models (capacitor / inductor / mixed).
- Context: The math model names (cap, ind) are accurate but opaque. The phase role names
  (ice = stable/structural, water = expressive/fluid, vapor = memory/entropic) communicate
  the intended behavior and map to the training philosophy. Role and model are kept
  separate so roles can be redefined per objective without changing routing math.
- Consequences: `phase_role_ui_spec.md` defines the role→model default mapping. The UI
  selector and macro pads are designed around roles, not models. Not yet fully implemented
  in the UI.

---

- Date: 2026-04-11
- Decision: Add a `05_integration/` documentation section and `archive/` directory.
  Move raw unstructured AI-generated content (reaper.md, Color.md) to archive.
  Promote hardware controller and 44/Reaper integration to first-class docs.
- Context: The vault contained raw conversational AI responses filed as docs. Their
  content had value (3-clock model, hardware sketch) but was inaccessible due to format.
  The 44 project had no documentation connecting it to slum.
- Consequences: archive/ holds raw material for reference; it is not part of the active
  doc map. New docs in 05_integration/ cover hardware, 44, and Reaper integration
  as explicit first-class concerns.
