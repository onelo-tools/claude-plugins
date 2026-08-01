# Flutter — Dart

Baked into this plugin at publish time from `@onelo/snippets` — the same
source the dashboard **SDK** tab and **/docs** render from. Insert it as-is;
never write an Onelo SDK call from memory and never adapt another platform's
snippet.

## install
<!-- onelo:snippet sdk=features lang=flutter field=install -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```dart
# pubspec.yaml
dependencies:
  onelo:
    git:
      url: https://github.com/onelo-tools/onelo-flutter
      ref: main
```
<!-- /onelo:snippet -->

## init
<!-- onelo:snippet sdk=features lang=flutter field=init -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```dart
import 'package:onelo/onelo.dart';

// Onelo Features keeps SEPARATE Test and Live snapshots. In dev use 'test'
// (your in-progress features show + auto-discover into the registry); production
// reads 'live'. Flutter has no process.env, so pass featureEnvironment
// explicitly — the SAME value your backend uses (ONELO_FEATURE_ENVIRONMENT) so
// app + server resolve the same snapshot.
final onelo = Onelo(
  publishableKey: 'onelo_pk_live_YOUR_KEY',
  apiUrl: 'https://api.onelo.tools',
  callbackScheme: 'myapp',
  featureEnvironment: 'test', // 'test' in dev/staging · 'live' (or omit) in prod
  // Status for a slug not yet in the snapshot. Fail-closed 'hidden' in prod;
  // pass FeatureStatus.enabled in dev to preview new gates before toggling them.
  // featureDefaultStatus: FeatureStatus.hidden,
  // Foreground/wake forces a REST resync (on by default); set false for poll+SSE only.
  // autoLifecycleRefresh: true,
);
```
<!-- /onelo:snippet -->

## usage
<!-- onelo:snippet sdk=features lang=flutter field=usage -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```dart
// If you use your own auth system, call onelo.identify(userId) after login so
// per-user/per-plan targeting applies. (Skip if using Onelo Auth — automatic.)
await onelo.identify(currentUser.id);

// SECURE MODE (recommended for paid apps): your backend signs the userId with
// the app secret_key — hash = HMAC-SHA256(secret_key, "user:" + userId) — so
// per-user targeting can't be spoofed with a raw id.
//   final hash = await myBackend.fetchOneloIdentity(currentUser.id);
//   await onelo.identify(currentUser.id, userIdHash: hash);

// Optionally await the first resolve so UI renders without a "hidden" flash:
await onelo.features.ready();

// Read features synchronously anywhere. Updates arrive automatically in REAL
// TIME over SSE (Deploy / kill-switch / plan change), with a 60s poll +
// foreground/wake resync as fallback.
//
// ⚠️ Render by STATUS — do NOT gate visibility with isEnabled: it's true only
// for enabled/new/beta, so it HIDES greyed (locked) / coming_soon / upsell.
final f = onelo.features.feature('advanced-export');
if (f.isVisible) {                    // false ONLY when status is hidden
  //   f.isGreyed   → 🔒 locked
  //   f.isUpsell   → "Available in ${f.planLabel}"  // Pro / Business / …
  //   f.badgeLabel → 'New' | 'Beta' | 'Coming Soon' | '🔒' (greyed) |
  //                  'Available in <plan>' (upsell) | null — render as-is
}

// Run gated code only when the feature is actually usable (enabled/new/beta):
if (onelo.features.isEnabled('advanced-export')) {
  // …
}

// REACT to changes — OneloFeatures is a ChangeNotifier, so widgets can rebuild
// the moment a flag flips (Deploy, plan change, resync):
//   ListenableBuilder(
//     listenable: onelo.features,
//     builder: (context, _) =>
//       onelo.features.isEnabled('advanced-export') ? EnabledView() : LockedView(),
//   )

// Escape hatch — force a REST reconcile (debounced to 1/sec). Rarely needed;
// SSE + poll reach the SDK automatically.
await onelo.features.refresh();

// GATED MODULE DOWNLOADS (optional) — mint a short-lived, per-user token so your
// CDN serves gated module code only to entitled users. Requires an identified
// user; throws OneloFeaturesException (notEntitled / notAuthenticated) otherwise.
//   final token = await onelo.features.moduleToken('advanced-export');
//   // send the token to your CDN; it verifies it server-side before serving.

// OPEN THE UPGRADE FLOW — onelo.openUpgrade(plan) opens the hosted upgrade in the
// system browser: an active subscriber gets "Change plan" (target pinned); a
// non-subscriber gets the store. The backend decides — you just pass the plan.
//
// A gated tile is TAPPABLE-to-upgrade exactly when the backend says so:
//   f.upgradeCta   → the dashboard "Tapping the feature opens the upgrade flow" toggle
//   f.requiredPlan → the plan slug to upgrade to (pass it to openUpgrade)
// This is the ONE gate — it works for BOTH an "Available in <plan>" upsell AND a
// LOCKED (padlock) tile. A locked tile is NOT inert: when upgradeCta is on it must
// open the upgrade on tap.
//
//   if (!f.isVisible) return const SizedBox.shrink();          // hidden → render nothing
//   final canUpgrade = f.upgradeCta && f.requiredPlan != null; // backend: this tap opens upgrade
//   ElevatedButton(
//     onPressed: f.isEnabled ? runExport                        // usable → run it
//         : canUpgrade ? () => onelo.openUpgrade(f.requiredPlan!)  // gated + tappable → upgrade
//         : null,                                               // gated, informational only → disabled
//     child: Text(f.isEnabled ? 'Export'
//         : f.isComingSoon ? 'Export (soon)'
//         : canUpgrade ? 'Upgrade to ${f.planLabel}'           // e.g. "Upgrade to Pro"
//         : (f.badgeLabel ?? 'Export')),                        // 🔒 / New / Beta / Available in <plan>
//   )
// .isEnabled → enabled/new/beta — USABLE; the ONE interactivity gate.
// .isVisible → false only for hidden → when false, render nothing.
// .isNew / .isBeta → usable; cosmetic badge only.   .badgeLabel → ready label or null
// .isGreyed / .isComingSoon → VISIBLE but BLOCKED; a locked tile is TAPPABLE→upgrade when upgradeCta.
// .isUpsell → VISIBLE but BLOCKED; tap → onelo.openUpgrade.   .reason .requiredPlan .requiredPlanLabel .planLabel
// .upgradeCta → true when the dashboard says a gated tap should open the upgrade flow. PAIR WITH
//   .requiredPlan (the target) → the correct tappable-to-upgrade gate for greyed AND upsell.
// .upgradeHint → UpgradeHint(requiredPlan, currentStatus)? — convenience that bundles the
//   plan-path check (non-null when reason=='plan' + a requiredPlan + a gated status). For the
//   tap gate prefer .upgradeCta + .requiredPlan, which also covers user-override upsells.
// .status → enabled | disabled | greyed | hidden | upsell | newFeature | beta | comingSoon
```
<!-- /onelo:snippet -->
