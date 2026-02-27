# circle.html — Implementation Plan

## Concept
A student moves their index finger in a circle (like swinging a yoyo). The app:
1. Tracks the circular motion using MediaPipe **HandLandmarker** (lighter than GestureRecognizer — no gesture classification needed, just landmark positions)
2. Draws the circular path on screen with angle θ shown
3. Projects the Y position rightward → live scrolling **sine wave**
4. Projects the X position downward → live scrolling **cosine wave**
5. Draws dotted projection lines connecting the circle point to both waves (the key "aha" visual)
6. Plays a sine tone at the rotation frequency via Web Audio API
7. Shows live labels: θ (degrees), ω (rad/s), f (Hz), T (seconds), sin(θ), cos(θ)

---

## Layout

```
┌──────────────────────────────────────────────────────┐
│  [θ]  [ω]  [f]  [T]          ← label panel (HTML)   │
│                                                      │
│  ┌─────────────┐   ┌──────────────────────────────┐  │
│  │             │   │  sin(θ)                      │  │
│  │   CIRCLE    │   │  ~~~~~~~~~~╮~~~~~~~~~~~~~~~~~│  │
│  │   REGION    │···│············●  ← current dot  │  │
│  │      ●      │   │                              │  │
│  └──────│──────┘   └──────────────────────────────┘  │
│         │                                            │
│  ┌──────▼──────┐                                     │
│  │  cos(θ)     │                                     │
│  │  ~│~~│~~│~  │                                     │
│  │   ●         │  ← current dot                      │
│  └─────────────┘                                     │
└──────────────────────────────────────────────────────┘
```

The dotted lines (···) are the **projection lines** — the key pedagogical element that makes the connection explicit.

---

## 1. File & Architecture

Single file `gesture-lab/circle.html`. Structure:

```
<head>  styles </head>
<body>
  #loading overlay
  #ui      — label panel (HTML div, updated via DOM each frame)
  #canvas  — full-screen drawing surface
  <video>  — hidden (input to MediaPipe only, not displayed)

  <script type="module">
    // IMPORTS
    // DOM REFS
    // CONSTANTS
    // STATE
    // AUDIO SYSTEM
    // MATH / DATA PROCESSING
    //   updateCenter(), updateAngle(), pushWaveBuffers()
    // RENDERING
    //   drawCircleRegion(), drawSineRegion(), drawCosineRegion()
    //   drawProjectionLines(), updateLabels()
    // MEDIAPIPE ENGINE
    //   loadModel(), loop()
    // BOOT
  </script>
</body>
```

---

## 2. MediaPipe Setup

Use **HandLandmarker** (not GestureRecognizer):

```javascript
import { HandLandmarker, FilesetResolver } from
  "https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@0.10.14/+esm";

handLandmarker = await HandLandmarker.createFromOptions(vision, {
  baseOptions: {
    modelAssetPath: "https://storage.googleapis.com/mediapipe-models/hand_landmarker/hand_landmarker/float16/1/hand_landmarker.task",
    delegate: "GPU",
  },
  runningMode: "VIDEO",
  numHands: 1,   // only one circling hand needed
});
```

In the loop, only landmark[8] (index fingertip) is used:

```javascript
const result = handLandmarker.detectForVideo(video, Date.now());
if (result.landmarks.length > 0) {
  const tip = result.landmarks[0][8];
  const x = (1 - tip.x) * canvas.width;   // mirror-correct
  const y =      tip.y  * canvas.height;
  processFrame(x, y, Date.now());
}
```

---

## 3. State Variables

```javascript
// Position history (for center estimation)
const HISTORY_LEN = 90;          // ~3s at 30fps
const posHistory  = [];           // [{x, y}]

// Geometry
let centerX = 0, centerY = 0;    // estimated center of circular path (px)
let radius  = 0;                  // current fingertip distance from center (px)
let maxRadius = 1;                // rolling max, for normalization

// Angle
let rawAngle     = 0;             // atan2 result (-π, π]
let prevRawAngle = null;
let cumulAngle   = 0;             // unwrapped (grows continuously)
let thetaRad     = 0;             // normalized [0, 2π)
let thetaDeg     = 0;             // for display

// Angular velocity
let omega        = 0;             // smoothed (rad/s)
let prevTimestamp = null;

// Derived values
let freqHz = 0, period = Infinity;
let sinVal = 0, cosVal = 0;

// Wave buffers (ring buffers)
const WAVE_BUF_LEN  = 300;
const sinBuffer     = new Float32Array(WAVE_BUF_LEN);
const cosBuffer     = new Float32Array(WAVE_BUF_LEN);
let   waveWriteIdx  = 0;

// State flags
let isActive     = false;          // true when radius >= MIN_RADIUS_PX
let lastHandSeen = 0;

// Layout regions (computed on resize)
let CIRCLE_CX, CIRCLE_CY, CIRCLE_R_MAX;
let SINE_X0, SINE_Y0, SINE_W, SINE_H;
let COS_X0,  COS_Y0,  COS_W,  COS_H;

// Projection line endpoints (set by wave drawers, read by projection drawer)
let sineEndpointX, sineEndpointY;
let cosEndpointX,  cosEndpointY;

// Smoothing constants
const EMA_OMEGA        = 0.08;
const EMA_RADIUS       = 0.05;
const MIN_RADIUS_PX    = 30;
const OMEGA_FADE_THRESH = 0.5;   // rad/s
```

