---
name: pixi-game-loop
description: Game loop architecture in PixiJS v8 — fixed timestep updates, scene and state machines, pause/resume, and separating update from render. Use when structuring an HTML5 game, managing game scenes or screens, or needing deterministic physics on a Pixi canvas.
---

# PixiJS v8 Game Loop

`app.ticker` gives you a variable-timestep loop. That is fine for tweens and cosmetic motion, and
wrong for physics and anything that must be deterministic.

## Fixed timestep

Variable delta means collision and physics behave differently at 30fps and 144fps — objects
tunnel through walls on slow frames. Accumulate elapsed time and step a constant amount:

```js
const FIXED_STEP = 1000 / 60;      // 16.667ms
const MAX_STEPS = 5;               // never spiral after a long stall

let accumulator = 0;

app.ticker.add((ticker) => {
  accumulator += Math.min(ticker.deltaMS, 250);   // clamp tab-restore spikes

  let steps = 0;
  while (accumulator >= FIXED_STEP && steps < MAX_STEPS) {
    fixedUpdate(FIXED_STEP / 1000);               // seconds
    accumulator -= FIXED_STEP;
    steps++;
  }
  if (steps === MAX_STEPS) accumulator = 0;       // drop the backlog

  render(accumulator / FIXED_STEP);               // 0..1 interpolation factor
});

function fixedUpdate(dt) {
  player.vy += GRAVITY * dt;
  player.px += player.vx * dt;
  player.py += player.vy * dt;
  resolveCollisions();
}

function render(alpha) {
  // Interpolate between previous and current physics state to avoid judder
  player.sprite.x = player.prevX + (player.px - player.prevX) * alpha;
  player.sprite.y = player.prevY + (player.py - player.prevY) * alpha;
}
```

The `MAX_STEPS` guard prevents the death spiral: a slow frame queues extra steps, which make the
next frame slower, which queues more. Capping steps drops time instead of freezing.

Store `prevX`/`prevY` at the start of each `fixedUpdate` for the interpolation in `render`.

## Scene management

Keep each screen self-contained with a consistent lifecycle, so teardown is never forgotten.

```js
class Scene extends Container {
  async enter() {}        // load assets, build the display list
  update(dt) {}           // per-frame logic
  exit() {}               // remove listeners, destroy resources
}

class SceneManager {
  constructor(app) {
    this.app = app;
    this.current = null;
    app.ticker.add(this.onTick);
  }

  onTick = (ticker) => {
    this.current?.update(ticker.deltaMS / 1000);
  };

  async change(SceneClass, params) {
    if (this.current) {
      this.current.exit();
      this.app.stage.removeChild(this.current);
      this.current.destroy({ children: true });
    }

    this.current = new SceneClass(this.app, params);
    this.app.stage.addChild(this.current);
    await this.current.enter();
  }

  destroy() {
    this.app.ticker.remove(this.onTick);
    this.current?.exit();
  }
}
```

Because the manager owns the single ticker callback, scenes never register their own — which is
what usually leaks.

```js
class GameScene extends Scene {
  async enter() {
    await Assets.loadBundle('level-1');
    this.player = new Player(Assets.get('player'));
    this.addChild(this.player);
    window.addEventListener('keydown', this.onKey);
  }

  update(dt) {
    this.player.update(dt);
  }

  exit() {
    window.removeEventListener('keydown', this.onKey);
    Assets.unloadBundle('level-1');
  }
}

await sceneManager.change(GameScene);
```

## State machines

For entity behaviour — idle, running, attacking, dead — a small state machine beats a growing pile
of booleans.

```js
class StateMachine {
  constructor(states, initial) {
    this.states = states;
    this.current = initial;
    this.states[initial].enter?.();
  }

  transition(next) {
    if (next === this.current) return;
    this.states[this.current].exit?.();
    this.current = next;
    this.states[next].enter?.();
  }

  update(dt) {
    this.states[this.current].update?.(dt);
  }
}

const hero = new StateMachine({
  idle: {
    enter: () => sprite.setState('idle'),
    update: () => { if (input.left || input.right) hero.transition('run'); },
  },
  run: {
    enter: () => sprite.setState('run'),
    update: (dt) => {
      body.x += input.axis * SPEED * dt;
      if (!input.left && !input.right) hero.transition('idle');
    },
  },
  jump: {
    enter: () => { sprite.setState('jump'); body.vy = -JUMP; },
    update: () => { if (body.grounded) hero.transition('idle'); },
  },
}, 'idle');
```

The `if (next === this.current) return` guard prevents re-entering a state every frame, which
would restart its animation and appear frozen.

## Pause and resume

```js
function pause() {
  app.ticker.stop();          // stops updates and rendering
}

function resume() {
  app.ticker.start();
}
```

To keep rendering (so a pause menu still animates) while freezing game logic, gate the update
instead:

```js
let paused = false;

app.ticker.add((ticker) => {
  if (!paused) updateGame(ticker.deltaMS);
  updateUI(ticker.deltaMS);       // menus keep animating
});
```

Auto-pause when the tab is hidden, otherwise the first frame back carries a huge delta:

```js
document.addEventListener('visibilitychange', () => {
  if (document.hidden) {
    paused = true;
  } else {
    paused = false;
    accumulator = 0;            // discard the gap
  }
});
```

## Layer ordering

Set up render layers once, at the top of the scene, rather than fighting `zIndex` later:

```js
class GameScene extends Scene {
  async enter() {
    this.background = new Container();
    this.world      = new Container();
    this.effects    = new Container();
    this.hud        = new Container();

    this.addChild(this.background, this.world, this.effects, this.hud);
    this.hud.interactiveChildren = true;
    this.background.eventMode = 'none';
  }
}
```

For ordering *within* the world layer (a y-sorted top-down game):

```js
this.world.sortableChildren = true;
sprite.zIndex = sprite.y;        // objects lower on screen draw in front
```

`sortableChildren` re-sorts every frame; on large worlds, sort only when something moves.

## Structure

```
src/
  main.js            app init, Assets.init, first scene
  SceneManager.js
  scenes/
    BootScene.js     loading bar
    MenuScene.js
    GameScene.js
  entities/
    Player.js
    Enemy.js
  systems/
    physics.js
    collision.js
    input.js
```

Keep entities as data + behaviour and let scenes own the wiring; that keeps `update` functions
testable without a renderer.
