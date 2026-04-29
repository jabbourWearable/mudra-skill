# Mudra Plugin

This plugin provides skills for generating Mudra Band apps.

Use `/mudra-master` (or just describe an app idea) to build a 2D or 3D/XR app controlled by the Mudra Band wristband.

## Skills

- **mudra-master** — router: classifies prompt as 2D / 3D / ASK / DECLINE, then hands off
- **mudra-preview** — generates single-file HTML 2D apps
- **mudra-xr** — generates single-file HTML 3D/XR apps using XR Blocks

## Canonical Nine-Signal Table

| Signal | Type | Use for |
|---|---|---|
| `gesture` | discrete | finger pinches (index, middle, ring, little, thumb, grab) |
| `button` | discrete | hardware button press/release |
| `pressure` | analog | continuous squeeze force (0–1) |
| `navigation` | pointer | 2D cursor delta (x, y) — Pointer mode |
| `nav_direction` | discrete | swipe direction (up/down/left/right) — Direction mode |
| `imu_acc` | analog | accelerometer (x, y, z) — IMU mode |
| `imu_gyro` | analog | gyroscope (x, y, z) — IMU mode |
| `snc` | analog | 3-channel bio signal, batched arrays per channel |
| `battery` | discrete | battery level |

**Motion-mode exclusivity (non-negotiable):** each app uses exactly one of `navigation` XOR `nav_direction` XOR `imu_acc`+`imu_gyro`. The additive signals (`gesture`, `pressure`, `snc`, `battery`) combine freely with any mode.

## Protocol Rules

- WebSocket at `ws://127.0.0.1:8766`
- Subscribe one signal per command, key `signal` (singular): `{ "command": "subscribe", "signal": "<name>" }`
- Never use `signals` (plural) or batch subscribe
- Always wrap with `MudraWebSocket` class (includes mock fallback)
- Every app must include: mock fallback, always-visible simulator panel, keyboard shortcuts, connection-status indicator
