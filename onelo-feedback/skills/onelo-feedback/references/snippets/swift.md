# Swift — iOS and macOS

Baked into this plugin at publish time from `@onelo/snippets` — the same
source the dashboard **SDK → Feedback** tab and **/docs** render from. Insert it
as-is; never write an Onelo SDK call from memory and never adapt another
platform's snippet.

## install
<!-- onelo:snippet sdk=feedback lang=swift field=install -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```swift
// Xcode → File → Add Package Dependencies:
// https://github.com/onelo-tools/onelo-swift
```
<!-- /onelo:snippet -->

## init
<!-- onelo:snippet sdk=feedback lang=swift field=init -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```swift
import OneloSwift

// Feedback lives on the top-level Onelo object (not OneloAuth).
let onelo = Onelo(
  publishableKey: "onelo_pk_live_YOUR_KEY",
  baseURL: URL(string: "https://api.onelo.tools")!
)
```
<!-- /onelo:snippet -->

## usage
<!-- onelo:snippet sdk=feedback lang=swift field=usage -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```swift
// SwiftUI — attach .feedbackSheet ONCE to your root/content view. This is
// REQUIRED: open() only sets isPresented; without the modifier NOTHING shows.
// Hold `onelo` as a @StateObject / shared instance so the sheet observes it.
YourRootView()
    .feedbackSheet(onelo.feedback)

// Then trigger open() from anywhere (a button, menu item, …):
// Anonymous — best for public-facing apps; the report isn't tied to a person
Task { try await onelo.feedback.open() }
Task { try await onelo.feedback.open(options: .init(type: "bug")) }

// Identified — INSIDE your app, pass the signed-in user so you know who reported it
Task { try await onelo.feedback.open(options: .init(type: "bug", area: "checkout", userId: currentUser.id)) }

// macOS / AppKit (no SwiftUI view, so no .feedbackSheet) — opens a standalone
// window that shows immediately and loads the form inside:
onelo.feedback.openAsWindow(options: .init(type: "bug"))
```
<!-- /onelo:snippet -->
