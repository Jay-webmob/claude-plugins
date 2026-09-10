---
name: pixi-graphics
description: Drawing vector shapes in PixiJS v8 with the Graphics API — rects, circles, paths, strokes, fills, gradients, holes, and GraphicsContext reuse. Use when drawing shapes, lines, paths, or procedural vector art on a Pixi canvas.
---

# PixiJS v8 Graphics

The Graphics API is the single most-changed part of v8. The workflow **reversed**.

## Shape first, then style

```js
// ✗ v7 — style first, then draw, then endFill
g.beginFill(0xff0000);
g.lineStyle(2, 0x000000);
g.drawRect(0, 0, 100, 50);
g.endFill();
```

```js
// ✓ v8 — define the shape, then fill/stroke it
g.rect(0, 0, 100, 50)
 .fill({ color: 0xff0000 })
 .stroke({ width: 2, color: 0x000000 });
```

Every `draw*` lost its prefix, "line" became "stroke", and `endFill()` no longer exists.

| v7 | v8 |
|---|---|
| `drawRect` | `rect` |
| `drawCircle` | `circle` |
| `drawEllipse` | `ellipse` |
| `drawRoundedRect` | `roundRect` |
| `drawPolygon` | `poly` |
| `drawStar` | `star` |
| `beginFill` / `endFill` | `fill()` after the shape |
| `lineStyle` | `stroke()` after the shape |
| `beginHole` / `endHole` | `cut()` |

## Basic shapes

```js
import { Graphics } from 'pixi.js';

const g = new Graphics();

g.rect(0, 0, 100, 60).fill(0x3498db);
g.circle(200, 30, 30).fill({ color: 0xe74c3c });
g.roundRect(250, 0, 100, 60, 12).fill(0x2ecc71).stroke({ width: 3, color: 0xffffff });
g.ellipse(420, 30, 50, 25).fill(0xf1c40f);
g.star(520, 30, 5, 30).fill(0x9b59b6);

app.stage.addChild(g);
```

`fill()` accepts a bare colour or an options object. Colours accept hex numbers (`0xff0000`), CSS
strings (`'red'`, `'#ff0000'`), and `rgba()` strings.

Each shape call is styled by the `fill`/`stroke` that follows it, so one `Graphics` can hold many
independently-styled shapes — that is cheaper than many `Graphics` objects.

## Paths and lines

```js
const line = new Graphics();

line.moveTo(0, 0)
    .lineTo(100, 50)
    .lineTo(200, 0)
    .stroke({ width: 4, color: 0xffffff, cap: 'round', join: 'round' });
```

Stroke options: `width`, `color`, `alpha`, `alignment` (0 = outside, 0.5 = centred, 1 = inside),
`cap` (`'butt' | 'round' | 'square'`), `join` (`'miter' | 'round' | 'bevel'`), `miterLimit`.

Curves:

```js
g.moveTo(0, 100)
 .bezierCurveTo(50, 0, 150, 200, 200, 100)
 .stroke({ width: 3, color: 0x00ff00 });

g.moveTo(0, 0).quadraticCurveTo(50, 100, 100, 0).stroke({ width: 2, color: 0xff00ff });
g.moveTo(0, 0).arcTo(100, 0, 100, 100, 20).stroke({ width: 2, color: 0x00ffff });
```

Close a shape with `.closePath()` before filling to guarantee a clean edge.

## Holes with cut()

```js
const donut = new Graphics();

donut.circle(100, 100, 80)
     .circle(100, 100, 40)
     .cut()                       // second shape becomes a hole in the first
     .fill(0xffcc00);
```

## Gradients

```js
import { FillGradient } from 'pixi.js';

const gradient = new FillGradient(0, 0, 0, 200);
gradient.addColorStop(0, 0xff0000);
gradient.addColorStop(1, 0x0000ff);

g.rect(0, 0, 200, 200).fill(gradient);
```

`FillGradient(x0, y0, x1, y1)` defines a linear gradient in local coordinates. Gradients work on
`stroke()` too.

## Reusing geometry with GraphicsContext

`GraphicsGeometry` is gone; `GraphicsContext` replaces it. Building one context and sharing it
across many `Graphics` instances tessellates the shape **once** — the correct approach when
drawing hundreds of identical shapes.

```js
import { Graphics, GraphicsContext } from 'pixi.js';

const shieldContext = new GraphicsContext()
  .roundRect(0, 0, 40, 48, 6)
  .fill(0x336699)
  .stroke({ width: 2, color: 0xffffff });

for (let i = 0; i < 500; i++) {
  const shield = new Graphics(shieldContext);   // shares tessellated geometry
  shield.position.set(Math.random() * 800, Math.random() * 600);
  app.stage.addChild(shield);
}
```

Destroy a shared context yourself when done — `shieldContext.destroy()`. Destroying one `Graphics`
that borrowed it must not destroy the context, so pass `destroy({ context: false })` in that case.

## Redrawing

`Graphics` is fast when it is **not** rebuilt every frame. Rebuilding re-tessellates on the CPU.

```js
// ✗ Re-tessellates every single frame
app.ticker.add(() => {
  g.clear();
  g.circle(x, y, 20).fill(0xff0000);
});

// ✓ Draw once, then transform
g.circle(0, 0, 20).fill(0xff0000);
app.ticker.add((ticker) => {
  g.position.set(x, y);
  g.scale.set(1 + Math.sin(ticker.lastTime / 500) * 0.1);
});
```

Only call `clear()` and redraw when the shape's *geometry* actually changes — moving, scaling,
rotating, tinting, and fading are all transform-level and need no redraw.

Shapes of roughly 100 points or fewer perform comparably to sprites. Beyond that, consider
rendering once to a texture (see `pixi-performance`).

## Graphics as a mask

```js
const mask = new Graphics().circle(100, 100, 60).fill(0xffffff);
container.mask = mask;
app.stage.addChild(mask);   // the mask must be in the scene graph
```

The fill colour is irrelevant — only coverage matters. A mask must be added to the stage (or share
a parent with what it masks) to take effect.
