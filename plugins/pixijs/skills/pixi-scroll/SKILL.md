---
name: pixi-scroll
description: Scroll-driven animation with PixiJS v8 for animated websites — linking canvas animation to scroll position, GSAP ScrollTrigger integration, pinned sections, image transitions, and reduced-motion support. Use when building a scrollytelling site, a WebGL landing page, or scroll-linked canvas effects.
---

# Scroll-Driven Animation with PixiJS

The pattern behind most "animated website" work: a fixed full-screen canvas whose contents are
driven by scroll progress.

## Structure

Keep the canvas fixed and let normal DOM content scroll over or beside it. Do not try to scroll
the canvas itself.

```html
<div id="canvas-host"></div>
<main>
  <section class="panel">…</section>
  <section class="panel">…</section>
</main>
```

```css
#canvas-host { position: fixed; inset: 0; z-index: 0; pointer-events: none; }
main { position: relative; z-index: 1; }
.panel { min-height: 100vh; }
```

`pointer-events: none` lets clicks reach the DOM. Remove it only if the canvas itself is
interactive.

## Reading scroll progress

Never animate directly inside a scroll event — it fires more often than frames render, and the
result stutters. Record the value, apply it on the ticker.

```js
// ✗ Fires far more often than the render loop
window.addEventListener('scroll', () => {
  sprite.y = window.scrollY * 0.5;
});
```

```js
// ✓ Record in the event, smooth on the ticker
let targetScroll = 0;
let currentScroll = 0;

window.addEventListener('scroll', () => {
  targetScroll = window.scrollY;
}, { passive: true });

app.ticker.add((ticker) => {
  const dt = ticker.deltaMS / 1000;
  const t = 1 - Math.exp(-6 * dt);           // frame-rate-independent smoothing
  currentScroll += (targetScroll - currentScroll) * t;

  const progress = currentScroll / (document.body.scrollHeight - window.innerHeight);
  updateScene(progress);                      // progress is 0..1
});
```

`{ passive: true }` on the listener keeps scrolling off the main thread's critical path.

The smoothing is what gives the "expensive" feel — the canvas trails the DOM slightly rather than
snapping.

## Per-section progress

For scrollytelling, each section needs its own 0→1 range. `IntersectionObserver` tells you which
sections are on screen; `getBoundingClientRect` gives the progress within them.

```js
const sections = [...document.querySelectorAll('.panel')];
const active = new Set();

const observer = new IntersectionObserver(
  (entries) => {
    for (const e of entries) {
      if (e.isIntersecting) active.add(e.target);
      else active.delete(e.target);
    }
  },
  { threshold: 0 }
);

sections.forEach((s) => observer.observe(s));

app.ticker.add(() => {
  for (const section of active) {
    const rect = section.getBoundingClientRect();
    // 0 when the section's top hits the viewport bottom, 1 when its bottom hits the top
    const progress = 1 - (rect.bottom / (window.innerHeight + rect.height));
    updateSection(section.dataset.scene, clamp01(progress));
  }
});
```

Only measuring visible sections keeps this cheap — `getBoundingClientRect` forces layout, so
calling it for fifty off-screen sections every frame is wasteful.

## GSAP ScrollTrigger

For anything with pinning or complex sequencing, ScrollTrigger is worth the dependency. It tweens
plain objects, so it drives Pixi properties directly.

```js
import gsap from 'gsap';
import ScrollTrigger from 'gsap/ScrollTrigger';

gsap.registerPlugin(ScrollTrigger);

gsap.to(sprite, {
  x: 800,
  rotation: Math.PI,
  ease: 'none',
  scrollTrigger: {
    trigger: '#section-2',
    start: 'top bottom',
    end: 'bottom top',
    scrub: 1,              // ties progress to scroll; the number adds smoothing
  },
});
```

`scrub: 1` is the key setting — it links the tween to scroll position with one second of catch-up
smoothing. `scrub: true` links it rigidly.

Pinned sections:

```js
ScrollTrigger.create({
  trigger: '#hero',
  start: 'top top',
  end: '+=2000',
  pin: true,
  scrub: true,
  onUpdate: (self) => {
    updateScene(self.progress);       // 0..1 across the pinned distance
  },
});
```

Refresh after layout changes, and clean up on teardown:

```js
ScrollTrigger.refresh();
ScrollTrigger.getAll().forEach((t) => t.kill());
gsap.killTweensOf(sprite);
```

Killing tweens matters — a live tween holding a destroyed Pixi object throws on the next frame.

## Displacement on scroll

A common effect: warp imagery in proportion to scroll velocity.

```js
import { DisplacementFilter, Assets, Sprite } from 'pixi.js';

const map = new Sprite(await Assets.load('/displacement.jpg'));
map.texture.source.addressMode = 'repeat';
app.stage.addChild(map);

const displacement = new DisplacementFilter({ sprite: map, scale: 0 });
imageContainer.filters = [displacement];

let lastScroll = 0;

app.ticker.add((ticker) => {
  const velocity = Math.abs(currentScroll - lastScroll);
  lastScroll = currentScroll;

  const target = Math.min(velocity * 2, 80);
  displacement.scale.x += (target - displacement.scale.x) * 0.1;
  displacement.scale.y = displacement.scale.x;

  map.x += 0.5 * ticker.deltaTime;
});
```

Filters are expensive (see `pixi-filters`) — apply to a container, not per image.

## Scroll-linked frame sequences

Scrubbing an image sequence is the classic product-reveal effect. Preload every frame first, or it
stutters on the first pass.

```js
const frames = await Assets.load(
  Array.from({ length: 120 }, (_, i) => `/seq/frame_${String(i).padStart(4, '0')}.webp`)
);
const urls = Object.keys(frames);

const sprite = new Sprite(frames[urls[0]]);
app.stage.addChild(sprite);

function updateScene(progress) {
  const index = Math.min(Math.floor(progress * urls.length), urls.length - 1);
  sprite.texture = frames[urls[index]];
}
```

For long sequences, a video texture or a packed spritesheet uses far less memory than 120 separate
textures.

## Parallax layers

```js
const layers = [
  { container: skyLayer,   factor: 0.1 },
  { container: midLayer,   factor: 0.4 },
  { container: frontLayer, factor: 0.9 },
];

function updateScene() {
  for (const { container, factor } of layers) {
    container.y = -currentScroll * factor;
  }
}
```

## Reduced motion

Large scroll-driven motion triggers vestibular disorders. Respect the OS setting — this is an
accessibility requirement, not a nicety.

```js
const reduceMotion = window.matchMedia('(prefers-reduced-motion: reduce)').matches;

if (reduceMotion) {
  app.ticker.stop();
  renderStaticPoster();          // show the final composition, no animation
} else {
  startScrollAnimation();
}
```

Keep the content legible without the animation. If the canvas carries meaning, provide a DOM
equivalent.

## Performance

- Cap resolution: `resolution: Math.min(devicePixelRatio, 2)`.
- Pause when off-screen — an `IntersectionObserver` on the host calling `app.ticker.stop()`.
- Don't rebuild `Graphics` per frame; transform instead (see `pixi-graphics`).
- Lazy-load the Pixi bundle so it never blocks first paint:

```js
const { initScene } = await import('./scene.js');
```

- Test on a mid-range phone. A scroll site that stutters on mobile is worse than a static image.
