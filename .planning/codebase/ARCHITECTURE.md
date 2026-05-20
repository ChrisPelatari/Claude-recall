<!-- refreshed: 2026-05-20 -->
# Architecture

**Analysis Date:** 2026-05-20

## System Overview

```text
┌──────────────────────────────────────────────────────────────────────┐
│                     Entry / App lifecycle                            │
│  `AIMemoryReaderApp` (@main, SwiftUI Scene)                          │
│  `AppDelegate` (macOS only — buffers cold-start file URLs)           │
│  `App/AIMemoryReaderApp.swift`                                       │
└───────────────┬─────────────────────────────┬────────────────────────┘
                │                             │
                ▼                             ▼
┌──────────────────────────────┐  ┌────────────────────────────────────┐
│  Views (SwiftUI tree)        │  │  AppState (@Observable @MainActor) │
│  `Views/ContentView.swift`   │  │  `Models/AppState.swift`           │
│   ├─ macOS: NavigationSplit  │◀─┤   • rootNode / selectedFile        │
│   │   Sidebar + Detail + TOC │  │   • availableSources               │
│   ├─ iOS: NavigationStack    │  │   • search state                   │
│   │   List + Detail          │  │   • file-watch / debounce          │
│   ├─ UpdateBanner            │  │   • URL-scheme dispatch            │
│   ├─ FileFindBar             │──▶ writes through to                  │
│   ├─ MarkdownEditorView      │  │   `SettingsStore.shared`           │
│   └─ FindableReadableView    │  └──────────────┬─────────────────────┘
└──────────────────────────────┘                 │
                                                 ▼
┌──────────────────────────────────────────────────────────────────────┐
│  Utilities (stateless services / singletons)                         │
│  `Utilities/FileTreeBuilder.swift`   `Utilities/FileWatcher.swift`   │
│  `Utilities/SearchService.swift`     `Utilities/BookmarkStore.swift` │
│  `Utilities/UpdateChecker.swift`     `Utilities/PDFExporter.swift`   │
│  `Utilities/MemoryReaderTheme.swift` `Utilities/WindowChrome.swift`  │
│  `Utilities/SplashCodeSyntaxHighlighter.swift`                       │
│  `Utilities/LocalImageProvider.swift`                                │
│  `Utilities/ReadableMarkdownRenderer.swift`                          │
└───────┬────────────────────────┬────────────────────┬────────────────┘
        │                        │                    │
        ▼                        ▼                    ▼
┌────────────────────┐  ┌─────────────────┐  ┌──────────────────────────┐
│  Local filesystem  │  │ UserDefaults +  │  │ GitHub Releases API      │
│  (FSEvents,        │  │ NSUbiquitous    │  │ (UpdateChecker, daily,   │
│  FileManager,      │  │ KeyValueStore   │  │  skipped when sandboxed) │
│  NSOpenPanel,      │  │ (SettingsStore, │  │                          │
│  UIDocumentPicker, │  │  BookmarkStore) │  │                          │
│  security-scoped   │  │                 │  │                          │
│  bookmarks)        │  │                 │  │                          │
└────────────────────┘  └─────────────────┘  └──────────────────────────┘
```

## Component Responsibilities