---

## 4. The Math

### 4A. Center Estimation

Rolling average of recent fingertip positions converges to the center of a circular path:

```javascript
function updateCenter(x, y) {
  posHistory.push({ x, y });
  if (posHistory.length > HISTORY_LEN) posHistory.shift();
  if (posHistory.length < 10) { centerX = x; centerY = y; return; }

  let sx = 0, sy = 0;
  for (const p of posHistory) { sx += p.x; sy += p.y; }
  centerX = sx / posHistory.length;
  centerY = sy / posHistory.length;
}
```

Edge cases:
- **First 10 frames**: use current position as stub center, `isActive = false`
- **Hand stationary**: radius ≈ 0, `isActive = false`, waves silent
- **Hand moving linearly**: omega stays low, audio silent — acceptable for classroom use

### 4B. Angle Calculation & Unwrapping

```javascript
function updateAngle(x, y, nowMs) {
  const dx = x - centerX, dy = y - centerY;
  radius = Math.sqrt(dx*dx + dy*dy);

  // Rolling max radius (decays slowly)
  maxRadius = radius > maxRadius
    ? radius
    : Math.max(MIN_RADIUS_PX, maxRadius * (1-EMA_RADIUS) + radius * EMA_RADIUS);

  rawAngle = Math.atan2(dy, dx);

  if (prevRawAngle === null) {
    prevRawAngle = rawAngle; prevTimestamp = nowMs; cumulAngle = rawAngle; return;
  }

  // Unwrap: keep delta in [-π, π] to avoid jump at the ±π boundary
  let delta = rawAngle - prevRawAngle;
  if (delta >  Math.PI) delta -= 2 * Math.PI;
  if (delta < -Math.PI) delta += 2 * Math.PI;

  cumulAngle   += delta;
  prevRawAngle  = rawAngle;

  // Normalize for display and sin/cos
  thetaRad = ((rawAngle % (2*Math.PI)) + 2*Math.PI) % (2*Math.PI);
  thetaDeg = thetaRad * 180 / Math.PI;

  const norm = radius / maxRadius;           // scale factor [0,1]
  sinVal = norm * Math.sin(thetaRad);        // Y projection, normalized
  cosVal = norm * Math.cos(thetaRad);        // X projection, normalized

  // Angular velocity (EMA smoothed)
  const dt = (nowMs - prevTimestamp) / 1000;
  prevTimestamp = nowMs;
  if (dt > 0 && dt < 0.2) {
    const omegaRaw = delta / dt;
    omega = EMA_OMEGA * omegaRaw + (1 - EMA_OMEGA) * omega;
  }

  freqHz  = Math.abs(omega) / (2 * Math.PI);
  period  = freqHz > 0.001 ? 1 / freqHz : Infinity;
  isActive = radius >= MIN_RADIUS_PX;
}
```

**Why unwrap?** Without it, `atan2` jumps ±2π when the hand crosses 0°/360°, causing a giant spurious omega spike. Clamping delta to [-π, π] is physically valid since no hand can move half a turn in one frame.

### 4C. Wave Buffers

```javascript
function pushWaveBuffers() {
  sinBuffer[waveWriteIdx] = isActive ? sinVal : 0;
  cosBuffer[waveWriteIdx] = isActive ? cosVal : 0;
  waveWriteIdx = (waveWriteIdx + 1) % WAVE_BUF_LEN;
}

// Read ring buffer in chronological order (oldest → newest)
function getBuffer(buf) {
  const out = new Float32Array(WAVE_BUF_LEN);
  for (let i = 0; i < WAVE_BUF_LEN; i++)
    out[i] = buf[(waveWriteIdx + i) % WAVE_BUF_LEN];
  return out;  // out[WAVE_BUF_LEN-1] = most recent sample
}
```

