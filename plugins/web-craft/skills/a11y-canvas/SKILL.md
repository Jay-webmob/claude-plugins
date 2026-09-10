---
name: a11y-canvas
description: Making canvas and WebGL content accessible — fallback content, the parallel/mirror DOM pattern, drawFocusIfNeeded, keyboard alternatives, and why hit regions are dead. Use when adding accessibility to a canvas, PixiJS, three.js, WebGL, or any pixel-rendered UI.
---

# Canvas and WebGL Accessibility

This is the hardest accessibility problem on the web, and there is **no declarative fix**.

Canvas is immediate-mode: it paints pixels and produces **no accessibility tree whatsoever**. A
screen reader encountering `<canvas>` finds an empty element. Every semantic, every focusable
target, every state change, and every keyboard interaction must be constructed by hand in parallel.

The HTML Standard is explicit — authors "must also provide content that, when presented to the
user, conveys essentially the same function or purpose as the canvas's bitmap", and it advises
authors "should not use the canvas element in a document when a more suitable element is
available."

**So the first question is always: does this need to be canvas?** A chart with 8 data points, a
menu, a form, a layout — these belong in the DOM, where accessibility is free. Reserve canvas for
what genuinely requires it: games, image manipulation, tens of thousands of sprites, real 3D.

## Hit regions are dead — never use them

```js
// ✗ Removed from the standard. Ships in NO browser.
ctx.addHitRegion({ id: 'play-button', control: playButton });
ctx.removeHitRegion('play-button');
ctx.clearHitRegions();
```

