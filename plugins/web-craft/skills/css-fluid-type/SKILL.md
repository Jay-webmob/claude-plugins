---
name: css-fluid-type
description: Fluid typography and spacing with CSS clamp() — deriving the preferred value from two size/viewport pairs, and the WCAG zoom constraint that makes a rem term mandatory. Use when building a responsive type scale, sizing text fluidly, or replacing breakpoint-based font sizes.
---

# Fluid Typography with clamp()

`clamp(MIN, PREFERRED, MAX)` replaces breakpoint step-changes with a continuous curve. The whole
skill is deriving `PREFERRED` correctly — and keeping it accessible.

## The derivation

Given a size at two viewport widths, solve the line through both points.

Target: **24px at 320px viewport → 48px at 1280px**.

```
slope     = (48 − 24) / (1280 − 320) = 24 / 960 = 0.025
vw term   = 0.025 × 100              = 2.5vw
intercept = 24 − (0.025 × 320)       = 24 − 8 = 16px = 1rem
```

```css
font-size: clamp(1.5rem, 2.5vw + 1rem, 3rem);
```

Verify the endpoints:
- At 320px: `2.5vw` = 8px, `+ 16px` = **24px** ✓ (equals MIN)
- At 1280px: `2.5vw` = 32px, `+ 16px` = **48px** ✓ (equals MAX)

As a reusable function:

```js
function fluid(minPx, maxPx, minVw = 320, maxVw = 1280, root = 16) {
  const slope = (maxPx - minPx) / (maxVw - minVw);
  const vw = (slope * 100).toFixed(4);
  const intercept = (minPx - slope * minVw) / root;
  return `clamp(${minPx / root}rem, ${vw}vw + ${intercept.toFixed(4)}rem, ${maxPx / root}rem)`;
}

fluid(24, 48);  // "clamp(1.5rem, 2.5vw + 1rem, 3rem)"
```

Or in pure CSS:

```css
:root {
  --min-vw: 320;
  --max-vw: 1280;
}

.fluid {
  --min: 24;
  --max: 48;
  --slope: calc((var(--max) - var(--min)) / (var(--max-vw) - var(--min-vw)));
  --intercept: calc(var(--min) - var(--slope) * var(--min-vw));
  font-size: clamp(
    calc(var(--min) * 1px),
    calc(var(--slope) * 100vw + var(--intercept) * 1px),
    calc(var(--max) * 1px)
  );
}
```

## The accessibility constraint

**Never write a `vw`-only preferred value.**

```css
/* ✗ Fails WCAG 1.4.4 Resize Text — vw does not respond to zoom */
font-size: clamp(1.5rem, 4vw, 3rem);
```

Viewport units don't change when a user zooms, so text pinned to `vw` between its bounds refuses to
grow. **1.4.4 Resize Text (AA)** requires text to scale to 200% without loss of content.

Including a `rem` term restores zoom response, because `rem` tracks the root font size:

```css
/* ✓ The rem term scales with zoom and user font-size preference */
font-size: clamp(1.5rem, 2.5vw + 1rem, 3rem);
```

There is a second, sharper constraint. Analysis of the 500%-zoom case yields:

> **max ≤ 2.5 × min**

If the maximum is more than 2.5× the minimum, the clamp fails resize requirements at high zoom
regardless of the rem term. `clamp(1rem, …, 4rem)` is a 4× ratio and fails. `clamp(1.5rem, …,
3rem)` is 2× and passes.

Check it when you write the scale:

```js
const ratioOk = maxPx <= 2.5 * minPx;
```

Keeping ratios at or under 2.5 is also better design — a heading that quadruples between phone and
desktop rarely looks intentional at both ends.

## A type scale

Pick a ratio, generate steps, fluidise each. Common ratios: 1.2 (minor third), 1.25 (major third),
1.333 (perfect fourth), 1.5 (perfect fifth).

Ratio 1.2 at 320px widening to 1.25 at 1280px — computed with the function above, endpoints
verified:

```css
:root {
  --step--1: clamp(0.8333rem, 0.1112vw + 0.8111rem, 0.9rem);    /* 13.3 → 14.4px */
  --step-0:  clamp(1rem,      0.2083vw + 0.9583rem, 1.125rem);  /* 16.0 → 18.0px  body */
  --step-1:  clamp(1.2rem,    0.3437vw + 1.1313rem, 1.406rem);  /* 19.2 → 22.5px */
  --step-2:  clamp(1.44rem,   0.5297vw + 1.3341rem, 1.758rem);  /* 23.0 → 28.1px */
  --step-3:  clamp(1.728rem,  0.7820vw + 1.5716rem, 2.197rem);  /* 27.6 → 35.2px */
  --step-4:  clamp(2.074rem,  1.1210vw + 1.8498rem, 2.747rem);  /* 33.2 → 43.9px */
  --step-5:  clamp(2.488rem,  1.5755vw + 2.1729rem, 3.433rem);  /* 39.8 → 54.9px */
}

h1 { font-size: var(--step-5); }
h2 { font-size: var(--step-3); }
p  { font-size: var(--step-0); }
```

Every step above stays within the 2.5× rule. [Utopia](https://utopia.fyi) generates these.

## Fluid spacing

The same maths applies to space, and fluid space is what keeps a fluid type scale from looking
cramped:

```css
:root {
  --space-s: clamp(0.75rem, 0.5vw + 0.6rem, 1rem);
  --space-m: clamp(1.5rem,  1vw + 1.2rem,   2rem);
  --space-l: clamp(3rem,    2vw + 2.4rem,   4rem);
}

section { padding-block: var(--space-l); }
```

## Line length and line height

Fluid size without a measure limit produces 200-character lines on desktop.

```css
.prose {
  max-width: 65ch;            /* 45–75ch is the readable range */
  font-size: var(--step-0);
  line-height: 1.6;
}

h1, h2, h3 {
  line-height: 1.1;           /* large text needs proportionally less */
  text-wrap: balance;         /* even ragged edges on short headings */
}

p { text-wrap: pretty; }      /* avoids orphans */
```

Line height should fall as size rises. A unitless value on a fluid size handles this partly, but
headings usually want an explicit tighter value.

Serif faces tolerate slightly longer measures and want slightly more leading than sans at the same
size.

## When not to use clamp

- **Component-driven sizing** — a card that must look right in both a sidebar and a full-width grid
  cares about *its own* width, not the viewport. Use container query units. See
  `css-container-queries`.
- **Discrete layout changes** — a two-column-to-one-column switch is a breakpoint, not a curve.
- **Anything that must hit exact values** — clamp interpolates continuously; if a design demands
  exactly 16/20/24, use steps.

## Related skills

| Need | Skill |
|---|---|
| Component-relative sizing | `css-container-queries` |
| Grid, subgrid, `:has()`, layers | `css-layout` |
| Viewport units and safe areas | `css-viewport` |
| WCAG 1.4.4 and zoom requirements | `a11y-core` |
