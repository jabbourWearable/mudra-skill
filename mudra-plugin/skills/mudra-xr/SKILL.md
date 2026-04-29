---
name: mudra-xr
version: 0.1.0
description: Generate a single-file Mudra-controlled 3D/XR app using XR Blocks. Use when the user describes a 3D, XR, VR, or AR experience controlled by the Mudra Band.
---

# Mudra XR Skill

Generate a complete, working single-file HTML 3D/XR app controlled by Mudra Band signals.
The output runs in any modern Chromium browser — no build step, no server, no headset required.

## Invocation

This skill activates when the user:
- Explicitly runs `/mudra-xr <prompt>`
- Describes a 3D, XR, VR, or AR experience involving the Mudra Band
- Uses trigger phrases: "3D with Mudra", "XR Mudra app", "WebXR + Mudra",
  "Mudra in VR / AR", "XR Blocks + band", or close paraphrases
- Mentions depth, stereo, hand tracking, or spatial context alongside Mudra

Prefer this skill over `mudra-preview` (2D) when the prompt mentions 3D, XR, VR, AR,
a headset, depth, stereo, or spatial hand tracking.

## Steps

### 1. Read build rules

Read the complete `references/promt.md` from the skill base directory.
That file is the source of truth for signal protocol, dependency pins, MudraClient
implementation, simulator panel, keyboard shortcuts, pre-write checklist, and binding patterns.
Follow every rule in it.

### 2. Parse input

Extract from the user's prompt:
- **`templateOverride`**: an inline `[template=<id>]` tag (e.g., `[template=2_hands]`)
- **`motionModeHint`**: an inline `[mode=pointer|direction|imu]` tag

If neither tag is present, infer them from the prompt language (Step 3).

### 3. Infer Mudra signals

Map the user's intent to the required Mudra signals using the Signal Inference Reference
in `references/promt.md`. Enforce motion-mode exclusivity (Section 8 of promt.md):

- Discrete actions → `gesture` or `button`
- Analog control → `pressure`
- Continuous directional movement → `navigation` + `button` (Pointer mode)
- Discrete directional swipes → `nav_direction` (Direction mode)
- Tilt / orientation → `imu_acc` + `imu_gyro` (IMU mode)
- Biometric → `snc`

If the prompt maps to two motion modes, ask one disambiguation question — do not
auto-pick silently. If no motion mode is needed, use mode = none.

### 4. Pick motion mode

Exactly one of: `pointer`, `direction`, `imu`, or `none`.
If `motionModeHint` is present, use it. If the inference step above is unambiguous, use that.
Default to `direction` if there is motion-language in the prompt but no clear winner.

### 5. Select seed template

**If `templateOverride` is present**: use the named template directly.
Resolve the id against the selection table in Section 12 of `references/promt.md`.
If the id does not match any row, reject with:
```
Unknown template id: "<id>". Valid ids: 0_basic, 1_ui, 2_hands, 3_depth, 4_stereo,
5_camera, 6_ai, 7_ai_live, 8_objects, 9_xr-toggle, heuristic_hand_gestures, meshes,
planes, uikit, depthmap, depthmesh, game_rps, gestures_custom, gestures_heuristic,
lighting, mesh_detection, modelviewer, paint, planar-vst, reticle, skybox_agent,
sound, ui, virtual-screens, 3dgs-walkthrough, aisimulator, balloonpop, ballpit,
drone, gemini-icebreakers, gemini-xrobject, math3d, measure, occlusion, rain,
screenwiper, splash, webcam_gestures, xremoji, xrpoet
```

**Keyword-based selection (no override)**:

Score every row in Section 12's table against the user's prompt using this algorithm:

1. **Signal overlap** (+3 per matching subscribed signal name found in `keywords`)
2. **Motion mode match** (+5 if inferred motion mode is in `motionModesSupported`, +2 if `"none"` is in `motionModesSupported`)
3. **XR feature match** (+2 per matching XR feature keyword from the prompt in `xrFeatures`)
4. **Keyword overlap** (+1 per keyword from the `keywords` array that appears in the lowercased prompt)

