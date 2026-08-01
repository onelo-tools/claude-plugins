# Android — Kotlin

Baked into this plugin at publish time from `@onelo/snippets` — the same
source the dashboard **SDK** tab and **/docs** render from. Insert it as-is;
never write an Onelo SDK call from memory and never adapt another platform's
snippet.

## install
<!-- onelo:snippet sdk=auth lang=android field=install -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```kotlin
// build.gradle.kts (app/module)
// mavenCentral() is standard in Android projects — add it to settings.gradle.kts
// dependencyResolutionManagement { repositories { mavenCentral() } } if missing.
implementation("tools.onelo:onelo-android:1.+")
```
<!-- /onelo:snippet -->

## init
<!-- onelo:snippet sdk=auth lang=android field=init -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```kotlin
import com.onelo.android.Onelo
import com.onelo.android.OneloConfig

// Initialize once (e.g. in Application.onCreate or a singleton)
val onelo = Onelo(
    config = OneloConfig(
        publishableKey = "onelo_pk_live_YOUR_KEY",
        apiUrl = "https://api.onelo.tools",
    ),
    context = applicationContext,
)
```
<!-- /onelo:snippet -->

## usage
<!-- onelo:snippet sdk=auth lang=android field=usage -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```kotlin
// 1. Register launcher once in your Activity onCreate
val launcher = onelo.auth.registerLauncher(this) { session ->
    if (session != null) {
        // user signed in
    }
}

// 2. Trigger sign-in (e.g. on a button click). The hosted page opens in a WebView and
//    returns the session automatically — NO AndroidManifest / deep-link setup needed
//    for email/password sign-in (only social OAuth, further down, needs a scheme).
lifecycleScope.launch {
    onelo.auth.loadAuthView(launcher)
}

// Get current session (getSession is a suspend fn — call from a coroutine)
lifecycleScope.launch {
    val session = onelo.auth.getSession()
    // session?.user → OneloUser
}

// Subscribe to auth changes
lifecycleScope.launch {
    onelo.auth.onAuthStateChange().collect { session ->
        // session is OneloSession? — null means signed out
    }
}

// Sign out (revokes server-side too, not just locally)
lifecycleScope.launch { onelo.auth.signOut() }

// ── Paid access (entitlement) ───────────────────────────────
// hasActiveAccess reflects the user's paid entitlement (false when signed out).
if (onelo.auth.hasActiveAccess) { /* unlock paid features */ }
// Re-check after a purchase / cancellation (suspend):
lifecycleScope.launch { onelo.auth.revalidateEntitlement() }

// ── Automatic remote logout ─────────────────────────────────
// A server-side revoke (ban, account deletion, refund lapse) clears the session in
// REAL TIME over SSE (with a heartbeat fallback) — zero code. onAuthStateChange
// emits null; onelo.auth.isUserRevoked is true after a remote revoke (show a toast).

// ── Social sign-in (OAuth — paid plans) ─────────────────────
// 1) Set callbackScheme in OneloConfig (e.g. "myapp") and register
//    <scheme>://callback in your AndroidManifest on a deep-link Activity.
// 2) Forward the redirect from that Activity:
//      override fun onNewIntent(intent: Intent) { onelo.auth.handleOAuthRedirect(intent.data) }
// 3) Open the provider (google / github / apple) in the system browser:
//      lifecycleScope.launch { onelo.auth.signInWithOAuth(this@MainActivity, "google") }
//    onelo.auth.oauthProviders lists which providers are enabled for this app.

// ── If using your own auth system ──────────────────────────
// When you have your own user database, call identify() after your login so the
// features SDK can apply per-user/per-plan targeting. Without it, targeted features
// fall back to "hidden" and you'll see a Logcat warning at runtime.
// lifecycleScope.launch { onelo.identify(currentUser.id) }

// ── Legal-consent gate (Terms / Privacy updates) ────────────
// Register a launcher ONCE in onCreate — that's all. When you publish a version with
// "Require in-app acceptance" ON (works for BOTH Terms of Service AND Privacy Policy),
// the SDK AUTOMATICALLY shows a full-screen blocking gate the user must accept, or
// Sign out. It auto-presents on sign-in, on the live update push, and on resume — no
// polling, no manual call. (Opt out with OneloConfig(autoPresentConsentGate = false).)
// onelo.consent.registerLauncher(this, onelo.auth) { result ->
//     if (result.declined) { /* SDK already signed the user out — route to sign-in */ }
// }
```
<!-- /onelo:snippet -->
