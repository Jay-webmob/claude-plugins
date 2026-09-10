---
name: responsive-images
description: Responsive images — srcset and sizes, picture for art direction, aspect-ratio to prevent layout shift, modern formats, and lazy loading. Use when adding images to a page, fixing CLS from images, or optimizing image delivery.
---

# Responsive Images

Two distinct jobs, often conflated:

- **Resolution switching** — same image, different sizes. Use `srcset` + `sizes` on `<img>`.
- **Art direction** — *different* crops or images per breakpoint. Use `<picture>` with `<source>`.

## Resolution switching

```html
<img
  src="photo-800.jpg"
  srcset="photo-400.jpg 400w, photo-800.jpg 800w, photo-1600.jpg 1600w"
  sizes="(min-width: 60rem) 50vw, 100vw"
  width="1600"
  height="900"
  alt="Harbour at dawn"
>
```

`srcset` with `w` descriptors declares each file's **intrinsic pixel width**. `sizes` tells the
browser how wide the image will *render* — the browser combines that with DPR to pick a file.

Two rules that break this silently:

1. **`sizes` relative units resolve against the document root, not the `<img>`.** An `em` in
   `sizes` is root-relative — this trips people constantly.
2. **Never mix `w` and `x` descriptors** in one `srcset`.

For a fixed-size image, `x` descriptors are simpler, and `src` acts as the 1× fallback:

```html
<img src="logo.png" srcset="logo.png 1x, logo@2x.png 2x, logo@3x.png 3x" width="120" height="40" alt="Acme">
```

### sizes="auto"

```html
<img loading="lazy" sizes="auto, 100vw" srcset="…" width="1600" height="900" alt="…">
```

**Only valid with `loading="lazy"`.** If `auto` can't be resolved the browser falls back to the
remaining entries — so **always append a real fallback**, as above.

## Art direction

```html
<picture>
  <source media="(min-width: 60rem)" srcset="hero-wide.avif" type="image/avif">
  <source media="(min-width: 60rem)" srcset="hero-wide.webp" type="image/webp">
  <source srcset="hero-square.avif" type="image/avif">
  <source srcset="hero-square.webp" type="image/webp">
  <img src="hero-square.jpg" width="800" height="800" alt="Product on a workbench">
</picture>
```

Order matters — **the browser takes the first matching `<source>`**, so list narrower formats and
larger breakpoints first. The `<img>` is the mandatory fallback and carries `alt`, `width`,
`height`, and any CSS classes.

Format order by efficiency: **AVIF → WebP → JPEG/PNG**.

## Preventing layout shift

CLS from images is entirely avoidable. Always set `width` and `height`:

```html
<img src="photo.jpg" width="1600" height="900" alt="…">
```

Modern browsers compute `aspect-ratio` from those attributes and reserve the box before the image
loads — even when CSS resizes it:

```css
img {
  max-width: 100%;
  height: auto;        /* preserves the reserved ratio */
}
```

`height: auto` is required — without it, an explicit CSS height overrides the reservation.

When intrinsic dimensions are unknown:

```css
.thumb {
  aspect-ratio: 16 / 9;
  object-fit: cover;
  width: 100%;
}
```

`object-fit: cover` crops to fill; `contain` letterboxes; `object-position` moves the focal point.

## Loading strategy

```html
<!-- Above the fold / the LCP element -->
<img src="hero.jpg" fetchpriority="high" decoding="async" width="1600" height="900" alt="…">

<!-- Below the fold -->
<img src="later.jpg" loading="lazy" decoding="async" width="800" height="600" alt="…">
```

**Never lazy-load the LCP image.** It's the most common self-inflicted LCP regression: the lazy
attribute defers the very element the metric measures.

Preload a critical hero when it's discovered late (e.g. set in CSS or injected by JS):

```html
<link rel="preload" as="image" href="hero.avif" type="image/avif" fetchpriority="high">
```

`decoding="async"` keeps decode off the main thread. `loading="lazy"` on offscreen images is
otherwise a free win.

## Background images

CSS backgrounds can't use `srcset`, but `image-set()` covers DPR and format:

```css
.hero {
  background-image: image-set(
    url('hero.avif') type('image/avif') 1x,
    url('hero@2x.avif') type('image/avif') 2x,
    url('hero.jpg') type('image/jpeg') 1x
  );
}
```

Background images are invisible to assistive technology and are discovered late by the preloader.
Use `<img>` for anything meaningful — including most heroes.

## Alt text

```html
<img src="chart.png" alt="Revenue grew from £12k in January to £48k in June">
<img src="divider.svg" alt="">                        <!-- decorative: empty, not missing -->
<a href="/profile"><img src="avatar.jpg" alt="Your profile"></a>   <!-- describe the destination -->
```

- `alt=""` marks decoration and is correctly ignored. **Omitting `alt` entirely** makes some screen
  readers announce the filename.
- Don't start with "Image of" — the role is already announced.
- Describe the *information*, not the picture. For a chart, give the finding.
- For a complex image, put the full description in text and reference it with `aria-describedby`.

## SVG

```html
<!-- Decorative inline SVG -->
<svg aria-hidden="true" focusable="false"> … </svg>

<!-- Meaningful inline SVG -->
<svg role="img" aria-labelledby="chart-title">
  <title id="chart-title">Revenue by quarter</title>
  …
</svg>
```

`focusable="false"` prevents an old IE/Edge focus stop and is still worth including.

## Related skills

| Need | Skill |
|---|---|
| CLS/LCP thresholds and measurement | `web-vitals` |
| Grid and layout | `css-layout` |
| Alt text and roles in depth | `a11y-aria` |
| GPU texture formats (very different rules) | `three-assets` |