| Component | Responsibility | File |
|-----------|----------------|------|
| `AIMemoryReaderApp` | `@main` Scene; instantiates `AppState`, wires `.onOpenURL`, registers macOS menu commands | `AIMemoryReader/Sources/App/AIMemoryReaderApp.swift` |
| `AppDelegate` (macOS) | Restores bookmarks on launch, stops scopes on terminate, buffers cold-start `open:` URLs until SwiftUI is ready | `AIMemoryReader/Sources/App/AIMemoryReaderApp.swift` |
| `AppState` | Single `@Observable @MainActor` hub for all UI state, source selection, search, file watching, and URL-scheme dispatch | `AIMemoryReader/Sources/Models/AppState.swift` |
| `AISource` | Static catalog of 8 supported AI agents + custom user-added sources; detects which exist on disk and contain supported files | `AIMemoryReader/Sources/Models/AISource.swift` |
| `FileNode` | `@Observable` reference-type tree node for the sidebar (id is its absolute path) | `AIMemoryReader/Sources/Models/FileNode.swift` |
| `SettingsStore` | iCloud-KV + UserDefaults dual-write for synced preferences | `AIMemoryReader/Sources/Models/SettingsStore.swift` |
| `AppTheme` / `ThemePalette` | Theme enum + per-theme color palette resolved via `.environment(\.themePalette, …)` | `AIMemoryReader/Sources/Models/AppTheme.swift` |
| `BookmarkStore` | Security-scoped bookmark persistence (UserDefaults), runtime sandbox detection | `AIMemoryReader/Sources/Utilities/BookmarkStore.swift` |
| `FileTreeBuilder` | Pure function: build a `FileNode` tree from a root URL, filtering to `.md`/`.json` and pruning empty dirs | `AIMemoryReader/Sources/Utilities/FileTreeBuilder.swift` |
| `FileWatcher` (macOS) | FSEvents stream wrapper that posts `.fileWatcherDidDetectChange` | `AIMemoryReader/Sources/Utilities/FileWatcher.swift` |
| `SearchService` | Substring search across all `.md`/`.json` under a root, or within a single file | `AIMemoryReader/Sources/Utilities/SearchService.swift` |
| `UpdateChecker` | Daily GitHub Releases poll, semver compare, dismissed-version persistence | `AIMemoryReader/Sources/Utilities/UpdateChecker.swift` |
| `PDFExporter` (macOS) | Renders a `Markdown` view through `NSHostingView` + `NSPrintOperation` to produce a PDF | `AIMemoryReader/Sources/Utilities/PDFExporter.swift` |
| `WindowChrome` (macOS) | Tints the title bar to match the current theme | `AIMemoryReader/Sources/Utilities/WindowChrome.swift` |
| `MemoryReaderTheme` | Builds a `MarkdownUI.Theme` from a `ThemePalette` | `AIMemoryReader/Sources/Utilities/MemoryReaderTheme.swift` |
| `SplashCodeSyntaxHighlighter` | Splash for Swift; hand-rolled keyword highlighter for Python/JS/JSON/Bash/YAML/Markdown | `AIMemoryReader/Sources/Utilities/SplashCodeSyntaxHighlighter.swift` |
| `LocalImageProvider` (macOS) | Resolves relative/absolute markdown image refs against the document's parent directory | `AIMemoryReader/Sources/Utilities/LocalImageProvider.swift` |
| `ReadableMarkdownRenderer` (macOS) | Converts markdown → `NSAttributedString` for find-in-page highlighting in an `NSTextView` | `AIMemoryReader/Sources/Utilities/ReadableMarkdownRenderer.swift` |
| `ContentView` | Platform router → `MacContentView` (`NavigationSplitView`) or `iOSContentView` (`NavigationStack`) | `AIMemoryReader/Sources/Views/ContentView.swift` |
| `SidebarView` (macOS) | AI source picker + search + file tree | `AIMemoryReader/Sources/Views/SidebarView.swift` |
| `DetailView` / `MarkdownDetailView` (macOS) | Renders selected markdown/JSON, hosts find bar + TOC + edit mode + PDF export | `AIMemoryReader/Sources/Views/DetailView.swift` |
| `TOCView` / `TOCParser` / `MarkdownSplitter` | Heading extraction + section splitting used by both macOS and iOS detail views | `AIMemoryReader/Sources/Views/TOCView.swift` |
| `FileFindBar` (macOS) | ⌘F find-bar UI with previous/next/escape behavior | `AIMemoryReader/Sources/Views/FileFindBar.swift` |
| `FindableReadableView` (macOS) | `NSTextView` wrapper showing the pre-rendered attributed document with NSRange-precise match highlighting | `AIMemoryReader/Sources/Views/FindableReadableView.swift` |
| `MarkdownEditorView` (macOS) | `NSTextView`-backed editor used in edit mode | `AIMemoryReader/Sources/Views/MarkdownEditorView.swift` |
| `UpdateBanner` (macOS) | Banner shown above the split view when `UpdateChecker.state == .available` | `AIMemoryReader/Sources/Views/UpdateBanner.swift` |

## Pattern Overview

**Overall:** Single-source-of-truth state hub + platform-isolated SwiftUI presentation.

**Key Characteristics:**
- One `AppState` `@Observable` `@MainActor` instance owned by `AIMemoryReaderApp` and injected via `.environment(appState)`; every view reads from it via `@Environment(AppState.self)`.
- Strict Swift 6 concurrency (`SWIFT_STRICT_CONCURRENCY: complete`) — `AppState` and `UpdateChecker` are `@MainActor`; `SettingsStore` and `BookmarkStore` are `@unchecked Sendable` singletons relying on thread-safe `UserDefaults` and `NSUbiquitousKeyValueStore`; `FileWatcher` is `Sendable` and posts back through `NotificationCenter`.
- Stateless utilities + a few `.shared` singletons (`SettingsStore`, `BookmarkStore`, `UpdateChecker`).
- No reactive framework other than SwiftUI/`@Observable`; no Combine pipelines, no dependency-injection container, no MVVM ViewModels per view.

