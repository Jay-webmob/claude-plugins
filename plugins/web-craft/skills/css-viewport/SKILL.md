---
name: css-viewport
description: Viewport units and mobile browser chrome — dvh/svh/lvh, the dynamic-unit scroll jump, safe area insets, and viewport-fit=cover. Use when a full-height layout breaks on mobile, content hides behind a notch or home indicator, or 100vh causes scrolling.
---

# Viewport Units and Safe Areas

## The 100vh problem

On mobile, browser chrome (URL bar, toolbar) expands and retracts as you scroll. `100vh` refers to
the **large** viewport — chrome retracted — so `height: 100vh` overflows the visible area whenever
the chrome is showing.

```css
/* ✗ Overflows on mobile whenever browser chrome is visible */
.hero { height: 100vh; }
```

CSS now gives four families:

| Prefix | Meaning |
|---|---|
| `sv*` | **Small** viewport — browser UI **expanded**. Smallest guaranteed area. Stable. |
| `lv*` | **Large** viewport — browser UI **retracted**. Stable. |
| `dv*` | **Dynamic** — tracks the current state. **Not stable.** |
| (none) | `vh`/`vw`/`vi`/`vb` are **equivalent to `lv*`** |

So `100vh` ≡ `100lvh`. That equivalence is why the classic bug exists.

Each family has the full set: `svh svw svi svb svmin svmax`, and likewise for `lv*` and `dv*`.

## Which to use

**`dvh` is not the automatic answer.** MDN warns: "The sizes of the dynamic viewport-percentage
units are not stable, even when the viewport itself is unchanged", and using them "can cause the
content to resize while a user is scrolling a page. This can lead to [a] degraded user
experience."

A `100dvh` hero literally resizes mid-scroll as the URL bar animates, dragging the whole page with
it.

```css
/* ✓ Guaranteed to fit — nothing is cut off, no resize on scroll */
.hero { min-height: 100svh; }

/* ✓ Best of both: fits when chrome is up, fills when retracted */
.hero { min-height: 100svh; }
@supports (height: 100dvh) {
  .hero { min-height: min(100dvh, 100lvh); }
}
```

Guidance:

- **`svh`** — anything that must never be clipped: a hero with a call-to-action, a full-screen
  form, a modal.
- **`dvh`** — elements that *should* track the chrome, typically a fixed bottom bar. Accept the
  animation there; it's the correct behaviour.
- **`lvh` / `vh`** — when you want the tallest interpretation and clipping is fine.
- Prefer **`min-height`** over `height` so content taller than the viewport still scrolls.

## Safe areas

Notches, rounded corners, and home indicators overlap the viewport. `env()` exposes the insets.

**Requires `viewport-fit=cover`**, otherwise the values are always 0:

```html
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
```

```css
.app {
  padding-top: env(safe-area-inset-top, 0px);
  padding-bottom: env(safe-area-inset-bottom, 0px);
  padding-inline: env(safe-area-inset-left, 0px) env(safe-area-inset-right, 0px);
}
```

The insets return `0px` on a rectangular viewport with no obstructions, so the same CSS is safe
everywhere. Always pass the fallback second argument for older parsers.

Combine with your own spacing rather than replacing it:

```css
.bottom-bar {
  padding-bottom: calc(1rem + env(safe-area-inset-bottom, 0px));
}

.header {
  padding-top: calc(0.75rem + env(safe-area-inset-top, 0px));
}
```

`safe-area-max-inset-*` gives the static maximum of each dynamic counterpart — useful to reserve
space once rather than reflowing as UI features appear and disappear.

## Full-bleed inside a constrained container

```css
.container {
  width: min(100% - 2rem, 75rem);
  margin-inline: auto;
}

.full-bleed {
  width: 100vw;
  margin-inline: calc(50% - 50vw);
}
```

Note `100vw` **includes the scrollbar** on desktop, which causes horizontal overflow. Use `100dvw`
where supported, or:

```css
.full-bleed {
  width: 100%;
  margin-inline: calc(50% - 50vw + var(--scrollbar, 0px) / 2);
}
```

```js
document.documentElement.style.setProperty(
  '--scrollbar',
  `${window.innerWidth - document.documentElement.clientWidth}px`
);
```

Or sidestep it entirely with `scrollbar-gutter: stable` on the root.

## Mobile essentials

```css
html {
  -webkit-text-size-adjust: 100%;   /* stop iOS inflating text on rotate */
}

body {
  overscroll-behavior-y: none;      /* disable pull-to-refresh where inappropriate */
}

canvas, .interactive-surface {
  touch-action: none;               /* stop browser pan/zoom stealing pointer input */
}

input, select, textarea {
  font-size: max(16px, 1rem);       /* below 16px, iOS Safari zooms on focus */
}
```

The 16px input rule catches people constantly: any smaller and iOS zooms the page when the field
receives focus.

Never disable zoom — it fails WCAG 1.4.4:

```html
<!-- ✗ Blocks pinch-zoom -->
<meta name="viewport" content="width=device-width, maximum-scale=1, user-scalable=no">
```

## Keyboard-aware layout

The on-screen keyboard shrinks the visual viewport without changing the layout viewport:

```js
if ('virtualKeyboard' in navigator) {
  navigator.virtualKeyboard.overlaysContent = true;   // then use env(keyboard-inset-*)
}

// Broader support: track the visual viewport
visualViewport?.addEventListener('resize', () => {
  document.documentElement.style.setProperty('--vvh', `${visualViewport.height}px`);
});
```

## Related skills

| Need | Skill |
|---|---|
| Canvas sizing, DPR, letterbox vs fill | `pixi-responsive` |
| Container-relative breakpoints | `css-container-queries` |
| Fluid sizing | `css-fluid-type` |
| Layout shift from images | `responsive-images` |
