---
description: Scaffold a new PixiJS v8 project (vanilla TS, React, or game) with Vite and correct v8 setup.
---

Scaffold a new PixiJS v8 project. Argument: `$ARGUMENTS` (optional — may name a template such as
`vanilla`, `react`, or `game`, and/or a target directory).

## Steps

1. **Determine the template.** If `$ARGUMENTS` does not specify one, ask which of:
   - `vanilla` — TypeScript + Vite, plain Pixi canvas. Default.
   - `react` — React + `@pixi/react` v8.
   - `game` — TypeScript + Vite with scene manager, fixed timestep, and input handling.

2. **Determine the target directory.** Use `$ARGUMENTS` if given; otherwise ask. Never scaffold
   into a non-empty directory without confirming with the user first.

3. **Read the relevant skills before writing code** — `pixi-v8-core` at minimum, plus
   `pixi-react` for the React template and `pixi-game-loop`, `pixi-input`, `pixi-camera` for the
   game template. Every generated file must use the v8 API.

4. **Generate the project:**

   - `package.json` with `pixi.js@^8`, `vite`, `typescript` (plus `react`, `react-dom`,
     `@pixi/react@^8` for the React template)
   - `vite.config.ts`
   - `tsconfig.json` with `"strict": true`
   - `index.html` including `<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">`
   - `src/main.ts` (or `src/main.tsx`)
   - `src/style.css` with `html, body { margin: 0; height: 100%; overflow: hidden; }` and
     `canvas { display: block; touch-action: none; }`
   - `public/` for assets
   - `.gitignore`

5. **Install and verify.** Run `npm install`, then `npm run build`. A build failure means the
   generated code is wrong — fix it before reporting success. Report the actual result.

## Required v8 correctness

Every generated project must:

- Use `const app = new Application(); await app.init({...})` — never the sync constructor
- Append `app.canvas`, never `app.view`
- Load textures with `await Assets.load(...)` before constructing any `Sprite`
- Use shape-then-style Graphics: `g.rect(...).fill(...)`, never `beginFill`/`drawRect`
- Scale animation by `ticker.deltaTime`
- Set `resolution: Math.min(window.devicePixelRatio, 2)` and `autoDensity: true`
- Include teardown: `app.destroy(true, { children: true, texture: true })`

## Vanilla entry point

```ts
import { Application, Assets, Sprite } from 'pixi.js';
import './style.css';

async function main() {
  const app = new Application();

  await app.init({
    background: '#1099bb',
    resizeTo: window,
    antialias: true,
    resolution: Math.min(window.devicePixelRatio, 2),
    autoDensity: true,
  });

  document.getElementById('app')!.appendChild(app.canvas);

  const texture = await Assets.load('/logo.png');
  const sprite = new Sprite(texture);
  sprite.anchor.set(0.5);
  sprite.position.set(app.screen.width / 2, app.screen.height / 2);
  app.stage.addChild(sprite);

  app.ticker.add((ticker) => {
    sprite.rotation += 0.02 * ticker.deltaTime;
  });
}

main();
```

For the `game` template, additionally scaffold `src/SceneManager.ts`, `src/scenes/`,
`src/input/Keyboard.ts`, and a fixed-timestep loop as described in the `pixi-game-loop` skill.

Finish by telling the user how to run it (`npm run dev`) and what was verified.
