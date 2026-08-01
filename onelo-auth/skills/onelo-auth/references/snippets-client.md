# Onelo Auth — client snippets

The exact integration code for every client platform, **baked into this file when
the plugin was published** — straight from `@onelo/snippets`, the one place all
Onelo code lives (it also powers the dashboard **SDK** tab and the public
**/docs**).

Read only the section for the platform you detected. Each has three parts:

- **install** — how the package is added
- **init** — creating the client (module level, exactly once)
- **usage** — signing in and gating the app on the result

Substitute `onelo_pk_live_YOUR_KEY` and `https://api.onelo.tools` with the developer's real
values before inserting. Never guess a key.

> **Do not "improve" these snippets.** Their comments carry the routing and
> gating warnings a developer will re-read months later. Insert them as they are.

Backend verification lives in [snippets-backend.md](snippets-backend.md).

---

## NPM

### install
<!-- onelo:snippet sdk=auth lang=npm field=install -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
npm install github:onelo-tools/onelo-js
```
<!-- /onelo:snippet -->

### init
<!-- onelo:snippet sdk=auth lang=npm field=init -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
import { Onelo } from '@onelo/js'

// Initialize once (e.g. in a module-level singleton)
const onelo = new Onelo({
  publishableKey: 'onelo_pk_live_YOUR_KEY',
  apiUrl: 'https://api.onelo.tools',
})
```
<!-- /onelo:snippet -->

### usage
<!-- onelo:snippet sdk=auth lang=npm field=usage -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
// Sign in — pick ONE presentation. loadAuthView() auto-routes
// waitlist / paywall-store automatically when those are enabled for your app.

// A) Embedded (the default): an in-page iframe modal. First add your site's
//    origin to Allowed Origins in the dashboard (on a test key, localhost
//    works already).
const session = await onelo.loadAuthView()

// B) Popup: a top-level window, works on any origin — like an OAuth popup.
//    Nothing to configure.
//    const session = await onelo.loadAuthView({ mode: 'popup' })

// Read the current session + react to changes
const current = await onelo.auth.getSession()
const unsub = onelo.auth.onAuthStateChange((s) => {
  console.log(s ? 'signed in' : 'signed out', s?.user)
})

// ── Paywall hard-gate (apps that sell access) ───────────────
// loadAuthView() (above) auto-routes a signed-in user with NO plan to the store
// and resolves with a session ONLY once they're entitled — if they close the
// store without buying, it resolves null. So gate your app on the RESULT; never
// render paid UI just because sign-in happened. (Unlike Electron you don't manage
// a window — your framework re-renders on state — but this is the same entitlement
// the SDK enforces server-side on every call.)
const entitled = !onelo.auth.paywallEnabled || (await onelo.auth.hasActiveAccess())
if (session && entitled) {
  // → render your app
} else {
  // Not entitled: loadAuthView() already showed the store. Keep your locked
  // screen; call onelo.loadAuthView() again to re-open it.
}

// Sign out (also revokes the session server-side)
await onelo.auth.signOut()

// ── Using your own auth system? ─────────────────────────────
// Call identify() after your login so per-user/per-plan feature targeting applies.
await onelo.identify(currentUser.id)

// ── Legal-consent gate (Terms / Privacy updates) ────────────
// When you publish a version with "Require in-app acceptance" ON in the
// dashboard (works for BOTH Terms of Service AND Privacy Policy), signed-in
// users get a BLOCKING full-screen gate they must accept — or Sign out — before
// continuing. Auto-presents on sign-in and the moment you publish (over SSE).
// "Notify only" versions never block. Nothing to wire — it's on by default.
//
// Return to sign-in on ANY sign-out (the gate's Sign out, a session revoke, or
// an expired session): re-render your app to the signed-out state here.
onelo.auth.onAuthStateChange((s) => { if (!s) { /* show your login screen */ } })
//
// Custom UI instead of the built-in gate (new Onelo({ autoPresentConsentGate: false })):
//   onelo.consent.onConsentRequired(() => { /* re-read + render your own banner */ })
//   const pending = await onelo.consent.requiredConsents()
//   await onelo.consent.acceptConsent(pending[0].versionId)
```
<!-- /onelo:snippet -->

---

## SWIFT

### install
<!-- onelo:snippet sdk=auth lang=swift field=install -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```swift
// Xcode → File → Add Package Dependencies:
// https://github.com/onelo-tools/onelo-swift
```
<!-- /onelo:snippet -->

### init
<!-- onelo:snippet sdk=auth lang=swift field=init -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```swift
import OneloSwift

// 1. Register a URL scheme for the login callback.
//    In Xcode: target → Info → URL Types → add your scheme (e.g. "myapp").
//    This lets the hosted login page redirect back to your app after sign-in.

