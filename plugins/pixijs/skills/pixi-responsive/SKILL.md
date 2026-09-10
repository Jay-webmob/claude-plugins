---
name: pixi-responsive
description: Responsive and retina-correct PixiJS v8 canvases — resizeTo, devicePixelRatio, letterbox vs fill vs fixed-width scaling, safe areas, and orientation changes. Use when a Pixi canvas must adapt to window size, look crisp on retina, or work across desktop and mobile.
---

# Responsive PixiJS Canvases

Two independent problems: **resolution** (crispness on high-DPI screens) and **layout** (how the
scene adapts to different aspect ratios).

## Resolution

```js
await app.init({
  resolution: window.devicePixelRatio || 1,
  autoDensity: true,          // sets CSS size so the canvas isn't physically larger
});
```

`resolution` multiplies the backing buffer; `autoDensity` keeps the CSS size correct. Setting
`resolution` without `autoDensity` gives a canvas twice its intended size on retina.

Full retina resolution means 4× the pixels to shade. On mobile that is often the largest single
performance cost — cap it:

```js
await app.init({
  resolution: Math.min(window.devicePixelRatio || 1, 2),
  autoDensity: true,
});
```

Handle displays changing (dragging between monitors):

```js
window.addEventListener('resize', () => {
  const dpr = Math.min(window.devicePixelRatio || 1, 2);
  if (app.renderer.resolution !== dpr) {
    app.renderer.resolution = dpr;
    app.renderer.resize(window.innerWidth, window.innerHeight);
  }
});
```

## Auto-resizing

```js
await app.init({ resizeTo: window });          // or a container element
```

`resizeTo` handles the canvas dimensions but **not your layout** — repositioning content is on
you. Pixi emits a resize you can hook:

```js
app.renderer.on('resize', (width, height) => {
  layout(width, height);
});

function layout(w, h) {
  title.position.set(w / 2, 60);
  playButton.position.set(w / 2, h - 120);
  background.width = w;
  background.height = h;
}

layout(app.screen.width, app.screen.height);   // run once at startup
```

When resizing to an element rather than the window, that element needs real dimensions — a `<div>`
with no height collapses to zero and renders nothing.

## Three scaling strategies

**Fluid** — the scene fills the viewport and layout reflows. Best for UI-heavy and web work.

```js
function layout(w, h) {
  header.position.set(w / 2, 40);
  grid.width = w - 80;
}
```

**Letterbox** — a fixed design resolution scaled uniformly, with bars on the extra space.
Guarantees identical composition everywhere. Best for games.

```js
const DESIGN_W = 1920;
const DESIGN_H = 1080;

const stage = new Container();
app.stage.addChild(stage);

function layout(w, h) {
  const scale = Math.min(w / DESIGN_W, h / DESIGN_H);
  stage.scale.set(scale);
  stage.position.set((w - DESIGN_W * scale) / 2, (h - DESIGN_H * scale) / 2);
}
```

**Fill/crop** — scale by the larger factor so no bars appear, accepting that edges get cut.

```js
function layout(w, h) {
  const scale = Math.max(w / DESIGN_W, h / DESIGN_H);
  stage.scale.set(scale);
  stage.position.set((w - DESIGN_W * scale) / 2, (h - DESIGN_H * scale) / 2);
}
```

Use fill for backgrounds, and keep anything important inside a "safe" central area — the same
principle as video title-safe zones.

## Hybrid: fixed world, fluid UI

Usually the best of both — the game scales uniformly, the HUD anchors to real screen edges.

```js
const world = new Container();      // letterboxed, fixed design size
const hud = new Container();        // unscaled, anchored to actual edges
app.stage.addChild(world, hud);

function layout(w, h) {
  const scale = Math.min(w / DESIGN_W, h / DESIGN_H);
  world.scale.set(scale);
  world.position.set((w - DESIGN_W * scale) / 2, (h - DESIGN_H * scale) / 2);

  scoreLabel.position.set(20, 20);
  pauseButton.position.set(w - 60, 20);
}
```

