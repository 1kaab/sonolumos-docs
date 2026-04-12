# GMH Phase Model (Geometry, Medium, Hysteresis)

## Purpose
Define the GMH philosophy as an execution model for motion generation, genome evolution, and UI monitoring.

## Phase meanings
### Geometry (ice)
- Stable structure with relative permanence.
- Identity-like features derived directly from the sample.
- Serves as the foundation of a sample or composition fingerprint.

### Medium (water)
- Change and movement (flow).
- Phase of contact and transfer.
- Most expressive and typically dominant in motion.

### Hysteresis (vapor)
- Memory and entropy.
- Time-transcendent behavior: a convergence toward "now" from both past and future signals.
- More uncertainty can be more truth in this phase.

## Execution model
### Phase partitioning
- Treat parameters as three phase blocks: geometry, medium, hysteresis.
- Each phase produces its own output curve or field.
- A weighted sum produces the final motion and the visual fingerprint.

### FOV as genome product
- FOV is not a fixed input; it is the outcome of GMH evolution.
- Each phase has its own params and can be trained toward different objectives.
- Phase weights become part of the genome and can be saved per FOV mode.

### Genome evolution (proper)
- Phase-separated genome:
  - Geometry genes: slow mutation, stability bias.
  - Medium genes: moderate mutation, expressive bias.
  - Hysteresis genes: faster mutation, exploration bias.
- Optionally evolve phase weights (ice vs water vs vapor dominance).
- Seed can be fixed, evolved, or derived from audio fingerprints depending on objective.

## UI monitoring (GMH dashboard)
- GMH multilayer plot with selectable phases (ice/water/vapor).
- 2D macro controls per phase (up to 3 per phase).
- Live param readout for the selected phase.
- Dominance indicator (percent weight per phase).
- Training overlay:
  - Dot size shrinks near optimal regions, grows where penalties appear.
  - Trails reveal phase evolution over time.

## Training philosophy
- Objectives guide pattern discovery, not rigid outcomes.
- Training should allow organic evolution while staying grounded in sample identity.
- Visual feedback is part of selection, not a replacement for audio-based evaluation.
- Long-term: integrate bio-signal inputs as feedback or as training constraints.

## Trainer layer
- The KLC trainer (Crystal / Language / Cell) optimizes GMH parameters via scoring objectives.
- KLC produces trajectories that feed the output pipeline (API/serial/export).
- See [[01_architecture/klc_training_model]] for the current training roadmap.

## Key files
- engine/geometry.py (planned)
- engine/medium.py
- engine/temporal.py
- engine/motion.py
- ui/tabs/dynamics_tab.py
- ui/tabs/settings_tab.py
- ui/tabs/training_tab.py