// 2. Set up auth in your App entry point
@main struct MyApp: App {
    @StateObject var auth = OneloAuth(
        config: .init(
            publishableKey: "onelo_pk_live_YOUR_KEY",
            apiUrl: URL(string: "https://api.onelo.tools")!,
            callbackScheme: "myapp"  // replace with your URL scheme from step 1
        )
    )
    var body: some Scene {
        WindowGroup {
            // OneloAuthView handles login, signup and password reset.
            OneloAuthView(auth: auth) {
                ContentView()
                    .environmentObject(auth)
            }
        }
    }
}
```
<!-- /onelo:snippet -->

### usage
<!-- onelo:snippet sdk=auth lang=swift field=usage -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```swift
// 4. OneloAuthView always opens the centrally-hosted sign-in page —
//    on both Free and Paid plans. Email / password stays inside the
//    embedded WKWebView. Social providers (Google, GitHub, Apple — paid
//    plans) are handed off to ASWebAuthenticationSession automatically — required
//    by Apple App Store Review and gives buyers their cached
//    SSO session in Safari, so no re-login + 2FA every time. Branding
//    (colors, logo, copy) is configured in the Onelo dashboard.

// 5. Access the current user anywhere in your app
struct ContentView: View {
    @EnvironmentObject var auth: OneloAuth

    var body: some View {
        VStack {
            if let user = auth.currentSession?.user {
                Text("Hello, \(user.email ?? "user")")
                Button("Sign out") {
                    Task { try? await auth.signOut() }
                }
            }
        }
    }
}

// ── Automatic remote-logout ─────────────────────────────────
// You don't need to write any code for this — OneloAuthView observes
// auth.currentSession (which is @Published) and AUTOMATICALLY shows
// the sign-in screen when the session is cleared remotely. Delivery
// is instant (sub-second) over a Server-Sent Events channel the SDK
// opens after sign-in, with a 13-minute heartbeat fallback in case
// the long-lived connection is blocked by a corporate firewall.
// Triggers that clear the local session:
//   • Customer self-requested account deletion via /customer/portal.
//   • Refund issued AND no eligible free-tier plan to fall back to
//     (the app you sell becomes unusable for that buyer; SDK kicks them
//     out so they can re-purchase or close the app).
//   • You suspended/banned the user in the Onelo admin dashboard.
//   • Hard-delete by the 30-day account-deletion cron job.
//
// REQUIREMENTS so the auto-logout actually shows the sign-in screen:
//   1. OneloAuthView MUST be your ROOT view — above any NavigationStack,
//      TabView, or custom router. If it's nested inside, navigation pop
//      stays on whatever screen the user was on when revoked.
//      ❌ NavigationStack { OneloAuthView(auth: auth) { ContentView() } }
//      ✅ OneloAuthView(auth: auth) { NavigationStack { ContentView() } }
//   2. In ContentView observe auth via @EnvironmentObject / @ObservedObject —
//      never capture it into a local 'let' (snapshot won't update on revoke).
//      ❌ let user = auth.currentSession?.user; if user != nil { … }
//      ✅ if let user = auth.currentSession?.user { … }
//   3. If you maintain your own NavigationPath, reset it on revoke so the
//      stack doesn't outlive the session:
//        .onChange(of: auth.currentSession) { newSession in
//            if newSession == nil { navigationPath = .init() }
//        }
//
// Full working example with NavigationStack + path reset:
//
//   @main struct MyApp: App {
//       @StateObject var auth = OneloAuth(config: .init(/* … */))
//       var body: some Scene {
//           WindowGroup {
//               // OneloAuthView wraps the whole NavigationStack so that on
//               // remote-revoke the entire stack is replaced by the hosted
//               // sign-in page — no leftover pushed screens.
//               OneloAuthView(auth: auth) {
//                   RootView().environmentObject(auth)
//               }
//           }
//       }
//   }
//
//   struct RootView: View {
//       @EnvironmentObject var auth: OneloAuth
//       @State private var path = NavigationPath()
//
//       var body: some View {
//           NavigationStack(path: $path) {
//               HomeView()
//                   .navigationDestination(for: Route.self) { /* … */ }
//           }
//           // Belt-and-suspenders: reset the path on revoke so deep
//           // links don't survive the logout. OneloAuthView already
//           // re-presents the sign-in page; this just clears the stack
//           // behind it so the user lands on HomeView after re-login.
//           .onChange(of: auth.currentSession) { newSession in
//               if newSession == nil { path = NavigationPath() }
//           }
//       }
//   }
//
// If you want to show a one-time "your account was deleted" toast
// before re-presenting the sign-in screen, observe the
// auth.isUserRevoked flag (also @Published):
//
//   .onChange(of: auth.isUserRevoked) { revoked in
//       if revoked { showAccountDeletedToast() }
//   }
//
// ── AppKit / multi-window apps ──────────────────────────────
// The "OneloAuthView as root" pattern above assumes a single SwiftUI
// WindowGroup. Menu-bar utilities, floating widgets, and multi-window
// editors (separate NSWindow instances for face / bubble / chat / auth)
// don't fit that shape — your main user-facing window is NOT the one
// hosting OneloAuthView, so the SwiftUI re-render on revoke can't
// hide it. Observe auth.$currentSession on a Combine pipeline and
// drive your windows yourself:
//
//   import Combine
//
//   private var cancellables = Set<AnyCancellable>()
//   private var hadSession = false
//   private var isSigningOut = false  // set true around your own auth.signOut() call
//
//   auth.$currentSession.sink { [weak self] session in
//       guard let self else { return }
//       let has = session != nil
//       defer { self.hadSession = has }
//       // Fire only on non-nil → nil (the actual revoke moment).
//       // Skip the user-initiated Sign out path you already handle.
//       if self.hadSession && !has && !self.isSigningOut {
//           Task { @MainActor in
//               self.mainWindow.orderOut(nil)
//               self.otherWindows.forEach { $0.orderOut(nil) }
//               self.showAuthWindow()   // your NSWindow hosting OneloAuthView
//           }
//       }
//   }.store(in: &cancellables)
//
// Note: OneloAuthView hosted in a hidden / minimized / off-screen
// NSWindow will NOT react to remote-revoke. Bring the window forward
// (orderFrontRegardless / makeKeyAndOrderFront) BEFORE you expect the
// hosted page to appear.

