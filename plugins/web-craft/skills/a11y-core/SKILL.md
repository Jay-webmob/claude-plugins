---
name: a11y-core
description: WCAG 2.2 Level AA conformance — the nine new 2.2 criteria, exact contrast ratios, target sizes, focus requirements, and keyboard access. Use when auditing accessibility, checking contrast or tap targets, fixing keyboard navigation, or making a web UI WCAG compliant.
---

# WCAG 2.2 Level AA

AA is the de facto legal target (EN 301 549, ADA Title II, Section 508). WCAG is
backwards-compatible: content conforming to 2.2 also conforms to 2.1 and 2.0.

Full criterion tables in [reference.md](reference.md).

## What changed in 2.2

**Nine new success criteria** were added and — unusually — **one was removed**.

| New SC | Level |
|---|---|
| 2.4.11 Focus Not Obscured (Minimum) | AA |
| 2.4.12 Focus Not Obscured (Enhanced) | AAA |
| 2.4.13 Focus Appearance | AAA |
| 2.5.7 Dragging Movements | AA |
| 2.5.8 Target Size (Minimum) | AA |
| 3.2.6 Consistent Help | A |
| 3.3.7 Redundant Entry | A |
| 3.3.8 Accessible Authentication (Minimum) | AA |
| 3.3.9 Accessible Authentication (Enhanced) | AAA |

**4.1.1 Parsing was removed** — marked "Obsolete and removed". In 2.1/2.0 it now carries the note
that it "should be considered as always satisfied for any content using HTML or XML". Browsers
and assistive tech no longer parse markup independently, so duplicate IDs and unclosed tags fail
other criteria (1.3.1, 4.1.2) if they fail anything at all.

Do not report 4.1.1 violations. Tools built for WCAG 2.0/2.1 still do.

The two new AA criteria with the widest real-world impact are **2.5.8 Target Size** and
**2.4.11 Focus Not Obscured** — check these first on any existing codebase.

## Contrast

| Criterion | Level | Normal text | Large text |
|---|---|---|---|
| 1.4.3 Contrast (Minimum) | AA | **4.5:1** | **3:1** |
| 1.4.6 Contrast (Enhanced) | AAA | **7:1** | **4.5:1** |

**1.4.11 Non-text Contrast** (AA) requires **3:1** for UI component states and for parts of
graphics needed to understand content.

"Large scale" is 18pt, or 14pt bold. The spec's own conversion is **1pt = 1.333px**, so in CSS
pixels that is **24px normal, 18.5px bold**.

Branch on size *before* choosing the threshold — misclassifying text size is the most common
source of false pass/fail:

```js
function requiredRatio(fontSizePx, isBold) {
  const isLarge = fontSizePx >= 24 || (isBold && fontSizePx >= 18.5);
  return isLarge ? 3.0 : 4.5;   // AA
}
```

Exceptions: **incidental** (inactive components, pure decoration, not visible, or part of a
picture with significant other visual content) and **logotypes** (text in a logo or brand name
has no contrast requirement).

Report contrast as numbers, not adjectives. "12px #AAAAAA on white is 2.3:1, below the 4.5:1
minimum — use #595959 for 7.0:1" is actionable; "hard to read" is not.

## Target size

**2.5.8 Target Size (Minimum)**, AA: targets are at least **24×24 CSS px**.
**2.5.5 Target Size (Enhanced)**, AAA: at least **44×44 CSS px**.

The spacing exception is **geometric, not dimensional**. An undersized target passes if a
**24px-diameter circle centred on its bounding box** does not intersect the circle of any other
target.

```js
// ✗ Naive check — false-positives on compliant dense toolbars and inline links
const passes = el.offsetWidth >= 24 && el.offsetHeight >= 24;

// ✓ Size OR the spacing exception
function passes2_5_8(target, others) {
  const r = target.getBoundingClientRect();
  if (r.width >= 24 && r.height >= 24) return true;

  const cx = r.left + r.width / 2;
  const cy = r.top + r.height / 2;

  return others.every((o) => {
    const b = o.getBoundingClientRect();
    const ox = b.left + b.width / 2;
    const oy = b.top + b.height / 2;
    return Math.hypot(cx - ox, cy - oy) >= 24;   // circles of radius 12 must not intersect
  });
}
```

