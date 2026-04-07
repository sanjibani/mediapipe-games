# Finger Fruit Slicer: Core Logic & Learning Notes

## What This Project Teaches
- Tracking **multiple fingertip** landmarks (not just palm center)
- Calculating **finger velocity** between frames
- **Swipe gesture detection** via speed thresholds
- Line-to-circle collision for slice detection
- Drawing animated trails behind moving fingers

---

## 1. Multiple Fingertip Tracking

In Hand Pong we used one landmark (#9, palm center). Here we track individual fingertips.

### The 21 Hand Landmarks — Fingertips highlighted:

```
       [4]  [8] [12] [16] [20]    ← FINGERTIP landmarks (we track these)
        |    |    |    |    |
        3    7   11   15   19
        |    |    |    |    |
        2    6   10   14   18
        |    |    |    |    |
        1    5    9   13   17
              \   |   /
               \  |  /
                  0                ← wrist
```

### Code:

```js
const TIP_INDICES = [8, 12]; // index finger tip, middle finger tip

for (const hand of results.landmarks) {
  for (const tipIdx of TIP_INDICES) {
    const lm = hand[tipIdx];
    const x = (1 - lm.x) * W;  // mirror for webcam
    const y = lm.y * H;
    fingerTips.push({ x, y, vx, vy });
  }
}
```

### Why landmarks 8 and 12?
- **8** = index fingertip — the natural "pointing" finger, most intuitive for slicing
- **12** = middle fingertip — often extends alongside index finger in a slicing motion
- We track both so you can slice with either or both fingers
- We set `numHands: 2` so you can use both hands

---

## 2. Finger Velocity Calculation

This is the **key new concept**. We need to know not just WHERE the finger is, but HOW FAST it's moving.

### Code:

```js
// Store previous frame positions
fingerTips._prev = fingerTips.map(f => ({ x: f.x, y: f.y }));

// Next frame: calculate velocity
for (const prev of previousPositions) {
  const d = Math.sqrt((prev.x - x) ** 2 + (prev.y - y) ** 2);
  // Find closest previous point (approximate finger matching)
  vx = x - closest.x;  // pixels moved in x since last frame
  vy = y - closest.y;  // pixels moved in y since last frame
}

const speed = Math.sqrt(vx * vx + vy * vy);
```

### What velocity tells us:

```
speed < 5    → finger is hovering/still → NOT a swipe
speed >= 5   → finger is moving fast → IS a swipe → check for fruit collision
```

### Why we match by proximity:
MediaPipe doesn't guarantee consistent landmark ordering between frames. We find the previous finger position closest to the current one. In practice, fingers don't teleport between frames, so the closest previous point is always the right one.

---

## 3. Swipe-Based Collision Detection

Unlike Head Butt Ball (where you just overlap), slicing requires the finger to be **moving fast enough** when it touches the fruit.

### Code:

```js
function checkSlice() {
  for (const finger of fingerTips) {
    const speed = Math.sqrt(finger.vx ** 2 + finger.vy ** 2);
    if (speed < 5) continue;  // MUST be swiping — hovering doesn't count

    for (const fruit of fruits) {
      const dx = finger.x - fruit.x;
      const dy = finger.y - fruit.y;
      const dist = Math.sqrt(dx * dx + dy * dy);

      if (dist < fruit.r + 20) {  // within fruit radius + generous margin
        fruit.sliced = true;
        // trigger effects...
      }
    }
  }
}
```

### The speed threshold is what makes this a "slicer":
- Without it, you could just hover your finger over fruits to "slice" them — boring
- With `speed >= 5`, you must make a deliberate swiping motion
- This creates the satisfying physical gesture of cutting through fruit

---

## 4. Fruit Physics (Projectile Motion)

Fruits launch upward from the bottom and arc back down — classic projectile motion.

### Code:

```js
function spawnFruit() {
  fruits.push({
    x: randomX,
    y: H + 40,                    // start below screen
    vx: (Math.random() - 0.5) * 4, // slight horizontal drift
    vy: -(8 + Math.random() * 4),   // launch upward (negative = up)
    gravity: 0.12,                   // pulls back down each frame
  });
}

// Each frame:
f.x += f.vx;
f.y += f.vy;
f.vy += f.gravity;  // gravity slows upward motion, then accelerates downward
```

### The arc shape:

```
                  *  *  *
               *           *
             *               *
           *                   *
          *                     *
        *                         *
───────*───────────────────────────*────── screen bottom
     launch                      miss!
```

Fruits that fall past the bottom without being sliced count as a miss.

---

## 5. Swipe Trail Rendering

The glowing trail behind the finger makes slicing feel juicy.

### Code:

```js
// Add trail points when finger is moving
if (speed > 2) {
  fingerTrailPoints.push({ x, y, life: 1 });
}

// Decay trail points
fingerTrailPoints = fingerTrailPoints.filter(p => {
  p.life -= 0.06;  // fade over ~17 frames
  return p.life > 0;
});

// Draw trail as connected line segments
for (let i = 1; i < fingerTrailPoints.length; i++) {
  ctx.globalAlpha = p1.life * 0.8;
  ctx.lineWidth = 3 + p1.life * 4;  // thicker when fresh
  ctx.shadowColor = '#ffeaa7';
  ctx.shadowBlur = 15 * p1.life;    // glow fades with life
  ctx.beginPath();
  ctx.moveTo(p0.x, p0.y);
  ctx.lineTo(p1.x, p1.y);
  ctx.stroke();
}
```

### Why per-segment drawing instead of one path?
Each segment has its own opacity and width based on its `life` value. Fresh trail points are bright and thick; old ones are faint and thin. A single `ctx.stroke()` can only use one lineWidth.

---

## 6. Slice Half Animation

When a fruit is sliced, two "halves" fly apart — this sells the cut.

### Code:

```js
function spawnHalf(fruit, dir) {  // dir = -1 (left) or 1 (right)
  particles.push({
    x: fruit.x, y: fruit.y,
    vx: dir * (2 + Math.random() * 3),  // fly apart
    vy: -3 - Math.random() * 2,         // pop up slightly
    emoji: fruit.type.emoji,
    rotation: fruit.rotation,
    rotSpeed: dir * 0.15,               // spin as they fly
    gravity: 0.15,
  });
}
```

Each half is drawn clipped to a rectangle (showing only half the emoji) and spins away with gravity.

---

## 7. Key Differences from Previous Games

| Aspect | Hand Pong | Head Butt Ball | Fruit Slicer |
|--------|-----------|---------------|-------------|
| Tracking | 1 hand landmark | 1 face landmark | 2-4 fingertip landmarks |
| Input | Position only | Position only | Position + VELOCITY |
| Gesture | None | None | Swipe (speed threshold) |
| Collision | Rectangle vs circle | Circle vs circle | Moving point vs circle |
| Objects | 1 ball (persistent) | Spawned from top | Launched from bottom (arcing) |
| Effect | Bounce | Burst | Slice halves + juice splash |

---

## 8. Key Takeaways for Next Levels

| Concept | Used Here | Builds Toward |
|---------|-----------|---------------|
| Fingertip tracking | Landmarks 8, 12 | Level 3 needs all 5 tips for gesture classification |
| Velocity calculation | `vx = x - prevX` | Level 6 air drums needs downward velocity spikes |
| Speed threshold | `speed >= 5` for swipe | Same pattern for any gesture-based trigger |
| Trail rendering | Fading line segments | Visual feedback pattern for any motion game |
| Projectile motion | `vy += gravity` | Standard physics for any game with arcing objects |
