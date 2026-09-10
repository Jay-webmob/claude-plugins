---
name: motion-smooth-scroll
description: Smooth scrolling with Lenis — setup, all constructor options, GSAP ScrollTrigger wiring, syncing with a canvas RAF loop, and the accessibility tradeoffs. Use when adding smooth or inertial scroll, building scroll-driven animation, or synchronizing scroll with a canvas or WebGL scene.
---

# Smooth Scroll with Lenis

Lenis intercepts wheel/touch input and interpolates the scroll position, giving inertial scrolling
and — more usefully — a single RAF-driven source of scroll truth that animation and canvas work can
sync to.

**Current version: 1.3.26.** `npm i lenis`

## Setup

```js
import Lenis from 'lenis';
import 'lenis/dist/lenis.css';

const lenis = new Lenis();

function raf(time) {
  lenis.raf(time);
  requestAnimationFrame(raf);
}

requestAnimationFrame(raf);
```

`autoRaf` defaults to **`false`** — you must drive `raf()` yourself, or pass `autoRaf: true`. A
Lenis instance that appears to do nothing is almost always a missing RAF loop.

## Options and defaults

| Option | Default | Notes |
|---|---|---|
| `duration` | `1.2` | Seconds to settle. Ignored when `lerp` is set |
| `lerp` | `0.1` | Linear interpolation factor; alternative to `duration` |
| `easing` | `(t) => Math.min(1, 1.001 - Math.pow(2, -10 * t))` | Exponential ease-out |
| `smoothWheel` | `true` | Smooth mouse wheel |
| `syncTouch` | `false` | Smooth touch. **Leave off** — see below |
| `syncTouchLerp` | `0.075` | Lerp when `syncTouch` is on |
| `touchMultiplier` | `1` | Touch delta scaling |
| `wheelMultiplier` | `1` | Wheel delta scaling |
| `orientation` | `'vertical'` | `'vertical'` \| `'horizontal'` |
| `gestureOrientation` | `'vertical'` | Which gestures are captured |
| `wrapper` | `window` | Scroll container |
| `content` | `document.documentElement` | Scrolled content |
| `autoRaf` | `false` | Drive `raf()` internally |
| `autoResize` | `true` | Watch for size changes |
| `overscroll` | `true` | Allow overscroll behaviour |
| `infinite` | `false` | Loop scrolling |
| `anchors` | `false` | Handle anchor links |
| `allowNestedScroll` | `false` | Let nested scrollers work |
| `prevent` | `undefined` | `(node) => boolean` to exclude subtrees |
| **`respectReducedMotion`** | **`true`** | Honours `prefers-reduced-motion` |

Use **either** `duration` **or** `lerp`, not both — `duration` is ignored when `lerp` is set.

## Accessibility

`respectReducedMotion` defaults to `true`. Under `prefers-reduced-motion: reduce`, Lenis forces
lerp to 1 and makes programmatic scrolls jump instantly, while continuing to run so anything synced
to it stays in sync. **Do not set this to `false`.**

Honest caveats about smooth scroll generally:

- It **overrides a deliberate OS/browser behaviour** users have muscle memory for.
- Hijacked scroll can feel laggy or "slippery", and some users find it nauseating.
- Native scrollbar dragging, keyboard paging, and find-in-page can behave unexpectedly.
- It adds a dependency and a per-frame cost to something the browser does for free.

Use it when scroll-linked animation genuinely needs a smoothed, frame-synced value. Don't add it to
a content site for decoration.

Let nested scrollers keep native behaviour:

```js
const lenis = new Lenis({
  prevent: (node) => node.classList.contains('native-scroll') || node.closest('[data-lenis-prevent]'),
});
```

Lenis honours `data-lenis-prevent` on a scrollable element out of the box.

## GSAP ScrollTrigger

The exact wiring — Lenis drives GSAP's ticker, and ScrollTrigger updates on Lenis's scroll event:

```js
import gsap from 'gsap';
import ScrollTrigger from 'gsap/ScrollTrigger';
import Lenis from 'lenis';

gsap.registerPlugin(ScrollTrigger);

const lenis = new Lenis();

lenis.on('scroll', ScrollTrigger.update);

gsap.ticker.add((time) => {
  lenis.raf(time * 1000);          // GSAP passes seconds; Lenis wants milliseconds
});

gsap.ticker.lagSmoothing(0);
```

Three details that break this if missed:

1. **`time * 1000`** — GSAP's ticker is in seconds, `lenis.raf()` expects milliseconds.
2. **`lagSmoothing(0)`** — GSAP otherwise compensates for lag in a way that desyncs Lenis.
3. **Don't also run your own `requestAnimationFrame` loop** — GSAP's ticker is now the loop.

Cleanup:

```js
function destroy() {
  gsap.ticker.remove(rafCallback);
  lenis.destroy();
  ScrollTrigger.getAll().forEach((t) => t.kill());
  gsap.killTweensOf(target);
}
```

Unkilled ScrollTriggers and tweens holding destroyed objects throw on the next frame — the most
common teardown bug in scroll sites.

## Syncing with a canvas

Give the canvas the smoothed value rather than reading `window.scrollY`:

```js
const lenis = new Lenis();
let progress = 0;

lenis.on('scroll', ({ scroll, limit }) => {
  progress = limit > 0 ? scroll / limit : 0;
});

// One loop drives both — never two competing RAF loops
function frame(time) {
  lenis.raf(time);
  updateScene(progress);
  renderer.render(scene, camera);
  requestAnimationFrame(frame);
}
requestAnimationFrame(frame);
```

With PixiJS, drive Lenis from the Pixi ticker instead of a second loop:

```js
app.ticker.add((ticker) => {
  lenis.raf(ticker.lastTime);
  updateScene(progress);
});
```

See `pixi-scroll` for the Pixi-side patterns, and `three-scroll` for the 3D equivalent.

## API

```js
lenis.scrollTo('#section-3', { offset: -80, duration: 1.5 });
lenis.scrollTo(1200, { immediate: true });
lenis.stop();        // e.g. when a modal opens
lenis.start();
lenis.resize();      // after a layout change if autoResize is off
lenis.destroy();

lenis.on('scroll', ({ scroll, limit, velocity, direction, progress }) => { … });
```

`stop()` when opening a modal, `start()` on close — otherwise the background scrolls behind it.

## Framework wrappers

`lenis` ships React, Vue, and Nuxt bindings (they appear as optional peers):

```jsx
import { ReactLenis } from 'lenis/react';

export default function App() {
  return (
    <ReactLenis root options={{ lerp: 0.1 }}>
      <Page />
    </ReactLenis>
  );
}
```

## Do you need it?

Native CSS covers a lot at zero cost:

```css
html { scroll-behavior: smooth; }        /* smooth anchor jumps only */

@media (prefers-reduced-motion: reduce) {
  html { scroll-behavior: auto; }
}
```

Native **scroll-driven animations** now cover many scroll-linked effects without JS at all:

```css
@keyframes reveal {
  from { opacity: 0; transform: translateY(2rem); }
  to   { opacity: 1; transform: none; }
}

.card {
  animation: reveal linear both;
  animation-timeline: view();
  animation-range: entry 0% cover 40%;
}
```

These run off the main thread. Reach for Lenis when you need a *smoothed, shared* scroll value for
canvas/WebGL sync — not merely for reveal-on-scroll.

## Related skills

| Need | Skill |
|---|---|
| Durations, easing, orchestration | `motion-craft` |
| PixiJS scroll-driven canvas | `pixi-scroll` |
| Scroll-driven 3D | `three-scroll` |
| Reduced motion as a requirement | `a11y-core` |