// ── Automatic legal-consent gate ────────────────────────────
// You don't need to write any code for this. When you publish a
// material update to your Terms of Service in the Onelo dashboard,
// OneloAuthView automatically shows the updated document — in the
// SAME hosted WKWebView used for sign-in — the next time the user
// opens the app after the effective date. The user must tap
// "Accept & Continue" to reach your content(), or "Sign out".
// There is no "Decline" button: nothing to mis-tap.
//   • Terms of Service → blocks the app until accepted (legal
//     requirement in DE/PL: silent "acceptance by continued use"
//     is not valid — an explicit tap is required).
//   • Privacy Policy / DPA / Cookies → users are notified by email
//     and the app is not blocked (GDPR information duty) — UNLESS you
//     published the version with "Require acceptance", which shows the
//     same blocking gate as Terms.
//
// This automatic gate only works WHILE OneloAuthView is mounted — i.e. when it
// wraps your content() as the root view (the pattern in step 2).
//
// ⚠️ IF YOU USE OneloAuthView ONLY FOR SIGN-IN and switch to your own UI after
// login (OneloAuthView is no longer in the view tree), the automatic gate
// cannot fire — nothing is left to host it. In that case add ONE line to your
// post-login root so the gate is enforced on YOUR UI:
//
//   struct HomeView: View {
//       @EnvironmentObject var auth: OneloAuth
//       var body: some View {
//           MyAppContent()
//               .oneloConsentGate(auth: auth)   // ← blocking gate on your own UI
//       }
//   }
//
// .oneloConsentGate presents a full-cover hosted gate OVER your UI when a
// blocking version is pending (Accept & Continue / Sign out, no dismiss). It
// re-checks on appear, on app-foreground, and on the real-time SSE push — so a
// running, logged-in app shows the gate instantly when you publish.
//
//   • Terms of Service → blocks the app until accepted (legal requirement in
//     DE/PL: silent "acceptance by continued use" is not valid).
//   • Privacy / DPA / Cookies → notified by email only; never blocks the app
//     (GDPR information duty) unless you published it with "Require acceptance".
//
// Advanced (optional): read pending consents yourself, e.g. for a custom banner —
//   let pending = await auth.requiredConsents()
//   let mustAccept = pending.filter { $0.blocking }   // blocking, past effective
// Recording acceptance from your own UI:
//   try await auth.acceptConsent(versionId: requirement.versionId)

// ── If using your own auth system ───────────────────────────
// When you have your own user database, call identify() after your login so the
// Features SDK can apply per-user/per-plan targeting. Without it, targeted features
// fall back to "hidden" and you'll see a console warning at runtime.
// identify() lives on the full Onelo client (Auth + Features bundled) — OneloAuth
// on its own does not expose it. See the Features SDK docs for the call.
```
<!-- /onelo:snippet -->

---

## MACOS

### install
<!-- onelo:snippet sdk=auth lang=macos field=install -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```swift
// Xcode → File → Add Package Dependencies:
// https://github.com/onelo-tools/onelo-swift
```
<!-- /onelo:snippet -->

### init
<!-- onelo:snippet sdk=auth lang=macos field=init -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```swift
import OneloSwift

// 1. Register a URL scheme for the login callback.
//    In Xcode: target → Info → URL Types → add your scheme (e.g. "myapp").
//    This lets the hosted login page redirect back to your app after sign-in.

// 2. Set up auth in your App entry point
@main struct MyApp: App {
    @StateObject var auth = OneloAuth(
        config: .init(
            publishableKey: "onelo_pk_live_YOUR_KEY",
            apiUrl: URL(string: "https://api.onelo.tools")!,
            callbackScheme: "myapp"  // replace with your URL scheme from step 1
        )
    )
    var body: some Scene {
        WindowGroup {
            // OneloAuthView handles login, signup and password reset.
            OneloAuthView(auth: auth) {
                ContentView()
                    .environmentObject(auth)
            }
        }
    }
}
```
<!-- /onelo:snippet -->

### usage
<!-- onelo:snippet sdk=auth lang=macos field=usage -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```swift
// 4. OneloAuthView always opens the centrally-hosted sign-in page —
//    on both Free and Paid plans. Email / password stays inside the
//    embedded WKWebView. Social providers (Google, GitHub, Apple — paid
//    plans) are handed off to ASWebAuthenticationSession automatically — required
//    by Apple App Store Review and gives buyers their cached
//    SSO session in Safari, so no re-login + 2FA every time. Branding
//    (colors, logo, copy) is configured in the Onelo dashboard.

