# Character Selection Design

**Date:** 2026-02-19

## Overview

Add 5 themed, canvas-drawn characters to Flappy Synth. A picker on the idle/title screen lets the player cycle through characters using keyboard arrows or click/tap zones. Selection persists across sessions via `localStorage`.

## Characters

| Index | Name | Shape | Color |
|---|---|---|---|
| 0 | Orb | Glowing circle (existing drawBird) | Cyan `#00f5ff` |
| 1 | Rocket | Nose triangle + body rect + fins | Magenta `#ff2079` |
| 2 | UFO | Ellipse saucer + dome | Neon green `#39ff14` |
| 3 | Ghost | Arc top + scalloped wavy bottom | Purple `#b829dd` |
| 4 | Pixel Bird | Pixel-art rectangles | Gold `#f7b731` |

Each definition: `{ name, color, draw(ctx, bird) }`. Particle trail color matches each character's `color`.

## Picker UI

- `<` and `>` arrows drawn on canvas flanking the bobbing bird on the idle screen
- Character name displayed below the bird
- **Keyboard:** ArrowLeft / ArrowRight cycle characters (idle state only)
- **Mouse/touch:** canvas divided into thirds — left third (x < W/3) cycles back, right third (x > W*2/3) cycles forward, center third starts game (unchanged flap behavior)
- Selected index persisted as `flappy_char` in `localStorage`

## Code Changes

### New: `CHARS` array
Array of 5 objects: `{ name, color, draw(ctx, bird) }`. Each `draw` receives the canvas context and the bird object (for position and radius).

### Modified: `game` object
Add `charIndex: parseInt(localStorage.getItem('flappy_char') || '0')`.

### Modified: `drawBird()`
Replace hardcoded draw logic with `CHARS[game.charIndex].draw(ctx, game.bird)`.

### Modified: `drawParticles()`
Replace hardcoded `#00f5ff` with `CHARS[game.charIndex].color`.

### Modified: `drawOverlay()` idle section
Add `<` / `>` arrows and character name below the bird.

### Modified: keydown handler
Add `ArrowLeft` / `ArrowRight` cases that cycle `game.charIndex` (only in `idle` state) and save to `localStorage`.

### Modified: canvas click + touchstart handlers
In `idle` state, check click x-position: left third → prev char, right third → next char, center → `flap()`. In all other states, always call `flap()`.

## Out of Scope

- Unlockable characters
- Animated character previews beyond the existing idle bob
- Per-character particle shapes
