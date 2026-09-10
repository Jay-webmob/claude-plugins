---
name: pixi-migration
description: Migrating a PixiJS v7 (or v6) codebase to v8 — ordered upgrade procedure, package consolidation, codemod-style find-and-replace patterns, and verification. Use when upgrading Pixi versions, fixing v7 code that breaks on v8, or resolving errors after a Pixi update.
---

# Migrating PixiJS v7 → v8

Work in this order. Later steps depend on earlier ones, and doing Graphics before the imports
compile produces a confusing wall of errors.

## 1. Packages

v8 consolidated the `@pixi/*` sub-packages into one `pixi.js`.

```bash
npm uninstall @pixi/app @pixi/sprite @pixi/graphics @pixi/text @pixi/assets @pixi/core \
              @pixi/display @pixi/ticker @pixi/math @pixi/events @pixi/mesh @pixi/text-bitmap
npm install pixi.js@^8
```

Related packages need matching majors:

```bash
npm install @pixi/react@^8 pixi-filters@^6 @pixi/sound@^6
```

Update imports:

```js
// v7
import { Application } from '@pixi/app';
import { Sprite } from '@pixi/sprite';

// v8
import { Application, Sprite } from 'pixi.js';
```

If the code uses a `PIXI` global, switch to named imports — the ESM builds no longer expose it.

## 2. Application init

```js
// v7
const app = new Application({ width: 800, height: 600, backgroundColor: 0x1099bb });
document.body.appendChild(app.view);
```

```js
// v8
const app = new Application();
await app.init({ width: 800, height: 600, background: 0x1099bb });
document.body.appendChild(app.canvas);
```

This forces the entry point to become async. Wrap top-level setup:

```js
async function main() {
  const app = new Application();
  await app.init({ /* … */ });
  document.body.appendChild(app.canvas);
  // … the rest of the old synchronous setup
}

main();
```

Every `app.view` becomes `app.canvas`.

## 3. Asset loading

The subtlest break, because it often *appears* to work. `Texture.from(url)` no longer fetches.

```js
// v7 — loads the file
const texture = Texture.from('bunny.png');

// v8 — must preload
const texture = await Assets.load('bunny.png');
```

`Texture.from()` still resolves cached textures, so a demo with a preloaded atlas keeps working
while a CDN-hosted asset silently renders empty. Audit **every** `Texture.from` call.

If the code still uses the v6 `Loader`, it is gone entirely — port to `Assets` (see `pixi-assets`).

```js
// v7
Assets.add('hero', 'hero.png');

// v8 — options object
Assets.add({ alias: 'hero', src: 'hero.png' });
```

Replace `BaseTexture`:

```js
// v7
const base = new BaseTexture(imageElement);
const tex = new Texture(base);

// v8
const source = new ImageSource({ resource: imageElement });
const tex = new Texture({ source });
```

## 4. Graphics

The largest mechanical change: style-then-draw became draw-then-style.

```js
// v7
g.beginFill(0xff0000);
g.lineStyle(2, 0x000000);
g.drawRect(0, 0, 100, 50);
g.endFill();

// v8
g.rect(0, 0, 100, 50)
 .fill({ color: 0xff0000 })
 .stroke({ width: 2, color: 0x000000 });
```

Find-and-replace pairs (verify each — argument order and grouping change):

| Find | Replace |
|---|---|
| `.drawRect(` | `.rect(` |
| `.drawCircle(` | `.circle(` |
| `.drawEllipse(` | `.ellipse(` |
| `.drawRoundedRect(` | `.roundRect(` |
| `.drawPolygon(` | `.poly(` |
| `.drawStar(` | `.star(` |
| `.beginFill(c)` | `.fill(c)` — **move after** the shape |
| `.lineStyle({w, c})` | `.stroke({ width, color })` — **move after** the shape |
| `.endFill()` | delete |
| `.beginHole()` / `.endHole()` | `.cut()` |
| `GraphicsGeometry` | `GraphicsContext` |

The reordering cannot be automated safely — each block needs a human pass. Grep for `beginFill` to
find every site:

```bash
grep -rn "beginFill\|lineStyle\|drawRect\|drawCircle\|endFill" src/
```

## 5. Scene graph renames

```bash
grep -rn "DisplayObject\|\.name =\|NineSlicePlane\|SimpleMesh\|SimplePlane\|SimpleRope" src/
```

| v7 | v8 |
|---|---|
| `DisplayObject` | `Container` |
| `.name` | `.label` |
| `NineSlicePlane` | `NineSliceSprite` |
| `SimpleMesh` | `MeshSimple` |
| `SimplePlane` | `MeshPlane` |
| `SimpleRope` | `MeshRope` |

Be careful with `.name` — the replace must not touch unrelated objects that legitimately have a
`name` property. Check each hit.

## 6. Events

```js
// v7
sprite.interactive = true;
sprite.buttonMode = true;

// v8
sprite.eventMode = 'static';
sprite.cursor = 'pointer';
```

`interactive = false` becomes `eventMode = 'none'`. See `pixi-interaction` for the full mode list.

## 7. Text

```js
// v7
const t = new Text('Hello', { fontSize: 24, fill: 0xffffff, strokeThickness: 4, stroke: 0x000000 });

// v8
const t = new Text({
  text: 'Hello',
  style: { fontSize: 24, fill: 0xffffff, stroke: { color: 0x000000, width: 4 } },
});
```

`stroke` and `dropShadow` take objects. Gradient fills use `FillGradient` instead of a colour
array.

## 8. Filters

Custom filters need the most rewriting — v7's loose uniform object no longer works.

```js
// v8
const filter = new Filter({
  glProgram: new GlProgram({ vertex, fragment }),
  resources: {
    myUniforms: { uTime: { value: 0, type: 'f32' } },
  },
});
```

Shaders must be GLSL 300 es (`in`/`out`, `texture()` not `texture2D()`) and uniforms need WGSL type
strings. See `pixi-filters` for the required vertex boilerplate.

## 9. ParticleContainer

```js
// v7 — accepted sprites
const pc = new ParticleContainer(10000, { position: true });
pc.addChild(new Sprite(texture));

// v8 — Particle objects, explicit bounds
const pc = new ParticleContainer({
  dynamicProperties: { position: true, rotation: false },
  boundsArea: new Rectangle(0, 0, 800, 600),
});
pc.addParticle(new Particle({ texture, x, y }));
```

Particles cannot have children, filters, or event handlers. If the old code relied on those, use a
regular `Container`.

## 10. Verify

```bash
# No v7 API should remain
grep -rn "app\.view\|BaseTexture\|DisplayObject\|beginFill\|drawRect\|endFill\|lineStyle\|\.interactive =" src/

npm run build
```

Then check at runtime, since several breakages fail silently rather than at build time:

- [ ] Canvas appears and renders (catches missing `await app.init()`)
- [ ] **All textures visible, none blank** — blank quads mean an un-preloaded `Texture.from`
- [ ] Shapes have correct fills *and* strokes
- [ ] Buttons and hover states respond (`eventMode`)
- [ ] Text renders in the right font at the right weight
- [ ] Filters render, custom shaders compile (check the console for GLSL errors)
- [ ] No console warnings about deprecated APIs
- [ ] Frame rate matches or beats v7

## Common post-migration errors

| Error / symptom | Cause |
|---|---|
| `Cannot read properties of null (reading 'render')` | `app.init()` not awaited |
| Sprites render blank/white | `Texture.from` without `Assets.load` |
| `g.beginFill is not a function` | Graphics not migrated |
| `Cannot set property name` | `.name` → `.label` |
| Clicks do nothing | Still using `interactive = true` |
| `BaseTexture is not exported` | Needs a `TextureSource` subtype |
| Shader fails to compile | GLSL 100 syntax, or missing uniform `type` |
| Blurry text | `resolution` not set on `Text` |

## Strategy

For a large codebase, migrate incrementally rather than all at once: get it **building** first
(imports, init, renames), then fix runtime behaviour area by area — assets, then graphics, then
events, then filters. Commit after each step so a regression is easy to bisect.

Run `/pixi-audit` afterwards to catch anything missed.
