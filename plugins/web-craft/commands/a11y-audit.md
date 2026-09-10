---
description: Audit a project for WCAG 2.2 AA conformance, including canvas and WebGL accessibility that automated tools cannot detect.
---

Audit this project's accessibility. Argument: `$ARGUMENTS` (optional — a path, file, or URL to
scope the audit to; audit the whole project if empty).

## Steps

1. **Identify the stack** from `package.json` and the source layout: plain HTML/CSS, React, Vue,
   canvas/WebGL, PixiJS, three.js. Report it — it determines which checks apply.

2. **Locate UI source**: components, templates, stylesheets, and any canvas/WebGL rendering code.
   Scope to `$ARGUMENTS` if given.

3. **Run a grep pass** for candidates:

   ```bash
   grep -rn "addHitRegion\|outline: *none\|outline:none\|aria-hidden\|tabindex=\"[1-9]\|user-scalable=no\|maximum-scale=1\|onpaste" src/
   grep -rn "<canvas\|getContext(\|new Application\|<Canvas" src/
   ```

   Candidates only — grep produces false positives, and the most important issues (missing
   keyboard paths, unlabelled controls, stale ARIA state) do not grep at all.

4. **Launch the `a11y-reviewer` agent** with the file list and the identified stack. It verifies
   each candidate in context and covers what grep cannot: canvas semantics, keyboard reachability,
   computed contrast, target-size geometry, and ARIA state correctness.

5. **If a dev server or URL is available**, also run the live checks — these find things static
   analysis cannot:
   - `chrome-devtools-mcp`: `navigate_page` → `take_snapshot`. An empty or `generic`-heavy
     accessibility tree is itself a finding. See the `verify-browser` skill.
   - `lighthouse_audit(mode: "navigation")` for the automated baseline.
   - Resize to 375px and re-snapshot to check the mobile path.

6. **Report findings** grouped by severity:
   - **Seizure risk** — anything flashing more than 3×/second (2.3.1, Level A). Always first.
   - **Blockers** — content unreachable by keyboard, canvas with no accessible representation.
   - **AA failures** — with criterion ID, level, and computed values.
   - **ARIA correctness** — roles without keyboard models, stale state, hidden focusables.
   - **Structure and content** — headings, landmarks, labels, alt text.

7. **State the limits explicitly.** A static audit cannot verify screen reader announcement
   quality, focus order logic, or whether alt text is accurate. Say what still needs a human, and
   never describe the result as proof of conformance.

Do not modify files. This command reports only.
