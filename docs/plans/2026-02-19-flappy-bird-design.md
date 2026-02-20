# Flappy Bird — Neon Synthwave Design

**Date:** 2026-02-19

## Overview

A browser-playable Flappy Bird clone with a Neon Synthwave aesthetic. Delivered as a single self-contained `index.html` file — no dependencies, no build tools.

## Architecture

- **Platform**: HTML5 Canvas, vanilla JavaScript
- **File**: Single `index.html` (embedded CSS + JS)
- **Game loop**: `requestAnimationFrame`-driven, delta-time capped to prevent spiral-of-death on tab switch
- **State**: Plain JS object `game` holding all mutable state

## Game States

| State | Description |
|---|---|
| `idle` | Bird bobs gently, "Press Space or Tap to Start" overlay shown |
| `playing` | Physics active, pipes scroll, score increments |
| `dead` | Screen flash, bird falls, Game Over overlay with score + best |

## Physics

- **Gravity**: constant `+0.5 px/frame` acceleration
- **Jump velocity**: `-9 px/frame` instantaneous on Space / click / touch
- **Terminal velocity**: capped at `+12 px/frame` downward
- **Pipe speed**: `3 px/frame`, fixed
- **Pipe spawn**: every 90 frames, gap height `160px`, gap Y randomized within safe bounds
- **Collision**: AABB against each pipe rect and floor/ceiling bounds

## Scoring

- +1 point each time the bird's X passes a pipe pair's X
- Best score persisted in `localStorage`

## Visual Design — Neon Synthwave

### Palette

| Element | Color |
|---|---|
| Background | `#0d0221` deep indigo |
| Bird | `#00f5ff` cyan with glow |
| Pipes | `#ff2079` magenta with bright caps |
| Ground line | `#f7b731` gold |
| Score text | `#ffffff` white |
| Accent glow | `#b829dd` purple |

### Effects

- **Bird glow**: layered `shadowBlur` on canvas context (cyan, large radius)
- **Particle trail**: 6–8 small fading particles emitted each frame behind bird
- **Pipe glow**: `shadowBlur` in magenta on pipe draw
- **Death flash**: white `fillRect` at alpha 0.6, fading to 0 over ~20 frames
- **City silhouette**: static procedurally-drawn dark skyline at bottom, scrolling slowly
- **Stars**: ~80 static white dots drawn once on background layer

## File Structure

```
/
├── index.html        ← entire game
└── docs/
    └── plans/
        └── 2026-02-19-flappy-bird-design.md
```

## Out of Scope

- Audio / sound effects
- Sprite sheets or external images
- Mobile-specific UI chrome
- Difficulty progression beyond initial ramp
