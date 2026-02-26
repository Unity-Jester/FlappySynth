# FlappySynth Audio System Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add fully procedural audio (Web Audio API) to FlappySynth — chiptune SFX with per-character variations and an adaptive synthwave background music track.

**Architecture:** Three objects (`Audio` coordinator, `SFX`, `Music`) inserted into `index.html` between the `CHARS` array (line 218) and the `game` object (line 220). Audio starts on first user interaction. Music adapts to game state (idle/playing/dead).

**Tech Stack:** Web Audio API (OscillatorNode, GainNode, BiquadFilterNode, noise via AudioBuffer), all inline in `index.html`.

---

### Task 1: Audio Coordinator + AudioContext Lifecycle

**Files:**
- Modify: `index.html:219` (insert between CHARS and game object)
- Modify: `index.html:245-246` (keydown handler — add Audio.init())
- Modify: `index.html:252` (click handler — add Audio.init())
- Modify: `index.html:262` (touchstart handler — add Audio.init())

**Step 1: Add the Audio coordinator object after CHARS (line 218)**

Insert after line 218 (`];` closing CHARS) and before line 220 (`const game = {`):

```javascript
// ─── Audio ──────────────────────────────────────────────────────────────────
const Audio = {
  ctx: null,
  masterGain: null,
  ready: false,

  init() {
    if (this.ready) return;
    this.ctx = new (window.AudioContext || window.webkitAudioContext)();
    this.masterGain = this.ctx.createGain();
    this.masterGain.gain.value = 0.5;
    this.masterGain.connect(this.ctx.destination);
    SFX.init(this.ctx, this.masterGain);
    Music.init(this.ctx, this.masterGain);
    this.ready = true;
  },

  setState(state) {
    if (!this.ready) return;
    Music.setState(state);
  },
};
```

**Step 2: Add Audio.init() calls to input handlers**

In the `keydown` handler (line 245), add as first line inside the callback:
```javascript
Audio.init();
```

In the `click` handler (line 252), add as first line inside the callback:
```javascript
Audio.init();
```

In the `touchstart` handler (line 262), add as first line inside the callback:
```javascript
Audio.init();
```

**Step 3: Add stub SFX and Music objects**

Insert right after the Audio object (these will be fleshed out in later tasks):

```javascript
const SFX = {
  ctx: null, out: null,
  init(ctx, out) { this.ctx = ctx; this.out = out; },
  flap(charIdx) {},
  score() {},
  die() {},
  start() {},
  charSwitch() {},
  whoosh() {},
  fanfare() {},
  particleTick() {},
  idleHum: { start() {}, stop() {} },
};

const Music = {
  ctx: null, out: null,
  init(ctx, out) { this.ctx = ctx; this.out = out; },
  setState(state) {},
};
```

**Step 4: Verify in browser**

Open `index.html` in browser. Tap/click/space. Verify no console errors. Game should play exactly as before (audio stubs are silent no-ops).

**Step 5: Commit**

```bash
git add index.html
git commit -m "feat: add Audio coordinator with AudioContext lifecycle and SFX/Music stubs"
```

---

### Task 2: Core SFX — Flap, Score, Death, Game Start

**Files:**
- Modify: `index.html` — replace SFX stub with real implementations

**Step 1: Implement SFX helper and core sounds**

Replace the `SFX` stub object with:

