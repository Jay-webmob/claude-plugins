---
name: pixi-filters
description: Filters and shaders in PixiJS v8 — built-in filters, the pixi-filters library, custom Filter with GlProgram and typed uniforms, blend modes, and filter performance. Use when adding blur, glow, displacement, color grading, or custom WebGL/WGSL shader effects to a Pixi canvas.
---

# PixiJS v8 Filters

Filters are post-processing shaders applied to a `Container` or `Sprite`. In v8, custom filters
require an explicit `GlProgram` and **typed** uniforms.

## Built-in filters

```js
import { BlurFilter, ColorMatrixFilter, NoiseFilter, AlphaFilter } from 'pixi.js';

const blur = new BlurFilter({ strength: 8, quality: 4 });
sprite.filters = [blur];

const grayscale = new ColorMatrixFilter();
grayscale.desaturate();
container.filters = [grayscale];
```

`filters` is always an **array**, even for one filter. Set `filters = null` to remove.

`ColorMatrixFilter` has ready-made presets — `desaturate()`, `sepia()`, `negative()`,
`contrast(n)`, `saturate(n)`, `brightness(n)`, `hue(deg)`, `night(n)`, `polaroid()`. Combine by
chaining; each multiplies onto the current matrix. Call `reset()` to clear.

## The pixi-filters package

Most effects live in the separate `pixi-filters` package:

```bash
npm install pixi-filters
```

```js
import { GlowFilter, OutlineFilter, DropShadowFilter, ShockwaveFilter } from 'pixi-filters';

sprite.filters = [new GlowFilter({ distance: 15, outerStrength: 2, color: 0x00ffff })];
```

Includes glow, outline, drop shadow, bulge/pinch, CRT, glitch, godray, shockwave, reflection,
tilt-shift, zoom blur, and more. Check the package version supports Pixi v8 — v6+ of
`pixi-filters` targets Pixi v8.

## Displacement

Warps a container using a texture's red/green channels as an offset map. The classic water/heat
effect.

```js
import { DisplacementFilter, Assets, Sprite, WRAP_MODES } from 'pixi.js';

const mapTexture = await Assets.load('displacement-map.png');
const mapSprite = new Sprite(mapTexture);
mapSprite.texture.source.addressMode = 'repeat';   // required for seamless scrolling

app.stage.addChild(mapSprite);                     // must be in the scene graph

const displacement = new DisplacementFilter({ sprite: mapSprite, scale: 40 });
container.filters = [displacement];

app.ticker.add((ticker) => {
  mapSprite.x += 1 * ticker.deltaTime;
  mapSprite.y += 0.6 * ticker.deltaTime;
});
```

Two things people miss: the map sprite must be added to the stage, and `addressMode = 'repeat'`
is what makes the scroll seamless.

## Custom filters

v8 requires a `GlProgram` plus explicit typed uniform declarations — v7's loose parameter object
no longer works.

```js
import { Filter, GlProgram } from 'pixi.js';

const vertex = `
  in vec2 aPosition;
  out vec2 vTextureCoord;

  uniform vec4 uInputSize;
  uniform vec4 uOutputFrame;
  uniform vec4 uOutputTexture;

  vec4 filterVertexPosition() {
    vec2 position = aPosition * uOutputFrame.zw + uOutputFrame.xy;
    position.x = position.x * (2.0 / uOutputTexture.x) - 1.0;
    position.y = position.y * (2.0 * uOutputTexture.z / uOutputTexture.y) - uOutputTexture.z;
    return vec4(position, 0.0, 1.0);
  }

  vec2 filterTextureCoord() {
    return aPosition * (uOutputFrame.zw * uInputSize.zw);
  }

  void main() {
    gl_Position = filterVertexPosition();
    vTextureCoord = filterTextureCoord();
  }
`;

const fragment = `
  in vec2 vTextureCoord;
  out vec4 finalColor;

  uniform sampler2D uTexture;
  uniform float uTime;
  uniform float uIntensity;

  void main() {
    vec2 uv = vTextureCoord;
    uv.x += sin(uv.y * 20.0 + uTime) * uIntensity;
    finalColor = texture(uTexture, uv);
  }
`;

const wobble = new Filter({
  glProgram: new GlProgram({ vertex, fragment, name: 'wobble-filter' }),
  resources: {
    wobbleUniforms: {
      uTime:      { value: 0.0,  type: 'f32' },
      uIntensity: { value: 0.02, type: 'f32' },
    },
  },
});

sprite.filters = [wobble];

app.ticker.add((ticker) => {
  wobble.resources.wobbleUniforms.uniforms.uTime += 0.05 * ticker.deltaTime;
});
```

Key points:

- The **vertex shader boilerplate above is required** — Pixi supplies `uInputSize`,
  `uOutputFrame`, and `uOutputTexture`, and the position math maps the filter quad correctly.
  Copy it verbatim unless you know why you're changing it.
- Uniform `type` strings use **WGSL names even for WebGL**: `f32`, `vec2<f32>`, `vec3<f32>`,
  `vec4<f32>`, `mat3x3<f32>`, `i32`.
- Uniforms are read and written through
  `filter.resources.<groupName>.uniforms.<uniformName>`.
- The input texture is `uTexture`; coordinates come in as `vTextureCoord`.
- GLSL 300 es syntax: `in`/`out` rather than `varying`/`attribute`, and `texture()` not
  `texture2D()`.

To also support the WebGPU backend, supply a `gpuProgram` with WGSL alongside `glProgram`. With
only `glProgram`, force the WebGL backend via `preference: 'webgl'` at init.

## Filter area and padding

A filter renders its target into a texture sized to the object's bounds. Effects that spread
outward — glow, blur, drop shadow — get clipped unless you add padding.

```js
const blur = new BlurFilter({ strength: 10 });
blur.padding = 30;
```

Conversely, pin an explicit area to avoid recalculating bounds every frame:

```js
container.filterArea = new Rectangle(0, 0, 800, 600);
```

## Blend modes

Blend modes are set per object, not as filters:

```js
sprite.blendMode = 'add';       // 'normal' | 'add' | 'multiply' | 'screen' | 'overlay' | …
```

`'add'` is the standard choice for glows, fire, and light. Remember from `pixi-performance` that
**a blend mode change breaks the sprite batch** — group same-blend objects into one layer.

## Performance

Every filtered object renders to its own texture, so each filter is at minimum an extra draw call
plus a full-screen-ish fill.

- Filter the **container**, not each of its children.
- Prefer a baked texture over a live filter for static decoration.
- Blur is expensive — lower `quality` (default 4) before lowering `strength`.
- Large `filterArea` means more pixels shaded; keep it tight.
- On mobile, budget one or two filtered layers, not a stack per object.

```js
// ✗ 50 filtered objects → 50 render targets
sprites.forEach((s) => { s.filters = [new GlowFilter()]; });

// ✓ One filtered container
const glowLayer = new Container();
sprites.forEach((s) => glowLayer.addChild(s));
glowLayer.filters = [new GlowFilter()];
```
