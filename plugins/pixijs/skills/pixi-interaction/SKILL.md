---
name: pixi-interaction
description: Pointer and touch interaction in PixiJS v8 — eventMode, hit areas, click/tap/hover events, dragging, and cursor styles. Use when making Pixi sprites clickable, adding hover states, building drag-and-drop, or handling touch on a canvas.
---

# PixiJS v8 Interaction

Objects are **not interactive by default**. The v7 `interactive = true` boolean is gone; v8 uses
`eventMode`.

```js
// ✗ v7
sprite.interactive = true;
sprite.buttonMode = true;
```

```js
// ✓ v8
sprite.eventMode = 'static';
sprite.cursor = 'pointer';
```

## eventMode

| Value | Behaviour | Use for |
|---|---|---|
| `'none'` | Ignored entirely, not even for children | Fastest — decorative art |
| `'passive'` | Self not interactive, children still are | Plain containers |
| `'auto'` | Interactive only if a parent is listening | Rare |
| `'static'` | Interactive; assumed not to move on its own | Buttons, UI, most cases |
| `'dynamic'` | Interactive and moves independently | Objects that must fire hover while the pointer is still |

Default to `'static'`. Use `'dynamic'` only when an object moves under a stationary pointer and
must still emit `pointerover` — it costs an extra hit-test pass per frame.

## Events

```js
const button = new Sprite(texture);
button.eventMode = 'static';
button.cursor = 'pointer';

button.on('pointerdown', (event) => {
  console.log('pressed at', event.global.x, event.global.y);
});

button.on('pointerup',    () => button.scale.set(1));
button.on('pointerover',  () => button.tint = 0xdddddd);
button.on('pointerout',   () => button.tint = 0xffffff);
```

Prefer the `pointer*` family — it covers mouse, touch, and pen in one path. The `mouse*` and
`touch*` events exist but require handling both.

| Event | Fires when |
|---|---|
| `pointerdown` / `pointerup` | Press and release |
| `pointerupoutside` | Released after leaving the object — **essential** for resetting button state |
| `pointertap` | A complete click/tap |
| `pointerover` / `pointerout` | Enter and leave |
| `pointermove` | Pointer moves over the object |
| `globalpointermove` | Every move anywhere — needed for dragging |
| `wheel` | Scroll wheel |

Forgetting `pointerupoutside` is the usual cause of buttons stuck in a pressed state.

The event object carries `global` (stage coordinates), `client` (browser coordinates), `button`,
`pointerType` (`'mouse' | 'touch' | 'pen'`), and `getLocalPosition(container)`.

## Hit areas

By default the hit test uses the object's bounds — for a `Container` that means the union of its
children, recomputed as they move. An explicit `hitArea` is both cheaper and more predictable.

```js
import { Rectangle, Circle } from 'pixi.js';

button.hitArea = new Rectangle(0, 0, 200, 60);
knob.hitArea   = new Circle(0, 0, 30);
```

A `hitArea` also lets you make the target **larger than the art** — important for small touch
targets, which should be at least ~44px.

Skip whole subtrees that need no hit-testing:

```js
backgroundLayer.eventMode = 'none';          // never tested
particleLayer.interactiveChildren = false;   // tested itself, children skipped
```

On a busy scene this is one of the cheapest wins available.

## Dragging

Listen for moves globally, not on the sprite — otherwise a fast drag outstrips the pointer and
drops the object.

```js
let dragTarget = null;

sprite.eventMode = 'static';
sprite.cursor = 'grab';
sprite.on('pointerdown', onDragStart);

app.stage.eventMode = 'static';
app.stage.hitArea = app.screen;                 // stage must be hit-testable
app.stage.on('pointerup', onDragEnd);
app.stage.on('pointerupoutside', onDragEnd);

function onDragStart(event) {
  dragTarget = this;
  dragTarget.alpha = 0.7;
  dragTarget.cursor = 'grabbing';

  // Preserve grab offset so the sprite doesn't jump to the cursor
  const local = event.getLocalPosition(dragTarget.parent);
  dragTarget.dragOffset = { x: dragTarget.x - local.x, y: dragTarget.y - local.y };

  app.stage.on('globalpointermove', onDragMove);
}

function onDragMove(event) {
  if (!dragTarget) return;
  const local = event.getLocalPosition(dragTarget.parent);
  dragTarget.position.set(local.x + dragTarget.dragOffset.x, local.y + dragTarget.dragOffset.y);
}

function onDragEnd() {
  if (!dragTarget) return;
  app.stage.off('globalpointermove', onDragMove);
  dragTarget.alpha = 1;
  dragTarget.cursor = 'grab';
  dragTarget = null;
}
```

Two details that are easy to miss: `app.stage.hitArea = app.screen` (without it the stage receives
nothing), and using `getLocalPosition(parent)` so dragging works inside a scaled or scrolled
container.

## A reusable button

```js
function makeButton(texture, onPress) {
  const btn = new Sprite(texture);
  btn.anchor.set(0.5);
  btn.eventMode = 'static';
  btn.cursor = 'pointer';

  let isDown = false;

  btn.on('pointerdown', () => { isDown = true; btn.scale.set(0.95); });
  btn.on('pointerup', () => {
    if (isDown) onPress();
    isDown = false;
    btn.scale.set(1);
  });
  btn.on('pointerupoutside', () => { isDown = false; btn.scale.set(1); });
  btn.on('pointerover', () => { btn.tint = 0xcccccc; });
  btn.on('pointerout',  () => { btn.tint = 0xffffff; isDown = false; btn.scale.set(1); });

  return btn;
}
```

## Cleanup

Listeners keep references alive. Remove them on teardown, or call `destroy()`, which removes
listeners registered on that object:

```js
button.off('pointerdown', handler);
button.removeAllListeners();
button.destroy();
```

Listeners you attached to `app.stage` are **not** cleaned up by destroying a child — remove those
explicitly, as in `onDragEnd` above.

## Touch notes

- Pixi handles multi-touch; each pointer has a distinct `event.pointerId`.
- Prevent page scroll/zoom over the canvas with CSS: `canvas { touch-action: none; }`
- There is no hover on touch — never hide information behind `pointerover` alone.