```javascript
const SFX = {
  ctx: null, out: null,

  init(ctx, out) {
    this.ctx = ctx;
    this.out = out;
  },

  // ── helpers ──
  _osc(type, freq, duration, gain) {
    const t = this.ctx.currentTime;
    const g = this.ctx.createGain();
    g.gain.setValueAtTime(gain, t);
    g.gain.exponentialRampToValueAtTime(0.001, t + duration);
    g.connect(this.out);
    const o = this.ctx.createOscillator();
    o.type = type;
    o.frequency.setValueAtTime(freq, t);
    o.connect(g);
    o.start(t);
    o.stop(t + duration);
    return { osc: o, gain: g, t };
  },

  _noise(duration, gain, freq) {
    const t = this.ctx.currentTime;
    const sr = this.ctx.sampleRate;
    const buf = this.ctx.createBuffer(1, sr * duration, sr);
    const data = buf.getChannelData(0);
    for (let i = 0; i < data.length; i++) data[i] = Math.random() * 2 - 1;
    const src = this.ctx.createBufferSource();
    src.buffer = buf;
    const filt = this.ctx.createBiquadFilter();
    filt.type = 'bandpass';
    filt.frequency.value = freq || 4000;
    filt.Q.value = 1;
    const g = this.ctx.createGain();
    g.gain.setValueAtTime(gain, t);
    g.gain.exponentialRampToValueAtTime(0.001, t + duration);
    src.connect(filt);
    filt.connect(g);
    g.connect(this.out);
    src.start(t);
    src.stop(t + duration);
  },

  // ── per-character waveforms ──
  _charWave: ['sine', 'sawtooth', 'square', 'triangle', 'square'],

  // ── sounds ──
  flap(charIdx) {
    if (!this.ctx) return;
    const wave = this._charWave[charIdx] || 'square';
    const { osc, t } = this._osc(wave, 300, 0.08, 0.18);
    osc.frequency.exponentialRampToValueAtTime(500, t + 0.06);
    // UFO ring mod
    if (charIdx === 2) {
      const mod = this.ctx.createOscillator();
      const modG = this.ctx.createGain();
      mod.frequency.value = 40;
      modG.gain.value = 200;
      mod.connect(modG);
      modG.connect(osc.frequency);
      mod.start(t);
      mod.stop(t + 0.08);
    }
    // Ghost vibrato
    if (charIdx === 3) {
      const vib = this.ctx.createOscillator();
      const vibG = this.ctx.createGain();
      vib.frequency.value = 8;
      vibG.gain.value = 30;
      vib.connect(vibG);
      vibG.connect(osc.frequency);
      vib.start(t);
      vib.stop(t + 0.08);
    }
    // Rocket distortion (clip via waveshaper)
    if (charIdx === 1) {
      const ws = this.ctx.createWaveShaper();
      const k = 20;
      const curve = new Float32Array(256);
      for (let i = 0; i < 256; i++) {
        const x = (i * 2) / 256 - 1;
        curve[i] = ((Math.PI + k) * x) / (Math.PI + k * Math.abs(x));
      }
      ws.curve = curve;
      osc.disconnect();
      osc.connect(ws);
      ws.connect(this.out);
    }
  },

  score() {
    if (!this.ctx) return;
    const notes = [523, 659, 784]; // C5, E5, G5
    notes.forEach((f, i) => {
      const delay = i * 0.05;
      const t = this.ctx.currentTime + delay;
      const g = this.ctx.createGain();
      g.gain.setValueAtTime(0.15, t);
      g.gain.exponentialRampToValueAtTime(0.001, t + 0.12);
      g.connect(this.out);
      const o = this.ctx.createOscillator();
      o.type = 'triangle';
      o.frequency.setValueAtTime(f, t);
      o.connect(g);
      o.start(t);
      o.stop(t + 0.12);
    });
  },

  die() {
    if (!this.ctx) return;
    // Noise burst
    this._noise(0.3, 0.25, 2000);
    // Falling pitch
    const { osc, t } = this._osc('sawtooth', 400, 0.35, 0.2);
    osc.frequency.exponentialRampToValueAtTime(80, t + 0.3);
  },

  start() {
    if (!this.ctx) return;
    const { osc, t } = this._osc('square', 200, 0.25, 0.12);
    osc.frequency.exponentialRampToValueAtTime(600, t + 0.2);
  },

  charSwitch() {},
  whoosh() {},
  fanfare() {},
  particleTick() {},
  idleHum: { start() {}, stop() {} },
};
```

**Step 2: Wire SFX triggers into existing game code**

In `flap()` function: after `game.bird.vy = JUMP_VEL;` add:
```javascript
SFX.flap(game.charIndex);
```

