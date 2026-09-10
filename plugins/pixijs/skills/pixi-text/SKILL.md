---
name: pixi-text
description: Rendering text in PixiJS v8 — Text vs BitmapText vs HTMLText, TextStyle options, web fonts, gradients, word wrap, and why per-frame text needs BitmapText. Use when displaying labels, scores, HUD text, or any typography on a Pixi canvas.
---

# PixiJS v8 Text

Three text classes with very different performance characteristics. Choosing wrong is a common
cause of frame drops.

| Class | How it works | Use for |
|---|---|---|
| `Text` | Renders to a canvas, uploads a texture | Static or rarely-changing text |
| `BitmapText` | Quads from a pre-baked font atlas | **Text that changes every frame** |
| `HTMLText` | Renders HTML/CSS via SVG foreignObject | Rich inline markup; slowest |

## Constructors take an options object

```js
// ✗ v7
const label = new Text('Score: 0', { fontSize: 24, fill: 0xffffff });
```

```js
// ✓ v8
import { Text } from 'pixi.js';

const label = new Text({
  text: 'Score: 0',
  style: { fontSize: 24, fill: 0xffffff },
});
```

## The per-frame text trap

Changing `Text.text` re-renders to a canvas and re-uploads a texture — expensive. Doing it every
frame will tank the frame rate.

```js
// ✗ Full canvas re-render + GPU upload, 60 times a second
app.ticker.add(() => {
  fpsLabel.text = `FPS: ${app.ticker.FPS.toFixed(0)}`;
});
```

Two fixes. Update less often:

```js
let acc = 0;
app.ticker.add((ticker) => {
  acc += ticker.deltaMS;
  if (acc >= 500) {                       // twice a second is plenty
    acc = 0;
    fpsLabel.text = `FPS: ${ticker.FPS.toFixed(0)}`;
  }
});
```

Or use `BitmapText`, where changing the string only rearranges quads:

```js
import { Assets, BitmapText } from 'pixi.js';

await Assets.load('fonts/game.fnt');

const score = new BitmapText({
  text: 'Score: 0',
  style: { fontFamily: 'GameFont', fontSize: 32 },
});

app.ticker.add(() => {
  score.text = `Score: ${points}`;        // cheap
});
```

Rule of thumb: **anything updating more than a few times a second should be `BitmapText`** —
scores, timers, FPS counters, damage numbers.

You can generate a bitmap font at runtime from an installed font:

```js
import { BitmapFont } from 'pixi.js';

BitmapFont.install({
  name: 'GameFont',
  style: { fontFamily: 'Arial', fontSize: 32, fill: 0xffffff },
  chars: [['0', '9'], ['a', 'z'], ['A', 'Z'], ' .,:!?'],
});
```

Restrict `chars` to what you actually draw — the full Unicode range makes an enormous atlas.

## TextStyle

```js
const style = {
  fontFamily: 'Arial',
  fontSize: 36,
  fontStyle: 'italic',
  fontWeight: 'bold',
  fill: 0xffffff,
  stroke: { color: 0x000000, width: 4 },     // options object in v8
  align: 'center',                            // multiline alignment
  wordWrap: true,
  wordWrapWidth: 400,
  lineHeight: 44,
  letterSpacing: 1,
  dropShadow: {
    color: 0x000000,
    blur: 4,
    distance: 3,
    angle: Math.PI / 6,
    alpha: 0.5,
  },
};
```

Note `stroke` and `dropShadow` take objects in v8; v7's flat `strokeThickness` / `dropShadowBlur`
properties are gone.

Gradient fills use `FillGradient`, not an array of colours:

```js
import { FillGradient } from 'pixi.js';

const gradient = new FillGradient(0, 0, 0, 40);
gradient.addColorStop(0, 0xffdd00);
gradient.addColorStop(1, 0xff6600);

const title = new Text({ text: 'LEVEL UP', style: { fontSize: 40, fill: gradient } });
```

## Web fonts

Load the font before constructing the `Text`, or the first paint uses a fallback:

```js
await Assets.load({ alias: 'Heading', src: 'fonts/heading.woff2' });

const heading = new Text({
  text: 'Welcome',
  style: { fontFamily: 'Heading', fontSize: 48, fill: 0xffffff },
});
```

If the font is loaded via CSS `@font-face` instead, await `document.fonts.ready` before creating
any `Text`.

## Resolution and crispness

Text baked at resolution 1 looks soft on retina displays:

```js
const label = new Text({
  text: 'Crisp',
  style: { fontSize: 24, fill: 0xffffff },
  resolution: window.devicePixelRatio || 1,
});
```

Scaling a `Text` up after creation blurs it — it is a bitmap. Increase `fontSize` instead of
`scale`. If text must animate in scale, render it large and scale *down*.

## Anchoring and layout

```js
label.anchor.set(0.5);                       // centre both axes
label.position.set(app.screen.width / 2, 40);
```

Measure without rendering:

```js
import { CanvasTextMetrics, TextStyle } from 'pixi.js';

const metrics = CanvasTextMetrics.measureText('Hello world', new TextStyle(style));
console.log(metrics.width, metrics.height, metrics.lines);
```

## HTMLText

```js
import { HTMLText } from 'pixi.js';

const rich = new HTMLText({
  text: 'Press <b>SPACE</b> to <i style="color:#ff0">jump</i>',
  style: { fontSize: 20, fill: 0xffffff },
});
```

Convenient for inline markup, but it renders through an SVG foreignObject — the slowest option,
and font loading inside it is unreliable across browsers. Avoid for anything that updates often.

## Cleanup

`Text` owns a generated texture. Destroy it explicitly:

```js
label.destroy({ texture: true, textureSource: true });
```

Creating `Text` objects per frame without destroying them is a steady GPU memory leak.