// 5. Access the current user anywhere in your app
struct ContentView: View {
    @EnvironmentObject var auth: OneloAuth

    var body: some View {
        VStack {
            if let user = auth.currentSession?.user {
                Text("Hello, \(user.email ?? "user")")
                Button("Sign out") {
                    Task { try? await auth.signOut() }
                }
            }
        }
    }
}

// ── Automatic remote-logout ─────────────────────────────────
// You don't need to write any code for this — OneloAuthView observes
// auth.currentSession (which is @Published) and AUTOMATICALLY shows
// the sign-in screen when the session is cleared remotely. Delivery
// is instant (sub-second) over a Server-Sent Events channel the SDK
// opens after sign-in, with a 13-minute heartbeat fallback in case
// the long-lived connection is blocked by a corporate firewall.
// Triggers that clear the local session:
//   • Customer self-requested account deletion via /customer/portal.
//   • Refund issued AND no eligible free-tier plan to fall back to
//     (the app you sell becomes unusable for that buyer; SDK kicks them
//     out so they can re-purchase or close the app).
//   • You suspended/banned the user in the Onelo admin dashboard.
//   • Hard-delete by the 30-day account-deletion cron job.
//
// REQUIREMENTS so the auto-logout actually shows the sign-in screen:
//   1. OneloAuthView MUST be your ROOT view — above any NavigationStack,
//      TabView, or custom router. If it's nested inside, navigation pop
//      stays on whatever screen the user was on when revoked.
//      ❌ NavigationStack { OneloAuthView(auth: auth) { ContentView() } }
//      ✅ OneloAuthView(auth: auth) { NavigationStack { ContentView() } }
//   2. In ContentView observe auth via @EnvironmentObject / @ObservedObject —
//      never capture it into a local 'let' (snapshot won't update on revoke).
//      ❌ let user = auth.currentSession?.user; if user != nil { … }
//      ✅ if let user = auth.currentSession?.user { … }
//   3. If you maintain your own NavigationPath, reset it on revoke so the
//      stack doesn't outlive the session:
//        .onChange(of: auth.currentSession) { newSession in
//            if newSession == nil { navigationPath = .init() }
//        }
//
// Full working example with NavigationStack + path reset:
//
//   @main struct MyApp: App {
//       @StateObject var auth = OneloAuth(config: .init(/* … */))
//       var body: some Scene {
//           WindowGroup {
//               // OneloAuthView wraps the whole NavigationStack so that on
//               // remote-revoke the entire stack is replaced by the hosted
//               // sign-in page — no leftover pushed screens.
//               OneloAuthView(auth: auth) {
//                   RootView().environmentObject(auth)
//               }
//           }
//       }
//   }
//
//   struct RootView: View {
//       @EnvironmentObject var auth: OneloAuth
//       @State private var path = NavigationPath()
//
//       var body: some View {
//           NavigationStack(path: $path) {
//               HomeView()
//                   .navigationDestination(for: Route.self) { /* … */ }
//           }
//           // Belt-and-suspenders: reset the path on revoke so deep
//           // links don't survive the logout. OneloAuthView already
//           // re-presents the sign-in page; this just clears the stack
//           // behind it so the user lands on HomeView after re-login.
//           .onChange(of: auth.currentSession) { newSession in
//               if newSession == nil { path = NavigationPath() }
//           }
//       }
//   }
//
// If you want to show a one-time "your account was deleted" toast
// before re-presenting the sign-in screen, observe the
// auth.isUserRevoked flag (also @Published):
//
//   .onChange(of: auth.isUserRevoked) { revoked in
//       if revoked { showAccountDeletedToast() }
//   }
//
// ── AppKit / multi-window apps ──────────────────────────────
// The "OneloAuthView as root" pattern above assumes a single SwiftUI
// WindowGroup. Menu-bar utilities, floating widgets, and multi-window
// editors (separate NSWindow instances for face / bubble / chat / auth)
// don't fit that shape — your main user-facing window is NOT the one
// hosting OneloAuthView, so the SwiftUI re-render on revoke can't
// hide it. Observe auth.$currentSession on a Combine pipeline and
// drive your windows yourself:
//
//   import Combine
//
//   private var cancellables = Set<AnyCancellable>()
//   private var hadSession = false
//   private var isSigningOut = false  // set true around your own auth.signOut() call
//
//   auth.$currentSession.sink { [weak self] session in
//       guard let self else { return }
//       let has = session != nil
//       defer { self.hadSession = has }
//       // Fire only on non-nil → nil (the actual revoke moment).
//       // Skip the user-initiated Sign out path you already handle.
//       if self.hadSession && !has && !self.isSigningOut {
//           Task { @MainActor in
//               self.mainWindow.orderOut(nil)
//               self.otherWindows.forEach { $0.orderOut(nil) }
//               self.showAuthWindow()   // your NSWindow hosting OneloAuthView
//           }
//       }
//   }.store(in: &cancellables)
//
// Note: OneloAuthView hosted in a hidden / minimized / off-screen
// NSWindow will NOT react to remote-revoke. Bring the window forward
// (orderFrontRegardless / makeKeyAndOrderFront) BEFORE you expect the
// hosted page to appear.