In `flap()` function: after `game.state = 'playing';` add:
```javascript
Audio.setState('playing');
SFX.start();
```

In `killBird()`: after `game.flash = 0.7;` add:
```javascript
Audio.setState('dead');
SFX.die();
```

In `resetGame()`: after `game.flash = 0;` add:
```javascript
Audio.setState('idle');
```

In `update()` where scoring happens (the `game.score++;` line): after `game.score++;` add:
```javascript
SFX.score();
```

**Step 3: Verify in browser**

Play the game. Verify:
- Flap sound on each jump (varies per character)
- Rising sweep on game start
- Score ding when passing pipes
- Death crunch on collision
- No audio glitches or errors

**Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add core SFX — flap (per-character), score, death, game start"
```

---

### Task 3: Secondary SFX — Character Switch, Whoosh, Fanfare, Particle Tick, Idle Hum

**Files:**
- Modify: `index.html` — fill in remaining SFX stubs

**Step 1: Implement remaining SFX methods**

Replace the empty stubs in the SFX object:

```javascript
charSwitch() {
  if (!this.ctx) return;
  this._osc('square', 800, 0.03, 0.1);
},

whoosh() {
  if (!this.ctx) return;
  this._noise(0.08, 0.04, 1500);
},

fanfare() {
  if (!this.ctx) return;
  const notes = [523, 659, 784, 1047]; // C5 E5 G5 C6
  notes.forEach((f, i) => {
    const t = this.ctx.currentTime + i * 0.08;
    const g = this.ctx.createGain();
    g.gain.setValueAtTime(0.15, t);
    g.gain.exponentialRampToValueAtTime(0.001, t + 0.2);
    g.connect(this.out);
    const o = this.ctx.createOscillator();
    o.type = 'square';
    o.frequency.setValueAtTime(f, t);
    o.connect(g);
    o.start(t);
    o.stop(t + 0.2);
  });
},

particleTick() {
  if (!this.ctx) return;
  this._osc('sine', 1200 + Math.random() * 400, 0.02, 0.02);
},
```

Replace the `idleHum` stub with:

```javascript
idleHum: {
  _osc: null, _gain: null,
  start(sfx) {
    if (this._osc) return;
    const t = sfx.ctx.currentTime;
    this._gain = sfx.ctx.createGain();
    this._gain.gain.setValueAtTime(0, t);
    this._gain.gain.linearRampToValueAtTime(0.06, t + 1);
    this._gain.connect(sfx.out);
    this._osc = sfx.ctx.createOscillator();
    this._osc.type = 'sine';
    this._osc.frequency.value = 55;
    const lfo = sfx.ctx.createOscillator();
    const lfoG = sfx.ctx.createGain();
    lfo.frequency.value = 0.3;
    lfoG.gain.value = 4;
    lfo.connect(lfoG);
    lfoG.connect(this._osc.frequency);
    lfo.start(t);
    this._lfo = lfo;
    this._osc.connect(this._gain);
    this._osc.start(t);
  },
  stop(sfx) {
    if (!this._osc) return;
    const t = sfx.ctx.currentTime;
    this._gain.gain.linearRampToValueAtTime(0, t + 0.5);
    this._osc.stop(t + 0.6);
    this._lfo.stop(t + 0.6);
    this._osc = null;
    this._lfo = null;
    this._gain = null;
  },
},
```

**Step 2: Wire remaining SFX triggers**

In `setChar()` function, after `localStorage.setItem(...)`:
```javascript
SFX.charSwitch();
```

In `update()` where pipe scoring happens, after `SFX.score();`:
```javascript
if (game.score > game.best) SFX.fanfare();
```

Wait — best score is checked in `killBird()`, not at scoring time. So instead, in `killBird()` after `localStorage.setItem('flappy_best', game.best);` add:
```javascript
SFX.fanfare();
```

In `update()` where pipe passes bird (we need to detect pipe passing bird.x, not just scoring). Add after the pipe scoring block inside the `for (const p of game.pipes)` loop, after the scoring `if` block:
```javascript
// Whoosh: pipe just passed bird
if (p.scored && !p.whooshed && p.x + p.w < game.bird.x) {
  p.whooshed = true;
  SFX.whoosh();
}
```

In `update()` where particles spawn (inside `if (game.frame % 2 === 0)`), add after `game.particles.push(...)`:
```javascript
SFX.particleTick();
```

In `Audio.setState()`, manage idle hum:
```javascript
setState(state) {
  if (!this.ready) return;
  if (state === 'idle') SFX.idleHum.start(SFX);
  else SFX.idleHum.stop(SFX);
  Music.setState(state);
},
```

Also start idle hum in `Audio.init()` since game starts in idle:
```javascript
// At end of init(), after this.ready = true:
if (game.state === 'idle') SFX.idleHum.start(SFX);
```

**Step 3: Verify in browser**

- Character switch: click to hear short blip
- Pipe whoosh: subtle noise as pipes pass behind bird
- Best score fanfare: beat your best, hear ascending melody on death
- Particle tick: very quiet subtle ticks during play
- Idle hum: low drone on title screen, stops when playing

**Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add secondary SFX — char switch, whoosh, fanfare, particle tick, idle hum"
```

