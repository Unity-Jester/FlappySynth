# Flappy Bird — Neon Synthwave Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build a single-file browser-playable Flappy Bird clone with a Neon Synthwave aesthetic.

**Architecture:** Single `index.html` with embedded CSS and JS. HTML5 Canvas renders everything. A `requestAnimationFrame` game loop drives all updates. All mutable state lives in one `game` object.

**Tech Stack:** HTML5 Canvas API, vanilla JavaScript (ES6), no external dependencies.

---

### Task 1: HTML Scaffold + Canvas Setup

**Files:**
- Create: `index.html`

**Step 1: Create the file with canvas and minimal CSS**

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Flappy Synth</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body {
      background: #0d0221;
      display: flex;
      justify-content: center;
      align-items: center;
      height: 100vh;
      overflow: hidden;
    }
    canvas {
      display: block;
      /* keep pixel-perfect on HiDPI but scale via CSS */
      image-rendering: pixelated;
    }
  </style>
</head>
<body>
  <canvas id="game"></canvas>
  <script>
    const canvas = document.getElementById('game');
    const ctx = canvas.getContext('2d');

    const W = 480;
    const H = 640;
    canvas.width  = W;
    canvas.height = H;

    // Scale canvas to fit viewport while preserving aspect ratio
    function resize() {
      const scale = Math.min(window.innerWidth / W, window.innerHeight / H);
      canvas.style.width  = Math.floor(W * scale) + 'px';
      canvas.style.height = Math.floor(H * scale) + 'px';
    }
    window.addEventListener('resize', resize);
    resize();

    // Verify: fill with background colour
    ctx.fillStyle = '#0d0221';
    ctx.fillRect(0, 0, W, H);
  </script>
</body>
</html>
```

**Step 2: Open in browser and verify**

Open `index.html` in a browser. Expected: deep indigo `#0d0221` rectangle centred on screen, scales on window resize.

**Step 3: Commit**

```bash
git init
git add index.html docs/
git commit -m "feat: initial scaffold with canvas setup"
```

---

### Task 2: Game State Object + Loop

**Files:**
- Modify: `index.html` (JS section)

**Step 1: Replace the verify fill with game state and loop**

Replace everything after `resize();` with:

```js
// ─── Constants ───────────────────────────────────────────────────────────────
const GRAVITY      = 0.5;
const JUMP_VEL     = -9;
const TERM_VEL     = 12;
const PIPE_SPEED   = 3;
const PIPE_INTERVAL= 90;   // frames between pipe spawns
const PIPE_GAP     = 160;
const GROUND_Y     = H - 40;

// ─── Game state ──────────────────────────────────────────────────────────────
const game = {
  state: 'idle',   // 'idle' | 'playing' | 'dead'
  frame: 0,
  score: 0,
  best:  parseInt(localStorage.getItem('flappy_best') || '0'),
  bird: { x: 120, y: H / 2, vy: 0, radius: 14 },
  pipes: [],
  particles: [],
  flash: 0,        // death flash alpha (0–1)
};

// ─── Input ───────────────────────────────────────────────────────────────────
function flap() {
  if (game.state === 'idle') {
    game.state = 'playing';
  }
  if (game.state === 'playing') {
    game.bird.vy = JUMP_VEL;
  }
  if (game.state === 'dead') {
    resetGame();
  }
}
document.addEventListener('keydown', e => { if (e.code === 'Space') { e.preventDefault(); flap(); } });
canvas.addEventListener('click',     flap);
canvas.addEventListener('touchstart', e => { e.preventDefault(); flap(); }, { passive: false });

function resetGame() {
  game.state    = 'idle';
  game.frame    = 0;
  game.score    = 0;
  game.bird     = { x: 120, y: H / 2, vy: 0, radius: 14 };
  game.pipes    = [];
  game.particles= [];
  game.flash    = 0;
}

// ─── Game loop ────────────────────────────────────────────────────────────────
let lastTime = 0;
function loop(ts) {
  const dt = Math.min(ts - lastTime, 50); // cap at 50 ms
  lastTime = ts;

  update(dt);
  draw();

  requestAnimationFrame(loop);
}

function update(dt) {
  // placeholder
}

function draw() {
  ctx.fillStyle = '#0d0221';
  ctx.fillRect(0, 0, W, H);
}

requestAnimationFrame(loop);
```

**Step 2: Open in browser and verify**

Expected: solid indigo screen, no console errors. Press Space — no crash.

**Step 3: Commit**

```bash
git add index.html
git commit -m "feat: game state object and RAF loop"
```

---

### Task 3: Background — Stars + City Silhouette + Ground

**Files:**
- Modify: `index.html` (add before the loop section)

**Step 1: Generate static star data + city silhouette, draw in `draw()`**

