# Sonolumos Vault

This vault documents the **slum** (sonolumos_v3) motion engine and the **44** audio tool.
Both live in the same repository. See `[[01_architecture/overview]]` for the ecosystem map.

---

## Architecture
- [[01_architecture/overview]] — ecosystem map, signal chain, layer responsibilities
- [[01_architecture/gmh_phase_model]] — ice / water / vapor philosophy and execution model
- [[01_architecture/klc_training_model]] — KLC trainer, objectives, CLI spec
- [[01_architecture/namespace_map]] — signal key scheme, migration plan, central function list
- [[01_architecture/backend_separation_roadmap]] — UI decoupling, genome scope, headless training
- [[01_architecture/audio_streaming_pipeline]] — live audio input design, ring buffer, output adapters
- [[01_architecture/training_evolution_log]] — training run history and applied-params log

## Data flow
- [[02_data_flow/audio_pipeline]] — WAV upload → feature extraction
- [[02_data_flow/motion_pipeline]] — GMH stages, parameter tuning, outputs
- [[02_data_flow/routing_and_network]] — energy packets, transport options, OSC vs UDP vs TCP
- [[02_data_flow/network_topology]] — full system topology diagram
- [[02_data_flow/reaper_feedback_fitness]] — 44 feedback fitness loop (future)

## UI
- [[03_ui/dynamics_tab]] — FOV canvas, macro overlay, trajectory view, serial timing
- [[03_ui/routing_tab]] — signal flow graph, MIDI mappings, route persistence note
- [[03_ui/spectral_tab]] — spectral matrix, harmonic overlay, expression signals
- [[03_ui/settings_tab]] — all config subtabs, DEAP controls, export options
- [[03_ui/training_tab]] — training lifecycle, objectives, genome config, results view
- [[03_ui/testing_tab]] — OSC/HTTP/Blender/Serial/MIDI integration panels
- [[03_ui/phase_role_ui_spec]] — ice/water/vapor role selector spec (unimplemented)

## Engine
- [[04_engine/brownian]] — OU walk, legacy walk, parameters, outputs
- [[04_engine/nodes_and_namespaces]] — node protocol + namespace system (transition layer)
- [[04_engine/midi_runtime]] — MIDI CC/note mapping, drain loop, 8-knob bank
- [[04_engine/shaping_and_temporal]] — normalization, resampling, aliasing, domains
- [[04_engine/net_transport]] — OSC/UDP/TCP payload schema, command grammar
- [[04_engine/temporal_and_domains]] — (older; see shaping_and_temporal for expanded version)
- [[04_engine/streaming]] — streaming pipeline, state dataclasses, StreamingRouter, normalization design

## Export
- [[05_export/csv]] — CSV columns, Stellarium format, export FPS
- [[05_export/krita]] — Krita paint-op, Molit mapping table, build status, render config

## Integration
- [[05_integration/44_reaper]] — 44 sample tool, Reaper scripts, audio pipeline into slum
- [[05_integration/hardware_controller]] — Arduino pot/button/LED circuit and serial sketch

## Performance
- [[06_performance/debug_and_observability]] — failure patterns, capture procedure, metrics

## Reference
- [[glossary]] — key terms
- [[decisions]] — architecture decision records

---

## Conventions
- Use module names as headings.
- Add `Key files` sections with relative paths.
- Prefer ASCII text and short paragraphs.
- Mermaid diagrams in fenced ` ```mermaid ` blocks.
- Mark unimplemented features with `(planned)` or `(not yet wired)`.

## Quick update checklist
- Update `02_data_flow/` when state keys or signals change.
- Update `03_ui/` tab notes when layouts or behaviors change.
- Update `05_export/csv` when CSV columns change.
- Add an ADR in `[[decisions]]` for any major design choice.
- Update `01_architecture/training_evolution_log` when training runs are applied.
