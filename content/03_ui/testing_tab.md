# Testing Tab

## Purpose
Manual integration testing for external targets (Reaper/OSC, Stellarium HTTP, Blender, Arduino serial). This tab is not part of the training pipeline; it triggers external systems with the current state or local scripts.

## Panels
- OSC: Reaper transport status + play/stop via `engine/osc_server` (OSC over UDP).
- HTTP: Stellarium script trigger (`POST http://localhost:8090/api/scripts/run`, id = `sonolumos.ssc`).
- Blender: Launch local Blender with a Python script and optional CSV path (default `stellar.csv`), headless toggle.
- Serial: Arduino serial or simulated source, buffered snapshots sent to a canvas overlay.
- MIDI: placeholder (no current implementation).

## Notes
- Stellarium assumes local API is enabled and a script named `sonolumos.ssc` exists.
- Blender runs via `subprocess.Popen` (fire-and-forget).
- Arduino port/baud are stored in `state` and not validated beyond open errors.

## Key files
- ui/tabs/testing_tab.py
- engine/osc_server.py