---

### Task 4: Music Sequencer Infrastructure + Pad Layer

**Files:**
- Modify: `index.html` — replace Music stub with sequencer + pad

**Step 1: Implement Music object with sequencer and pad**

Replace the `Music` stub with:

```javascript
const Music = {
  ctx: null, out: null,
  bpm: 110,
  step: 0,
  stepTime: 0,
  state: 'idle',
  // Node references
  padOscs: null, padGain: null, padFilter: null,

  // Chord progression: i → VI → III → VII in A minor
  // Am (A3 C4 E4), F (F3 A3 C4), C (C3 E3 G3), G (G3 B3 D4)
  chords: [
    [220.00, 261.63, 329.63],   // Am
    [174.61, 220.00, 261.63],   // F
    [130.81, 164.81, 196.00],   // C
    [196.00, 246.94, 293.66],   // G
  ],

  init(ctx, out) {
    this.ctx = ctx;
    this.out = out;
    this.sixteenth = 60 / this.bpm / 4; // duration of one 16th note
  },

  setState(newState) {
    const prev = this.state;
    this.state = newState;
    if (!this.ctx) return;

    if (newState === 'idle') {
      this._startPad();
      this._stopRhythm();
      this._filterTo(800, 1.0);
    } else if (newState === 'playing') {
      if (!this.padOscs) this._startPad();
      this._startRhythm();
      this._filterTo(4000, 0.5);
    } else if (newState === 'dead') {
      this._stopRhythm();
      this._filterTo(300, 2.0);
      this._detunePad(15);
      // Restore after transition
      setTimeout(() => { if (this.state === 'dead') this._detunePad(0); }, 1500);
    }
  },

  _startPad() {
    if (this.padOscs) return;
    const t = this.ctx.currentTime;

    this.padFilter = this.ctx.createBiquadFilter();
    this.padFilter.type = 'lowpass';
    this.padFilter.frequency.value = 800;
    this.padFilter.Q.value = 1;
    this.padFilter.connect(this.out);

    this.padGain = this.ctx.createGain();
    this.padGain.gain.setValueAtTime(0, t);
    this.padGain.gain.linearRampToValueAtTime(0.12, t + 1);
    this.padGain.connect(this.padFilter);

    // Two detuned saws per chord note = warm pad
    const chord = this.chords[0];
    this.padOscs = [];
    chord.forEach(freq => {
      for (const detune of [-8, 8]) {
        const o = this.ctx.createOscillator();
        o.type = 'sawtooth';
        o.frequency.value = freq;
        o.detune.value = detune;
        o.connect(this.padGain);
        o.start(t);
        this.padOscs.push(o);
      }
    });

    // Start chord cycling
    this._chordIndex = 0;
    this._scheduleChordChange();
  },

  _scheduleChordChange() {
    const barDuration = this.sixteenth * 16;
    this._chordTimer = setInterval(() => {
      this._chordIndex = (this._chordIndex + 1) % this.chords.length;
      const chord = this.chords[this._chordIndex];
      const t = this.ctx.currentTime;
      if (!this.padOscs) return;
      // Update frequencies: 2 oscillators per note, 3 notes
      chord.forEach((freq, ni) => {
        const o1 = this.padOscs[ni * 2];
        const o2 = this.padOscs[ni * 2 + 1];
        if (o1) o1.frequency.linearRampToValueAtTime(freq, t + 0.3);
        if (o2) o2.frequency.linearRampToValueAtTime(freq, t + 0.3);
      });
    }, barDuration * 1000);
  },

  _detunePad(cents) {
    if (!this.padOscs) return;
    const t = this.ctx.currentTime;
    this.padOscs.forEach((o, i) => {
      const base = (i % 2 === 0) ? -8 : 8;
      o.detune.linearRampToValueAtTime(base + cents, t + 0.5);
    });
  },

  _filterTo(freq, duration) {
    if (!this.padFilter) return;
    const t = this.ctx.currentTime;
    this.padFilter.frequency.linearRampToValueAtTime(freq, t + duration);
  },

  // Rhythm stubs — filled in Task 5
  _startRhythm() {},
  _stopRhythm() {},
};
```

