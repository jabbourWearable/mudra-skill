# Mudra XR Skill — Build Rules

This document is the canonical reference for the `mudra-xr` Claude Code skill.
Read it in full before generating any app. Every rule below is non-negotiable
unless explicitly marked as configurable.

---

## Section 1 — Mudra Protocol

### WebSocket endpoint

```
ws://127.0.0.1:8766
```

Always construct the connection through `MudraClient` (Section 4).
Never use raw `new WebSocket(...)`.

### Nine canonical signals

| Signal | Category | Description |
|--------|----------|-------------|
| `gesture` | Discrete | Hand gesture events (tap, double_tap, twist, double_twist) |
| `button` | Discrete | Button hold / release |
| `pressure` | Analog | Finger pressure 0–100, normalized 0–1 |
| `navigation` | Motion (Pointer) | Continuous delta_x / delta_y cursor movement |
| `nav_direction` | Motion (Direction) | Discrete directional swipes: None, Right, Left, Up, Down, Roll Left, Roll Right |
| `imu_acc` | Motion (IMU) | Accelerometer values [x, y, z] m/s², frequency 1125 Hz |
| `imu_gyro` | Motion (IMU) | Gyroscope values [x, y, z] deg/s, frequency 1125 Hz |
| `snc` | Biometric | 3 de-interleaved channel arrays [[ch1], [ch2], [ch3]] |
| `battery` | Status | Battery level 0–100, charging boolean |

### Subscription handshake

Send one command per signal — never use plural `signals`, arrays, or batch commands:

```js
// CORRECT
ws.send(JSON.stringify({ command: 'subscribe', signal: 'gesture' }));
ws.send(JSON.stringify({ command: 'subscribe', signal: 'pressure' }));

// WRONG — never do this
ws.send(JSON.stringify({ command: 'subscribe', signals: ['gesture', 'pressure'] }));
ws.send(JSON.stringify({ command: 'subscribe', signal: ['gesture', 'pressure'] }));
```

### Full command surface

`subscribe`, `unsubscribe`, `get_subscriptions`, `enable`, `disable`,
`get_status`, `get_docs`, `trigger_gesture`

### Inbound message payload shapes

```js
// gesture
{ type: 'gesture', data: { type: 'tap'|'double_tap'|'twist'|'double_twist', confidence: 0–1, timestamp }, timestamp }

// button
{ type: 'button', data: { state: 'pressed'|'released', timestamp }, timestamp }

// pressure
{ type: 'pressure', data: { value: 0–100, normalized: 0–1, timestamp }, timestamp }

// navigation
{ type: 'navigation', data: { delta_x: number, delta_y: number, timestamp }, timestamp }

// nav_direction
{ type: 'nav_direction', data: { direction: 'Right'|'Left'|'Up'|'Down'|'Roll Left'|'Roll Right'|'None', timestamp }, timestamp }

// imu_acc
{ type: 'imu_acc', data: { values: [x, y, z], frequency: 1125, timestamp }, timestamp }

// imu_gyro
{ type: 'imu_gyro', data: { values: [x, y, z], frequency: 1125, timestamp }, timestamp }

// snc  — extend rolling buffers (500 samples/channel) with all samples per callback
{ type: 'snc', data: { values: [[ch1_samples], [ch2_samples], [ch3_samples]], timestamp }, timestamp }

// battery
{ type: 'battery', data: { level: 0–100, charging: boolean, timestamp }, timestamp }

// connection_status
{ type: 'connection_status', data: { status: 'connected'|'disconnected', message: string }, timestamp }
```

---

## Section 2 — Canonical Dependency Pins

Use the exact versions below. Never use `@latest` or version ranges.
Import map `<script type="importmap">` must contain **only** the dependencies
the app actually uses — no unused entries.

```json
{
  "imports": {
    "three": "https://cdn.jsdelivr.net/npm/three@0.182.0/build/three.module.js",
    "three/addons/": "https://cdn.jsdelivr.net/npm/three@0.182.0/examples/jsm/",
    "troika-three-text": "https://cdn.jsdelivr.net/gh/protectwise/troika@028b81cf308f0f22e5aa8e78196be56ec1997af5/packages/troika-three-text/src/index.js",
    "troika-three-utils": "https://cdn.jsdelivr.net/gh/protectwise/troika@v0.52.4/packages/troika-three-utils/src/index.js",
    "troika-worker-utils": "https://cdn.jsdelivr.net/gh/protectwise/troika@v0.52.4/packages/troika-worker-utils/src/index.js",
    "bidi-js": "https://esm.sh/bidi-js@%5E1.0.2?target=es2022",
    "webgl-sdf-generator": "https://esm.sh/webgl-sdf-generator@1.1.1/es2022/webgl-sdf-generator.mjs",
    "lit": "https://cdn.jsdelivr.net/gh/lit/dist@3/core/lit-core.min.js",
    "lit/": "https://esm.run/lit@3/",
    "xrblocks": "https://cdn.jsdelivr.net/npm/xrblocks@0.11.0/build/xrblocks.js",
    "xrblocks/addons/": "https://cdn.jsdelivr.net/npm/xrblocks@0.11.0/build/addons/"
  }
}
```

XR Blocks stylesheet (always `<link>` in `<head>`):

