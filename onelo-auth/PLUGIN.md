# onelo-auth

Adds **Onelo Auth** — hosted sign-in / sign-up — to an app.

Run `/onelo-auth` (or just ask Claude to add Onelo authentication). The skill
detects your platform, fetches the *current* snippet from the Onelo snippet API
— never a copy baked into the plugin — proposes where it goes, and wires it in
after you approve.

Covers web, Swift (iOS/macOS), Electron, React Native, Android, Flutter, plus the
backend token-verification snippets for Python, Node and PHP.

Requires an app in the [Onelo dashboard](https://onelo.tools) and its publishable
key (SDK tab).
