---
name: pixi-v8-core
description: Core PixiJS v8 setup and scene graph — creating an Application, the async init requirement, Containers, sprites, the ticker, and the v7→v8 breaking changes. Use for any PixiJS, Pixi, WebGL/WebGPU 2D canvas, HTML5 game, or animated-canvas-website work.
---

# PixiJS v8 Core

PixiJS v8 broke a large part of the v7 API. Most PixiJS code online is v7 and **will not run**
on v8. Before writing any Pixi code, check your output against these five rules.

## The five mistakes to never make

```js
// ✗ v7 — every one of these fails on v8
const app = new Application({ width: 800 });  // sync constructor
document.body.appendChild(app.view);          // .view
g.beginFill(0xff0000).drawRect(0, 0, 50, 50); // style-then-shape
const tex = Texture.from('bunny.png');        // does not load the file
const base = new BaseTexture(img);            // class removed
```

```js
// ✓ v8
const app = new Application();
await app.init({ width: 800 });               // async init, always awaited
document.body.appendChild(app.canvas);        // .canvas
g.rect(0, 0, 50, 50).fill(0xff0000);          // shape-then-style
const tex = await Assets.load('bunny.png');   // preload via Assets
// BaseTexture → TextureSource subtypes (ImageSource, CanvasSource, …)
```

See [reference.md](reference.md) for the complete v7→v8 change table.

## Minimal application

```js
import { Application, Assets, Sprite } from 'pixi.js';

const app = new Application();

await app.init({
  background: '#1099bb',
  resizeTo: window,          // or width/height for a fixed stage
  antialias: true,
  preference: 'webgl',       // 'webgpu' to opt in; 'webgl' is the default
});

document.body.appendChild(app.canvas);

const texture = await Assets.load('bunny.png');
const bunny = new Sprite(texture);

bunny.anchor.set(0.5);
bunny.position.set(app.screen.width / 2, app.screen.height / 2);
app.stage.addChild(bunny);

app.ticker.add((ticker) => {
  bunny.rotation += 0.1 * ticker.deltaTime;
});
```

Because `init` is async, top-level Pixi setup must live in an `async` function or a module with
top-level `await`. A sync `main()` that forgets to await `init()` produces confusing "renderer is
null" errors rather than a clear failure.

Renderer options can be split per backend when the two need different settings:

```js
await app.init({
  webgl: { antialias: true },
  webgpu: { antialias: false },
});
```

## Scene graph

`Container` is the base class for everything on screen. `DisplayObject` **no longer exists** — if
you are extending or type-annotating `DisplayObject`, replace it with `Container`.

```js
import { Container } from 'pixi.js';

const world = new Container();
world.label = 'world';        // v7 called this `.name`

world.addChild(playerSprite, enemySprite);
world.removeChild(enemySprite);

app.stage.addChild(world);
```

Children render in array order — later children draw on top. Transforms are inherited, so moving
`world` moves everything inside it.

Common container/sprite properties:

| Property | Notes |
|---|---|
| `position` / `x` / `y` | Position relative to parent |
| `scale`, `rotation` | `rotation` is radians; `angle` is degrees |
| `pivot` | Origin for rotation and scale |
| `anchor` | **Sprites only.** `0.5` centres the texture on its position |
| `alpha`, `visible` | `visible = false` skips rendering entirely |
| `renderable` | Skips drawing but keeps transforms/bounds live |
| `label` | Was `.name` in v7 |
| `zIndex` | Requires `sortableChildren = true` on the parent |

## The ticker

The ticker is Pixi's render loop. The callback receives a `Ticker`, not a raw number.

```js
app.ticker.add((ticker) => {
  // deltaTime ≈ 1.0 at 60fps, 2.0 at 30fps — scales motion to framerate
  sprite.x += 2 * ticker.deltaTime;

  // deltaMS is elapsed milliseconds; use for timers and physics
  elapsed += ticker.deltaMS;
});
```

**Always multiply per-frame motion by `deltaTime`.** Without it, animation runs at double speed on
a 120Hz display and half speed on a struggling one.

Remove callbacks when tearing down, or they keep running and leak:

```js
const onTick = (ticker) => { /* … */ };
app.ticker.add(onTick);
// later
app.ticker.remove(onTick);
```

For fixed-timestep game logic, see the `pixi-game-loop` skill.

## Teardown

Pixi allocates GPU resources that garbage collection cannot reclaim on its own. When destroying an
app — a route change, an unmounting component — release them:

```js
app.destroy(true, { children: true, texture: true });
```

The first argument removes the canvas from the DOM. Skipping this in single-page apps is the most
common source of Pixi memory leaks; see `pixi-react` for the component-lifecycle version.

## Where to go next

| Task | Skill |
|---|---|
| Drawing shapes, lines, paths | `pixi-graphics` |
| Loading textures, bundles, fonts | `pixi-assets` |
| Spritesheets, frame animation, tweens | `pixi-animation` |
| Clicks, taps, hover, drag | `pixi-interaction` |
| Rendering text | `pixi-text` |
| Frame rate, batching, culling | `pixi-performance` |
| Shaders and visual effects | `pixi-filters` |
| Upgrading an existing v7 codebase | `pixi-migration` |
