# PixiJS plugin for Claude Code

Makes Claude write correct [PixiJS v8](https://pixijs.com) — for animated websites and HTML5
games.

**16 skills · 1 review agent · 3 commands** · ~2,000 tokens always-on

---

## Table of contents

- [Why this exists](#why-this-exists)
- [Install](#install)
- [Quick start](#quick-start)
- [How it works](#how-it-works)
- [Commands](#commands)
- [Skills](#skills)
- [The review agent](#the-review-agent)
- [Common workflows](#common-workflows)
- [FAQ](#faq)
- [Companion plugins](#companion-plugins)
- [Development](#development)

---

## Why this exists

PixiJS v8 broke a large part of the v7 public API. Because v7 dominates the training data,
tutorials, and Stack Overflow answers that LLMs learned from, coding agents reliably emit v7 code
that looks plausible and **fails at runtime**:

```js
// What models tend to write — all four lines are broken on v8
const app = new Application({ width: 800 });
document.body.appendChild(app.view);
g.beginFill(0xff0000).drawRect(0, 0, 50, 50);
const tex = Texture.from('bunny.png');
```

```js
// What v8 actually requires
const app = new Application();
await app.init({ width: 800 });
document.body.appendChild(app.canvas);
g.rect(0, 0, 50, 50).fill(0xff0000);
const tex = await Assets.load('bunny.png');
```

Worse, several of these fail **silently**. `Texture.from()` on an unloaded URL renders a blank
sprite instead of throwing, so the mistake survives local development and breaks in production.

This plugin loads the v8 API as working knowledge, supplies patterns for real work, and ships an
agent that hunts the regressions down.

---

## Install

```bash
claude plugin marketplace add /path/to/PIXIjs
claude plugin install pixijs@pixijs
```

Verify it loaded:

```bash
claude plugin details pixijs
```

You should see 19 skills (16 knowledge skills + 3 commands), 1 agent.

> **Restart Claude Code after installing.** Skills and commands work immediately, but the
> `pixi-reviewer` agent only registers at session start.

---

## Quick start

### Start a new project

```
/pixi-init
```

Claude asks which template you want, then scaffolds it, runs `npm install`, and **verifies the
build passes** before reporting success:

| Template | You get |
|---|---|
| `vanilla` | TypeScript + Vite, plain Pixi canvas |
| `react` | React 19 + `@pixi/react` v8 with StrictMode-safe mounting |
| `game` | Scene manager, fixed-timestep loop, keyboard input, camera |

Skip the questions by passing arguments:

```
/pixi-init game ./my-game
```

### Or just ask

You don't need a command. The skills load automatically:

```
Build me a Pixi scene with a sprite that follows the mouse
Why is my Pixi text blurry?
Add a parallax background to my game
```

### Check existing code

```
/pixi-audit
```

Finds v7 API that breaks on v8, memory leaks, and performance problems.

---

## How it works

Three kinds of capability, and they behave differently:

**Skills** load automatically when your task matches. Ask about sprites and `pixi-animation`
loads; ask about clicks and `pixi-interaction` loads. You never invoke them by name. Each costs
~2–3k tokens **only when it fires** — the always-on cost is just the one-line descriptions.

**The agent** is spawned to audit code. It's read-only: it reports findings with `file:line` and
concrete fixes, and never edits your files.

**Commands** are workflows you invoke with `/`.

---

## Commands

### `/pixi-init [template] [directory]`

Scaffolds a working v8 project and proves it builds.

```
/pixi-init                      # asks which template and where
/pixi-init react                # React template, asks for directory
/pixi-init game ./my-game       # both specified
```

Every generated project uses `await app.init()`, `app.canvas`, `Assets.load()` before any sprite,
shape-then-style Graphics, `deltaTime`-scaled animation, capped resolution, and proper teardown.

If `npm run build` fails, Claude fixes it before telling you it's done.

### `/pixi-audit [path]`

Runs `pixi-reviewer` across your code.

```
/pixi-audit                     # whole project
/pixi-audit src/scenes/         # just one directory
```

Reports findings grouped by severity — runtime breakage first, then lifecycle, leaks,
performance, correctness. Read-only; it won't change anything.

### `/pixi-migrate [path]`

Drives a v7 → v8 upgrade in dependency order.

```
/pixi-migrate
```

It checks your git tree is clean first, counts affected sites so you know the scale, then works
through nine ordered steps — packages, init, assets, Graphics, renames, events, text, filters,
particles — **committing after each** so you can bisect a regression.

Ends by running `/pixi-audit` to catch anything missed, and tells you explicitly what it couldn't
migrate automatically (custom shaders usually need judgment).

---

## Skills

You never call these directly. They load when relevant.

### Core

| Skill | Covers |
|---|---|
| `pixi-v8-core` | Async `Application.init()`, containers, ticker, teardown, full v7→v8 change table |
| `pixi-graphics` | Shape-then-style drawing, paths, gradients, holes, `GraphicsContext` reuse |
| `pixi-assets` | `Assets.load`, bundles, manifests, background loading, fonts, unloading |
| `pixi-animation` | `deltaTime`, `AnimatedSprite`, spritesheets, tweening, easing, GSAP |
| `pixi-interaction` | `eventMode`, hit areas, dragging, buttons, touch |
| `pixi-text` | `Text` vs `BitmapText` vs `HTMLText`, styles, web fonts, resolution |
| `pixi-performance` | Batching, atlases, culling, render groups, `cacheAsTexture`, particles, pooling |
| `pixi-filters` | Built-ins, `pixi-filters`, custom `GlProgram` shaders, blend modes |

### Games

| Skill | Covers |
|---|---|
| `pixi-game-loop` | Fixed timestep, scene manager, state machines, pause, layers |
| `pixi-collision` | AABB, circles, resolution, spatial hashing, tilemaps, physics engines |
| `pixi-camera` | Follow, deadzone, zoom, clamping, parallax, screen shake, pixel snapping |
| `pixi-input` | Keyboard polling, gamepad, virtual joystick, buffering, coyote time |

### Web

| Skill | Covers |
|---|---|
| `pixi-react` | `@pixi/react` v8 `extend()`, manual mounting, StrictMode, Next.js SSR |
| `pixi-scroll` | Scroll-linked animation, GSAP ScrollTrigger, sequences, reduced motion |
| `pixi-responsive` | `resizeTo`, retina, letterbox vs fill, safe areas, orientation |
| `pixi-migration` | Ordered v7→v8 upgrade procedure with verification |

---

## The review agent

**`pixi-reviewer`** — read-only. Spawn it via `/pixi-audit`, or ask Claude to "review my Pixi code".

It checks five categories in severity order:

1. **v7 API on v8** — the full breakage table. Highest severity because these fail at runtime, and
   several fail *silently*.
2. **Async and lifecycle** — unawaited `init()`, sprites built before assets resolve, React
   StrictMode double-mount, Next.js SSR crashes.
3. **Leaks** — missing `destroy()`, orphaned ticker callbacks, unremoved listeners, unkilled GSAP
   tweens. The dominant bug class in SPAs.
4. **Performance** — per-frame `Graphics` rebuilds, per-frame `Text` updates, batch-breaking blend
   modes, per-object filters.
5. **Correctness** — motion not scaled by `deltaTime`, missing `pointerupoutside`, frame-rate-
   dependent lerps.

Findings look like this, not like vague advice:

> `main.js:88` rebuilds a 400-point Graphics every frame inside `ticker.add`; draw it once outside
> the loop and set `.position` instead.

---

## Common workflows

**Starting fresh**

```
/pixi-init game ./my-game
→ then just describe what you want built
```

**Inheriting a Pixi codebase**

```
/pixi-audit
→ read the findings, fix the Category 1 items first (those are runtime breakage)
```

**Upgrading from v7**

```
/pixi-migrate
→ commits per step; bisect if something regresses
```

**Debugging "my sprites are invisible"**

Just ask. This is almost always `Texture.from()` without `Assets.load()` — `pixi-assets` loads and
explains the silent-failure case.

**Debugging "it drops frames"**

Ask, or run `/pixi-audit`. `pixi-performance` covers the CPU-vs-GPU diagnosis: halve
`renderer.resolution`; if the frame rate recovers you're GPU-bound, if nothing changes you're
CPU-bound. Those need opposite fixes.

---

## FAQ

**Do I need to invoke skills manually?**
No. They match on your task. Commands are the only thing you type.

**Does this work on PixiJS v7?**
The agent detects your version from `package.json` and won't report v7 API as an error in a v7
project. The skills teach v8 — use `/pixi-migrate` to upgrade.

**Will the agent edit my files?**
No. `pixi-reviewer` is read-only by design. Ask Claude to apply the fixes afterwards if you want
them.

**What's the token cost?**
~2,000 tokens always-on (just the skill descriptions). Each skill costs ~2–3k more, but only when
it actually fires.

**Why isn't the agent available?**
Plugin agents register at session start. Restart Claude Code after installing.

**Does it cover WebGPU?**
Yes — `preference: 'webgpu'` is covered in `pixi-v8-core`. Note that custom filters need a
`gpuProgram` (WGSL) alongside `glProgram`, or you must force the WebGL backend.

---

## Companion plugins

This plugin builds canvas things. These make them correct, accessible, and fast:

**[web-craft](../web-craft)** — the natural companion. Canvas/WebGL accessibility (`a11y-canvas`),
browser verification via chrome-devtools-mcp (`/verify-ui`), Core Web Vitals, modern CSS, and 3D
asset optimization. Cross-references these skills rather than repeating them.

**General design quality** — deliberately not covered here:

```bash
# Craft rules + a deterministic detector (Apache-2.0, 67k★)
claude plugin marketplace add pbakaus/impeccable
claude plugin install impeccable@impeccable

# Aesthetic direction, avoiding templated defaults (official Anthropic)
claude plugin install frontend-design@claude-plugins-official
```

**Video input** — for bug-repro recordings and design walkthroughs:

```bash
claude plugin marketplace add bradautomates/claude-video
claude plugin install watch@claude-video
```

Needs `ffmpeg` and `yt-dlp`. Note a recording shows you *when* something breaks; a profiler shows
you *why*.

---

## Development

```bash
claude plugin validate . --strict     # validate the manifest
claude plugin details pixijs          # component inventory + token cost
```

Plugin structure:

```
PIXIjs/
├── .claude-plugin/{plugin.json, marketplace.json}
├── skills/<name>/SKILL.md        # + optional reference.md
├── agents/pixi-reviewer.md
└── commands/{pixi-init,pixi-audit,pixi-migrate}.md
```

Every code sample in every skill is verified against real `pixi.js@8` — a scratch project
exercising the full documented API surface typechecks under `strict` with zero errors.

---

## License

MIT
