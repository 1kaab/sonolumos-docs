# Hardware Controller (Arduino / Serial)

## Purpose
A minimal physical control surface: one potentiometer, one button, one LED, communicating over
USB serial. The pot drives a selected slum parameter; the button cycles the active target.
This is the "tangible knob" complement to the MIDI and OSC interfaces.

## Circuit

```
Potentiometer (the knob)
  Pin 1  → 5V
  Pin 2  (wiper / middle) → A0
  Pin 3  → GND

Button (mode selector)
  Leg 1  → GND
  Leg 2  → D2
  (use INPUT_PULLUP — no resistor needed)

LED (mode indicator, single or RGB)
  D9 → 220Ω resistor → LED (+)
  LED (−) → GND
```

For an RGB LED, extend to three PWM pins (R/G/B). Hue per mode gives instant visual feedback
without numbers. See the color mapping section below.

## Arduino sketch

```c
const int POT_PIN = A0;
const int BTN_PIN = 2;
const int LED_PIN = 9;

int mode = 0;
const int NUM_MODES = 4;

int lastBtn = HIGH;
unsigned long lastDebounce = 0;

void setup() {
  pinMode(BTN_PIN, INPUT_PULLUP);
  pinMode(LED_PIN, OUTPUT);
  Serial.begin(115200);
}

void loop() {
  // button: debounced edge detect
  int btn = digitalRead(BTN_PIN);
  if (btn == LOW && lastBtn == HIGH && millis() - lastDebounce > 200) {
    mode = (mode + 1) % NUM_MODES;
    lastDebounce = millis();
  }
  lastBtn = btn;

  // LED: simple on/off per mode (replace with RGB/PWM for color)
  digitalWrite(LED_PIN, mode >= 2 ? HIGH : LOW);

  // pot: normalize and send
  float norm = analogRead(POT_PIN) / 1023.0;

  Serial.print("mode:");
  Serial.print(mode);
  Serial.print(",value:");
  Serial.println(norm, 4);

  delay(15); // ~65 Hz
}
```

Serial output format per line: `mode:N,value:0.XXXX\n`

## Python-side mapping (`ui/tabs/testing_tab.py` serial panel)

```python
MODE_MAP = {
    0: "brown_shape",
    1: "brown_floor",
    2: "brown_inertia",
    3: "brown_turn_noise",
}

def handle_serial_line(line: str):
    parts = dict(p.split(":") for p in line.strip().split(","))
    mode  = int(parts["mode"])
    value = float(parts["value"])
    param = MODE_MAP.get(mode)
    if param:
        state[param] = value
        bump_motion()
```

## Parameter bank selection (current defaults)
The four-mode brownian bank is a natural starting set. For a wider bank, use the 8-knob
layout from `[[04_engine/midi_runtime]]` applied to serial modes 0–7.

## Color-coding (RGB LED upgrade path)
Assign a fixed hue to each mode. The LED color becomes a muscle-memory indicator of what
the knob is currently shaping. Hue is circular — interpolate angularly, not as RGB values,
to avoid color flicker during mode transitions.

Example hue palette:
| Mode | Target | Hue |
|---|---|---|
| 0 | brown_shape | orange (30°) |
| 1 | brown_floor | blue (210°) |
| 2 | brown_inertia | green (120°) |
| 3 | brown_turn_noise | purple (270°) |

LED state conventions:
- Solid → active selection
- Fade → inactive / standby
- Pulse → input detected (motion triggered)

## Latency budget
- Arduino loop at 115200 baud, `delay(15)` → ~65 Hz updates.
- Python `readline()` with `timeout=0.1` adds up to 100ms worst case.
- `param_rate_hz` (default 20 Hz) throttles how often state is written.
- Total worst-case: 10–150ms from pot movement to plot update.
- To reduce: raise baud to 115200 (already above), keep line short (already `mode:N,value:X.XXXX\n`),
  raise `param_rate_hz` to 40 Hz.

## Upgrade path
1. Single LED → RGB LED → NeoPixel strip (one data pin, per-mode color + brightness = value).
2. Single pot → rotary encoder (relative, no jump on mode switch).
3. Single button → bank select button (two buttons: cycle param + cycle bank).
4. Serial → USB MIDI (same logic, cleaner integration with `midi_runtime.py`).

## Key files
- `ui/tabs/testing_tab.py` (Serial panel, Arduino connection + mapping)
- `engine/midi_runtime.py` (USB MIDI path, same param targets)
- `docs/obsidian/04_engine/midi_runtime.md`
