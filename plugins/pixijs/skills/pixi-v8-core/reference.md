# PixiJS v7 → v8 breaking change reference

Complete table of renames, removals, and behaviour changes. When reading or writing Pixi code,
treat anything in the left column as a bug.

## Application and renderer

| v7 | v8 |
|---|---|
| `new Application({ ... })` (synchronous) | `const app = new Application(); await app.init({ ... })` |
| `app.view` | `app.canvas` |
| `autoDetectRenderer()` sync | `await autoDetectRenderer()` |
| — | `preference: 'webgpu' \| 'webgl'` selects the backend |
| — | Per-backend options: `webgl: { … }, webgpu: { … }` |

## Scene graph

| v7 | v8 |
|---|---|
| `DisplayObject` | Removed — `Container` is the base class |
| `container.name` | `container.label` |
| `NineSlicePlane` | `NineSliceSprite` |
| `SimpleMesh` | `MeshSimple` |
| `SimplePlane` | `MeshPlane` |
| `SimpleRope` | `MeshRope` |
| `AnimatedSprite` unchanged in name | Constructor also accepts an options object |

## Graphics — the largest change

The workflow **reversed**: v7 set a style then drew; v8 defines the shape then styles it.
"Line" terminology became "Stroke", and `draw*` prefixes were dropped.

| v7 | v8 |
|---|---|
| `beginFill(color, alpha)` … `endFill()` | `.fill({ color, alpha })` after the shape; no `endFill` |
| `lineStyle({ width, color })` | `.stroke({ width, color })` after the shape |
| `drawRect(x, y, w, h)` | `rect(x, y, w, h)` |
| `drawCircle(x, y, r)` | `circle(x, y, r)` |
| `drawEllipse(...)` | `ellipse(...)` |
| `drawRoundedRect(...)` | `roundRect(...)` |
| `drawPolygon(...)` | `poly(...)` |
| `drawStar(...)` | `star(...)` |
| `beginHole()` / `endHole()` | `.cut()` |
| `GraphicsGeometry` | `GraphicsContext` |
| `graphics.clear()` | Still `clear()` |

```js
// v7
g.beginFill(0xff0000).lineStyle(2, 0x000000).drawRect(0, 0, 100, 50).endFill();

// v8
g.rect(0, 0, 100, 50).fill({ color: 0xff0000 }).stroke({ width: 2, color: 0x000000 });
```

## Textures and assets

| v7 | v8 |
|---|---|
| `Texture.from('file.png')` loads the file | Does **not** load; `await Assets.load('file.png')` first |
| `BaseTexture` | Removed — replaced by `TextureSource` subtypes |
| `new BaseTexture(image)` | `new ImageSource({ resource: image })` |
| — | Other sources: `CanvasSource`, `VideoSource`, `BufferSource`, `CompressedSource` |
| `Assets.add(alias, src)` | `Assets.add({ alias, src })` — options object |
| `Loader` (v6 legacy) | Removed — use `Assets` |

`Texture.from()` still works for textures **already in the cache**, which is why the mistake often
appears to work in a small demo and then fails once assets move to a CDN.

## Particles

| v7 | v8 |
|---|---|
| `ParticleContainer` accepts `Sprite` children | Accepts lightweight `Particle` objects (`IParticle`) |
| Children in the normal scene graph | Stored in `particleChildren`, outside the scene graph |
| Bounds computed automatically | You must supply `boundsArea` |

## Filters

Custom filters now require an explicit `GlProgram` and typed uniform declarations rather than
loose parameters.

```js
// v8
import { Filter, GlProgram } from 'pixi.js';

const filter = new Filter({
  glProgram: new GlProgram({ vertex, fragment }),
  resources: {
    myUniforms: {
      uIntensity: { value: 0.0, type: 'f32' },
    },
  },
});
```

Uniform `type` strings are required and follow WGSL naming (`f32`, `vec2<f32>`, `vec4<f32>`,
`mat3x3<f32>`), even for the WebGL backend.

## Text

| v7 | v8 |
|---|---|
| `new Text(str, style)` | `new Text({ text: str, style })` — options object |
| `BitmapText(str, style)` | `new BitmapText({ text: str, style })` |
| `TextStyle` fill accepts arrays for gradients | Gradients via `FillGradient` |

## Events

The v7 `interactive` boolean was replaced by `eventMode` in v7.2 and is the only option in v8.

| v7 | v8 |
|---|---|
| `sprite.interactive = true` | `sprite.eventMode = 'static'` |
| `interactive = false` | `eventMode = 'none'` (or `'passive'`) |
| `buttonMode = true` | `cursor = 'pointer'` |

`eventMode` values: `'none'` (ignored entirely, fastest), `'passive'` (children interactive,
self not), `'auto'`, `'static'` (interactive, doesn't move), `'dynamic'` (interactive and
moves independently — needed for hit-testing under a still pointer).

## Import paths

v8 consolidated the `@pixi/*` scoped sub-packages into the single `pixi.js` package.

```js
// v7
import { Application } from '@pixi/app';
import { Sprite } from '@pixi/sprite';

// v8
import { Application, Sprite } from 'pixi.js';
```

`@pixi/react`, `@pixi/sound`, and `pixi-filters` remain separate packages.

## Removed globals

The `PIXI` global namespace is gone in the ESM builds. Code doing `PIXI.Sprite` must switch to
named imports.
