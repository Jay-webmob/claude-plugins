---
name: three-scroll
description: Scroll-driven 3D experiences — the two architectures (canvas-owns-scroll vs DOM-owns-scroll), mapping scroll progress to camera and object state, and smoothing. Use when building a scrollytelling 3D site, a scroll-linked product reveal, or syncing a three.js scene to page scroll.
---

# Scroll-Driven 3D

The universal shape:

```
scroll position → normalize to progress p ∈ [0,1] → smooth p → map to camera/object state
                → apply by MUTATION inside useFrame (never setState)
```

There are **two architectures, and the top failure mode is mixing them.** Pick one per page.

## A: Canvas owns the scroll

drei's `<ScrollControls>` creates its own scroll container. HTML content goes inside
`<Scroll html>`.

```jsx
import { Canvas } from '@react-three/fiber';
import { ScrollControls, Scroll, useScroll } from '@react-three/drei';
import { useRef } from 'react';
import { useFrame } from '@react-three/fiber';

function Model() {
  const ref = useRef();
  const scroll = useScroll();

  useFrame(() => {
    ref.current.rotation.y = scroll.offset * Math.PI * 2;
    ref.current.position.y = scroll.range(0, 1 / 3) * -2;
  });

  return <mesh ref={ref}>…</mesh>;
}

<Canvas>
  <ScrollControls pages={3} damping={0.2}>
    <Model />
    <Scroll html>
      <section style={{ height: '100vh' }}>Intro</section>
      <section style={{ height: '100vh' }}>Detail</section>
    </Scroll>
  </ScrollControls>
</Canvas>
```

`<ScrollControls>` prop defaults: `pages={1}`, `distance={1}`, `damping={0.2}`, `eps={0.00001}`,
`horizontal={false}`, `infinite={false}`, `maxSpeed={Infinity}`, `prepend={false}`.

`useScroll()` returns:

| Member | Meaning |
|---|---|
| `offset` | 0–1 across the whole scroll, **already damped** |
| `delta` | 0–1, damped change since last frame |
| `range(start, distance)` | 0→1 across that slice |
| `curve(start, distance)` | **0→1→0** across that slice — for fade in/out |
| `visible(start, distance)` | Boolean |

`curve()` is the one to reach for when something should appear and then leave.

**Best for:** self-contained 3D experiences where the canvas is the page. **Not** for pages with
substantial ordinary DOM content, real SEO needs, or an existing scroll setup.

## B: DOM owns the scroll

The page scrolls normally; the canvas is fixed behind it and reads scroll progress.

```jsx
function Scene({ progress }) {
  const ref = useRef();

  useFrame((state, delta) => {
    // Damp toward the target inside the frame loop — never setState per frame
    const target = progress.current;
    ref.current.rotation.y += (target * Math.PI * 2 - ref.current.rotation.y) * (1 - Math.exp(-4 * delta));
  });

  return <mesh ref={ref}>…</mesh>;
}

export function Page() {
  const progress = useRef(0);

  useEffect(() => {
    const onScroll = () => {
      const max = document.body.scrollHeight - window.innerHeight;
      progress.current = max > 0 ? window.scrollY / max : 0;
    };
    window.addEventListener('scroll', onScroll, { passive: true });
    onScroll();
    return () => window.removeEventListener('scroll', onScroll);
  }, []);

  return (
    <>
      <div style={{ position: 'fixed', inset: 0, zIndex: 0 }}>
        <Canvas><Scene progress={progress} /></Canvas>
      </div>
      <main style={{ position: 'relative', zIndex: 1 }}>
        <section style={{ minHeight: '100vh' }}>…</section>
      </main>
    </>
  );
}
```

The critical detail: **scroll writes to a ref, the frame loop reads it.** Writing to state in the
scroll handler re-renders React at scroll frequency and destroys performance.

`{ passive: true }` keeps scrolling off the main thread's critical path.

**Best for:** marketing pages, scrollytelling with real content, anything needing normal document
flow and SEO.

## Never mix them

```jsx
// ✗ Two competing scroll sources — jitter, drift, or a frozen scene
<ScrollControls pages={3}>
  <SceneReadingWindowScrollY />
</ScrollControls>
```

If `<ScrollControls>` is present, `window.scrollY` is not the truth. If the DOM owns scroll, don't
mount `<ScrollControls>`.

## Smoothing

Always frame-rate independent:

```js
// ✗ Speed depends on refresh rate
value += (target - value) * 0.1;

// ✓ Same feel at 60Hz and 144Hz
value += (target - value) * (1 - Math.exp(-k * delta));   // k ≈ 3–6
```

With Lenis (see `motion-smooth-scroll`), take the smoothed value from Lenis and skip your own
damping — two smoothing layers feel laggy:

```js
lenis.on('scroll', ({ scroll, limit }) => {
  progress.current = limit > 0 ? scroll / limit : 0;
});
```

Drive one loop, not two:

```js
function frame(time) {
  lenis.raf(time);
  requestAnimationFrame(frame);
}
```

R3F runs its own loop; let Lenis have the external one and let `useFrame` read the ref.

## Camera paths

For a camera following a path, a curve beats hand-keyed positions:

```js
import * as THREE from 'three';

const path = new THREE.CatmullRomCurve3([
  new THREE.Vector3(0, 0, 10),
  new THREE.Vector3(5, 2, 5),
  new THREE.Vector3(0, 4, 0),
]);

useFrame(() => {
  const p = THREE.MathUtils.clamp(progress.current, 0, 1);
  path.getPointAt(p, state.camera.position);
  state.camera.lookAt(0, 0, 0);
});
```

`getPointAt` uses arc-length parameterisation, so motion is evenly paced — `getPoint` is not and
will speed up and slow down unevenly.

## Sectioned progress

Map global progress to per-section ranges:

```js
function sectionProgress(global, start, end) {
  return THREE.MathUtils.clamp((global - start) / (end - start), 0, 1);
}

const intro = sectionProgress(p, 0.0, 0.33);
const detail = sectionProgress(p, 0.33, 0.66);
```

With `<ScrollControls>`, `scroll.range()` and `scroll.curve()` do this for you.

## Performance and accessibility

- Pause rendering when the canvas is off screen — a fixed canvas keeps rendering while the user
  reads text over it:

```jsx
<Canvas frameloop={inView ? 'always' : 'never'}>
```

- Cap DPR (`dpr={[1, 2]}`) and consider `<AdaptiveDpr>` under load.
- Preload models before the scroll starts — a model popping in mid-scroll is worse than a wait.
- **Respect `prefers-reduced-motion`.** Scroll-driven 3D is exactly the vestibular trigger the
  setting exists for:

```jsx
const reduce = matchMedia('(prefers-reduced-motion: reduce)').matches;

<Canvas frameloop={reduce ? 'demand' : 'always'}>
```

Under reduced motion, render a static composed frame instead of the animation, and make sure the
page's content still makes sense without the 3D.

- Keep meaningful content in the DOM, not only in the scene — see `a11y-canvas`.

## Related skills

| Need | Skill |
|---|---|
| R3F setup, useFrame, drei | `three-r3f` |
| Model/texture optimization | `three-assets` |
| Lenis and GSAP wiring | `motion-smooth-scroll` |
| 2D scroll-driven canvas | `pixi-scroll` |
| Making the experience accessible | `a11y-canvas` |
