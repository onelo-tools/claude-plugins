# Feature classification rules (Phase 2.5)

The atom filter: for every candidate from Phase 2, evaluate these rules **in
order** — first match wins. This is what keeps the registry to things a user
would call "a thing the app can do", instead of every button and list tile.

## Contents
- Rule 0 — Linked trigger → keep (`trigger`)
- Rule 0.5 — Internal/dev surface → `internal` (ask the developer)
- Rule 1 — Backend route handler → `screen` (keep)
- Rule 2 — Screen-shaped name or path → `screen` (keep)
- Rule 3 — Atom-shaped name → drop (`atom-likely`)
- Rule 4 — Atom-shaped path → drop (`atom-likely`)
- Rule 5 — Language override
- Rule 6 — Default → `ambiguous` (keep, flag)
- Output

For every candidate from Phase 2, evaluate the rules below **in order**.
First match wins.

## Rule 0 — Linked trigger → keep (`trigger`)

If the candidate was matched by trigger discovery (Phase 2's trigger pass)
and links to an already-classified `screen` destination, classify as
`trigger` and keep — even if it would otherwise drop under Rule 3
(atom name like `*Button`) or Rule 4 (atom path like `/components/ui/`).

A trigger inherits its destination's feature name. The skill snippet
inserted in Phase 5 will use `isVisible` rather than `isEnabled` so
non-`hidden` statuses keep the trigger reachable.

This rule runs FIRST so that aggressive atom filtering (Rules 3-4)
doesn't drop legitimate triggers.

## Rule 0.5 — Internal/dev surface → `internal` (ask the developer)

If the symbol OR proposed_name contains one of these word-boundary segments
(case-insensitive): `Debug`, `Dev`, `Internal`, `Diagnostic`, `Test`,
`Playground`, `Sandbox` → classify as `internal`. Examples: `DebugWindowView`
→ internal; `debug-pipeline-window` → internal. A product feature that merely
contains the substring without word boundary (`Debugger` as a product) stays
a normal candidate — when unsure, classify `internal` and let Phase 3 ask.

`internal` candidates are NOT dropped and NOT silently kept: they go to a
separate section of the Phase 3 table with default action **skip**. The
developer can keep one with `keep #N` (legit use case: remotely toggling a
debug window). Every kept internal feature is renamed with the `internal-`
prefix (`internal-debug-pipeline`) so the dashboard's slug-prefix grouping
collects them in one group, clearly separated from product features.

## Rule 1 — Backend route handler → `screen` (keep)

If the language reference flagged the candidate as a route handler
(Express/Hono call, Next.js `route.ts`), classify as `screen` and keep.

Backend handlers are inherently the right unit; atom filtering does not
apply to them.

## Rule 2 — Screen-shaped name or path → `screen` (keep)

Symbol ends with one of these (case-insensitive) → keep:

```
Screen   Page    Tab     Window   Sheet    Modal
Wizard   Flow    FlowView         Onboarding
Activity Fragment ViewController
```

OR the file path contains one of these → keep:

```
/screens/    /pages/    /tabs/    /windows/    /flows/    /routes/
/wizards/    /onboarding/
```

## Rule 3 — Atom-shaped name → drop (`atom-likely`)

Symbol ends with one of these (case-insensitive) → drop:

```
Cell      Row       Item      Tile        Chip      Tag
Banner    Indicator Badge     Pill        Toast     Tooltip
Bar       Header    Footer    Divider     Skeleton  Placeholder
Avatar    Spinner   Loader    Icon        Thumbnail
Field     Input     Button    Label       Bubble
```

## Rule 4 — Atom-shaped path → drop (`atom-likely`)

File path contains one of these → drop:

```
/atoms/         /primitives/        /elements/
/components/atoms/    /components/primitives/    /components/ui/
/ui/atoms/      /ui/primitives/     /ui/
/design-system/    /designsystem/   /ds/
/widgets/atoms/    /shadcn/
```

## Rule 5 — Language override

The per-language reference file may add **its own** rules — extra atom
suffixes only meaningful in that ecosystem (e.g. `Tile` in Flutter,
`Style`/`Modifier` in SwiftUI), or extra disambiguators (e.g. Flutter's
"builds a `Scaffold` → screen" rule). Apply those after Rules 1–4.

## Rule 6 — Default → `ambiguous` (keep, flag)

Anything else stays in the proposal table but is marked `?` in the
classification column so the developer can drop it quickly.

## Output

Drop all `atom-likely` candidates from the list before Phase 3. Record
the dropped count and surface it in the proposal table header:

```
Found 12 instrumentable locations (skipped 47 atom-like; see Phase 2.5):
```

The developer can still re-add a dropped candidate manually in Phase 3
with `add feature <name> in <file> at line <N>`.
