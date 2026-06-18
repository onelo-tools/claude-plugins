---
name: onelo-features
description: Onelo Features instrumentation plugin for Claude Code. Scans your codebase and inserts onelo-features SDK calls automatically.
channel: staging
author: onelo-tools
---

> **Channel:** This is the **staging** branch — auto-refresh channel. The
> marketplace entry intentionally omits `version`, so Claude Code falls back
> to the git commit SHA as the cache key. Every push to `staging` invalidates
> the user's cache and is delivered on the next `/plugin update` — no manual
> version bump required.
>
> The `main` branch (production) pins an explicit `version` for stability,
> enforced by `.github/workflows/plugin-version-guard.yml` in the monorepo.

# onelo-features Plugin

Instruments your codebase with the [Onelo](https://onelo.tools) feature-flag SDK
for your language. The plugin works against the unified Onelo SDKs:

| Language | Package |
|---|---|
| JavaScript / TypeScript | `@onelo/js` |
| Swift (iOS / macOS) | `onelo-swift` (Swift Package Manager) |
| Kotlin (Android) | `onelo-android` (JitPack) |
| Flutter | `onelo` (pub.dev) |
| Python (FastAPI / Flask / Django) | direct REST integration |
| Go (net/http / Gin) | direct REST integration |

## Skills included

- `/onelo-features` — scans your project, proposes feature names, inserts SDK snippets

## Installation

```bash
/plugin marketplace add onelo-tools/claude-plugins
/plugin install onelo-features@onelo-tools
```

## Usage

Run from your project root:

```
/onelo-features
```

The skill detects your language(s), scans for feature-worthy locations
(components, route handlers, view-models), proposes kebab-case feature
names, waits for your review, then inserts SDK snippets at the right
spot in each file.

After instrumentation, head to the [Onelo dashboard](https://app.onelo.tools)
→ Features → Registry — your discovered features show up tagged
**JUST ADDED**. Configure visibility per feature, then click **Deploy** to
push the rules to every connected SDK in real time over SSE.

## Live snippet templates

Snippets are fetched live from `https://app.onelo.tools/api/snippets` so
the inserted code always matches what the dashboard's SDK page shows.
A hardcoded fallback (in `skills/onelo-features/references/*.md`) is used
when the dashboard is unreachable.

To point the plugin at a non-production dashboard during development:

```bash
export ONELO_API_BASE="https://st.onelo.tools"
```