```html
<link type="text/css" rel="stylesheet" href="https://xrblocks.github.io/css/xr.css" />
```

Lit bundle warning suppression (place before import map):

```html
<script>window.litDisableBundleWarning = true;</script>
```

---

## Section 3 — XR Blocks Lifecycle Composition

### Module-scope MudraClient

Instantiate `MudraClient` exactly once at module scope — not inside `init()`.
This ensures the WebSocket open-attempt starts immediately on page load so the
1500 ms timeout (Section 4) begins counting before the XR scene initializes.

```js
// Module scope — outside any class
const mudra = new MudraClient('ws://127.0.0.1:8766');

class MainScript extends xb.Script {
  init() {
    // Wire mudra handlers here after scene objects are created
    mudra.on('gesture', (data) => { ... });
    mudra.on('pressure', (data) => { ... });
  }
  update() { /* called every frame by xb */ }
}
```

### xb.Script class structure

Every generated app's top-level logic lives inside a class extending `xb.Script`.
The class name may be anything (`MainScript`, `CubeApp`, etc.) but must extend `xb.Script`.

```js
class MainScript extends xb.Script {
  /** Called once after XR Blocks and the WebGL context are ready. */
  init() {
    // Add lights, meshes, Mudra bindings here.
    this.add(new THREE.HemisphereLight(0xffffff, 0x666666, 3));
    // Place objects at xb.user.objectDistance in front of viewer:
    this.mesh.position.set(0, xb.user.height - 0.5, -xb.user.objectDistance);
  }

  /** Called every animation frame. Keep cheap — no allocations. */
  update() { }

  /** Fires when a pinch/controller-trigger starts in XR. */
  onSelectStart(event) { }

  /** Fires when a pinch/controller-trigger ends in XR. */
  onSelectEnd(event) { }

  /** Fires every frame while a pinch is held in XR. */
  onSelecting(event) { }
}
```

### Entry point

```js
document.addEventListener('DOMContentLoaded', function () {
  xb.add(new MainScript());
  xb.init(new xb.Options());
});
```

### Key spatial constants

- `xb.user.height` — floor-relative eye height (approx. 1.6 m)
- `xb.user.objectDistance` — recommended arm-length distance for objects (approx. 0.8 m)
- Y-up coordinate system; Z is toward the viewer (negative Z = in front of you)

---

## Section 4 — Mock WebSocket Fallback (MudraClient)

### Policy

- Start the WebSocket open attempt immediately on page load.
- If the WebSocket does not open within **1500 ms**, activate the mock automatically.
- If the WebSocket closes mid-session (band disconnect), flip to mock without a page reload.
- The mock fires exactly the same message format as the real device.
- App code must not branch on `_useMock` — it must receive the same messages either way.

### Connection-status state machine

```
[page load]
    │
    ▼
connecting ──── ws opens within 1500 ms ──→ connected
    │
    └── timeout or error ──────────────────→ simulated
                                                  │
connected ──── ws.onclose fires ───────────────→ disconnected-simulated
simulated ──── ws.onclose fires ───────────────→ (already simulated, no change)
```

### Required MudraClient implementation

Include this class verbatim in every generated app:

```js
class MudraClient {
  constructor(url) {
    this._handlers = {};
    this._subscriptions = new Set();
    this._timers = [];
    this._status = 'connecting';
    this._notifyStatus('connecting');

    const timeout = setTimeout(() => this._startMock(), 1500);

    try {
      this._ws = new WebSocket(url);
      this._ws.onopen = () => {
        clearTimeout(timeout);
        this._status = 'connected';
        this._notifyStatus('connected');
        this._subscriptions.forEach(sig =>
          this._ws.send(JSON.stringify({ command: 'subscribe', signal: sig }))
        );
      };
      this._ws.onmessage = (e) => {
        const msg = JSON.parse(e.data);
        if (this._handlers[msg.type]) this._handlers[msg.type](msg.data);
      };
      this._ws.onclose = () => {
        clearTimeout(timeout);
        if (this._status === 'connected') {
          this._status = 'disconnected-simulated';
          this._notifyStatus('disconnected-simulated');
          this._startMock();
        }
      };
      this._ws.onerror = () => {
        clearTimeout(timeout);
        this._startMock();
      };
    } catch (_) {
      clearTimeout(timeout);
      this._startMock();
    }
  }

  /** Register a handler for a signal type. Call before subscribe(). */
  on(signal, fn) {
    this._handlers[signal] = fn;
  }

  /** Subscribe to a signal. Safe to call before the WebSocket opens. */
  subscribe(signal) {
    this._subscriptions.add(signal);
    if (this._ws && this._ws.readyState === WebSocket.OPEN) {
      this._ws.send(JSON.stringify({ command: 'subscribe', signal }));
    }
  }

  /** Send an arbitrary command to the band service. */
  send(cmd) {
    if (this._ws && this._ws.readyState === WebSocket.OPEN) {
      this._ws.send(JSON.stringify(cmd));
    } else if (cmd.command === 'trigger_gesture') {
      this._dispatchMockGesture(cmd.data.type);
    }
  }

  /** Current connection status string. */
  get status() { return this._status; }

  _notifyStatus(s) {
    if (this._handlers['_status']) this._handlers['_status'](s);
  }

  _emit(payload) {
    if (this._handlers[payload.type]) this._handlers[payload.type](payload.data);
  }

  _dispatchMockGesture(type) {
    this._emit({ type: 'gesture', data: { type, confidence: 0.99, timestamp: Date.now() } });
  }

  _startMock() {
    if (this._status === 'simulated' || this._status === 'disconnected-simulated') return;
    const wasPreviouslyConnected = this._status === 'connected';
    this._status = wasPreviouslyConnected ? 'disconnected-simulated' : 'simulated';
    this._notifyStatus(this._status);
    // Passive mock: no auto-firing. Signals fire ONLY from sim-panel clicks,
    // keyboard shortcuts, or real WebSocket messages. This keeps the scene
    // stable and makes the sim panel the single, explicit source of motion.
  }

  destroy() {
    this._timers.forEach(t => clearInterval(t));
    if (this._ws) this._ws.close();
  }
}
```

