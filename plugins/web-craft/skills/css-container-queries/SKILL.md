---
name: css-container-queries
description: CSS container queries — @container, container-type, container query units (cqi/cqw/cqb), and style queries. Use when building reusable components that must adapt to their own width rather than the viewport, or replacing media queries in a design system.
---

# Container Queries

A media query asks how big the *window* is. A container query asks how big the *component's parent*
is. For any component reused in more than one context, the second question is the correct one.

**Baseline: widely available since August 2025** (Chrome/Edge 105, Safari 16, Firefox 110).

## Why they beat media queries for components

A card in a 300px sidebar and the same card in a 900px main column have identical viewport width.
A media query cannot tell them apart, so you end up threading context through props or classes:

```css
/* ✗ The card's layout now depends on where the author remembered to put a modifier */
.card--sidebar { display: block; }
@media (min-width: 900px) {
  .card:not(.card--sidebar) { display: grid; grid-template-columns: 200px 1fr; }
}
```

```css
/* ✓ The card measures itself; it is correct anywhere with no modifier */
.card-wrap { container-type: inline-size; }

.card { display: block; }

@container (min-width: 400px) {
  .card { display: grid; grid-template-columns: 200px 1fr; gap: 1.5rem; }
}
```

Drop that card into a sidebar, a modal, a grid cell, or a full-width hero and it adapts on its own.
This is what makes components genuinely portable.

## container-type

```css
.wrap {
  container-type: inline-size;
  container-name: card;         /* optional but recommended */
}

/* Shorthand */
.wrap { container: card / inline-size; }
```

| Value | Containment applied |
|---|---|
| `normal` | No size, no inline-size containment (style queries only) |
| `inline-size` | Style + **inline-size** containment — the one you want |
| `size` | Style + **size** (both axes) + inline-size containment |

**`size` is a trap unless you know you need it.** It applies containment on both axes, so — per MDN
— "If a contextual or explicit size is not available, elements with size containment will
collapse." A container typed `size` without an explicit height collapses to zero.

Use `inline-size` unless you are genuinely querying height and have set one.

## Querying

```css
@container card (min-width: 400px) { … }
@container (min-width: 400px) { … }              /* nearest ancestor container */
@container card (400px <= width <= 800px) { … }  /* range syntax */
@container card (min-width: 400px) and (min-height: 300px) { … }
```

Both `container-name` and the query condition are optional individually, but **at least one must be
present**.

**Styles apply to descendants of the container, never to the container element itself.** This is
the single most common mistake:

```css
/* ✗ .wrap IS the container — it cannot query itself */
.wrap { container-type: inline-size; }
@container (min-width: 400px) {
  .wrap { padding: 2rem; }        /* never matches */
}

/* ✓ Query from a wrapper, style the child */
@container (min-width: 400px) {
  .wrap > .card { padding: 2rem; }
}
```

So the pattern is always: **wrapper is the container, component is the child.**

## Container query units

Sized relative to the query container instead of the viewport:

| Unit | 1% of the container's |
|---|---|
| `cqw` | width |
| `cqh` | height |
| `cqi` | inline size |
| `cqb` | block size |
| `cqmin` | smaller of `cqi` / `cqb` |
| `cqmax` | larger of `cqi` / `cqb` |

`cqi` is the workhorse — it respects writing mode, where `cqw` is physical.

```css
.card-title {
  /* Scales with the card, not the window — the fluid-type idea, correctly scoped */
  font-size: clamp(1.125rem, 4cqi + 0.5rem, 2rem);
}
```

Same accessibility caveat as viewport units: keep a `rem` term so zoom still works, and honour the
`max ≤ 2.5 × min` rule from `css-fluid-type`.

## Style queries

Query a container's custom property value:

```css
.panel { --tone: danger; }

@container style(--tone: danger) {
  .panel-title { color: var(--color-danger); }
}
```

Two limits worth knowing: `!important` is allowed but **ignored** inside style queries, and the
global values `revert` and `revert-layer` are invalid there.

Style queries for arbitrary (non-custom) properties have thinner support than the size queries
above — treat custom properties as the supported surface today.

## A worked component

```css
.media-wrap { container: media / inline-size; }

.media {
  display: grid;
  gap: 1rem;
  grid-template-areas: 'img' 'body';
}

@container media (min-width: 30rem) {
  .media {
    grid-template-areas: 'img body';
    grid-template-columns: minmax(8rem, 30%) 1fr;
    align-items: center;
  }
}

@container media (min-width: 48rem) {
  .media { grid-template-columns: minmax(12rem, 40%) 1fr; gap: 2rem; }
  .media-title { font-size: var(--step-3); }
}
```

Use `rem` in container queries, not `px` — then breakpoints respond to the user's font size too.

## Media queries still have a job

Container queries do not replace media queries. Use media queries for things that are genuinely
about the device or the user:

```css
@media (prefers-reduced-motion: reduce) { … }
@media (prefers-color-scheme: dark) { … }
@media (pointer: coarse) { … }              /* touch — bigger targets */
@media print { … }
@media (min-width: 60rem) { .page { grid-template-columns: 16rem 1fr; } }  /* page shell */
```

Rule of thumb: **page shell → media query; component internals → container query; device/user
preference → media query.**

## Fallback

If you must support pre-2023 browsers:

```css
@supports not (container-type: inline-size) {
  @media (min-width: 900px) {
    .card { display: grid; grid-template-columns: 200px 1fr; }
  }
}
```

Given Baseline-widely-available status, a mobile-first single-column base is usually a sufficient
fallback on its own.

## Related skills

| Need | Skill |
|---|---|
| clamp() derivation and the zoom rule | `css-fluid-type` |
| Grid, subgrid, `:has()`, `@layer` | `css-layout` |
| Viewport units, safe areas | `css-viewport` |
