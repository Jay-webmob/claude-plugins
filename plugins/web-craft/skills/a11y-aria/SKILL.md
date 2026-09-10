---
name: a11y-aria
description: Using ARIA correctly — the four rules of ARIA, landmarks, live regions, accessible names, and the most common ARIA misuse patterns. Use when adding roles or aria-* attributes, building custom widgets, or announcing dynamic updates to screen readers.
---

# ARIA

ARIA changes only how assistive technology **perceives** an element. It adds no behaviour, no
keyboard handling, and no focus. Adding `role="button"` to a `<div>` gets you the announcement and
none of the function.

> **"No ARIA is better than bad ARIA."** Incorrect ARIA actively misleads — it is worse than
> omitting it.

## The four rules

The W3C's *Using ARIA* document contains **four** numbered rules. (The phrase "the five rules of
ARIA" is a widespread misquote — it circulates because an unnumbered closing note is often counted
as a fifth.)

**1. Use native HTML instead, if you can.** If a native element or attribute already provides the
semantics and behaviour you need, use it rather than repurposing an element and adding ARIA.

**2. Do not change native semantics unless you really have to.**

```html
<!-- ✗ -->
<h2 role="tab">Heading</h2>

<!-- ✓ Wrap, don't override -->
<div role="tab"><h2>Heading</h2></div>
```

**3. All interactive ARIA controls must be usable with the keyboard.** A `role="button"` must
respond to Enter and Space; a `role="slider"` must respond to arrow keys. Adopting a role means
adopting its entire keyboard model from the ARIA Authoring Practices Guide.

**4. Do not use `role="presentation"` or `aria-hidden="true"` on a focusable element.** Hiding a
focusable element from AT creates a focus stop that announces nothing.

```html
<!-- ✗ Focusable but invisible to a screen reader -->
<button aria-hidden="true">Save</button>
<div aria-hidden="true"><a href="/x">Link</a></div>
```

## Prefer native elements

| Instead of | Use |
|---|---|
| `<div role="button" tabindex="0">` | `<button>` |
| `<div role="checkbox">` | `<input type="checkbox">` |
| `<div role="navigation">` | `<nav>` |
| `<div role="main">` | `<main>` |
| `<span role="link" tabindex="0">` | `<a href>` |
| `<div role="dialog">` | `<dialog>` |
| `<div role="progressbar">` | `<progress>` |

Native elements bring focus behaviour, keyboard handling, form participation, and platform
conventions for free — and stay correct as browsers evolve.

## Accessible names

Precedence, highest first: `aria-labelledby` → `aria-label` → native label
(`<label for>`, `alt`, `<caption>`, element content) → `title`.

```html
<!-- Reference visible text — best, because it stays in sync and is translatable -->
<h2 id="billing">Billing address</h2>
<section aria-labelledby="billing"> … </section>

<!-- Invisible label for an icon-only control -->
<button aria-label="Close dialog"><svg aria-hidden="true">…</svg></button>

<!-- Describe in addition to naming -->
<input id="pw" aria-describedby="pw-help">
<p id="pw-help">At least 12 characters.</p>
```

`aria-label` **overrides visible text**, which breaks voice-control users who say what they see.
If there is a visible label, point at it with `aria-labelledby` rather than retyping it.

Mark decorative graphics `aria-hidden="true"` (inline SVG) or `alt=""` (images) so they aren't
announced. Icons inside a labelled button are always decorative.

## Landmarks

```html
<header>            <!-- role="banner" when a direct child of body -->
  <nav aria-label="Main"> … </nav>
</header>
<main id="main">    <!-- role="main" — exactly one per page -->
  <section aria-labelledby="results-heading"> … </section>
</main>
<aside aria-label="Related"> … </aside>
<footer>            <!-- role="contentinfo" when a direct child of body -->
</footer>
```

Screen reader users navigate by landmark, so this replaces a lot of tabbing. Distinguish repeated
landmarks of the same type by label:

```html
<nav aria-label="Main"> … </nav>
<nav aria-label="Breadcrumb"> … </nav>
```