---

## Section 5 — Simulator Panel

### Purpose

The simulator panel lets a user exercise every subscribed Mudra signal without
a band or XR headset. It is a **2D DOM overlay** (not spatial XR UI) and must
always be visible in flat-screen mode. It disappears automatically when the
browser transitions to an immersive WebXR session (native WebXR DOM suppression).

### DOM structure

```html
<div id="mudra-sim" style="
  position: fixed; bottom: 0; left: 0; right: 0;
  background: rgba(0,0,0,0.75); backdrop-filter: blur(4px);
  padding: 8px 12px; display: flex; flex-wrap: wrap; gap: 8px;
  z-index: 9999; font-family: system-ui, sans-serif;">
  <!-- One button group per subscribed signal -->
</div>
```

### Button groups per signal (render only subscribed signals)

| Signal | Buttons |
|--------|---------|
| `gesture` | `Tap`, `2Tap`, `Twist`, `2Twist` |
| `nav_direction` | `↑`, `↓`, `←`, `→`, `Roll L`, `Roll R` |
| `navigation` | `↑`, `↓`, `←`, `→` (each emits one delta event of ±8) |
| `pressure` | Slider `0–100` (label shows current value) |
| `button` | `Press`, `Release` |
| `imu_acc` | `Tilt X+`, `Tilt X−`, `Tilt Y+`, `Tilt Y−` (5-frame burst at ±2 m/s²) |
| `imu_gyro` | `Rot X+`, `Rot X−`, `Rot Y+`, `Rot Y−` (5-frame burst at ±10 deg/s) |
| `snc` | `Spike` (burst of elevated samples on all 3 channels) |

### Button firing rules

- Every button fires via the **same code path** as a real Mudra signal.
  Call the same handler that `mudra.on(signal, handler)` would invoke.
- For `gesture` buttons: also call `mudra.send({ command: 'trigger_gesture', data: { type } })`
  so the round-trip works when a real band is connected.
- Never duplicate logic between the simulator path and the real-signal path.

```js
// Example: gesture sim button wires to the same handler as real signals
function simGesture(type) {
  mudra.send({ command: 'trigger_gesture', data: { type } });
  handleGesture({ type, confidence: 1.0, timestamp: Date.now() });
}

// Example: pressure slider
pressureSlider.addEventListener('input', () => {
  const norm = pressureSlider.value / 100;
  handlePressure({ value: +pressureSlider.value, normalized: norm, timestamp: Date.now() });
});
```

### Visibility

- `#mudra-sim` is always visible in flat-screen mode.
- Do NOT conditionally hide it when the band is connected (both sources are valid per FR-014).
- The 2D DOM disappears automatically in immersive WebXR — no JS required.

---

## Section 6 — Keyboard Shortcuts + XR Blocks Precedence

### Canonical keyboard map

| Key | Signal / Action |
|-----|----------------|
| `Space` | `gesture` → tap |
| `Shift` | `button` → press (keydown) / release (keyup) |
| `[` | `pressure` decrease (−10, min 0) |
| `]` | `pressure` increase (+10, max 100) |
| `ArrowUp` | `nav_direction` → Up OR `navigation` delta_y +8 |
| `ArrowDown` | `nav_direction` → Down OR `navigation` delta_y −8 |
| `ArrowLeft` | `nav_direction` → Left OR `navigation` delta_x −8 |
| `ArrowRight` | `nav_direction` → Right OR `navigation` delta_x +8 |
| `q` | `imu_acc` tilt X+ burst |
| `e` | `imu_acc` tilt X− burst |
| `r` | `imu_gyro` rot Y+ burst |
| `f` | `imu_gyro` rot Y− burst |

For `imu` bursts: fire 5 synthetic frames at ±[2, 0, 9.81] m/s² (acc) or ±[10, 0, 0.5] deg/s (gyro).

### Attachment rule (critical)

Mudra keyboard handlers MUST attach with `capture: true` and call
`event.stopPropagation()` on every key the app subscribes to. This prevents
XR Blocks' desktop-simulator bubble-phase listeners from double-firing.

