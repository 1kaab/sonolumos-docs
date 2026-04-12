
## **1. Color wheels = geometry, not decoration**

  

A **color wheel** is a geometric model of color relationships.

  

### **The key idea:**

  

Color is **not linear** — it is **circular + radial**.

  

Most useful models decompose color into:

- **Hue** → _angle_ on a circle
    
- **Saturation** → _radius_ from center
    
- **Brightness / Value** → _height_ or scale
    

  

This is why wheels, cones, and cylinders appear everywhere.

---

## **2. The most useful geometric model for controllers: HSV / HSL**

  

### **HSV (Hue, Saturation, Value)**

  

Think of it as:

```
        Value ↑
              |
              |
     Saturation radius →
          ○────────○
         ○     |    ○
         ○     ●----○  → Hue (angle)
         ○          ○
          ○────────○
```

- **Hue**: angle ∈ [0°, 360°)
    
- **Saturation**: distance from center (0 = gray)
    
- **Value**: brightness (0 = black)
    

  

This maps _perfectly_ to:

- knobs (continuous)
    
- circular motion
    
- Brownian trajectories
    
- parameter selection
    

---

## **3. Why hue is circular (important insight)**

  

Red ≈ 0°

Green ≈ 120°

Blue ≈ 240°

Red again ≈ 360° → 0°

  

So:

- Color **wraps**
    
- Interpolation must be **angular**, not linear
    
- This matches **azimuth**, **phase**, **heading**, **periodic domains**
    

  

That’s not a coincidence.

---

## **4. Can a 4-pin RGB LED reproduce knob colors?**

  

### **Short answer:**

  

**Yes, functionally — with limitations.**

  

### **What a 4-pin RGB LED actually is:**

- 3 LEDs inside (Red, Green, Blue)
    
- One common pin (either **common anode** or **common cathode**)
    

  

You control it with **PWM** on 3 pins:

```
R ∈ [0..255]
G ∈ [0..255]
B ∈ [0..255]
```

That gives you:

- ~16 million combinations (theoretical)
    
- Enough to represent **any knob color state**
    
- Enough for **HSV→RGB mapping**
    

  

⚠️ But:

- Colors are **not perceptually uniform**
    
- Whites are often ugly
    
- Low saturation is hard
    
- Brightness varies per channel
    

  

For controllers: **totally acceptable**

---

## **5. Mapping knob colors to the LED (clean model)**

  

Let’s say:

- Each knob / parameter has a **fixed hue**
    
- Example:
    
    - shape → orange (30°)
        
    - floor → blue (210°)
        
    - inertia → green (120°)
        
    - turn_noise → purple (270°)
        
    

  

Then:

  

### **State model**

```
selected_param → hue angle
knob_value     → saturation or brightness
```

### **LED behavior**

- LED hue = selected knob’s hue
    
- LED brightness = knob value
    
- Saturation = constant (or slightly reduced)
    

  

This creates **muscle-memory color association**.

---

## **6. “Rotate between selected ones in a circular fashion”**

  

Yes — and this is where geometry shines.

  

### **What you’re describing**

- Multiple parameters exist
    
- Each has a hue on the wheel
    
- The LED:
    
    - either **jumps** to the active one
        
    - or **rotates smoothly** along the hue circle
        
    

  

### **Circular interpolation (correct way)**

  

You **must interpolate angles**, not RGB values.

  

Pseudo-logic:

```
current_hue
target_hue

delta = shortest_angle(target - current)
current += delta * smoothing
```

This:

- avoids color flicker
    
- respects circular wrap
    
- feels “alive” instead of digital
    

---

## **7. One LED mirroring many knobs — is it meaningful?**

  

Yes, if you follow these rules:

  

### **Rule 1: One LED = focus indicator**

- It shows **what you are editing now**
    
- Not all parameters at once
    

  

### **Rule 2: Color = semantic identity**

- Color never means “value”
    
- Color means “which domain / parameter”
    

  

### **Rule 3: Motion communicates state**

- Fade → inactive
    
- Pulse → armed
    
- Rotate → browsing modes
    
- Solid → locked selection
    

---

## **8. Why this fits your system** 

## **perfectly**

  

You already have:

- **Domains** (periodic vs clamped)
    
- **Brownian heading**
    
- **Azimuth / phase**
    
- **Knobs controlling modifiers**
    
- **Prediction / uncertainty**
    

  

A color wheel is:

  

