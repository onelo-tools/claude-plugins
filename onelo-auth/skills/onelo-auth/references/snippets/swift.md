# Swift — iOS and macOS

Baked into this plugin at publish time from `@onelo/snippets` — the same
source the dashboard **SDK** tab and **/docs** render from. Insert it as-is;
never write an Onelo SDK call from memory and never adapt another platform's
snippet.

> One snippet covers iOS and macOS — the SDK code is identical.

## install
<!-- onelo:snippet sdk=auth lang=macos field=install -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```swift
// Xcode → File → Add Package Dependencies:
// https://github.com/onelo-tools/onelo-swift
```
<!-- /onelo:snippet -->

## init
<!-- onelo:snippet sdk=auth lang=macos field=init -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
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

## usage
<!-- onelo:snippet sdk=auth lang=macos field=usage -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
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
