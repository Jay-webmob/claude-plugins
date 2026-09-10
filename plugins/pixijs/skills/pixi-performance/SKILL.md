---
name: pixi-performance
description: Optimizing PixiJS v8 performance — draw call batching, texture atlases, culling, render groups, cacheAsTexture, ParticleContainer, and profiling. Use when a Pixi canvas or game drops frames, stutters, or needs to render many thousands of objects.
---

# PixiJS v8 Performance

Diagnose before optimizing. Most Pixi slowdowns are one of: too many draw calls, per-frame CPU
work that should be cached, or unbounded object growth.

## Find the bottleneck first

Is it CPU or GPU? Halve the canvas resolution:

```js
app.renderer.resolution = 0.5;
```

If the frame rate recovers, you are **GPU-bound** (fill rate/overdraw). If nothing changes, you
are **CPU-bound** (JS logic, transforms, tessellation). The two demand opposite fixes — culling
helps the GPU-bound case and *hurts* the CPU-bound one.

Check the draw call count in Chrome's WebGL/WebGPU inspector or via a profiler. As a rough guide,
a scene doing thousands of draw calls per frame has a batching problem.

## Batching

Pixi automatically batches sprites into a single draw call, up to about **16 textures per batch**
(hardware-dependent). Breaking a batch means a new draw call.

What breaks batching:

1. **Blend mode changes.** A single `blendMode: 'add'` sprite between two normal ones splits one
   batch into three.
2. **Filters.** Any filtered object renders to its own texture — an unavoidable break.
3. **Masks.** Same cost as filters.
4. **Exceeding the texture limit** within one batch.
5. **Interleaving object types.** `sprite, graphic, sprite, graphic` produces four draw calls;
   `sprite, sprite, graphic, graphic` produces two.

So: group like with like, and group by blend mode.

```js
// ✗ Alternating types and blend modes — a draw call each
stage.addChild(sprite1, glowSprite, graphic1, sprite2, glowSprite2, graphic2);

// ✓ Grouped into layers — far fewer draw calls
const normalLayer = new Container();
const glowLayer = new Container();     // everything here uses blendMode 'add'
const shapeLayer = new Container();
stage.addChild(normalLayer, glowLayer, shapeLayer);
```

## Texture atlases

Sprites sharing one atlas batch together; sprites from separate PNGs may not.

```js
const sheet = await Assets.load('sprites/game.json');
const hero = new Sprite(sheet.textures['hero.png']);
const coin = new Sprite(sheet.textures['coin.png']);   // same atlas → same batch
```

Pack atlases with TexturePacker, free-tex-packer, or AssetPack. Keep each atlas at or below
2048×2048 for broad mobile support.

## Culling

**Off by default.** Enabling it skips off-screen objects — a win when GPU-bound, a loss when
CPU-bound, since the bounds checks themselves cost CPU.

```js
sprite.cullable = true;
sprite.cullArea = new Rectangle(0, 0, 64, 64);   // skips bounds calculation — faster
```

For a large scrolling world, cull at the container level rather than per sprite, and prefer
`cullArea` where you know the size up front. Do not enable culling on a scene that already fits
on screen — you pay the check and gain nothing.

## Render groups

`isRenderGroup: true` makes a container a self-contained rendering unit whose position, scale,
rotation, tint, and alpha are **offloaded to the GPU**.

```js
const world = new Container({ isRenderGroup: true });
const hud = new Container({ isRenderGroup: true });
stage.addChild(world, hud);
```

Best for static hierarchies and for separating logically distinct sections (world vs UI). The
structure inside can be static while individual properties still animate.

**Too many render groups degrades performance.** Most projects need none. Profile before adding
them, and add a handful, not dozens.

## cacheAsTexture

Renders a container once to a texture and reuses it. Replaces v7's `cacheAsBitmap`.

```js
staticBackground.cacheAsTexture(true);
```

Good for complex static content — a tile map, an elaborate vector illustration, a finished UI
panel. It costs texture memory and must be invalidated when contents change:

```js
panel.cacheAsTexture(false);   // rebuild after modifying children
panel.cacheAsTexture(true);
```

Never cache something that changes every frame — you would re-render to texture each time, which
is strictly worse than drawing it directly.

## Graphics

Tessellation happens on the CPU when geometry is built.

```js
// ✗ Re-tessellates every frame
app.ticker.add(() => {
  g.clear().circle(x, y, r).fill(0xff0000);
});

// ✓ Build once, transform after
g.circle(0, 0, r).fill(0xff0000);
app.ticker.add(() => { g.position.set(x, y); });
```

Share geometry across many identical shapes with `GraphicsContext` (see `pixi-graphics`). Shapes
of ~100 points or fewer perform about the same as sprites; complex ones do not.

## Text

`Text` re-renders a canvas and re-uploads a texture on every string change. Anything updating more
than a few times a second should be `BitmapText`. See `pixi-text`.

## ParticleContainer

For tens of thousands of simple, identical objects. In v8 it **no longer accepts sprites**:

```js
import { ParticleContainer, Particle } from 'pixi.js';

const particles = new ParticleContainer({
  dynamicProperties: {
    position: true,      // updated per frame
    rotation: false,     // static — cheaper
    color: false,
  },
  boundsArea: new Rectangle(0, 0, app.screen.width, app.screen.height),
});

for (let i = 0; i < 20000; i++) {
  particles.addParticle(new Particle({ texture, x: Math.random() * 800, y: Math.random() * 600 }));
}
```

Particles live in `particleChildren`, outside the scene graph — they cannot have children, filters,
or events. Mark only the properties you actually animate as dynamic, and supply `boundsArea`
yourself, since bounds are not computed automatically.

## Interaction cost

Hit-testing walks the scene graph on every pointer move.

```js
decorLayer.eventMode = 'none';                // whole subtree skipped
particleLayer.interactiveChildren = false;    // children skipped
button.hitArea = new Rectangle(0, 0, 200, 60); // avoids bounds recalculation
```

Prefer `eventMode: 'static'` over `'dynamic'` unless the object moves under a stationary pointer.

## Object pooling

Allocating and destroying objects each frame causes GC pauses that show up as stutter. Reuse:

```js
class BulletPool {
  constructor(texture, size = 200) {
    this.pool = Array.from({ length: size }, () => {
      const s = new Sprite(texture);
      s.visible = false;
      return s;
    });
  }

  acquire() {
    const bullet = this.pool.pop() ?? new Sprite(this.texture);
    bullet.visible = true;
    return bullet;
  }

  release(bullet) {
    bullet.visible = false;
    this.pool.push(bullet);
  }
}
```

## Resolution on mobile

Full retina resolution means 4× the pixels to shade — often the single biggest mobile win:

```js
await app.init({
  resolution: Math.min(window.devicePixelRatio, 2),   // cap at 2
  autoDensity: true,
});
```

## Checklist

| Symptom | Likely cause | Fix |
|---|---|---|
| High draw calls | Mixed types/blend modes, many atlases | Group by type and blend mode; combine atlases |
| CPU-bound, many objects | Per-frame Graphics rebuilds, per-frame `Text` | Cache geometry; use `BitmapText` |
| GPU-bound | Overdraw, high resolution, large filters | Cap resolution, cull, shrink filter areas |
| Periodic stutter | GC from per-frame allocation | Pool objects |
| Slow pointer response | Deep interactive tree | `eventMode: 'none'`, `interactiveChildren = false`, `hitArea` |
| Memory climbs steadily | Undestroyed textures/Text/ticker callbacks | `destroy()`, `Assets.unload`, `ticker.remove` |
