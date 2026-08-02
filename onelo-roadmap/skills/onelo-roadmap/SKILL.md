---
name: onelo-roadmap
description: Publishes an Onelo public roadmap — a hosted page listing the items you marked Public, linked or embedded on your site with no SDK code. Use when a developer wants to show what they are building, share a changelog-style roadmap, or asks how to make feature requests visible to users.
allowed-tools: Bash Glob Grep Read Edit
---

# Onelo Roadmap — starter

**There is no SDK code here.** The roadmap is a hosted public page. Your job is
to put its link somewhere people will find it, and to make sure the developer
understands what appears on it.

If you were expecting a snippet: there isn't one, and inventing an API call
would be wrong. This skill is short on purpose.

## Run checklist — copy into your FIRST reply and tick off IN ORDER

```
- [ ] 1 · Confirm the app has a slug → get the roadmap URL
- [ ] 2 · Explain what shows publicly (the Public flag) — this is the whole gotcha
- [ ] 3 · Propose where the link goes → WAIT for approval
- [ ] 4 · Insert the link (or iframe, if they want it in-page)
- [ ] 5 · Verify it opens signed-out
```

---

## Phase 1 — The URL

```
https://onelo.tools/roadmap/<teamSlug>/<appSlug>
```

Both slugs come from the dashboard; the exact URL is shown in **SDK → Roadmap**
with a copy button. If the app has no slug yet, the roadmap URL doesn't exist —
say so and stop.

The page is **public**: anyone with the link can open it, no sign-in, no
enablement toggle, no plan gate on viewing.

---

## Phase 2 — What actually shows (say this out loud)

**Only items marked Public in the dashboard appear.** Everything else — internal
plans, half-formed ideas, anything a competitor shouldn't read — stays hidden by
default.

This is the one thing developers get wrong. They link the page, look at it, see
almost nothing, and assume it's broken. It isn't: they haven't marked anything
Public yet.

Tell them before they paste the link anywhere.

---

## Phase 3 — Where to put it

A plain link is usually right:

- footer, next to Changelog
- inside the app's help or account menu
- at the end of a release note ("what's next")
- in the reply when someone requests a feature

If they want it **in-page**, an ordinary iframe works — it is a normal web page:

```html
<iframe
  src="https://onelo.tools/roadmap/YOUR_TEAM_SLUG/YOUR_APP_SLUG"
  style="width:100%;border:none;min-height:640px;"
  loading="lazy"
></iframe>
```

Unlike the store and waitlist embeds, there is **no official resize script** for
the roadmap, so give it a sensible `min-height` rather than expecting it to grow
to its content.

Propose the placement and wait for approval. Commands: `apply`, `skip #N`,
`cancel`.

---

## Phase 4 — Verify

1. Open the URL **in a private window** — it must render signed-out.
2. The items you expect are there. If it looks empty, check the Public flag
   before touching anything else.
3. If embedded, it fits without an inner scrollbar at your chosen `min-height`.

---

## Phase 5 — What this connects to

- **Where roadmap items come from** → `onelo-feedback`. Feature requests users
  submit are what you triage and then mark Public.
- **Shipping the thing** → `onelo-features`. A roadmap item that shipped behind a
  flag can be rolled out gradually.

---

## Troubleshooting

| Symptom | Cause |
|---|---|
| Page is empty | nothing is marked **Public** yet — the usual cause. |
| URL 404s | the app has no slug, or the team slug is wrong. Copy the URL from SDK → Roadmap rather than assembling it by hand. |
| An internal item is visible | it was marked Public. Unmark it in the dashboard. |
| Embedded roadmap is cut off | no resize script exists for this page — raise `min-height`. |
