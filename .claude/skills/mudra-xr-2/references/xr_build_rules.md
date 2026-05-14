# Mudra XR-2 — Build Rules

This document is the canonical reference for the `mudra-xr-2` Claude Code skill. It is referenced from `SKILL.md` by section number.

- **Mudra protocol** — sourced from the vendored `references/agent_protocol.json` (v2.0, byte-identical to the upstream `mudra-plugin/skills/mudra-master/mudra-preview/references/agent_protocol.json`). Where this file restates protocol facts, those restatements are convenience copies — the JSON is canonical.
- **XR Blocks corpus** — sourced from the vendored `references/xrpromt.md` (byte-identical to the repo-root `xrpromt.md`). Seed HTML, dependency pins, and architectural rules come from there.

## Table of Contents

- §1 — Mudra Protocol
- §2 — Canonical Dependency Pins
- §3 — Direction motion mode (`nav_direction`)
- §4 — MudraClient class source
- §5 — Simulator Panel
- §6 — Keyboard Map
- §7 — Reserved Keys (XR Blocks desktop simulator)
- §8 — Pre-Write Checklist
- §9 — Signal Inference Mapping
- §10 — Compatibility Rules + Valid Combinations
- §11 — Creative Proposals
- §12 — Theme + Badge
- §13 — Seed Catalog + Scoring Algorithm
- §14 — Seed Lookup Procedure in vendored `xrpromt.md`
- §15 — Background Lockdown (XR Blocks default only)
- §16 — Import-Map Rules
- §17 — AI-enabled Seed Gating
- §18 — Mode Toggle DOM + Behavior
- §19 — Status Indicator + Band-State Polling
- §20 — Vendoring Provenance
- §21 — Onboarding Overlay (every load) + AI-app extension
- §22 — Visible AI Chat I/O (mandatory when `usesAI`)

---

## §1 — Mudra Protocol

### WebSocket endpoint

```
ws://127.0.0.1:8766
```

Matches `agent_protocol.json.connection_urls.websocket`. Always go through `MudraClient` (§4); never use raw `new WebSocket(...)`.

### Nine canonical signals

| Signal | Category | Description |
|--------|----------|-------------|
| `gesture` | Discrete | Hand gesture events (`tap`, `double_tap`, `twist`, `double_twist`) |
| `button` | Discrete | Air-touch button press / release |
| `pressure` | Analog | Finger pressure 0–100, normalized 0–1 |
| `navigation` | Motion (Pointer) | Continuous `delta_x` / `delta_y` cursor movement |
| `nav_direction` | Motion (Direction) | Discrete directional swipes: `Up`, `Down`, `Left`, `Right`, `Roll Left`, `Roll Right`, `None` (see §3 — payload not in vendored JSON v2.0) |
| `imu_acc` | Motion (IMU+Biometric) | Accelerometer `[x, y, z]` m/s², ~100–1125 Hz |
| `imu_gyro` | Motion (IMU+Biometric) | Gyroscope `[x, y, z]` deg/s, ~100–1125 Hz |
| `snc` | Biometric | EMG values, 3 channels, high frequency (~1000 Hz) |
| `battery` | Status | Battery level 0–100, charging boolean |

Eight of the nine (everything except `nav_direction`) have full payload definitions in `agent_protocol.json.signals.<name>.data_format`. The `nav_direction` payload is documented in §3.

### Subscription handshake

Send one command per signal — never use plural `signals`, arrays, or batch commands. Source: `agent_protocol.json.websocket_api.commands.subscribe.critical_note`.

```js
// CORRECT
ws.send(JSON.stringify({ command: 'subscribe', signal: 'gesture' }));
ws.send(JSON.stringify({ command: 'subscribe', signal: 'pressure' }));

// WRONG — never do this
ws.send(JSON.stringify({ command: 'subscribe', signals: ['gesture', 'pressure'] }));
ws.send(JSON.stringify({ command: 'subscribe', signal: ['gesture', 'pressure'] }));
```

### Full command surface

From `agent_protocol.json.websocket_api.commands`:

`subscribe`, `unsubscribe`, `get_subscriptions`, `enable`, `disable`, `get_status`, `get_docs`, `trigger_gesture`.

Inbound `{type: 'connection_status', data: { status: 'connected'|'disconnected', message }, timestamp}` events are hints only — the authoritative band state comes from `{type: 'status'}` responses to `get_status` (see §19).

### Common mistakes (from `agent_protocol.json.websocket_api.common_mistakes`)

1. **WRONG**: `{command: 'enable', data: {signals: ['gesture', 'pressure']}}` — send separate `subscribe` commands, one per signal.
2. **WRONG**: `{command: 'subscribe', signals: ['gesture', 'pressure']}` — parameter is `signal` (singular).
3. **WRONG**: Forgetting to subscribe — no data received without subscription.

---

## §2 — Canonical Dependency Pins

**Single source of truth: `references/xrpromt.md` §"Reference Map"** (lines 21–32 of the vendored file). The block below is a verbatim copy of that Reference Map. Use these exact URLs for every generated app, regardless of what individual seed templates further down in `xrpromt.md` happen to embed.

**Why this rule exists**: some individual seed templates inside `xrpromt.md` embed `xrblocks@0.11.0` in their own import maps. Those embedded values are stale drift — the Reference Map at the top of the file is the canonical guidance the skill MUST honor. Generated apps that use `xrblocks@0.11.0` with `three@0.182.0` fail at runtime with `Failed to resolve module specifier "three/addons/postprocessing/Pass.js"` because that xrblocks build references a `three/addons/` layout that `three@0.182.0` does not expose. The Reference Map's `xrblocks@0.10.0` is the pairing that works.

```html
<link type="text/css" rel="stylesheet" href="https://xrblocks.github.io/css/xr.css" />
<script>window.litDisableBundleWarning = true;</script>
<script type="importmap">
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
    "xrblocks": "https://cdn.jsdelivr.net/npm/xrblocks@0.10.0/build/xrblocks.js",
    "xrblocks/addons/": "https://cdn.jsdelivr.net/npm/xrblocks@0.10.0/build/addons/"
  }
}
</script>
```

**Rules**:

1. Use `xrblocks@0.10.0` paired with `three@0.182.0`. Never `xrblocks@0.11.0`. Never any other combination.
2. Never use `@latest` or version ranges. Pinned only.
3. When adapting a seed template whose embedded `<script type="importmap">` uses `xrblocks@0.11.0` (drift from the Reference Map), **replace** the import-map block with the canonical one above before writing. Do not preserve the seed's drifted pins.
4. The import map MUST contain only the dependencies the generated app actually `import`s — strip unused entries per §16. Never strip `three/addons/` even if the user code does not directly import from it: xrblocks itself depends on `three/addons/...` transitively, and removing the prefix mapping makes those internal imports fail with `Failed to resolve module specifier "three/addons/..."`. Same applies to `xrblocks/addons/`.
5. Always include the XR Blocks stylesheet `<link>` and the `window.litDisableBundleWarning = true;` shim shown above. They are part of the canonical setup.

**Re-vendoring contract**: if `references/xrpromt.md`'s Reference Map ever changes versions, update this §2 to match in the same commit. Verify by running every quickstart prompt afterward (see §20).

---

## §3 — Direction motion mode (`nav_direction`)

**The vendored `agent_protocol.json` v2.0 references `nav_direction` only in compatibility rules** (under `signal_groups.direction_motion`, `compatibility.bundling_rules`, and `compatibility.cannot_combine`). It does not include a full `data_format`, `signals.nav_direction.data_format`, or `signal_inference.mapping` entry for it. This section fills that gap skill-side without modifying the vendored JSON (spec FR-005a + clarification Q3 option A).

### Payload

```js
{
  type: 'nav_direction',
  data: {
    direction: 'Right' | 'Left' | 'Up' | 'Down' | 'Roll Left' | 'Roll Right' | 'None',
    timestamp: 1234567890
  },
  timestamp: 1234567890
}
```

- `'None'` is the idle / no-motion value — handlers MUST ignore it.
- The six active directions are discrete swipes (not continuous deltas). Compare with `navigation` which emits per-frame `delta_x` / `delta_y`.

### Inference cues — when to pick `nav_direction` over `navigation`

If any of the cues below appear in the prompt, prefer Direction mode:

- `"swipe"`, `"swipe through"`, `"swipe up/down/left/right"`
- `"menu next / prev"`, `"page through"`, `"flip pages"`, `"flip cards"`
- `"throw left"`, `"throw right"` (discrete direction triggers)
- `"step left"`, `"step right"` (discrete steps)
- `"tilt to switch"` (when paired with a small menu, not continuous tilt)
- explicit cardinal direction triggers (`"north"`, `"east"`, `"compass"`)
- `"roll left"`, `"roll right"` (a wrist roll commits a discrete action)

If the prompt is about continuous movement (cursor, panning, steering, scrolling), prefer Pointer mode (`navigation`).

### Best practices

| Direction | Suggested binding |
|-----------|-------------------|
| `Up` / `Down` | menu prev / next, scroll-page, throttle ±1 |
| `Left` / `Right` | tab prev / next, deck left / right, undo / redo |
| `Roll Left` / `Roll Right` | rotate selection, alternate-mode switch, commit / cancel pair |

### Simulator panel + keyboard — contextual rendering

The panel MUST render **only** the directions the app actually handles. If only Up/Down are bound, render two buttons (`↑`, `↓`); arrow keys for unhandled directions are no-ops.

Keyboard binding for Direction mode is the exception to the §7 reserved-key rule: arrow keys MAY be bound when (and only when) `nav_direction` is the motion mode. See §6.

---

## §4 — MudraClient class source

Copy this class **verbatim** into every generated app's `<script type="module">`. It is the only WebSocket wrapper allowed by the skill.

Properties:
- Manual default; no auto-connect; `setMode()`-driven.
- Passive mock: no auto-firing intervals. Synthetic signals come only from the sim panel and keyboard handlers.
- While in Mudra mode, sends `{command:"get_status"}` immediately on `onopen` and every 2000 ms thereafter (per §19).
- Emits `_status` events with the four-state vocabulary: `"idle" | "connecting" | "connected" | "disconnected"`.

