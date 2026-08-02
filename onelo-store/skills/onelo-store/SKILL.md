---
name: onelo-store
description: Adds the Onelo Store to a product — embeds the hosted store page on a website with the official iframe snippet, or sets up the in-app store that Onelo Auth drives in native apps. Use when a developer wants to sell plans or subscriptions with Onelo, asks how to show their store, or reports that their store renders but purchases fail. On a pre-launch page it redirects to the waitlist embed, which becomes the store at launch.
allowed-tools: Bash Glob Grep Read Edit Write
---

# Onelo Store — starter

Onelo hosts the whole store: plans, checkout, tax, receipts. You never build a
pricing page or handle a payment.

There are **two completely different ways** to show it, and picking the wrong one
is the most common mistake. Decide first.

## Run checklist — copy into your FIRST reply and tick off IN ORDER

```
- [ ] 0 · Prerequisites: app exists, store options + Stripe connected
- [ ] 0b · PRE-LAUNCH? → use the WAITLIST embed instead, and stop (see below)
- [ ] 1 · Decide the surface: website embed or in-app? → say which and why
- [ ] 2 · Website → read the snippet file; In-app → read the setup guide
- [ ] 3 · Propose the change → WAIT for approval
- [ ] 4 · Insert + substitute the app slug
- [ ] 5 · Verify (the checks differ per surface)
```

---

## Phase 0b — Not launched yet? Embed the WAITLIST, not the store

Ask this first, before anything else:

**"Are you launched, or still collecting signups?"**

If they are pre-launch, **do not embed the store**. Embed the **waitlist**
instead — the waitlist URL and the store are the same address, so
`/waitlist/<slug>` shows the signup form now and starts showing your **store**
the moment you launch. Same embed, no code change, no redeploy, no second
snippet.

Launching is then one toggle in **Waitlist → Settings** (or a scheduled
timestamp under "Switch to store on", so it flips itself while nobody is
watching). "Auto-redirect to store" is ON by default — that is what makes the
page become the store.

→ Hand the work to the **`onelo-waitlist`** skill and stop here.

Only embed the store directly when the product is already selling, or when the
store lives on its own page separate from a waitlist that stays independent
("Auto-redirect to store" turned OFF).

---

## Phase 1 — Which surface? (decide before anything else)

Ask, or infer from the project:

**Website embed** — a marketing site, landing page or docs site where visitors
should see and buy plans. Plain HTML iframe, works in any stack.

**In-app store** — a Swift / Android / Flutter / React Native / Electron app
where users must have a plan to use the product. **No code at all**: the store
appears inside the Onelo sign-in view. It is set up in the dashboard and driven
by Onelo Auth.

A product often wants **both**: the embed on the marketing site, the gate inside
the app. Say so and do them one at a time.

---

## Phase 2 — Read the right file

**Website embed** → [references/snippets/html-embed.md](references/snippets/html-embed.md)

Two variants, pick ONE:
- **Full-width** — follows the browser width; use for the multi-column desktop
  store on wide screens.
- **In-container** — fills its slot; use to keep the store inside your page's
  column.

Both auto-resize their height to the content, so there is no inner scrollbar.

**In-app store** → [references/in-app-store.md](references/in-app-store.md)

That file is prose, not code — there is nothing to insert. It covers what to
turn on in the dashboard, why it needs Onelo Auth, and what the user then
experiences.

**Never write an Onelo snippet from memory** and never adapt one surface's code
into the other. The two point at different URLs with different gating.

---

## Phase 3 — Propose, then WAIT

Website embed:

```
Onelo Store — proposed changes

#1  src/pages/pricing.astro:42   insert the in-container store iframe
#2  (same file)                  + the auto-resize script

Slug: YOUR_APP_SLUG → onelo-tools
```

In-app: there is no diff. Your output is the dashboard checklist plus a pointer
to the `onelo-auth` skill if Auth is not wired yet.

Wait for approval. Commands: `apply`, `skip #N`, `explain #N`, `cancel`.

---

## Phase 4 — Insert (website embed only)

- Replace `YOUR_APP_SLUG` with the real slug — it appears **three times** per
  snippet (iframe `src`, iframe `id`, and `getElementById` in the script).
  Miss one and the auto-resize silently stops working.
- Keep the script. Without it the iframe keeps a fixed height and the store gets
  its own scrollbar inside your page.
- Change nothing else in the file.

---

## Phase 5 — Verify

**Website embed:**
1. The store renders and its height follows the content (no inner scrollbar).
2. Clicking a plan opens checkout — **not** a 404. A 404 on `/buy/…` means the
   access gate is off; see the troubleshooting table.
3. On a narrow screen the layout still fits (full-width breaks out of the column
   by design — check it doesn't cause horizontal scroll on the page).

**In-app store:**
1. A brand-new account with no plan is routed sign-in → store.
2. Closing the store without buying resolves `null` — the app must keep its
   locked screen.
3. After purchase the app renders and the plan shows in the dashboard.

---

## Phase 6 — What this connects to

- **Not signed in yet?** → the `onelo-auth` skill. The in-app store *requires*
  Auth; the website embed does not.
- **Letting subscribers change or cancel a plan?** → `onelo-customer-portal`.
- **Pre-launch?** → the `onelo-waitlist` skill. Its embed *becomes* this store at
  launch, so on a pre-launch page it is the one to insert. In native apps
  Waitlist mode routes sign-in to the waitlist and then to the store, also with
  no code change.
- **Look and feel** is entirely in the dashboard → **Branding**. Don't try to
  style the iframe's contents; it's cross-origin and it won't work.

---

## Troubleshooting

| Symptom | Cause |
|---|---|
| Store renders but the buy link 404s | "Require plan on sign-up" is OFF → the app is free, so there is nothing to buy. Turn it on in **Paywall → Store**. |
| Store shows the waitlist instead of plans | Waitlist mode is ON — that's the intended behaviour. Turn it off in **Waitlist → Settings** to open sales. |
| Embedded the store on a pre-launch page and now need a waitlist too | you embedded the wrong one — swap it for the waitlist embed, which becomes the store at launch. See the `onelo-waitlist` skill. |
| Iframe keeps a fixed height / has its own scrollbar | the resize `<script>` was dropped, or the slug in `getElementById` doesn't match the iframe `id`. |
| Nothing renders, console shows a frame error | wrong slug in `src` — check the hosted URL in the dashboard SDK → Store tab. |
| Native app: sign-in never shows the store | "Require plan on sign-up" is OFF, or there is no visible store option, or Stripe isn't connected. |
| Store is empty | no active products, or your Onelo plan doesn't include the Store capability. |

## Backend verification

There is **no backend snippet for Store yet** — the dashboard's "Verify access
(backend)" tab is a placeholder. If a developer needs to check entitlement
server-side, say so plainly rather than improvising an endpoint call.
