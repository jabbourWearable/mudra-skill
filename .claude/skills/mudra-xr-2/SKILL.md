---
name: mudra-xr-2
description: Generate a single-file Mudra-controlled 3D/XR app using XR Blocks, with the Mudra protocol sourced from the v2.0 agent_protocol.json. Use when the user describes a 3D, XR, VR, or AR experience controlled by the Mudra Band.
metadata:
  version: 1.3.0
---

# Mudra XR-2 Skill

Generate a complete, working single-file HTML 3D/XR app controlled by Mudra Band signals.
The output runs in any modern Chromium browser — no build step, no server, no headset required.

This skill is the v2.0-agent-protocol successor to the existing `mudra-xr` skill. The two skills coexist; this one is fully self-contained — every Mudra-protocol decision traces to `references/agent_protocol.json` (byte-identical to the v2.0 upstream), and every XR Blocks decision traces to `references/xrpromt.md` (byte-identical to the repo-root corpus).

The canonical protocol is in `references/agent_protocol.json` (v2.0). The XR Blocks corpus is in `references/xrpromt.md`. Long-form build rules live in `references/xr_build_rules.md` and are referenced below by section number.

## Mandatory features

**Mandatory feature (v1.0.0):** Every generated app MUST include a Manual/Mudra **Mode toggle** as defined in `references/xr_build_rules.md` §18 "Mode Toggle DOM + Behavior" and §4 "MudraClient class source". Manual is the default; Mudra opens a single WebSocket lazily and disables the simulator panel so signals come only from the band.

**Mandatory feature (v1.0.0):** Connection state MUST reflect the **band**, not the WebSocket. The Companion service accepts socket connections even when no band is paired, so flipping to "Connected" on `ws.onopen` is a lie. Every generated app MUST send `{command:"get_status"}` on open and poll it every 2 s while in Mudra mode, and only show "Connected" when the response has `data.device.state === "connected"`. See `references/xr_build_rules.md` §19 "Status Indicator + Band-State Polling".

**Removed in v1.0.0:** Do **not** render a "Band disconnected" overlay, toast, banner, or any other separate disconnect alert. Connection state is communicated **only** through the existing connection-status pill (which shows "Manual" / "Connecting…" / "Connected" / "Disconnected"). No extra notice, no banner, no modal. See `references/xr_build_rules.md` §19.

**Branding (v1.0.0):** The footer/badge text MUST be exactly **"Created by Mudra"** — never "Created with Mudra Studio", never "Powered by Mudra", never any other variant. Source: `agent_protocol.json.badge` and `references/xr_build_rules.md` §12.

**Contextual simulator panel (v1.0.0):** The simulator panel is always visible but renders **only** the sub-actions the generated app actually handles (e.g. omit Left/Right/Roll if `nav_direction` only handles Up/Down; omit Twist/2Twist if `gesture` only handles `tap`). Unused buttons are a pre-write checklist failure. See `references/xr_build_rules.md` §5.

**Mandatory feature (v1.2.0):** Every generated app MUST render a **combined top-center top bar** containing the mode toggle on the left and the connection pill on the right, both inside one pill-shaped container. See `references/xr_build_rules.md` §18 "Mode Toggle DOM + Behavior" and §19 "Status Indicator + Band-State Polling". **Removed in v1.2.0:** the separate top-left `#mode-toggle` element and the separate top-right `#mudra-status` element from v1.0.0 — they are now merged into one `#topbar` element.

**Mandatory feature (v1.2.0, updated v1.3.0):** Every generated app MUST show a **multi-step onboarding overlay on every page load** (no localStorage skip). The overlay is a centered modal panel with the title `Welcome to Mudra Band` in the top-left, an X close button in the top-right, exactly **three steps** (Welcome → Controls → Practice without a band), a step counter and `Back` / `Continue` (final-step label: `Get started`) buttons in the bottom-right. The bullet list in step 2 is **derived from the actual subscribed signals** the generated app handles — bespoke per app, not a template substitution. **Removed in v1.3.0**: the v1.2.0 single-panel onboarding with one CTA. **Out of scope in v1.3.0**: hero image / video at the top of the panel — to be added in a future version. See `references/xr_build_rules.md` §21 "Onboarding Overlay (every load, multi-step)".

