---
name: pixi-camera
description: Camera and viewport systems for PixiJS v8 — following a player, world-to-screen coordinates, zoom, clamping to world bounds, parallax layers, and screen shake. Use when building a scrolling game world, panning or zooming a Pixi canvas, or adding parallax depth.
---

# Camera and Viewport in PixiJS

Pixi has no camera object. A camera is just a `Container` you move in the **opposite** direction
of the point you want centred.

```js
const world = new Container();
app.stage.addChild(world);

// Centre the camera on (targetX, targetY)
world.x = app.screen.width / 2 - targetX * world.scale.x;
world.y = app.screen.height / 2 - targetY * world.scale.y;
```

Everything scrolling lives inside `world`; UI stays outside it so it doesn't move.

```js
const world = new Container();   // scrolls
const hud = new Container();     // fixed
app.stage.addChild(world, hud);
```

## A camera class

```js
class Camera {
  constructor(world, screen) {
    this.world = world;
    this.screen = screen;
    this.x = 0;
    this.y = 0;
    this.zoom = 1;
    this.bounds = null;          // { x, y, width, height } of the world
  }

  follow(target, dt, smoothing = 5) {
    // Frame-rate-independent smoothing — never a bare 0.1 lerp factor
    const t = 1 - Math.exp(-smoothing * dt);
    this.x += (target.x - this.x) * t;
    this.y += (target.y - this.y) * t;
  }

  clamp() {
    if (!this.bounds) return;
    const halfW = this.screen.width / (2 * this.zoom);
    const halfH = this.screen.height / (2 * this.zoom);

    // If the world is narrower than the view, centre it instead of clamping
    if (this.bounds.width < halfW * 2) {
      this.x = this.bounds.x + this.bounds.width / 2;
    } else {
      this.x = Math.max(this.bounds.x + halfW,
               Math.min(this.bounds.x + this.bounds.width - halfW, this.x));
    }

    if (this.bounds.height < halfH * 2) {
      this.y = this.bounds.y + this.bounds.height / 2;
    } else {
      this.y = Math.max(this.bounds.y + halfH,
               Math.min(this.bounds.y + this.bounds.height - halfH, this.y));
    }
  }

  apply() {
    this.world.scale.set(this.zoom);
    this.world.x = this.screen.width / 2 - this.x * this.zoom;
    this.world.y = this.screen.height / 2 - this.y * this.zoom;
  }

  update(target, dt) {
    this.follow(target, dt);
    this.clamp();
    this.apply();
  }
}
```

```js
const camera = new Camera(world, app.screen);
camera.bounds = { x: 0, y: 0, width: 3200, height: 1600 };

app.ticker.add((ticker) => {
  camera.update(player, ticker.deltaMS / 1000);
});
```

The `1 - Math.exp(-smoothing * dt)` form matters: a plain `+= diff * 0.1` moves the camera at a
speed that depends on frame rate, so the game feels different on a 144Hz monitor.

## Deadzone

Following exactly is nauseating for small movements. Only move once the target leaves a central
box:

```js
follow(target, dt, deadzone = { w: 120, h: 80 }) {
  const dx = target.x - this.x;
  const dy = target.y - this.y;

  if (Math.abs(dx) > deadzone.w / 2) {
    this.x += dx - Math.sign(dx) * (deadzone.w / 2);
  }
  if (Math.abs(dy) > deadzone.h / 2) {
    this.y += dy - Math.sign(dy) * (deadzone.h / 2);
  }
}
```

A wider horizontal than vertical deadzone suits platformers, where jumping shouldn't scroll.

## Coordinate conversion

Pixi provides this on any container — no manual math needed:

```js
const worldPos = world.toLocal(event.global);            // screen → world
const screenPos = world.toGlobal(sprite.position);       // world → screen
```

For pointer events specifically:

```js
app.stage.eventMode = 'static';
app.stage.hitArea = app.screen;

app.stage.on('pointerdown', (event) => {
  const worldPoint = world.toLocal(event.global);
  spawnAt(worldPoint.x, worldPoint.y);
});
```

