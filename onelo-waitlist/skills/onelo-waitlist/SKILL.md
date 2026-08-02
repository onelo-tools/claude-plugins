---
name: onelo-waitlist
description: Adds an Onelo Waitlist to a pre-launch product — embeds the hosted signup page, or wires a custom form, and sets up the launch switch that turns the same page into the store. Use when a developer wants to collect signups before launch, asks how to go from waitlist to selling, or reports that their waitlist page shows the store (or vice versa).
allowed-tools: Bash Glob Grep Read Edit Write
---

# Onelo Waitlist — starter

Collect signups before launch, then sell from the same place — without touching
the code again.

## Run checklist — copy into your FIRST reply and tick off IN ORDER

```
- [ ] 0 · Prerequisites: app exists, a waitlist exists with a slug
- [ ] 1 · Confirm the launch behaviour (redirect to store, or independent page)
- [ ] 2 · Read the snippet — or the custom-form guide if they insist
- [ ] 3 · Propose placement → WAIT for approval
- [ ] 4 · Insert + substitute the slug (three places!)
- [ ] 5 · Verify both phases: before launch AND after
```

---

## The idea that shapes everything

**The waitlist URL and the store are the same address.** `/waitlist/<slug>`
decides at request time what to show.

Embed it once on your pre-launch page and at launch that same embed starts
showing your store — no code change, no redeploy, no second snippet. That is why
a pre-launch page should use the **waitlist** embed rather than the store embed.

Never write launch detection into your code: no date check, no flag, no swapping
the iframe `src`. The hosted page already decides. See
[references/waitlist-modes.md](references/waitlist-modes.md) for the full truth
table.

---

## Phase 1 — Confirm the launch behaviour

Ask, because it changes what happens on the page they are about to build:

**"When you launch, should this page become your store?"**

- **Yes** (the default) — one page covers the whole lifecycle. Dashboard →
  Waitlist → Settings → **Auto-redirect to store** stays ON.
- **No** — the waitlist stays its own page and shows a branded "closed" state at
  launch; the store lives at its own URL. Turn **Auto-redirect to store** OFF.

Also mention the optional **"Switch to store on"** timestamp: the page flips
itself at that moment, so a launch can happen while nobody is watching. Leave it
blank to flip manually.

---

## Phase 2 — Read the snippet

**Hosted embed (recommended, and what the dashboard shows):**
[references/snippets/html-embed.md](references/snippets/html-embed.md)

Two variants, pick ONE:
- **Full-width** — follows the browser width; use it if you want the
  multi-column desktop store on wide screens after launch.
- **In-container** — fills its slot; keeps everything in your page's column.

Both auto-resize to their content and both handle the form *and* the store.

**Custom form** — only if the developer explicitly asks:
[references/custom-form.md](references/custom-form.md). Read the trade-offs in
that file to them first; a custom form loses consent handling, localisation and
the launch switch.

**Native apps** need nothing: `loadAuthView()` routes users to the waitlist while
Waitlist mode is on and to the store afterwards. If that is the whole ask, point
them at the `onelo-auth` skill and stop.

---

## Phase 3 — Propose placement, then WAIT

```
Onelo Waitlist — proposed changes

#1  src/pages/index.astro:88   insert the in-container waitlist iframe
#2  (same file)                + the auto-resize script

Slug: YOUR_WAITLIST_SLUG → early-access
Launch behaviour: Auto-redirect to store = ON (page becomes the store)
```

Put it where a visitor lands with intent — below the hero, or on a dedicated
`/early-access` page. Not in a footer.

Wait for approval. Commands: `apply`, `skip #N`, `explain #N`, `cancel`.

---

## Phase 4 — Insert

- Replace `YOUR_WAITLIST_SLUG` — it appears **three times** per snippet (iframe
  `src`, iframe `id`, `getElementById` in the script). Miss one and the
  auto-resize silently stops working.
- Keep the script. Without it the iframe holds a fixed height and gets its own
  scrollbar inside your page.
- `?lang=en` picks the locale — change it to match a translation configured in
  the dashboard, or drop it.
- Change nothing else.

---

## Phase 5 — Verify BOTH phases

Most integrations only test the phase they are in today. Test both.

**Now (Waitlist mode ON):**
1. The signup form renders and the height follows the content.
2. Submitting enrols the address — it appears in the dashboard.
3. Submitting the same address again shows a friendly "already on the list".

**After launch — flip Waitlist mode OFF in the dashboard and reload:**
4. With Auto-redirect ON, the same embed now shows the **store**.
5. With Auto-redirect OFF, it shows the branded **"closed"** state.
6. Turn Waitlist mode back ON when you are done testing.

---

## Phase 6 — What this connects to

- **Selling the plans** the page switches to → `onelo-store`.
- **Signing users in** → `onelo-auth`. In native apps Auth does the waitlist
  routing for you.
- **Consent / GDPR** — configure it on the waitlist itself; the hosted form
  renders the checkboxes and records consent. A custom form does not.
- **Branding** — colours, logo and fonts come from the dashboard and apply to the
  form, the store and the "closed" state alike.

---

## Troubleshooting

| Symptom | Cause |
|---|---|
| Waitlist page shows the store | Waitlist mode is OFF (or the "Switch to store on" time passed) and Auto-redirect is ON. Working as configured. |
| Waitlist page shows a "closed" screen | same, but Auto-redirect is OFF — that is the independent-page behaviour. |
| Store asks for an invite token | Waitlist mode is ON — that is the gate. Turn it off to open sales. |
| Iframe keeps a fixed height / inner scrollbar | the resize `<script>` was dropped, or the slug in `getElementById` doesn't match the iframe `id`. |
| Nothing renders | wrong slug in `src` — copy the hosted URL from the dashboard SDK → Waitlist tab. |
| Custom form "works" but nobody appears on the list | `join()` never throws; a failure resolves `{ success: false }`. The code isn't checking `success`. |
| Signups still arrive after launch | a custom form has no launch switch — it keeps collecting. The hosted embed does not have this problem. |