```js
class MudraClient {
  constructor(url) {
    this._url = url;
    this._handlers = {};
    this._subscriptions = new Set();
    this._mode = 'manual';
    this._state = 'idle';
    this._ws = null;
    this._statusPoll = null;
    this._reconnectTimer = null;
    this._reconnectIdx = 0;
    this._connToken = 0;
  }

  /** Register a handler. Reserved name: '_status' receives state transitions. */
  on(signal, fn) { this._handlers[signal] = fn; }

  /** Add a signal to the subscription set. Sent on onopen, and immediately if open. */
  subscribe(signal) {
    this._subscriptions.add(signal);
    if (this._ws && this._ws.readyState === WebSocket.OPEN) {
      this._ws.send(JSON.stringify({ command: 'subscribe', signal }));
    }
  }

  /** Send an arbitrary command. In Manual mode for 'trigger_gesture', synthesize locally. */
  send(cmd) {
    if (this._ws && this._ws.readyState === WebSocket.OPEN) {
      this._ws.send(JSON.stringify(cmd));
    } else if (this._mode === 'manual' && cmd.command === 'trigger_gesture') {
      this._emit({ type: 'gesture', data: { type: cmd.data.type, confidence: 1.0, timestamp: Date.now() } });
    }
  }

  /** Atomic mode switch. Cancels poll/reconnect, closes any open socket, then opens iff "mudra". */
  setMode(mode) {
    if (mode !== 'manual' && mode !== 'mudra') return;
    if (mode === this._mode) return;
    this._mode = mode;
    this._connToken++;
    this._stopStatusPoll();
    this._clearReconnect();
    if (this._ws) {
      try { this._ws.close(); } catch (_) {}
      this._ws = null;
    }
    if (mode === 'manual') {
      this._setState('idle');
    } else {
      this._openSocket();
    }
  }

  /** Public: synthesize a signal locally (sim panel / keyboard call this). */
  emitSynthetic(payload) { this._emit(payload); }

  /** Current state. */
  get state() { return this._state; }
  get mode() { return this._mode; }

  // ---------- internal ----------

  _openSocket() {
    const token = this._connToken;
    this._setState('connecting');
    try {
      const ws = new WebSocket(this._url);
      this._ws = ws;
      ws.onopen = () => {
        if (token !== this._connToken || this._mode !== 'mudra') { try { ws.close(); } catch (_) {} return; }
        for (const sig of this._subscriptions) {
          ws.send(JSON.stringify({ command: 'subscribe', signal: sig }));
        }
        ws.send(JSON.stringify({ command: 'get_status' }));
        this._startStatusPoll();
      };
      ws.onmessage = (e) => {
        if (token !== this._connToken || this._mode !== 'mudra') return;
        let msg; try { msg = JSON.parse(e.data); } catch (_) { return; }
        if (msg.type === 'status') {
          const ok = msg.data?.device?.state === 'connected';
          this._setState(ok ? 'connected' : 'disconnected');
          return;
        }
        if (msg.type === 'connection_status') {
          if (msg.data?.status === 'disconnected') this._setState('disconnected');
          else if (this._ws && this._ws.readyState === WebSocket.OPEN) {
            this._ws.send(JSON.stringify({ command: 'get_status' }));
          }
          return;
        }
        this._emit(msg);
      };
      ws.onerror = () => {
        if (token !== this._connToken || this._mode !== 'mudra') return;
        this._setState('disconnected');
        this._stopStatusPoll();
        this._scheduleReconnect();
      };
      ws.onclose = () => {
        if (token !== this._connToken || this._mode !== 'mudra') return;
        this._setState('disconnected');
        this._stopStatusPoll();
        this._scheduleReconnect();
      };
    } catch (_) {
      this._setState('disconnected');
      this._scheduleReconnect();
    }
  }

  _startStatusPoll() {
    this._stopStatusPoll();
    this._statusPoll = setInterval(() => {
      if (this._mode !== 'mudra') { this._stopStatusPoll(); return; }
      if (this._ws && this._ws.readyState === WebSocket.OPEN) {
        this._ws.send(JSON.stringify({ command: 'get_status' }));
      }
    }, 2000);
  }
  _stopStatusPoll() { if (this._statusPoll) { clearInterval(this._statusPoll); this._statusPoll = null; } }

  _scheduleReconnect() {
    this._clearReconnect();
    const delays = [1000, 2000, 5000, 5000, 5000];
    const delay = delays[Math.min(this._reconnectIdx, delays.length - 1)];
    this._reconnectIdx++;
    const token = this._connToken;
    this._reconnectTimer = setTimeout(() => {
      if (token !== this._connToken || this._mode !== 'mudra') return;
      this._openSocket();
    }, delay);
  }
  _clearReconnect() { if (this._reconnectTimer) { clearTimeout(this._reconnectTimer); this._reconnectTimer = null; } }

  _setState(s) {
    if (s === 'connected') this._reconnectIdx = 0;
    if (s === this._state) return;
    this._state = s;
    if (this._handlers['_status']) this._handlers['_status'](s);
  }

  _emit(msg) {
    const h = this._handlers[msg.type];
    if (h) h(msg.data);
  }
}
```

### Usage pattern

```js
const mudra = new MudraClient('ws://127.0.0.1:8766');
// Subscribe to handlers + signals before showing UI:
mudra.on('gesture', (data) => { /* ... */ });
mudra.on('nav_direction', (data) => { /* ... */ });
mudra.on('_status', (s) => { /* update #mudra-status */ });
mudra.subscribe('gesture');
mudra.subscribe('nav_direction');
// Manual is default. Toggle to Mudra opens the WebSocket.
document.getElementById('mode-mudra').addEventListener('click', () => mudra.setMode('mudra'));
document.getElementById('mode-manual').addEventListener('click', () => mudra.setMode('manual'));
```

The mock is implicit — when in Manual mode, the WebSocket is closed; signals come from the sim panel + keyboard which call `mudra.emitSynthetic(...)` directly. There is no auto-firing interval anywhere.

---

## §5 — Simulator Panel

Always-visible 2D DOM overlay rendered as a **centered pill bar at the bottom** of the viewport. Disappears automatically in immersive WebXR (native DOM suppression). Renders **only** the sub-actions the generated app actually handles — never extras (spec FR-023).

### DOM scaffold

```html
<div id="mudra-sim" aria-label="Mudra signal simulator">
  <!-- One group per subscribed signal: a label + buttons. Inline layout, no <fieldset>. -->
  <span class="lbl">Gesture</span>
  <button class="btn" id="sim-tap">Tap</button>
  <button class="btn" id="sim-twist">Twist</button>
  <!-- ...additional groups for other subscribed signals... -->
</div>
```

```css
#mudra-sim {
  position: fixed;
  bottom: 16px;
  left: 50%;
  transform: translateX(-50%);
  display: flex;
  gap: 6px;
  align-items: center;
  background: var(--card);
  backdrop-filter: blur(10px);
  padding: 8px 10px;
  border-radius: 999px;
  box-shadow: 0 6px 20px rgba(0,0,0,0.10);
  z-index: 25;
  font-family: 'Poppins', system-ui, sans-serif;
}
#mudra-sim .lbl {
  font-size: 11px;
  color: var(--text-secondary);
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 1px;
  padding: 0 8px 0 4px;
}
#mudra-sim .btn {
  background: var(--bg);
  color: var(--text);
  border: 1px solid var(--text-secondary);
  padding: 6px 12px;
  border-radius: 999px;
  font-weight: 600;
  font-size: 12px;
  cursor: pointer;
  font-family: inherit;
  transition: transform 0.1s ease, background 0.15s ease, color 0.15s ease, border-color 0.15s ease;
}
#mudra-sim .btn:hover { background: var(--primary); color: var(--bg); border-color: var(--primary); }
#mudra-sim .btn:active { transform: scale(0.95); }
#mudra-sim.disabled { pointer-events: none; opacity: 0.4; }
#mudra-sim.disabled .btn { cursor: not-allowed; }
```

While in Mudra mode the panel is greyed: add the `disabled` class. Source: spec FR-021 + §18.

### Multi-signal layout

When the app subscribes to multiple signals (e.g. `gesture` + `button`), render each as its own labelled group inside the same `#mudra-sim` pill bar, separated by a thin divider (`<span class="div"></span>` styled as `width: 1px; height: 18px; background: var(--text-secondary); opacity: 0.3;`). Keep the entire control set on one row when possible; allow wrapping only when the viewport is too narrow.

### Button groups per signal (maximal sets — render subset based on handlers)

| Signal | Maximal buttons |
|--------|-----------------|
| `gesture` | `Tap`, `2Tap`, `Twist`, `2Twist` |
| `button` | `Press`, `Release` (Press is `mousedown`; Release is `mouseup` so a held hold works) |
| `pressure` | Range slider `0–100` with live value label, OR two buttons `−10` / `+10` |
| `navigation` | `↑`, `↓`, `←`, `→` (each emits one delta event of ±3) |
| `nav_direction` | `↑`, `↓`, `←`, `→`, `Roll L`, `Roll R` (subset by handlers — see §3) |
| `imu_acc` | `Tilt X+`, `Tilt X−`, `Tilt Y+`, `Tilt Y−` (5-frame burst at ±2 m/s²) |
| `imu_gyro` | `Rot X+`, `Rot X−`, `Rot Y+`, `Rot Y−` (5-frame burst at ±10 deg/s) |
| `snc` | `Spike` (burst of elevated samples on all 3 channels) |
| `battery` | (no sim — read-only from device); render a read-only label if the app subscribes |

### Contextual rendering rule (mandatory)

For each subscribed signal, walk every `mudra.on('<signal>', handler)` and inspect which sub-actions the handler responds to. Render **only** those buttons. Examples:

- App's `gesture` handler only checks `data.type === 'tap'` → render `Tap`. Omit `2Tap`, `Twist`, `2Twist`.
- App's `nav_direction` handler only branches on `Up` / `Down` → render `↑`, `↓`. Omit `←`, `→`, `Roll L`, `Roll R`.
- App's `imu_acc` handler only uses X-axis → render `Tilt X+`, `Tilt X−`. Omit Y-axis buttons.