## Layers

**App / Entry layer:**
- Purpose: Process lifecycle, OS-event plumbing, top-level menus
- Location: `AIMemoryReader/Sources/App/`
- Contains: `AIMemoryReaderApp`, `AppDelegate`, `Notification.Name` extensions
- Depends on: `AppState`, `BookmarkStore`, `UpdateChecker`
- Used by: macOS / iOS runtime

**Models layer:**
- Purpose: State container + value types
- Location: `AIMemoryReader/Sources/Models/`
- Contains: `AppState`, `AISource`, `FileNode`, `SettingsStore`, `AppTheme` + `ThemePalette`
- Depends on: `Utilities/*`
- Used by: every View

**Views layer:**
- Purpose: SwiftUI presentation, NSViewRepresentable wrappers
- Location: `AIMemoryReader/Sources/Views/`
- Contains: macOS split-view shell, iOS navigation-stack shell, detail/TOC/find/editor/banner
- Depends on: `AppState` via `@Environment`, theme via `@Environment(\.themePalette)`, all utilities for rendering

**Utilities layer:**
- Purpose: Stateless services and OS adapters
- Location: `AIMemoryReader/Sources/Utilities/`
- Contains: file system, search, theming, syntax highlighting, PDF export, update checker, bookmarks, window chrome, image provider, find-mode renderer
- Depends on: Foundation, AppKit/UIKit (gated by `#if os(...)`), MarkdownUI, Splash
- Used by: `AppState` and Views

**Resources:**
- `AIMemoryReader/Sources/Resources/` — `Info.plist`, `PrivacyInfo.xcprivacy`, `Assets.xcassets`, and copies of the two entitlement files

## Data Flow

### Cold-start file open (Finder → AIMR, macOS)

1. macOS launches the bundle and invokes `AppDelegate.application(_:open:)` (`App/AIMemoryReaderApp.swift` line 38).
2. If SwiftUI hasn't mounted yet (`viewReady == false`), the URL is appended to `AppDelegate.pendingFileURLs`.
3. `AIMemoryReaderApp.body` builds the scene; `ContentView.onAppear` calls `AppDelegate.markViewReady()` (line 76).
4. `markViewReady()` flushes `pendingFileURLs` by posting `.openFileFromSystem` for each.
5. The scene's `.onReceive(NotificationCenter.default.publisher(for: .openFileFromSystem))` (line 71) calls `appState.openSingleFile(url)`.
6. `AppState.openSingleFile` rebuilds `rootNode` with just the dropped file, sets `selectedFile`, and starts watching the file's parent directory (`Models/AppState.swift` lines 230–253).

### URL-scheme open (`aimemoryreader://open?...`)

1. macOS / iOS invokes the SwiftUI `.onOpenURL` handler (`App/AIMemoryReaderApp.swift` line 62).
2. Non-file URLs are routed to `AppState.handleURL(_:)` (`Models/AppState.swift` line 265).
3. `handleURL` validates scheme = `aimemoryreader`, rejects path traversal (`..`) and non-absolute paths, standardizes the file URL, confirms `FileManager.fileExists`, then calls `openSingleFile`.
4. If a `heading=` query was present, it is stored in `pendingURLHeading`; `MarkdownDetailView` (macOS) and `iOSDetailView` (iOS) observe and scroll to a matching TOC entry, then clear `pendingURLHeading`.

### File watching with debounce (macOS only)

1. `AppState.selectSource(_:)` / `openSingleFile(_:)` / `openFolder()` end by calling the private `startWatching(_:)` (`Models/AppState.swift` line 342).
2. `startWatching` constructs a `FileWatcher`, calls `startStream()` (which builds an `FSEventStreamRef`, `Utilities/FileWatcher.swift` lines 17–47), and stores it in `activeStream`.
3. FSEvents fires on the `com.aitools.filewatcher` dispatch queue; the C callback posts `.fileWatcherDidDetectChange` on the main queue.
4. A `NotificationCenter` observer on `AppState` cancels any in-flight `debounceTask` and starts a new one that sleeps 500 ms before calling `handleFileSystemChange()` (`Models/AppState.swift` lines 357–371). FSEvents fire in bursts on save; without the debounce the sidebar would flicker.
5. `handleFileSystemChange()` (lines 398–424) snapshots the expanded directories and the currently-selected URL, calls `FileTreeBuilder.buildTree(at: rootURL)`, restores expansion + selection, and increments `fileChangeToken` so `MarkdownDetailView.onChange(of: fileChangeToken)` (`Views/DetailView.swift` line 109) reloads the visible file from disk.