**Mandatory feature (v1.2.0):** Each generated app MUST use a **bespoke palette appropriate to its concept** — not the universal Mudra dark theme from earlier versions. The CSS custom-property *names* (`--bg`, `--card`, `--primary`, `--accent`, `--text`, `--text-secondary`, `--success`, `--warning`, `--error`) remain canonical so styling logic works across apps, but the *values* are picked per app by the skill at generation time (e.g. pastel for a casual game, neon for a racing app, calm muted tones for a meditation timer). See `references/xr_build_rules.md` §12 "Theme + Badge" for palette constraints. The font remains Poppins.

**Versioning convention (v1.1.0):** Mandatory features above carry their introduction version. When this skill's `version` field is bumped, new mandatory features land here with their version stamp, and any superseded behavior is marked "Removed in vX.Y.Z" so reviewers can trace each requirement.

## Signal Compatibility (Non-Negotiable)

### Signal groups — pick ONE per app

- **Pointer mode**: `navigation` + `button` (continuous `delta_x` / `delta_y` cursor / pan / steer)
- **Direction mode**: `nav_direction` (discrete swipes — Up / Down / Left / Right / Roll Left / Roll Right)
- **IMU+Biometric bundle**: `imu_acc` + `imu_gyro` + `snc` (always all three together — never partial)
- **None**: no motion mode at all (apps driven only by `gesture` / `pressure` / `button` / `battery`)

The remaining additive signals (`gesture` OR `pressure`, plus `button`, plus `battery`) combine freely with the chosen motion mode (or with no motion mode) subject to the XOR rules below.

### XOR rules (all non-negotiable)

1. **`gesture` ⊕ `pressure`** — pick one; never combine them. Source: `agent_protocol.json.compatibility.cannot_combine[0]`.
2. **`navigation` ⊕ `nav_direction`** — pick one; never combine them. Source: `agent_protocol.json.compatibility.cannot_combine[1]`.
3. **(`navigation` or `nav_direction`) ⊕ IMU+Biometric bundle** — directional motion signals cannot be combined with the IMU+Biometric bundle (`imu_acc` / `imu_gyro` / `snc`). Source: `agent_protocol.json.compatibility.cannot_combine[2–7]`.

### IMU+Biometric bundling rule

`imu_acc`, `imu_gyro`, and `snc` are an **inseparable bundle**. If the user's prompt implies any one of them, subscribe to **all three**. Never subscribe to one or two of them alone. Source: `agent_protocol.json.signal_inference.bundling_rule` + `compatibility.bundling_rules[0]`.

### Disambiguation procedure

If the prompt maps to two **incompatible** groups (e.g. both `gesture` and `pressure`, or both directional motion and the IMU+Biometric bundle), ask **exactly one** disambiguation question — never silently pick one of two. This is the only case where a question is warranted; otherwise pick smart defaults and proceed.

For full compatibility tables (signal groups, valid combinations, every `cannot_combine` entry with its stated reason), see `references/xr_build_rules.md` §10 "Compatibility Rules + Valid Combinations".

## Invocation

This skill activates when the user:

- Explicitly runs `/mudra-xr-2 <prompt>`
- Describes a 3D, XR, VR, or AR experience involving the Mudra Band
- Uses trigger phrases: "3D with Mudra", "XR Mudra app", "WebXR + Mudra", "Mudra in VR / AR", "XR Blocks + band", or close paraphrases
- Mentions depth, stereo, hand tracking, or spatial context alongside Mudra

Decline (do not generate) when:

- The prompt is unambiguously 2D (flat/screen, dashboard, web page). Tell the user to use `/mudra-preview` or `/mudra-master` instead.
- The prompt is unrelated to Mudra-controlled app generation.

The skill never modifies any file under `.claude/skills/mudra-xr/`, `.claude/skills/mudra-master/`, or `gemini-gem/`. The eventual deletion of the old `mudra-xr` skill and any router cut-over are out of scope.

## Steps

### 1. Read build rules