If a button would have no observable effect, it MUST NOT appear. Unused buttons are a checklist failure (§8 item 6).

### Button firing rules

Every button fires via the **same code path** as a real Mudra signal — call `mudra.emitSynthetic(...)` so handlers registered via `mudra.on(...)` run. Never duplicate logic between the simulator path and the real-signal path.

```js
function simGesture(type) {
  // For gesture, also forward to the band when a real socket is open:
  mudra.send({ command: 'trigger_gesture', data: { type } });
  // And synthesize the message locally for Manual mode:
  if (mudra.mode === 'manual') {
    mudra.emitSynthetic({ type: 'gesture', data: { type, confidence: 1.0, timestamp: Date.now() } });
  }
}

function simPressure(value) {
  mudra.emitSynthetic({
    type: 'pressure',
    data: { value, normalized: value / 100, timestamp: Date.now() }
  });
}

function simNav(dx, dy) {
  mudra.emitSynthetic({
    type: 'navigation',
    data: { delta_x: dx, delta_y: dy, timestamp: Date.now() }
  });
}

function simNavDirection(direction) {
  mudra.emitSynthetic({
    type: 'nav_direction',
    data: { direction, timestamp: Date.now() }
  });
}
```

---

## §6 — Keyboard Map

Every subscribed signal MUST have at least one documented keyboard shortcut. Handlers are registered on `window` with `{ capture: true }` and call `event.stopPropagation()` on Mudra-claimed keys so XR Blocks' bubble-phase listeners don't double-fire.

### Canonical map

| Key | Signal | Action |
|-----|--------|--------|
| `Space` | `gesture` | tap |
| `D` (with Shift held) | `gesture` | double_tap (alternative: hold Shift then Space) |
| `T` | `gesture` | twist |
| `Shift+T` | `gesture` | double_twist |
| `Shift` keydown / keyup | `button` | press / release |
| `[` | `pressure` | value −= 10 (clamp 0) |
| `]` | `pressure` | value += 10 (clamp 100) |
| `I` | `navigation` | `delta_y += 3` (up) |
| `K` | `navigation` | `delta_y -= 3` (down) |
| `J` | `navigation` | `delta_x -= 3` (left) |
| `L` | `navigation` | `delta_x += 3` (right) |
| `ArrowUp` | `nav_direction` | `Up` — **only** if Direction mode (exception to §7) |
| `ArrowDown` | `nav_direction` | `Down` — Direction mode only |
| `ArrowLeft` | `nav_direction` | `Left` — Direction mode only |
| `ArrowRight` | `nav_direction` | `Right` — Direction mode only |
| `Shift+ArrowLeft` | `nav_direction` | `Roll Left` — Direction mode only, if handled |
| `Shift+ArrowRight` | `nav_direction` | `Roll Right` — Direction mode only, if handled |
| `U` | `imu_acc` | tilt X+ burst |
| `O` | `imu_acc` | tilt X− burst |
| `M` | `imu_gyro` | rot Y+ burst |
| `N` | `imu_gyro` | rot Y− burst |

For `imu` bursts: emit 5 synthetic frames at ±[2, 0, 9.81] m/s² (acc) or ±[10, 0, 0.5] deg/s (gyro).

### Attachment

```js
window.addEventListener('keydown', (e) => {
  // Skip when the user is typing in an input
  if (e.target instanceof HTMLInputElement || e.target instanceof HTMLTextAreaElement) return;
  switch (e.code) {
    case 'Space':       e.stopPropagation(); simGesture('tap'); break;
    case 'BracketLeft': e.stopPropagation(); simPressure(Math.max(0, currentPressure - 10)); break;
    case 'BracketRight':e.stopPropagation(); simPressure(Math.min(100, currentPressure + 10)); break;
    case 'KeyI':        e.stopPropagation(); simNav(0,  3); break;
    case 'KeyK':        e.stopPropagation(); simNav(0, -3); break;
    case 'KeyJ':        e.stopPropagation(); simNav(-3, 0); break;
    case 'KeyL':        e.stopPropagation(); simNav( 3, 0); break;
    // ... only the cases for signals the app actually subscribes to.
    // For Direction-mode arrow keys, see the exception in §7.
  }
}, { capture: true });
```

**Only intercept keys for signals the app actually subscribes to.** Do NOT `stopPropagation()` on keys the app does not handle — XR Blocks needs those (especially the reserved set in §7).

### Direction-mode arrow keys exception

When `nav_direction` is the motion mode, the app MAY bind `ArrowUp` / `ArrowDown` / `ArrowLeft` / `ArrowRight` (plus `Shift+Arrow*` for roll, if handled). This is the only allowed exception to §7's reserved-keys list because Direction mode pre-empts XR Blocks' walking — the user is on a flat 2D surface conceptually.

In any other motion mode (Pointer / IMU+Biometric / none), arrow keys remain reserved for XR Blocks — Pointer mode uses `I`/`J`/`K`/`L` and IMU uses `U`/`O`/`M`/`N`.

---

## §7 — Reserved Keys (XR Blocks desktop simulator)

The XR Blocks desktop simulator owns the keys and mouse gestures that let the user orbit, walk, and zoom the 3D scene during flat-browser review. Mudra MUST NOT claim or `stopPropagation()` any of these.

| Input | XR Blocks role |
|-------|----------------|
| `W` `A` `S` `D` | Walk camera forward / left / back / right |
| `ArrowUp` `ArrowDown` `ArrowLeft` `ArrowRight` | Camera nav (alt to WASD) — **exception**: Direction mode may bind these (§6) |
| `Q` `E` | Camera roll / vertical |
| `R` | Reset camera pose |
| Right-click drag | Orbit / look around |
| Mouse wheel | Zoom in / out |

**Why Mudra uses `I`/`J`/`K`/`L` and `U`/`O`/`M`/`N`**: these keys are explicitly off the XR Blocks reserved set, so the desktop simulator keeps full camera control while the band-driven keyboard shortcuts remain available. Never reassign a Mudra shortcut onto a reserved key — even when the app does not subscribe to a navigation signal.

The only exception is Direction motion mode, which may bind the arrow keys (§6).

---

## §8 — Pre-Write Checklist

Before writing the output HTML, verify all items. If any fails: regenerate once and re-check. If the second attempt fails: surface the failing items to the user and do not write.

| # | Check | Pass condition |
|---|-------|----------------|
| 1 | Single file | Exactly one `<html>` document; all CSS in `<style>`; all JS in `<script>` or `<script type="module">`; no `<link rel="stylesheet" href="./...">`, no external local references |
| 2 | Import map | One `<script type="importmap">` block; entries match §2 verbatim — especially the pairing `xrblocks@0.10.0` + `three@0.182.0` (NOT 0.11.0); the four mandatory entries (`three`, `three/addons/`, `xrblocks`, `xrblocks/addons/`) are present even if user code does not directly import from them (§16); only the OPTIONAL entries listed in §16 are stripped when unused |
| 3 | `xb.Script` entry | Exactly one `class … extends xb.Script` in the module body; `xb.add(new <Name>())` + `xb.init(new xb.Options())` on `DOMContentLoaded`; zero calls to `requestAnimationFrame(` |
| 4 | MudraClient | The §4 class is included verbatim; one `mudra = new MudraClient('ws://127.0.0.1:8766')` instance at module scope; constructor does NOT open a socket; `setMode()` is the only way to open one |
| 5 | Subscribe commands | Every used signal has exactly one `mudra.subscribe('<signal>')` call; no signal is subscribed that isn't handled; subscription uses key `signal` (singular) |
| 6 | Simulator panel | `<div id="mudra-sim">` present as a centered pill bar at the bottom (per §5); ONLY buttons for sub-actions actually handled; buttons call `mudra.emitSynthetic(...)` (not direct handler logic); `disabled` class applied while in Mudra mode |
| 7 | Keyboard bindings | `window.addEventListener('keydown', …, { capture: true })` present; `event.stopPropagation()` on every Mudra-claimed key; zero bindings on §7 reserved keys EXCEPT Direction-mode arrow keys |
| 8 | Combined top bar | **Single** `<div id="topbar">` containing the mode toggle on the left and the connection pill `<div id="mudra-status">` on the right (per §18 + §19); NO separate top-left or top-right elements; Manual is the default on load; pill text states are exactly `Manual mode` / `Connecting…` / `Connected` / `Disconnected`; band-state-driven (per §19); toggle remains clickable when disconnected |
| 9 | No disconnect overlay | No banner / toast / modal / inline alert for disconnect — the connection pill in `#topbar` is the only indicator (spec FR-012) |
| 10 | Onboarding overlay | `<div id="onboarding-overlay">` present (per §21); shows on every page load, no localStorage skip; four content slots (emoji+title, description, controls with `<span class="kbd">` hint, CTA label) are bespoke per app — NOT literal placeholder strings; CTA click + `Escape` key both dismiss the overlay. **If `usesAI`**: the overlay contains a fourth `.ob-body` step `#ob-step-ai` per §21 AI-app extension, the CTA is disabled until the AI key input matches `/^AIza[\w-]{30,}$/`, and the overlay cannot be dismissed without a valid key in `sessionStorage` |
| 11 | Background lockdown | ZERO `applyBackground_*` methods, ZERO background calls in `init()`, NO `options.simulator.scenePath` line anywhere. XR Blocks default room is the only allowed environment (§15) |
| 12 | Badge + AI key safety | Exactly one `<div class="mudra-badge">` containing the literal text `Created by Mudra` (no variants — spec FR-014). If the seed is AI-enabled (§17): regex scan `/AIza[A-Za-z0-9_-]{30,}\|sk-[A-Za-z0-9_-]{32,}/` returns zero matches; the key is read ONLY from `sessionStorage.getItem('mudra.gemini.apiKey')`; ZERO `prompt(` calls for the key; ZERO `localStorage` references |
| 12a | Visible AI chat I/O | If `usesAI` (§22): the 3D scene renders BOTH the user input echo AND the AI response as visible text; a fixed bespoke Purpose line is present; when the key is `null`, the response slot shows `Set up AI in the welcome panel` and NO Gemini call is made; TTS (if present) supplements rather than replaces visible text |
| 12b | Gemini model pin | If the app calls `generativelanguage.googleapis.com/v1beta/models/<id>:generateContent`, the captured `<id>` MUST equal `gemini-2.5-flash` — no preview / dated / `-latest` aliases (e.g. `gemini-2.5-flash-preview-09-2025`, `gemini-1.5-flash-latest`). Live API (`xb.core.ai.startLiveSession`) and image-gen (`gemini-2.5-flash-image`) are the only exceptions, and only when the app's purpose requires them |
| 13 | Bespoke palette | A `:root { ... }` block defines all nine canonical CSS variables (`--bg`, `--card`, `--primary`, `--accent`, `--text`, `--text-secondary`, `--success`, `--warning`, `--error`) with values picked appropriate to the app concept (per §12). The Mudra dark theme defaults are NOT used unless the concept genuinely calls for them. All chrome (`#topbar`, `#mudra-sim`, `#onboarding-overlay`, `.mudra-badge`) references the variables, not hard-coded hex values. Font is Poppins via `font-family: 'Poppins', system-ui, sans-serif`. |

