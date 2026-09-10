---
description: Audit PixiJS code for v7 API usage, leaks, and performance problems using the pixi-reviewer agent.
---

Audit this project's PixiJS code. Argument: `$ARGUMENTS` (optional — a path, file, or area to
scope the audit to; audit the whole project if empty).

## Steps

1. **Confirm the Pixi version** from `package.json`. Report it. If the project is on v7 and not
   mid-migration, tell the user that v7 API usage is expected and ask whether they want a v8
   migration assessment instead (`/pixi-migrate`).

2. **Find the Pixi source files** — anything importing from `pixi.js`, `@pixi/*`, or
   `@pixi/react`. Scope to `$ARGUMENTS` if provided.

3. **Run a fast grep pass** to locate candidate sites:

   ```bash
   grep -rn "app\.view\|BaseTexture\|DisplayObject\|beginFill\|endFill\|lineStyle\|drawRect\|drawCircle\|drawRoundedRect\|drawPolygon\|GraphicsGeometry\|NineSlicePlane\|SimpleMesh\|SimpleRope\|\.interactive *=\|buttonMode" src/
   ```

   Treat these as candidates only. Grep produces false positives — `drawRect` on a 2D canvas
   context and `.name =` on a plain object are not Pixi bugs.

4. **Launch the `pixi-reviewer` agent** to do the real audit, passing the file list and the
   confirmed Pixi version. The agent verifies each candidate by reading surrounding code and
   checks categories grep cannot see: unawaited `init()`, missing teardown, ticker callbacks never
   removed, per-frame `Graphics` rebuilds, batch-breaking blend modes, and frame-rate-dependent
   motion.

5. **Report the findings** grouped by severity:

   - **Breaks on v8** — v7 API that fails at runtime. Includes silent failures such as
     `Texture.from` on an unloaded URL, which renders blank rather than throwing.
   - **Lifecycle** — unawaited async init, React StrictMode double-mount, SSR crashes.
   - **Leaks** — undestroyed apps/textures/Text, orphaned ticker callbacks and window listeners,
     unkilled GSAP tweens.
   - **Performance** — per-frame rebuilds, batch breaks, per-object filters, uncapped resolution.
   - **Correctness** — missing `deltaTime` scaling, missing `pointerupoutside`, frame-rate-
     dependent lerps.

   Each finding gets `file:line`, what goes wrong at runtime, and the concrete fix.

6. **Summarize honestly.** State how many files were checked and what was clean. If nothing is
   wrong, say so plainly — do not invent findings to fill a report.

Do not modify any files. This command reports only. If the user wants the fixes applied, they can
ask afterwards.
