# Modelling the developer's features — the guided walkthrough

## Contents
- Why a conversation beats a scan
- Running the walkthrough
- What counts as a feature
- Sub-features: where the upsell lives
- The naming convention
- Threads: one feature across the whole stack
- Capabilities with no UI
- Output of this phase
- Gating philosophy — gate functionality, not atoms

## Why a conversation beats a scan

A scan finds **code shapes**: screens, handlers, buttons. A product's features
are a different thing — they are the answers to "what can this app do, and what
would I sell separately?" Only the developer knows that.

So the scan is **material, not a verdict**. It tells you where things live; the
developer tells you what they are. A registry built purely from a scan reads
like a file listing and is useless for pricing, which is what the registry is
for.

## Running the walkthrough

Offer it every time, but never force it (see SKILL.md Phase 1.5). When the
developer takes it, work through their product in this order — one question at a
time, never a wall of them.

**1. What does your product do, in 5–15 lines?**
Ask for the feature list as *they* would put it on a pricing page. Do not
interrupt with code yet. This list is the target; the scan is how you find where
each item lives.

**2. Which of these are free, and which are paid?**
The answer sorts the list into things you gate for availability and things you
gate for plan. Anything they hesitate on is usually a sub-feature (below).

**3. For each paid item: is the whole thing paid, or a part of it?**
"Feature flags are free, but targeting rules are paid" → that is a feature plus
a sub-feature, not two features.

**4. Now map each item to the code.**
Use the scan results. For each item on their list, show the candidate location(s)
you found and ask "is this it?" Items with no candidate are the interesting ones
— usually capabilities with no UI, or something not built yet.

**5. Where does each item END?**
Ask explicitly for anything user-facing: "when the user clicks this, where does
it finish — still in the app, or does it hit your backend?" This is what makes
the wrapping end-to-end (see Threads below).

## What counts as a feature

A feature is something the user (or the buyer) would recognise as **a thing the
product can do**. Three shapes qualify:

| Shape | Example | Where it's gated |
|---|---|---|
| **Destination** — a screen, tab, window, route | Analytics dashboard | in the screen itself |
| **Capability** — a user-invoked action | Export to PDF | in the handler + on the trigger |
| **Product capability** — a behaviour with no screen of its own | Email notifications, webhooks, remove branding | wherever the behaviour is decided |

The third row is the one a pure scan misses, and it is often the most valuable
to sell. See "Capabilities with no UI" below.

What does **not** qualify: buttons, list tiles, providers, internal helpers,
persistence, initialisation. The ordered rules that drop these live in
[classification-rules.md](classification-rules.md).

## Sub-features: where the upsell lives

A sub-feature is a **mode or extension inside a feature that already exists** —
not a smaller feature.

The tell: the parent is available on the cheaper plan and the child is what you
charge for.

| Parent | Sub-feature | Why it's split |
|---|---|---|
| feature flags | targeting rules & rollout | flags are the hook, targeting is the upgrade |
| authentication | social login | sign-in is table stakes; OAuth is the sell |
| customer portal | refunds | portal is standard; self-serve refunds are not |

Ask about this explicitly for every paid feature: **"is there a part of this you
want to sell separately?"** Developers rarely volunteer it, and it is the single
highest-value question in the walkthrough.

## The naming convention

kebab-case, and the hierarchy lives **in the name**:

```
feedback              ← the feature
feedback-bug          ← a sub-feature of it
feedback-attachments  ← another sub-feature
```

The prefix is the parent's full name plus a hyphen. Nothing in Onelo parses this
today — it is a convention the skill enforces so a human reading the registry
sees the structure, and so grouping can be added later without touching any
data. Gating and plan assignment work on the flat list either way.

Rules:
- name the feature, not the widget: `pdf-export`, never `export-button`
- a sub-feature must repeat its parent's name in full — `feedback-bug`, not
  `bug-feedback`
- never invent a parent that isn't itself a feature just to create a namespace

## Threads: one feature across the whole stack

**End-to-end means across the stack, not within one language.**