// ── Automatic legal-consent gate ────────────────────────────
// You don't need to write any code for this. When you publish a
// material update to your Terms of Service in the Onelo dashboard,
// OneloAuthView automatically shows the updated document — in the
// SAME hosted WKWebView used for sign-in — the next time the user
// opens the app after the effective date. The user must tap
// "Accept & Continue" to reach your content(), or "Sign out".
// There is no "Decline" button: nothing to mis-tap.
//   • Terms of Service → blocks the app until accepted (legal
//     requirement in DE/PL: silent "acceptance by continued use"
//     is not valid — an explicit tap is required).
//   • Privacy Policy / DPA / Cookies → users are notified by email
//     and the app is not blocked (GDPR information duty) — UNLESS you
//     published the version with "Require acceptance", which shows the
//     same blocking gate as Terms.
//
// This automatic gate only works WHILE OneloAuthView is mounted — i.e. when it
// wraps your content() as the root view (the pattern in step 2).
//
// ⚠️ IF YOU USE OneloAuthView ONLY FOR SIGN-IN and switch to your own UI after
// login (OneloAuthView is no longer in the view tree), the automatic gate
// cannot fire — nothing is left to host it. In that case add ONE line to your
// post-login root so the gate is enforced on YOUR UI:
//
//   struct HomeView: View {
//       @EnvironmentObject var auth: OneloAuth
//       var body: some View {
//           MyAppContent()
//               .oneloConsentGate(auth: auth)   // ← blocking gate on your own UI
//       }
//   }
//
// .oneloConsentGate presents a full-cover hosted gate OVER your UI when a
// blocking version is pending (Accept & Continue / Sign out, no dismiss). It
// re-checks on appear, on app-foreground, and on the real-time SSE push — so a
// running, logged-in app shows the gate instantly when you publish.
//
//   • Terms of Service → blocks the app until accepted (legal requirement in
//     DE/PL: silent "acceptance by continued use" is not valid).
//   • Privacy / DPA / Cookies → notified by email only; never blocks the app
//     (GDPR information duty) unless you published it with "Require acceptance".
//
// Advanced (optional): read pending consents yourself, e.g. for a custom banner —
//   let pending = await auth.requiredConsents()
//   let mustAccept = pending.filter { $0.blocking }   // blocking, past effective
// Recording acceptance from your own UI:
//   try await auth.acceptConsent(versionId: requirement.versionId)

// ── If using your own auth system ───────────────────────────
// When you have your own user database, call identify() after your login so the
// Features SDK can apply per-user/per-plan targeting. Without it, targeted features
// fall back to "hidden" and you'll see a console warning at runtime.
// identify() lives on the full Onelo client (Auth + Features bundled) — OneloAuth
// on its own does not expose it. See the Features SDK docs for the call.
```
<!-- /onelo:snippet -->

---

## ELECTRON

### install
<!-- onelo:snippet sdk=auth lang=electron field=install -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
npm install github:onelo-tools/onelo-electron#semver:*
```
<!-- /onelo:snippet -->

### init
<!-- onelo:snippet sdk=auth lang=electron field=init -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
import { Onelo } from '@onelo/electron'
import { app, BrowserWindow } from 'electron'

// 1. Register your deep-link protocol (do this before app is ready)
//    Replace 'myapp' with your own scheme — must be unique to your app.
app.setAsDefaultProtocolClient('myapp')

