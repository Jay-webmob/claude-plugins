---
name: verify-performance
description: Recording and analyzing performance traces with chrome-devtools-mcp — start_trace, insight analysis, CPU and network throttling, and heap snapshots for memory leaks. Use when diagnosing a slow page, a Core Web Vitals regression, jank, or a memory leak.
---

# Performance Traces

Measure before optimizing. A trace tells you which of the four LCP sub-parts dominates, or which
task blocks interaction — guessing usually optimises the wrong thing.

## The three trace tools

```
performance_start_trace(pageId, autoStop?, reload?, filePath?)
performance_stop_trace(pageId, filePath?)
performance_analyze_insight(insightName, insightSetId, pageId)
```

Standard load trace:

```
navigate_page("http://localhost:5173")            ← navigate FIRST
performance_start_trace(reload: true, autoStop: true)
```

**Two gotchas:**

1. **Navigate to the right URL before starting a trace** when using `reload` or `autoStop`.
   Otherwise you profile the wrong page.
2. **Only one trace can be active at a time** — Chrome allows a single tracing session per page.
   Stop before starting another.

The result summarises Core Web Vitals observations plus **"Available insight sets"** with ids like
`NAVIGATION_0` (one trace can span several navigations).

## Insights

```
performance_analyze_insight(insightName: "LCPBreakdown", insightSetId: "NAVIGATION_0", pageId)
```

Documented `insightName` values:

| Insight | Answers |
|---|---|
| `LCPBreakdown` | Which of the four LCP sub-parts dominates |
| `LCPDiscovery` | Was the LCP resource discovered late? |
| `RenderBlocking` | Which CSS/JS blocks first paint |
| `DocumentLatency` | Server response and redirect cost |
| `ThirdParties` | What third-party code costs |

`LCPBreakdown` first, always — it tells you whether to fix the server (TTFB), the discovery
(preload), the download (format/size), or the render (blocking resources). See `web-vitals` for
what each sub-part means.

## Lab vs field

The trace summary reports **lab** values from that recording — your machine, your network. It may
separately enrich with **CrUX p75 field** data from real users.

**These are different numbers and must not be conflated.** Read the labels; report which you're
quoting. Disable field enrichment with `--no-performance-crux` for deterministic offline runs.

## Throttle first

An untested-on-slow-hardware performance claim is worthless.

```
emulate(cpuThrottlingRate: 4, networkConditions: "Slow 4G")
```

4–6× CPU throttling approximates a mid-tier Android device. Most performance problems are invisible
at 1× on a developer laptop.

```
emulate(cpuThrottlingRate: 4, networkConditions: "Slow 4G", viewport: { width: 375, height: 667 })
performance_start_trace(reload: true, autoStop: true)
```

That's the configuration that matches how the p75 of your users actually experience the page.

## Interaction performance

INP cannot be measured by a load trace — it needs real interactions. Trace manually around them:

```
performance_start_trace(autoStop: false)
click(uid_of_button)
wait_for("Results")
performance_stop_trace()
```

Then look for long tasks between the input and the next paint. See `web-vitals` for the yielding
and paint-first patterns that fix them.

## Memory leaks

Thirteen heap tools. The core comparison workflow:

```
take_heapsnapshot()                    ← baseline
… perform the suspect cycle several times (open/close, navigate away and back) …
take_heapsnapshot()                    ← after
compare_heapsnapshots()                ← what grew?
```

Then drill in:

| Tool | Use |
|---|---|
| `get_heapsnapshot_class_nodes` | Which classes have growing instance counts |
| `get_heapsnapshot_retainers` | What holds a specific object alive |
| `get_heapsnapshot_retaining_paths` | The full chain from a GC root |
| `get_heapsnapshot_dominators` | Which objects account for the most retained size |
| `get_heapsnapshot_duplicate_strings` | Wasted memory from repeated strings |

Repeat the cycle several times before the second snapshot — a single pass is lost in noise.

The usual culprits: event listeners never removed, timers/RAF loops never cancelled, detached DOM
nodes still referenced, and — in canvas/WebGL work — GPU resources never destroyed. For those, see
`pixi-performance` and `three-assets`; a JS heap snapshot does **not** show GPU memory.

## Reporting

Give the measurement, the condition it was taken under, and the cause:

> LCP 4.1s (lab, 4× CPU throttle, Slow 4G, 375×667). `LCPBreakdown` attributes 2.6s to resource
> load delay — the hero is a CSS background, so the preload scanner cannot discover it.
> Fix: move it to `<img fetchpriority="high">`, or add `<link rel="preload" as="image">`.

Not "the page is slow, consider optimizing images".

Always state the throttling. An unthrottled desktop number reported as a result is misleading even
when technically accurate.

## Related skills

| Need | Skill |
|---|---|
| Metric definitions and thresholds | `web-vitals` |
| Driving and inspecting the page | `verify-browser` |
| GPU/VRAM budgets | `three-assets` |
| Canvas render performance | `pixi-performance` |
