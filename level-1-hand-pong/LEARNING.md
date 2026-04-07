# Level 1 — Hand Pong: Core Logic & Learning Notes

## What This Project Teaches
- Setting up MediaPipe hand tracking in the browser
- Accessing the webcam via `getUserMedia`
- Reading hand landmark coordinates
- 2D Canvas game loop (update → draw → repeat)
- Basic ball physics and collision detection

---

## 1. MediaPipe Hand Model Setup

MediaPipe runs **entirely in your browser** — no API keys, no token costs, no server calls. The ML model downloads once (~10MB) and runs locally via WebAssembly + WebGL.

### Code (from `index.html` lines 174-189):

```js
const { HandLandmarker, FilesetResolver } = await import(
  'https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@0.10.18/vision_bundle.mjs'
);

const vision = await FilesetResolver.forVisionTasks(
  'https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@0.10.18/wasm'
);

handLandmarker = await HandLandmarker.createFromOptions(vision, {
  baseOptions: {
    modelAssetPath: 'https://storage.googleapis.com/mediapipe-models/hand_landmarker/hand_landmarker/float16/1/hand_landmarker.task',
    delegate: 'GPU',
  },
  runningMode: 'VIDEO',
  numHands: 1,
});
```

### What each piece does:
- **FilesetResolver** — loads the WebAssembly runtime that MediaPipe needs to run ML in the browser
- **HandLandmarker** — the actual hand detection model
- `delegate: 'GPU'` — runs inference on your GPU via WebGL (much faster than CPU)
- `runningMode: 'VIDEO'` — optimized for continuous frames (tracks hand between frames instead of re-detecting from scratch each time)
- `numHands: 1` — only track one hand (cheaper computation)

---

## 2. Webcam Access

### Code (from `index.html` lines 194-202):

```js
const stream = await navigator.mediaDevices.getUserMedia({
  video: { width: W, height: H, facingMode: 'user' }
});
video.srcObject = stream;
await new Promise((resolve, reject) => {
  video.onloadedmetadata = resolve;
  video.onerror = reject;
});
await video.play();
```

### What happens:
- `getUserMedia` is the standard browser API to access the camera
- The video stream feeds into a hidden `<video>` element
- That element serves two purposes:
  - Displayed faintly as the game background (CSS `opacity: 0.25`, `scaleX(-1)` to mirror)
  - Fed as input to MediaPipe every frame

---

## 3. Hand Detection — THE KEY PART

### The 21 Hand Landmarks

MediaPipe returns 21 landmark points per detected hand:

```
        8   12  16  20       ← fingertips
        |   |   |   |
        7   11  15  19
        |   |   |   |
    4   6   10  14  18
    |   |   |   |   |
    3   5   9   13  17       ← landmark 9 = middle finger base (palm center)
    |    \ | /  /
    2      0                 ← 0 = wrist
    |
    1
```

Each landmark has `x`, `y`, `z` as **normalized values (0.0 to 1.0)** relative to the image dimensions.

### Code (from `index.html` lines 224-245):

```js
function detectHands() {
  if (!handLandmarker || useMouseFallback) return;
  if (video.readyState < 2) return;           // video not ready yet
  if (video.currentTime === lastVideoTime) return;  // skip duplicate frames

  lastVideoTime = video.currentTime;
  const results = handLandmarker.detectForVideo(video, performance.now());

  if (results.landmarks && results.landmarks.length > 0) {
    const lm = results.landmarks[0];
    // Landmark 9 = middle finger MCP (center of palm)
    // (1 - x) mirrors it because webcam is flipped
    handX = (1 - lm[9].x) * W;
    handDetected = true;
  } else {
    handDetected = false;
  }
}
```

### Why `(1 - lm[9].x)`?
The webcam image is naturally mirrored (like looking in a mirror). We flip the x-coordinate so moving your hand **right** moves the paddle **right**. Without this, controls feel inverted.

### Why landmark 9?
Landmark 9 is the base of the middle finger — effectively the **center of your palm**. It's the most stable point for general "where is the hand" tracking. Fingertips (landmarks 4, 8, 12, 16, 20) jitter more.

---

## 4. Paddle Smoothing (Lerp)

### Code (from `index.html` line 260):

```js
paddle.x += (handX - paddle.w / 2 - paddle.x) * 0.3;
paddle.x = Math.max(0, Math.min(W - paddle.w, paddle.x));
```