Pick the highest-scoring row. Ties resolved by:
1. Prefer `direction` motion mode over others (most app-agnostic)
2. Prefer templates over samples, samples over demos (simpler baseline)
3. If still tied, pick `0_basic` as the universal fallback

**Examples**:
- Prompt with "hands" + "gesture" + no motion mode → `2_hands` or `heuristic_hand_gestures`
- Prompt with "ui" + "panel" + "button" → `1_ui` or `uikit`
- Prompt with "depth" → `3_depth` or `depthmap`
- Prompt with "ai" + "gemini" + "photo" → `6_ai`
- No clear match → `0_basic` (universal fallback)

### 5b. Select background

Pick **exactly one** background from the 5-row catalog in Section 14 of `references/promt.md`:
`starfield`, `gradient_sky`, `solid_studio`, `grid_cyber`, `skybox_texture`.

**Inline override**: if the prompt contains `[bg=<id>]` (e.g., `[bg=starfield]`), use it verbatim.

**Explicit user preference**: if the prompt contains an obvious background phrase
("starfield", "sunset sky", "cyber grid", "in the forest"), pick the matching row regardless of scoring.

**Keyword scoring** (no override, no explicit phrase): use Section 14's scoring —
`+2` per keyword match in the prompt, `+1` for signal fit, `+1` for template pairing.
Tie-break: `solid_studio` → `starfield` → `gradient_sky` → `grid_cyber` → `skybox_texture`.
If every row scores 0, default to `solid_studio`.

**Examples**:
- "solar system" → `starfield`
- "spatial menu" / "ui panel" → `solid_studio`
- "cyberpunk dashboard" → `grid_cyber`
- "sunset meditation" → `gradient_sky`
- "forest walkthrough" → `skybox_texture`

### 6. Adapt the template with Mudra bindings

Starting from the seed template HTML:
1. Add the `MudraClient` class (Section 4 of promt.md) verbatim inside the `<script type="module">`.
2. Instantiate `mudra` at module scope.
3. Copy the chosen `applyBackground_<id>()` method body verbatim from Section 14 into the `xb.Script` subclass.
4. Call `this.applyBackground_<id>()` as the **first line** of `init()` — before lights, meshes, or Mudra wiring.
5. Call `mudra.subscribe('<signal>')` for every required signal inside `init()`.
6. Wire `mudra.on('<signal>', handler)` for each subscribed signal using the binding
   patterns from Section 11 of promt.md.
7. Add the simulator panel `<div id="mudra-sim">` (Section 5) with one button group
   per subscribed signal.
8. Add the status indicator `<div id="mudra-status">` (Section 7) wired to `mudra.on('_status', …)`.
9. Add the keyboard handler `window.addEventListener('keydown', …, { capture: true })`
   (Section 6) for every subscribed signal.
10. Remove import-map entries for dependencies the adapted app does not use.

### 7. Run the pre-write checklist

Verify all 10 items from Section 10 of promt.md before writing.
If any item fails: regenerate once and re-check.
If the second attempt also fails: surface the failing items to the user; do not write.

### 8. Write the output file

Target path: `preview/<concept-name>.html` (short kebab-case derived from the concept).
If that file exists: try `preview/<concept-name>-2.html`, then `-3.html`, etc.
Never overwrite. Create `preview/` if it does not exist.

### 9. Report

Print the absolute path to the written file and a one-line summary:
- Template used
- Motion mode
- Subscribed signals

## Quick reference

- WebSocket endpoint: `ws://127.0.0.1:8766`
- Always use `MudraClient` — never raw `new WebSocket(...)`
- Subscribe one signal per command: `{ command: 'subscribe', signal: '<name>' }`
- Motion modes are mutually exclusive: Pointer (`navigation`+`button`) / Direction (`nav_direction`) / IMU (`imu_acc`+`imu_gyro`)
- Free-combining signals: `gesture`, `pressure`, `snc`, `battery`
- Keyboard handlers: `{ capture: true }` + `stopPropagation()` on Mudra-claimed keys
- Import map: exact pinned versions only — no `@latest`
- Output: one `.html` file in `preview/`, zero external local references
