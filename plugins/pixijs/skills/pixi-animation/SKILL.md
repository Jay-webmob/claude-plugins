---
name: pixi-animation
description: Animating in PixiJS v8 — the ticker and deltaTime, AnimatedSprite and spritesheet frame animation, tweening, easing, and timeline sequencing. Use when animating sprites, building frame-by-frame character animation, or moving anything over time on a Pixi canvas.
---

# PixiJS v8 Animation

Two kinds of animation: **frame animation** (swapping textures, e.g. a walk cycle) and
**property animation** (interpolating position/scale/alpha over time). Both are driven by the
ticker.

## The ticker and deltaTime

```js
app.ticker.add((ticker) => {
  sprite.x += 2 * ticker.deltaTime;
});
```

The callback receives a `Ticker` object, not a bare number.

| Property | Meaning |
|---|---|
| `ticker.deltaTime` | Frames elapsed, normalised to 60fps (`1.0` at 60fps, `2.0` at 30fps) |
| `ticker.deltaMS` | Milliseconds since last frame — use for timers and physics |
| `ticker.lastTime` | Total ms since the ticker started |
| `ticker.FPS` | Current measured frame rate |

**Always scale motion by `deltaTime`** (or `deltaMS`). Unscaled, `sprite.x += 2` runs twice as
fast on a 120Hz monitor and crawls on a slow frame.

```js
// ✗ Speed depends on the user's refresh rate
sprite.x += 2;

// ✓ Consistent regardless of frame rate
sprite.x += 2 * ticker.deltaTime;

// ✓ Explicit units — 120 pixels per second
sprite.x += 120 * (ticker.deltaMS / 1000);
```

Cap the frame rate when you don't need 120fps:

```js
app.ticker.maxFPS = 60;
```

Always remove callbacks on teardown — an orphaned ticker callback keeps a whole scene alive:

```js
const onTick = (ticker) => { /* … */ };
app.ticker.add(onTick);
app.ticker.remove(onTick);
```

## AnimatedSprite

```js
import { Assets, AnimatedSprite } from 'pixi.js';

const sheet = await Assets.load('sprites/hero.json');

const run = new AnimatedSprite(sheet.animations['hero_run']);
run.animationSpeed = 0.2;      // fraction of a frame advanced per tick
run.loop = true;
run.anchor.set(0.5);
run.play();

app.stage.addChild(run);
```

`sheet.animations` is built automatically from frames named with a trailing index
(`hero_run_01.png`, `hero_run_02.png`, …).

| Member | Notes |
|---|---|
| `animationSpeed` | Frames advanced per tick. `1` ≈ 60fps; negative plays in reverse |
| `loop` | Defaults to `true` |
| `onComplete` | Fires when a non-looping animation ends |
| `onLoop` | Fires on each loop restart |
| `onFrameChange` | Fires per frame — useful for footstep sounds or hit frames |
| `currentFrame` | Read the current index |
| `play()` / `stop()` | Start and pause |
| `gotoAndPlay(n)` / `gotoAndStop(n)` | Jump to a frame |

Constructor also accepts an options object:

```js
const explosion = new AnimatedSprite({
  textures: sheet.animations['explosion'],
  animationSpeed: 0.4,
  loop: false,
});

explosion.onComplete = () => explosion.destroy();
explosion.play();
```

## Swapping animation states

Keep one `AnimatedSprite` and swap `textures` rather than creating a new sprite per state.

```js
class Hero extends AnimatedSprite {
  constructor(sheet) {
    super(sheet.animations['hero_idle']);
    this.sheet = sheet;
    this.state = 'idle';
    this.anchor.set(0.5, 1);
    this.animationSpeed = 0.15;
    this.play();
  }

  setState(name) {
    if (this.state === name) return;      // don't restart the same animation
    this.state = name;
    this.textures = this.sheet.animations[`hero_${name}`];
    this.play();
  }
}

hero.setState(isMoving ? 'run' : 'idle');
```

The `if (this.state === name) return` guard matters — reassigning `textures` every frame resets
the animation to frame 0 and the sprite appears frozen.

## Tweening

Pixi has no built-in tween system. For a few simple moves, hand-roll it:

```js
function tween({ from, to, duration, ease = (t) => t, onUpdate, onComplete }) {
  let elapsed = 0;

  const step = (ticker) => {
    elapsed += ticker.deltaMS;
    const t = Math.min(elapsed / duration, 1);
    onUpdate(from + (to - from) * ease(t));

    if (t === 1) {
      app.ticker.remove(step);
      onComplete?.();
    }
  };

  app.ticker.add(step);
  return () => app.ticker.remove(step);   // cancel handle
}

tween({
  from: sprite.x,
  to: 400,
  duration: 800,
  ease: (t) => 1 - Math.pow(1 - t, 3),    // easeOutCubic
  onUpdate: (v) => { sprite.x = v; },
});
```

Useful easings:

```js
const easeInQuad    = (t) => t * t;
const easeOutQuad   = (t) => t * (2 - t);
const easeInOutQuad = (t) => (t < 0.5 ? 2 * t * t : -1 + (4 - 2 * t) * t);
const easeOutCubic  = (t) => 1 - Math.pow(1 - t, 3);
const easeOutBack   = (t) => 1 + 2.70158 * Math.pow(t - 1, 3) + 1.70158 * Math.pow(t - 1, 2);
const easeOutElastic = (t) =>
  t === 0 || t === 1 ? t : Math.pow(2, -10 * t) * Math.sin((t * 10 - 0.75) * ((2 * Math.PI) / 3)) + 1;
```

For anything with sequencing, staggering, or timelines, use **GSAP** — it tweens plain object
properties, so it drives Pixi objects directly:

```js
import gsap from 'gsap';

gsap.to(sprite, { x: 400, duration: 0.8, ease: 'power3.out' });
gsap.to(sprite.scale, { x: 1.5, y: 1.5, duration: 0.4, yoyo: true, repeat: -1 });

const tl = gsap.timeline();
tl.to(logo, { alpha: 1, duration: 0.5 })
  .to(logo.scale, { x: 1.2, y: 1.2, duration: 0.3 }, '-=0.2')
  .to(subtitle, { alpha: 1, duration: 0.4 });
```

Kill GSAP tweens on teardown — `gsap.killTweensOf(sprite)` — or they keep referencing destroyed
objects and throw on the next frame.

## Common motion patterns

```js
let elapsed = 0;

app.ticker.add((ticker) => {
  elapsed += ticker.deltaMS;
  const t = elapsed / 1000;

  bob.y      = baseY + Math.sin(t * 2) * 20;                       // float up and down
  pulse.scale.set(1 + Math.sin(t * 4) * 0.1);                      // pulse
  orbiter.x  = cx + Math.cos(t) * radius;                          // orbit
  orbiter.y  = cy + Math.sin(t) * radius;
  spinner.rotation += 0.05 * ticker.deltaTime;                     // constant spin

  // Frame-rate-independent smoothing toward a target
  const smoothing = 1 - Math.pow(0.001, ticker.deltaMS / 1000);
  follower.x += (target.x - follower.x) * smoothing;
});
```

Plain `follower.x += (target.x - follower.x) * 0.1` is the common shortcut, but its speed varies
with frame rate; the `Math.pow` form above does not.

## Delta spikes

After a tab is backgrounded, the first frame can report a huge delta and teleport everything.
Clamp it:

```js
app.ticker.add((ticker) => {
  const dt = Math.min(ticker.deltaMS, 100);   // never step more than 100ms
  update(dt);
});
```

For deterministic physics, use a fixed timestep — see `pixi-game-loop`.
