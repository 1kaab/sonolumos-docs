# Net Transport

## Summary
Net transport provides a UI-agnostic layer for sending and receiving motion/event payloads over OSC, UDP, or TCP. It exposes a single interface (`start`, `stop`, `send`, `poll`) so the engine and CLI can talk to external systems without coupling to a specific protocol.

## Use cases
- Send motion frames (FOV/Az/Alt + optional debug) to Reaper/SLUM/serial bridges.
- Receive live control or feedback messages (future).
- Push low-rate config/log data over a reliable channel (TCP).

## Protocols
- **OSC**: uses `pythonosc`. Sends a JSON string to `osc_prefix`; receives any OSC message and converts it to `{"address", "args"}`. If a single string arg is valid JSON, it is parsed into a dict.
- **UDP**: raw JSON datagrams; no delivery guarantee, minimal overhead.
- **TCP**: line-delimited JSON messages; reliable delivery, higher latency. Send opens a short-lived client connection per message.

## TransportConfig
Defaults from `engine/net_transport.py`:
- `protocol`: `osc` | `udp` | `tcp`
- `mode`: `send` | `receive` | `both`
- `host`: target host for send (default `127.0.0.1`)
- `port`: target port for send (default `9000`)
- `bind_host`: listen host for receive (default `0.0.0.0`)
- `bind_port`: listen port for receive (default `9001`)
- `osc_prefix`: OSC address for send (default `/slum`)
- `tcp_timeout`: client connect timeout (default `1.0`)
- `udp_max_bytes`: max UDP payload (default `65507`)

Config path: `cfg.network.connection` (persisted via Settings → Network).

## Payload format
The transport layer only JSON-encodes/decodes. A small runtime helper (`engine/net_runtime.py`) builds a canonical envelope for motion payloads:
```json
{
  "schema": "slum.motion.v1",
  "type": "motion",
  "ts": 1700000000.0,
  "seq": 12,
  "meta": {
    "profile": "motion",
    "fps": 30,
    "duration_s": 10.0,
    "source": "ui"
  },
  "payload": {
    "series": {
      "fov": [60.1, 60.2],
      "az": [0.01, 0.02],
      "alt": [-0.01, 0.0],
      "dx": [0.02, 0.03],
      "dy": [-0.01, -0.02]
    },
    "taps": {
      "fov_geom": [0.4, 0.41]
    }
  }
}
```
Other suggested `type` values: `event`, `training` (future).

## Usage sketch
```python
from sonolumos_v3.engine.net_transport import TransportConfig, build_transport

cfg = TransportConfig(protocol="udp", mode="send", host="127.0.0.1", port=9002)
transport = build_transport(cfg)
transport.start()
transport.send({"type": "motion", "ts": 0.0, "payload": {"fov": 0.5}})
transport.stop()
```

## Runtime behavior
- Receive modes push decoded messages into an internal queue.
- `poll()` drains the queue; the engine or UI can call it per frame or on a timer.
- UDP drops oversized packets; TCP waits for newline to parse a message.

## Integration status
- Implemented in `engine/net_transport.py` with a helper manager in `engine/net_runtime.py`.
- Settings → Network stores connection + payload config and apply starts/stops the transport.
- Existing `engine/osc_server.py` still handles Reaper transport status (play/stop/time).

## Marker/command events (planned)
If Reaper can emit marker events over OSC/UDP, marker text can be treated as a command string:
```json
{
  "schema": "slum.event.v1",
  "type": "event",
  "event": "marker",
  "payload": {
    "text": "set fov_blend_g=0.4",
    "pos_s": 12.3
  }
}
```
Planned: a small, safe command grammar (`set key=value`, `toggle key`, `profile motion`) with a whitelist of allowed keys. No `eval`.

Next step: add a receive poll loop + command parser so marker/event payloads can update live state.

## Key files
- `engine/net_transport.py`
- `engine/net_runtime.py`
- `engine/osc_server.py`