A feature starts where the user touches it and ends where it actually finishes.
If the button is in Swift and the work completes in a Python handler, the
feature spans both — and gating only the button leaves the endpoint open. The
feature is then "off" visually and on in reality.

So for each feature, establish its **thread**:

```
feedback
  ├─ Swift   ReportBugButton.swift:42    (trigger)
  ├─ Swift   FeedbackSheet.swift:18      (destination)
  └─ Python  routes/feedback.py:87       (handler — where it ends)
```

Every point on the thread gets gated, **with the same feature name**. That is not
just tidiness: Onelo keys the registry by name, so declaring `feedback` from
Swift and `feedback` from Python produces **one registry entry tagged with both
platforms** — not two entries someone has to reconcile by hand. Different names
on the two sides silently create two features.

You cannot infer a thread reliably from code alone — a scan sees an HTTP call,
not which endpoint answers it. Ask (walkthrough question 5), show the thread you
believe in, and let the developer correct it.

## Capabilities with no UI

Some real product features have no screen and no button: "email notifications",
"webhooks", "remove branding". The scan's capability pass requires a UI call
site, so these get dropped as noise — correctly for internal plumbing, wrongly
for genuine product capabilities.

Rule: **do not silently drop them.** If a candidate looks like a product
capability but has no UI trigger, surface it in the proposal marked
`needs-confirmation` and ask. The developer either recognises it as a feature
they sell, or tells you it's plumbing — and either answer is cheap. Dropping it
in silence is the expensive outcome, because they never learn it was missed.

Signals worth surfacing this way: a boolean read from settings or config that
changes product behaviour; a branch on a plan or tier; a toggle persisted per
tenant or per app; an integration that can be switched off.

## Output of this phase

Before proposing any code change, you should have:

1. the developer's feature list, in their words
2. each one mapped to a **thread** through the stack
3. sub-features identified, named with the parent prefix
4. anything unresolved marked `needs-confirmation` rather than dropped

Only then move to the proposal table.

---

## Gating philosophy — gate functionality, not atoms

A feature flag is a contract with the user: "this part of the product is
on/off for you right now." Therefore, gate **whole pieces of functionality**
the user perceives as "a thing the app can do" — screens, tabs, windows,
modals, route handlers, end-to-end flows. **Do not** gate the small UI
building blocks those screens are made of.

If you flag `ResultBanner` separately from `GameBoard`, you can produce
nonsensical states — the banner hidden but the board still running, leaving
a half-finished game with no result UI. Atoms are implementation details of
a screen; the screen is the unit users (and your analytics, and your billing)
actually reason about.

**The opposite failure mode — orphaned triggers — is just as nonsensical.**
If you gate `SettingsWindow` but leave the toolbar button that opens it
visible, users click it and the app silently fails — they conclude the app
is broken. A feature isn't a single component; it's the **destination plus
every entry point that reaches it**. Phase 2 grep patterns include trigger
detection per language; Phase 2.5's new Rule 0 keeps triggers linked to
their destination so a feature's full surface gates as one unit. Triggers
default to `isVisible` (hide only when the feature's status is explicitly
`hidden`) so admin-controlled states like `greyed` (paid upsell) or
`coming_soon` (teaser) keep the trigger reachable without any code change.

The full classification rules — what counts as a screen, what counts as an
atom, what to skip — live in **Phase 2.5** below. Each language's reference
file only adds the *language-specific* signals (e.g. SwiftUI's
`WindowGroup`, Flutter's `Scaffold` rule, Next.js's `app/<route>/page.tsx`
routing convention) on top of those generic rules.

**One gate per presentation layer.** Gate the screen in the screen (its
`body` / `viewDidLoad` / render function) — NOT in its ViewModel. A
ViewModel guard leaves the UI rendered but inert: the user sees the screen,
clicks, and nothing happens — indistinguishable from a bug. ViewModel/handler
guards are reserved for capabilities (action handlers), where the UI trigger
is gated separately with `isVisible`.

---