**Step 2: Verify in browser**

- Start game → idle state should have warm pad drone cycling through Am→F→C→G
- Play → pad filter opens (brighter sound)
- Die → pad filter sweeps down, slight detune
- Restart → back to mellow idle pad

**Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add Music sequencer with adaptive pad layer and chord progression"
```

---

### Task 5: Music Rhythm Layers — Kick, Hi-hat, Bass

**Files:**
- Modify: `index.html` — replace rhythm stubs in Music object

**Step 1: Add rhythm scheduling and layer implementations**

Add these properties to the Music object (after the `chords` array):

```javascript
rhythmTimer: null,
kickGain: null,
hatGain: null,
bassOsc: null, bassGain: null, bassFilter: null,
```

Replace `_startRhythm` and `_stopRhythm` stubs:

```javascript
_startRhythm() {
  if (this.rhythmTimer) return;
  const t = this.ctx.currentTime;

  // Bass setup: continuous osc, we modulate frequency per step
  this.bassFilter = this.ctx.createBiquadFilter();
  this.bassFilter.type = 'lowpass';
  this.bassFilter.frequency.value = 600;
  this.bassFilter.connect(this.out);
  this.bassGain = this.ctx.createGain();
  this.bassGain.gain.value = 0;
  this.bassGain.connect(this.bassFilter);
  this.bassOsc = this.ctx.createOscillator();
  this.bassOsc.type = 'sawtooth';
  this.bassOsc.frequency.value = this.chords[0][0] / 2;
  this.bassOsc.connect(this.bassGain);
  this.bassOsc.start(t);

  this.step = 0;
  const stepDur = this.sixteenth;

  this.rhythmTimer = setInterval(() => {
    const now = this.ctx.currentTime;
    const s = this.step % 16;

    // Kick on 0, 4, 8, 12
    if (s % 4 === 0) this._kick(now);

    // Hi-hat on every even step, accent on off-beats
    if (s % 2 === 0) this._hat(now, s % 4 === 2 ? 0.08 : 0.05);

    // Bass: 8th notes (every 2 steps)
    if (s % 2 === 0) {
      const chord = this.chords[this._chordIndex || 0];
      const root = chord[0] / 2; // one octave below
      this.bassGain.gain.setValueAtTime(0.13, now);
      this.bassGain.gain.exponentialRampToValueAtTime(0.001, now + stepDur * 1.8);
      this.bassOsc.frequency.setValueAtTime(root, now);
    }

    this.step++;
  }, stepDur * 1000);
},

