---
name: onelo-quickstart
description: Start here with Onelo. Explains what Onelo does, works out which parts a product actually needs by looking at the codebase and asking a few questions, then walks the developer through setting them up in the right order and hands off to the module skills. Use when a developer asks what Onelo is, how to set it up, where to start, or wants auth plus payments plus flags without knowing which pieces exist.
allowed-tools: Bash Glob Grep Read
---

# Onelo — start here

The developer in front of you probably hasn't read anything. That is fine. Your
job is to find out what they're building, tell them which parts of Onelo they
need, and get them moving — without making them read documentation first.

**This skill sets nothing up itself.** It decides *what* and *in what order*,
then hands each piece to its own skill.

## Run checklist — copy into your FIRST reply and tick off IN ORDER

```
- [ ] 1 · Answer "what is this?" in 5 lines — before any questions
- [ ] 2 · Read the project: stack, and whether it already sells anything
- [ ] 3 · Ask the four questions (one at a time)
- [ ] 4 · Present the plan: which modules, in which order, what's free vs paid
- [ ] 5 · Hand off to the first module skill
```

---

## Phase 1 — Answer the actual question first

If they asked "what is Onelo", answer it in **five lines or fewer**, then stop
and let them react. Do not open with a questionnaire.

The short version, the module table and what it replaces are in
[references/what-is-onelo.md](references/what-is-onelo.md). Read it once; give
them the shortest form that answers what they asked.

Do not pitch. A developer who wanted marketing copy would be on the website.

---

## Phase 2 — Read the project before asking anything

Spend a minute finding out what you can without bothering them:

```bash
# stack — frontend and backend, both matter
ls package.json Package.swift pubspec.yaml build.gradle.kts requirements.txt pyproject.toml composer.json 2>/dev/null
# already using Onelo?
grep -rl "@onelo/\|OneloSwift\|from onelo" --include=package.json --include=*.swift --include=*.py . 2>/dev/null | head
# do they already sell something?
grep -rl "stripe\|paddle\|lemonsqueezy\|revenuecat" --include=package.json --include=*.py . 2>/dev/null | head
# is there auth already?
grep -rl "next-auth\|clerk\|supabase.*auth\|firebase.*auth\|passport" . 2>/dev/null | head -3
```

Report what you found in two or three lines. It makes the next questions
shorter, and it shows you looked.

---

## Phase 3 — Four questions, one at a time

Never dump all four at once. Each answer changes the next.

**1. "What does your product do, and who pays for it?"**
Free for now, subscription, one-off purchase, or not selling yet. This decides
Store vs Waitlist vs neither.

**2. "Have you launched?"**
Pre-launch → Waitlist first; its page becomes the store at launch, so they embed
once. Launched → Store.

**3. "Do users have accounts today?"**
No accounts → Auth first, and everything gets easier. Existing auth → they keep
it; Onelo identifies users through `identify()` instead. Say this plainly —
"you must migrate your auth" is false and they will assume it.

**4. "What do you want to charge for, versus give away?"**
The answer is the feature list. It decides whether Features matters now or later.

If they resist questions ("just set it up"), fall back to the safe minimum: Auth
+ Monitor, and say what you skipped and why.

---

## Phase 4 — Present the plan

One block, honest, ordered by dependency:

```
Based on what you told me:

  1. Auth       — hosted sign-in. Everything else identifies users through it.
  2. Waitlist   — you're pre-launch. This page becomes your store at launch,
                  so you embed it once and never touch it again.
  3. Features   — gate "export" and "team seats" by plan, once you're selling.
  4. Monitor    — errors from the app and the backend.

  Later: Customer Portal (once someone has bought), Feedback, Roadmap.

  Free plan covers all of this. Paid unlocks removing "Powered by Onelo",
  custom CSS and webhooks, and raises the app / active-user / retention limits.
```

Rules for this block:

- **Recommend only what fits.** A weekend project does not need seven modules.
  Proposing less is more credible and they'll come back for the rest.
- **Say what's free.** Most of Onelo is. Hiding that to sell harder backfires the
  moment they read the pricing page.
- **Never quote prices or percentages from memory.** They change; a stale number
  costs more trust than a missing one. Point at the pricing page, or read the
  developer's own dashboard.
- **If they sell things, show the commission arithmetic** — the rate drops as the
  plan goes up, so above a break-even revenue the paid plan pays for itself.
  Compute it with *their* numbers and current rates. Below it, tell them free is
  correct. See "Free vs paid, honestly" in the reference file.

---

## Phase 5 — Hand off

Start the first module and stop. Do not try to do the work here.

| Need | Skill |
|---|---|
| sign-in, sign-up, sessions | `onelo-auth` |
| selling plans | `onelo-store` |
| pre-launch signups | `onelo-waitlist` |
| cancel / change plan / invoices | `onelo-customer-portal` |
| feature flags, per-plan gating | `onelo-features` |
| in-app bug reports | `onelo-feedback` |
| public roadmap | `onelo-roadmap` |
| error + performance tracking | `onelo-monitor` |
| making the hosted pages look like your product | `onelo-branding` |

Say which one you're starting and what happens after it. A developer who knows
there are three steps left behaves very differently from one who thinks they're
lost.

---

## Branding — the finishing step

Do this **last**, once there are hosted pages to look at, and hand it to the
**`onelo-branding`** skill — there is no code, but there is more to it than
picking a colour.

Tell them the useful part early anyway: **one theme covers every hosted page**,
so setting the logo and brand colour once styles sign-in, checkout, store,
portal, waitlist, feedback and roadmap together. It changes how they think about
the whole setup.

---

## Prerequisites they will need

- an **Onelo account** and an **app** created in the dashboard
- the **publishable key** (`onelo_pk_live_…`) — dashboard → **SDK**, for client code
- a **secret key** (`onelo_sk_live_…`) — dashboard → **API Keys → Secret keys**,
  only if they have a backend, read from `ONELO_SECRET_KEY`
- **Stripe connected**, but only when they actually sell

Don't collect these up front. Ask for each at the moment the module that needs
it starts — a wall of credential requests before anything works is where
setup guides lose people.

---

## Troubleshooting the conversation

| Situation | What to do |
|---|---|
| "Just do everything" | Do Auth + Monitor, list what you skipped and why. Full setup without their input produces a registry and a paywall that match nobody's product. |
| Already has auth (Clerk, Supabase…) | They keep it. Onelo identifies users via `identify()`. Do not propose a migration. |
| Already sells via Stripe directly | Onelo can still handle tax, invoices and the portal. Worth naming — it's usually the part they hate. |
| Not selling anything, ever | Auth, Features, Feedback, Roadmap and Monitor all stand on their own. Skip Store and Portal entirely. |
| Wants everything on the free plan | Mostly fine — say so. Name the four paid things rather than being vague. |