```js
window.addEventListener('keydown', (e) => {
  switch (e.code) {
    case 'Space':
      e.stopPropagation();
      simGesture('tap');
      break;
    case 'BracketLeft':
      e.stopPropagation();
      adjustPressure(-10);
      break;
    case 'BracketRight':
      e.stopPropagation();
      adjustPressure(+10);
      break;
    case 'ArrowUp':
      e.stopPropagation();
      handleNavDirection({ direction: 'Up', timestamp: Date.now() });
      break;
    // ... etc for subscribed keys only
  }
}, { capture: true });
```

Only intercept keys for signals the app actually subscribes to. Do NOT
`stopPropagation` on keys that no Mudra signal handles — XR Blocks needs those.

---

## Section 7 — Connection-Status Indicator

### Required DOM element

Every generated app must include exactly one visible element that reflects
the current `MudraClient` status:

```html
<div id="mudra-status" style="
  position: fixed; top: 8px; right: 12px;
  padding: 4px 10px; border-radius: 999px;
  font-size: 0.8rem; font-family: system-ui, sans-serif;
  background: rgba(0,0,0,0.6); color: #fff;
  z-index: 9999;">Connecting…</div>
```

### Text states

| MudraClient status | textContent |
|-------------------|-------------|
| `connecting` | `Connecting…` |
| `connected` | `Connected` |
| `simulated` | `Simulated` |
| `disconnected-simulated` | `Disconnected — simulated` |

### Wiring

```js
mudra.on('_status', (s) => {
  const chip = document.getElementById('mudra-status');
  const labels = {
    'connecting': 'Connecting…',
    'connected': 'Connected',
    'simulated': 'Simulated',
    'disconnected-simulated': 'Disconnected — simulated',
  };
  chip.textContent = labels[s] ?? s;
});
```

### Visibility

- Visible in flat-screen mode at all times.
- Disappears automatically in immersive XR (native DOM suppression).
- Place in the top-right corner by default; adapt if the template uses that space.

---

## Section 8 — Motion-Mode Exclusivity (Non-Negotiable)

### Signal-to-mode classification

| Mode | Signals owned (mutually exclusive) | Signals that combine freely |
|------|-----------------------------------|-----------------------------|
| **Pointer** | `navigation`, `button` | `gesture`, `pressure`, `snc`, `battery` |
| **Direction** | `nav_direction` | `gesture`, `pressure`, `snc`, `battery` |
| **IMU** | `imu_acc`, `imu_gyro` | `gesture`, `pressure`, `snc`, `battery` |
| *(none)* | — | `gesture`, `pressure`, `snc`, `battery` |

### XOR rule

An app MUST subscribe to **at most one** motion mode. Mixing is a
constitutional violation (Principle III) and triggers a pre-write failure.

### Illegal combinations

```
// REJECT these signal sets
navigation + imu_acc
navigation + imu_gyro
navigation + nav_direction
nav_direction + imu_acc
nav_direction + imu_gyro
button + nav_direction        // button belongs to Pointer mode only
```

### Inference priority for ties

When the user prompt maps to multiple modes equally, ask one disambiguation
question — do not silently pick one.

Default to `direction` mode (`nav_direction`) when there is no navigation
language in the prompt at all and a motion mode is required by the template.

---

## Section 9 — AI API Key Handling

### Default lifecycle: in-memory per-request

The API key lives in a local `let` variable, populated once via an in-UI
dialog prompt on the first AI call. It is never persisted automatically.

```js
let apiKey = null;

async function ensureApiKey() {
  if (!apiKey) {
    apiKey = prompt('Enter your Gemini API key:');
  }
  return apiKey;
}
```

### Opt-in persistence lifecycles

| Lifecycle | Storage | How to opt in |
|-----------|---------|---------------|
| Per-request (default) | JS variable, lost on reload | Default — no opt-in needed |
| Session | `sessionStorage` | Only when the generator explicitly chose this |
| Permanent | `localStorage` | Only when the generator explicitly chose this |

### Rules

1. **Never bake a key into the HTML source.** The pre-write regex scan
   (`/sk-[A-Za-z0-9_-]{32,}|AIza[A-Za-z0-9_-]{35}/`) must return zero matches.
2. Always prompt the user at the time of the first AI call — not on page load.
3. Use a `<dialog>` element or `prompt()` — never auto-read from URL params.
4. If using `sessionStorage` / `localStorage`, document the lifecycle choice
   prominently in the HTML's `<title>` or a visible UI element.

---

## Section 10 — Pre-Write Checklist + Collision Handling

Before calling `Write` to emit a generated app, verify all nine items:

| # | Check | Pass condition |
|---|-------|----------------|
| 1 | Single file | Exactly one `<html>` document; all CSS in `<style>`; all JS in `<script>` or `<script type="module">` |
| 2 | Import map | One `<script type="importmap">` block; contents match canonical pins (Section 2) exactly; no unused entries |
| 3 | xb.Script entry | Top-level logic inside `class <Name> extends xb.Script`; `xb.add(new <Name>())` + `xb.init(new xb.Options())` on `DOMContentLoaded` |
| 4 | MudraClient | One `MudraClient` instance at module scope; URL = `ws://127.0.0.1:8766`; 1500 ms timeout wired |
| 5 | Subscribe commands | Every used signal has exactly one `mudra.subscribe('<signal>')` call; none outside the signal set |
| 6 | Simulator panel | `<div id="mudra-sim">` present; one button group per subscribed signal; buttons fire via handler, not inline `onclick` |
| 7 | Keyboard bindings | `window.addEventListener('keydown', …, { capture: true })` present; `event.stopPropagation()` on every Mudra-claimed key |
| 8 | Status indicator | `<div id="mudra-status">` present; wired to `mudra.on('_status', …)` |
| 9 | AI-key safety | If `usesAI`: zero API-key strings in source (run regex scan); key obtained via in-UI dialog; lifecycle matches plan |
| 10 | Background | Exactly one `applyBackground_<id>()` method in the class; called as the first line of `init()`; id matches one of the five catalog rows (Section 14) |

