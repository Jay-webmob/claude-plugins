---
name: pixi-assets
description: Loading textures, spritesheets, fonts, and audio in PixiJS v8 with the Assets API — Assets.load, bundles, manifests, background loading, progress bars, and unloading. Use when loading images, sprites, atlases, or any asset for a Pixi canvas or game.
---

# PixiJS v8 Assets

In v8, `Texture.from(url)` **no longer loads the file**. Everything is loaded through `Assets`.

```js
// ✗ v7 — silently gives an empty texture in v8
const texture = Texture.from('bunny.png');
const sprite = new Sprite(texture);
```

```js
// ✓ v8
const texture = await Assets.load('bunny.png');
const sprite = new Sprite(texture);
```

`Texture.from()` still resolves textures **already in the cache**, which is why this bug often
survives a local demo and then breaks when assets move to a CDN or load slower.

## Loading

```js
import { Assets } from 'pixi.js';

// Single asset
const texture = await Assets.load('images/hero.png');

// Several at once — resolves to an object keyed by URL
const assets = await Assets.load(['images/hero.png', 'images/enemy.png']);
const hero = assets['images/hero.png'];

// With an alias, so the rest of the codebase never repeats the path
await Assets.load({ alias: 'hero', src: 'images/hero.png' });
const heroTex = Assets.get('hero');
```

Loading the same URL twice returns the same cached texture rather than re-fetching.

Supported out of the box: PNG, JPG, GIF, WebP, AVIF, SVG, MP4, WebM, JSON, TTF/WOFF/WOFF2, and
compressed textures (Basis, DDS, KTX).

## Bundles and manifests

For anything beyond a handful of files, declare a manifest up front and load per-scene bundles.

```js
await Assets.init({
  basePath: 'https://cdn.example.com/assets/',
  manifest: {
    bundles: [
      {
        name: 'menu',
        assets: [
          { alias: 'logo', src: 'ui/logo.png' },
          { alias: 'button', src: 'ui/button.png' },
        ],
      },
      {
        name: 'level-1',
        assets: [
          { alias: 'tiles', src: 'levels/tiles.json' },
          { alias: 'player', src: 'chars/player.json' },
        ],
      },
    ],
  },
});

await Assets.loadBundle('menu');
const logo = new Sprite(Assets.get('logo'));
```

`Assets.init()` must be called **once**, before any load, and only once — a second call throws.
`basePath` is prefixed to every relative `src`, so switching to a CDN is a one-line change.

## Progress bars

Both `load` and `loadBundle` take a progress callback receiving `0`–`1`.

```js
await Assets.loadBundle('level-1', (progress) => {
  loadingBar.width = 400 * progress;
});
```

## Background loading

Fetch upcoming assets during idle time so the next scene starts instantly. These run at low
priority and are interrupted by any explicit `load` call.

```js
await Assets.loadBundle('level-1');          // needed now — blocks
Assets.backgroundLoadBundle(['level-2']);    // fetched quietly, not awaited

// Later — resolves immediately if the background load finished
await Assets.loadBundle('level-2');
```

## Unloading

Textures hold GPU memory that garbage collection cannot reclaim. Unload when leaving a level.

```js
await Assets.unload('images/hero.png');
await Assets.unloadBundle('level-1');
```

Anything still displaying an unloaded texture renders as an empty/white quad — remove or
re-point sprites *before* unloading.

## Spritesheets

Load the JSON atlas, not the image; Pixi resolves the image and slices the frames.

```js
const sheet = await Assets.load('sprites/characters.json');

const idle = new Sprite(sheet.textures['hero_idle_01.png']);
const runAnim = new AnimatedSprite(sheet.animations['hero_run']);
```

`sheet.animations` is populated automatically for frames named with a trailing number
(`hero_run_01.png`, `hero_run_02.png`, …). See `pixi-animation`.

## Fonts

```js
await Assets.load({ alias: 'Game', src: 'fonts/game.woff2' });

const label = new Text({
  text: 'Score: 0',
  style: { fontFamily: 'Game', fontSize: 32, fill: 0xffffff },
});
```

Load the font **before** constructing the `Text`, or the first render falls back to a system font
and only corrects itself on the next text change.

## Detecting supported formats

```js
await Assets.init({
  manifest,
  texturePreference: {
    format: ['avif', 'webp', 'png'],   // first supported format wins
    resolution: [2, 1],                // retina, falling back to 1x
  },
});
```

Provide the same asset in several formats and Pixi picks the best the browser supports.

## Typical startup

```js
async function boot() {
  const app = new Application();
  await app.init({ resizeTo: window, background: '#000' });
  document.body.appendChild(app.canvas);

  await Assets.init({ manifest });
  await Assets.loadBundle('menu', (p) => updateLoadingBar(p));

  Assets.backgroundLoadBundle(['level-1']);
  startMenu(app);
}

boot();
```

Order matters: init the app, init Assets, load what the first screen needs, background-load the
rest, then build the scene.
