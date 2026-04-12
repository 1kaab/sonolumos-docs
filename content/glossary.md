# Glossary

- audio_features: dataclass with per-frame audio descriptors.
- brownian pan: audio-driven random walk mixed into az/alt (modifier or override).
- cfg: committed config loaded from data/settings.json.
- cfg_draft: editable config before apply.
- energy packet: named band RMS feature from a frequency range.
- GMH: Geometry / Medium / Hysteresis phase model for motion generation.
- KLC: Crystal / Language / Cell trainer that optimizes GMH parameters and scores trajectories.
- motion: dict of time-aligned arrays (fov, az, alt, etc).
- taps: intermediate features captured during routing.
- preview fps: UI resample rate for display.
