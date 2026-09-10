---
name: motion-craft
description: Motion design for interfaces — duration ranges, easing choice, which properties are cheap to animate, orchestration, and reduced-motion handling. Use when adding transitions, animating UI state, or deciding how long and how a thing should move.
---

# Motion Craft

Motion should explain a change, not decorate one. If removing an animation loses no information,
it was decoration.

## Animate only cheap properties

`transform` and `opacity` are composited — the GPU handles them without layout or paint. Everything
else costs a frame.

```css
/* ✗ Layout on every frame, and shifts neighbours (hurts CLS) */
.panel { transition: height 300ms, top 300ms, width 300ms; }

/* ✓ Composited */
.panel { transition: transform 300ms, opacity 300ms; }
```

| Property | Cost |
|---|---|
| `transform`, `opacity` | Composite only — cheap |
| `filter`, `backdrop-filter` | Paint; GPU-accelerated but not free |
| `color`, `background-color`, `box-shadow` | Paint |
| `width`, `height`, `top`, `margin`, `padding` | **Layout** — most expensive |

For layout-like effects, use `transform: scale()` with `transform-origin`, or the FLIP technique.
Modern alternatives:

```css
/* Animate to auto height */
.accordion { interpolate-size: allow-keywords; transition: height 300ms; }

/* Entry/exit without JS */
.toast { transition: opacity 200ms, display 200ms allow-discrete; }
@starting-style { .toast { opacity: 0; } }
```

Use `will-change` sparingly and remove it after — it permanently promotes a layer and costs memory.

## Duration

| Range | Use for |
|---|---|
| **100–150ms** | Hover, focus, small state changes, button presses |
| **200–300ms** | Dropdowns, tooltips, toggles, small panels |
| **300–500ms** | Modals, drawers, page-level transitions |
| **> 500ms** | Only deliberate, decorative moments |

Larger distances need longer durations; small elements moving far still feel slow at 150ms and
sluggish at 500ms. Exits are typically **faster** than entrances — the user has decided, so get out
of the way.

Anything an interface does frequently should be at the short end. A 400ms dropdown feels broken by
the twentieth use.

## Easing

```css
:root {
  --ease-out:      cubic-bezier(0.22, 1, 0.36, 1);      /* entrances — the default */
  --ease-in:       cubic-bezier(0.64, 0, 0.78, 0);      /* exits */
  --ease-in-out:   cubic-bezier(0.65, 0, 0.35, 1);      /* moves within the viewport */
  --ease-standard: cubic-bezier(0.2, 0, 0, 1);
}
```

- **ease-out** for anything entering or responding to input — fast start reads as responsive. This
  is the right default and covers most cases.
- **ease-in** for exits.
- **ease-in-out** for elements moving from one on-screen place to another.
- **linear** only for continuous motion: spinners, marquees, scroll-linked progress.

Avoid `ease` (the CSS default) — it's subtly mushy. And avoid bounce/elastic easing in UI; it reads
as toy-like and is a commonly flagged anti-pattern. Springs suit direct manipulation (drag,
dismiss) where the user's gesture supplies the energy.

## Orchestration

**One authored moment beats scattered effects.** A page where every section fades-and-slides on
scroll and every card lifts on hover reads as generic — that specific combination is one of the
most recognisable "AI-generated" tells.

Pick one thing to be memorable and keep everything else quiet.

When several elements must move together, stagger them:

```css
.item { animation: reveal 400ms var(--ease-out) both; }
.item:nth-child(1) { animation-delay: 0ms; }
.item:nth-child(2) { animation-delay: 60ms; }
.item:nth-child(3) { animation-delay: 120ms; }
```

Keep stagger steps at **30–80ms**. Larger reads as a slow cascade; cap the total so the last item
isn't waiting a second.

## Reduced motion

Non-negotiable. See `a11y-core` for the WCAG criteria.

```css
@media (prefers-reduced-motion: reduce) {
  *,
  *::before,
  *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
}
```

The blanket override is a safety net, not a design. Better is to keep a *cross-fade* while removing
*movement* — reduced motion targets vestibular triggers (large translation, parallax, zoom,
rotation), not opacity:

```css
.panel {
  transition: transform 300ms var(--ease-out), opacity 300ms;
}

@media (prefers-reduced-motion: reduce) {
  .panel {
    transition: opacity 150ms;   /* keep the fade, drop the movement */
    transform: none;
  }
}
```

In JS:

```js
const reduce = matchMedia('(prefers-reduced-motion: reduce)');
if (reduce.matches) { /* jump to the end state */ }
reduce.addEventListener('change', apply);   // users change this mid-session
```

The OS flag is a preference signal, not consent — for anything substantial, offer an in-app toggle
too.

## Native view transitions

```js
if (document.startViewTransition) {
  document.startViewTransition(() => updateDOM());
} else {
  updateDOM();
}
```

```css
::view-transition-old(root) { animation-duration: 200ms; }

.card { view-transition-name: var(--card-id); }   /* morph between states */

@media (prefers-reduced-motion: reduce) {
  ::view-transition-group(*),
  ::view-transition-old(*),
  ::view-transition-new(*) { animation: none !important; }
}
```

Always feature-detect, and always silence transitions under reduced motion.

## Performance

- Budget: **60fps**, so ~16ms per frame. Verify with a trace (`verify-performance`), not by eye.
- Animating many elements at once is a layer-count problem — batch into one transformed parent.
- Canvas/WebGL animation is a different discipline: see `pixi-animation` and `three-r3f`, where
  `deltaTime` scaling and ticker discipline matter more than CSS easing.

## Scope note

This skill covers motion **mechanics** — timing, easing, cost, accessibility. It deliberately does
not cover general visual design, anti-pattern detection, or aesthetic critique.

For that, install **[impeccable](https://github.com/pbakaus/impeccable)** (Apache-2.0), which
carries a large craft rule set and a deterministic detector:

```bash
claude plugin marketplace add pbakaus/impeccable
claude plugin install impeccable@impeccable
```

Anthropic's official `frontend-design` plugin covers aesthetic direction and avoiding templated
defaults. These are complementary, better maintained for that purpose, and not duplicated here.

## Related skills

| Need | Skill |
|---|---|
| Smooth scroll and scroll-linked motion | `motion-smooth-scroll` |
| Reduced motion as a WCAG requirement | `a11y-core` |
| Frame budgets and traces | `verify-performance` |
| Canvas animation | `pixi-animation` |