### File collision rule

Target: `preview/<concept-name>.html`. If that file exists, try `preview/<concept-name>-2.html`, then `-3.html`, etc. Never overwrite. Create `preview/` if missing.

---

## §9 — Signal Inference Mapping

Source: `agent_protocol.json.signal_inference.mapping` (verbatim). Lowercase the prompt, strip inline tags, and run each entry against the prompt; pick the signals whose keywords appear.

| Signal | Keywords (from `agent_protocol.json`) |
|--------|---------------------------------------|
| `gesture` | tap, click, trigger, action, button press, drum, hit, select |
| `button` | hold, press and hold, drag, push-to-talk, sprint, charge |
| `pressure` | slide, volume, size, intensity, throttle, opacity, brush, zoom, analog |
| `navigation` | move, up/down, left/right, steer, cursor, pan, scroll, direction, arrow |
| `imu_acc + imu_gyro + snc` (bundled) | tilt, orientation, angle, rotate, 3D, balance, level, muscle, EMG, biometric, fatigue, nerve |

`nav_direction` cues are listed in §3 (skill-local, since the vendored JSON does not include them).

### Bundling rule (mandatory)

`imu_acc`, `imu_gyro`, `snc` come together. If any one matches, subscribe to all three. Source: `agent_protocol.json.signal_inference.bundling_rule`.

### Disambiguation

Ask **one** question only when the prompt maps to two **incompatible** groups (e.g. both `gesture` and `pressure`, or `navigation` and the IMU+Biometric bundle). Source: `agent_protocol.json.signal_inference.when_ambiguous`. Never silently pick one of two.

Examples of legitimate disambiguation prompts:

- *"Pinch to fire and squeeze to charge"* — `gesture` (tap) + `pressure` are mutually exclusive. Ask: "Which interaction model fits better: discrete pinches (gesture) or analog squeeze (pressure)?"
- *"Swipe to navigate and tilt to look around"* — `nav_direction` (Direction) + IMU bundle are mutually exclusive. Ask: "Which fits the spatial idea better: discrete swipes (Direction) or continuous tilt (IMU+Biometric)?"

---

## §10 — Compatibility Rules + Valid Combinations

Source: `agent_protocol.json.compatibility` (verbatim).

### Signal groups (from `compatibility.signal_groups`)

- **discrete**: `gesture`, `button`
- **analog**: `pressure`
- **pointer_motion**: `navigation`, `button`
- **direction_motion**: `nav_direction`
- **imu_biometric**: `imu_acc`, `imu_gyro`, `snc`

### Bundling rules (from `compatibility.bundling_rules`)

1. `imu_acc`, `imu_gyro`, and `snc` are always subscribed together — using any one requires all three.
2. `gesture` and `pressure` are mutually exclusive — never combine them.
3. `navigation` and `nav_direction` are mutually exclusive — never combine them.
4. The imu_biometric bundle (`imu_acc + imu_gyro + snc`) and `navigation` are mutually exclusive.

(Skill-local extension: `nav_direction` and the imu_biometric bundle are also mutually exclusive, per `compatibility.cannot_combine` entries.)

### Valid combinations (from `compatibility.valid_combinations`)

- `gesture + button`
- `pressure + button`
- `navigation + button`
- `nav_direction` (alone or with `gesture` OR `pressure`, plus `button`)
- `imu_acc + imu_gyro + snc`
- `imu_acc + imu_gyro + snc + gesture`
- `imu_acc + imu_gyro + snc + button`
- `navigation + button + gesture`

`battery` may be added to any of the above (status-only, no interaction conflict).

### Cannot combine (from `compatibility.cannot_combine`)

| Signals | Reason |
|---------|--------|
| `gesture`, `pressure` | Mutually exclusive analog vs discrete control — pick one interaction model |
| `navigation`, `nav_direction` | Mutually exclusive motion modes — `navigation` is continuous pointer, `nav_direction` is discrete swipes |
| `navigation`, `imu_acc` | Different firmware targets — hardware limitation; also `imu_acc` belongs to the imu_biometric bundle which is incompatible with `navigation` |
| `navigation`, `imu_gyro` | Same as above for `imu_gyro` |
| `navigation`, `snc` | The imu_biometric bundle is incompatible with `navigation` |
| `nav_direction`, `imu_acc` | Different motion modes cannot be combined |
| `nav_direction`, `imu_gyro` | Same |
| `nav_direction`, `snc` | The imu_biometric bundle is incompatible with `nav_direction` |

### Guidance

If a user's concept needs both directional movement and orientation/biometrics, explain the conflict and recommend the better fit. If they want any of `imu_acc`, `imu_gyro`, `snc`, subscribe to all three.

---

## §11 — Creative Proposals

Source: `agent_protocol.json.creative_proposals` (verbatim).

When the user's concept maps cleanly to one signal, propose **one** complementary signal — one sentence, no more. Honor the `forbidden_proposals` list.

| User's concept | Proposed complement |
|----------------|---------------------|
| Drum kit (`gesture`) | Button hold could sustain a note. |
| Drawing app (`pressure`) | Button could toggle between draw and erase mode. |
| Racing game (`navigation`) | Button for boost, tap on `nav_direction` for turbo. |
| Music player (`gesture`) | Navigation could add a seek scrubber. |
| Biometrics dashboard (IMU+SNC bundle) | All three sensors come together — button could trigger a snapshot. |

### Forbidden proposals (mandatory)

- Never propose `pressure` to complement `gesture` — they are mutually exclusive.
- Never propose `gesture` to complement `pressure` — they are mutually exclusive.
- Never propose `snc` alone — it must always come with `imu_acc` and `imu_gyro`.

---

## §12 — Theme + Badge

**Each generated app picks a bespoke palette appropriate to its concept** — not the universal Mudra dark theme. The CSS custom-property *names* below are canonical and MUST appear in every generated app; the *values* are picked per app by the skill at generation time. The font remains Poppins.

### Canonical variable names (always present)

```css
:root {
  --bg: <bespoke>;
  --card: <bespoke>;          /* card / panel surface — semi-opaque on top of --bg looks great */
  --primary: <bespoke>;       /* brand / action color — saturated, used for active states + CTAs */
  --accent: <bespoke>;        /* secondary highlight */
  --text: <bespoke>;          /* primary text — must have AA contrast vs --bg */
  --text-secondary: <bespoke>;/* muted labels, dividers, secondary copy */
  --success: <bespoke>;       /* green-ish — "Connected" pill */
  --warning: <bespoke>;       /* amber/yellow — "Connecting…" pill */
  --error: <bespoke>;         /* red — "Disconnected" pill */
}
body {
  font-family: 'Poppins', system-ui, sans-serif;
  background: var(--bg);
  color: var(--text);
}
```

Apply these variables to `#topbar`, `#mudra-sim`, `#onboarding-overlay`, `.mudra-badge`, and any app-specific chrome (HUD, score, end panels, etc.).

### Palette-picking guidance

The skill picks palette values that match the **emotional register of the app concept**. Anchor each palette to the central concept, not to the Mudra brand. Examples (the skill SHOULD invent fresh palettes — these are illustrative, not exhaustive):

| Concept | Mood | Sample palette |
|---------|------|----------------|
| Casual game ("fluffy bird", "candy collector") | Cheerful, soft | `--bg-top: #ffe5ee → --bg-bot: #b8e7ff` gradient, `--primary: #ff8fb1`, `--card: #ffffffcc` |
| Racing / arcade | Energetic, high-contrast | `--bg: #0a0a14`, `--primary: #00ffe1`, `--accent: #ff00aa`, `--text: #fff` |
| Meditation / breathwork | Calm, muted | `--bg: linear-gradient(#3b2d52 → #1a2042)`, `--primary: #c7b9ff`, `--card: #ffffff14` |
| Drum kit / music | Bold, rhythmic | `--bg: #15121f`, `--primary: #ffd166`, `--accent: #ef476f` |
| Dashboard / biometrics | Clinical, focused | `--bg: #f3f5f7`, `--primary: #2a6df4`, `--text: #1a2a3a` |
| Sci-fi / space | Cosmic, deep | `--bg: #000`, `--primary: #77eae9` (the Mudra cyan still works here), `--accent: #b388ff` |

### Constraints (mandatory)