## Zoom

Naïve zoom drifts because it zooms toward the world origin. To zoom toward a specific screen
point — the cursor, or the screen centre — preserve that point's world position:

```js
zoomAt(screenX, screenY, factor) {
  const before = this.world.toLocal({ x: screenX, y: screenY });

  this.zoom = Math.max(0.25, Math.min(4, this.zoom * factor));
  this.apply();

  const after = this.world.toLocal({ x: screenX, y: screenY });
  this.x -= after.x - before.x;
  this.y -= after.y - before.y;
  this.apply();
}
```

```js
app.canvas.addEventListener('wheel', (e) => {
  e.preventDefault();
  camera.zoomAt(e.offsetX, e.offsetY, e.deltaY > 0 ? 0.9 : 1.1);
}, { passive: false });
```

Note `{ passive: false }` — without it, `preventDefault()` is ignored and the page scrolls.

## Parallax

Background layers move at a fraction of camera speed; foreground layers move faster.

```js
const layers = [
  { container: new Container(), factor: 0.2 },   // distant sky
  { container: new Container(), factor: 0.5 },   // hills
  { container: new Container(), factor: 1.0 },   // gameplay layer
  { container: new Container(), factor: 1.3 },   // foreground foliage
];

layers.forEach((l) => app.stage.addChild(l.container));

function applyParallax(camera) {
  for (const { container, factor } of layers) {
    container.x = app.screen.width / 2 - camera.x * factor * camera.zoom;
    container.y = app.screen.height / 2 - camera.y * factor * camera.zoom;
    container.scale.set(camera.zoom);
  }
}
```

For an endlessly repeating background, use `TilingSprite` and scroll its `tilePosition` — no
layer duplication needed:

```js
import { TilingSprite } from 'pixi.js';

const sky = new TilingSprite({ texture: skyTexture, width: app.screen.width, height: app.screen.height });

app.ticker.add(() => {
  sky.tilePosition.x = -camera.x * 0.2;
});
```

## Screen shake

Apply as an offset after `apply()`, so it never corrupts the camera's real position:

```js
class Camera {
  shake(intensity, duration) {
    this.shakeIntensity = intensity;
    this.shakeRemaining = duration;
  }

  applyShake(dt) {
    if (this.shakeRemaining <= 0) return;
    this.shakeRemaining -= dt;

    const decay = Math.max(this.shakeRemaining / this.shakeDuration, 0);
    const amount = this.shakeIntensity * decay;

    this.world.x += (Math.random() * 2 - 1) * amount;
    this.world.y += (Math.random() * 2 - 1) * amount;
  }
}
```

Decaying the intensity is what makes it feel like an impact rather than a rattle.

## Culling a large world

Only draw what the camera can see. Compute the visible rect in world space and skip the rest:

```js
function visibleRect(camera, screen) {
  const halfW = screen.width / (2 * camera.zoom);
  const halfH = screen.height / (2 * camera.zoom);
  return {
    x: camera.x - halfW, y: camera.y - halfH,
    w: halfW * 2,        h: halfH * 2,
  };
}

app.ticker.add(() => {
  const view = visibleRect(camera, app.screen);
  for (const chunk of chunks) {
    chunk.visible = aabb(chunk.bounds, view);
  }
});
```

Toggling `visible` on a few large chunks is much cheaper than Pixi's per-sprite `cullable`. See
`pixi-performance` for when built-in culling helps and when it hurts.

## Pixel-art snapping

Sub-pixel camera positions make pixel art shimmer. Round the final transform:

```js
apply() {
  this.world.scale.set(this.zoom);
  this.world.x = Math.round(this.screen.width / 2 - this.x * this.zoom);
  this.world.y = Math.round(this.screen.height / 2 - this.y * this.zoom);
}
```

Also set `texture.source.scaleMode = 'nearest'` on pixel-art textures to keep edges hard.
