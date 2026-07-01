---
name: onelo-monitor
description: Onelo Monitor instrumentation plugin for Claude Code. Audits monitor usage for anti-patterns and instruments new operations correctly.
channel: staging
author: onelo-tools
---

> **Channel:** This is the **staging** branch — auto-refresh channel. The
> marketplace entry intentionally omits `version`, so Claude Code falls back
> to the git commit SHA as the cache key. Every push to `staging` invalidates
> the user's cache and is delivered on the next `/plugin update` — no manual
> version bump required.
>
> The `main` branch (production) pins an explicit `version` for stability.

# onelo-monitor Plugin

Audits and instruments the [Onelo](https://onelo.tools) Monitor SDK (error +
performance tracking that powers Feature Health and incidents).

Unlike feature-flag instrumentation, monitoring is **selective** — this plugin
does NOT mass-wrap code. It has two jobs:

- **Audit** existing instrumentation for anti-patterns (error-only features that
  never auto-resolve, failure-mode names, `event()` that should be `track()`,
  missing facet meta).
- **Instrument** one operation at a time with the right primitive, name, and meta.

## Supported SDKs

| Language | Package | Status |
|---|---|---|
| Swift (iOS / macOS) | `onelo-swift` (SwiftPM) | ✅ supported |
| Python (FastAPI / Django / Flask / …) | `onelo-python` | ✅ supported |
| JavaScript / TypeScript (web & SSR/Node) | `@onelo/js` | ✅ supported |
| Electron / React Native / Kotlin / Flutter / Go | — | ⏳ deferred (surface may be outdated) |

## Skills included

- `/onelo-monitor` — audit existing instrumentation, or instrument a new operation

## Installation

```bash
/plugin marketplace add onelo-tools/claude-plugins
/plugin install onelo-monitor@onelo-tools
```
