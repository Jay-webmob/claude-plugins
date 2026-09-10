---
name: pixi-input
description: Input handling for PixiJS v8 games — keyboard state polling, gamepad API, virtual joysticks, touch controls, and input remapping. Use when adding keyboard controls, gamepad support, or mobile touch controls to a Pixi game.
---

# Input Handling for PixiJS Games

Pixi's event system covers pointer interaction with display objects (see `pixi-interaction`).
Keyboard and gamepad are not Pixi's concern — you wire those to the DOM yourself.

## Poll state, don't react to events

Game logic runs on the ticker, so it needs to ask "is the key down *now*", not react whenever the
OS fires a repeat event. Keyboard repeat delay makes event-driven movement stutter on the first
key press.

```js
// ✗ Movement stutters — OS repeat delay, and misses frames between events
window.addEventListener('keydown', (e) => {
  if (e.code === 'ArrowRight') player.x += 5;
});
```

```js
// ✓ Record state, read it every frame
class Keyboard {
  constructor() {
    this.keys = new Set();
    this.pressed = new Set();      // this frame only
    this.released = new Set();

    window.addEventListener('keydown', this.onDown);
    window.addEventListener('keyup', this.onUp);
    window.addEventListener('blur', this.onBlur);
  }

  onDown = (e) => {
    if (!this.keys.has(e.code)) this.pressed.add(e.code);   // ignore auto-repeat
    this.keys.add(e.code);
  };

  onUp = (e) => {
    this.keys.delete(e.code);
    this.released.add(e.code);
  };

  // Releasing focus mid-key leaves it stuck down forever without this
  onBlur = () => {
    this.keys.clear();
  };

  isDown(code)      { return this.keys.has(code); }
  wasPressed(code)  { return this.pressed.has(code); }
  wasReleased(code) { return this.released.has(code); }

  // Call at the END of each frame
  update() {
    this.pressed.clear();
    this.released.clear();
  }

  destroy() {
    window.removeEventListener('keydown', this.onDown);
    window.removeEventListener('keyup', this.onUp);
    window.removeEventListener('blur', this.onBlur);
  }
}
```

```js
const input = new Keyboard();

app.ticker.add((ticker) => {
  const dt = ticker.deltaMS / 1000;

  if (input.isDown('ArrowLeft'))  player.vx = -SPEED;   // held
  if (input.wasPressed('Space'))  player.jump();        // edge-triggered, once

  updateGame(dt);
  input.update();                                        // clear edge state last
});
```

The distinction matters: movement uses `isDown` (continuous), jumping uses `wasPressed`
(once per press — otherwise holding space jumps every frame).

Use `e.code` (physical key, `'KeyW'`) rather than `e.key` (layout-dependent, `'w'` vs `'W'` vs
`'z'` on AZERTY). `e.code` keeps WASD in the same physical place on every layout.

## An action layer

Don't scatter key codes through gameplay code — map them to named actions once, so remapping and
gamepad support come free.

```js
const bindings = {
  left:   ['ArrowLeft', 'KeyA'],
  right:  ['ArrowRight', 'KeyD'],
  jump:   ['Space', 'KeyW', 'ArrowUp'],
  attack: ['KeyJ', 'KeyZ'],
};

class Actions {
  constructor(keyboard, bindings) {
    this.kb = keyboard;
    this.bindings = bindings;
  }

  isDown(action)     { return this.bindings[action].some((k) => this.kb.isDown(k)); }
  wasPressed(action) { return this.bindings[action].some((k) => this.kb.wasPressed(k)); }

  axis(neg, pos) {
    return (this.isDown(pos) ? 1 : 0) - (this.isDown(neg) ? 1 : 0);
  }
}

const actions = new Actions(input, bindings);
player.vx = actions.axis('left', 'right') * SPEED;
if (actions.wasPressed('jump')) player.jump();
```

## Gamepad

The Gamepad API is poll-only — there are no button events. Read it each frame.

```js
class Gamepad {
  constructor(index = 0) {
    this.index = index;
    this.prevButtons = [];
  }

  poll() {
    const pad = navigator.getGamepads()[this.index];
    if (!pad) { this.pad = null; return; }

    this.pad = pad;
    this.buttons = pad.buttons.map((b) => b.pressed);
  }

  isDown(i)     { return this.buttons?.[i] ?? false; }
  wasPressed(i) { return (this.buttons?.[i] ?? false) && !this.prevButtons[i]; }

  axis(i, deadzone = 0.15) {
    const v = this.pad?.axes[i] ?? 0;
    return Math.abs(v) < deadzone ? 0 : v;      // deadzone kills stick drift
  }

  update() { this.prevButtons = this.buttons ? [...this.buttons] : []; }
}
```