Read the complete `references/xr_build_rules.md` from the skill base directory. That file is the source of truth for signal protocol, dependency pins, the `MudraClient` class, simulator panel layout, keyboard shortcuts, mode toggle, status indicator, pre-write checklist, background catalog, AI gating, and seed lookup procedure. Follow every rule in it.

Read the vendored `references/agent_protocol.json` for any protocol detail not summarized in `xr_build_rules.md`. The vendored JSON is the canonical source for signal payload shapes, valid combinations, compatibility rules, creative proposals, theme colors, and badge HTML/CSS.

### 2. Parse input

Extract from the user's prompt:

- **`templateOverride`**: an inline `[template=<id>]` tag (e.g., `[template=2_hands]`). Strip from the prompt before further inference.
- **`bgOverride`**: an inline `[bg=<id>]` tag (e.g., `[bg=starfield]`). Strip from the prompt.
- **`motionModeHint`**: an inline `[mode=pointer|direction|imu|none]` tag. Strip from the prompt.

If duplicate keys appear in the prompt, the last occurrence wins.
Unknown ids in any of the three tags MUST be rejected with the catalog's valid-id list (`xr_build_rules.md` §13 for templates, §15 for backgrounds, modes are exactly `pointer|direction|imu|none`).

### 3. Infer Mudra signals

Map the user's intent to Mudra signals using `references/agent_protocol.json` `signal_inference.mapping` — see `xr_build_rules.md` §9. Enforce all compatibility rules from `xr_build_rules.md` §10 (mirrored from `agent_protocol.json.compatibility`):

- Discrete actions → `gesture` OR `button` (never both gesture+pressure)
- Analog control → `pressure` OR `button` (never both gesture+pressure)
- Continuous directional movement → `navigation` + `button` (Pointer mode)
- Discrete directional swipes → `nav_direction` (Direction mode) — see §3
- Tilt / orientation / biometrics → `imu_acc` + `imu_gyro` + `snc` (always all three together — IMU+Biometric bundle)

**Critical grouping rules** (from `agent_protocol.json.compatibility.bundling_rules`):

1. `gesture` and `pressure` are mutually exclusive — pick one.
2. `navigation` and `nav_direction` are mutually exclusive — pick one.
3. The IMU+Biometric bundle (`imu_acc` + `imu_gyro` + `snc`) cannot combine with `navigation` or `nav_direction`.
4. `imu_acc`, `imu_gyro`, and `snc` are always subscribed together — using any one requires all three.

If the prompt maps to two incompatible groups, ask **one** disambiguation question — never silently pick. This is the only place a question is warranted (`agent_protocol.json.core_behavior.rules`).

If the user's concept maps cleanly to one signal, optionally propose one complementary signal per `xr_build_rules.md` §11 (mirroring `agent_protocol.json.creative_proposals`). Honor the `forbidden_proposals` list.

### 4. Pick motion mode

Exactly one of: `pointer`, `direction`, `imu`, or `none`.
If `motionModeHint` is present, use it. If Step 3's inference is unambiguous, use that.
Default to `direction` if there is motion-language in the prompt but no clear winner.

### 5. Select seed template

**If `templateOverride` is present**: use the named template directly. Resolve the id against the catalog in `xr_build_rules.md` §13. If the id does not match any row, reject with the valid-id list.

**Keyword-based selection (no override)**: score every row in §13's catalog against the user's lowercased prompt using the algorithm in §13 (signal overlap +3, motion mode match +5/+2, XR feature match +2, keyword overlap +1). Pick the highest-scoring row. Ties resolved by: prefer `direction` motion mode → prefer template over sample over demo → prefer `0_basic` as universal fallback.

**AI-enabled seeds** (`6_ai`, `7_ai_live`, `gemini-icebreakers`, `gemini-xrobject`, `aisimulator`, and any other AI-enabled row in §13) MUST be selected only when the user prompt explicitly names AI / Gemini / LLM behavior, OR when `templateOverride` forces it. See `xr_build_rules.md` §17.

### 5a. Locate seed HTML in vendored `xrpromt.md`

Use the heading-anchored lookup from `xr_build_rules.md` §14:

1. Open `references/xrpromt.md`.
2. Locate the heading `### Template: <id>`, `### Sample: <id>`, `### Demo: <id>`, or `### Gallery: <id>` matching the selected id.
3. Capture the first ```` ```html ... ``` ```` fenced block that follows the heading.
4. That block is the seed `source_html` to adapt.