### Retry policy

If any check fails:
1. Regenerate once and re-run the full checklist.
2. If the second attempt also fails, surface the specific failing check(s)
   to the user and do **not** write the file.

### File collision / auto-suffix rule

- Target filename: `preview/<name>.html`
- If that file already exists, try `preview/<name>-2.html`, then `-3.html`, etc.
- Never overwrite an existing file.
- Always report the final written path to the user.

---

## Section 11 — Signal → XR Binding Patterns

Use these as code snippets when adapting a template.
Copy and adapt — do not invent new patterns from scratch.

### gesture.tap → onSelectStart forwarding

```js
// In init():
mudra.on('gesture', (data) => {
  if (data.type === 'tap') this.onSelectStart({ source: 'mudra' });
  if (data.type === 'double_tap') this.onSelectEnd({ source: 'mudra' });
});
mudra.subscribe('gesture');
```

### pressure → scale / color mapping

```js
// In init():
mudra.on('pressure', (data) => {
  const s = 0.5 + data.normalized * 1.5;           // scale 0.5 … 2.0
  this.mesh.scale.setScalar(s);
  const hue = data.normalized * 0.8;               // hue 0 (red) … 0.8 (blue)
  this.mesh.material.color.setHSL(hue, 0.9, 0.5);
});
mudra.subscribe('pressure');
```

### imu_acc / imu_gyro → camera / object orientation

```js
// In init():
mudra.on('imu_acc', (data) => {
  const [ax, ay] = data.values;
  this.mesh.rotation.x = THREE.MathUtils.clamp(ax * 0.1, -Math.PI / 4, Math.PI / 4);
  this.mesh.rotation.z = THREE.MathUtils.clamp(ay * 0.1, -Math.PI / 4, Math.PI / 4);
});
mudra.subscribe('imu_acc');
```

### nav_direction → spatial menu step

```js
// In init():
mudra.on('nav_direction', (data) => {
  switch (data.direction) {
    case 'Up':    this.menuIndex = Math.max(0, this.menuIndex - 1); break;
    case 'Down':  this.menuIndex = Math.min(this.items.length - 1, this.menuIndex + 1); break;
    case 'Right': this.selectItem(this.menuIndex); break;
    case 'Left':  this.goBack(); break;
  }
  this.updateMenuHighlight();
});
mudra.subscribe('nav_direction');
```

### snc → visual feedback overlay

```js
// In init():
mudra.on('snc', (data) => {
  const ch1 = data.values[0];
  const latest = ch1[ch1.length - 1];               // most recent sample
  const norm = Math.min(1, Math.abs(latest) / 500); // normalize
  this.overlay.material.opacity = norm * 0.7;
});
mudra.subscribe('snc');
```

### navigation → continuous cursor / pan

```js
// In init():
mudra.on('navigation', (data) => {
  this.cursorX = THREE.MathUtils.clamp(this.cursorX + data.delta_x * 0.005, -1, 1);
  this.cursorY = THREE.MathUtils.clamp(this.cursorY - data.delta_y * 0.005, -1, 1);
  this.cursor.position.set(this.cursorX, this.cursorY, -xb.user.objectDistance);
});
mudra.subscribe('navigation');
```

---

## Section 12 — Template Selection Table

One row per asset in `assets/`. The skill's keyword-based selector scores each row
against the user's prompt (signal names, motion keywords, XR feature words).