Other exceptions: **inline** (target inside a sentence or block of text), **user agent control**,
**essential**, and **equivalent** (another ≥24px control on the same page does the same thing).

A hit area can be larger than the painted control — pad the target rather than enlarging the art:

```css
.icon-button {
  position: relative;
}
.icon-button::after {
  content: '';
  position: absolute;
  inset: -8px;          /* extends the tap target without changing the visual */
}
```

## Focus

**2.4.11 Focus Not Obscured (Minimum)**, AA, verbatim: "When a user interface component receives
keyboard focus, the component is not **entirely** hidden due to author-created content."

Sticky headers and footers are the usual offender — a focused element scrolled behind a sticky bar
fails. `scroll-padding` fixes it:

```css
html {
  scroll-padding-top: 5rem;      /* height of the sticky header */
  scroll-padding-bottom: 4rem;
}
```

**Never remove the focus indicator without replacing it:**

```css
/* ✗ Fails 2.4.7 Focus Visible */
:focus { outline: none; }

/* ✓ Visible, and only for keyboard users */
:focus-visible {
  outline: 2px solid currentColor;
  outline-offset: 2px;
}
```

**2.4.13 Focus Appearance** (AAA) quantifies the indicator: an area at least as large as a **2 CSS
px thick perimeter** of the component, with a **3:1 contrast ratio** between focused and unfocused
states.

## Keyboard

Everything operable by pointer must be operable by keyboard (2.1.1), and focus must never be
trapped (2.1.2).

```js
// ✗ A div is not focusable and has no role
<div onClick={submit}>Submit</div>

// ✓ Native element — free focus, free Enter/Space, free role
<button onClick={submit}>Submit</button>
```

If you must make a non-interactive element interactive, you owe it `tabindex="0"`, a role, and the
full keyboard model — see `a11y-aria`.

Never use positive `tabindex` values. `tabindex="0"` puts an element in natural DOM order;
`tabindex="-1"` makes it programmatically focusable only. A positive value reorders the entire
page tab sequence and is almost always a bug.

Provide a skip link as the first focusable element:

```html
<a href="#main" class="skip-link">Skip to main content</a>
```

```css
.skip-link {
  position: absolute;
  left: -9999px;
}
.skip-link:focus {
  left: 1rem;
  top: 1rem;
}
```

## Motion

**2.3.3 Animation from Interactions** (AAA), verbatim: "Motion animation triggered by interaction
can be disabled, unless the animation is essential."

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

The OS setting is a preference signal, not consent — pair it with an in-app toggle for anything
substantial. See `motion-craft`.

Separately, **2.2.2 Pause, Stop, Hide** (A) requires a control for any automatic motion lasting
more than five seconds.

## Forms

- Every input needs a programmatic label (`<label for>`, `aria-label`, or `aria-labelledby`).
  Placeholder text is **not** a label — it disappears on input.
- **3.3.1 Error Identification**: describe the error in text, never by colour alone.
- **3.3.7 Redundant Entry** (new in 2.2, Level A): don't ask for the same information twice in one
  process unless re-entry is essential.
- **3.3.8 Accessible Authentication** (new, AA): no cognitive function test (puzzle, memorization,
  transcription) unless there's an alternative. **This means allowing paste into password fields.**

```html
<!-- ✗ Blocking paste fails 3.3.8 -->
<input type="password" onpaste="return false">
```

## Related skills

| Need | Skill |
|---|---|
| Canvas, WebGL, or `<canvas>`-rendered UI | `a11y-canvas` |
| ARIA roles, landmarks, live regions | `a11y-aria` |
| axe-core, Lighthouse, pa11y, CI | `a11y-testing` |
| Verifying in a real browser | `verify-browser` |