### Auto-detected AI sources

1. On `AppState.init()` and on every `refreshSources()` call, `AISource.detectAllAvailable()` runs (`Models/AISource.swift` line 174).
2. `detectAvailable()` (iOS short-circuits to `[]`, line 132) iterates `AISource.allSources` — eight built-in agents (`.claude`, `.codex`, `.gemini`, `.cursor`, `.continue`, `.config/github-copilot`, `.aider`, `.openclaw/workspace`) — and keeps those where `exists` and `containsSupportedFiles` are both true (lines 41–53).
3. Custom sources are loaded from `SettingsStore.shared.customAISourcePaths` and appended; only paths that still exist on disk are surfaced (`loadCustomSources()` line 142).
4. On launch, `AppState.restoreOrAutoSelect()` (`Models/AppState.swift` lines 163–189) prefers the persisted `selectedSourceID`, then a saved local folder, then the first available source.

### Today panel

1. Each `AISource` exposes a computed `todayMemoryFile: URL?` that checks for `memory/YYYY-MM-DD.md` under the source root using `DateFormatter` with format `yyyy-MM-dd` (`Models/AISource.swift` lines 56–65).
2. `AppState.autoSelectTodayFile(for:)` (`Models/AppState.swift` lines 448–455) finds the node in the tree, expands the path, and sets `todayFileNode` so the sidebar / iOS list can show a "Today" badge (`Views/ContentView.swift` lines 203–211).

### Search

1. The sidebar search field binds to `AppState.searchQuery`; `performSearch()` runs on `.userInitiated` global queue (`Models/AppState.swift` lines 291–327).
2. Single-file mode searches the current file via `SearchService.searchInFile(query:fileURL:)` (returns one `SearchResult` per matching line).
3. Directory mode walks the root via `FileManager.enumerator` filtered to `FileNode.supportedExtensions`, returns the first match per file (`Utilities/SearchService.swift` lines 38–74).
4. `selectSearchResult(_:)` finds the existing node in the tree (`AppState.findNode`) and expands the path to it via `expandPathTo(node:)`.

### Update banner

1. `MacContentView` calls `await UpdateChecker.shared.checkIfDue()` from a `.task` modifier on initial appearance (`Views/ContentView.swift` line 38).
2. `checkIfDue()` returns early when sandboxed, then enforces the 24-hour `checkInterval` against the `updateChecker.lastCheckAt` UserDefaults timestamp (`Utilities/UpdateChecker.swift` lines 33–43).
3. On success, `handle(release:)` strips a leading `v` from the tag and runs a strict per-component semver compare (`versionIsNewer`, lines 123–132). If newer and not on the user's dismissed list, state becomes `.available(...)`.
4. `MacContentView` switches on `UpdateChecker.shared.state` (line 26) and inserts `UpdateBanner` above the split view; Skip / Later / Download write through `UpdateChecker.skipVersion(tag:)` / `dismissThisSession()` / `NSWorkspace.shared.open(_:)`.

**State Management:**
- All shared mutable state lives on the single `AppState` instance, observed via `@Environment(AppState.self)`.
- Persistent state goes through `SettingsStore.shared` (iCloud-synced settings) or `BookmarkStore.shared` (device-local bookmarks).
- View-local state uses `@State`, e.g. `MarkdownDetailView.isFindActive`, `rawContent`, `editableContent`, `tocEntries`.

## Key Abstractions

**`AppState` (state hub):**
- Purpose: Owns every piece of state the UI reads (sidebar tree, selection, search, theme, file-change notifications)
- Examples: `Models/AppState.swift`
- Pattern: `@Observable` reference type, `@MainActor`-isolated. Property `didSet` blocks write through to `SettingsStore.shared` for things that need to persist (e.g. `selectedSourceID`, `appTheme`).

**`AISource` (catalog + auto-detection):**
- Purpose: Identify which AI agents are installed on this machine and expose their memory directories
- Examples: `Models/AISource.swift` — `allSources` (built-in agents), `detectAvailable()`, `loadCustomSources()`, `todayMemoryFile`
- Pattern: Value type (`struct AISource: Identifiable, Hashable`) with static helpers for the catalog and a separate persisted list of user-added paths.

