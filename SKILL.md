---
name: mudra-band-ultimate
version: 1.0.15
description: Build, iterate, and support production-quality Mudra Band experiences using Mudra Companion signals, strict protocol handling, and interaction-first UX systems. Use when requests mention Mudra Band, Mudra Companion, gesture control, pressure/button/navigation/IMU/SNC signals, wearable interaction prototypes, or users need robust webapp demos with fallback controls and real-time feedback.
---

# Mudra Band Ultimate

## Overview

Go from user intent to a reliable Mudra interactive app with strong UX feel, correct protocol usage, and a fast testing loop.

## Workflow

1. Infer intent from context, fill gaps with smart defaults, propose a brief concept, then build. Don't run through a fixed question list — only ask when there's genuine ambiguity (e.g., navigation vs IMU conflict).

2. Select signals and enforce compatibility:
- map discrete actions to `gesture` or `button`
- map analog control to `pressure`
- map directional control to `navigation`
- map directional gestures to `nav_direction`
- map orientation/rotation to `imu_acc + imu_gyro`
- map biometric use cases to `snc` — **note**: SNC data arrives as 3 de-interleaved channel arrays `[[ch1_samples], [ch2_samples], [ch3_samples]]`, not a flat array. Each message contains a batch of samples per channel. Extend rolling buffers (500 samples/channel) with all samples per callback, use latest sample per channel for real-time display.
- follow infer-first rules in `references/signal-inference.md`

3. Build with protocol-safe contract:
- connect `ws://127.0.0.1:8766`
- subscribe one signal per command with `signal` (singular)
- valid signals: `gesture`, `button`, `pressure`, `navigation`, `nav_direction`, `imu_acc`, `imu_gyro`, `snc`, `battery`
- support full command surface: `subscribe`, `unsubscribe`, `get_subscriptions`, `enable`, `disable`, `get_status`, `get_docs`, `trigger_gesture`
- handle `connection_status`
- include fallback controls (keyboard/mouse/touch)
- include no-device simulation path via `trigger_gesture`

4. Apply interaction-quality standards:
- define interaction loop (1-3 second cadence)
- immediate visual/motion feedback per action
- state-driven UI (`idle`, `active`, `success`, `error`)
- motion system (spring/easing-out, no hard cuts)
- target smooth responsiveness and mobile/desktop support
- prioritize feel, responsiveness, and visual feedback over feature completeness

5. Finish with concise testing steps:
- physical-device path
- simulation path
- success criteria

## Signal Compatibility (hard rule)

Three mutually exclusive motion groups — pick ONE per app:
- **Pointer mode**: `navigation` + `button` (continuous cursor/drag + air touch)
- **Direction mode**: `nav_direction` (discrete directional gestures: None, Right, Left, Up, Down, Roll Left, Roll Right)
- **IMU mode**: `imu_acc` + `imu_gyro` (orientation/tilt/rotation)

Never combine:
- `navigation + imu_acc`
- `navigation + imu_gyro`
- `nav_direction + imu_acc`
- `nav_direction + imu_gyro`
- `navigation + nav_direction` (same physical hand movement)
- `button + nav_direction` (button is part of pointer mode)

All other signals (`gesture`, `pressure`, `snc`, `battery`) combine freely with any group.

When conflict appears, explain limitation and recommend one path.

## Build Defaults

- Default platform: webapp.
- Default style: Mudra dark theme.
- Always include Mudra badge text in output UI.
- Always prefer interactive assets over static decoration.
- Start from `assets/mudra-ultimate-template.html` and modify to match the user's concept rather than writing from scratch. The template is a webapp with connection handling, all signal handlers, fallback controls, a telemetry HUD, and the dark theme.

## References

For standard builds, this SKILL.md provides sufficient guidance. Load references only when needed:

- `references/agent_protocol.live.v2.json`: **Load when exact data formats, architecture patterns, or edge case handling are needed.** Live protocol snapshot with full signal schemas, command specs, theme CSS, code examples, and creative proposal patterns.
- `references/signal-inference.md`: **Load when intent-to-signal mapping is ambiguous.** Infer-first mapping table with the navigation-vs-IMU disambiguation rule.

## Assets

- `assets/mudra-ultimate-template.html`: baseline webapp template with connection handling, telemetry, and fallback controls. Use as the starting point for new apps.
