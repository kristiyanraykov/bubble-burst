# Bubble Burst Game — Build Reference

## Overview

A browser-based soap bubble popping game. Translucent bubbles float upward across the screen with gentle physics, and the player taps/clicks them to pop. Scoring rewards speed (streak multiplier) and precision (smaller bubbles = more points).

## Tech Stack

| Layer | Tool | Why |
|-------|------|-----|
| Rendering | **Pixi.js 7** (Canvas/WebGL) | Hardware-accelerated 2D. Handles hundreds of layered transparent sprites without breaking a sweat. |
| Animation/Polish | **GSAP 3** | Pop effects, particle bursts, score popups, spawn animations, combo text. Runs alongside the game loop as a reactive presentation layer. |
| UI | **Vanilla DOM + CSS** | Score, streak, popped count. No framework needed for a handful of elements. |
| Game Loop | **Pixi ticker** (`app.ticker.add`) | Drives bubble movement and physics each frame. GSAP handles its own internal ticker for tweens. |

### Why Not React

React's reconciler and virtual DOM add overhead with no benefit here. All game state lives outside the DOM — bubbles are Pixi display objects updated in a RAF loop. The UI is four text elements. A framework would only introduce a boundary to manage between React's world and the game loop.

### Why Not Raw Canvas

Pixi gives us WebGL acceleration, display object hierarchy (layers), built-in hit testing for pointer events on individual bubbles, and alpha blending that looks correct. Doing that from scratch with `CanvasRenderingContext2D` is a lot of boilerplate for worse performance.

## Architecture

```
Single HTML file (prototype)
├── Pixi Application (full viewport)
│   ├── bgLayer        — ambient glow spots
│   ├── bubbleLayer     — live bubble containers
│   └── fxLayer         — pop particles, rings, score text
├── DOM HUD overlay     — score, streak, popped count
└── Game loop (ticker)  — movement, spawning, cleanup
```

### Key Separation

- **Simulation** (ticker): bubble position, velocity, wobble, wall bounce, spawn timing, streak timeout. Pure data updates every frame.
- **Presentation** (GSAP): triggered reactively on events (spawn, pop). GSAP tweens Pixi display object properties (`scale`, `alpha`, `x`, `y`) directly. Never drives the core loop.
- **UI** (DOM): updated via simple `textContent` assignments when score/streak changes.

## Bubble Rendering

Each bubble is a `PIXI.Container` with layered graphics to simulate soap film:

1. **Body** — circle filled at ~12% opacity with the base color
2. **Rim** — 1.5px stroke in a complementary hue at 30% opacity
3. **Inner ring** — thinner stroke at a shifted hue, 15% opacity
4. **Specular highlight** — white ellipse, rotated, top-left, 35% opacity
5. **Secondary highlight** — smaller white ellipse, top-right
6. **Color reflection** — shifted-hue ellipse, bottom-right, 10% opacity

Colors are generated via HSL with random hue, then converted to hex for Pixi. The hue is shifted +40° for shine and +180° for the rim to mimic iridescence.

## Bubble Physics

Simple but effective floating behavior:

- **Base velocity**: slight upward drift (`vy` negative), minor horizontal component
- **Wobble**: sinusoidal offset on X using per-bubble `wobbleSpeed`, `wobbleAmp`, and `wobbleOffset`
- **Breathing**: scale oscillates ±3% over time
- **Wall bounce**: soft reflection off left/right edges with 0.8 damping
- **Despawn**: removed when they float off the top of the viewport

No collision between bubbles — they pass through each other, which actually feels natural for soap bubbles.

## Pop Effect Sequence

All GSAP, triggered when a bubble's `pointerdown` fires:

1. **White flash** — circle at pop location, scales to 1.4x and fades over 0.25s
2. **Expanding ring** — colored stroke circle, scales to 2.2x and fades over 0.5s
3. **Droplet particles** — 8–14 small colored circles scatter outward with slight downward gravity, shrink and fade over 0.4–0.7s
4. **Score popup** — text rises 60px and fades, with elastic scale-in
5. **Bubble removal** — quick scale to 1.3x then alpha to 0

Each droplet gets a slightly shifted hue and its own tiny specular highlight for a wet/glassy look.

## Scoring System

- **Base points**: `Math.round(80 / radius * 10)` — smaller bubbles score higher
- **Streak multiplier**: pops within 1200ms of each other increment the streak (capped at 10x)
- **Streak decay**: resets to 0 after 1200ms of inactivity
- **Combo labels**: at streak ≥3, centered text appears ("Nice!", "Amazing!", up to "BUBBLE GOD" at 10)

## Spawning

- New bubble every 800ms from below the viewport
- Capped at 18 simultaneous live bubbles
- 10 bubbles seeded at random positions on load for immediate gameplay
- Spawn animation: elastic scale-in from 0 + alpha fade-in

## Extending This

### Audio (Howler.js)

```js
const popSound = new Howl({
  src: ['pop.mp3'],
  volume: 0.4,
  rate: 0.8 + Math.random() * 0.4, // pitch variation
});
// Call popSound.play() inside popBubble()
```

### Splitting Bubbles

On pop, spawn 2–3 smaller bubbles at the pop location with outward velocity. Recursion stops below a minimum radius.

### Timed Challenge Mode

Add a 60-second countdown. Track total score. Show a results screen with stats (biggest streak, average bubble size, accuracy if you track misses).

### Converting to Vite + TypeScript

The single-file structure maps directly to modules:

```
src/
  main.ts           — Pixi app init, layer setup, game loop
  bubble.ts          — createBubble(), bubble interface/type
  physics.ts         — updateBubble(), wall bounce, despawn
  effects.ts         — popBubble() with all GSAP sequences
  scoring.ts         — streak logic, point calculation
  ui.ts              — DOM updates, combo display
  utils.ts           — hslToHex, constants
```

Install deps:

```bash
npm create vite@latest bubble-burst -- --template vanilla-ts
cd bubble-burst
npm install pixi.js gsap
# optional: npm install howler
```

### Performance Notes

- Pixi's WebGL renderer handles the transparency layering efficiently
- GSAP tweens are fire-and-forget with `onComplete` cleanup — no memory leaks
- Dead bubbles are filtered from the array every ~2 seconds (ticker frame 120 modulo)
- Resize listener updates the renderer dimensions but doesn't recreate bubbles
