# Android — Kotlin

Baked into this plugin at publish time from `@onelo/snippets` — the same
source the dashboard **SDK** tab and **/docs** render from. Insert it as-is;
never write an Onelo SDK call from memory and never adapt another platform's
snippet.

## install
<!-- onelo:snippet sdk=features lang=android field=install -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```kotlin
// build.gradle.kts (app/module)
// mavenCentral() is standard in Android projects — add it to settings.gradle.kts
// dependencyResolutionManagement { repositories { mavenCentral() } } if missing.
implementation("tools.onelo:onelo-android:1.+")
```
<!-- /onelo:snippet -->

## init
<!-- onelo:snippet sdk=features lang=android field=init -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```kotlin
import com.onelo.android.Onelo
import com.onelo.android.OneloConfig

val config = OneloConfig(
    publishableKey = "onelo_pk_live_YOUR_KEY",
    apiUrl = "https://api.onelo.tools",
    // Features environment: "test" | "live" — a SEPARATE Test and Live snapshot. In dev
    // use "test" (your in-progress features show + auto-discover into the dashboard
    // Registry); production reads "live" (Registry is read-only). NOT decided by your key.
    // Set the SAME value on your backend so app + server resolve the same snapshot. Omit /
    // "live" for release builds.
    featureEnvironment = "test",
)
val onelo = Onelo(config, this)
```
<!-- /onelo:snippet -->

## usage
<!-- onelo:snippet sdk=features lang=android field=usage -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```kotlin
// ── IDENTIFY (skip if you use Onelo Auth — automatic). identify() is suspend. ──
lifecycleScope.launch { onelo.identify(currentUser.id) }

// Feature state is available synchronously via feature(name). Updates push in REAL TIME
// the moment an admin clicks Deploy (SSE); the SDK AUTO-refreshes when the app returns to
// the FOREGROUND and re-resolves when the signed-in user's plan changes — newly-unlocked
// features appear with NO restart. In Compose, collect onelo.features.updates to recompose.

// ── CORRECT rendering pattern (Jetpack Compose) — copy this shape ────────────
// Two rules: (1) HIDDEN or DISABLED → render NOTHING; (2) otherwise render, and let
// f.badgeLabel show the padlock / hint. A GATED feature — GREYED (dashboard "Locked",
// shows 🔒) or UPSELL (shows "Available in <plan>") — becomes tappable → the upgrade flow
// ONLY when the dashboard's "Tapping the feature opens the upgrade flow" toggle is on.
// That toggle is f.upgradeCta; the backend sends it for greyed, upsell AND coming_soon
// (any plan-gated status with a requiredPlan) — so gate the tap on upgradeCta + requiredPlan.
@Composable
fun AdvancedExport(onelo: Onelo) {
    // RECOMPOSE when features change (SSE / poll / foreground). This is the Compose
    // equivalent of Swift's @Observable OneloFeatures — WITHOUT collecting updates the UI
    // shows a stale one-time snapshot and a dashboard Deploy won't appear until the next
    // recomposition. Re-read feature() keyed on the tick.
    val tick by onelo.features.updates.collectAsState()
    val f = remember(tick) { onelo.features.feature("advanced-export") }
    val scope = rememberCoroutineScope()
    when {
        !f.isVisible() || f.isDisabled() -> Unit          // HIDDEN or DISABLED → render NOTHING
        else -> {
            // Backend's own signal: upgradeCta (the "tapping opens upgrade" toggle) is set
            // ONLY when a requiredPlan exists — for greyed, upsell AND coming_soon. Gate on
            // both (NOT on isGreyed()/isUpsell(), which would miss a coming_soon upgrade tile).
            val gatedUpgrade = f.upgradeCta && f.requiredPlan != null
            Button(
                // onelo.openUpgrade(plan) opens the hosted upgrade flow (suspend → coroutine);
                // or route to your own paywall. Non-gated taps run the real feature.
                onClick = {
                    if (gatedUpgrade) scope.launch { onelo.openUpgrade(f.requiredPlan ?: "") }
                    else openExport()
                },
                enabled = f.isEnabled() || gatedUpgrade,  // gated status WITHOUT an upgrade
                                                          // CTA (informational) stays DISABLED
            ) {
                Text(if (f.isComingSoon()) "Export (coming soon)" else "Export")
                // badgeLabel renders it all: 🔒 (greyed) · "Available in <plan>" (upsell)
                // · New / Beta / Coming Soon. Null → render nothing.
                f.badgeLabel?.let { Text("  $it") }
            }
        }
    }
}

// Escape hatch — force a REST reconcile (debounced 1/sec, suspend). Rarely needed: the SDK
// auto-refreshes on foreground and pushes over SSE.
lifecycleScope.launch { onelo.features.refresh() }

// ── Status → UI ──────────────────────────────────────────────────────────────
// Interactivity = isEnabled() (ENABLED/NEW/BETA) OR a gated feature WITH an upgrade CTA.
// The tappable-to-upgrade gate is f.upgradeCta && f.requiredPlan != null (covers greyed,
// upsell AND coming_soon). Never gate on isGreyed()/isUpsell() alone — you'd miss coming_soon.
// .isEnabled()    → ENABLED, NEW, BETA — USABLE (NEW/BETA also carry a cosmetic badge).
// .isVisible()    → false ONLY for HIDDEN. ALSO hide DISABLED via isDisabled().
// .isGreyed()     → GREYED (dashboard "Locked") — VISIBLE with 🔒; blocked. Tappable →
//                   upgrade ONLY when f.upgradeCta.
// .isUpsell()     → UPSELL — VISIBLE with "Available in <plan>"; blocked. Tappable →
//                   upgrade ONLY when f.upgradeCta.
// .isComingSoon() → COMING_SOON — VISIBLE, blocked. Tappable → upgrade when f.upgradeCta
//                   (a plan-gated coming_soon CAN carry an upgrade CTA — don't assume inert).
// .badgeLabel     → ready-made: 🔒 (greyed) · "Available in <plan>" (upsell) · New/Beta/Coming Soon · null.
// .upgradeHint    → UpgradeHint(requiredPlan, currentStatus) when plan-gated — for upgrade-prompt UI.
// .upgradeCta     → dashboard "tap opens upgrade" toggle; applies to greyed + upsell.
// .requiredPlan / .planLabel → the plan that unlocks it (slug / human label).
// .updates        → StateFlow — collectAsState() in Compose to recompose on live changes.
// .status         → ENABLED | NEW | BETA | COMING_SOON | GREYED | UPSELL | HIDDEN | DISABLED
```
<!-- /onelo:snippet -->
