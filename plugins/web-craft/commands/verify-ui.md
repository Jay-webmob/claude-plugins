---
description: Verify a running page in a real browser — render, interaction, responsive breakpoints, console errors, and screenshots.
---

Verify the UI actually works in a browser rather than inferring it from the code. Argument:
`$ARGUMENTS` (optional — a URL, or a description of the flow to verify).

Requires `chrome-devtools-mcp`. If it is not connected, say so and stop — do not substitute
guesses for verification. Setup is in the `verify-browser` skill:

```json
{ "mcpServers": { "chrome-devtools": { "command": "npx", "args": ["-y", "chrome-devtools-mcp@latest"] } } }
```

## Steps

1. **Get a URL.** Use `$ARGUMENTS` if given. Otherwise look for a dev server script in
   `package.json` and start it, or ask. Wait until it actually responds before navigating.

2. **Navigate and snapshot.**
   - `navigate_page(url)`
   - `take_snapshot()` — the accessibility tree, with a `uid` per element. Cheaper than a
     screenshot and the basis for every action.

3. **Exercise the flow.** Use uids, not CSS selectors: `click`, `fill`, `fill_form`, `hover`,
   `press_key`. After each meaningful step, `wait_for` expected text rather than sleeping, then
   re-snapshot to confirm state changed as intended.

   Include keyboard paths: `press_key("Tab")` to check focus order and `press_key("Escape")` to
   check dismissal and focus return.

4. **Check the console and network.**
   - `list_console_messages(types: ["error", "warning"], includeStackTraces: true)` — source-mapped
     stacks give real file and line.
   - `list_network_requests(resourceTypes: ["fetch", "xhr"])` — look for 4xx/5xx the UI swallows.

   A page can look correct and still throw on every frame. Always check.

5. **Verify responsive behaviour at real sizes:**
   ```
   resize_page(375, 667)   → snapshot + screenshot
   resize_page(768, 1024)  → snapshot + screenshot
   resize_page(1440, 900)  → snapshot + screenshot
   ```
   Confirm no horizontal overflow, that touch targets stay ≥24px, and that mobile navigation works.

6. **Screenshot for visual proof** — `take_screenshot(fullPage: true)`, or pass a `uid` for a
   single component.

7. **Report** what you verified, what broke, and — explicitly — what you could not check.

## Bounded verification

Verify in **bounded passes, not a loop**: build fully → inspect once, batching all viewports and
checks → fix everything in one batch → at most one confirming pass → stop.

Open-ended self-QA burns time and tokens for diminishing returns. If a second pass still finds
problems, the approach needs rethinking, not more polish.

## Reporting

Report only what you observed. "Console clean at 375/768/1440; the mobile menu opens, traps focus
correctly, and closes on Escape" is a verification. "Looks good" is not.

If something could not be tested — a flow behind auth, a payment step — say so plainly rather than
implying full coverage.
