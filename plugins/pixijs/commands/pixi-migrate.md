---
description: Migrate a PixiJS v7 codebase to v8, following the ordered upgrade procedure with verification at each step.
---

Migrate this project from PixiJS v7 (or v6) to v8. Argument: `$ARGUMENTS` (optional — a path to
scope the migration to).

Read the `pixi-migration` skill first and follow its ordered procedure. The order matters: doing
Graphics before imports compile produces a wall of unrelated errors.

## Steps

1. **Assess before changing anything.**
   - Confirm the current version in `package.json`.
   - Confirm the repo is a git repo with a clean working tree. If it is not clean, stop and tell
     the user — this migration touches many files and they need a way back.
   - Count the affected sites so the user knows the scale:

     ```bash
     grep -rc "beginFill\|drawRect\|app\.view\|BaseTexture\|DisplayObject\|\.interactive *=" src/
     ```

   - Report the assessment and confirm the user wants to proceed before editing.

2. **Migrate in this order, committing after each step** so a regression can be bisected:

   1. **Packages** — replace `@pixi/*` with `pixi.js@^8`; bump `@pixi/react`, `pixi-filters`,
      `@pixi/sound` to matching majors. Consolidate imports.
   2. **Application init** — sync constructor to `await app.init()`; `app.view` to `app.canvas`.
      This makes entry points async; wrap them.
   3. **Assets** — audit every `Texture.from`. Trace whether the asset was preloaded; add
      `Assets.load` where not. Convert `Assets.add(alias, src)` to the options-object form.
      Replace `BaseTexture` with the right `TextureSource` subtype.
   4. **Graphics** — reorder style-then-draw into draw-then-style. This cannot be done by blind
      find-and-replace; each block needs reading. This is usually the bulk of the work.
   5. **Scene graph renames** — `DisplayObject`→`Container`, `.name`→`.label`,
      `NineSlicePlane`→`NineSliceSprite`, the `Simple*`→`Mesh*` meshes. Check each `.name` hit
      individually; plain objects legitimately have `name`.
   6. **Events** — `interactive`/`buttonMode` to `eventMode`/`cursor`.
   7. **Text** — options-object constructor; `stroke` and `dropShadow` as objects.
   8. **Filters** — custom filters to `GlProgram` + typed uniforms, shaders to GLSL 300 es.
   9. **ParticleContainer** — sprites to `Particle` objects, add `boundsArea`.

3. **Verify.** Run the build. Then check the runtime list from the `pixi-migration` skill —
   several v8 breakages fail silently rather than at build time, so a green build is not
   sufficient:

   - Canvas renders at all (catches unawaited `init()`)
   - **No blank/white sprites** (catches un-preloaded `Texture.from`)
   - Fills *and* strokes correct
   - Interaction works
   - Filters compile — check the console for GLSL errors

4. **Run `/pixi-audit`** at the end to catch anything missed.

5. **Report** what changed, how many files, what the build did, and — explicitly — anything you
   could not migrate automatically and left for the user. Custom shaders and ParticleContainer
   rewrites often need judgment; say so rather than silently leaving them broken.