// 2. Initialize the SDK in your Electron main process. ONE instance owns auth,
//    features, consent and paywall — they SHARE the session. Use onelo.auth
//    everywhere: a separate `new OneloElectronAuth(...)` is a DIFFERENT session,
//    so the consent gate's "Sign out" would fire on onelo.auth and your
//    onSessionChange handler (on the other instance) would never run — the app
//    would look like sign-out did nothing.
const onelo = new Onelo({
  publishableKey: 'onelo_pk_live_YOUR_KEY',
  apiUrl: 'https://api.onelo.tools',
  protocol: 'myapp', // must match the scheme above
  // Your app id (same on macOS + Windows). REQUIRED for codesign attestation:
  // the SDK sends it on every request and a code-signed build without it is
  // rejected (bundle_id_mismatch) once enforcement is active. Match your
  // electron-builder appId / CFBundleIdentifier.
  bundleId: 'com.company.app',
})
const auth = onelo.auth // shorthand — the SAME instance the consent gate signs out
```
<!-- /onelo:snippet -->

### usage
<!-- onelo:snippet sdk=auth lang=electron field=usage -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
// 3. Auto-show auth on app launch — no button needed.
//
// This is your paywall hard-gate: a user who signs in but never completes a
// purchase never reaches your app — the window stays hidden until they hold
// access. (Swift's OneloAuthView gates its content for you; on web you gate on
// the RESULT of loadAuthView(); Electron can't touch your BrowserWindow, so
// YOU own the gate — this is it.)
//
// ONE gate decides whether the app window may appear. "Allowed in" = signed in
// AND, on a paywalled app, holding active access. Do NOT treat presentAuthWindow()
// returning as proof of entry: closing the store without buying, a declined
// consent gate, or an OAuth hand-off to the system browser all resolve WITHOUT a
// session — so re-check real state, never the return value. (Electron, unlike the
// native SDKs, has no auth wrapper that re-shows login for you; this is it.)
let mainWindow            // created on ready (below)
let gating = false        // re-entrancy guard — never stack two auth windows

async function enforceGate() {
  if (!mainWindow || gating) return
  gating = true
  try {
    await auth.whenReady().catch(() => {})  // config loaded → paywall flag is accurate
    const allowedIn = async () => {
      const session = await auth.getSession()
      if (!session) return false                              // not signed in
      // Only paywalled apps require active access; others enter on a session alone.
      return !auth.isPaywallEnabled() || (await auth.hasActiveAccess())
    }
    if (await allowedIn()) { mainWindow.show(); return }
    mainWindow.hide()
    // null = standalone auth window (no app window behind it). May resolve with
    // NO session (store closed without buying / OAuth punted to the browser) —
    // that's why we re-check below instead of showing unconditionally.
    // presentAuthWindow shows its own graceful "Try again" window on a flow
    // failure (attestation / offline), so it won't throw for those — but a
    // revoked key still throws. Catch it so enforceGate never leaks an unhandled
    // rejection that would leave mainWindow permanently hidden (blank app).
    await auth.presentAuthWindow(null)
    if (await allowedIn()) mainWindow.show()   // still not allowed in → stay hidden
  } catch (err) {
    console.warn('[Onelo] auth gate error — keeping app window hidden:', err)
  } finally {
    gating = false
  }
}

// Re-run the gate on launch, on every sign-in/out (revoke, session expiry, a
// declined consent gate — saveSession/signOut both fire this), and after an
// OAuth deep-link exchange.
auth.onSessionChange(() => { void enforceGate() })

// OAuth finishes in the system browser and returns via your deep-link scheme.
// Exchanging it fires onSessionChange → enforceGate reveals the app. Registered
// at top level so a cold-start deep-link isn't missed.
app.on('open-url', (_event, url) => {
  // A present-but-failed code exchange throws — swallow it so the gate stays put
  // (the user retries); onSessionChange reveals the app only on a real session.
  auth.handleDeepLink(url).catch(() => {})
})

app.whenReady().then(async () => {
  mainWindow = new BrowserWindow({ show: false /* your options */ })
  mainWindow.loadFile('index.html')  // load now; the gate decides WHEN to show
  await enforceGate()
})

// Sign out — the onSessionChange handler above returns the app to sign-in.
await auth.signOut()

// ── If using your own auth system ───────────────────────────
// When you have your own user database, features SDK still needs to know who is
// signed in. Call identify() after your login so per-user/per-plan targeting works.
// Without it, targeted features fall back to "hidden" and you'll see a console warning.
await onelo.identify(currentUser.id)

// ── Legal-consent gate (Terms / Privacy updates) ────────────
// Uses the full Onelo SDK — const onelo = new Onelo({ ... }) — which exposes
// onelo.consent (same instance as features/paywall). When you publish a version
// with "Require in-app acceptance" ON in the dashboard — this works for BOTH
// Terms of Service AND Privacy Policy — signed-in users get a BLOCKING gate they
// must accept to keep using the app; Decline signs them out. "Notify only"
// versions never block.
//
// REQUIRED for a true hard block: register your main window so the gate opens
// MODALLY over the app. Without it the gate can't block the app and the SDK logs
// a warning (Electron can't hide your window for you the way native SDKs do).
onelo.consent.setGateParent(mainWindow)
// The gate then auto-presents on sign-in AND the instant you publish a blocking
// version (pushed live over SSE) — no polling, no restart.
//
// Belt-and-braces (recommended): also gate your OWN post-login UI before showing it.
const pending = await onelo.consent.requiredConsents()
if (pending.some(c => c.blocking)) {
  await onelo.consent.presentGateIfNeeded(mainWindow)   // resolves after Accept
}
//
// Custom UI instead of the hosted gate (new Onelo({ autoPresentConsentGate: false })):
//   onelo.consent.onConsentRequired(() => { /* re-read + render your own banner */ })
//   const p = await onelo.consent.requiredConsents()
//   await onelo.consent.acceptConsent(p[0].versionId)
```
<!-- /onelo:snippet -->

---

## REACTNATIVE

### install
<!-- onelo:snippet sdk=auth lang=reactnative field=install -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
npm install github:onelo-tools/onelo-react-native react-native-keychain react-native-webview react-native-get-random-values
```
<!-- /onelo:snippet -->

### init
<!-- onelo:snippet sdk=auth lang=reactnative field=init -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
// PKCE needs a cryptographically-secure RNG. React Native has no built-in Web
// Crypto, so import this polyfill as the FIRST line of your app ENTRY (index.js),
// before anything else — it backs crypto.getRandomValues with the platform CSPRNG
// (iOS SecRandomCopyBytes / Android SecureRandom). Without it, sign-in throws.
import 'react-native-get-random-values'
import { Onelo } from '@onelo/react-native'

// ONE instance owns auth, features, consent, store and paywall — they SHARE the
// session. Always use onelo.auth: a second `new Onelo(...)` is a DIFFERENT
// session, so the consent gate's "Sign out" and your auth listener would run on
// different instances and the app would look like sign-out did nothing.
const onelo = new Onelo({
  publishableKey: 'onelo_pk_live_YOUR_KEY',
  apiUrl: 'https://api.onelo.tools',
  callbackScheme: 'myapp', // pick a scheme UNIQUE to your app + register it natively (see usage below)
  // Your app id / package name, sent as X-Bundle-Id for bundle attestation.
  // iOS auto-derives it from the native module, but ANDROID needs it here —
  // without it, requests are rejected (bundle_id_mismatch) once enforcement is on.
  bundleId: 'com.company.app',
})
```
<!-- /onelo:snippet -->

