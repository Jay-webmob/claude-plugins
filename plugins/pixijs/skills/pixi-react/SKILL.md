---
name: pixi-react
description: Integrating PixiJS v8 with React — @pixi/react v8 with extend(), declarative pixi components, and the manual useEffect mounting pattern with correct cleanup under StrictMode. Use when embedding a Pixi canvas in a React, Next.js, or Vite app.
---

# PixiJS v8 with React

Two approaches: the declarative `@pixi/react` renderer, or manual mounting in a `useEffect`. Both
require careful cleanup — React remounts effects in StrictMode, and a Pixi app that isn't
destroyed leaks a WebGL context.

## @pixi/react v8

```bash
npm install pixi.js@^8 @pixi/react@^8
```

v8 of the library is a rewrite. Components are **not** exported — you register the Pixi classes you
need with `extend()`, then use lowercase `pixi`-prefixed intrinsic elements.

```jsx
import { Application, extend } from '@pixi/react';
import { Container, Graphics, Sprite } from 'pixi.js';

extend({ Container, Graphics, Sprite });     // must run before render

export function Scene() {
  return (
    <Application background="#1099bb" resizeTo={window}>
      <pixiContainer x={100} y={100}>
        <pixiSprite texture={texture} anchor={0.5} />
      </pixiContainer>
    </Application>
  );
}
```

**Using a component you never passed to `extend()` throws.** This is the most common error with
the library — the design keeps bundles small by not importing all of Pixi.

Call `extend()` once at module scope, not inside a component body.

`<Application>` accepts any `ApplicationOptions` as props and handles the async `init()` for you.

### Drawing with Graphics

```jsx
import { useCallback } from 'react';

function Box({ width, height, color }) {
  const draw = useCallback((g) => {
    g.clear();
    g.rect(0, 0, width, height).fill(color);    // v8 shape-then-style
  }, [width, height, color]);

  return <pixiGraphics draw={draw} />;
}
```

Memoize the `draw` callback — an inline arrow re-tessellates the geometry on every render.

### Per-frame updates

```jsx
import { useTick } from '@pixi/react';
import { useRef } from 'react';

function Spinner() {
  const ref = useRef(null);

  useTick((ticker) => {
    if (ref.current) ref.current.rotation += 0.05 * ticker.deltaTime;
  });

  return <pixiSprite ref={ref} texture={texture} anchor={0.5} />;
}
```

Animate through refs, **never** through React state. `setState` at 60fps re-renders the whole tree
and will destroy performance.

### Loading assets

```jsx
import { Assets } from 'pixi.js';
import { useEffect, useState } from 'react';

function useTexture(src) {
  const [texture, setTexture] = useState(null);

  useEffect(() => {
    let cancelled = false;
    Assets.load(src).then((t) => { if (!cancelled) setTexture(t); });
    return () => { cancelled = true; };
  }, [src]);

  return texture;
}

function Hero() {
  const texture = useTexture('/hero.png');
  if (!texture) return null;                   // don't render until loaded
  return <pixiSprite texture={texture} />;
}
```

The `cancelled` flag prevents setting state after unmount when the load resolves late.

## Manual mounting

More control, and the right choice when Pixi drives the whole canvas and React only hosts it.

```jsx
import { useEffect, useRef } from 'react';
import { Application, Assets, Sprite } from 'pixi.js';

export function PixiCanvas() {
  const hostRef = useRef(null);
  const appRef = useRef(null);

  useEffect(() => {
    let destroyed = false;
    const app = new Application();
    appRef.current = app;

    (async () => {
      await app.init({ resizeTo: hostRef.current, background: '#000', antialias: true });

      // StrictMode may have unmounted us while init() was awaiting
      if (destroyed) {
        app.destroy(true, { children: true, texture: true });
        return;
      }

      hostRef.current.appendChild(app.canvas);

      const texture = await Assets.load('/hero.png');
      if (destroyed) return;

      const sprite = new Sprite(texture);
      sprite.anchor.set(0.5);
      sprite.position.set(app.screen.width / 2, app.screen.height / 2);
      app.stage.addChild(sprite);

      app.ticker.add((ticker) => {
        sprite.rotation += 0.01 * ticker.deltaTime;
      });
    })();

    return () => {
      destroyed = true;
      app.destroy(true, { children: true, texture: true });
      appRef.current = null;
    };
  }, []);

  return <div ref={hostRef} style={{ width: '100%', height: '100vh' }} />;
}
```

Three things this handles that naïve versions do not:

1. **The `destroyed` flag.** `app.init()` is async; React 18 StrictMode mounts, unmounts, and
   remounts effects in development. Without the guard you append a canvas after cleanup ran and
   end up with two canvases and a leaked WebGL context.
2. **`app.destroy(true, {...})`** in the cleanup — the `true` removes the canvas from the DOM, and
   the options release children and textures.
3. **Empty dependency array.** Re-creating the app on prop changes is almost never right; push
   prop updates into the running app instead.

### Passing props into a running app

```jsx
useEffect(() => {
  const app = appRef.current;
  if (!app) return;
  app.renderer.background.color = backgroundColor;
}, [backgroundColor]);
```

Expose an imperative API from your Pixi code and call it from effects, rather than tearing down
and rebuilding.

## Next.js and SSR

Pixi needs `window` and will crash during server rendering. Load it client-side only:

```jsx
'use client';

import dynamic from 'next/dynamic';

const PixiCanvas = dynamic(() => import('./PixiCanvas'), { ssr: false });

export default function Page() {
  return <PixiCanvas />;
}
```

The `'use client'` directive alone is not enough — Next still pre-renders client components on the
server. `ssr: false` is what prevents it.

## Sizing

A canvas sized by CSS percentage without a resize observer will not re-render at the new size.
`resizeTo` handles it:

```js
await app.init({ resizeTo: hostRef.current });
```

Give the host element real dimensions — a `<div>` with no height collapses to zero and renders
nothing. See `pixi-responsive`.

## Checklist

| Issue | Cause | Fix |
|---|---|---|
| Two canvases in dev | StrictMode double-mount | `destroyed` flag + `app.destroy()` in cleanup |
| "Cannot read property of null" on nav | App destroyed but ticker still running | Destroy the app, remove ticker callbacks |
| Blank canvas | Host element has no height | Give the host explicit dimensions |
| Crash on `next build` | Pixi imported during SSR | `dynamic(..., { ssr: false })` |
| "not extended" error | Component missing from `extend()` | Add the class to `extend()` |
| Frame rate collapse | State updated per frame | Use refs and `useTick`, not `setState` |
| Memory grows across routes | App never destroyed | `app.destroy(true, { children: true, texture: true })` |
