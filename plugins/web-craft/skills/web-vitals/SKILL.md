---
name: web-vitals
description: Core Web Vitals — exact LCP, INP, and CLS thresholds, what drives each metric, field vs lab measurement, and how to fix regressions. Use when improving page speed, diagnosing slow interactions or layout shift, or interpreting a Lighthouse or CrUX report.
---

# Core Web Vitals

## The thresholds

Assessed at the **75th percentile** of page loads, segmented across mobile and desktop.

| Metric | Good | Needs improvement | Poor |
|---|---|---|---|
| **LCP** Largest Contentful Paint | ≤ **2.5 s** | 2.5–4.0 s | > 4.0 s |
| **INP** Interaction to Next Paint | ≤ **200 ms** | 200–500 ms | > 500 ms |
| **CLS** Cumulative Layout Shift | ≤ **0.1** | 0.1–0.25 | > 0.25 |

p75 matters: fixing the median while the tail stays slow moves nothing. Optimise the slow sessions.

## Field vs lab — do not conflate

- **Lab** — a synthetic run on your machine (Lighthouse, a DevTools trace). Reproducible, good for
  debugging, and **not** what determines your score.
- **Field** — real users, from the Chrome UX Report (CrUX). This is the p75 that counts.

A green Lighthouse score on a fast desktop proves very little. Throttle CPU and network before
drawing conclusions, and check field data when you have it. See `verify-performance` for the
`chrome-devtools-mcp` trace loop, whose summary reports **lab** values and may separately enrich
with **CrUX p75 field** data — read the labels.

Note INP cannot be measured in a standard lab load at all: it requires real interactions.

## LCP

The render time of the largest image, video poster, or text block in the viewport.

Four sub-parts — find which dominates before optimising:

1. **Time to first byte** — server and redirects
2. **Resource load delay** — time between TTFB and the resource starting to load
3. **Resource load duration** — the download itself
4. **Element render delay** — time between load finishing and paint

*Load delay* is the one most often overlooked and most often the culprit: the browser didn't
discover the resource early enough.

```html
<!-- Discovered late: injected by JS or referenced in CSS -->
<link rel="preload" as="image" href="hero.avif" type="image/avif" fetchpriority="high">

<img src="hero.avif" fetchpriority="high" width="1600" height="900" alt="…">
```

Fixes, roughly by impact:

- **Never `loading="lazy"` the LCP element** — the single most common self-inflicted regression.
- Serve AVIF/WebP at the right size (`responsive-images`).
- Eliminate render-blocking CSS/JS in the head; inline critical CSS.
- `font-display: swap` (or `optional`) so text isn't invisible while a webfont loads.
- Preconnect to critical third-party origins.

```css
@font-face {
  font-family: 'Inter';
  src: url('/fonts/inter.woff2') format('woff2');
  font-display: swap;
  size-adjust: 105%;      /* match fallback metrics to reduce swap shift */
}
```

## INP

Replaced FID in March 2024. Measures the **full** interaction — input delay, processing, and the
next paint — reporting roughly the worst interaction of the visit.

The usual cause is long tasks blocking the main thread.

```js
// ✗ One long task blocks paint for the entire loop
function process(items) {
  items.forEach(heavyWork);
  render();
}

// ✓ Yield so the browser can paint between chunks
async function process(items) {
  for (const [i, item] of items.entries()) {
    heavyWork(item);
    if (i % 50 === 0) await scheduler.yield?.() ?? new Promise((r) => setTimeout(r, 0));
  }
  render();
}
```

Give visual feedback *before* the heavy work — INP measures time to the **next paint**, so
painting a pending state immediately is a legitimate and large win:

```js
button.addEventListener('click', async () => {
  button.setAttribute('aria-busy', 'true');       // paints immediately
  await new Promise(requestAnimationFrame);       // let it render
  await doExpensiveThing();                       // then work
  button.removeAttribute('aria-busy');
});
```

Other fixes: debounce input handlers, move heavy work to a Web Worker, virtualize long lists, and
avoid layout thrash (batch reads, then writes).

## CLS

Sum of unexpected layout shifts.

**Score = impact fraction × distance fraction.** A **session window** is a burst of shifts with
under 1 second between them and at most 5 seconds total; CLS is the **largest** such window.

Shifts within 500ms of user input are excluded — which is why *expected* movement (opening an
accordion) doesn't count, but a late-loading ad does.

Causes and fixes:

| Cause | Fix |
|---|---|
| Images without dimensions | `width`/`height` attributes, or `aspect-ratio` |
| Ads/embeds/iframes | Reserve the box with `min-height` or `aspect-ratio` |
| Webfont swap | `size-adjust`, `ascent-override`, or `font-display: optional` |
| Content injected above existing content | Reserve space, or insert below the fold |
| Animating `width`/`height`/`top` | Animate `transform` instead |

```css
/* ✓ Reserve space before content arrives */
.ad-slot { min-height: 250px; }
.embed { aspect-ratio: 16 / 9; }
```

Animate only `transform` and `opacity` — they're composited and never trigger layout:

```css
/* ✗ Triggers layout every frame, and can shift neighbours */
.panel { transition: height 300ms; }

/* ✓ Composited */
.panel { transition: transform 300ms; }
```

## Measuring in the field

```js
import { onLCP, onINP, onCLS } from 'web-vitals';

function send(metric) {
  navigator.sendBeacon('/analytics', JSON.stringify({
    name: metric.name,
    value: metric.value,
    rating: metric.rating,      // 'good' | 'needs-improvement' | 'poor'
    id: metric.id,
  }));
}

onLCP(send);
onINP(send);
onCLS(send);
```

`sendBeacon` survives page unload, where `fetch` may not.

Use the **attribution build** to learn *which element* caused a problem, not just the number:

```js
import { onLCP } from 'web-vitals/attribution';

onLCP((metric) => {
  console.log(metric.attribution.element);       // the actual LCP element
  console.log(metric.attribution.url);
});
```

## Budgets

```json
{
  "budgets": [{
    "resourceSizes": [
      { "resourceType": "script", "budget": 170 },
      { "resourceType": "image", "budget": 500 },
      { "resourceType": "total", "budget": 1000 }
    ],
    "timings": [
      { "metric": "largest-contentful-paint", "budget": 2500 },
      { "metric": "cumulative-layout-shift", "budget": 0.1 }
    ]
  }]
}
```

The ~170 KB JavaScript figure is a commonly cited mid-tier-mobile guideline, not a spec — treat it
as a starting budget to argue about, and measure on a real mid-range device.

## Related skills

| Need | Skill |
|---|---|
| Running traces in a real browser | `verify-performance` |
| Driving and inspecting a page | `verify-browser` |
| Image delivery and CLS | `responsive-images` |
| WebGL/GPU budgets (different rules) | `three-assets` |
