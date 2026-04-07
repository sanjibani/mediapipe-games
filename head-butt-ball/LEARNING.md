# Head Butt Ball: Core Logic & Learning Notes

## What This Project Teaches
- MediaPipe **Face Landmarker** (different from Hand Landmarker)
- Mapping forehead position to a game hitbox
- Spawning and managing multiple game objects
- Progressive difficulty ramping (speed, frequency, hitbox shrink)
- Circle-circle collision detection

---

## 1. Face Landmarker Setup

Same pattern as Hand Pong but with `FaceLandmarker` instead of `HandLandmarker`.

### Code (from `index.html`):

```js
const { FaceLandmarker, FilesetResolver } = await import(
  'https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@0.10.18/vision_bundle.mjs'
);

faceLandmarker = await FaceLandmarker.createFromOptions(vision, {
  baseOptions: {
    modelAssetPath: 'https://storage.googleapis.com/mediapipe-models/face_landmarker/face_landmarker/float16/1/face_landmarker.task',
    delegate: 'GPU',
  },
  runningMode: 'VIDEO',
  numFaces: 1,
  minFaceDetectionConfidence: 0.5,
  minTrackingConfidence: 0.5,
});
```

### Key differences from Hand Landmarker:
- Different model file (`face_landmarker.task` vs `hand_landmarker.task`)
- Returns **478 face landmarks** (vs 21 hand landmarks)
- Has confidence thresholds for both detection and tracking

---

## 2. Face Landmarks — Which Point for a "Headbutt"?

MediaPipe Face Landmarker returns 478 points mapping the entire face mesh. Key landmarks:

```
         10          ← forehead center (WE USE THIS)
        /  \
      67    297      ← temple left/right
     /        \
   234   1   454     ← cheek, nose tip, cheek
     \   |   /
      61 199 291     ← jaw line
        \ | /
         152         ← chin bottom
```

### Code:

```js
const lm = results.faceLandmarks[0];
// Landmark 10 = forehead center — the "headbutt" point
headX = (1 - lm[10].x) * W;   // mirror X for webcam
headY = lm[10].y * H;
```

### Why landmark 10?
- It's the top-center of the forehead — exactly where you'd headbutt a ball
- It moves predictably with head movement
- Using nose tip (landmark 1) would feel wrong since you'd be "face-bumping" not "head-butting"

---

## 3. Circle-Circle Collision Detection

The simplest collision check: two circles collide when the distance between their centers is less than the sum of their radii.

### Code:

```js
const dx = b.x - headX;
const dy = b.y - headY;
const dist = Math.sqrt(dx * dx + dy * dy);

if (dist < headRadius + b.r) {
  // HEADBUTT! Ball is hit
  b.alive = false;
  score++;
}
```

### Visual explanation:

```
    headRadius = 40        b.r = 12

     ┌──────┐           ┌────┐
     │  HEAD │ ←─ dist ─→│BALL│
     └──────┘           └────┘

  Collision when: dist < 40 + 12 = 52
```

This is **much simpler** than rectangle collision (used in Hand Pong's paddle). Circle collision is just one distance check — no need to compare edges.

---

## 4. Progressive Difficulty System

Three things change as your score increases:

### Code:

```js
function updateDifficulty() {
  difficultyLevel = Math.floor(score / 5);  // level up every 5 points

  // 1. Balls fall faster
  // vy starts at 1.5 and increases by 0.3 per level
  vy = 1.5 + difficultyLevel * 0.3;

  // 2. Balls spawn more frequently
  // Interval shrinks from 90 frames → minimum 25 frames
  spawnInterval = Math.max(25, 90 - difficultyLevel * 8);

  // 3. Head hitbox shrinks
  // From 40px radius → minimum 22px radius
  headRadius = Math.max(22, 40 - difficultyLevel * 2);
}
```

### Difficulty curve:

| Score | Level | Ball Speed | Spawn Rate | Head Radius | Angles? | Multi-spawn? |
|-------|-------|-----------|------------|-------------|---------|-------------|
| 0-4   | 0     | 1.5       | every 90f  | 40px        | No      | No          |
| 5-9   | 1     | 1.8       | every 82f  | 38px        | 50%     | No          |
| 10-14 | 2     | 2.1       | every 74f  | 36px        | 50%     | 40%         |
| 15-19 | 3     | 2.4       | every 66f  | 34px        | More    | 40%         |
| 25+   | 5     | 3.0       | every 50f  | 30px        | Steep   | 30% triples |

### Ball angle logic:

```js
// Starts straight down (vx = 0)
// After score 5: 50% chance of slight angle
if (score >= 5 && Math.random() > 0.5) {
  vx = (Math.random() - 0.5) * 2;      // gentle diagonal
}
// After score 15: 60% chance of steeper angle
if (score >= 15 && Math.random() > 0.4) {
  vx = (Math.random() - 0.5) * 3.5;    // aggressive diagonal
}
```

---

## 5. Managing Multiple Game Objects

Unlike Hand Pong (one ball), this game has an **array of balls** and must handle spawning, updating, and cleaning up.

### Spawn pattern:

```js
spawnTimer++;
if (spawnTimer >= spawnInterval) {
  spawnBall();
  // Multi-spawn at higher scores
  if (score >= 10 && Math.random() > 0.6) spawnBall();  // 2 at once
  if (score >= 25 && Math.random() > 0.7) spawnBall();  // sometimes 3
  spawnTimer = 0;
}
```

### Cleanup pattern:

```js
// Mark dead (hit or missed)
b.alive = false;

// Remove dead balls after update loop
balls = balls.filter(b => b.alive);
```

This **mark-then-sweep** pattern avoids modifying the array while iterating over it.

---

## 6. Key Differences from Hand Pong

| Aspect | Hand Pong | Head Butt Ball |
|--------|-----------|---------------|
| Tracking | Hand (21 landmarks) | Face (478 landmarks) |
| Landmark used | #9 (palm center) | #10 (forehead) |
| Axes used | x only (paddle is horizontal) | x AND y (head moves in 2D) |
| Collision | Rectangle (paddle) vs circle (ball) | Circle vs circle |
| Objects | 1 ball | Many balls, spawned over time |
| Difficulty | Speed only | Speed + frequency + hitbox size + angles |
| Game feel | Controlled rallying | Frantic survival |

---

## 7. Key Takeaways for Next Levels

| Concept | Used Here | Builds Toward |
|---------|-----------|---------------|
| FaceLandmarker | Forehead tracking | Level 4 head-dodge uses same model |
| 478 face landmarks | Just used #10 | Could use more for expressions |
| Circle collision | `dist < r1 + r2` | Same math in all games |
| Object pooling | Mark-then-sweep array | Standard pattern for particles, enemies |
| Difficulty ramp | Score-based intervals | Reusable in any arcade game |
| Multi-spawn | Random chance at thresholds | Creates emergent difficulty |
