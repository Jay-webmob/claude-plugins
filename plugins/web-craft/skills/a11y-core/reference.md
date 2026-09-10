# WCAG 2.2 reference tables

Exact values and criterion IDs. Quote these rather than recalling numbers.

## Conformance levels

**A** — minimum. **AA** — the legal/regulatory target referenced by EN 301 549, ADA Title II
rulemaking, and Section 508. **AAA** — enhanced; W3C states it is not required as a general policy
for entire sites because it is not achievable for all content.

WCAG 2.2 became a W3C Recommendation on **5 October 2023**.

## Contrast — exact thresholds

| SC | Level | Applies to | Ratio |
|---|---|---|---|
| 1.4.3 Contrast (Minimum) | AA | Normal text | **4.5:1** |
| 1.4.3 | AA | Large text | **3:1** |
| 1.4.6 Contrast (Enhanced) | AAA | Normal text | **7:1** |
| 1.4.6 | AAA | Large text | **4.5:1** |
| 1.4.11 Non-text Contrast | AA | UI components, meaningful graphics | **3:1** |

**Large scale** = "at least 18 point or 14 point bold". The Understanding doc's conversion:
"The ratio between sizes in points and CSS pixels is 1pt = 1.333px, therefore 14pt and 18pt are
equivalent to approximately 18.5px and 24px."

→ **24px normal / 18.5px bold** in CSS pixels.

Exceptions for 1.4.3 and 1.4.6:
- **Incidental** — inactive UI components, pure decoration, not visible to anyone, or part of a
  picture that contains significant other visual content.
- **Logotypes** — text that is part of a logo or brand name has no contrast requirement.

Computing the ratio:

```js
function luminance([r, g, b]) {
  const a = [r, g, b].map((v) => {
    v /= 255;
    return v <= 0.03928 ? v / 12.92 : Math.pow((v + 0.055) / 1.055, 2.4);
  });
  return 0.2126 * a[0] + 0.7152 * a[1] + 0.0722 * a[2];
}

function contrastRatio(fg, bg) {
  const l1 = luminance(fg);
  const l2 = luminance(bg);
  return (Math.max(l1, l2) + 0.05) / (Math.min(l1, l2) + 0.05);
}
```

Ratios range 1:1 (identical) to 21:1 (black on white).

## Target size

| SC | Level | Minimum |
|---|---|---|
| 2.5.8 Target Size (Minimum) | AA | **24 × 24 CSS px** |
| 2.5.5 Target Size (Enhanced) | AAA | **44 × 44 CSS px** |

**2.5.8 exceptions:**
- **Spacing** — verbatim: "Undersized targets (those less than 24 by 24 CSS pixels) are positioned
  so that if a 24 CSS pixel diameter circle is centered on the bounding box of each, the circles do
  not intersect another target or the circle for another undersized target."
- **Equivalent** — another control on the same page achieves the same thing at ≥24×24.
- **Inline** — the target is in a sentence or block of text.
- **User agent control** — size is determined by the user agent and not modified by the author.
- **Essential** — a particular presentation is legally required or essential to the information.

**2.5.5 exceptions:** Equivalent (≥44×44 elsewhere on the page), Inline, User Agent Control,
Essential.

## Focus

| SC | Level | Requirement |
|---|---|---|
| 2.4.7 Focus Visible | AA | Keyboard focus indicator is visible |
| 2.4.11 Focus Not Obscured (Min) | AA | Focused component not **entirely** hidden by author content |
| 2.4.12 Focus Not Obscured (Enh) | AAA | No part of the focused component is hidden |
| 2.4.13 Focus Appearance | AAA | ≥ area of a **2 CSS px perimeter**, **3:1** focused-vs-unfocused contrast |

2.4.11 note: content the *user* opened may obscure the focused component, provided the user can
reveal it without advancing the keyboard focus.

## The nine criteria new in WCAG 2.2

| SC | Level | Summary |
|---|---|---|
| 2.4.11 Focus Not Obscured (Minimum) | AA | Sticky headers/footers must not fully hide focus |
| 2.4.12 Focus Not Obscured (Enhanced) | AAA | Stricter version of the above |
| 2.4.13 Focus Appearance | AAA | Quantified focus indicator size and contrast |
| 2.5.7 Dragging Movements | AA | Single-pointer alternative to any drag operation |
| 2.5.8 Target Size (Minimum) | AA | 24×24 CSS px, with the geometric spacing exception |
| 3.2.6 Consistent Help | A | Help mechanisms appear in the same relative order across pages |
| 3.3.7 Redundant Entry | A | Don't re-ask for information already provided in the same process |
| 3.3.8 Accessible Authentication (Min) | AA | No cognitive function test without an alternative |
| 3.3.9 Accessible Authentication (Enh) | AAA | No cognitive function test at all |

**3.3.8 in practice:** allow paste into password fields, allow password managers, and don't require
transcription, puzzles, or memorization. Object recognition and personal-content identification are
permitted alternatives.

## The removed criterion

**4.1.1 Parsing** — "Obsolete and removed" in WCAG 2.2.

W3C's rationale, verbatim: specifications "and browsers have improved how they handle parsing
errors. Also, previously assistive technology did their own markup parsing. Now they rely on the
browser." And: "with today's technology, accessibility issues that would have failed 4.1.1, will
fail other criteria, such as Info and Relationships (SC 1.3.1) or Name, Role, Value (SC 4.1.2).
Therefore 4.1.1 is no longer needed for accessibility."

In WCAG 2.1 and 2.0 it now carries the note: "This Success Criteria should be considered as always
satisfied for any content using HTML or XML."

Tools built against 2.0/2.1 still report duplicate-ID and unclosed-tag violations under 4.1.1.
Those are no longer conformance failures on their own.

## Motion and timing

| SC | Level | Requirement |
|---|---|---|
| 2.2.2 Pause, Stop, Hide | A | Control for automatic motion > 5 seconds |
| 2.3.1 Three Flashes or Below | A | No more than **3 flashes per second** |
| 2.3.3 Animation from Interactions | AAA | Interaction-triggered motion can be disabled |

2.3.3 verbatim: "Motion animation triggered by interaction can be disabled, unless the animation is
essential to the functionality or the information being conveyed."

2.3.1 is a **seizure safety** criterion, not a comfort preference.

## Text resize

**1.4.4 Resize Text** (AA): text can be resized up to **200%** without loss of content or
functionality.

This is what constrains fluid typography — see `css-fluid-type` for the derived
`max <= 2.5 × min` rule and why a `rem` term in `clamp()` is mandatory.

**1.4.10 Reflow** (AA): content reflows without two-dimensional scrolling at 320 CSS px width
(equivalent to 400% zoom on a 1280px viewport).

**1.4.12 Text Spacing** (AA): no loss of content when the user sets line-height 1.5×, paragraph
spacing 2×, letter-spacing 0.12em, word-spacing 0.16em.

## POUR

The four principles all criteria hang from:

- **Perceivable** — information must be presentable in ways users can perceive.
- **Operable** — interface components must be operable.
- **Understandable** — information and operation must be understandable.
- **Robust** — content must work with current and future user agents and assistive technologies.