> a **visual projection of state space**

  

Your LED is not UI bling — it’s a **state vector projection**.

---

## **9. Practical hardware note (important)**

  

A single 4-pin RGB LED is fine for now, but:

  

### **Later upgrade path:**

- **WS2812 / NeoPixel**
    
    - Built-in PWM + color correction
        
    - Perfect for hue rotation
        
    - One data pin
        
    - Much smoother fades
        
    

  

But don’t start there — your current plan is correct.

---

## **10. Summary (takeaway)**

- Color wheels are **geometry**, not decoration
    
- Hue = angle (periodic domain)
    
- RGB LEDs can represent knob colors well enough
    
- One LED can:
    
    - mirror selected knob color
        
    - rotate between parameters
        
    - communicate focus and mode
        
    
- This aligns naturally with:
    
    - azimuth
        
    - brownian heading
        
    - phase
        
    - periodic domains
        
    

  

  

## **Goal**

- **1 physical knob (potentiometer)**
    
- **1 button** to select _which parameter_ the knob controls
    
- **LED** to indicate current selection - color selected in wheel

- **USB Serial → sl
    

---

## **2) Wiring (exact)**

  

### **Potentiometer (the knob)**

```
Pot pin 1 → 5V
Pot pin 2 (middle) → A0
Pot pin 3 → GND
```

### **Button (mode selector)**

```
Button leg 1 → GND
Button leg 2 → D2
```

→ Use **INPUT_PULLUP** in code (no resistor needed)

  

### **LED (mode indicator)**

```
D9 → 220Ω resistor → LED (+)
LED (–) → GND
```

---

## **3) What the button does (concept)**

  

Each press cycles the knob through parameters:

```
MODE 0 → brown_shape
MODE 1 → brown_floor
MODE 2 → brown_inertia
MODE 3 → brown_turn_noise
```

- LED **off** = modes 0–1
    
- LED **on** = modes 2–3
    
    (or later: blink patterns)
    

---

## **4) Arduino logic (clean + stable)**

  

### **Arduino sketch (ready to flash)**

```
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
  // ---- button handling (debounced) ----
  int btn = digitalRead(BTN_PIN);
  if (btn == LOW && lastBtn == HIGH && millis() - lastDebounce > 200) {
    mode = (mode + 1) % NUM_MODES;
    lastDebounce = millis();
  }
  lastBtn = btn;

  // LED logic
  digitalWrite(LED_PIN, mode >= 2 ? HIGH : LOW);

  // ---- knob read ----
  int raw = analogRead(POT_PIN); // 0..1023
  float norm = raw / 1023.0;

  // send as text (simple & debuggable)
  Serial.print("mode:");
  Serial.print(mode);
  Serial.print(",value:");
  Serial.println(norm, 4);

  delay(15); // ~65 Hz
}
```

---

## **5) What your app receives**

  

Example serial lines:

```
mode:0,value:0.5321
mode:1,value:0.1023
mode:2,value:0.8741
```

---

## **6) App-side mapping (Python logic)**

```
MODE_MAP = {
    0: "brown_shape",
    1: "brown_floor",
    2: "brown_inertia",
    3: "brown_turn_noise",
}

def handle_serial_line(line: str):
    parts = dict(p.split(":") for p in line.strip().split(","))
    mode = int(parts["mode"])
    value = float(parts["value"])

    param = MODE_MAP.get(mode)
    if not param:
        return

    state[param] = value
    bump_motion()
```

---

## **7) Why this design is GOOD**

- ✅ **One knob, infinite parameters**
    
- ✅ No jitter (single ADC)
    
- ✅ No UI jumping (you control when updates apply)
    
- ✅ Easy to extend (add more modes / LEDs later)
    
- ✅ Works over **Serial now**, **MIDI later**
    

---

## **8) Natural next upgrades**

- Replace LED with **RGB LED** → color per parameter
    
- Long-press button → fine/coarse mode
    
- Swap pot → **rotary encoder** later (drop-in conceptually)
    
- Add second button → bank select
    

---

## **Mental model (important)**

  

> **The knob never means “a parameter”**

> It means **“control focus”**

> The button selects _what the knob is currently shaping_

  

This matches your **node / modifier / tap** architecture perfectly.

  

If you want, next I can:

- Convert this to **USB MIDI CC**
    
- Add **value latching** (no jumps when switching mode)
    
- Design a **2-knob + 2-button** “brown pan” controller block
