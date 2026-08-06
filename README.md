# Onelo Claude Code plugins

One marketplace, all Onelo developer tooling for [Claude Code](https://claude.com/claude-code).
Add it once and get every plugin below; updates arrive on `/plugin marketplace update`.

```bash
/plugin marketplace add onelo-tools/claude-plugins
/plugin marketplace update onelo-tools      # already added it before? do this or the install 404s
/plugin install onelo@onelo-tools
/reload-plugins
```

`onelo` is a **bundle**: it ships no skills of its own and declares every plugin
below as a dependency, so one install pulls them all in. Then run
`/onelo-quickstart` — it works out which parts your product needs.

Prefer to pick your own? Install any single one:

```bash
/plugin install onelo-auth@onelo-tools
```

## Plugins

| Plugin | Command | What it does |
|---|---|---|
| **onelo** | — | **Bundle.** Installs every plugin below in one command. |
| **onelo-quickstart** | `/onelo-quickstart` | **Start here.** Explains what Onelo does, works out which parts your product needs, and walks you through them in order. |
| **onelo-auth** | `/onelo-auth` | Wires Onelo's hosted sign-in into your app — detects the platform, inserts the snippet, and gates your UI on the resolved session. |
| **onelo-store** | `/onelo-store` | Sells plans: embeds the hosted store on a website, or sets up the in-app store that Onelo Auth drives in native apps. |
| **onelo-customer-portal** | `/onelo-customer-portal` | Adds a "Manage subscription" entry point — Onelo's hosted portal for cancel, change plan, refunds and invoices. |
| **onelo-waitlist** | `/onelo-waitlist` | Collects pre-launch signups on a page that turns itself into your store at launch. |
| **onelo-features** | `/onelo-features` | Scans your codebase for instrumentable features and inserts Onelo Features SDK calls. |
| **onelo-feedback** | `/onelo-feedback` | Wires in-app bug reports and feature requests into Onelo's hosted form, with the reporter and their active features attached. |
| **onelo-roadmap** | `/onelo-roadmap` | Publishes a public roadmap page and links or embeds it — no SDK code. |
| **onelo-branding** | `/onelo-branding` | Brands every hosted page from one theme, and writes Custom CSS for what the controls can't express. |
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
├── onelo-quickstart/
├── onelo-auth/
│   ├── .claude-plugin/plugin.json
│   └── skills/onelo-auth/
│       ├── SKILL.md
│       └── references/snippets/      #   one file per platform (baked)
├── onelo-store/
├── onelo-customer-portal/
├── onelo-waitlist/
├── onelo-feedback/
├── onelo-roadmap/
├── onelo-branding/
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
