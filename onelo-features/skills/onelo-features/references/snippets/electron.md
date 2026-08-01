# Electron

Baked into this plugin at publish time from `@onelo/snippets` — the same
source the dashboard **SDK** tab and **/docs** render from. Insert it as-is;
never write an Onelo SDK call from memory and never adapt another platform's
snippet.

## install
<!-- onelo:snippet sdk=features lang=electron field=install -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
npm install github:onelo-tools/onelo-electron#semver:*
```
<!-- /onelo:snippet -->

## init
<!-- onelo:snippet sdk=features lang=electron field=init -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
import { Onelo } from '@onelo/electron'

const onelo = new Onelo({
  publishableKey: 'onelo_pk_live_YOUR_KEY',
  apiUrl: 'https://api.onelo.tools',
  bundleId: 'com.company.app', // your app id — REQUIRED for codesign attestation (a signed build 403s without it)
  // Feature environment: 'test' | 'live' — which features env this app READS
  // (drafts/QA vs production), INDEPENDENT of your key. Electron runs in Node,
  // so the SDK AUTO-READS process.env.ONELO_FEATURE_ENVIRONMENT: set it to
  // 'test' while developing (enables discovery — features auto-register in your
  // dashboard Registry) and 'live' (or leave unset) in production (registry is
  // read-only). Set the SAME value in your backend so both read one environment.
  // To pass it explicitly instead of the env var:
  //   featureEnvironment: process.env.ONELO_FEATURE_ENVIRONMENT,
  // Status for a slug not yet in the snapshot. Fail-closed 'hidden' in prod; set
  // 'enabled' in dev to preview new gates before toggling them in the dashboard.
  // featureDefaultStatus: 'hidden',
  // Re-sync on OS resume / app re-activate (on by default). false = poll + SSE only.
  // autoLifecycleRefresh: true,
})
```
<!-- /onelo:snippet -->

## usage
<!-- onelo:snippet sdk=features lang=electron field=usage -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
// If you use your own auth system, call await onelo.identify(userId) after login
// so per-user/per-plan targeting can apply. (Skip if using Onelo Auth — automatic.)

// REACTIVITY — the SDK runs in the MAIN process. subscribe() fires on every cache
// change (an admin's Deploy over SSE, a plan change, a lifecycle resync); forward it
// to your window so the renderer re-renders. Returns an unsubscribe fn.
const unsubscribe = onelo.features.subscribe(() => {
  mainWindow.webContents.send('onelo:features-changed')
})
// Expose feature state to the renderer over IPC. ⚠️ Send featureSnapshot(name) —
// a PLAIN object with every getter (isEnabled/isVisible/badgeLabel/upgradeHint…)
// materialized as data. A raw feature() instance loses its getters over IPC's
// structured-clone, so the renderer would see them all as undefined and hide
// every tile. In the renderer: ipcRenderer.on('onelo:features-changed', refetch)
// and ipcRenderer.invoke('onelo:feature', name) → this handler.
ipcMain.handle('onelo:feature', (_e, name) => onelo.features.featureSnapshot(name))

// Feature states are available immediately — no init() needed. In the MAIN process
// read them directly (getters work here); to the renderer send featureSnapshot(name).
const f = onelo.features.feature('advanced-export')

// ⚠️ Render by STATUS, not a bare isEnabled — 'if (isEnabled)' HIDES the greyed
// (locked), upsell and coming-soon states. And a GATED tile is tappable-to-UPGRADE
// only when the backend says so: f.upgradeCta (the dashboard "Tapping the feature
// opens the upgrade flow" toggle) + f.requiredPlan — covers greyed, upsell AND
// coming_soon. onelo.openUpgrade(plan) opens the hosted upgrade in the system
// browser (subscriber → "Change plan" pinned; non-subscriber → store).
if (!f.isVisible || f.isDisabled) {
  // 'hidden' or 'disabled' → render NOTHING
} else {
  const canUpgrade = f.upgradeCta && f.requiredPlan != null
  if (f.isEnabled) {
    // enabled / new / beta → usable; run the feature on click
  } else if (canUpgrade) {
    // greyed / upsell / coming_soon + upgrade CTA → on click, open the upgrade flow:
    //   onelo.openUpgrade(f.requiredPlan)
    // Render f.badgeLabel: '🔒' (greyed) · 'Available in Pro' (upsell) · 'Coming Soon'.
  } else {
    // gated but informational (no upgrade CTA) → render f.badgeLabel, disable the click.
  }
}

// Escape hatch for edge cases — clears local cache (and notifies subscribers).
// You should rarely need this; deploys reach the SDK automatically.
onelo.features.invalidateCache()
// On teardown: unsubscribe()

// State helpers on feature(name):
// .isEnabled   → enabled | new | beta — the feature is USABLE (the ONE interactivity gate).
// .isVisible   → false ONLY for 'hidden' → when false, render nothing. ALSO hide 'disabled' via .isDisabled.
// .isDisabled  → 'disabled' — a killed feature; render nothing (isVisible is still true for it).
// .isNew / .isBeta → USABLE; cosmetic badge only (feature works).
// .isGreyed     → 'greyed'      — VISIBLE with 🔒; blocked. Tappable → upgrade when upgradeCta.
// .isComingSoon → 'coming_soon' — VISIBLE; blocked. Tappable → upgrade when upgradeCta (a
//                 plan-gated coming_soon CAN carry an upgrade CTA — don't assume it's inert).
// .isUpsell     → 'upsell'      — VISIBLE with "Available in <plan>"; blocked. Tap → onelo.openUpgrade.
// .upgradeCta   → dashboard "tap opens upgrade" toggle. PAIR WITH .requiredPlan → the tappable gate.
// .badgeLabel  → ready-made label ('New', '🔒', 'Available in Pro', 'Coming Soon', …) or null
// .requiredPlan / .requiredPlanLabel → plan that unlocks it (slug / human label).
// .upgradeHint → { requiredPlan, currentStatus } when plan-gated — for upgrade-prompt UI.
// .status      → 'enabled' | 'new' | 'beta' | 'greyed' | 'upsell' | 'coming_soon' | 'hidden' | 'disabled'
//
// RENDER?/INTERACTIVE?  enabled·new·beta = show + interactive · greyed·coming_soon·upsell = show +
//   (tap→upgrade when upgradeCta, else blocked) · hidden·disabled = don't render
```
<!-- /onelo:snippet -->
