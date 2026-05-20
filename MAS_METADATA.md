# Mac App Store metadata for Claude Recall

Drop-in copy for App Store Connect when the time comes to submit.

## Basics

- **App name:** Claude Recall
- **Subtitle (max 30 chars):** Native viewer for CLAUDE.md
- **Bundle ID:** `com.pelatari.claude-recall`
- **Primary category:** Developer Tools
- **Secondary category:** Productivity
- **Age rating:** 4+ (no objectionable content)
- **Price:** Free, or $4.99 one-time (decide before submission)
- **Pricing tier (if paid):** Tier 5 ($4.99)

## Promotional text (170 chars — shown at top of listing, can be updated without re-review)

> Browse and edit the CLAUDE.md, AGENTS.md, and AI-agent memory files scattered across your home directory. GitHub-style markdown, file watching, JSON viewer.

## Description (4000 chars max)

```
Claude Recall is the native macOS app for browsing every CLAUDE.md, AGENTS.md, and AI-agent memory file your AI coding assistants have written across your projects — beautifully rendered, instantly searchable, fast.

Built for engineers who use Claude Code, Codex, Cursor, Gemini, Continue, GitHub Copilot, Aider, or OpenClaw daily — and who are tired of opening these files in TextEdit one at a time.

KEY FEATURES

• Auto-discovery — Detects memory directories for 8 popular AI agents (Claude Code, Codex, Cursor, Gemini, Continue, GitHub Copilot, Aider, OpenClaw) and any custom folder you add.

• GitHub-style markdown — Powered by MarkdownUI. Code blocks, tables, lists, blockquotes, all rendered exactly like you see them on GitHub.

• Live file watching — When Claude (or any agent) writes a new memory entry, the app refreshes instantly. No reload button.

• Edit mode — Press ⌘E to switch from reading to editing. Syntax highlighting for headers, bold, italic, code, links. Auto-saves 2 seconds after the last keystroke, or ⌘S to save now.

• In-page find — ⌘F. Real character-level highlighting, not just "scroll to nearest match."

• Today panel — Highlights today's `memory/YYYY-MM-DD.md` if present, for projects that log per-day notes.

• JSON / JSONL viewer — Pretty-prints Claude Code's `~/.claude/projects/*.json` NDJSON session telemetry. Chunked rendering so multi-MB transcripts don't crash anything.

• URL scheme + CLI — Other tools can open files in Claude Recall via `clauderecall://open?path=…&heading=…` or the `crec` CLI shell script. A companion Claude Code plugin ships the upstream `/aimr` slash command (upstream-only).

• Dark and light themes — Follows the system appearance.

• Native — Built in Swift + SwiftUI. No Electron, no web view. 3 MB universal binary (Apple Silicon + Intel).

WHO THIS IS FOR

• Engineers who already use Claude Code daily and want a single pane of glass for all their CLAUDE.md files
• Tech leads rolling out memory conventions across a team
• Anyone curious what Claude has been writing about their projects in the background

WHAT IT WON'T DO

• Talk to your AI agents directly — this is a *viewer*, not a chat client
• Sync over iCloud — file watching is local only
• Convert memory files between agent formats — pull requests welcome on GitHub

The macOS app pairs with a free iPhone reader on the same App Store.
```

## Keywords (100 chars total, comma-separated)

```
claude,claude code,codex,cursor,gemini,markdown,memory,ai agent,llm,viewer
```

## What's New (release notes — first submission)

```
First release on the App Store. Claude Recall has been on GitHub for several months as version 0.4.2; this is the same code, sandboxed for the App Store. See LICENSING.md in the repo for details.
```

## URLs

- **Marketing URL:** https://chrispelatari.github.io/Claude-recall/
- **Support URL:** https://github.com/ChrisPelatari/Claude-recall/issues
- **Privacy policy URL:** https://chrispelatari.github.io/Claude-recall/privacy.html (write this before submission — see below)

## Privacy policy (must be at the URL above before submission)

Single page covering:
- App is local-only
- Reads files in user-selected folders (sandbox)
- Persists folder bookmarks and theme preference in UserDefaults
- Optional iCloud KVStore sync for app settings (no document content)
- No analytics, no telemetry, no third-party SDKs
- No network calls
- No data collection of any kind

## App Privacy ("Nutrition label") answers in App Store Connect

- Data collected: **None**
- Data linked to user: **None**
- Tracking: **No**

## Screenshots (required: at least one set; recommended: 5 at 2560×1600)

1. Main window with sidebar + markdown preview (the existing `home.png` works, but at MAS resolution)
2. Edit mode with line numbers
3. Today panel highlighted
4. JSON viewer rendering a Claude session telemetry file
5. Dark mode of (1)

App Store automatically generates the iPhone screenshots from the App Preview videos, or you can supply separately.

## Review notes (private to Apple App Review)

```
Hello reviewer,

This app reads markdown and JSON files from user-selected folders. It has no network access, no telemetry, no third-party SDKs.

It has been distributed as a GitHub release under GPL-3.0 since [date] (currently v0.4.2 as of this submission). The source code submitted here is identical to the public repository: https://github.com/ChrisPelatari/Claude-recall. See LICENSING.md in the repo for the dual-license arrangement.

To test:
1. Launch the app.
2. ⌘O to open a folder containing markdown or JSON files. Any folder works — for a quick test, point it at the macOS Documentation folder or any project's docs directory.
3. Click a .md file in the sidebar to see it rendered. ⌘E to enter edit mode.

No login required. No paywall. No in-app purchases.

Thank you for reviewing.
```

## Code changes — status

- [x] Replace auto-detection with NSOpenPanel + security-scoped bookmarks. **Done** in `BookmarkStore.swift` (commit `a9c6ebc`). The store no-ops in unsandboxed builds, so the GitHub-release ZIP is unaffected.
- [x] Persist bookmark data in UserDefaults, restore on launch. **Done** — wired into `AppDelegate.applicationWillFinishLaunching`.
- [x] "Grant access" affordance in sidebar empty state. **Done** — `SidebarView.emptyState` shows the new copy when `appState.needsSandboxGrant` is true.
- [x] `privacy.html` published. **Done** — at https://chrispelatari.github.io/Claude-recall/privacy.html.
- [ ] Generate 5 App Store screenshots at 2560×1600. (Best done by you running the app locally; AppleScript / `screencapture` route is brittle.)
- [ ] Cut a fresh build with sandbox entitlements active and the privacy manifest bundled. **Blocked by signing** — requires Apple Developer team `LFUDWMQGY3` (Kollo Inc.) and a provisioning profile from Apple's servers, which Xcode fetches via Xcode → Settings → Accounts.
- [ ] Test the app inside the sandbox end-to-end (exercise grant flow, open a folder, edit a file, watch a save, JSON viewer).

## Archive command (run this from a Mac signed into the Apple Dev account)

Once your Apple Developer team is added in Xcode → Settings → Accounts and a Mac App Distribution / Mac Installer Distribution cert exists in the keychain:

```bash
cd ~/Project/claude-recall      # adjust path
xcodegen generate

xcodebuild \
  -project ClaudeRecall.xcodeproj \
  -scheme ClaudeRecall \
  -configuration Release \
  -destination 'generic/platform=macOS' \
  -archivePath build/Claude Recall.xcarchive \
  ARCHS="arm64 x86_64" ONLY_ACTIVE_ARCH=NO \
  CODE_SIGN_STYLE=Automatic \
  DEVELOPMENT_TEAM=LFUDWMQGY3 \
  archive

# Then export as a Mac App Store .pkg:
cat > /tmp/crec-exportoptions.plist <<'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>method</key>           <string>app-store</string>
    <key>teamID</key>           <string>LFUDWMQGY3</string>
    <key>uploadSymbols</key>    <true/>
    <key>uploadBitcode</key>    <false/>
</dict>
</plist>
EOF

xcodebuild \
  -exportArchive \
  -archivePath build/Claude Recall.xcarchive \
  -exportOptionsPlist /tmp/crec-exportoptions.plist \
  -exportPath build/Claude Recall-export

# Upload to App Store Connect (xcrun altool is being deprecated; use notarytool's sibling Transporter command):
xcrun altool --upload-app -f build/Claude Recall-export/*.pkg -t macos \
  -u <your-apple-id-email> -p '@keychain:AC_PASSWORD'

# Or open Transporter.app from the App Store and drag the .pkg in.
```

## App Store Connect listing setup (one-time, in browser)

1. Sign in at https://appstoreconnect.apple.com with the same Apple ID tied to the `LFUDWMQGY3` team.
2. **My Apps → +** → **New App**.
3. Platform: macOS. Name: `Claude Recall`. Primary language: English (US). Bundle ID: `com.pelatari.claude-recall` (must match `project.yml`). SKU: `crec-001` (anything unique). User access: Full access. Click **Create**.
4. Once created, fill in the listing using the copy in this file (description, keywords, support URL, marketing URL, privacy policy URL).
5. **App Privacy → Edit** → answer "Data Not Collected." Save.
6. **Pricing and Availability** → Free (or paid tier 5 / $4.99). Worldwide availability.
7. Once the binary is uploaded via `xcodebuild -exportArchive` + Transporter, it appears under **TestFlight** within ~30 min. Then promote to **App Store → 1.0.0 (or 0.4.2) → Submit for Review**.

Expected first-review time: 1-3 days. New account on this Team may get extra-careful first review.