### What is lerp?
**Linear interpolation** — instead of snapping the paddle instantly to your hand position, it moves **30% of the remaining distance** each frame.

```
Frame 1: hand at 400, paddle at 100 → paddle moves to 100 + (400-100)*0.3 = 190
Frame 2: hand at 400, paddle at 190 → paddle moves to 190 + (400-190)*0.3 = 253
Frame 3: hand at 400, paddle at 253 → paddle moves to 253 + (400-253)*0.3 = 297
... converges toward 400
```

The `0.3` is the **smoothing factor**:
- Lower (0.1) = smoother but laggy, feels floaty
- Higher (0.8) = snappy but jittery, shows hand tracking noise
- 0.3 is a good balance for game controls

The second line clamps the paddle within canvas bounds.

---

## 5. Ball Physics

### Code (from `index.html` lines 263-292):

```js
// Move ball by its velocity each frame
ball.x += ball.vx;
ball.y += ball.vy;

// Wall bounce — flip velocity on the axis that hit the wall
if (ball.x - ball.r <= 0) { ball.x = ball.r; ball.vx *= -1; }
if (ball.x + ball.r >= W) { ball.x = W - ball.r; ball.vx *= -1; }
if (ball.y - ball.r <= 0) { ball.y = ball.r; ball.vy *= -1; }

// Paddle collision
if (
  ball.vy > 0 &&                                    // ball moving downward
  ball.y + ball.r >= paddle.y &&                     // ball reached paddle height
  ball.y + ball.r <= paddle.y + paddle.h + Math.abs(ball.vy) &&  // within paddle thickness
  ball.x >= paddle.x &&                              // within paddle left edge
  ball.x <= paddle.x + paddle.w                      // within paddle right edge
) {
  ball.vy *= -1;
  ball.y = paddle.y - ball.r;

  // Angle based on WHERE it hits the paddle
  const hitPos = (ball.x - paddle.x) / paddle.w;  // 0 = left edge, 1 = right edge
  ball.vx = ball.speed * (hitPos - 0.5) * 2.5;    // center = straight, edges = angled

  score++;
  ball.speed = Math.min(10, 4 + score * 0.15);    // gradually increase speed
}
```

### Hit position → angle mapping:
```
hitPos = 0.0 (left edge)   → vx = speed * (-0.5) * 2.5 = strong left
hitPos = 0.5 (center)      → vx = speed * (0.0) * 2.5  = straight up
hitPos = 1.0 (right edge)  → vx = speed * (0.5) * 2.5  = strong right
```

This gives the player **control** — you can aim the ball by hitting it with different parts of the paddle.

---

## 6. The Game Loop

### Code (from `index.html`):

```js
function loop() {
  detectHands();                // 1. Read hand position from webcam via ML model
  update();                     // 2. Move ball, check collisions, update score
  draw();                       // 3. Render everything to canvas
  requestAnimationFrame(loop);  // 4. Repeat at ~60fps
}
```

### The pipeline every single frame:

```
Camera frame → MediaPipe ML model → 21 hand landmarks → grab x of landmark 9
    → smooth it (lerp) → use as paddle position → physics tick → render
```

`requestAnimationFrame` syncs with your monitor's refresh rate (usually 60fps). The browser calls `loop()` right before each screen repaint, giving smooth animation.

---

## 7. Key Takeaways for Next Levels

| Concept | Used Here | Builds Toward |
|---------|-----------|---------------|
| `HandLandmarker` setup | Load model, configure GPU | Same setup in all hand-based games |
| Single landmark (palm center) | `lm[9].x` for paddle x | Level 2 uses fingertip landmarks |
| Normalized coordinates (0-1) | Convert to pixel space | Same pattern everywhere |
| Lerp smoothing | `paddle.x += (target - paddle.x) * 0.3` | Used in every game for smooth controls |
| `requestAnimationFrame` loop | 60fps game loop | Foundation for all games |
| Mouse fallback | Graceful degradation | Good practice for all webcam projects |

---

## No API Costs

MediaPipe is **on-device inference** — Google open-sourced the models and runtime. Everything runs client-side in WebAssembly + WebGL. There are:
- No API calls to any server
- No API key needed
- No per-request cost or rate limits
- No internet needed after the first model download (~10MB, cached by browser)
