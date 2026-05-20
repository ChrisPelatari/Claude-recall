# Technology Stack

**Analysis Date:** 2026-05-20

## Languages

**Primary:**
- Swift 6.0 - All application code (`SWIFT_VERSION: "6.0"` in `project.yml`)

**Secondary:**
- Bash - CLI launcher (`aimr` shell script at repo root)
- Python 3 - Invoked from `aimr` only for `urllib.parse.quote` URL encoding

## Runtime

**Environment:**
- macOS 15.0+ (Sequoia) for the primary `AIMemoryReader` target
- iOS 17.0+ for the `AIMemoryReader-iOS` target
- Both built against Xcode 16.0

**Concurrency Mode:**
- `SWIFT_STRICT_CONCURRENCY: complete` on both targets — every type that crosses an actor boundary must be `Sendable`, and all mutable state on `AppState` is `@MainActor`-isolated.

## Build Tooling

**Project Generation:**
- **XcodeGen** ≥ 2.40 (`minimumXcodeGenVersion: "2.40"` in `project.yml`)
- The single source of truth is `project.yml`. `AIMemoryReader.xcodeproj` is regenerated via `xcodegen generate` after any `project.yml` edit; do not hand-edit the `.xcodeproj`.

**Build System:**
- `xcodebuild` for both targets (commands in `CLAUDE.md` and `MAS_METADATA.md`)
- Two schemes share one source tree:
  - `AIMemoryReader` → macOS app
  - `AIMemoryReader-iOS` → iOS app
- No test target exists.

**Package Manager:**
- Swift Package Manager (SPM), declared in the `packages:` block of `project.yml`.

## Frameworks

**UI:**
- **SwiftUI** - Primary UI framework (`@Observable`, `NavigationSplitView`, `WindowGroup`, etc.)
- **AppKit** - Used inside `#if os(macOS)` for `NSOpenPanel`, `NSSavePanel`, `NSPrintOperation`, `NSTextView`, `NSHostingView`, `FSEventStream`, `NSApplicationDelegate`, `NSWorkspace`. Files that depend on AppKit are entirely wrapped in `#if os(macOS)` blocks (e.g. `Utilities/PDFExporter.swift`, `Utilities/WindowChrome.swift`, `Utilities/ReadableMarkdownRenderer.swift`, `Utilities/LocalImageProvider.swift`, `Views/DetailView.swift`, `Views/SidebarView.swift`).
- **UIKit** - Used inside `#if os(iOS)` for `UIDocumentPickerViewController` (`Views/ContentView.swift` lines 223–263) and `UIFont` (`Utilities/SplashCodeSyntaxHighlighter.swift` line 10).

**Markdown:**
- **MarkdownUI** (`swift-markdown-ui`) - Block-level markdown rendering with theming. Imported in `Utilities/MemoryReaderTheme.swift`, `Utilities/PDFExporter.swift`, `Utilities/LocalImageProvider.swift`, `Utilities/SplashCodeSyntaxHighlighter.swift`, `Views/ContentView.swift`, `Views/DetailView.swift`.
- **Foundation `AttributedString(markdown:)`** - Used in `Utilities/ReadableMarkdownRenderer.swift` to produce `NSAttributedString` for the find-in-page `NSTextView`, where MarkdownUI's renderer can't expose character offsets.

**Syntax Highlighting:**
- **Splash** (JohnSundell) - Tokenizes Swift code blocks. Used in `Utilities/SplashCodeSyntaxHighlighter.swift`. For non-Swift languages, the same file falls back to a hand-rolled keyword highlighter (`basicHighlight(_:language:)`).

**File System:**
- **FSEvents** (CoreServices) - Directory watcher in `Utilities/FileWatcher.swift` (macOS only).
- **FileManager** - Tree walking in `Utilities/FileTreeBuilder.swift` and `Utilities/SearchService.swift`.

**Persistence:**
- **UserDefaults** + **NSUbiquitousKeyValueStore** - Dual-write through `Models/SettingsStore.swift` so settings sync via iCloud KV across the user's Macs.
- **Security-Scoped Bookmarks** - `Utilities/BookmarkStore.swift` persists per-folder grants for the sandboxed (MAS) build using UserDefaults only (bookmarks are device-local; deliberately not iCloud-synced).

## Key Dependencies

**SPM packages declared in `project.yml`:**

| Package | Version | Source | Purpose |
|---------|---------|--------|---------|
| `swift-markdown-ui` (product `MarkdownUI`) | `from: "2.4.0"` | `https://github.com/gonzalezreal/swift-markdown-ui` | Themed markdown rendering for read view and PDF export |
| `Splash` | `from: "0.16.0"` | `https://github.com/JohnSundell/Splash` | Swift-specific syntax highlighting inside fenced code blocks |

Both packages are bound to both targets.

## Configuration

**Project configuration:**
- `project.yml` - XcodeGen spec (targets, deployment targets, SPM deps, build settings, entitlement assignment)
- `AIMemoryReader.entitlements` - macOS sandbox + user-selected R/W + iCloud KVS (active for Mac App Store build)
- `AIMemoryReader-iOS.entitlements` - iOS iCloud KVS only
- `AIMemoryReader/Sources/Resources/Info.plist` - URL scheme registration (`aimemoryreader://`), markdown document type, document browser support
- `AIMemoryReader/Sources/Resources/PrivacyInfo.xcprivacy` - App Privacy manifest declaring FileTimestamp + UserDefaults API usage and no tracking
- Bundle identifiers: `com.aitools.ai-memory-reader` (macOS), `com.aitools.ai-memory-reader-ios` (iOS)
- Marketing version: `0.4.2`; build numbers `4` (macOS), `3` (iOS)

**Distribution-channel switch:**
- The Mac App Store build uses `CODE_SIGN_ENTITLEMENTS: AIMemoryReader.entitlements` (sandboxed).
- The GitHub-release ZIP build overrides `CODE_SIGN_ENTITLEMENTS` to `""` so the same source ships unsandboxed and ad-hoc signed (comment at `project.yml` lines 42–44).

**Runtime sandbox detection:**
- `BookmarkStore.isSandboxed` checks `ProcessInfo.processInfo.environment["APP_SANDBOX_CONTAINER_ID"]` to gate sandbox-only code paths (`Utilities/BookmarkStore.swift` lines 23–25). This is read by `AppState.needsSandboxGrant` (`Models/AppState.swift` lines 80–88) and by `UpdateChecker.checkIfDue()` (`Utilities/UpdateChecker.swift` line 35).

## Platform Requirements

**Development:**
- Xcode 16.0
- XcodeGen ≥ 2.40 (Homebrew: `brew install xcodegen`)
- Apple Developer Team `LFUDWMQGY3` (Kollo Inc.) required for MAS submission; not needed for the GitHub-release build.

**Deployment targets:**
- macOS 15.0 (`MACOSX_DEPLOYMENT_TARGET: "15.0"`)
- iOS 17.0 (`IPHONEOS_DEPLOYMENT_TARGET: "17.0"`)
- `SUPPORTS_MAC_DESIGNED_FOR_IPHONE_IPAD: NO` on the iOS target — iOS app is not Mac-Catalyst-on-Apple-Silicon eligible.

**Distribution:**
- GitHub Releases ZIP (unsandboxed, ad-hoc signed, GPL-3.0)
- Mac App Store (sandboxed, Apple EULA) — see `MAS_METADATA.md` for the archive command and submission playbook

---

*Stack analysis: 2026-05-20*
