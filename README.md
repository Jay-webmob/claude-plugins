# Claude Code plugins

Frontend and creative-coding plugins for [Claude Code](https://claude.com/claude-code).

| Plugin | What it does |
|---|---|
| **[pixijs](plugins/pixijs)** | Makes Claude write correct PixiJS v8 — 16 skills, 1 review agent, 3 commands |
| **[web-craft](plugins/web-craft)** | Accessibility, modern CSS, browser-verified performance, motion, 3D assets — 17 skills, 2 agents, 3 commands |

---

## Install

```bash
claude plugin marketplace add Jay-webmob/claude-plugins
claude plugin install pixijs@jay-plugins
claude plugin install web-craft@jay-plugins
```

Restart Claude Code afterwards — skills and commands work immediately, but review agents only
register at session start.

> **Note on `npx`:** these plugins are markdown skills, not executables, so there is nothing for
> `npx` to run. They are *distributed* via npm and *installed* with `claude plugin install`.

---

## What they cover

**pixijs** — building 2D canvas things. PixiJS v8 broke a large part of the v7 API, and because v7
dominates training data, agents reliably emit code that fails at runtime — often *silently*.
This plugin makes v8 the default and ships an agent that hunts regressions.

**web-craft** — making them correct, accessible, and fast. Owns the areas general design tools
leave uncovered: canvas/WebGL accessibility (automated tools see `<canvas>` as one empty element),
GPU/VRAM budgets, and browser-verified truth via chrome-devtools-mcp.

They compose: **pixijs builds it → web-craft proves it.**

---

## Development

```bash
npm run validate       # validate the marketplace and both plugin manifests
npm test               # validate + check the WCAG reference is current
npm run publish:dry    # dry-run both npm publishes
```

Install from this working tree instead of npm while iterating:

```bash
claude plugin marketplace add /path/to/claude-plugins
```

### Publishing

Both plugins are published to npm and referenced from the marketplace by package name.

```bash
npm run publish:dry     # inspect what would be published
npm run publish:all     # publish both, public access
```

Bump the version in three places, keeping them in sync:

1. `plugins/<name>/package.json` — the npm package version
2. `plugins/<name>/.claude-plugin/plugin.json` — the plugin manifest version
3. `.claude-plugin/marketplace.json` — the `version` field and the `source.version` range

Then tag:

```bash
git tag pixijs--v0.2.0 && git push --tags
```

`claude plugin tag` validates that the manifest and marketplace entry agree before tagging.

---

## Layout

```
claude-plugins/
├── .claude-plugin/marketplace.json    # npm-sourced marketplace
├── plugins/
│   ├── pixijs/
│   │   ├── .claude-plugin/plugin.json
│   │   ├── skills/  agents/  commands/
│   │   └── package.json               # published as pixijs-claude-plugin
│   └── web-craft/
│       ├── .claude-plugin/plugin.json
│       ├── skills/  agents/  commands/
│       ├── scripts/pull-refs.mjs      # W3C reference ingestion
│       └── package.json               # published as web-craft-claude-plugin
└── package.json                       # npm workspaces root
```

---

## Companions

Deliberately not covered here — install alongside:

```bash
# General design craft + a 61-rule detector (Apache-2.0)
claude plugin marketplace add pbakaus/impeccable
claude plugin install impeccable@impeccable

# Aesthetic direction (official Anthropic)
claude plugin install frontend-design@claude-plugins-official

# Video input for bug repros and design walkthroughs
claude plugin marketplace add bradautomates/claude-video
claude plugin install watch@claude-video
```

---

## License

MIT