| id | path | keywords | motionModesSupported | xrFeatures |
|----|------|----------|---------------------|------------|
| `0_basic` | `assets/templates/0_basic.html` | `["basic","cylinder","pinch","color","simple","starter"]` | `["none","pointer"]` | `["input"]` |
| `1_ui` | `assets/templates/1_ui.html` | `["ui","spatial","panel","text","sdf","font","button","draggable","troika"]` | `["pointer"]` | `["input","ui"]` |
| `2_hands` | `assets/templates/2_hands.html` | `["hands","hand","pinch","gesture","joints","finger","hand-tracking"]` | `["pointer"]` | `["hands","input"]` |
| `3_depth` | `assets/templates/3_depth.html` | `["depth","mesh","depth-sensing","plane","environment","occlusion"]` | `["none"]` | `["depth-sensing","mesh-detection"]` |
| `4_stereo` | `assets/templates/4_stereo.html` | `["stereo","video","passthrough","camera","background","environment","feed"]` | `["none"]` | `["camera","passthrough"]` |
| `5_camera` | `assets/templates/5_camera.html` | `["camera","video","passthrough","texture","scene","background"]` | `["none","pointer"]` | `["camera","input"]` |
| `6_ai` | `assets/templates/6_ai.html` | `["ai","gemini","query","vision","photo","capture","llm","multimodal"]` | `["pointer"]` | `["input","camera"]` |
| `7_ai_live` | `assets/templates/7_ai_live.html` | `["ai","gemini","live","speech","transcription","voice","microphone","audio","real-time"]` | `["pointer"]` | `["input"]` |
| `8_objects` | `assets/templates/8_objects.html` | `["objects","detection","model","3d","place","anchor","ar","environment"]` | `["pointer"]` | `["mesh-detection","input"]` |
| `9_xr-toggle` | `assets/templates/9_xr-toggle.html` | `["toggle","xr","enter","exit","session","button","transition"]` | `["none","pointer"]` | `["input"]` |
| `heuristic_hand_gestures` | `assets/templates/heuristic_hand_gestures.html` | `["gesture","heuristic","hand","recognition","custom","pattern","hand-tracking"]` | `["none"]` | `["hands"]` |
| `meshes` | `assets/templates/meshes.html` | `["mesh","environment","scan","plane","floor","wall","surface"]` | `["none"]` | `["mesh-detection"]` |
| `planes` | `assets/templates/planes.html` | `["plane","floor","wall","surface","anchor","environment","horizontal","vertical"]` | `["none"]` | `["plane-detection"]` |
| `uikit` | `assets/templates/uikit.html` | `["ui","kit","component","widget","button","icon","material","text","panel","menu"]` | `["pointer"]` | `["input","ui"]` |
| `depthmap` | `assets/samples/depthmap.html` | `["depth","map","visualization","color","gradient","environment","scan"]` | `["none"]` | `["depth-sensing"]` |
| `depthmesh` | `assets/samples/depthmesh.html` | `["depth","mesh","wireframe","environment","scan","geometry"]` | `["none"]` | `["depth-sensing","mesh-detection"]` |
| `game_rps` | `assets/samples/game_rps.html` | `["game","rps","rock","paper","scissors","gesture","compete","fun","hand"]` | `["none"]` | `["hands"]` |
| `gestures_custom` | `assets/samples/gestures_custom.html` | `["gesture","custom","recognize","train","pose","hand-tracking","hands"]` | `["none"]` | `["hands"]` |
| `gestures_heuristic` | `assets/samples/gestures_heuristic.html` | `["gesture","heuristic","recognize","hand","pose","pinch","open","fist"]` | `["none"]` | `["hands"]` |
| `lighting` | `assets/samples/lighting.html` | `["lighting","light","shadow","scene","animals","3d","models","environment"]` | `["pointer"]` | `["input"]` |
| `mesh_detection` | `assets/samples/mesh_detection.html` | `["mesh","detection","environment","scan","ar","surface"]` | `["none"]` | `["mesh-detection"]` |
| `modelviewer` | `assets/samples/modelviewer.html` | `["model","viewer","3d","gltf","glb","object","rotate","inspect","load"]` | `["pointer"]` | `["input"]` |
| `paint` | `assets/samples/paint.html` | `["paint","draw","brush","stroke","canvas","art","color","gesture"]` | `["pointer"]` | `["input","hands"]` |
| `planar-vst` | `assets/samples/planar-vst.html` | `["plane","vst","passthrough","video","portal","ar","surface"]` | `["none"]` | `["plane-detection","camera"]` |
| `reticle` | `assets/samples/reticle.html` | `["reticle","cursor","pointer","aim","gaze","target","floor","placement"]` | `["none","pointer"]` | `["plane-detection","input"]` |
| `skybox_agent` | `assets/samples/skybox_agent.html` | `["skybox","sky","background","ai","gemini","generate","environment","image"]` | `["pointer"]` | `["input"]` |
| `sound` | `assets/samples/sound.html` | `["sound","audio","music","spatial","3d-audio","positional","play"]` | `["pointer"]` | `["input"]` |
| `ui` | `assets/samples/ui.html` | `["ui","panel","button","menu","list","text","interface","spatial"]` | `["pointer"]` | `["input","ui"]` |
| `virtual-screens` | `assets/samples/virtual-screens.html` | `["screen","virtual","window","share","stream","desktop","browser","display"]` | `["pointer"]` | `["input"]` |
| `3dgs-walkthrough` | `assets/demos/3dgs-walkthrough.html` | `["gaussian","splat","3dgs","scene","walkthrough","room","photo","realistic"]` | `["imu","direction"]` | `["input"]` |
| `aisimulator` | `assets/demos/aisimulator.html` | `["ai","simulate","gemini","agent","roleplay","character","conversation","npc"]` | `["pointer"]` | `["input"]` |
| `balloonpop` | `assets/demos/balloonpop.html` | `["balloon","pop","game","gesture","fun","pinch","particle","explosion"]` | `["none"]` | `["hands","input"]` |
| `ballpit` | `assets/demos/ballpit.html` | `["ball","physics","pit","throw","gravity","interact","fun","ammo"]` | `["pointer"]` | `["input","hands"]` |
| `drone` | `assets/demos/drone.html` | `["drone","fly","navigate","control","imu","tilt","direction","pilot"]` | `["imu","direction"]` | `["input"]` |
| `gemini-icebreakers` | `assets/demos/gemini-icebreakers.html` | `["gemini","icebreaker","question","conversation","ai","social","fun"]` | `["pointer"]` | `["input"]` |
| `gemini-xrobject` | `assets/demos/gemini-xrobject.html` | `["gemini","object","recognize","camera","ar","label","identify","vision"]` | `["pointer"]` | `["input","camera"]` |
| `math3d` | `assets/demos/math3d.html` | `["math","3d","formula","graph","equation","plot","visualization","education"]` | `["pointer"]` | `["input","ui"]` |
| `measure` | `assets/demos/measure.html` | `["measure","ruler","tape","distance","ar","depth","spatial","length"]` | `["none"]` | `["depth-sensing","mesh-detection"]` |
| `occlusion` | `assets/demos/occlusion.html` | `["occlusion","depth","animal","model","shadow","ar","realistic","environment"]` | `["pointer"]` | `["input","depth-sensing"]` |
| `rain` | `assets/demos/rain.html` | `["rain","particle","weather","atmosphere","shader","visual","instanced"]` | `["none","pointer"]` | `["input"]` |
| `screenwiper` | `assets/demos/screenwiper.html` | `["wiper","screen","wipe","clear","gesture","brush","clean","effect"]` | `["pointer"]` | `["input","hands"]` |
| `splash` | `assets/demos/splash.html` | `["splash","paint","decal","physics","shoot","color","ball","impact"]` | `["pointer"]` | `["input"]` |
| `webcam_gestures` | `assets/demos/webcam_gestures.html` | `["webcam","mediapipe","gesture","hand","camera","tracking","no-headset","flat"]` | `["none"]` | `[]` |
| `xremoji` | `assets/demos/xremoji.html` | `["emoji","expression","face","gesture","fun","balloon","tensorflow","social"]` | `["none"]` | `["hands"]` |
| `xrpoet` | `assets/demos/xrpoet.html` | `["poem","poetry","ai","gemini","generate","creative","camera","writing"]` | `["pointer"]` | `["input","camera"]` |