**`FileNode` (sidebar tree):**
- Purpose: Tree representation of the selected source's directory, with mutable `isExpanded` and `children`
- Examples: `Models/FileNode.swift`
- Pattern: `@Observable` class (reference identity needed so SwiftUI tree expansion and selection survive rebuilds). `id` is the absolute path string.

**`BookmarkStore.isSandboxed` (distribution-channel gate):**
- Purpose: Single boolean that runtime code checks to decide whether to use sandbox-only paths (security-scoped bookmarks, "grant access" UI) or trust raw paths
- Examples: `Utilities/BookmarkStore.swift` line 23–25
- Pattern: Lazily-evaluated static `let`; checked from `AppState.needsSandboxGrant` and `UpdateChecker.checkIfDue()`. The same binary detects its own deployment channel via `APP_SANDBOX_CONTAINER_ID`.

## Entry Points

**`AIMemoryReaderApp` (`@main`):**
- Location: `AIMemoryReader/Sources/App/AIMemoryReaderApp.swift` line 50
- Triggers: Process launch on both macOS and iOS
- Responsibilities: Instantiate `AppState`; attach `@NSApplicationDelegateAdaptor(AppDelegate.self)` on macOS; build the `WindowGroup`; wire `.onOpenURL`, `.handlesExternalEvents`, `WindowChrome.apply`, and the macOS `.commands` block (File / Edit / View / Help menus).

**`AppDelegate` (macOS only):**
- Location: `AIMemoryReader/Sources/App/AIMemoryReaderApp.swift` line 14
- Triggers: `NSApplication` lifecycle
- Responsibilities: `applicationWillFinishLaunching` → `BookmarkStore.shared.restoreOnLaunch()`; `applicationWillTerminate` → `BookmarkStore.shared.stopAllAccess()`; `application(_:open:)` → buffers / dispatches file URLs.

## Architectural Constraints

- **Threading:** All UI state mutation is `@MainActor` via `AppState`. Search runs on `DispatchQueue.global(qos: .userInitiated)` then bounces back to `.main`. File watching uses a dedicated `com.aitools.filewatcher` dispatch queue and posts notifications on `.main`. Settings I/O is allowed off-main because `UserDefaults` and `NSUbiquitousKeyValueStore` are thread-safe.
- **Global state:** Singletons exist for `SettingsStore.shared`, `BookmarkStore.shared`, `UpdateChecker.shared`. These are intentional — they are process-level resources. No other module-level singletons or mutable globals.
- **Cross-platform isolation:** Done by `#if os(macOS)` / `#if os(iOS)` blocks inside shared files, not by separate file trees. iOS lacks the entirety of `FileWatcher.swift`, `PDFExporter.swift`, `WindowChrome.swift`, `LocalImageProvider.swift`, `ReadableMarkdownRenderer.swift`, `FileFindBar.swift`, `FindableReadableView.swift`, `MarkdownEditorView.swift`, `UpdateBanner.swift`, `SidebarView.swift`, `DetailView.swift`, plus the macOS halves of `AppState`, `AISource`, `ContentView`, and `AIMemoryReaderApp`. Adding any AppKit-only API outside `#if os(macOS)` will break the iOS build.
- **Sandbox gating:** Sandboxed (MAS) builds cannot read arbitrary paths. `AppState.needsSandboxGrant` decides whether the sidebar shows the "Grant access" empty state. New file-access code must call through `BookmarkStore` or be inside a folder the user has granted, or it will silently fail under sandbox.
- **No network in sandboxed builds:** `AIMemoryReader.entitlements` does not grant `com.apple.security.network.client`. The MAS binary cannot make outbound calls; `UpdateChecker.checkIfDue()` already short-circuits there, but any new network code must add the entitlement and update privacy disclosures.

## Anti-Patterns

### Reaching for `UserDefaults` directly

**What happens:** New persisted preference uses `UserDefaults.standard.set(_:forKey:)`.
**Why it's wrong:** Settings won't sync across the user's Macs via iCloud KV. `SettingsStore` exists exactly so prefs round-trip through `NSUbiquitousKeyValueStore`.
**Do this instead:** Add a key to `SettingsStore.Key`, expose a typed accessor on `SettingsStore`, and read/write through `SettingsStore.shared`. See `Models/SettingsStore.swift` for the pattern. (Exception: security-scoped bookmarks and `UpdateChecker` state are deliberately UserDefaults-only — see `BookmarkStore.swift` lines 9–14 and `UpdateChecker.swift` lines 25–27.)