1. **Contrast**: `--text` vs `--bg` MUST clear WCAG AA contrast (4.5:1 for body text). The skill should mentally check; if unsure, default toward higher contrast.
2. **Status colors recognizable**: `--success` reads green, `--warning` reads amber/yellow, `--error` reads red. The status pill relies on these conventions for accessibility.
3. **Backgrounds may be gradients**: `--bg` can be a linear-gradient value if the app concept benefits (e.g. casual game's sky-to-pink gradient). Use `background: var(--bg)` on `body`; the gradient applies cleanly.
4. **`backdrop-filter: blur(10px)`** on `#topbar`, `#mudra-sim`, and `.onboarding-panel` is mandatory — it ties the chrome to the scene background.
5. **Font is Poppins** always. Do not substitute. Load via system-ui fallback (no need to add a `<link>` to Google Fonts; the system fallback is acceptable).

The legacy Mudra dark palette (`--bg: #000`, `--primary: #77EAE9`, `--card: #181e21`, `--text: #f8fafc`, `--text-secondary: #94a3b8`, `--success: #22c55e`, `--warning: #eab308`, `--error: #ef4444`) from `agent_protocol.json.theme` remains a valid fallback when the app concept is genuinely "minimal" or "tooling-like" and no other palette feels natural. But it is NO LONGER the universal default.

### Badge — literal text "Created by Mudra"

```html
<div class="mudra-badge">Created by Mudra</div>
```

```css
.mudra-badge {
  position: fixed;
  bottom: 12px;
  right: 14px;
  font-size: 11px;
  color: var(--text-secondary);
  opacity: 0.7;
  font-family: inherit;
  letter-spacing: 0.04em;
  pointer-events: none;
  z-index: 5;
}
```

Place fixed bottom-right by default. **Never** use a different text variant (no "Powered by", no "Created with Mudra Studio"). Use only `var(--text-secondary)` for the color — never a hard-coded hex — so the badge automatically matches the bespoke palette.

If the badge's default fixed position would collide with `#mudra-sim` at the bottom of the screen, lift it to `bottom: 56px` so it sits just above the simulator pill bar.

---

## §13 — Seed Catalog + Scoring Algorithm

Seeds are templates, samples, demos, and gallery examples cataloged in `references/xrpromt.md`. Below is a representative subset; the full catalog lives in `xrpromt.md` (locate by heading per §14). When scoring, treat any heading in `xrpromt.md` as a valid seed id even if it's not in this short table.

### Representative catalog rows

| `id` | category | motion_modes_supported | xr_features | keywords |
|------|----------|------------------------|-------------|----------|
| `0_basic` | template | `direction`, `none` | minimal | basic, fallback, pinch, color, demo |
| `1_ui` | template | `direction`, `none` | text, panel | ui, panel, button, text, menu, dialog |
| `2_hands` | template | `direction`, `none` | hands | hands, gesture, finger, pinch |
| `3_depth` | template | `direction`, `none` | depth | depth, distance, near, far |
| `4_stereo` | template | `none` | stereo | stereo, 3d, parallax |
| `5_camera` | template | `direction`, `none` | camera | camera, video, pass-through |
| `6_ai` | template | `direction`, `none`, **ai** | gemini | ai, gemini, llm, describe, identify, vision |
| `7_ai_live` | template | `direction`, `none`, **ai** | gemini, live | ai, gemini, live, voice, conversation, real-time |
| `8_objects` | template | `direction`, `none` | object_detection | object, detect, world, recognize |
| `9_xr-toggle` | template | `direction`, `none` | xr-mode | toggle, enter xr, exit xr, mode |
| `heuristic_hand_gestures` | sample | `direction`, `none` | hands, gestures | gesture, heuristic, hand, recognize |
| `meshes` | sample | `direction`, `none` | meshes | mesh, room, scene-mesh |
| `planes` | sample | `direction`, `none` | planes | plane, surface, table, floor, wall |
| `uikit` | sample | `direction`, `none` | text, panel | ui, kit, panel, button |
| `depthmap` | sample | `direction`, `none` | depth | depthmap, raw, near-far |
| `depthmesh` | sample | `direction`, `none` | depth, meshes | depth, mesh |
| `game_rps` | demo | `direction`, `none` | hands | rock paper scissors, game, hand, gesture |
| `gestures_custom` | sample | `direction`, `none` | hands | custom gesture, gesture, trained |
| `gestures_heuristic` | sample | `direction`, `none` | hands | gesture, heuristic |
| `lighting` | sample | `direction`, `none` | lighting | light, hdr, illumination |
| `mesh_detection` | sample | `direction`, `none` | meshes | mesh, detect, scene |
| `modelviewer` | sample | `direction`, `none` | models | model, viewer, gltf |
| `paint` | demo | `pointer`, `direction`, `none` | hands | paint, draw, brush, finger |
| `planar-vst` | sample | `direction`, `none` | planes, video | plane, video, vst |
| `reticle` | sample | `pointer`, `direction`, `none` | reticle | reticle, target, aim |
| `skybox_agent` | sample | `direction`, `none` | sky | skybox, environment |
| `sound` | sample | `direction`, `none` | audio | sound, audio, spatial |
| `ui` | sample | `direction`, `none` | text, panel | ui, kit, panel |
| `virtual-screens` | sample | `direction`, `none` | screens | screen, monitor, window, virtual |
| `3dgs-walkthrough` | demo | `imu`, `pointer`, `direction`, `none` | gaussian-splat | walk, splat, gaussian, scan |
| `aisimulator` | gallery | `direction`, `none`, **ai** | gemini | ai, simulator, gemini |
| `balloonpop` | demo | `direction`, `none` | hands | balloon, pop, game |
| `ballpit` | demo | `direction`, `none` | physics | ball, pit, physics, throw |
| `drone` | demo | `pointer`, `direction` | flight | drone, fly, control, hover |
| `gemini-icebreakers` | gallery | `none`, **ai** | gemini | gemini, icebreaker, ai, chat |
| `gemini-xrobject` | gallery | `direction`, `none`, **ai** | gemini, models | gemini, xrobject, ai, model |
| `math3d` | demo | `direction`, `none` | math | math, 3d, graph, plot |
| `measure` | demo | `direction`, `none` | measure | measure, ruler, distance |
| `occlusion` | sample | `direction`, `none` | depth | occlusion, depth, hide |
| `rain` | demo | `none` | weather | rain, weather, particles |
| `screenwiper` | demo | `pointer` | hands | wiper, clean, wipe, screen |
| `splash` | demo | `none` | splash | splash, intro, landing |
| `webcam_gestures` | sample | `direction`, `none` | hands, camera | webcam, gesture, camera |
| `xremoji` | demo | `direction`, `none` | emoji | emoji, react, expression |
| `xrpoet` | gallery | `none`, **ai** | gemini, text | gemini, poet, poem, ai |

(`**ai**` mark in the table means AI-enabled; selection gated by §17.)

### Scoring algorithm

For each candidate row, score against the lowercased prompt (after stripping inline tags):

1. **Signal overlap**: +3 per matching subscribed signal name found in `keywords`.
2. **Motion mode match**: +5 if inferred motion mode is in `motion_modes_supported`; +2 if `none` is in `motion_modes_supported`.
3. **XR feature match**: +2 per matching XR feature keyword from the prompt in `xr_features`.
4. **Keyword overlap**: +1 per keyword from the row's `keywords` array that appears in the lowercased prompt.

### Tie-breakers (data-model.md Entity 4)

1. Prefer `direction` motion mode over others (most app-agnostic).
2. Prefer templates over samples, samples over demos, demos over gallery.
3. If still tied, pick `0_basic` as the universal fallback.

### Examples

- `"hand-tracking demo"` → `2_hands` or `heuristic_hand_gestures` (hands keyword strong; `direction` motion mode tie-break).
- `"a 3D color picker controlled by pinch"` → `0_basic` (pinch + color → keywords overlap with `0_basic`).
- `"spatial menu I swipe through"` → `1_ui` or `uikit` (ui + swipe + Direction mode).
- `"tilt-controlled spaceship"` → IMU mode → `3dgs-walkthrough` (only catalog row supporting `imu`) or fall back to a template; verify with §14 lookup.
- No clear match → `0_basic` (universal fallback).

---

## §14 — Seed Lookup Procedure in vendored `xrpromt.md`

The vendored `references/xrpromt.md` is the single source of seed HTML — no `assets/` folder duplicates it (research R1).

### Lookup steps

1. Open `references/xrpromt.md` (vendored, byte-identical to repo root).
2. Locate the heading matching the selected id. Headings use one of the four prefixes:
   - `### Template: <id>`
   - `### Sample: <id>`
   - `### Demo: <id>`
   - `### Gallery: <id>`
3. Below the heading (within ~5 lines) is a `File: <path>` line — informational only.
4. Capture the **first** ```` ```html ... ``` ```` fenced block that follows the heading. That block is the seed `source_html`.
5. If no matching heading exists, the id is invalid → reject with the valid-id list (gather every `### Template:`/`### Sample:`/`### Demo:`/`### Gallery:` heading from `xrpromt.md`).

### Read budget

When reading `xrpromt.md`, prefer `Read` with `offset` + `limit` after locating the heading line number via `grep` — never read the entire 33k-line file. Typical seed block is 80–200 lines; allow up to 300 lines after the heading.

```bash
# Locate the heading line number:
grep -n "^### Template: 2_hands$" references/xrpromt.md
# Then Read with offset = <line number>, limit = 300
```

---

## §15 — Background Lockdown (XR Blocks default only)

**Custom backgrounds are forbidden.** Every generated XR app uses the XR
Blocks default room and nothing else. There is no catalog, no helper, no
override, no `[bg=...]` tag, no `scenePath` override.

### Hard rules — apply unconditionally

1. The `xb.Script` subclass MUST NOT contain any `applyBackground_*` method.
2. `init()` MUST NOT call any background helper. It starts with lights,
   meshes, and Mudra wiring.
3. The entry point MUST NOT set `options.simulator.scenePath` — not to
   `null`, not to a path. Leave it alone so XR Blocks renders its default
   room.
4. No standalone environment domes/floors/skies: no `THREE.SphereGeometry`
   skybox `Mesh`, no `THREE.Points` starfield, no `THREE.GridHelper`
   environment floor, no equirectangular `TextureLoader().load(...)` for
   background purposes. (Per-scene geometry the app itself needs is fine —
   the ban is on standalone background environments.)
5. Prompt cues like "in space", "starfield", "sunset sky", "cyberpunk
   vibe", "with a forest backdrop", and even literal `[bg=<id>]` tags MUST
   be IGNORED for background purposes. They may still inform template /
   motion-mode selection.
6. If the user explicitly insists on a custom background, decline and
   remind them that the skill is locked to the XR Blocks default room.

### Pre-write regex (checklist §8 item 11)

The generated source MUST satisfy ALL of the following:

- `/applyBackground_/` → zero matches.
- `/options\.simulator\.scenePath/` → zero matches.

If either pattern matches, the file fails the pre-write checklist and is
not written.

---

## §16 — Import-Map Rules

1. **Pinned versions only** — use the URLs in §2 verbatim. No `@latest`, no `^x.y`, no `~x.y`. The pairing `xrblocks@0.10.0` + `three@0.182.0` is mandatory; do not substitute other versions even if a seed template embeds them.
2. **No unused application entries** — every USER-CODE-LEVEL import in the module body must have a matching import-map key. But: the four entries below are MANDATORY and MUST stay even if user code does not reference them directly, because XR Blocks imports from them transitively.
3. **CDN-only** — all dependencies load from the canonical CDN URLs. No local `<script src="./..."/>` references.
4. **One import map per file** — exactly one `<script type="importmap">` block.

### Mandatory entries (never strip)

| Key | Must keep | Reason |
|-----|-----------|--------|
| `three` | **always** | Required by xrblocks. |
| `three/addons/` | **always** | xrblocks internally imports `three/addons/postprocessing/Pass.js` and other addon files. Removing this entry breaks xrblocks load with `Failed to resolve module specifier`. |
| `xrblocks` | **always** | Entry point. |
| `xrblocks/addons/` | **always** | xrblocks loads its own addon files via this prefix. Removing it breaks scene init. |
| `lit`, `lit/` | **always when xrblocks panels render** | xrblocks UI components are Lit-based. Keep these unless you have positively verified that no `xb.*UI*` / `xb.Panel` / `xb.ModelViewer` / `xb.ai` component is in use. When in doubt, keep them. |

### Optional entries (strip when unused)

| Key | Keep when |
|-----|-----------|
| `troika-three-text` | when 3D text is used (e.g. in `1_ui`, `uikit`, AI seeds) |
| `troika-three-utils` | when `troika-three-text` is kept |
| `troika-worker-utils` | when `troika-three-text` is kept |
| `bidi-js` | when `troika-three-text` is kept |
| `webgl-sdf-generator` | when `troika-three-text` is kept |

### Stripping procedure

After adapting the seed, walk the module body and list every distinct `import ... from '<name>'`. Compare against the **optional** keys above and strip any that have no match. **Never strip the mandatory entries** — they exist to satisfy xrblocks's transitive imports, not the user code's direct imports. When in doubt, keep an entry rather than risk a runtime resolution error.

Source: `references/xrpromt.md` Reference Map. If the vendored corpus's URLs ever drift, re-vendor (see §20) and update §2 accordingly.

---

## §17 — AI-enabled Seed Gating

Source: spec FR-019a + clarification Q4.

### AI-enabled seed ids

The following catalog rows (or their equivalents in `xrpromt.md`) are AI-enabled:

- `6_ai` — template
- `7_ai_live` — template
- `gemini-icebreakers` — gallery
- `gemini-xrobject` — gallery
- `aisimulator` — gallery
- `xrpoet` — gallery
- Any other heading in `xrpromt.md` whose seed HTML references `xb.ai` / `Gemini` / `API_KEY`.

### Selection rule

AI-enabled seeds MUST be selected only when ONE of these conditions holds:

1. The user's prompt explicitly contains one of: `AI`, `Gemini`, `LLM`, `language model`, `describe with AI`, `vision model`, `chatbot`, `narrate with AI`, `prompt the AI`, `Gemini live`, `voice AI`. The match is **substring + word-boundary insensitive** but does not match incidental cues like `"smart"`, `"clever"`, `"narrate"` (standalone), `"describe"` (standalone) — those alone are NOT triggers.
2. The user supplies inline `[template=<id>]` pointing at an AI-enabled seed.

### Generated app rules

When an AI-enabled seed IS selected:

1. The API key is **NEVER embedded** in the file. Pre-write regex scan
   (`/AIza[A-Za-z0-9_-]{30,}|sk-[A-Za-z0-9_-]{32,}/`) MUST return zero matches.
2. The key is obtained **exclusively through the onboarding overlay**
   (§21) — not via `prompt()`, not via URL params, not via `localStorage`,
   not lazily on first AI call.
3. Storage: `sessionStorage` under the literal key `mudra.gemini.apiKey`
   (per-tab; cleared on tab close). Read it with
   `sessionStorage.getItem('mudra.gemini.apiKey')` and treat `null` as
   "AI inert — show the placeholder, do not call Gemini".
4. The overlay's primary CTA (Continue / Start) is **disabled** until
   the input matches `/^AIza[\w-]{30,}$/`. While the key is missing, the
   user CANNOT dismiss the overlay (Escape, `×` close, backdrop click all
   no-op until a valid key is entered and written). Once a valid key is
   in `sessionStorage`, the overlay returns to normal dismiss behavior
   on subsequent loads.
5. The app MUST also satisfy §22 (Visible AI Chat I/O): on-screen
   rendering of the user input AND the AI response, plus a fixed
   one-sentence Purpose line authored from the user's prompt.

### Decline mode

If the prompt does not satisfy condition 1 or 2 but the scoring algorithm in §13 nevertheless picks an AI-enabled seed (because of high keyword overlap), demote that seed and pick the next-highest non-AI-enabled seed. If only AI-enabled seeds remain, fall back to `0_basic`.

---

## §18 — Mode Toggle DOM + Behavior (combined top bar)

Manual is the default on load. The Mode toggle MUST remain clickable and keyboard-focusable at all times — including while the band is disconnected. **The mode toggle and connection pill live inside a single `#topbar` element** centered at the top of the viewport (the v1.0.0 split with the toggle top-left and pill top-right is removed in v1.2.0).

### DOM

```html
<div id="topbar">
  <div class="mode-toggle" role="radiogroup" aria-label="Mode">
    <button id="mode-manual" role="radio" aria-checked="true" class="mode-opt active">Manual</button>
    <button id="mode-mudra"  role="radio" aria-checked="false" class="mode-opt">Mudra</button>
  </div>
  <div id="mudra-status" class="conn-pill conn-manual" role="status" aria-live="polite">Manual mode</div>
</div>
```

```css
#topbar {
  position: fixed;
  top: 12px;
  left: 50%;
  transform: translateX(-50%);
  display: flex;
  align-items: center;
  gap: 12px;
  background: var(--card);
  backdrop-filter: blur(10px);
  padding: 8px 12px;
  border-radius: 999px;
  box-shadow: 0 6px 24px rgba(0,0,0,0.18);
  z-index: 30;
  font-family: 'Poppins', system-ui, sans-serif;
}
.mode-toggle {
  display: inline-flex;
  gap: 4px;
  background: var(--bg);
  padding: 3px;
  border-radius: 999px;
  border: 1px solid var(--text-secondary);
}
.mode-opt {
  background: transparent;
  color: var(--text-secondary);
  border: none;
  padding: 6px 14px;
  border-radius: 999px;
  cursor: pointer;
  font-weight: 600;
  font-size: 13px;
  font-family: inherit;
  transition: background 0.15s ease, color 0.15s ease;
}
.mode-opt.active {
  background: var(--primary);
  color: var(--card);
  box-shadow: 0 2px 6px rgba(0,0,0,0.2);
}
```

### Wiring

```js
const btnManual = document.getElementById('mode-manual');
const btnMudra  = document.getElementById('mode-mudra');
const simPanel  = document.getElementById('mudra-sim');

function applyMode(mode) {
  btnManual.setAttribute('aria-checked', mode === 'manual' ? 'true' : 'false');
  btnMudra.setAttribute('aria-checked',  mode === 'mudra'  ? 'true' : 'false');
  btnManual.classList.toggle('active', mode === 'manual');
  btnMudra.classList.toggle('active',  mode === 'mudra');
  simPanel.classList.toggle('disabled', mode === 'mudra');
  mudra.setMode(mode);
}

btnManual.addEventListener('click', () => applyMode('manual'));
btnMudra .addEventListener('click', () => applyMode('mudra'));

// Default: manual
applyMode('manual');
```

### Atomicity

Every mode flip MUST atomically: cancel any reconnect timer, stop the status-poll, close any open socket, reset the connection token, then either (Manual) leave the state at `idle` or (Mudra) open a new socket. The `MudraClient.setMode(...)` from §4 implements this.

### Disappearance in immersive XR

Like the simulator panel, status pill, and badge — the `#topbar` is a 2D DOM overlay and disappears automatically when the browser enters an immersive WebXR session.

---

## §19 — Status Indicator + Band-State Polling

The connection pill `#mudra-status` lives **inside `#topbar`** to the right of the mode toggle (per §18). It is the **only** disconnect indicator — no banner / toast / modal / inline alert. Spec FR-012.

### Styling

The pill changes background + color per state via CSS classes. It uses `--success` / `--warning` / `--error` from the bespoke palette (§12), so the colors always feel cohesive with the rest of the app.

```css
.conn-pill {
  display: inline-block;
  padding: 5px 12px;
  border-radius: 999px;
  font-size: 12px;
  font-weight: 600;
  border: 1px solid transparent;
  font-family: inherit;
}
.conn-manual       { background: transparent;         color: var(--text-secondary); border-color: var(--text-secondary); }
.conn-connecting   { background: var(--warning);      color: var(--bg); }
.conn-connected    { background: var(--success);      color: var(--card); }
.conn-disconnected { background: var(--error);        color: var(--card); }
```

### Text states (mandatory)

| State | textContent | CSS class |
|-------|-------------|-----------|
| `idle` (Manual) | `Manual mode` | `conn-manual` |
| `connecting` | `Connecting…` | `conn-connecting` |
| `connected` | `Connected` | `conn-connected` |
| `disconnected` | `Disconnected` | `conn-disconnected` |

### Wiring

```js
const pill = document.getElementById('mudra-status');
const LABELS = {
  idle: 'Manual mode',
  connecting: 'Connecting…',
  connected: 'Connected',
  disconnected: 'Disconnected',
};
const CLASSES = {
  idle: 'conn-manual',
  connecting: 'conn-connecting',
  connected: 'conn-connected',
  disconnected: 'conn-disconnected',
};
mudra.on('_status', (s) => {
  pill.classList.remove('conn-manual', 'conn-connecting', 'conn-connected', 'conn-disconnected');
  pill.classList.add(CLASSES[s] ?? 'conn-manual');
  pill.textContent = LABELS[s] ?? s;
});
```

### Band-state polling (mandatory)

The WebSocket handshake only proves the Companion service is up — it does NOT prove the user's Mudra Band is paired. The Companion accepts socket connections even when no band is bonded, so flipping the pill to `Connected` on `ws.onopen` is a lie.

The source of truth is the `status` response to `{command:"get_status"}`. The `MudraClient` in §4 implements the poll:

```
> {"command":"get_status"}
< {"type":"status","data":{"device":{"state":"connected", ...}, ...}, "timestamp": ...}
< {"type":"status","data":{"device":{"state":"disconnected", ...}, ...}, "timestamp": ...}
```

Rules:

1. On `ws.onopen` (Mudra mode): stay in `connecting`; send all `subscribe` commands; send `{command:"get_status"}`; start a poll that re-sends `{command:"get_status"}` every **2000 ms** while `mode === "mudra"` and the socket is `OPEN`.
2. On inbound `{type:"status"}` (Mudra mode):
   - `data?.device?.state === "connected"` → `setState("connected")`.
   - Else → `setState("disconnected")`. Keep the socket open; do not close it. The next poll picks the band up when it pairs.
3. On inbound `{type:"connection_status"}`: a hint only. On `disconnected`, flip the pill. On `connected`, send a fresh `{command:"get_status"}` and let the `status` handler do the transition.
4. On WS `error` / `close` (Mudra mode): `setState("disconnected")`, stop the poll, schedule reconnect (1 s / 2 s / 5 s / 5 s / 5 s, reset on next `connected`).
5. On Manual mode: stop the poll inside `setMode("manual")` (the §4 `MudraClient` handles this in `_stopStatusPoll`).

Poll interval: 2000 ms exactly (between 1 s and 5 s; 2 s mandated). The pill MAY sit on `Connecting…` for up to one poll cycle after entering Mudra mode while the first `status` round-trips — that is correct behavior.

### No disconnect overlay (mandatory)

Spec FR-012. Do not render any of:

- A modal dialog announcing disconnection
- A toast notification on disconnect
- A banner across the top / bottom of the viewport
- An inline alert inside the scene

The pill is the only allowed disconnect cue. If the simulator panel grey-out is sufficient as a secondary visual cue, that is permitted (CSS opacity reduction only — no extra DOM).

---

## §20 — Vendoring Provenance

This skill is fully self-contained. The two upstream source files are vendored byte-identical into `references/`:

| Vendored file | Upstream source | Last vendored | Re-vendoring procedure |
|---------------|-----------------|---------------|------------------------|
| `references/agent_protocol.json` | `mudra-plugin/skills/mudra-master/mudra-preview/references/agent_protocol.json` (v2.0) | 2026-05-13 | `cp <upstream> .claude/skills/mudra-xr-2/references/agent_protocol.json && diff <upstream> .claude/skills/mudra-xr-2/references/agent_protocol.json` — `diff` MUST be empty |
| `references/xrpromt.md` | repo-root `xrpromt.md` | 2026-05-13 | `cp xrpromt.md .claude/skills/mudra-xr-2/references/xrpromt.md && diff xrpromt.md .claude/skills/mudra-xr-2/references/xrpromt.md` — `diff` MUST be empty |

### Maintainer rules

1. **Never edit the vendored files in place.** They must stay byte-identical to upstream. Edits go upstream, then are re-vendored.
2. **Re-vendor on protocol bumps.** When upstream `agent_protocol.json` ships a new version, re-vendor and re-run the skill's quickstart smoke prompts (see `specs/006-mudra-xr-2/quickstart.md`).
3. **Re-vendor on XR Blocks pin bumps.** When `xrpromt.md` updates the import-map URLs or pinned versions, re-vendor and bump §2 of this file if needed.
4. **Do not symlink.** Vendoring is byte-identical *copying* — symlinks defeat the self-contained-skill goal (research R4).

### Bump tracking

If, during normal evolution of `mudra-xr-2`, this skill needs rules that diverge from the upstream protocol (e.g. a future `nav_direction` payload extension), document the divergence in the section that introduces it (e.g. §3) and update §20 with a "skill-local extensions" note. The vendored JSON itself stays byte-identical.

---

## §21 — Onboarding Overlay (every load, multi-step)

Every generated app MUST show a **multi-step onboarding overlay** on **every page load**. No `localStorage` skip; the overlay is intentionally re-shown each time. The layout follows the XR Blocks default "Welcome to XR Blocks" modal: title in the top-left, X close button in the top-right, body in the middle, and a `Continue` / `Back` CTA pair plus a step counter at the bottom-right.

### Step structure (mandatory — exactly 3 steps)

| # | Title | Body |
|---|-------|------|
| 1 | `Welcome to Mudra Band` | Bespoke intro to the app: emoji + app name as a sub-heading, one-sentence description, one-sentence note about the mode toggle (Manual is default; Mudra opens the band connection). |
| 2 | `Controls` | Bullet list — one bullet per signal binding the app actually uses. Each bullet has a **bold action label** (matching the app concept), the gesture/key with inline `<span class="kbd">…</span>` hints, and a brief plain-English description. |
| 3 | `Practice without a band` | Bespoke note explaining how to use the simulator panel at the bottom + keyboard shortcuts while in Manual mode, then switch to Mudra mode when a band is paired. |

The bullet list in step 2 is **derived from the actual subscribed signals** at generation time — not template-substituted. For an app that subscribes to `nav_direction` (Up/Down only) + `gesture` (tap), the list contains exactly two bullets. For an app that subscribes to the full IMU+Biometric bundle, the list contains tilt + roll + muscle-activity bullets. Never list signals the app does not handle.

### DOM

```html
<div id="onboarding-overlay">
  <div class="onboarding-panel">
    <button class="ob-close" id="ob-close" aria-label="Close">×</button>
    <h1 class="ob-title" id="ob-title">Welcome to Mudra Band</h1>

    <div class="ob-body" id="ob-step-1">
      <h2>{emoji} {app name}</h2>
      <p>{one-sentence description}</p>
      <p>{one-sentence mode toggle note}</p>
    </div>

    <div class="ob-body" id="ob-step-2" hidden>
      <h2>Controls</h2>
      <p>{one-sentence preface}</p>
      <ul>
        <li><strong>{action label}:</strong> {gesture/key with <span class="kbd">…</span> hints} — {brief description}.</li>
        <!-- one <li> per subscribed signal binding -->
      </ul>
    </div>

    <div class="ob-body" id="ob-step-3" hidden>
      <h2>Practice without a band</h2>
      <p>{bespoke note about simulator + keyboard}</p>
      <ul>
        <li><strong>Simulator panel:</strong> the pill bar at the bottom of the screen has one button per action listed above. Click each to fire the signal.</li>
        <li><strong>Keyboard:</strong> {list the relevant <span class="kbd">…</span> shortcuts}.</li>
        <li><strong>Switch to Mudra mode:</strong> when your band is paired, click <strong>Mudra</strong> in the top bar.</li>
      </ul>
    </div>

    <div class="ob-footer">
      <span class="ob-counter" id="ob-counter">1 of 3</span>
      <div class="ob-actions">
        <button class="ob-back" id="ob-back" hidden>Back</button>
        <button class="ob-next cta" id="ob-next">Continue</button>
      </div>
    </div>
  </div>
</div>
```

### Styling

Uses the bespoke palette variables from §12. The panel is centered, opaque-ish card surface (no header video / hero image — those are explicitly out of scope), with the X in the top-right corner and the action row in the bottom-right.

```css
#onboarding-overlay {
  position: fixed; inset: 0;
  display: flex; justify-content: center; align-items: center;
  z-index: 40;
  background: rgba(0, 0, 0, 0.45);
  backdrop-filter: blur(6px);
  font-family: 'Poppins', system-ui, sans-serif;
}
.onboarding-panel {
  position: relative;
  background: var(--card);
  backdrop-filter: blur(14px);
  border-radius: 28px;
  padding: 36px 40px 28px;
  width: min(560px, 92vw);
  max-height: 88vh;
  overflow-y: auto;
  box-shadow: 0 16px 48px rgba(0, 0, 0, 0.35);
  border: 1px solid rgba(255,255,255,0.08);
  color: var(--text);
}
.ob-title {
  font-size: 26px;
  font-weight: 800;
  color: var(--text);
  margin: 0 0 22px 0;
  letter-spacing: 0.3px;
}
.ob-close {
  position: absolute;
  top: 18px; right: 18px;
  width: 36px; height: 36px;
  border-radius: 50%;
  background: rgba(0,0,0,0.55);
  color: var(--text);
  border: none;
  font-size: 18px;
  cursor: pointer;
  display: flex; align-items: center; justify-content: center;
  transition: background 0.15s ease;
}
.ob-close:hover { background: rgba(0,0,0,0.75); }
.ob-body h2 {
  font-size: 20px;
  font-weight: 700;
  color: var(--text);
  margin: 0 0 10px 0;
}
.ob-body p {
  font-size: 14.5px;
  color: var(--text-secondary);
  line-height: 1.55;
  margin: 0 0 14px 0;
}
.ob-body ul {
  margin: 8px 0 0 0;
  padding-left: 22px;
  color: var(--text-secondary);
  font-size: 14.5px;
  line-height: 1.7;
}
.ob-body li { margin-bottom: 6px; }
.ob-body li strong { color: var(--text); font-weight: 700; }
.ob-body .kbd {
  display: inline-block;
  background: var(--bg);
  border: 1px solid var(--text-secondary);
  border-bottom-width: 2px;
  border-radius: 6px;
  padding: 1px 7px;
  font-family: 'Courier New', monospace;
  font-size: 12px;
  color: var(--text);
  margin: 0 2px;
}
.ob-footer {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-top: 24px;
  gap: 12px;
}
.ob-counter {
  font-size: 12px;
  color: var(--text-secondary);
  letter-spacing: 1px;
}
.ob-actions { display: flex; gap: 8px; }
.ob-back {
  background: transparent;
  color: var(--text-secondary);
  border: 1px solid var(--text-secondary);
  padding: 9px 18px;
  border-radius: 999px;
  font-weight: 600;
  font-size: 14px;
  cursor: pointer;
  font-family: inherit;
}
.ob-back:hover { color: var(--text); border-color: var(--text); }
.ob-next {
  background: var(--primary);
  color: var(--card);
  border: none;
  padding: 10px 22px;
  border-radius: 999px;
  font-weight: 700;
  font-size: 14px;
  cursor: pointer;
  font-family: inherit;
  box-shadow: 0 4px 14px rgba(0, 0, 0, 0.25);
  transition: transform 0.15s ease;
}
.ob-next:hover { transform: translateY(-1px); }
.ob-next:active { transform: translateY(1px); }
```

### Step navigation logic

```js
(() => {
  const STEPS = 3;
  let cur = 1;
  const overlay = document.getElementById('onboarding-overlay');
  const back    = document.getElementById('ob-back');
  const next    = document.getElementById('ob-next');
  const counter = document.getElementById('ob-counter');
  const close   = document.getElementById('ob-close');

  const show = (n) => {
    for (let i = 1; i <= STEPS; i++) {
      document.getElementById(`ob-step-${i}`).hidden = (i !== n);
    }
    back.hidden = (n === 1);
    next.textContent = (n === STEPS) ? 'Get started' : 'Continue';
    counter.textContent = `${n} of ${STEPS}`;
  };
  const dismiss = () => { if (overlay && overlay.parentNode) overlay.remove(); };

  next.addEventListener('click', () => {
    if (cur < STEPS) { cur++; show(cur); }
    else dismiss();
  });
  back.addEventListener('click', () => { if (cur > 1) { cur--; show(cur); } });
  close.addEventListener('click', dismiss);
  window.addEventListener('keydown', (e) => {
    if (!overlay.parentNode) return;
    if (e.key === 'Escape')   { e.preventDefault(); dismiss(); }
    if (e.key === 'PageDown') { e.preventDefault(); if (cur < STEPS) { cur++; show(cur); } else dismiss(); }
    if (e.key === 'PageUp')   { e.preventDefault(); if (cur > 1)     { cur--; show(cur); } }
  }, { capture: true });

  show(1);
})();
```

### Authoring rules (the skill writes these per app, NOT a template substitution)

For each generated app the skill MUST write fresh values for these slots:

1. **Step 1 — emoji + app name** sub-heading, **one-sentence description**, **mode-toggle note**.
2. **Step 2 — bullet list**: one `<li>` per signal binding the app actually wires (gesture taps, button holds, pressure squeezes, navigation deltas, nav_direction arrows, IMU tilt, SNC muscle activity). Each bullet uses a bold action label that fits the app concept (e.g. **Change channel** instead of **Up/Down arrow**), followed by the gesture/key hints in `<span class="kbd">…</span>` and a one-line plain-English description.
3. **Step 3 — practice note**: the simulator panel + keyboard list is also tailored to the actual signal set. Skip signals the app does not subscribe to.

The skill MUST NOT emit a single generic onboarding for every app. Reviewers should be able to identify the app from step 1 and verify the signal coverage from step 2.

### Hero / header media

**Out of scope for v1.2.0.** Do NOT add an `<img>`, `<video>`, or 3D-render preview to the onboarding panel. The XR Blocks default modal shown to the user includes a hero image; the Mudra Band variant intentionally omits it for now. (Future v1.3.0+ may add one.)

### Pre-write checklist additions (referenced from §8)

When verifying generated output, in addition to the §8 items 1–12, also check:

- The overlay DOM exists with id `onboarding-overlay`.
- Exactly three `.ob-body` step containers (`#ob-step-1`, `#ob-step-2`, `#ob-step-3`).
- The step counter shows `1 of 3` on first render.
- The `Continue` button advances; on the last step its label becomes `Get started` and it dismisses the overlay.
- The X button, the `Escape` key, and the final-step CTA all dismiss the overlay.
- `PageUp` / `PageDown` navigate between steps while the overlay is visible (using these instead of arrow keys avoids clashing with Direction-mode `nav_direction` bindings).
- Step 2's bullet list contains exactly one `<li>` per signal binding the app handles (no more, no less).
- No hero `<img>` / `<video>` / `<canvas>` inside the panel.
- The overlay does NOT use a `localStorage` flag to skip future loads.

### AI-app extension — insert step 1.5 "AI Setup" (mandatory when `usesAI`)

For AI apps only, the overlay grows to **four** `.ob-body` containers
(`#ob-step-1`, `#ob-step-ai`, `#ob-step-2`, `#ob-step-3`). The step
counter shows `1 of 4` … `4 of 4` (or skip the counter from the AI step;
either is fine as long as it is consistent).

#### `#ob-step-ai` DOM

```html
<section class="ob-body" id="ob-step-ai" hidden>
  <h3>🔑 AI Setup</h3>
  <p class="ob-lede">
    This app uses Google Gemini. Paste your API key to continue —
    it lives only in this browser tab (<code>sessionStorage</code>)
    and is sent only to Google's API.
    <a href="https://aistudio.google.com/" target="_blank" rel="noopener">Get a key →</a>
  </p>
  <input
    id="ob-ai-key"
    type="password"
    autocomplete="off"
    spellcheck="false"
    placeholder="Paste Gemini API key (starts with AIza…)"
    aria-label="Gemini API key"
  />
  <p class="ob-ai-hint" data-role="hint"></p>
</section>
```

Bespoke CSS additions (inside the existing onboarding `<style>` block,
using the same palette variables already in use):

```css
#ob-step-ai input {
  width: 100%; box-sizing: border-box;
  padding: 10px 12px; border: 1px solid var(--text-secondary);
  border-radius: 8px; background: var(--bg); color: var(--text);
  font: 13px/1.4 ui-monospace, SFMono-Regular, Menlo, monospace;
}
#ob-step-ai input:focus { outline: 2px solid var(--primary); border-color: var(--primary); }
#ob-step-ai .ob-ai-hint { margin: 6px 2px 0; font-size: 0.75rem; color: var(--error); min-height: 1em; }
.ob-cta:disabled { opacity: 0.45; cursor: not-allowed; }
```

#### JS wiring (extends the existing multi-step IIFE)

```js
const KEY_NAME  = "mudra.gemini.apiKey";
const KEY_REGEX = /^AIza[\w-]{30,}$/;

const hasValidStoredKey = () =>
  KEY_REGEX.test((sessionStorage.getItem(KEY_NAME) || "").trim());

const aiStep = document.getElementById("ob-step-ai");
if (aiStep) {
  const input = aiStep.querySelector("#ob-ai-key");
  const hint  = aiStep.querySelector('[data-role="hint"]');
  const cta   = document.querySelector("#onboarding-overlay .ob-cta");

  // If a valid key is already stored, skip the AI step entirely on this load.
  if (hasValidStoredKey()) {
    aiStep.dataset.skip = "true";
  } else {
    const refresh = () => {
      const v = input.value.trim();
      const ok = KEY_REGEX.test(v);
      cta.disabled = (currentStep === "ob-step-ai") ? !ok : cta.disabled;
      hint.textContent = (!v || ok) ? "" : "Key should start with \"AIza\" and be ~39 chars.";
    };
    input.addEventListener("input", refresh);

    // When the user clicks Continue on the AI step, persist the key.
    cta.addEventListener("click", () => {
      if (currentStep !== "ob-step-ai") return;
      const v = input.value.trim();
      if (KEY_REGEX.test(v)) sessionStorage.setItem(KEY_NAME, v);
    }, { capture: true });

    // Block dismissal while the key is missing.
    const blockClose = (e) => {
      if (hasValidStoredKey()) return;
      e.preventDefault(); e.stopPropagation();
      input.focus();
    };
    document.querySelector("#onboarding-overlay .ob-close")?.addEventListener("click", blockClose, { capture: true });
    window.addEventListener("keydown", (e) => {
      if (e.key === "Escape" && !hasValidStoredKey()) blockClose(e);
    }, { capture: true });
  }
}
```

The step-order array used by the multi-step navigator MUST include
`"ob-step-ai"` after `"ob-step-1"`. The skill flag `usesAI` is the only
condition under which `#ob-step-ai` is rendered into the DOM in the
first place — non-AI apps emit the original three-step overlay
unchanged.

---

## §22 — Visible AI Chat I/O (mandatory when `usesAI`)

Every AI-using app MUST render the conversation as on-screen text inside
the 3D scene — TTS / `speechSynthesis` is optional, never a substitute.

### Required visible elements

| Element | What it shows | How to render |
|---------|---------------|---------------|
| **Purpose line** | One short sentence stating what the app does, authored from the user's prompt at generation time. Visible at all times. | Troika `Text` or a top row in an `xb.SpatialPanel`. |
| **User input echo** | The latest user message (speech transcript OR typed input). Updates immediately on capture. | Highlighted/bordered row in the panel, prefixed `💬 You: …`. |
| **AI response** | The most recent AI reply in full readable text. Wraps / scrolls. | `xb.ScrollingTroikaTextView` or a tall row in the panel, prefixed `🤖 AI: …`. |
| **Listening / Thinking indicator** | Idle / listening / thinking states distinguishable at a glance. | Single-word status text + avatar pulse. |

### Rules

1. **Both sides of every exchange must be visible.** Voice-only output
   is a checklist failure. The chat panel must accumulate at least the
   last user turn AND the last AI turn simultaneously.
2. **The Purpose line is fixed** for the lifetime of the app. Author it
   from the user's original prompt — never use a generic placeholder.
3. **Typed input fallback**: when `webkitSpeechRecognition` /
   `SpeechRecognition` are missing, render an `<input type="text">` in
   an XR Blocks panel OR a 2D overlay below the simulator panel. Both
   echo and reply still render in the 3D scene.
4. **No-key placeholder**: when
   `sessionStorage.getItem('mudra.gemini.apiKey')` is `null`, the chat
   panel renders the literal text `Set up AI in the welcome panel` in
   the response slot — do not call Gemini.
5. **TTS** (`speechSynthesis`) is allowed but optional. If used, it
   speaks the AI response in addition to displaying it. Never as a
   replacement.

### Pre-write checklist additions (referenced from §8)

- The chat panel exists in the 3D scene and renders BOTH user input
  echo AND AI response text.
- The Purpose line is non-empty and bespoke (not "AI Chat" or another
  generic).
- No code path calls Gemini when the key is `null`.
- No code path replaces visible text with TTS-only output.