_stopRhythm() {
  if (this.rhythmTimer) {
    clearInterval(this.rhythmTimer);
    this.rhythmTimer = null;
  }
  if (this.bassOsc) {
    this.bassOsc.stop(this.ctx.currentTime + 0.1);
    this.bassOsc = null;
    this.bassGain = null;
    this.bassFilter = null;
  }
},

_kick(t) {
  const o = this.ctx.createOscillator();
  o.type = 'sine';
  o.frequency.setValueAtTime(160, t);
  o.frequency.exponentialRampToValueAtTime(40, t + 0.08);
  const g = this.ctx.createGain();
  g.gain.setValueAtTime(0.25, t);
  g.gain.exponentialRampToValueAtTime(0.001, t + 0.15);
  o.connect(g);
  g.connect(this.out);
  o.start(t);
  o.stop(t + 0.15);
},

_hat(t, vol) {
  const sr = this.ctx.sampleRate;
  const dur = 0.04;
  const buf = this.ctx.createBuffer(1, sr * dur, sr);
  const d = buf.getChannelData(0);
  for (let i = 0; i < d.length; i++) d[i] = Math.random() * 2 - 1;
  const src = this.ctx.createBufferSource();
  src.buffer = buf;
  const hp = this.ctx.createBiquadFilter();
  hp.type = 'highpass';
  hp.frequency.value = 7000;
  const g = this.ctx.createGain();
  g.gain.setValueAtTime(vol, t);
  g.gain.exponentialRampToValueAtTime(0.001, t + dur);
  src.connect(hp);
  hp.connect(g);
  g.connect(this.out);
  src.start(t);
  src.stop(t + dur);
},
```

**Step 2: Verify in browser**

- Start playing → kick drum on the beat, hi-hats in between, bass following chord roots
- Die → rhythm stops cleanly
- Restart → back to idle (pad only, no rhythm)
- Play again → rhythm restarts

**Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add rhythm layers — kick, hi-hat, bass with chord-following"
```

---

### Task 6: Music Arp Layer

**Files:**
- Modify: `index.html` — add arp layer to Music object rhythm scheduling

**Step 1: Add arp to the rhythm scheduler**

Add properties to Music object:

```javascript
arpGain: null, arpFilter: null,
```

In `_startRhythm()`, after bass setup (before `this.step = 0`), add:

```javascript
// Arp setup
this.arpFilter = this.ctx.createBiquadFilter();
this.arpFilter.type = 'lowpass';
this.arpFilter.frequency.value = 3000;
this.arpFilter.connect(this.out);
this.arpGain = this.ctx.createGain();
this.arpGain.gain.value = 0.08;
this.arpGain.connect(this.arpFilter);
```

Inside the `setInterval` callback, add after the bass block:

```javascript
// Arp: 16th notes, arpeggiate chord tones up an octave
const chord = this.chords[this._chordIndex || 0];
const arpNote = chord[s % 3] * 2; // cycle through chord tones, up one octave
this._arpNote(this.ctx.currentTime, arpNote, stepDur * 0.8);
```

Add the arp note method:

```javascript
_arpNote(t, freq, dur) {
  const o = this.ctx.createOscillator();
  o.type = 'square';
  o.frequency.setValueAtTime(freq, t);
  const g = this.ctx.createGain();
  g.gain.setValueAtTime(0.08, t);
  g.gain.exponentialRampToValueAtTime(0.001, t + dur);
  o.connect(g);
  g.connect(this.arpFilter || this.out);
  o.start(t);
  o.stop(t + dur);
},
```

In `_stopRhythm()`, clean up arp nodes:

```javascript
this.arpGain = null;
this.arpFilter = null;
```

**Step 2: Add idle arp (gentle, slow)**

In `setState('idle')` block, schedule a slow gentle arp. Add a separate idle arp timer:

```javascript
// In setState, 'idle' block:
this._startIdleArp();

// In setState, 'playing' block (before _startRhythm):
this._stopIdleArp();

// In setState, 'dead' block:
this._stopIdleArp();
```

