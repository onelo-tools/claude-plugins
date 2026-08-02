# Swift — iOS and macOS

Baked into this plugin at publish time from `@onelo/snippets` — the same
source the dashboard **SDK → Customer Portal** tab and **/docs** render from.
Insert it as-is; never write an Onelo SDK call from memory and never adapt
another platform's snippet.

## install
<!-- onelo:snippet sdk=customer-portal lang=swift field=install -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```swift
// Xcode → File → Add Package Dependencies:
// https://github.com/onelo-tools/onelo-swift
```
<!-- /onelo:snippet -->

## init
<!-- onelo:snippet sdk=customer-portal lang=swift field=init -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```swift
// ────────────────────────────────────────────────────────────────
// PRECONDITION — Onelo SDK already initialised
// ────────────────────────────────────────────────────────────────
// This snippet assumes you've already followed the Auth setup:
//   • URL scheme registered in Info.plist
//   • OneloAuth instantiated as @StateObject in your App entry
//   • User is signed in (auth.currentSession != nil)
// See the Auth tab if you haven't done that yet.

import OneloSwift
import AuthenticationServices
```
<!-- /onelo:snippet -->

## usage
<!-- onelo:snippet sdk=customer-portal lang=swift field=usage -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```swift
// ────────────────────────────────────────────────────────────────
// STEP 1 — Add a "Manage subscription" button anywhere in your UI
// ────────────────────────────────────────────────────────────────
// The portal is a hosted page on Onelo — branded with your app's
// colors and logo from the dashboard. It handles cancel, refund
// (within the refund window you set per product), and shows past
// invoices. The user returns to your app via your callback scheme.

struct ManageSubscriptionButton: View {
    @EnvironmentObject var auth: OneloAuth

    var body: some View {
        Button("Manage subscription") {
            Task {
                do {
                    try await auth.openCustomerPortal(from: ContextProvider.shared)
                } catch OneloError.notAuthenticated {
                    // User signed out — route them to sign-in
                } catch {
                    print("Portal error: \(error)")
                }
            }
        }
    }
}

// ────────────────────────────────────────────────────────────────
// STEP 2 — Provide a presentation anchor for the WebAuth session
// ────────────────────────────────────────────────────────────────
// ASWebAuthenticationSession needs a window to attach to. One shared
// provider works for the whole app — declare it once.

final class ContextProvider: NSObject, ASWebAuthenticationPresentationContextProviding {
    static let shared = ContextProvider()
    func presentationAnchor(for session: ASWebAuthenticationSession) -> ASPresentationAnchor {
        #if os(macOS)
        return NSApplication.shared.windows.first ?? ASPresentationAnchor()
        #else
        return UIApplication.shared.connectedScenes
            .compactMap { ($0 as? UIWindowScene)?.keyWindow }
            .first ?? ASPresentationAnchor()
        #endif
    }
}

// ────────────────────────────────────────────────────────────────
// ALTERNATIVE A — open in the user's real system browser
// ────────────────────────────────────────────────────────────────
// Menubar / accessory (LSUIElement) apps can't host an ASWebAuth window,
// and some flows prefer the real browser (real domain + saved cards /
// password manager). Use openCustomerPortalInBrowser() instead — then wire
// the return deep-link back so hard account events (deletion / revoke)
// clear the local session instantly:
//
//   try await auth.openCustomerPortalInBrowser()
//
//   // SwiftUI — on your top-level scene / view:
//   .onOpenURL { url in auth.handlePortalCallback(url) }

// ────────────────────────────────────────────────────────────────
// ALTERNATIVE B — stay fully in-app (embedded WebView sheet)
// ────────────────────────────────────────────────────────────────
// Present OneloCustomerPortalView — it fetches the URL, renders the portal
// in a WKWebView, clears the session on hard events, and calls onDismiss
// when the user is done:
//
//   .sheet(isPresented: $showPortal) {
//       OneloCustomerPortalView(auth: auth) { showPortal = false }
//   }
```
<!-- /onelo:snippet -->