---

## Section 13 — Template Inference + Override

*(Populated after Phase 5 / T032.)*

---

## Section 14 — Background Catalog

Every generated XR app must call exactly **one** background helper from the
catalog below inside `init()`, before adding any other scene content. The
helper is a method on the `xb.Script` subclass — copy its body into the class
and call it from `init()` first thing.

### Selection algorithm

Score every row against the user's prompt:

1. `+2` per keyword from the background's `keywords` that appears in the lowercased prompt.
2. `+1` if the background's `fits_signals` overlaps the inferred signal set.
3. `+1` if the background's `pairs_with_templates` contains the chosen template id.

Tie-break order: `solid_studio` (safest, UI-friendly) → `starfield` → `gradient_sky` → `grid_cyber` → `skybox_texture`.

If no background scores above 0, default to `solid_studio`.

### Catalog

| id | keywords | fits_signals | pairs_with_templates | use_case |
|----|----------|--------------|----------------------|----------|
| `starfield` | `["space","star","galaxy","cosmos","universe","planet","solar","nebula","night","astronomy","orbit"]` | any | `0_basic`, `8_objects`, `lighting` | Deep-space / astronomy scenes |
| `gradient_sky` | `["sky","gradient","sunset","sunrise","horizon","dawn","dusk","pastel","dreamy","ethereal","atmosphere","weather"]` | any | `0_basic`, `rain`, `lighting` | Open-air, emotive, atmospheric |
| `solid_studio` | `["menu","ui","panel","studio","minimal","clean","product","showcase","interface","card","dashboard"]` | any | `1_ui`, `uikit`, `ui`, `virtual-screens` | UI panels, product demos, clean showcases |
| `grid_cyber` | `["cyber","grid","matrix","retro","synthwave","vapor","tron","neon","arcade","game","futuristic","sci-fi"]` | any | `0_basic`, `ballpit`, `drone`, `balloonpop` | Synthwave / cyberpunk / game scenes |
| `skybox_texture` | `["outdoor","forest","mountain","beach","desert","photo","photographic","immersive","panorama","skybox","environment","real-world"]` | any | `0_basic`, `3_depth`, `8_objects` | Photoreal / immersive outdoor |

### Drop-in snippets

All five helpers follow the same signature: `applyBackground_<id>()`. They create
a sky dome / points / grid / ground and add it via `this.add(...)`. None of them
touch `scene.background` or `renderer.setClearColor` — that path requires `xb.core`
access which may not be stable across XR Blocks versions.