Standard mapping: buttons 0–3 are A/B/X/Y, 4–5 shoulders, 6–7 triggers, 12–15 the d-pad; axes 0/1
are the left stick, 2/3 the right.

A deadzone is mandatory — analogue sticks report small nonzero values at rest, so without it the
player drifts.

Gamepads only appear after the user presses a button, and `navigator.getGamepads()` returns a live
snapshot that must be re-read each frame — caching it gives stale state.

## Virtual joystick

```js
import { Container, Graphics } from 'pixi.js';

class VirtualJoystick extends Container {
  constructor(radius = 60) {
    super();
    this.radius = radius;
    this.value = { x: 0, y: 0 };

    this.base = new Graphics().circle(0, 0, radius).fill({ color: 0xffffff, alpha: 0.2 });
    this.knob = new Graphics().circle(0, 0, radius * 0.4).fill({ color: 0xffffff, alpha: 0.5 });
    this.addChild(this.base, this.knob);

    this.eventMode = 'static';
    this.hitArea = new Circle(0, 0, radius * 1.5);   // generous touch target

    this.on('pointerdown', this.onDown);
    this.on('pointerup', this.onUp);
    this.on('pointerupoutside', this.onUp);
  }

  onDown = (e) => {
    this.pointerId = e.pointerId;                    // track this finger only
    this.on('globalpointermove', this.onMove);
    this.onMove(e);
  };

  onMove = (e) => {
    if (e.pointerId !== this.pointerId) return;      // ignore other fingers

    const local = e.getLocalPosition(this);
    const dist = Math.hypot(local.x, local.y);
    const clamped = Math.min(dist, this.radius);
    const angle = Math.atan2(local.y, local.x);

    this.knob.position.set(Math.cos(angle) * clamped, Math.sin(angle) * clamped);
    this.value.x = (Math.cos(angle) * clamped) / this.radius;
    this.value.y = (Math.sin(angle) * clamped) / this.radius;
  };

  onUp = () => {
    this.off('globalpointermove', this.onMove);
    this.pointerId = null;
    this.knob.position.set(0, 0);
    this.value.x = this.value.y = 0;
  };
}
```

Tracking `pointerId` is what makes it work alongside a second thumb on a jump button.

Disable browser gestures over the canvas:

```css
canvas {
  touch-action: none;
  -webkit-user-select: none;
  user-select: none;
}
```

## Combining sources

Merge every source behind the same action names so gameplay code never branches on device:

```js
class InputManager {
  constructor() {
    this.keyboard = new Keyboard();
    this.gamepad = new Gamepad();
    this.joystick = null;
  }

  get moveX() {
    const kb = (this.keyboard.isDown('KeyD') ? 1 : 0) - (this.keyboard.isDown('KeyA') ? 1 : 0);
    return kb || this.gamepad.axis(0) || this.joystick?.value.x || 0;
  }

  get jumpPressed() {
    return this.keyboard.wasPressed('Space') || this.gamepad.wasPressed(0);
  }

  update() {
    this.gamepad.poll();
    this.keyboard.update();
    this.gamepad.update();
  }
}
```

## Buffering and coyote time

Two small forgiving touches that make platformers feel far better:

```js
let jumpBuffer = 0;      // remember a jump pressed slightly too early
let coyote = 0;          // allow a jump slightly after walking off a ledge

function update(dt) {
  if (input.jumpPressed) jumpBuffer = 0.12;
  jumpBuffer = Math.max(0, jumpBuffer - dt);

  coyote = player.grounded ? 0.1 : Math.max(0, coyote - dt);

  if (jumpBuffer > 0 && coyote > 0) {
    player.vy = -JUMP_FORCE;
    jumpBuffer = 0;
    coyote = 0;
  }
}
```

## Cleanup

Window listeners outlive Pixi objects and keep whole scenes alive:

```js
input.destroy();
```

Call it in a scene's `exit()` (see `pixi-game-loop`) or a React cleanup (see `pixi-react`).
