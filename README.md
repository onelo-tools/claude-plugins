# Onelo Claude Code plugins

One marketplace, all Onelo developer tooling for [Claude Code](https://claude.com/claude-code).
Add it once and get every plugin below; updates arrive on `/plugin marketplace update`.

```bash
/plugin marketplace add onelo-tools/claude-plugins
/plugin install onelo-auth@onelo-tools
/plugin install onelo-features@onelo-tools
/plugin install onelo-monitor@onelo-tools
```

## Plugins

| Plugin | Command | What it does |
|---|---|---|
| **onelo-auth** | `/onelo-auth` | Wires Onelo's hosted sign-in into your app — detects the platform, inserts the snippet, and gates your UI on the resolved session. |
| **onelo-features** | `/onelo-features` | Scans your codebase for instrumentable features and inserts Onelo Features SDK calls. |
| **onelo-monitor** | `/onelo-monitor` | Audits existing Onelo Monitor usage for anti-patterns and instruments new operations. |

## The code these plugins insert

Every Onelo snippet lives in exactly one place — the `@onelo/snippets` package in
the Onelo monorepo — and the dashboard **SDK** tab, the public **/docs** pages and
these plugins all render from it.

Plugins do **not** fetch snippets at runtime. They are **baked in at publish
time**: each skill ships one file per platform under
`skills/<name>/references/snippets/`, holding the real code with the production
install refs and API URL already substituted. Every baked block is stamped with
the `@onelo/snippets` version it came from.

So a published skill is closed: no network call, no stale fallback copy, and no
chance of handing you a snippet that happened to be mid-rework the moment you
ran it.

## Channels

- **This repo (`onelo-tools/claude-plugins`, branch `main`)** — production, and
  what `/plugin marketplace add` installs. Updated only by an explicit manual
  release.
- **`onelo-tools/claude-plugins-staging` (private, branch `staging`)** — internal
  testing. Every push to the monorepo touching the plugins or the snippets syncs
  there automatically. Nothing there is visible to users.

A release promotes the tested private artifact here verbatim, so what you install
is byte-for-byte what was validated in staging.

## Layout

```
claude-plugins/
├── .claude-plugin/marketplace.json   # lists every plugin (relative ./ sources)
├── onelo-auth/
│   ├── .claude-plugin/plugin.json
│   └── skills/onelo-auth/
│       ├── SKILL.md
│       └── references/snippets/      #   one file per platform (baked)
├── onelo-features/                   # one plugin
│   ├── .claude-plugin/plugin.json    #   name + pinned version
│   └── skills/onelo-features/
│       ├── SKILL.md                  #   instructions + platform table
│       └── references/
│           ├── snippets/             #   one file per platform (baked)
│           ├── classification-rules.md
│           └── <lang>-patterns.md    #   per-language guidance
└── onelo-monitor/
```

Each plugin keeps its own `plugin.json`; the single marketplace at the repo root
is what users add.
