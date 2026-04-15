# Signal Inference

Use this as the default behavior for intent-to-signal mapping.

## Rule

- Map user intent to signals from context.
- Do not ask signal-selection questions when intent is clear.
- Ask only when there is genuine ambiguity.

## Mapping

- `gesture`: tap, click, trigger, action, button press, drum, hit, select
- `button`: hold, press and hold, drag, push-to-talk, sprint, charge
- `pressure`: slide, volume, size, intensity, throttle, opacity, brush, zoom, analog
- `navigation`: move, up/down, left/right, steer, cursor, pan, scroll, direction, arrow
- `nav_direction`: swipe, directional gesture, menu direction, card swipe, flick — directions: None, Right, Left, Up, Down, Roll Left, Roll Right (+ reverse variants)
- `imu_acc + imu_gyro`: tilt, orientation, angle, rotate, 3D, balance, level
- `snc`: muscle, EMG, biometric, fatigue, nerve

## Ambiguity Rules

When concept could map to either `navigation` or `imu_acc + imu_gyro`, ask one clarifying question and recommend the better fit:
- use `navigation` (+`button`) for continuous directional movement/cursor/panning/drag
- use `imu_acc + imu_gyro` for orientation/tilt/rotation

When concept could use either `navigation` or `nav_direction`, pick based on control style:
- use `navigation` (+`button`) for **continuous** pointer/cursor control (smooth deltas)
- use `nav_direction` for **discrete** directional gestures (swipe-like, menu selection)
- these cannot be combined (same physical hand movement)
