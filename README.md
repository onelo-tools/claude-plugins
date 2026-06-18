# Onelo Claude Code plugins

One marketplace, all Onelo developer tooling for [Claude Code](https://claude.com/claude-code).
Add it once and get every plugin below; updates arrive on `/plugin marketplace update`.

```bash
/plugin marketplace add onelo-tools/claude-plugins
/plugin install onelo-features@onelo-tools
/plugin install onelo-monitor@onelo-tools
```

## Plugins

| Plugin | Command | What it does |
|---|---|---|
| **onelo-features** | `/onelo-features` | Scans your codebase and inserts Onelo Features SDK calls. |
| **onelo-monitor** | `/onelo-monitor` | Audits existing Onelo Monitor usage for anti-patterns and instruments new operations (Swift, Python). |

## Channels

- **`staging`** branch — auto-refresh. `marketplace.json` omits per-plugin
  `version`, so Claude Code uses the git commit SHA as the cache key; every push
  is delivered on the next `/plugin marketplace update`.
- **`main`** branch — production; pins explicit versions for stability.

## Layout

```
claude-plugins/
├── .claude-plugin/marketplace.json   # lists every plugin (relative ./ sources)
├── onelo-features/                   # one plugin
│   ├── .claude-plugin/plugin.json
│   └── skills/onelo-features/...
└── onelo-monitor/
    ├── .claude-plugin/plugin.json
    └── skills/onelo-monitor/...
```

Each plugin keeps its own `plugin.json`; the single marketplace at the repo root
is what users add.
