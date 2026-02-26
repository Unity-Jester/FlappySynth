# FlappySynth Sound Design

**Date:** 2026-02-26
**Status:** Approved

## Overview

Add procedural audio to FlappySynth using the Web Audio API. All audio is generated programmatically — no audio files. Chiptune-style sound effects for gameplay actions, synthwave background music with adaptive state changes.

## Architecture (Approach B: Separated SFX + Music)

Three objects inside `index.html`:

```
Audio (coordinator)
├── ctx: AudioContext
├── masterGain → ctx.destination
├── init()          // resume ctx on first interaction
├── setState(state) // notify SFX + Music of state changes
│
├── SFX
│   ├── one-shot sounds via oscillators
│   ├── each sound is a function: flap(), score(), die(), etc.
│   └── per-character flap variations keyed by charIndex
│
└── Music
    ├── sequencer scheduled via ctx.currentTime
    ├── layers: kick, hihat, bass, pad, arp/lead
    ├── adaptive filter/gain changes per game state
    └── chord progression cycling
```

**Initialization:** AudioContext created suspended at load. Resumed on first user interaction (idempotent). No UI elements added.

**Placement:** Between `CHARS` array and `game` object (~line 218).

## Sound Effects Catalog

| Event | Trigger | Synthesis |
|-------|---------|-----------|
| Flap | Jump handler | Square wave pitch sweep up (300→500Hz, ~80ms). Per-character waveform variation. |
| Score | Pipe passed in `update()` | Ascending 3-note arpeggio (C5→E5→G5), triangle wave, ~150ms |
| Death | `killBird()` | Noise burst + falling saw sweep (400→80Hz, ~300ms) |
| Game start | idle→playing | Rising sweep with reverb tail (~200ms) |
| Character switch | Arrow/click in idle | Single-cycle square wave click (~30ms) |
| Pipe whoosh | Pipe passes bird x | Filtered noise burst, quiet, subtle |
| Best score fanfare | New best in dead state | 4-note ascending melody, bright waveform, delay effect |
| Particle trail | Synced to particle emit | Tiny quiet tick at bird pitch, barely audible |
| Idle hum | While in idle state | Low continuous drone, gentle oscillation, fades with state |

### Per-Character Flap Variations

| Character | Flap Twist |
|-----------|-----------|
| Orb (cyan) | Pure sine wave, clean and smooth |
| Rocket (magenta) | Sawtooth with slight distortion, aggressive |
| UFO (green) | Ring-modulated tone, alien warble |
| Ghost (purple) | Soft triangle wave with vibrato, eerie |
| Pixel Bird (gold) | Classic square wave, most 8-bit |

## Music System

**Sequencer:** ~110 BPM, 16-step loop, scheduled via `AudioContext.currentTime` (no setInterval).

**Chord progression:** 4 bars cycling (i → VI → III → VII).

### Layers

| Layer | Waveform | Role | Pattern |
|-------|----------|------|---------|
| Kick | Sine pitch drop 160→40Hz | Rhythmic anchor | Beats 1, 5, 9, 13 |
| Hi-hat | Filtered noise, short | Texture | 8th/16th notes, velocity variation |
| Bass | Sawtooth, low-passed | Harmonic foundation | Root notes, 8th note pattern |
| Pad | Detuned saw pair, low-passed | Warmth | Sustained chords, slow filter sweep |
| Arp/Lead | Square/triangle | Energy/melody | Arpeggiated chord tones, 16ths |

### Adaptive Behavior

| State | Kick | Hi-hat | Bass | Pad | Arp | Filter |
|-------|------|--------|------|-----|-----|--------|
| Idle | Off | Off | Off | Full, warm | Slow gentle | LP ~800Hz, mellow |
| Playing | Full | Full | Full | Full | Full energy | Opens to ~4kHz+ |
| Dead | Off | Off | Off | Swells, detuned | Off | Sweeps down, reverb |

### Transitions

- **idle → playing:** Layers fade in ~0.5s, filter opens. Kick enters on next beat boundary.
- **playing → dead:** Layers cut except pad (swells, detunes). Filter sweeps down ~1s.
- **dead → idle:** Crossfade back over ~1s.

## Integration Points

1. **AudioContext lifecycle:** `Audio.init()` called in existing `keydown`/`click`/`touchstart` handlers (idempotent).
2. **State transitions:**
   - `idle → playing` (input handler): `Audio.setState('playing')` + `SFX.start()`
   - `playing → dead` (`killBird()`): `Audio.setState('dead')` + `SFX.die()`
   - `dead → idle` (`resetGame()`): `Audio.setState('idle')`
3. **Inline SFX triggers:**
   - Flap: at `bird.vy = JUMP_VEL` → `SFX.flap(charIndex)`
   - Score: at score increment → `SFX.score()` + best check → `SFX.fanfare()`
   - Character switch: at `charIndex` change → `SFX.charSwitch()`
   - Pipe whoosh: at pipe-passing detection → `SFX.whoosh()`
   - Particle trail: at particle emit → `SFX.particleTick()`
4. **No new HTML elements.** No UI changes.
