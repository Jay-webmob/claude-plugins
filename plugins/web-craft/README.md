# Web Craft

Accessibility, modern CSS, browser-verified performance, motion, and 3D asset optimization for
Claude Code.

**17 skills · 2 review agents · 3 commands** · ~2,200 tokens always-on

---

## Table of contents

- [What this covers — and what it deliberately doesn't](#what-this-covers--and-what-it-deliberately-doesnt)
- [Install](#install)
- [Quick start](#quick-start)
- [How it works](#how-it-works)
- [Commands](#commands)
- [Skills](#skills)
- [Review agents](#review-agents)
- [Common workflows](#common-workflows)
- [FAQ](#faq)
- [Reference generation](#reference-generation)
- [Companion plugins](#companion-plugins)
- [Development](#development)

---

## What this covers — and what it deliberately doesn't

There are already good tools for general UI craft and aesthetic direction. This plugin does not
compete with them. It owns the areas they leave uncovered:

- **Canvas and WebGL accessibility** — automated tools see `<canvas>` as one empty element and
  report nothing. Nothing else in the ecosystem covers this properly.
- **GPU and VRAM budgets** — where 3D and canvas projects actually fail, and where page-level
  performance advice stops.
- **Browser-verified truth** — driving a real browser instead of inferring behaviour from source.

Everything here is built from primary sources with exact values: WCAG criterion IDs and contrast
ratios from W3C, the geometric 2.5.8 spacing test, worked `clamp()` derivations, verified npm
versions, and VRAM arithmetic.

Companion to the [pixijs](../PIXIjs) plugin, which it cross-references rather than duplicates.

---

## Install

```bash
claude plugin marketplace add /path/to/web-craft
claude plugin install web-craft@web-craft
```

Verify it loaded:

```bash
claude plugin details web-craft
```

You should see 21 skills (17 knowledge skills + 3 commands, plus one shared name), 2 agents.

> **Restart Claude Code after installing.** Skills and commands work immediately, but the two
> agents only register at session start.

**Optional but recommended** — `/verify-ui` and `/verify-performance` need a browser:

```json
{
  "mcpServers": {
    "chrome-devtools": {
      "command": "npx",
      "args": ["-y", "chrome-devtools-mcp@latest"]
    }
  }
}
```

Requires Node LTS and current-stable Chrome. Everything else works without it.

---

## Quick start

### Audit accessibility

```
/a11y-audit
```

Checks WCAG 2.2 Level AA — including canvas and WebGL, which automated tools cannot see at all.

### Verify a page actually works

```
/verify-ui http://localhost:5173
```

Drives a real browser: renders the page, exercises the flow by keyboard, checks the console at
three viewport sizes, and screenshots the result.

### Shrink 3D assets

```
/optimize-assets
```

Inspects models, reports VRAM (not just file size), compresses, and verifies the output still
loads.

### Or just ask

Skills load automatically:

```
Why is my heading blurry at 200% zoom?
Make this card responsive without media queries
My WebGL scene crashes on mobile
How do I make a canvas game accessible?
```

---

## How it works

**Skills** load automatically when your task matches — you never invoke them by name. Each costs
~2–3k tokens **only when it fires**.

**Agents** are read-only auditors. They report `file:line` with computed values and concrete fixes,
and never edit your files.

**Commands** are workflows you invoke with `/`.

---

## Commands

### `/a11y-audit [path]`

WCAG 2.2 Level AA audit, including canvas accessibility.

```
/a11y-audit                     # whole project
/a11y-audit src/components/     # scoped
```

Greps for candidates, then spawns `a11y-reviewer` to verify each in context. If a dev server is
running, it also takes a live accessibility-tree snapshot — an empty tree is itself a finding.

Reports by severity: seizure risk → keyboard blockers → AA failures → ARIA → structure.

**It tells you what it cannot prove.** Screen reader announcement quality, focus order logic, and
alt-text accuracy all need a human, and the report says so rather than implying conformance.

### `/verify-ui [url]`

Drives a real browser instead of guessing from source.

```
/verify-ui                              # finds your dev server
/verify-ui http://localhost:3000
/verify-ui "check the checkout flow"
```

The loop: navigate → snapshot the accessibility tree → act on elements by `uid` → `wait_for` →
check console and network → screenshot. Then repeats at 375px, 768px, and 1440px.

Uses **bounded verification** — build fully, inspect once batching everything, fix in one batch,
at most one confirming pass, stop. Open-ended self-QA burns tokens for diminishing returns.

Requires `chrome-devtools-mcp`. If it isn't connected, the command says so and stops rather than
substituting guesses.

### `/optimize-assets [path]`

Inspect, compress, and verify GLB/glTF models and textures.

```
/optimize-assets
/optimize-assets public/models/
```

Inventories assets by real size, inspects before compressing (geometry-heavy vs texture-heavy
decides the strategy), **confirms with you before modifying anything**, never overwrites sources,
and verifies the output still renders.

Reports the **VRAM** change, not just file size — the number that actually determines whether a
phone crashes.

---

## Skills

### Accessibility

| Skill | Covers |
|---|---|
| `a11y-core` | WCAG 2.2 AA — the 9 new criteria, contrast, target size, focus, keyboard |
| `a11y-canvas` | Mirror DOM, `drawFocusIfNeeded`, fallback content, why hit regions are dead |
| `a11y-aria` | The four rules of ARIA, landmarks, live regions, common misuse |
| `a11y-testing` | axe-core, Lighthouse, pa11y — and what automation genuinely proves |

### CSS

| Skill | Covers |
|---|---|
| `css-container-queries` | `@container`, `container-type`, `cqi` units, style queries |
| `css-fluid-type` | `clamp()` derivation and the WCAG zoom constraint |
| `css-layout` | `auto-fit` vs `auto-fill`, subgrid, `:has()`, `@layer` |
| `css-viewport` | `dvh`/`svh`/`lvh`, safe areas, the dynamic-unit scroll jump |
| `responsive-images` | `srcset`/`sizes`, `<picture>`, `aspect-ratio`, CLS |
| `design-tokens` | Token layers, OKLCH, theming, contrast as a constraint |

### Performance and verification

| Skill | Covers |
|---|---|
| `web-vitals` | LCP/INP/CLS thresholds, field vs lab, fixes per metric |
| `verify-browser` | The snapshot → act → verify loop with chrome-devtools-mcp |
| `verify-performance` | Traces, insights, throttling, heap snapshots |

### Motion

| Skill | Covers |
|---|---|
| `motion-smooth-scroll` | Lenis setup, options, GSAP wiring, canvas sync |
| `motion-craft` | Durations, easing, cheap properties, reduced motion |

### 3D

| Skill | Covers |
|---|---|
| `three-r3f` | R3F 9 / drei 10 / three 0.186, the `useFrame` rule, disposal |
| `three-assets` | `gltf-transform` and `gltfpack`, Draco vs Meshopt, VRAM maths |
| `three-scroll` | Canvas-owns vs DOM-owns scroll, camera paths, smoothing |

---

## Review agents

Both are read-only. Spawn via the commands, or just ask Claude to review.

### `a11y-reviewer`

WCAG 2.2 AA, checking canvas **first** because nothing else does.

- **Canvas/WebGL** — dead `addHitRegion`, missing fallback content, no keyboard path, invisible
  focus, proxies hidden with `display:none` (which removes them from the tree).
- **WCAG criteria** — with IDs and levels. Computes contrast from actual hex values. Applies the
  **geometric** 2.5.8 spacing test rather than a naive `width >= 24` check that false-positives on
  compliant dense toolbars.
- **ARIA** — roles without keyboard models, stale state, `aria-hidden` on focusables.
- **Structure and motion** — headings, landmarks, labels, reduced-motion paths.

It will **not** report SC 4.1.1 Parsing — that was removed in WCAG 2.2.

> **1.4.3 Contrast (Minimum), AA** — `src/Card.tsx:34`
> `#999999` on `#FFFFFF` is **2.85:1**; 16px text requires 4.5:1.
> Fix: `#595959` → 7.0:1.

### `web-perf-reviewer`

Core Web Vitals risks **and** GPU budgets — the second is where most reviewers stop.

- **GPU/VRAM** — texture dimensions (`w × h × 4 × 1.333`), PNG-as-GPU-texture, WebGL context
  limits (**8 on Android, 16 elsewhere**), uncapped DPR, missing disposal, draw calls.
- **LCP / INP / CLS** — lazy-loaded LCP elements, render-blocking resources, long tasks,
  `setState` in `useFrame`, unreserved image boxes.
- **Leaks** — listeners, RAF loops, tickers, GSAP tweens, StrictMode double-mount.

It **never asserts a metric value it did not measure** — it identifies risks and points you at
`/verify-performance` for the actual trace.

---

## Common workflows

**Shipping a new page**

```
/verify-ui           → does it actually render and work?
/a11y-audit          → is it usable by keyboard and screen reader?
```

**"It's slow on mobile"**

Ask, or run `/verify-performance`. Throttle first (4× CPU, Slow 4G) — an unthrottled desktop
number is misleading. `LCPBreakdown` tells you whether to fix the server, the discovery, the
download, or the render.

**"My WebGL scene crashes on phones"**

Almost always VRAM. Four 4096² textures is **341 MB** regardless of PNG file size, because PNG
decodes to raw RGBA before upload. `/optimize-assets` resizes and converts to KTX2.

**"Text is blurry when I zoom"**

`css-fluid-type`. Usually a `vw`-only `clamp()` with no `rem` term, or a max more than **2.5×** the
min — both fail WCAG 1.4.4.

**Building a canvas game accessibly**

`a11y-canvas` covers the mirror-DOM pattern. The short version: keep every menu, setting, and
dialog in real DOM, and never render UI into the canvas.

---

## FAQ

**Do I need to invoke skills manually?**
No. They match on your task. Commands are the only thing you type.

**Do I need chrome-devtools-mcp?**
Only for `/verify-ui` and `/verify-performance`. The other 15 skills work without it.

**Will the agents edit my files?**
No. Both are read-only by design.

**Does `/a11y-audit` prove my site is accessible?**
No, and it says so. Automated coverage is disputed — Deque's 2021 study found **57.38% of issues**
detectable automatically, while the traditional 20–40% figure counts *machine-testable criteria*.
Both measure different things. Manual keyboard, zoom, and screen reader passes are mandatory.

**Why does it say "the four rules of ARIA"?**
Because W3C's *Using ARIA* document has four numbered rules. "Five rules of ARIA" is a widespread
misquote.

**What's the token cost?**
~2,200 tokens always-on. Each skill costs ~2–3k more, but only when it fires.

---

## Reference generation

`scripts/pull-refs.mjs` regenerates `references/` from openly-licensed sources.

```bash
node scripts/pull-refs.mjs            # regenerate
node scripts/pull-refs.mjs --check    # verify freshness (exit 1 if stale)
```

Currently ingests the **W3C WCAG 2.2 criterion list** (87 criteria, 24 at Level AA) under the W3C
Document License. The script **fails closed** — it writes nothing if the fetch or parse looks
wrong, and refuses to prune a large share of files without `--force-prune`. Running it twice
produces zero git diff.

**Deliberately not ingested:** Apple's Human Interface Guidelines (copyrighted, and the popular
skill repo carries no license file) and MDN prose (CC-BY-SA share-alike, incompatible with bulk
inclusion in an MIT plugin). Short factual values from MDN are quoted in hand-written skills,
which is fine.

---

## Companion plugins

**[pixijs](../PIXIjs)** — building 2D canvas things. This plugin verifies what that one builds.

**General design quality** — deliberately not covered here:

```bash
# Craft rules + a deterministic detector (Apache-2.0, 67k★)
claude plugin marketplace add pbakaus/impeccable
claude plugin install impeccable@impeccable

# Aesthetic direction, avoiding templated defaults (official Anthropic)
claude plugin install frontend-design@claude-plugins-official
```

`design-tokens` and `motion-craft` point at these explicitly rather than duplicating their rules.

**Video input** — for bug-repro recordings and design walkthroughs:

```bash
claude plugin marketplace add bradautomates/claude-video
claude plugin install watch@claude-video
```

Needs `ffmpeg` and `yt-dlp`. A recording shows you *when* something breaks; `/verify-performance`
shows you *why*.

---

## Development

```bash
claude plugin validate . --strict     # validate the manifest
claude plugin details web-craft       # component inventory + token cost
```

Plugin structure:

```
web-craft/
├── .claude-plugin/{plugin.json, marketplace.json}
├── scripts/pull-refs.mjs         # open-licensed doc ingestion
├── skills/<name>/SKILL.md        # + optional reference.md
├── agents/{a11y-reviewer,web-perf-reviewer}.md
└── commands/{a11y-audit,verify-ui,optimize-assets}.md
```

---

## License

MIT. Generated reference content is W3C material under the W3C Document License.