### 5b. Select background

**Default: no background helper applied** — the XR Blocks default studio room renders, which is what almost every seed template in `xrpromt.md` does and what users expect. Do NOT inject a custom dome/floor unless the user explicitly asks for one. Theme cues like "in space", "at night", "cyberpunk vibe" are NOT background requests.

Follow `xr_build_rules.md` §15:

1. Inline override `[bg=<id>]` — opt in to the named catalog entry verbatim, no inference.
2. Explicit naming by the user (e.g. "starfield background", "with a sunset sky", "solid studio backdrop") — opt in to that catalog entry.
3. Anything else → **no `applyBackground_*` method**, no call from `init()`, no `scenePath` override. XR Blocks renders its default room.

### 6. Adapt the template with Mudra bindings

Starting from the seed HTML:

1. Add the `MudraClient` class (`xr_build_rules.md` §4) verbatim inside `<script type="module">`. Manual default; no auto-connect; `setMode()`-driven; passive mock; 2-second `get_status` poll while in Mudra mode.
2. Instantiate `mudra` at module scope.
3. **Background**: if step 5b selected a catalog entry, copy the `applyBackground_<id>()` body from `xr_build_rules.md` §15 into the `xb.Script` subclass. If step 5b selected "no background helper" (the default for most prompts), skip this — do NOT add any `applyBackground_*` method.
4. If a background helper was added in step 3, call `this.applyBackground_<id>()` as the **first line** of `init()` — before lights, meshes, or Mudra wiring. If no helper was added, skip this — `init()` starts directly with the scene content.
4a. Configure XR Blocks room per §15:
   - **No background helper (default path)**: leave `options.simulator.scenePath` alone. XR Blocks renders its default room — this is what nearly every seed template in `xrpromt.md` does.
   - **Immersive backgrounds** (`starfield`, `gradient_sky`, `grid_cyber`, `skybox_texture`): add `options.simulator.scenePath = null;` before `xb.init(options)` to hide the default room so the dome is the full environment.
   - **`solid_studio`** (opt-in): do **not** set `scenePath`. The default room provides UI-friendly spatial grounding behind the studio dome.
5. Call `mudra.subscribe('<signal>')` for every required signal inside `init()`.
6. Wire `mudra.on('<signal>', handler)` for each subscribed signal using the binding patterns in `xr_build_rules.md` §3 (Direction mode), and the per-signal best practices in `agent_protocol.json.signals.<name>.best_practices` for `gesture` and `button`.
7. Add the **combined top-center top bar** `<div id="topbar">` (`xr_build_rules.md` §18 + §19) containing the mode toggle on the left (`Manual` + `Mudra` buttons, Manual default) and the connection pill on the right. Wire both buttons via `mudra.setMode(...)`. The connection pill text is `Manual mode` / `Connecting…` / `Connected` / `Disconnected`. **Never render a separate disconnect overlay/banner/toast.**
8. Add the simulator panel `<div id="mudra-sim">` (`xr_build_rules.md` §5) as a centered pill bar at the bottom, with one button group **only for the sub-actions the app actually handles** (omit Roll L/Roll R if `nav_direction` only handles Up/Down; omit Twist/2Twist/2Tap if `gesture` only handles `tap`). The panel is greyed (`opacity: 0.4`, `pointer-events: none`) while in Mudra mode.
9. Add the **onboarding overlay** `<div id="onboarding-overlay">` (`xr_build_rules.md` §21) as a centered modal that appears on every page load. The skill MUST author the title (emoji + name), one-sentence description, and primary-control instructions **bespoke to the app concept** at generation time — not template-substituted. Include the keyboard hint via inline `<span class="kbd">…</span>`. A single CTA button dismisses the overlay (removes it from the DOM).
10. Add the keyboard handler `window.addEventListener('keydown', …, { capture: true })` (`xr_build_rules.md` §6) for every subscribed signal. Call `event.stopPropagation()` on Mudra-claimed keys. Do **not** bind any key in `xr_build_rules.md` §7 (XR Blocks reserved keys), except the Direction-mode exception for arrow keys when `nav_direction` is the motion mode.
11. Add the footer badge `<div class="mudra-badge">` using the literal text `Created by Mudra` (`xr_build_rules.md` §12). The badge styling uses the bespoke palette CSS variables — never variants of the text.
12. **Pick a bespoke palette appropriate to the app concept** (`xr_build_rules.md` §12) and emit it as a `:root { --bg: …; --card: …; --primary: …; --accent: …; --text: …; --text-secondary: …; --success: …; --warning: …; --error: … }` block at the top of `<style>`. The variable *names* are canonical; the *values* are bespoke per app. Apply them to `#topbar`, `#mudra-sim`, `#onboarding-overlay`, `.mudra-badge`, and any app-specific chrome.
13. Strip import-map entries the adapted app does not actually `import` (`xr_build_rules.md` §16). All retained URLs must match the Reference Map in `references/xrpromt.md` verbatim. **Never strip the four mandatory entries** (`three`, `three/addons/`, `xrblocks`, `xrblocks/addons/`).

