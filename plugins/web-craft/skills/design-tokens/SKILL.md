---
name: design-tokens
description: Design token systems in CSS — naming layers, color spaces (OKLCH), theming with custom properties, dark mode, and generating accessible color scales. Use when setting up a design system, defining CSS variables, implementing theming, or building a color palette that meets contrast requirements.
---

# Design Tokens

Tokens are named design decisions. The value of a token system is that a change happens in one
place and the contrast maths is done once, correctly.

## Three layers

Skipping the middle layer is the usual mistake — it's what makes theming possible.

```css
:root {
  /* 1. Primitives — raw values, no meaning. Never used directly in components. */
  --blue-500: oklch(0.55 0.18 250);
  --blue-600: oklch(0.48 0.19 250);
  --gray-50:  oklch(0.98 0.005 250);
  --gray-900: oklch(0.20 0.02 250);

  /* 2. Semantic — meaning, not appearance. This is what components consume. */
  --color-surface: var(--gray-50);
  --color-text: var(--gray-900);
  --color-accent: var(--blue-500);
  --color-accent-hover: var(--blue-600);

  /* 3. Component — only when a component genuinely needs its own knob */
  --button-bg: var(--color-accent);
  --button-radius: var(--radius-md);
}
```

Name semantic tokens for **role**, never appearance. `--color-danger`, not `--color-red` — the day
danger becomes orange, the name shouldn't lie.

## OKLCH

Prefer OKLCH over hex/HSL for generated palettes. It is perceptually uniform: equal lightness
values *look* equally light across hues, which HSL badly fails.

```css
--blue-500: oklch(0.55 0.18 250);
/*                 L    C    H
                   |    |    └── hue 0–360
                   |    └── chroma, 0 = gray
                   └── lightness 0–1 */
```

In HSL, `hsl(60 100% 50%)` (yellow) and `hsl(240 100% 50%)` (blue) claim the same lightness and are
wildly different in perceived brightness. In OKLCH they match. That's what makes a generated scale
usable without hand-tuning every step.

Derive variants from a base without new literals:

```css
.button {
  --base: oklch(0.55 0.18 250);
  background: var(--base);
}
.button:hover  { background: oklch(from var(--base) calc(l - 0.07) c h); }
.button:active { background: oklch(from var(--base) calc(l - 0.12) c h); }
.button:disabled { background: oklch(from var(--base) l calc(c * 0.3) h); }
```

Provide a fallback if you must support older browsers:

```css
--color-accent: #3b6ef5;
@supports (color: oklch(0 0 0)) {
  --color-accent: oklch(0.55 0.18 250);
}
```

## Contrast is a token constraint

A palette that fails contrast is a broken palette. Bake the requirement into the system rather than
discovering it in an audit.

From `a11y-core`: **4.5:1** for normal text, **3:1** for large text (≥24px, or ≥18.5px bold) and for
UI components.

```css
:root {
  --color-text:         var(--gray-900);   /* 16.1:1 on --color-surface ✓ */
  --color-text-muted:   var(--gray-600);   /* 7.0:1  ✓ */
  --color-text-subtle:  var(--gray-500);   /* 4.6:1  ✓ — at the limit, body-text only */
  --color-border:       var(--gray-300);   /* 1.9:1  — decorative only, NOT for UI state */
  --color-border-strong: var(--gray-500);  /* 4.6:1  ✓ — safe for focus rings and inputs */
}
```

Annotate the ratio in a comment next to each text token. It documents the constraint and makes a
regression obvious in review.

The trap: a border token that passes as decoration but is then reused for an input outline, where
it must meet 3:1 under 1.4.11. Keep separate tokens for decorative and functional borders.

## Theming

Redefine semantic tokens; leave primitives and components untouched.

```css
:root {
  color-scheme: light dark;
}

[data-theme='light'] {
  --color-surface: var(--gray-50);
  --color-text: var(--gray-900);
}

[data-theme='dark'] {
  --color-surface: var(--gray-900);
  --color-text: var(--gray-50);
  --color-accent: var(--blue-400);   /* lighter — saturated blues fail on dark */
}
```

Respect the system preference and allow an override:

```css
@media (prefers-color-scheme: dark) {
  :root:not([data-theme='light']) {
    --color-surface: var(--gray-900);
    --color-text: var(--gray-50);
  }
}
```

Set `color-scheme` so form controls, scrollbars, and the caret follow the theme — otherwise you get
a light scrollbar on a dark page.

Dark-mode specifics:
- Pure black (`#000`) with pure white text is uncomfortable; use a near-black surface.
- **Recompute contrast for dark mode.** An accent that passes on white often fails on near-black.
- Reduce saturation slightly — vivid colours appear to vibrate on dark backgrounds.
- Shadows barely read; use lighter surfaces for elevation instead.

Avoid a theme-flash on load:

```html
<script>
  const t = localStorage.getItem('theme');
  if (t) document.documentElement.dataset.theme = t;
</script>
```

Inline in `<head>`, before stylesheets.

## Spacing and radii

```css
:root {
  --space-3xs: 0.25rem;
  --space-2xs: 0.5rem;
  --space-xs:  0.75rem;
  --space-s:   1rem;
  --space-m:   1.5rem;
  --space-l:   2rem;
  --space-xl:  3rem;
  --space-2xl: 4rem;

  --radius-sm: 0.25rem;
  --radius-md: 0.5rem;
  --radius-lg: 0.75rem;
  --radius-full: 9999px;
}
```

Use `rem` so spacing scales with user font size. A small named set beats a 24-step numeric scale —
fewer choices produce more consistent results.

Radii should vary with element size rather than being one value everywhere: a large card and a
small chip sharing `--radius-md` looks wrong at one of the two.

For fluid spacing that scales with viewport, see `css-fluid-type`.

## Naming

```css
--color-{role}-{variant}     /* --color-text-muted, --color-accent-hover */
--space-{t-shirt}            /* --space-s, --space-l */
--radius-{size}
--shadow-{elevation}
--step-{n}                   /* type scale */
```

Consistency matters more than the specific convention. Avoid encoding values in names
(`--space-16`) — the moment it becomes 18px the name is a lie.

## Scope note

This skill covers the **mechanics** of a token system: layering, colour space, contrast
constraints, theming.

It does **not** cover aesthetic direction — what palette to choose, what makes a design distinctive,
or how to avoid templated defaults. For that:

- **[impeccable](https://github.com/pbakaus/impeccable)** (Apache-2.0) — a large craft rule set with
  a deterministic detector:
  ```bash
  claude plugin marketplace add pbakaus/impeccable
  claude plugin install impeccable@impeccable
  ```
- **`frontend-design`** (official Anthropic plugin) — aesthetic direction and avoiding generic
  defaults:
  ```bash
  claude plugin install frontend-design@claude-plugins-official
  ```

Those are actively maintained for that purpose. This plugin does not duplicate them.

## Related skills

| Need | Skill |
|---|---|
| Contrast requirements and exact ratios | `a11y-core` |
| Fluid type and space scales | `css-fluid-type` |
| Cascade layers for token organisation | `css-layout` |
| Motion timing tokens | `motion-craft` |
