# Routing and Network

## Summary
Routing maps audio features (including energy packets) to FOV/Az/Alt generators. Energy packets are defined in Settings -> Network and shown in the Network tab as nodes.

## Flow
```mermaid
graph TD
  Packets[energy packets (settings)] --> Analyze[analyze_audio with band]
  Analyze --> Extra[audio_features.extra]
  Extra --> Features[features_to_dict]
  Features --> Routing[routing config]
  Routing --> Motion[motion build]
  UI[network graph] --> Routing
```

## Key concepts
- Energy packet = a named band RMS curve derived from a frequency range.
- Packets live in `state["energy_packets"]` and are applied against loaded audio.
- Packet nodes are treated like features in the routing graph.
- `brown_source` can be set to any feature or packet.

## Transport options (OSC vs TCP/UDP)
- **OSC**: message format; typically runs over **UDP**. Good for realtime control, low overhead, but not reliable delivery.
- **UDP (raw)**: lowest latency; no delivery guarantee; good for frequent small packets (motion frames).
- **TCP**: reliable, ordered delivery; higher latency; better for config, logs, and guaranteed commands.

## Connection (Settings → Network)
Connection controls persist to `cfg.network` and are applied when Settings are applied:
- Protocol: `osc`, `udp`, `tcp`
- Host + port (target and listen)
- Mode: `send`, `receive`, `both`
- OSC prefix (send)
- Payload profile + series keys (motion/event/training)

Implementation status:
1) **Abstract transport layer**: `engine/net_transport.py` with a common `send()`/`poll()` API (implemented).
2) **Runtime helper**: `engine/net_runtime.py` applies config + builds motion payloads (implemented).
3) **UI wiring**: Settings → Network controls write to config and start/stop the selected backend (implemented).
4) **Legacy OSC**: `engine/osc_server.py` remains for Reaper transport status.

## Key files
- ui/tabs/routing_tab.py
- ui/tabs/settings_tab.py
- engine/audio.py
- engine/routing.py
- engine/net_transport.py
- engine/osc_server.py
- ui/state.py

## See also
- [[02_data_flow/network_topology]]
- [[01_architecture/namespace_map]]