### usage
<!-- onelo:snippet sdk=auth lang=reactnative field=usage -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
import { OneloAuthGate, ConsentGateModal } from '@onelo/react-native'

// ── REQUIRED NATIVE SETUP: register your deep-link scheme ────
// Social sign-in (paid plans; and store upgrades / some Stripe redirects) hands off to the
// SYSTEM BROWSER, which returns to your app via the callbackScheme you set above.
// Register the SAME scheme natively or the browser can't reopen the app and sign-in
// never finishes (you'd be stuck on a "Return to the app" page):
//   • Android — android/app/src/main/AndroidManifest.xml, inside your <activity>:
//       <intent-filter>
//         <action android:name="android.intent.action.VIEW" />
//         <category android:name="android.intent.category.DEFAULT" />
//         <category android:name="android.intent.category.BROWSABLE" />
//         <data android:scheme="myapp" />
//       </intent-filter>
//   • iOS — ios/<App>/Info.plist → URL Types → URL Schemes: myapp
// Plain in-app WebView sign-in works without this; the system-browser return needs it.

// ── Flagship: declarative auth + paywall gate ───────────────
// OneloAuthGate is the React Native analogue of Swift's OneloAuthView. Wrap your
// app in it and it drives the CENTRALLY-HOSTED sign-in page automatically — no
// button, no custom form, no branded pre-auth screen (just a neutral background
// while it resolves). It reveals children ONLY when the user is signed in AND
// (the app has no paywall OR they hold active access). Mount <ConsentGateModal>
// near the root so a blocking Terms/Privacy update can draw over the app.
function App() {
  return (
    <>
      <OneloAuthGate onelo={onelo}>
        {/* Your app — shown only when signed in AND (no paywall OR entitled) */}
        <RootNavigator />
      </OneloAuthGate>
      <ConsentGateModal onelo={onelo} />
    </>
  )
}

// ── Paywall hard-gate (apps that sell access) ───────────────
// This is the entitlement wall. OneloAuthGate reveals children ONLY when signed
// in AND (no paywall OR entitled). For a paywalled app it routes a
// signed-in-but-unentitled user to the hosted STORE and keeps children hidden
// until they buy — closing the store WITHOUT purchasing leaves them on the
// sign-in / store surface, never inside your app. This is the SAME entitlement
// the backend enforces server-side on every call. If you build your OWN gate
// instead of OneloAuthGate, check both yourself — never render paid UI just
// because sign-in happened:
const paywalled = onelo.auth.isPaywallEnabled()
const entitled = !paywalled || (await onelo.auth.hasActiveAccess())
if (entitled) {
  // → render your app
} else {
  // Not entitled — send them to the hosted store (see the Store tab):
  //   await onelo.store.open()
}

// ── Sign out / return to login ──────────────────────────────
// signOut() clears the session (and revokes it server-side); OneloAuthGate
// re-shows the hosted sign-in page automatically. Subscribe to auth changes to
// react to REMOTE revokes too (refund, ban, account-deletion cron) — the session
// becomes null and OneloAuthGate returns to sign-in with no code from you:
const unsub = onelo.auth.onAuthStateChange((session) => {
  if (!session) { /* signed out — OneloAuthGate shows sign-in again */ }
})
await onelo.auth.signOut()

// ── Legal-consent gate (Terms / Privacy updates) ────────────
// <ConsentGateModal onelo={onelo} /> (mounted above) auto-presents a BLOCKING
// full-screen gate — the hosted document in a WebView — when you publish a
// version with "Require in-app acceptance" ON (works for BOTH Terms of Service
// AND Privacy Policy). The user must Accept & Continue, or Sign out. It checks on
// sign-in, on every app-foreground, and the instant you publish (over SSE) — no
// polling, no restart. "Notify only" versions never block. Nothing else to wire.
//
// Custom UI instead of the hosted gate (opt out of auto-present):
//   const onelo = new Onelo({ ..., autoPresentConsentGate: false })
//   onelo.consent.onConsentRequired(() => { /* re-read + render your own banner */ })
//   const pending = await onelo.consent.requiredConsents()
//   const mustAccept = pending.filter(c => c.blocking)   // blocking, past effective
//   await onelo.consent.acceptConsent(pending[0].versionId)

// ── If using your own auth system ───────────────────────────
// When you have your own user database, call identify() after your login so the
// features SDK can apply per-user/per-plan targeting. Without it, targeted
// features fall back to "hidden" and you'll see a console warning at runtime.
await onelo.identify(currentUser.id)

