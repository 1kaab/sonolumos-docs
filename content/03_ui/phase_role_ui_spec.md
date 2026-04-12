# Phase Role UI Spec (Ice / Water / Vapor / Genome)

## Purpose
Expose the GMH philosophy in the UI without changing the underlying math models.
Keep the interface intuitive, phase‑scoped, and ready for training/monitoring.

## Terms
- **Phase Role**: semantic meaning (ice, water, vapor, genome).
- **Model**: the math mode (cap/ind/mixed/waveform).
- **Weight**: phase contribution to the final motion (geo/med/hys blend).

## Role → Model mapping (default)
- **Ice** → Model: Cap | Weights: geo 0.75, med 0.15, hys 0.10
- **Water** → Model: Ind | Weights: geo 0.20, med 0.60, hys 0.20
- **Vapor** → Model: Mixed | Weights: geo 0.15, med 0.20, hys 0.65
- **Genome** → Model: Mixed | Weights: learned per objective

## UI placement
- **Phase Role selector**: replaces (or sits above) fov_mode in the UI.
- **Model selector** (cap/ind/mixed/waveform): still visible, but driven by Phase Role defaults.
- **Phase weight strip**: three sliders or a triangle widget (geo/med/hys).
- **Macro pad**: one M macro per phase; only the active phase is shown.

## Interaction rules
- Selecting a **Phase Role** updates:
  - Default model (cap/ind/mixed)
  - Default phase weights
  - Active macro pad
- **Precision by size**:
  - Macro dot size shrinks as optimizer converges.
  - Dot grows when exploration/penalty increases.
- **Preview**:
  - Role changes should be reversible with a restore action.

## Data bindings (minimum)
- `fov_role`: "ice" | "water" | "vapor" | "genome"
- `fov_mode`: "Cap" | "Ind" | "Mixed" | "Waveform"
- `phase_weights`: {geo, med, hys}
- Phase macro bindings:
  - Ice → geometry params
  - Water → medium params
  - Vapor → hysteresis params

## Non‑goals (for now)
- No changes to engine math models.
- No additional routes or extra macros beyond one M per phase.

## Implementation notes
- Keep **model** and **role** separate. Role is semantic; model is math.
- Role mapping can be tuned per objective without breaking routing.
- Later: genome can learn weights and override role defaults.
