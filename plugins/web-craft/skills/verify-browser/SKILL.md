---
name: verify-browser
description: Verifying UI in a real browser with chrome-devtools-mcp — the snapshot/act/verify loop, screenshots, console and network inspection, and viewport emulation. Use when you need to confirm a page actually renders, works, and has no console errors rather than assuming from the code.
---

# Verifying UI in a Real Browser

Reading code proves intent. Driving the page proves behaviour. When a change is visual or
interactive, verify it.

`chrome-devtools-mcp` is the official tool for this: **57 tools across 12 categories**. (Blog posts
citing 26, 29, or 44 tools are stale — the count grows by version.)

## Setup

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

Requires Node LTS and current-stable Chrome (or Chrome for Testing). Useful flags:

| Flag | Effect |
|---|---|
| `--slim` | Fewer tools — cuts tool-definition token overhead |
| `--headless` | No visible window |
| `--isolated` | Fresh profile per run |
| `--no-performance-crux` | Skip CrUX field-data enrichment (offline/deterministic runs) |

Running both `chrome-devtools-mcp` and `playwright-mcp` permanently doubles tool-definition tokens.
Pick per task.

## The verification loop

```
navigate_page → take_snapshot → act by uid → wait_for → console/network → take_screenshot
```

1. **`navigate_page`** to the URL.
2. **`take_snapshot`** returns the **accessibility tree** as text, with a `uid` per element. This
   is the key idea: actions target semantic uids, not brittle CSS selectors — and it's far cheaper
   in tokens than a screenshot.
3. **Act** — `click`, `fill`, `fill_form`, `hover`, `press_key`, `type_text`, `drag`,
   `upload_file`, `click_at` — using those uids. Many accept `includeSnapshot` to return fresh
   state in the same round trip.
4. **`wait_for`** text to confirm the result, rather than sleeping.
5. **`list_console_messages`** and **`list_network_requests`** to catch errors the UI hides.
6. **`take_screenshot`** for visual proof.

## Snapshot vs screenshot

| | `take_snapshot` | `take_screenshot` |
|---|---|---|
| Returns | Accessibility tree (text) | Pixels |
| Cost | Cheap | Expensive |
| Use for | Finding and acting on elements, checking semantics | Layout, spacing, colour, visual regressions |

Default to snapshots. Screenshot when the question is genuinely visual.

A snapshot doubles as a free accessibility smoke test: if the tree is empty, unlabelled, or shows
`generic` where a button should be, that's a real finding — see `a11y-canvas` for why a canvas
snapshot is empty by design.

`take_screenshot` accepts `uid` to capture a single element, plus `fullPage`, `format`, `quality`,
and `filePath`.

## Console and network

```
list_console_messages(pageId, types, includeStackTraces, includePreservedMessages, pageIdx, pageSize)
get_console_message(msgid, pageId)
list_network_requests(pageId, resourceTypes, includePreservedRequests, pageIdx, pageSize)
get_network_request(pageId, reqid, requestFilePath, responseFilePath)
```

`includeStackTraces` gives **source-mapped** stacks — the difference between "error in bundle.js"
and a real file and line. `includePreservedMessages` survives navigation, so you don't lose errors
thrown during a redirect.

**Always check the console after a change.** A page can look perfect and still be throwing on every
frame.

## Emulation

```
emulate(pageId, viewport, cpuThrottlingRate, networkConditions, colorScheme, geolocation, userAgent, extraHttpHeaders)
resize_page(pageId, width, height)
```

Verify responsive work at real sizes rather than trusting the CSS:

```
resize_page(width: 375, height: 667)    → snapshot/screenshot
resize_page(width: 768, height: 1024)   → snapshot/screenshot
resize_page(width: 1440, height: 900)   → snapshot/screenshot
```

`colorScheme` tests dark mode. `cpuThrottlingRate` (4–6× is a reasonable mid-tier mobile proxy) and
`networkConditions` matter before any performance claim — a fast desktop hides the regressions
users actually hit.

## Lighthouse

```
lighthouse_audit(pageId, device, mode, outputDirPath)
```

Covers **accessibility, SEO, best practices, and agentic browsing** — and **excludes performance**.
For performance use `performance_start_trace` (see `verify-performance`).

`mode`: `navigation` (reload, then audit) or `snapshot` (audit current state — use after logging in
or opening a modal).

Remember what a 100 accessibility score means: no *automated* failures. See `a11y-testing`.

## Playwright MCP vs chrome-devtools MCP

| | chrome-devtools-mcp | playwright-mcp |
|---|---|---|
| Best at | **Debugging** — why did this happen | **Driving** — did this happen |
| Strengths | Performance traces, CWV, Lighthouse, heap snapshots, source-mapped console, CORS/network root cause | Cross-browser, robust automation, test generation |
| Browsers | Chrome only | Chromium, Firefox, WebKit |

Use chrome-devtools-mcp to diagnose from the browser's perspective; Playwright to assert from the
user's perspective across browsers.

## Bounded verification

Verify in **bounded passes, not a loop**:

1. Build the change fully.
2. Inspect **once**, batching everything — all viewports, console, network, screenshots.
3. Fix everything found in one batch.
4. At most **one** confirming pass.
5. Stop.

Open-ended self-QA burns the user's time and tokens for diminishing returns. Two passes is
plenty; if the second still finds problems, the issue is the approach, not the polish.

## A worked check

Verifying a responsive nav with a mobile menu:

```
navigate_page("http://localhost:5173")
resize_page(375, 667)
take_snapshot()                          → is the toggle exposed as a button? aria-expanded="false"?
click(uid_of_toggle)
take_snapshot()                          → aria-expanded="true"? are links reachable?
press_key("Escape")
take_snapshot()                          → closed, and focus returned to the toggle?
list_console_messages(types: ["error"])  → clean?
resize_page(1440, 900)
take_snapshot()                          → nav inline, toggle hidden?
take_screenshot(fullPage: true)          → visual proof
```

That sequence catches broken ARIA state, missing Escape handling, lost focus, and console errors —
none of which are visible from reading the CSS.

## Related skills

| Need | Skill |
|---|---|
| Traces, insights, CWV | `verify-performance` |
| Metric thresholds | `web-vitals` |
| Accessibility criteria | `a11y-core` |
| Why a canvas snapshot is empty | `a11y-canvas` |
