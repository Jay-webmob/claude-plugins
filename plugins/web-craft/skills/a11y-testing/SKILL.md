---
name: a11y-testing
description: Testing accessibility with axe-core, Lighthouse, and pa11y, plus the manual checks automation cannot perform. Use when setting up accessibility CI, running an audit, or deciding how much automated tools actually prove.
---

# Testing Accessibility

Automated tools are a regression floor, not a conformance proof. Run them, then do the manual pass
— the manual pass is where most real issues are.

## How much automation actually catches

Sources genuinely disagree, and the disagreement is **methodological, not factual**. Cite the
framing, not a bare number:

- **~57%** — Deque's 2021 study found **57.38% of accessibility issues** were detectable
  automatically (anonymized data across 2,000+ audits, 13,000+ pages/page states, ~300,000 issues,
  using axe-core). This counts **volume of issues found in real audits**.
- **~20–40%** — the traditional figure counts **which WCAG success criteria are machine-testable**.

Both are correct measures of different things. Issue-volume skews high because a handful of
machine-detectable problems (missing alt, low contrast, unlabelled inputs) repeat across many
pages; criterion coverage skews low because most criteria need human judgment.

What automation can **never** decide: whether alt text is *accurate*, whether focus order is
*logical*, whether an error message is *understandable*, whether a live region announcement is
*useful*, or whether a custom widget's keyboard model matches user expectation. Nor can it audit
canvas — see `a11y-canvas`.

Never report "0 violations" as "accessible". Report it as "0 automated violations; manual checks
pending or complete".

## axe-core

The engine underneath Lighthouse's accessibility category and most other tooling. MPL-2.0.

```bash
npm i -D @axe-core/cli
npx axe https://example.com
```

In Playwright:

```js
import AxeBuilder from '@axe-core/playwright';
import { test, expect } from '@playwright/test';

test('home page has no automated a11y violations', async ({ page }) => {
  await page.goto('/');

  const results = await new AxeBuilder({ page })
    .withTags(['wcag2a', 'wcag2aa', 'wcag21a', 'wcag21aa', 'wcag22aa'])
    .analyze();

  expect(results.violations).toEqual([]);
});
```

Include `wcag22aa` explicitly — default tag sets in older configurations stop at 2.1 and miss the
new criteria.

In Jest with jsdom:

```js
import { axe, toHaveNoViolations } from 'jest-axe';
expect.extend(toHaveNoViolations);

test('button is accessible', async () => {
  const { container } = render(<Button>Save</Button>);
  expect(await axe(container)).toHaveNoViolations();
});
```

jsdom has no layout engine, so contrast and target-size rules cannot run there. Component-level
axe tests catch naming and role problems; run browser-based tests for anything visual.

Results have four buckets: `violations`, `passes`, `incomplete` (needs review), and `inapplicable`.
**Read `incomplete`** — that's where axe flags things it could not decide, and it is easy to skip.

## Lighthouse

```bash
npx lighthouse https://example.com --only-categories=accessibility --view
```

Scores are weighted by axe user-impact assessments. Lighthouse also lists **manual check items**
that do not affect the score — those are prompts for you, not passes.

A 100 accessibility score means "no automated failures found", not "accessible". Say so when
reporting one.

## pa11y

Useful for crawling many URLs in CI.

```bash
npm i -D pa11y pa11y-ci
npx pa11y --runner axe https://example.com
npx pa11y-ci --sitemap https://example.com/sitemap.xml
```

`.pa11yci`:

```json
{
  "defaults": {
    "standard": "WCAG2AA",
    "runners": ["axe", "htmlcs"],
    "timeout": 30000
  },
  "urls": ["https://example.com/", "https://example.com/pricing"]
}
```

Running both `axe` and `htmlcs` runners surfaces slightly different sets; the union is broader than
either alone.

## The manual pass

Ordered by how much they find per minute spent.

**1. Keyboard only.** Unplug the mouse.
- Can you reach every interactive element?
- Is focus **visible** at every stop?
- Is the order logical and matching visual order?
- Can you escape every component (no traps)?
- Do custom widgets respond to their expected keys?
- Is anything reachable that shouldn't be (hidden menu items still tabbable)?

**2. Zoom to 200%** (WCAG 1.4.4) and **400% / 320px width** (1.4.10 Reflow). Content must reflow
without horizontal scrolling and nothing may be cut off.

**3. Screen reader.** NVDA (Windows, free), VoiceOver (macOS, ⌘F5), or Narrator.
- Are headings a sensible outline?
- Do images have meaningful alt, and is decoration silent?
- Are form errors announced?
- Do dynamic updates announce (live regions)?
- Do controls announce name, role, and state?

**4. Reduced motion.** Enable it at the OS level and confirm motion actually stops.

**5. Contrast.** Spot-check computed values against `a11y-core` thresholds — especially disabled
states, placeholder text, and text over images.

**6. Content sense.** Does link text make sense out of context ("read more" doesn't)? Are error
messages specific?

## CI

```yaml
- name: Accessibility
  run: |
    npm run build
    npm run preview &
    npx wait-on http://localhost:4173
    npx pa11y-ci --config .pa11yci
```

Two rules that keep an a11y gate alive rather than disabled:

1. **Fail the build on new violations, not on the existing backlog.** Baseline what's already
   there, gate on regressions.
2. **Never auto-fix a11y issues mechanically.** Adding `alt=""` to every image to clear a rule
   makes the score green and the page worse.

## Reporting

For each issue give: the **criterion ID and level**, the **element** (`file:line` or selector),
what a user actually experiences, and the concrete fix.

> **1.4.3 Contrast (Minimum), AA** — `src/components/Card.tsx:34`
> Body text `#999999` on `#FFFFFF` is 2.85:1, below the 4.5:1 minimum for text under 24px.
> Low-vision users cannot read the card description.
> Fix: `#595959` gives 7.0:1.

Not: "improve contrast in cards".

## Related skills

| Need | Skill |
|---|---|
| The criteria and exact numbers | `a11y-core` |
| ARIA correctness | `a11y-aria` |
| Canvas/WebGL (automation is blind here) | `a11y-canvas` |
| Driving a real browser | `verify-browser` |
