---
name: css-layout
description: Modern CSS layout — Grid auto-fit vs auto-fill, subgrid, the :has() selector, and cascade layers with @layer. Use when building responsive grids, aligning nested content, styling parents based on children, or organizing CSS specificity.
---

# Modern CSS Layout

## auto-fit vs auto-fill

The single most useful responsive grid line — and the pair most often confused.

```css
.grid {
  display: grid;
  gap: 1.5rem;
  grid-template-columns: repeat(auto-fit, minmax(16rem, 1fr));
}
```

Per MDN: `auto-fill` resolves to the largest number of repetitions that does not overflow the
container. `auto-fit` "behaves as `auto-fill`, except that after placing grid items any empty
repeated tracks are collapsed."

The behavioural difference appears **only when items are fewer than the tracks that fit**:

| | 3 items in a 1200px container (min 16rem) |
|---|---|
| `auto-fill` | Creates ~4 tracks; 3 items sit at 16rem each, **empty 4th track holds space** |
| `auto-fit` | Creates ~4 tracks, **collapses the empty one**; 3 items stretch to fill the row |

- **`auto-fit`** — you want items to grow and fill the row. The usual choice for card grids.
- **`auto-fill`** — you want a stable rhythm, so two items don't become two enormous cards.

A collapsed track is treated as a single fixed track of size zero, so the gap around it collapses
too.

Guard the minimum against overflow on narrow screens, where `16rem` may exceed the container:

```css
grid-template-columns: repeat(auto-fit, minmax(min(16rem, 100%), 1fr));
```

## Subgrid

Lets a nested grid adopt its parent's tracks, so content in separate children aligns.

**Baseline: newly available September 2023** (Firefox 71 in 2019, Chrome/Edge 117, Safari 16).

The classic problem: cards in a row whose titles, bodies, and footers should line up even when
title lengths differ.

```css
.cards {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(16rem, 1fr));
  gap: 1.5rem;
}

.card {
  display: grid;
  grid-row: span 3;              /* card spans the three implicit rows */
  grid-template-rows: subgrid;   /* and inherits their sizing */
  gap: 0.5rem;
}
```

Now every card's title, body, and footer share a baseline row across the whole grid. Without
subgrid this needs equal-height hacks or JavaScript.

`subgrid` works on `grid-template-columns`, `grid-template-rows`, or both.

## :has()

The parent selector. **Baseline widely available December 2023.**

```css
/* Style a card that contains an image */
.card:has(img) { grid-template-columns: 200px 1fr; }

/* A form field with an invalid input */
.field:has(input:invalid) { border-color: var(--color-danger); }

/* A label whose checkbox is checked */
label:has(:checked) { font-weight: 600; }

/* Previous sibling — impossible before :has() */
.row:has(+ .row-selected) { border-bottom-color: transparent; }

/* Page-level state without a JS class toggle */
body:has(dialog[open]) { overflow: hidden; }
```

That last one is worth internalising: a lot of "add a class to `<body>`" JavaScript disappears.

Three rules:

1. It takes the **specificity of its most specific argument** (like `:is()` and `:not()`).
2. It **cannot be nested inside another `:has()`**.
3. It is **not a forgiving selector list** — one invalid selector invalidates the whole thing.
   (`:is()` and `:where()` are forgiving; `:has()` is not.)

Pseudo-elements are neither valid inside `:has()` nor valid as its anchor.

Use `:where()` when you want zero specificity:

```css
:where(.prose) a { text-decoration: underline; }   /* trivially overridable */
```

## Cascade layers

`@layer` controls precedence explicitly, so you stop escalating specificity to win.

```css
@layer reset, base, components, utilities;   /* declare order once, up front */

@layer base {
  a { color: var(--link); }
}

@layer components {
  .button a { color: inherit; }
}

@layer utilities {
  .text-muted { color: var(--muted); }
}
```

**Order for normal declarations, lowest → highest:**

1. User-agent styles
2. User styles
3. Author styles **in layers — first-declared layer is lowest**
4. **Unlayered author styles (highest)**

That fourth point surprises people: **unlayered CSS beats every layer.** Migrating to layers means
layering *everything*, or the unmigrated remainder silently wins.

**For `!important` the layer order inverts**, lowest → highest:

1. Unlayered author `!important`
2. Author `!important` in layers — **first-declared layer now highest**
3. User `!important`
4. UA `!important`
5. Transitions

So a `!important` in your `reset` layer beats one in `utilities` — the exact opposite of normal
declarations.

Practical use — quarantine a third-party stylesheet so your own CSS always wins:

```css
@layer vendor, app;
@import url('vendor.css') layer(vendor);
```

## Logical properties

Prefer these over physical directions; they follow writing mode for free:

```css
.card {
  padding-inline: 1.5rem;      /* left/right in LTR */
  padding-block: 1rem;         /* top/bottom */
  margin-inline: auto;
  border-inline-start: 2px solid;
  inset-inline-start: 0;
}
```

`inline` = text direction axis, `block` = perpendicular. In vertical writing modes they swap
automatically, which physical properties cannot do.

## Intrinsic sizing

```css
.sidebar { width: min(20rem, 100%); }               /* never overflows */
.container { width: min(100% - 2rem, 75rem); }      /* padding + max width in one line */
.hero { padding-block: max(4rem, 8vh); }            /* floor on short viewports */
.thing { width: clamp(16rem, 50%, 40rem); }
```

`min()`, `max()`, and `clamp()` remove a great many media queries.

The centred-container pattern:

```css
.container {
  width: min(100% - 2rem, 75rem);
  margin-inline: auto;
}
```

## Related skills

| Need | Skill |
|---|---|
| Component-relative breakpoints | `css-container-queries` |
| Fluid type and space | `css-fluid-type` |
| Viewport units, safe areas | `css-viewport` |
| Images and layout shift | `responsive-images` |
