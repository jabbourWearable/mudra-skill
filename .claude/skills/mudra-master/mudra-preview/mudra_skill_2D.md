---
name: mudra-preview
version: 1.0.0
description: Generate a working Mudra Band interactive app preview as a single-file HTML. Use when the user describes a Mudra-controlled experience (gesture, pressure, navigation, IMU, SNC), wants to prototype a Mudra Companion app, or asks to build/preview a Mudra app.
---

# Mudra Preview

Generate a complete, working single-file HTML app controlled by Mudra Band signals.

## Steps

1. **Read the full instructions** from `references/promt.md` inside the skill base directory. That file contains the complete protocol contract, signal compatibility rules, mock WebSocket implementation, build defaults, and sample catalog — follow all of it.

2. **Infer intent** from the user's description (or the args passed to this skill). Fill gaps with smart defaults. Ask only if there is genuine ambiguity (e.g., navigation vs IMU conflict).

3. **Select the best-matching template** from `assets/` inside the skill base directory. Use the selection rule from `references/promt.md` (motion mode → interaction pattern → signal overlap).

4. **Generate the app** as a single self-contained HTML file:
   - Follow every rule in `references/promt.md` (protocol contract, signal compatibility, mock WebSocket, theme, fallbacks)
   - Choose a color palette that matches the concept — never default to dark unless the concept calls for it
   - Include the Mudra badge, connection indicator, telemetry HUD, simulator panel (see below), and keyboard fallbacks

### Simulator Panel (Required)

Every generated app MUST include a **compact, always-visible simulator panel** with one button set per subscribed signal. The user must be able to trigger every signal with a click — so they can test without the band, or alongside it.

**Layout rules**
- One thin horizontal strip (usually in or just below the header, or pinned to the bottom edge).
- **Always visible** — do NOT hide when the band is connected.
- Group by signal. Use short labels so it stays compact (e.g., `Tap`, `2Tap`, `Twist`, `↑`, `↓`, `Roll L`).
- Reuse the existing `.btn` style from the chosen template.
- No drawers, no collapse, no accordion — one click to fire.

**Buttons per signal** (only render the groups for signals the app actually subscribes to)

| Signal | Buttons |
|---|---|
| `gesture` | `Tap`, `2Tap`, `Twist`, `2Twist` |
| `nav_direction` | `↑`, `↓`, `←`, `→`, `Roll L`, `Roll R` |
| `navigation` | `↑`, `↓`, `←`, `→` (each click emits one delta event of ±8) |
| `pressure` | Slider `0–100%` (or `−` / `+` buttons if horizontal space is tight) |
| `button` | `Press`, `Release` |
| `imu_acc` | `Tilt X`, `Tilt Y`, `Tilt Z` (each fires a 5-frame burst at ±2 m/s²) |
| `imu_gyro` | `Rot X`, `Rot Y`, `Rot Z` (each fires a 5-frame burst at ±10 deg/s) |
| `snc` | `Spike` (injects a burst of elevated samples on all 3 channels) |

**How each button must fire**
- `gesture` buttons → `ws.send(JSON.stringify({ command: 'trigger_gesture', data: { type: '<name>' } }))`. This works both when connected to the real band (round-trips through the service) and when the mock is active (mock echoes it back).
- **All other signals** → emit locally by calling the same handler `ws.onmessage` would dispatch. Example:
  ```js
  function simDir(direction) {
    // same path the real signal would take
    handleNavDirection({ direction, timestamp: Date.now() });
    hudDir.textContent = direction;
  }
  ```
  Always update the telemetry HUD from the sim button, same as the real signal would.

5. **Save the output** to `preview/<concept-name>.html` in the project root (create the `preview/` directory if it doesn't exist). Use a short kebab-case name derived from the concept (e.g., `preview/drum-machine.html`).

6. **Report the file path** so the user can open it in a browser immediately.

## Quick Reference

- WebSocket endpoint: `ws://127.0.0.1:8766`
- Always wrap with `MudraWebSocket` — never raw `new WebSocket(...)`
- Subscribe one signal per command: `{ "command": "subscribe", "signal": "<name>" }`
- Motion modes are mutually exclusive: Pointer (`navigation`+`button`) / Direction (`nav_direction`) / IMU (`imu_acc`+`imu_gyro`)
- All other signals (`gesture`, `pressure`, `snc`, `battery`) combine freely with any mode