// ── Manual mount (lower-level, without OneloAuthGate) ───────
// Prefer OneloAuthGate above. If you must drive the hosted flow yourself, mount
// <AuthModal> and open it with loadAuthView() (auto-routes waitlist / paywall
// store); useModalState() re-renders when the modal opens/closes:
//   import { AuthModal, useModalState } from '@onelo/react-native'
//   const modal = useModalState(onelo.auth)
//   <AuthModal visible={modal.visible} hostedUrl={modal.url}
//              onResult={modal.onResult ?? (() => {})} />
//   const session = await onelo.auth.loadAuthView()
```
<!-- /onelo:snippet -->

---

## ANDROID

### install
<!-- onelo:snippet sdk=auth lang=android field=install -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```kotlin
// build.gradle.kts (app/module)
// mavenCentral() is standard in Android projects — add it to settings.gradle.kts
// dependencyResolutionManagement { repositories { mavenCentral() } } if missing.
implementation("tools.onelo:onelo-android:1.+")
```
<!-- /onelo:snippet -->

### init
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

### usage
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

---

## FLUTTER

### install
<!-- onelo:snippet sdk=auth lang=flutter field=install -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```dart
# pubspec.yaml:
dependencies:
  onelo:
    git:
      url: https://github.com/onelo-tools/onelo-flutter
      ref: main
```
<!-- /onelo:snippet -->

### init
<!-- onelo:snippet sdk=auth lang=flutter field=init -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```dart
import 'package:onelo/onelo.dart';

// Email/password hosted sign-in works out of the box — the in-app WebView
// captures the callback. You ONLY need to register the callback scheme for
// SOCIAL sign-in (signInWithOAuth — paid plans), which opens a system browser:
//   iOS → Info.plist: add "myapp" under CFBundleURLTypes → CFBundleURLSchemes
//   Android → AndroidManifest.xml, register flutter_web_auth_2's CallbackActivity
//     (a plain MainActivity intent-filter is NOT enough — OAuth will hang):
//     <activity android:name="com.linusu.flutter_web_auth_2.CallbackActivity"
//               android:exported="true">
//       <intent-filter>
//         <action android:name="android.intent.action.VIEW"/>
//         <category android:name="android.intent.category.DEFAULT"/>
//         <category android:name="android.intent.category.BROWSABLE"/>
//         <data android:scheme="myapp"/>
//       </intent-filter>
//     </activity>

// Initialize once (e.g. in main.dart)
final onelo = Onelo(
  publishableKey: 'onelo_pk_live_YOUR_KEY',
  apiUrl: 'https://api.onelo.tools',
  callbackScheme: 'myapp',
);
```
<!-- /onelo:snippet -->

### usage
<!-- onelo:snippet sdk=auth lang=flutter field=usage -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```dart
// Wrap your app — shows hosted sign-in automatically when not signed in
void main() {
  runApp(
    MaterialApp(
      home: OneloAuthView(
        auth: onelo.auth,
        child: MyHomePage(),
      ),
    ),
  );
}

// Access current user
final user = onelo.auth.currentSession?.user;

// Sign out (revokes server-side too, not just locally)
await onelo.auth.signOut();

// ── Paid access (entitlement) ───────────────────────────────
// hasActiveAccess reflects the user's paid entitlement (false when signed out).
if (onelo.auth.hasActiveAccess) { /* unlock paid features */ }
// Re-check after a purchase / cancellation:
await onelo.auth.revalidateEntitlement();

// ── Automatic remote logout ─────────────────────────────────
// A server-side revoke (ban, account deletion, refund lapse) clears the session
// in REAL TIME (SSE, with a 13-min heartbeat fallback) — zero code. onelo.auth is
// a ChangeNotifier: route to sign-in when currentSession becomes null;
// onelo.auth.isUserRevoked is true after a remote revoke (show a toast).

// If using Onelo Auth — features identify automatically after sign-in.
// If using your own auth system — call identify() manually after login:
await onelo.identify(currentUser.id);

// ── Legal-consent gate (Terms / Privacy updates) ────────────
// Nest OneloConsentGate INSIDE OneloAuthView.child, so a declined consent →
// sign-out falls back to the sign-in screen. It hard-blocks the app whenever a
// BLOCKING legal version is pending (publish a version with "Require in-app
// acceptance" — typically Terms; Privacy only if you turn it on). The user must
// Accept (on the hosted page) or Sign out — no dismiss. A live publish
// auto-presents in REAL TIME via SSE (no polling, no manual trigger), and only
// ONE gate ever shows even if several are mounted.
// MaterialApp(
//   home: OneloAuthView(
//     auth: onelo.auth,
//     child: OneloConsentGate(
//       consent: onelo.consent,
//       child: HomeScreen(),
//     ),
//   ),
// );
//
// Advanced / build your own UI (optional) — onelo.consent is a ChangeNotifier:
//   final pending = await onelo.consent.requiredConsents(); // [] if none due
//   if (onelo.consent.hasBlockingConsent) { /* render your own blocking screen */ }
//   await onelo.consent.acceptConsent(onelo.consent.pendingBlockingConsent!.versionId);
```
<!-- /onelo:snippet -->