---

## 5. Rendering Pipeline

Drawing order every frame:

```
1. ctx.clearRect()                — black background
2. drawRegionBorders()            — faint outlines for each region
3. drawCircleRegion()             — axis, ghost circle, radius line, θ arc, dot
4. drawSineRegion()               — sin wave scrolling right, sets sineEndpointX/Y
5. drawCosineRegion()             — cos wave scrolling down, sets cosEndpointX/Y
6. drawProjectionLines()          — dotted lines: circle dot → wave endpoints
7. updateLabels()                 — write θ, ω, f, T, sin, cos to #ui DOM
8. drawStatusOverlay()            — "move your finger in a circle" if !isActive
```

### Layout computation (on load + resize)

```javascript
function computeLayout() {
  const W = canvas.width, H = canvas.height, PAD = 20;
  const circleW = W * 0.38, circleH = H * 0.55;

  CIRCLE_CX    = PAD + circleW * 0.5;
  CIRCLE_CY    = PAD + circleH * 0.5;
  CIRCLE_R_MAX = Math.min(circleW, circleH) * 0.42;

  SINE_X0 = PAD + circleW + PAD;  SINE_Y0 = PAD;
  SINE_W  = W - SINE_X0 - PAD;   SINE_H  = circleH;

  COS_X0  = PAD;                  COS_Y0  = PAD + circleH + PAD;
  COS_W   = circleW;              COS_H   = H - COS_Y0 - PAD;
}
```

### Circle region key elements

- **Axis cross**: faint dashed horizontal + vertical lines
- **Ghost circle**: faint circle at `CIRCLE_R_MAX` showing expected path
- **θ arc**: amber arc from 0 to thetaRad, with "θ" label at midpoint
- **Radius line**: center → spinning dot
- **Spinning dot**: cyan filled circle at `(CIRCLE_CX + CIRCLE_R_MAX·cos θ, CIRCLE_CY + CIRCLE_R_MAX·sin θ)`
- **Axis intercept dots**: small green dot at `(dotX, CIRCLE_CY)` (sin intercept), small pink dot at `(CIRCLE_CX, dotY)` (cos intercept)

### Sine wave region (scrolling right)

- Newest sample = rightmost point
- Y-axis: sin value [-1,1] maps to [bottom, top] of region
- Store `sineEndpointX = SINE_X0 + SINE_W`, `sineEndpointY = SINE_Y0 + SINE_H/2 - sinVal * scaleY`

### Cosine wave region (scrolling down)

- Newest sample = topmost point, scrolls downward
- X-axis: cos value [-1,1] maps to [left, right] of region
- Store `cosEndpointX = COS_X0 + COS_W/2 + cosVal * scaleX`, `cosEndpointY = COS_Y0`

### Projection lines (the key visual)

```javascript
function drawProjectionLines() {
  if (!isActive) return;
  const dotX = CIRCLE_CX + CIRCLE_R_MAX * Math.cos(thetaRad);
  const dotY = CIRCLE_CY + CIRCLE_R_MAX * Math.sin(thetaRad);

  ctx.setLineDash([5, 6]);

  // Dot → sine wave current sample
  ctx.beginPath();
  ctx.moveTo(dotX, dotY); ctx.lineTo(sineEndpointX, sineEndpointY);
  ctx.strokeStyle = 'rgba(0,255,120,0.45)'; ctx.lineWidth = 1.2; ctx.stroke();

  // Dot → cosine wave current sample
  ctx.beginPath();
  ctx.moveTo(dotX, dotY); ctx.lineTo(cosEndpointX, cosEndpointY);
  ctx.strokeStyle = 'rgba(255,80,200,0.45)'; ctx.stroke();

  ctx.setLineDash([]);
}
```

---

## 6. Audio System

```javascript
function startAudio() {
  if (audioStarted) return;
  audioStarted = true;
  audioCtx   = new AudioContext();
  oscillator = audioCtx.createOscillator();
  gainNode   = audioCtx.createGain();
  oscillator.type = 'sine';          // matches the concept being taught!
  oscillator.connect(gainNode);
  gainNode.connect(audioCtx.destination);
  gainNode.gain.value        = 0;
  oscillator.frequency.value = 220;
  oscillator.start();
}

function updateAudio() {
  if (!audioStarted || !isActive || Math.abs(omega) < OMEGA_FADE_THRESH) {
    gainNode.gain.setTargetAtTime(0, audioCtx.currentTime, 0.3);
    return;
  }
  // 1 Hz rotation → 220 Hz tone (A3). Proportional mapping.
  const audioFreq = Math.max(55, Math.min(880, freqHz * 220));
  oscillator.frequency.setTargetAtTime(audioFreq, audioCtx.currentTime, 0.05);

  // Amplitude scales with radius (bigger circle = louder)
  const gain = Math.min(radius / maxRadius, 1) * 0.4;
  gainNode.gain.setTargetAtTime(gain, audioCtx.currentTime, 0.08);
}
```

