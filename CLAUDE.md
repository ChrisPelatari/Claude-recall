# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Despite the directory name `Claude-recall`, this is **Claude Recall** — a native macOS + iOS SwiftUI app for browsing AI-agent memory files (`CLAUDE.md`, `AGENTS.md`, `GEMINI.md`, `~/.claude/projects/*.jsonl`, etc.). Current version: `0.4.2`.

Public repo lives at `github.com/ChrisPelatari/Claude-recall`; this directory is the author's working copy.

## Build & run

The Xcode project is generated from `project.yml` via **XcodeGen** — `.xcodeproj` is checked in but treated as a derived artifact. After editing `project.yml`, regenerate before opening Xcode:

```bash
xcodegen generate
```

Build commands:

```bash
# macOS (default scheme)
xcodebuild -project ClaudeRecall.xcodeproj -scheme ClaudeRecall \
  -destination 'platform=macOS' build

# iOS simulator (uses the second target — note the -iOS suffix)
xcodebuild -project ClaudeRecall.xcodeproj -scheme ClaudeRecall-iOS \
  -destination 'generic/platform=iOS Simulator' build
```

Mac App Store archive command (requires Apple team `LFUDWMQGY3` / Kollo Inc.): see `MAS_METADATA.md`.

There is no test target.

## Two distribution channels share one source

This is critical to understand before editing anything related to file access or entitlements:

| Channel | Sandboxed? | Entitlements | License to recipient |
|---|---|---|---|
| GitHub release ZIP | No (unsandboxed, ad-hoc signed) | none active | GPL-3.0 |
| Mac App Store | Yes | `ClaudeRecall.entitlements` (iCloud KV + sandbox) | Apple EULA |

The same source builds both. Sandbox-only behavior is gated at runtime by `BookmarkStore.isSandboxed`, which reads `APP_SANDBOX_CONTAINER_ID`. **Never assume the app can read arbitrary paths**; on MAS builds it can only see folders the user granted via `NSOpenPanel`, persisted as security-scoped bookmarks in `BookmarkStore`.

Entitlements are wired per-config in `project.yml`:
- **Debug** — `CODE_SIGN_ENTITLEMENTS: ""`, `CODE_SIGN_IDENTITY: "-"`. Local dev builds with sign-to-run-locally; no Apple Dev team needed; iCloud KVS silently falls back to UserDefaults.
- **Release** — `CODE_SIGN_ENTITLEMENTS: ClaudeRecall.entitlements`. Sandbox + iCloud-KVS active. Used for the MAS archive command in `MAS_METADATA.md`. For a GitHub-release ZIP (Release config but unsandboxed), override `CODE_SIGN_ENTITLEMENTS=""` at archive time.

MAS submission is currently blocked on user adding the Apple Dev team (`LFUDWMQGY3` / Kollo Inc.) in Xcode → Settings → Accounts — code work is done.

## Architecture

Single-`AppState` SwiftUI app, Swift 6 with `SWIFT_STRICT_CONCURRENCY: complete`. Layout:

```
ClaudeRecall/Sources/
  App/        ClaudeRecallApp.swift + AppDelegate (macOS)
  Models/     AppState, AISource, FileNode, SettingsStore, AppTheme
  Views/      ContentView → Sidebar / Detail / TOC / FileFindBar / UpdateBanner / MarkdownEditor
  Utilities/  FileTreeBuilder, FileWatcher, SearchService, BookmarkStore,
              MemoryReaderTheme, SplashCodeSyntaxHighlighter, PDFExporter,
              UpdateChecker, WindowChrome, LocalImageProvider, ReadableMarkdownRenderer
  Resources/  Info.plist, PrivacyInfo.xcprivacy, Assets.xcassets, entitlements
```

### Things that are easy to break

- **Cross-platform isolation is by `#if os(...)` inside shared files**, not by separate file trees. iOS lacks: `FSEvents` (file watcher), `NSOpenPanel`, `NSTextView` (the edit-mode editor), `PDFExporter`, window chrome. Adding new macOS-only APIs without `#if os(macOS)` will break the iOS build.
- **All persistence routes through `SettingsStore.shared`**, not `UserDefaults` directly. `SettingsStore` double-writes to `NSUbiquitousKeyValueStore` + `UserDefaults` so settings sync across the user's Macs via iCloud KV. New persisted preferences belong here. (Documents themselves are *not* synced — only settings.)
- **Bookmarks are device-local.** `BookmarkStore` deliberately uses `UserDefaults` only — security-scoped bookmark blobs are bound to the granting Mac and would fail to resolve on another device. Don't migrate them into `SettingsStore`.
- **Markdown rendering = `MarkdownUI` + `MemoryReaderTheme`.** A previous custom `MarkdownRenderer.swift` was removed; do not reintroduce one. Code-block highlighting flows through `SplashCodeSyntaxHighlighter` (Splash for Swift, fallback keyword highlighting for other languages).
- **`AppState.handleURL` rejects `..` and non-absolute paths** from URL-scheme callers. Keep that guard — the URL scheme is callable by any app via `open`.
- **File watching debounces 500ms** before rebuilding the tree (`AppState.handleFileSystemChange`). FSEvents fire in bursts on save; without the debounce the sidebar flickers.
- **No network calls except `UpdateChecker`** (daily GitHub releases poll, skippable). The privacy posture is "fully local"; new outbound calls require updating `PrivacyInfo.xcprivacy`, the App Store privacy nutrition label in `MAS_METADATA.md`, and `docs/privacy.html`.

### Cross-cutting flows

- **Auto-discovery**: `AISource.detectAllAvailable()` walks `~/.<agent>/` for the 8 supported agents plus any custom paths persisted via `AISource.addCustomSource`. New agents are added to `AISource.allSources`.
- **URL scheme** (`clauderecall://open?path=...&heading=...`): registered in `Info.plist`; entry point is `AppState.handleURL`. The `crec` CLI (repo root) is a shell wrapper that URL-encodes via `python3` and calls `open`.
- **Today panel**: `AISource.todayMemoryFile` looks for `memory/YYYY-MM-DD.md` under the current source; `AppState.autoSelectTodayFile` auto-expands and selects it.
- **Cold-start file open** (Finder → Claude Recall): `AppDelegate.application(_:open:)` may fire before SwiftUI mounts. URLs are buffered in `AppDelegate.pendingFileURLs` and flushed when `ContentView.onAppear` calls `markViewReady()`. Don't move that wiring.

## Versioning

Bump `MARKETING_VERSION` **on both `ClaudeRecall` and `ClaudeRecall-iOS` targets in `project.yml`** (and `CURRENT_PROJECT_VERSION`), then `xcodegen generate`. Don't edit `.xcodeproj` directly.

## Planning docs in the repo

`PLAN.md` / `V2-PLAN.md` / `V3-PLAN.md` are historical changelogs of completed milestones — not active plans. `CLAUDE_CLOUD_MEMORY_SPEC.md` is a forward-looking spec (claude.ai server-side memory sync) that is **not yet implemented**; don't act on it without explicit confirmation. `MAS_METADATA.md` is the App Store submission playbook and includes the archive command.