Add after `resetGame()` definition and before the loop:

```js
// ─── Static background layers ─────────────────────────────────────────────
const stars = Array.from({ length: 80 }, () => ({
  x: Math.random() * W,
  y: Math.random() * (GROUND_Y - 60),
  r: Math.random() * 1.5 + 0.3,
  a: Math.random() * 0.5 + 0.5,
}));

// City silhouette: array of {x, w, h} blocks
const cityBlocks = [];
{
  let cx = 0;
  while (cx < W + 60) {
    const bw = 20 + Math.random() * 40;
    const bh = 30 + Math.random() * 80;
    cityBlocks.push({ x: cx, w: bw, h: bh });
    cx += bw + 2 + Math.random() * 8;
  }
}
```

**Step 2: Replace the `draw()` stub with full background draw**

```js
function draw() {
  // Sky
  ctx.fillStyle = '#0d0221';
  ctx.fillRect(0, 0, W, H);

  // Stars
  stars.forEach(s => {
    ctx.globalAlpha = s.a;
    ctx.fillStyle = '#ffffff';
    ctx.beginPath();
    ctx.arc(s.x, s.y, s.r, 0, Math.PI * 2);
    ctx.fill();
  });
  ctx.globalAlpha = 1;

  // City silhouette
  ctx.fillStyle = '#1a0533';
  cityBlocks.forEach(b => {
    ctx.fillRect(b.x, GROUND_Y - b.h, b.w, b.h);
  });

  // Ground glow line
  ctx.shadowColor = '#f7b731';
  ctx.shadowBlur = 10;
  ctx.strokeStyle = '#f7b731';
  ctx.lineWidth = 2;
  ctx.beginPath();
  ctx.moveTo(0, GROUND_Y);
  ctx.lineTo(W, GROUND_Y);
  ctx.stroke();
  ctx.shadowBlur = 0;
}
```

**Step 3: Open in browser and verify**

Expected: star field, dark purple city silhouette, gold ground line.

**Step 4: Commit**

```bash
git add index.html
git commit -m "feat: background stars, city silhouette, ground line"
```

---

### Task 4: Bird Rendering + Idle Bob + Physics

**Files:**
- Modify: `index.html`

**Step 1: Add `drawBird()` helper after the `draw()` function**

```js
function drawBird() {
  const b = game.bird;
  // outer glow
  ctx.shadowColor = '#00f5ff';
  ctx.shadowBlur = 30;
  ctx.fillStyle = '#00f5ff';
  ctx.beginPath();
  ctx.arc(b.x, b.y, b.radius, 0, Math.PI * 2);
  ctx.fill();
  // inner bright core
  ctx.shadowBlur = 8;
  ctx.fillStyle = '#ffffff';
  ctx.beginPath();
  ctx.arc(b.x, b.y, b.radius * 0.45, 0, Math.PI * 2);
  ctx.fill();
  ctx.shadowBlur = 0;
}
```

**Step 2: Call `drawBird()` at the end of `draw()`**

Add `drawBird();` before the closing `}` of `draw()`.

**Step 3: Add idle bob + playing physics to `update()`**

```js
function update(dt) {
  game.frame++;

  if (game.state === 'idle') {
    // gentle bob
    game.bird.y = H / 2 + Math.sin(game.frame * 0.05) * 10;
    return;
  }

  if (game.state === 'playing') {
    // Gravity
    game.bird.vy = Math.min(game.bird.vy + GRAVITY, TERM_VEL);
    game.bird.y += game.bird.vy;

    // Hit floor or ceiling
    if (game.bird.y + game.bird.radius >= GROUND_Y || game.bird.y - game.bird.radius <= 0) {
      killBird();
    }
    return;
  }

  if (game.state === 'dead') {
    // Keep falling off screen
    game.bird.vy = Math.min(game.bird.vy + GRAVITY, TERM_VEL);
    game.bird.y += game.bird.vy;
    if (game.flash > 0) game.flash -= 0.03;
  }
}

function killBird() {
  if (game.state === 'dead') return;
  game.state = 'dead';
  game.flash = 0.7;
  if (game.score > game.best) {
    game.best = game.score;
    localStorage.setItem('flappy_best', game.best);
  }
}
```

**Step 4: Open in browser and verify**

Expected: glowing cyan orb bobbing at centre in idle. Press Space: orb falls with gravity, hits ground and stops updating (dead state). Press Space again: resets.

**Step 5: Commit**

```bash
git add index.html
git commit -m "feat: bird rendering, idle bob, physics, death"
```

---

### Task 5: Particle Trail

**Files:**
- Modify: `index.html`

**Step 1: Add particle spawn + update to `update()` playing block**

Inside the `if (game.state === 'playing')` block, before `return`, add:

```js
// Spawn particle
if (game.frame % 2 === 0) {
  game.particles.push({
    x: game.bird.x - game.bird.radius,
    y: game.bird.y + (Math.random() - 0.5) * 8,
    vx: -(Math.random() * 1.5 + 0.5),
    vy: (Math.random() - 0.5) * 1.2,
    life: 1.0,
    r: Math.random() * 4 + 2,
  });
}
// Update particles
game.particles = game.particles.filter(p => {
  p.x += p.vx;
  p.y += p.vy;
  p.life -= 0.06;
  return p.life > 0;
});
```

Also update particles in dead state — add after `game.flash -= 0.03` line:

```js
game.particles = game.particles.filter(p => {
  p.x += p.vx; p.y += p.vy; p.life -= 0.06; return p.life > 0;
});
```

**Step 2: Add `drawParticles()` helper and call it in `draw()`**

```js
function drawParticles() {
  game.particles.forEach(p => {
    ctx.globalAlpha = p.life * 0.8;
    ctx.shadowColor = '#00f5ff';
    ctx.shadowBlur = 6;
    ctx.fillStyle = '#00f5ff';
    ctx.beginPath();
    ctx.arc(p.x, p.y, p.r * p.life, 0, Math.PI * 2);
    ctx.fill();
  });
  ctx.globalAlpha = 1;
  ctx.shadowBlur = 0;
}
```

Call `drawParticles();` in `draw()` just before `drawBird();`.

**Step 3: Open in browser and verify**

Expected: cyan particle trail streams behind the bird during play.

**Step 4: Commit**

```bash
git add index.html
git commit -m "feat: cyan particle trail"
```

---

### Task 6: Pipes — Spawn, Scroll, Render

**Files:**
- Modify: `index.html`

**Step 1: Add pipe spawning + scrolling to `update()` playing block**

Before the particle section in the playing block:

```js
// Spawn pipes
if (game.frame % PIPE_INTERVAL === 0) {
  const minY = 80;
  const maxY = GROUND_Y - PIPE_GAP - 80;
  const gapTop = minY + Math.random() * (maxY - minY);
  game.pipes.push({
    x: W + 30,
    gapTop,
    gapBot: gapTop + PIPE_GAP,
    scored: false,
    w: 52,
  });
}

// Move pipes
game.pipes = game.pipes.filter(p => {
  p.x -= PIPE_SPEED;
  return p.x + p.w > -10;
});
```

**Step 2: Add `drawPipes()` helper**

```js
function drawPipes() {
  game.pipes.forEach(p => {
    const capH = 20;
    const capW = p.w + 8;
    const capOff = (capW - p.w) / 2;

    ctx.shadowColor = '#ff2079';
    ctx.shadowBlur = 18;

    // Top pipe body
    ctx.fillStyle = '#c0006a';
    ctx.fillRect(p.x, 0, p.w, p.gapTop - capH);

    // Top pipe cap
    ctx.fillStyle = '#ff2079';
    ctx.fillRect(p.x - capOff, p.gapTop - capH, capW, capH);

    // Bottom pipe body
    ctx.fillStyle = '#c0006a';
    ctx.fillRect(p.x, p.gapBot + capH, p.w, GROUND_Y - p.gapBot - capH);

    // Bottom pipe cap
    ctx.fillStyle = '#ff2079';
    ctx.fillRect(p.x - capOff, p.gapBot, capW, capH);

    ctx.shadowBlur = 0;
  });
}
```

Call `drawPipes();` in `draw()` after drawing the background, before particles/bird.

**Step 3: Open in browser and verify**

Expected: magenta glowing pipes scroll from right to left. Bird falls through them (no collision yet).

**Step 4: Commit**

```bash
git add index.html
git commit -m "feat: pipe spawning, scrolling, neon rendering"
```

---

### Task 7: Collision Detection + Scoring

**Files:**
- Modify: `index.html`

**Step 1: Add collision check + score increment inside the playing update block**

After moving pipes, add:

```js
const b = game.bird;
for (const p of game.pipes) {
  const bLeft  = b.x - b.radius;
  const bRight = b.x + b.radius;
  const bTop   = b.y - b.radius;
  const bBot   = b.y + b.radius;

  const inXRange = bRight > p.x - 4 && bLeft < p.x + p.w + 4;

  if (inXRange && (bTop < p.gapTop || bBot > p.gapBot)) {
    killBird();
    break;
  }

  // Score: bird just passed pipe centre
  if (!p.scored && b.x > p.x + p.w / 2) {
    p.scored = true;
    game.score++;
  }
}
```

**Step 2: Open in browser and verify**

Expected: game enters dead state when bird hits a pipe or floor/ceiling. Score increments each time bird passes a pipe pair.

**Step 3: Commit**