Audio is a **sine wave** — pedagogically correct, since sin(t) is what an oscillator produces.

Frequency mapping: **1 Hz rotation → 220 Hz audio**. Faster swinging → higher pitch → more compressed waveform on screen. Students can both hear and see the frequency change simultaneously.

---

## 7. Educational Annotations

### Always visible
- Region labels: "CIRCLE", "sin(θ)", "cos(θ)" in matching colors
- Axis labels on circle diagram: x-axis, y-axis
- Formula line under each wave: `"y = sin(θ)"`, `"x = cos(θ)"`

### Active (hand circling)
- `θ` label at midpoint of angle arc
- `sin(θ) = X.XXX` and `cos(θ) = X.XXX` near the wave endpoints
- Rotation direction: `↺ CCW` or `↻ CW`
- `P(cos θ, sin θ)` label trailing the spinning dot

### Inactive (not circling)
- Pulsing ghost circle animation in the circle region
- Centered instruction: **"Move your index finger in a circle"**
- Sub-instruction: "Like swinging a yoyo — point your finger and spin it"

---

## 8. Edge Cases

| Situation | Behavior |
|---|---|
| Hand not visible | Waves freeze, audio fades after 300ms, instruction overlay shown |
| Radius < 30px | `isActive = false`, waves flat, audio silent |
| Very fast rotation (>5Hz) | Audio capped at 880Hz, wave becomes dense (still correct) |
| Very slow rotation (<0.2Hz) | omega < threshold, audio silent, slow wave visible |
| CW ↔ CCW switch | Handled naturally by unwrapper — delta changes sign, EMA smooths in ~5 frames |
| dt > 200ms (tab backgrounded) | omega update skipped for that frame via `dt < 0.2` guard |

---

## 9. Implementation Steps

| Step | What | Verify |
|---|---|---|
| 1 | HTML skeleton, loading overlay, hidden video, canvas | Page loads, loading bar shows |
| 2 | HandLandmarker boot, log tip coordinates | Fingertip x/y visible in console |
| 3 | `computeLayout()`, draw empty region boxes | Layout correct at different window sizes |
| 4 | `updateCenter()` + `updateAngle()`, show θ and ω in label panel | Theta rotates 0→360 when circling |
| 5 | Spinning dot on circle diagram (radius line, arc, dot) | Dot follows finger motion on diagram |
| 6 | Wave buffers + `drawSineRegion()` + `drawCosineRegion()` | Sinusoidal scrolling waves appear |
| 7 | `drawProjectionLines()` | Dotted lines connect dot to wave endpoints |
| 8 | `updateLabels()` — all six values live | Numbers stable and readable |
| 9 | Audio system — tone at rotation frequency | Tone heard, pitch changes with speed |
| 10 | Edge cases, instruction overlay, polish | Graceful behavior in all conditions |

---

## Color Palette

| Element | Color |
|---|---|
| Background | `#000` |
| Spinning dot | `#00eeff` (cyan) |
| θ arc + label | `rgba(255,220,80,0.7)` (amber) |
| Sine wave + line | `rgba(0,255,120,0.85)` (green) |
| Cosine wave + line | `rgba(255,80,200,0.85)` (pink) |
| Projection lines | same as waves, 45% opacity |
| Region borders | `rgba(255,255,255,0.06)` |

## Constants Summary

| Constant | Value | Why |
|---|---|---|
| `HISTORY_LEN` | 90 | ~3s at 30fps for center convergence |
| `WAVE_BUF_LEN` | 300 | ~5 visible cycles at 1Hz/60fps |
| `EMA_OMEGA` | 0.08 | Smooth but responsive to speed changes |
| `MIN_RADIUS_PX` | 30 | Ignores micro-tremors |
| `OMEGA_FADE_THRESH` | 0.5 rad/s | Silences audio when nearly stationary |
| Audio multiplier | 220 | 1 Hz rotation → A3 (220 Hz) |