Add methods:

```javascript
_idleArpTimer: null,

_startIdleArp() {
  if (this._idleArpTimer) return;
  let step = 0;
  this._idleArpTimer = setInterval(() => {
    const chord = this.chords[this._chordIndex || 0];
    const freq = chord[step % 3] * 2;
    const t = this.ctx.currentTime;
    const o = this.ctx.createOscillator();
    o.type = 'triangle';
    o.frequency.value = freq;
    const g = this.ctx.createGain();
    g.gain.setValueAtTime(0.04, t);
    g.gain.exponentialRampToValueAtTime(0.001, t + 0.4);
    o.connect(g);
    g.connect(this.padFilter || this.out);
    o.start(t);
    o.stop(t + 0.4);
    step++;
  }, this.sixteenth * 4 * 1000); // quarter note pace
},

_stopIdleArp() {
  if (this._idleArpTimer) {
    clearInterval(this._idleArpTimer);
    this._idleArpTimer = null;
  }
},
```

**Step 3: Verify in browser**

- Idle: gentle slow arpeggiated notes over the pad
- Playing: fast 16th note arp, bright and energetic
- Die: arp stops with rhythm

**Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add arp layer — fast 16th notes in play, gentle quarter notes in idle"
```

---

### Task 7: Polish and Tuning

**Files:**
- Modify: `index.html` — volume balancing, transition smoothing, edge cases

**Step 1: Add master volume balancing**

Review all gain values and ensure the mix is balanced. Key targets:
- Pad: 0.12 (background wash)
- Kick: 0.25 (punchy)
- Hi-hat: 0.05-0.08 (subtle texture)
- Bass: 0.13 (present but not overpowering)
- Arp: 0.08 (melodic interest)
- SFX flap: 0.18 (clear feedback)
- SFX score: 0.15 (rewarding ding)
- SFX death: 0.2-0.25 (dramatic)
- Master gain: 0.5 (overall headroom)

Adjust any values that sound off after testing.

**Step 2: Handle edge cases**

- Multiple rapid flaps: each `flap()` call creates new nodes; they naturally overlap and decay. Fine.
- Tab switching: AudioContext may suspend. Add resume check in the game loop:

In `Audio` object, add:
```javascript
resume() {
  if (this.ctx && this.ctx.state === 'suspended') this.ctx.resume();
},
```

Call `Audio.resume()` at the top of each input handler (after `Audio.init()`).

- Prevent `exponentialRampToValueAtTime` error by ensuring values never reach exactly 0 (use 0.001).

**Step 3: Smooth idle→playing transition timing**

The pad filter and rhythm should transition smoothly. Add a 0.5s ramp for rhythm layer volumes when starting:

In `_startRhythm()`, after creating `bassGain`:
```javascript
// Fade in over 0.5s
this.bassGain.gain.setValueAtTime(0, this.ctx.currentTime);
```

The per-note gain envelopes already handle this naturally, so the fade-in happens organically as steps begin playing.

**Step 4: Verify in browser**

- Play through multiple full game cycles (idle → play → die → idle → play)
- Verify no crackling, no stuck oscillators, no audio buildup
- Test rapid tapping, character switching during idle
- Test tab switch and return

**Step 5: Commit**

```bash
git add index.html
git commit -m "feat: polish audio — volume balance, tab resume, transition smoothing"
```

---

## Summary

| Task | Description | Est. Lines |
|------|-------------|-----------|
| 1 | Audio coordinator + stubs + init wiring | ~30 |
| 2 | Core SFX (flap, score, die, start) + triggers | ~120 |
| 3 | Secondary SFX (switch, whoosh, fanfare, tick, hum) + triggers | ~60 |
| 4 | Music sequencer + pad layer | ~90 |
| 5 | Music rhythm layers (kick, hat, bass) | ~80 |
| 6 | Music arp layer (play + idle) | ~50 |
| 7 | Polish, volume balance, edge cases | ~15 |

Total: ~445 lines of audio code added to `index.html`.