Don't write the role into the label — "Main navigation" is announced as "Main navigation
navigation".

## Live regions

The only way to announce a change that isn't tied to focus movement.

```html
<div id="status" role="status" aria-live="polite" class="visually-hidden"></div>
<div id="alerts" role="alert" aria-live="assertive"></div>
```

```js
status.textContent = '3 results found';        // announced at the next pause
alerts.textContent = 'Payment failed';         // interrupts immediately
```

| Value | Behaviour | Use for |
|---|---|---|
| `polite` | Waits for a pause | Status, results counts, saved indicators |
| `assertive` | Interrupts | Errors, timeouts, anything urgent |
| `off` | Not announced | Default |

`role="status"` implies `aria-live="polite"`; `role="alert"` implies `assertive`.

**The live region must exist in the DOM before you update it.** Injecting an element that already
contains text usually announces nothing:

```js
// ✗ Often silent — the region and its content arrive together
container.innerHTML = '<div role="alert">Error</div>';

// ✓ Region present up front, text set later
const region = document.getElementById('alerts');
region.textContent = 'Error';
```

Keep messages short, and don't stack multiple assertive regions — they queue and talk over the
user.

Hide live regions visually without removing them from the tree:

```css
.visually-hidden {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip-path: inset(50%);
  white-space: nowrap;
  border: 0;
}
```

Never use `display: none` — that removes it from the accessibility tree entirely.

## States

```html
<button aria-expanded="false" aria-controls="menu">Menu</button>
<div id="menu" hidden> … </div>

<button aria-pressed="true">Bold</button>              <!-- toggle -->
<input aria-invalid="true" aria-describedby="err">     <!-- validation -->
<button aria-busy="true">Saving…</button>              <!-- in progress -->
<li aria-current="page">Dashboard</li>                 <!-- current item -->
```

**Update these in JavaScript when state changes.** A stale `aria-expanded="false"` on an open menu
is worse than no attribute — it actively lies.

`aria-current` accepts `page`, `step`, `location`, `date`, `time`, or `true`.

## Common misuse

```html
<!-- ✗ Redundant — the role is already implied -->
<button role="button">

<!-- ✗ Contradicts the native role -->
<a href="/x" role="button">

<!-- ✗ Hides a focusable element -->
<button aria-hidden="true">

<!-- ✗ Label duplicated and now out of sync with visible text -->
<button aria-label="Submit">Send</button>

<!-- ✗ Announced as "Save button button" -->
<button aria-label="Save button">

<!-- ✗ role without the keyboard model or tabindex -->
<div role="button" onclick="save()">Save</div>

<!-- ✗ Reorders the whole page's tab sequence -->
<input tabindex="3">

<!-- ✗ Placeholder is not a label — it vanishes on input -->
<input placeholder="Email">
```

Misuse ranked by real-world damage:
1. **`role` without keyboard support** — announced as interactive, unusable by keyboard.
2. **State attributes never updated** — the UI reports the wrong state.
3. **`aria-hidden` on focusable content** — silent focus stops.
4. **`aria-label` overriding visible text** — breaks voice control.
5. **Redundant roles on native elements** — harmless but noisy, and a sign the code is guessing.

## Custom widgets

If you build one, implement the full APG keyboard model. A menu button, for example, owes you:
Enter/Space/Down to open, Up/Down to move, Escape to close and restore focus, Home/End, typeahead,
and focus return on selection.

```js
// role="button" on a non-button — the minimum you owe
el.setAttribute('role', 'button');
el.setAttribute('tabindex', '0');
el.addEventListener('keydown', (e) => {
  if (e.key === 'Enter' || e.key === ' ') {
    e.preventDefault();     // Space would scroll the page
    activate();
  }
});
```

Every line of that is free with `<button>`.

## Related skills

| Need | Skill |
|---|---|
| WCAG numbers and criteria | `a11y-core` |
| Canvas and WebGL semantics | `a11y-canvas` |
| Automated and manual testing | `a11y-testing` |