```bash
git add index.html
git commit -m "feat: AABB collision detection and scoring"
```

---

### Task 8: HUD — Score Display During Play

**Files:**
- Modify: `index.html`

**Step 1: Add `drawHUD()` and call it in `draw()` after everything else**

```js
function drawHUD() {
  if (game.state === 'playing' || game.state === 'dead') {
    ctx.shadowColor = '#ffffff';
    ctx.shadowBlur = 12;
    ctx.fillStyle = '#ffffff';
    ctx.font = 'bold 52px monospace';
    ctx.textAlign = 'center';
    ctx.fillText(game.score, W / 2, 90);
    ctx.shadowBlur = 0;
  }
}
```

Call `drawHUD();` as the last call in `draw()`.

**Step 2: Open in browser and verify**

Expected: white glowing score number at top-centre during play.

**Step 3: Commit**

```bash
git add index.html
git commit -m "feat: score HUD display"
```

---

### Task 9: Overlays — Idle Screen + Game Over Screen + Death Flash

**Files:**
- Modify: `index.html`

**Step 1: Add `drawOverlay()` helper**

```js
function drawOverlay() {
  // Death flash
  if (game.flash > 0) {
    ctx.globalAlpha = game.flash;
    ctx.fillStyle = '#ffffff';
    ctx.fillRect(0, 0, W, H);
    ctx.globalAlpha = 1;
  }

  if (game.state === 'idle') {
    // Title
    ctx.shadowColor = '#00f5ff';
    ctx.shadowBlur = 20;
    ctx.fillStyle = '#00f5ff';
    ctx.font = 'bold 42px monospace';
    ctx.textAlign = 'center';
    ctx.fillText('FLAPPY SYNTH', W / 2, H / 2 - 40);
    ctx.shadowBlur = 0;

    // Prompt
    ctx.fillStyle = 'rgba(255,255,255,0.75)';
    ctx.font = '20px monospace';
    ctx.fillText('SPACE / TAP TO START', W / 2, H / 2 + 10);
  }

  if (game.state === 'dead') {
    // Dim background
    ctx.fillStyle = 'rgba(13, 2, 33, 0.65)';
    ctx.fillRect(0, 0, W, H);

    ctx.shadowColor = '#ff2079';
    ctx.shadowBlur = 24;
    ctx.fillStyle = '#ff2079';
    ctx.font = 'bold 46px monospace';
    ctx.textAlign = 'center';
    ctx.fillText('GAME OVER', W / 2, H / 2 - 60);
    ctx.shadowBlur = 0;

    ctx.fillStyle = '#ffffff';
    ctx.font = '28px monospace';
    ctx.fillText(`SCORE  ${game.score}`, W / 2, H / 2);
    ctx.fillText(`BEST   ${game.best}`, W / 2, H / 2 + 40);

    ctx.fillStyle = 'rgba(255,255,255,0.6)';
    ctx.font = '18px monospace';
    ctx.fillText('SPACE / TAP TO RESTART', W / 2, H / 2 + 100);
  }
}
```

Call `drawOverlay();` as the very last call in `draw()` (after `drawHUD()`).

**Step 2: Open in browser and verify**

- Idle: "FLAPPY SYNTH" title in cyan glow, start prompt visible.
- Play: score only.
- Die: white flash, then "GAME OVER" overlay with score, best, restart prompt.

**Step 3: Commit**

```bash
git add index.html
git commit -m "feat: idle/game-over overlays and death flash"
```

---

### Task 10: Draw Order Audit + Final Polish

**Files:**
- Modify: `index.html`

**Step 1: Confirm `draw()` call order**

The final `draw()` body should call helpers in this exact order:

```
1. background fill (#0d0221)
2. stars
3. city silhouette
4. ground line
5. drawPipes()
6. drawParticles()
7. drawBird()
8. drawHUD()
9. drawOverlay()
```

Reorder if needed to match.

**Step 2: Pipe spawn timing — skip first frame**

Change pipe spawn condition so pipes don't appear on frame 0:

```js
if (game.frame % PIPE_INTERVAL === 0 && game.frame > 0) {
```

**Step 3: Open in browser and do a full play-through**

Verify:
- [ ] Idle bob works
- [ ] First flap starts game
- [ ] Pipes scroll at steady speed
- [ ] Bird dies on pipe hit and floor hit
- [ ] Score increments per pipe pair
- [ ] Best score saves across page reloads (localStorage)
- [ ] Game Over overlay shows correct score/best
- [ ] Restarting from Game Over works
- [ ] Scales correctly on window resize

**Step 4: Commit**

```bash
git add index.html
git commit -m "feat: draw order audit and final polish — game complete"
```

---

## Complete

The game is fully playable in any modern browser by opening `index.html`. No server required.