HUD text stays crisp at native resolution instead of being scaled up and blurred.

## Anchoring helper

```js
const ANCHOR = {
  topLeft:     (w, h) => ({ x: 0,     y: 0 }),
  topRight:    (w, h) => ({ x: w,     y: 0 }),
  center:      (w, h) => ({ x: w / 2, y: h / 2 }),
  bottomLeft:  (w, h) => ({ x: 0,     y: h }),
  bottomRight: (w, h) => ({ x: w,     y: h }),
};

function anchorTo(obj, where, offset = { x: 0, y: 0 }) {
  obj._anchorSpec = { where, offset };
}

function applyAnchors(root, w, h) {
  for (const child of root.children) {
    const spec = child._anchorSpec;
    if (spec) {
      const p = ANCHOR[spec.where](w, h);
      child.position.set(p.x + spec.offset.x, p.y + spec.offset.y);
    }
    if (child.children?.length) applyAnchors(child, w, h);
  }
}
```

## Mobile safe areas

Notches and home indicators overlap the canvas. Read the CSS environment variables:

```css
:root {
  --safe-top: env(safe-area-inset-top, 0px);
  --safe-bottom: env(safe-area-inset-bottom, 0px);
}
```

```js
function safeInsets() {
  const s = getComputedStyle(document.documentElement);
  return {
    top: parseFloat(s.getPropertyValue('--safe-top')) || 0,
    bottom: parseFloat(s.getPropertyValue('--safe-bottom')) || 0,
  };
}

function layout(w, h) {
  const safe = safeInsets();
  pauseButton.position.set(w - 60, safe.top + 20);
  joystick.position.set(100, h - safe.bottom - 100);
}
```

Requires `viewport-fit=cover` in the viewport meta tag:

```html
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover" />
```

## Orientation

```js
window.addEventListener('orientationchange', () => {
  // iOS reports stale dimensions immediately after the event
  setTimeout(() => {
    app.renderer.resize(window.innerWidth, window.innerHeight);
    layout(app.screen.width, app.screen.height);
  }, 100);
});
```

To require landscape, prompt with a DOM overlay rather than trying to rotate the canvas:

```js
const isPortrait = window.matchMedia('(orientation: portrait)').matches;
rotatePrompt.style.display = isPortrait ? 'flex' : 'none';
```

## Debouncing

Desktop resize fires continuously while dragging; `app.renderer.resize` reallocates buffers.

```js
let resizeTimer;
window.addEventListener('resize', () => {
  clearTimeout(resizeTimer);
  resizeTimer = setTimeout(() => layout(app.screen.width, app.screen.height), 100);
});
```

Use `ResizeObserver` when sizing to an element rather than the window:

```js
const ro = new ResizeObserver(([entry]) => {
  const { width, height } = entry.contentRect;
  app.renderer.resize(width, height);
  layout(width, height);
});
ro.observe(hostElement);

// Cleanup
ro.disconnect();
```

## Required CSS

```css
html, body { margin: 0; height: 100%; overflow: hidden; }
canvas { display: block; touch-action: none; }
```

`display: block` removes the inline-element gap below the canvas; `touch-action: none` stops
browser pan/zoom gestures from stealing input.

## Checklist

| Symptom | Cause | Fix |
|---|---|---|
| Blurry on retina | Resolution not set | `resolution: devicePixelRatio, autoDensity: true` |
| Canvas twice its size | `resolution` without `autoDensity` | Add `autoDensity: true` |
| Blank canvas | Host element has no height | Give the host explicit dimensions |
| Slow on phones | Full retina resolution | Cap at 2 |
| UI off-screen on mobile | No safe-area handling | `env(safe-area-inset-*)` + `viewport-fit=cover` |
| Layout wrong after rotate | iOS stale dimensions | Resize after a ~100ms delay |
| Stutter while resizing | Reallocating per event | Debounce |
| Small gap under canvas | Inline display | `canvas { display: block; }` |