### Reintroducing a custom markdown renderer

**What happens:** A new file `MarkdownRenderer.swift` is added that parses markdown and emits SwiftUI views directly.
**Why it's wrong:** A previous custom renderer was removed because it diverged from CommonMark and missed edge cases. MarkdownUI + `MemoryReaderTheme` is the canonical renderer; `ReadableMarkdownRenderer` is the only other accepted path, and only because `NSTextView` requires character-level NSRanges.
**Do this instead:** Extend `Utilities/MemoryReaderTheme.swift` (`MarkdownUI.Theme.memoryReader`) for new block / inline styling. Use `Utilities/ReadableMarkdownRenderer.swift` only when AppKit text APIs are involved.

### Adding macOS APIs without `#if os(macOS)`

**What happens:** A new utility imports `AppKit` at the top of a shared file without a platform guard.
**Why it's wrong:** The iOS target compiles the same `AIMemoryReader/Sources` tree and will fail at compile time.
**Do this instead:** Either wrap the whole file in `#if os(macOS) … #endif` (see `Utilities/PDFExporter.swift`, `Utilities/WindowChrome.swift`), or branch inside the function (see `Models/AISource.swift` lines 22–35 for the dual macOS/iOS `url` computed property).

### Bypassing the URL-scheme guard

**What happens:** New code calls `AppState.openSingleFile(_:)` directly from a URL scheme path with no validation.
**Why it's wrong:** The URL scheme is callable by any installed app via `open aimemoryreader://...`. Without the existing `..` and absolute-path checks, a malicious caller could try directory-traversal tricks.
**Do this instead:** Route all URL-scheme inputs through `AppState.handleURL(_:)` (`Models/AppState.swift` lines 265–287). Keep the `path.contains("..")` and `path.hasPrefix("/")` guards in place.

### Removing the FSEvents debounce

**What happens:** Code reacts to `.fileWatcherDidDetectChange` synchronously and immediately calls `handleFileSystemChange()`.
**Why it's wrong:** FSEvents fires in rapid bursts on save (often 3–6 events within a few milliseconds). Rebuilding the tree on each event causes visible flicker and wastes CPU.
**Do this instead:** Preserve the 500 ms debounce in `startWatching` (`Models/AppState.swift` lines 363–369).

## Error Handling

**Strategy:** Best-effort, never crash, fall back to a sensible empty/default state. AIMR has no error reporting back-channel, so silent fallback is the deliberate policy for non-actionable failures.

**Patterns:**
- File reads use `try? Data(contentsOf:)`; failures result in an empty result rather than an error surface (`Utilities/SearchService.swift` lines 17–20, 53–56; `Utilities/FileTreeBuilder.swift` line 13).
- The iOS detail view surfaces a `loadError: String?` for the user-visible error case (`Views/ContentView.swift` lines 270–288).
- Bookmark resolution failures drop the bookmark from storage (`Utilities/BookmarkStore.swift` lines 168–177) so the user can re-grant; stale bookmarks are refreshed in place (lines 152–169).
- `UpdateChecker` distinguishes manual (`Help → Check for Updates`) failures (which surface an error state) from background failures (which silently revert to `.idle`) — see `performCheck(manual:)` lines 85–92.

## Cross-Cutting Concerns

**Logging:** `print(...)` to stderr only, in `Utilities/FileWatcher.swift` and `Models/AppState.swift`'s `handleFileSystemChange()`. No log framework, no log levels.

**Validation:** URL-scheme inputs are validated in `AppState.handleURL`. File extensions are validated against `FileNode.supportedExtensions = ["md", "json"]` (`Models/FileNode.swift` line 12).

**Authentication:** None.

**Theming:** `AppTheme` enum (`.standard` / `.eyeCare`) → `ThemePalette` struct injected into the SwiftUI environment as `\.themePalette`. Every view that needs themed colors reads `@Environment(\.themePalette) private var palette` and branches on `palette.isEyeCare` to choose between system-semantic colors and Solarized palette colors. Theme changes also trigger `WindowChrome.apply(for:)` on macOS to repaint the title bar.

**Distribution-channel gating:** `BookmarkStore.isSandboxed` is checked in `AppState.needsSandboxGrant` (sidebar empty state), `BookmarkStore.restoreOnLaunch()` (no-op when unsandboxed), and `UpdateChecker.checkIfDue()` (skip when sandboxed because MAS handles updates).

---

*Architecture analysis: 2026-05-20*