```js
// ── 1. Starfield ──────────────────────────────────────────────────────────
applyBackground_starfield() {
  const starGeom = new THREE.BufferGeometry();
  const N = 2000;
  const positions = new Float32Array(N * 3);
  for (let i = 0; i < N; i++) {
    const r = 40 + Math.random() * 40;
    const theta = Math.random() * Math.PI * 2;
    const phi = Math.acos(2 * Math.random() - 1);
    positions[i*3]   = r * Math.sin(phi) * Math.cos(theta);
    positions[i*3+1] = r * Math.sin(phi) * Math.sin(theta);
    positions[i*3+2] = r * Math.cos(phi);
  }
  starGeom.setAttribute('position', new THREE.BufferAttribute(positions, 3));
  const starMat = new THREE.PointsMaterial({ color: 0xffffff, size: 0.15, sizeAttenuation: true });
  this.add(new THREE.Points(starGeom, starMat));

  // Dim black dome so the default XR Blocks background doesn't show through
  const domeGeom = new THREE.SphereGeometry(90, 32, 16);
  const domeMat = new THREE.MeshBasicMaterial({ color: 0x000008, side: THREE.BackSide });
  this.add(new THREE.Mesh(domeGeom, domeMat));
}

// ── 2. Gradient Sky ───────────────────────────────────────────────────────
applyBackground_gradient_sky() {
  const skyGeom = new THREE.SphereGeometry(80, 32, 16);
  const skyMat = new THREE.ShaderMaterial({
    uniforms: {
      topColor:    { value: new THREE.Color(0x0077ff) },
      bottomColor: { value: new THREE.Color(0xffaa44) },
    },
    vertexShader: `
      varying vec3 vWorldPosition;
      void main() {
        vec4 wp = modelMatrix * vec4(position, 1.0);
        vWorldPosition = wp.xyz;
        gl_Position = projectionMatrix * viewMatrix * wp;
      }`,
    fragmentShader: `
      uniform vec3 topColor;
      uniform vec3 bottomColor;
      varying vec3 vWorldPosition;
      void main() {
        float h = normalize(vWorldPosition).y;
        gl_FragColor = vec4(mix(bottomColor, topColor, smoothstep(-0.2, 0.8, h)), 1.0);
      }`,
    side: THREE.BackSide,
    depthWrite: false,
  });
  this.add(new THREE.Mesh(skyGeom, skyMat));
}

// ── 3. Solid Studio ───────────────────────────────────────────────────────
applyBackground_solid_studio() {
  // Dark neutral dome
  const domeGeom = new THREE.SphereGeometry(80, 32, 16);
  const domeMat = new THREE.MeshBasicMaterial({ color: 0x1a1a22, side: THREE.BackSide });
  this.add(new THREE.Mesh(domeGeom, domeMat));

  // Subtle floor for grounding
  const floorGeom = new THREE.PlaneGeometry(10, 10);
  const floorMat = new THREE.MeshStandardMaterial({ color: 0x2a2a32, roughness: 0.9, metalness: 0.1 });
  const floor = new THREE.Mesh(floorGeom, floorMat);
  floor.rotation.x = -Math.PI / 2;
  floor.position.y = 0;
  this.add(floor);
}

// ── 4. Grid / Cyber ───────────────────────────────────────────────────────
applyBackground_grid_cyber() {
  // Very dark dome
  const domeGeom = new THREE.SphereGeometry(80, 32, 16);
  const domeMat = new THREE.MeshBasicMaterial({ color: 0x050014, side: THREE.BackSide });
  this.add(new THREE.Mesh(domeGeom, domeMat));

  // Neon magenta/cyan grid on the floor
  const grid = new THREE.GridHelper(40, 40, 0xff00ff, 0x00ffff);
  grid.position.y = 0;
  this.add(grid);

  // Horizon line for added depth
  const horizonGeom = new THREE.RingGeometry(19.8, 20, 64);
  const horizonMat = new THREE.MeshBasicMaterial({ color: 0xff00ff, side: THREE.DoubleSide, transparent: true, opacity: 0.6 });
  const horizon = new THREE.Mesh(horizonGeom, horizonMat);
  horizon.rotation.x = -Math.PI / 2;
  horizon.position.y = 0.01;
  this.add(horizon);
}

// ── 5. Skybox Texture (equirectangular) ───────────────────────────────────
// Default URL is overridable. Any equirectangular JPG/PNG works.
applyBackground_skybox_texture(url = 'https://threejs.org/examples/textures/equirectangular/royal_esplanade_1k.jpg') {
  const skyGeom = new THREE.SphereGeometry(80, 64, 32);
  const skyMat = new THREE.MeshBasicMaterial({ side: THREE.BackSide });
  const sky = new THREE.Mesh(skyGeom, skyMat);
  this.add(sky);

  new THREE.TextureLoader().load(
    url,
    (tex) => {
      tex.mapping = THREE.EquirectangularReflectionMapping;
      tex.colorSpace = THREE.SRGBColorSpace;
      skyMat.map = tex;
      skyMat.color.set(0xffffff);
      skyMat.needsUpdate = true;
    },
    undefined,
    () => { skyMat.color.set(0x223344); skyMat.needsUpdate = true; }  // fallback on error
  );
}
```

### Wiring into the adapted app

Call the chosen helper as the first line of `init()`:

```js
class MainScript extends xb.Script {
  init() {
    this.applyBackground_starfield();           // <-- exactly one background call
    this.add(new THREE.HemisphereLight(0xffffff, 0x444444, 2));
    // ... rest of scene setup
  }

  applyBackground_starfield() { /* body copied from catalog */ }
}
```

### Rules

1. Include **exactly one** `applyBackground_*` method in the class.
2. Call it from the **first line** of `init()`, before adding lights or scene content.
3. Don't mix two background helpers in one app.
4. Don't modify the helper bodies — copy verbatim. Tweaks go in the scene code
   that follows the helper call (e.g., add extra lights, change fog, add props).
5. If the user explicitly asks for a background in their prompt (e.g., "on a
   starfield", "sunset sky"), use that one regardless of scoring.