`addHitRegion` / `removeHitRegion` / `clearHitRegions` were **removed from the WHATWG HTML Living
Standard** (tracked in whatwg/html issue #3407), are marked obsolete on MDN, and Firefox only ever
implemented them behind a preference (Mozilla bug 966591). Tutorials still recommend them.

Split the two concerns instead: **hit-test with geometry, express semantics in the DOM.**

```js
// ✓ Hit-testing via Path2D
const buttonPath = new Path2D();
buttonPath.roundRect(20, 20, 120, 44, 8);
ctx.fill(buttonPath);

canvas.addEventListener('pointerdown', (e) => {
  const r = canvas.getBoundingClientRect();
  const x = e.clientX - r.left;
  const y = e.clientY - r.top;
  if (ctx.isPointInPath(buttonPath, x, y)) play();
});
```

`isPointInStroke()` does the same for stroked paths.

## Fallback content

Content *inside* the canvas element is the accessibility surface. It is not shown to sighted users
but **is** exposed to assistive technology and is focusable.

```html
<!-- ✗ Nothing here for a screen reader -->
<canvas id="game" width="800" height="600"></canvas>

<!-- ✓ Real, focusable, labelled controls -->
<canvas id="game" width="800" height="600">
  <p>Interactive tone matrix. Toggle cells to build a rhythm.</p>
  <button id="cell-0-0" aria-pressed="false">Row 1, beat 1</button>
  <button id="cell-0-1" aria-pressed="false">Row 1, beat 2</button>
</canvas>
```

For purely decorative canvas, remove it from the tree entirely:

```html
<canvas aria-hidden="true"></canvas>
```

For a static generated image, a label may be enough:

```html
<canvas role="img" aria-label="Sales rose from 12k in January to 48k in June"></canvas>
```

`role="img"` tells AT to treat it as a single graphic and stop looking for structure inside.

## drawFocusIfNeeded

The one canvas API that genuinely helps. It draws the platform focus ring on the current path
**only when the given fallback element actually has focus** — so keyboard focus becomes visible in
the pixels:

```js
function drawCell(path, element) {
  ctx.fill(path);
  ctx.drawFocusIfNeeded(path, element);   // no-op unless `element` is focused
}
```

It also tells the browser where the focused element is on screen, which drives screen-magnifier
tracking. Call it during your normal render pass for whichever fallback element is focused.

## The mirror DOM

For anything genuinely interactive, the production-scale answer is a **parallel DOM**: real
focusable elements, one per interactive scene object, kept in sync with the scenegraph and
positioned over it.

Figma documented this publicly — because they render everything in canvas, their accessibility
tree was effectively a single `<input>` regardless of how many layers a file contained. They built
a "Mirror DOM" to fix it.

```html
<div class="canvas-wrap">
  <canvas id="scene" aria-hidden="true"></canvas>
  <div id="a11y-layer" role="application" aria-label="Node editor"></div>
</div>
```

```css
.canvas-wrap { position: relative; }

#a11y-layer {
  position: absolute;
  inset: 0;
  pointer-events: none;      /* the canvas keeps mouse input */
}

#a11y-layer button {
  position: absolute;
  pointer-events: auto;      /* but proxies stay keyboard- and AT-reachable */
  background: none;
  border: 0;
  padding: 0;
  color: transparent;        /* invisible, NOT display:none — that removes it from the tree */
}

#a11y-layer button:focus-visible {
  outline: 2px solid #fff;
  outline-offset: 2px;
}
```

```js
class MirrorDOM {
  constructor(layer) {
    this.layer = layer;
    this.proxies = new Map();
  }

  sync(nodes) {
    const seen = new Set();

    for (const node of nodes) {
      seen.add(node.id);
      let el = this.proxies.get(node.id);

      if (!el) {
        el = document.createElement('button');
        el.addEventListener('click', () => node.activate());
        el.addEventListener('focus', () => this.onFocus(node));
        this.layer.appendChild(el);
        this.proxies.set(node.id, el);
      }

      // Keep the accessible name and state in sync with the scene
      el.textContent = node.label;
      el.setAttribute('aria-pressed', String(node.selected));

      // Position over the rendered object so magnifiers track correctly
      const b = node.getScreenBounds();
      el.style.left = `${b.x}px`;
      el.style.top = `${b.y}px`;
      el.style.width = `${Math.max(b.width, 24)}px`;    // honour 2.5.8
      el.style.height = `${Math.max(b.height, 24)}px`;
    }

    // Remove proxies for objects that left the scene, or focus lands on nothing
    for (const [id, el] of this.proxies) {
      if (!seen.has(id)) {
        el.remove();
        this.proxies.delete(id);
      }
    }
  }

  onFocus(node) {
    node.scrollIntoView();      // keep the focused object on screen (2.4.11)
    node.highlight();           // and visible in the pixels
  }
}
```

Sync on scene change, not every frame — rewriting DOM at 60fps is its own performance problem.
Debounce, or sync only when the object set or labels change.

**`color: transparent` and `opacity: 0` keep elements in the accessibility tree.
`display: none` and `visibility: hidden` remove them.** That distinction is the whole trick.

## Keyboard alternatives

Pointer-driven canvas interactions need a keyboard path. Never leave drag as the only option —
**2.5.7 Dragging Movements** (new in WCAG 2.2, AA) requires a single-pointer alternative, and
keyboard users need one regardless.

```js
canvas.addEventListener('keydown', (e) => {
  const step = e.shiftKey ? 10 : 1;
  switch (e.key) {
    case 'ArrowLeft':  moveSelection(-step, 0); break;
    case 'ArrowRight': moveSelection(step, 0);  break;
    case 'ArrowUp':    moveSelection(0, -step); break;
    case 'ArrowDown':  moveSelection(0, step);  break;
    case 'Enter':
    case ' ':          activateSelection(); e.preventDefault(); break;   // stop page scroll
    case 'Tab':        return;                                          // let focus move naturally
  }
});
```

Announce state changes through a live region — see `a11y-aria`:

```html
<div id="status" role="status" aria-live="polite" class="visually-hidden"></div>
```

```js
status.textContent = `Moved node to column ${col}, row ${row}`;
```

## Games

Full parity is often not achievable, and pretending otherwise helps nobody. What is achievable:

- **All menus, settings, and dialogs in real DOM.** These are the parts that must work, and they
  are ordinary UI. Never render menus into the canvas.
- Remappable controls (see `pixi-input` for the action-layer pattern).
- No reliance on colour alone for game state.
- Captions for dialogue and meaningful sound; visual cues for audio events.
- Adjustable text size and a high-contrast option.
- Difficulty and speed options — the broadest-impact accessibility feature in games.
- Respect `prefers-reduced-motion` for screen shake, parallax, and flashing.

**Flashing content that flashes more than three times per second can trigger seizures** (WCAG
2.3.1, Level A). This is a safety issue, not a preference — check any strobe, muzzle flash, or
rapid colour cycle.

## Verification

Automated tools cannot audit canvas — they see one empty element. Manual checks:

1. Tab through with the mouse unplugged. Can you reach and operate everything?
2. Is focus **visible** at every stop (`drawFocusIfNeeded` or a proxy `:focus-visible`)?
3. Turn on a screen reader (NVDA/JAWS on Windows, VoiceOver on macOS). Is anything announced?
4. Does state change get announced, or does the UI change silently?
5. At 200% zoom (WCAG 1.4.4), is the canvas UI still usable?
6. With `prefers-reduced-motion: reduce`, does motion actually stop?

## Related skills

| Need | Skill |
|---|---|
| WCAG numbers, contrast, target sizes | `a11y-core` |
| Live regions, roles, landmarks | `a11y-aria` |
| PixiJS pointer events and hit areas | `pixi-interaction` |
| PixiJS keyboard/gamepad input | `pixi-input` |
| Driving a real browser to verify | `verify-browser` |