### 7. Run the pre-write checklist

Verify all 12 items in `xr_build_rules.md` §8 before writing. If any item fails: regenerate once and re-check. If the second attempt also fails: surface the failing items to the user; do not write.

### 8. Write the output file

Target path: `preview/<concept-name>.html` (short kebab-case derived from the concept).
If that file exists: try `preview/<concept-name>-2.html`, then `-3.html`, etc.
Never overwrite. Create `preview/` if it does not exist.

### 9. Report

Print the absolute path to the written file and a one-line summary:

```
Wrote <absolute-path>
  template: <id>   motion: <pointer|direction|imu|none>   signals: <comma-separated subscribed signals>
```

If bundling rules forced extra subscriptions (e.g. user asked for `snc` only and got the full IMU+Biometric bundle), append a one-line note explaining why.

## Quick reference

- WebSocket endpoint: `ws://127.0.0.1:8766` (matches `agent_protocol.json.connection_urls.websocket`)
- Always use `MudraClient` from `xr_build_rules.md` §4 — never raw `new WebSocket(...)`
- Subscribe one signal per command: `{ command: 'subscribe', signal: '<name>' }` — key `signal` (singular), never `signals` (plural)
- Motion modes are mutually exclusive: Pointer (`navigation`+`button`) / Direction (`nav_direction`) / IMU+Biometric (`imu_acc`+`imu_gyro`+`snc`)
- IMU+Biometric bundle: `imu_acc`, `imu_gyro`, `snc` always subscribed together — never partially
- `gesture` and `pressure` are mutually exclusive — never combine them
- Free-combining signals: `gesture` OR `pressure`, plus `button`, `battery`
- Navigation sensitivity is gentle by default: sim button + keyboard `I`/`J`/`K`/`L` emit `±3` per event; cursor multiplier on inbound `delta_x`/`delta_y` is `0.002`. Raise only when the prompt explicitly asks for fast/snappy movement.
- **Reserved for XR Blocks desktop simulator** — Mudra never claims these: `W`/`A`/`S`/`D` and arrow keys (camera walk, *except* in Direction mode where arrows are allowed for `nav_direction`), `Q`/`E` (roll/vertical), `R` (reset), right-click drag (orbit), mouse wheel (zoom). Mudra navigation uses `I`/`J`/`K`/`L`; Mudra IMU uses `U`/`O`/`M`/`N`. See `xr_build_rules.md` §7.
- Keyboard handlers: `{ capture: true }` + `stopPropagation()` on Mudra-claimed keys
- Import map: exact pinned versions from `xrpromt.md` §"Reference Map" (lines 21–32 of the vendored file). Mandatory pairing: `xrblocks@0.10.0` + `three@0.182.0`. Never use `xrblocks@0.11.0` (it requires a `three/addons/` layout that breaks against `three@0.182.0`). Never strip the four mandatory entries `three`, `three/addons/`, `xrblocks`, `xrblocks/addons/` — xrblocks imports from them transitively. See §2 and §16 of `xr_build_rules.md`.
- Output: one `.html` file in `preview/`, zero external local references, no embedded API keys (AI-enabled apps ship with `const API_KEY = "";`).
