# External Integrations

**Analysis Date:** 2026-05-20

Claude Recall is a **local-first** app. The only outbound network call is the daily GitHub Releases poll for the update banner, and even that is suppressed on Mac App Store builds (Apple handles updates there). There are no analytics, telemetry, auth providers, databases, or third-party APIs.

## Inbound Surfaces (other apps / the user can invoke Claude Recall)

### URL Scheme: `clauderecall://`

- **Registered in:** `ClaudeRecall/Sources/Resources/Info.plist` (`CFBundleURLTypes` block)
- **Scheme name:** `clauderecall`
- **Entry point:** `AppState.handleURL(_:)` in `ClaudeRecall/Sources/Models/AppState.swift` lines 265–287, dispatched from `ClaudeRecallApp.body`'s `.onOpenURL { … }` at `ClaudeRecall/Sources/App/ClaudeRecallApp.swift` lines 62–68.
- **Supported URLs:**
  - `clauderecall://open?path=<absolute-path>` — open a file
  - `clauderecall://open?path=<absolute-path>&heading=<text>` — open file and scroll to a heading whose title contains `<text>` (iOS handling: `iOSDetailView.onChange(of: pendingURLHeading)` at `Views/ContentView.swift` lines 320–327)
- **Security guards:** Rejects paths containing `..` and any non-absolute path before resolving (`AppState.swift` lines 274–276). The URL scheme is callable by any other app via `open`, so these checks must remain in place.

### `crec` CLI

- **Location:** `crec` at the repo root (Bash, 85 lines)
- **Purpose:** Thin wrapper that resolves a path to absolute, URL-encodes it via `python3 -c "import urllib.parse; print(urllib.parse.quote(...))"`, and shells out to `open "clauderecall://open?path=..."`
- **Subcommands:**
  - `crec open <file.md>` — open a file
  - `crec open <file.md> --heading "Section Title"` (or `-h`)
- **Dependencies:** `bash`, `open`, `python3` (system Python is sufficient — used only for `urllib.parse.quote`)
- **Not installed automatically.** Distribution model is "copy this script onto your `PATH`."

### File-Open Handler (Finder → Claude Recall)

- **macOS:** `AppDelegate.application(_:open:)` at `ClaudeRecall/Sources/App/ClaudeRecallApp.swift` lines 38–46. Cold-start URLs that arrive before SwiftUI mounts are buffered in `AppDelegate.pendingFileURLs` and flushed when `ContentView.onAppear` calls `AppDelegate.markViewReady()` (line 76–78). Routes to `AppState.openSingleFile(_:)` via the `openFileFromSystem` `NotificationCenter` channel.
- **iOS:** Same `.onOpenURL` handler in `ClaudeRecallApp.body` — branches on `url.isFileURL` and calls `AppState.openSingleFile(_:)`.
- **Document types:** Declared in `Info.plist` via `CFBundleDocumentTypes` (`net.daringfireball.markdown`, `public.plain-text`) and `UTImportedTypeDeclarations` (extensions `md`, `markdown`, `mdown`). `LSHandlerRank` is `Alternate` so Claude Recall doesn't steal `.md` from the user's chosen default editor.

### Drag and Drop (macOS)

- **Handler:** `MacContentView.handleDrop(_:)` in `ClaudeRecall/Sources/Views/ContentView.swift` lines 54–110
- **Accepts:** `UTType.fileURL` items; routes single folders into `AppState.loadDirectory`, single files into `AppState.openSingleFile`, multi-file drops into the parent directory with the first file selected.

### iOS Document Picker

- **Component:** `DocumentPickerView` (UIViewControllerRepresentable wrapping `UIDocumentPickerViewController`) at `ClaudeRecall/Sources/Views/ContentView.swift` lines 227–263.
- **Content types:** `.md`, `.json`, `.plainText`.
- **Security-scoped URLs:** `documentPicker(_:didPickDocumentsAt:)` calls `url.startAccessingSecurityScopedResource()` and intentionally **does not** call `stopAccessingSecurityScopedResource()` while the user is viewing the file. This is required for the Files-app integration on iOS.

### macOS Open Panels

- **Folder/file picker:** `AppState.openFolder()` (`Models/AppState.swift` lines 201–227) uses `NSOpenPanel` filtered to `.md` and `.json`. Returns either a folder (loaded into the sidebar tree) or a single file (opened directly).
- **Sandbox grant picker:** `BookmarkStore.requestAccess(...)` (`Utilities/BookmarkStore.swift` lines 42–63) presents a folder-only `NSOpenPanel` and persists a security-scoped bookmark on the result.
- **Custom AI-source picker:** `AppState.addCustomSource()` (`Models/AppState.swift` lines 125–143) opens a directory `NSOpenPanel` and registers the chosen path as a custom AI source via `AISource.addCustomSource(path:)`.
- **PDF save panel:** `PDFExporter.exportToPDF(...)` (`Utilities/PDFExporter.swift` lines 16–60) uses `NSSavePanel` filtered to `.pdf`.

## Outbound Network Calls

### GitHub Releases API (daily update poll)

- **Caller:** `UpdateChecker` in `ClaudeRecall/Sources/Utilities/UpdateChecker.swift`
- **Endpoint:** `https://api.github.com/repos/ChrisPelatari/Claude-recall/releases/latest`
- **HTTP:** `GET`, `Accept: application/vnd.github+json`, 10 s timeout, `URLSession.shared`
- **Frequency:** At most once per 24 h (`checkInterval: TimeInterval = 60 * 60 * 24`, lines 27 and 38–42). Manual trigger from the Help menu → "Check for Updates…" bypasses the throttle (line 47–49 + `ClaudeRecallApp.swift` lines 157–159).
- **Skipped when sandboxed:** `UpdateChecker.checkIfDue()` returns early if `BookmarkStore.isSandboxed` is true (lines 33–36) — MAS users get updates through the App Store.
- **State persistence (UserDefaults only, not iCloud):**
  - `updateChecker.lastCheckAt` — throttle timestamp
  - `updateChecker.dismissedVersions` — array of tags the user clicked "Skip This Version" on
- **Banner UI:** `Views/UpdateBanner.swift`; "Download" opens the release page in the default browser via `NSWorkspace.shared.open(_:)`. Claude Recall does not download or apply updates in-app.

### Outbound link-outs (open in user's default browser)

Not API calls, but the only other places Claude Recall sends the user externally — all via `NSWorkspace.shared.open(_:)` from the Help menu in `App/ClaudeRecallApp.swift`:

- `https://github.com/ChrisPelatari/Claude-recall#readme` — Help (line 152)
- `https://ko-fi.com/nvwalj` — Donations (line 164)
- `https://github.com/ChrisPelatari/Claude-recall` — Star on GitHub (line 170)
- `https://github.com/ChrisPelatari/Claude-recall/issues/new` — Report an issue (line 178)
- Remote image URLs inside markdown documents — `LocalImageProvider` (`Utilities/LocalImageProvider.swift` lines 27–43) falls back to `AsyncImage` for `http`/`https` image scheme.

## Data Storage

**Local-only. No remote databases, file stores, or backends.**

### Settings (iCloud KV synced)

- **Manager:** `SettingsStore.shared` (`Models/SettingsStore.swift`)
- **Dual-write:** Every `set(_:forKey:)` writes both to `UserDefaults.standard` and `NSUbiquitousKeyValueStore.default`, then calls `iCloud.synchronize()` (lines 85–90). Reads prefer iCloud and fall back to UserDefaults (lines 67–83).
- **External-change observer:** `iCloudDidChange(_:)` merges remote changes into local UserDefaults on `NSUbiquitousKeyValueStoreServerChange` or `InitialSyncChange` reasons (lines 94–112), posting `.settingsDidSyncFromiCloud` so UI can react.
- **Migration:** `migrateLocalToiCloudIfNeeded()` pushes existing local values to iCloud once, gated by `settingsStore_migrated_v1` flag (lines 117–137).
- **Synced keys:**
  - `recentFolders: [String]` — last 5 opened folder paths
  - `customAISourcePaths: [String]` — user-added AI source directories
  - `lastSelectedSourceID: String?` — restore-on-launch selection
  - `lastLocalFolderPath: String?` — restore-on-launch local folder
  - `appTheme: String?` — `.standard` or `.eyeCare`
- **Container identifier:** `$(TeamIdentifierPrefix)$(CFBundleIdentifier)` (both entitlement files). Note that the macOS and iOS targets use different bundle IDs (`com.pelatari.claude-recall` vs `com.pelatari.claude-recall-ios`), so iCloud KV **does not sync between Mac and iPhone** — only across multiple Macs (or across multiple iPhones/iPads).
- **Documents are not synced** — only settings/preferences. AI memory files themselves stay on each device.

### Security-scoped bookmarks (device-local)

- **Manager:** `BookmarkStore.shared` (`Utilities/BookmarkStore.swift`)
- **Storage:** `UserDefaults.standard` key `securityScopedBookmarks_v1`, a `[String: Data]` map keyed by the on-disk path.
- **Why UserDefaults and not SettingsStore:** Security-scoped bookmark blobs are bound to the granting Mac and would fail to resolve elsewhere; iCloud-syncing them would silently break (`BookmarkStore.swift` lines 9–14).
- **Lifecycle:** `AppDelegate.applicationWillFinishLaunching` calls `BookmarkStore.shared.restoreOnLaunch()`; `applicationWillTerminate` calls `stopAllAccess()`.

### Other UserDefaults keys (not iCloud-synced)

- `securityScopedBookmarks_v1` (above)
- `updateChecker.lastCheckAt`
- `updateChecker.dismissedVersions`
- `settingsStore_migrated_v1`

## Authentication & Identity

**None.** Claude Recall has no user accounts, no API keys, no OAuth flow. It reads local files and displays them.

## Monitoring & Observability

**Error tracking:** None.
**Logs:** `print(...)` to stderr from `FileWatcher` and `AppState.handleFileSystemChange`. No structured logging framework, no log aggregation.
**Crash reporting:** Apple's built-in `os` crash logs only.

## Webhooks & Callbacks

**Incoming:** None.
**Outgoing:** None.

## Environment Configuration

**No environment variables are required to run Claude Recall.**

The only env var the app reads is `APP_SANDBOX_CONTAINER_ID`, which macOS sets automatically for sandboxed processes (`BookmarkStore.isSandboxed`, `Utilities/BookmarkStore.swift` lines 23–25). It is not set by the user.

**No `.env` files exist in this repo.** No secrets are stored anywhere in source.

## Privacy Posture (declared)

- `Info.plist` declares no microphone, camera, location, photos, or contacts usage descriptions.
- `ClaudeRecall.entitlements` enables only sandbox, user-selected R/W files, app/document-scoped bookmarks, and iCloud KV (no `com.apple.security.network.client`, so the sandboxed MAS build literally cannot make network calls — which is why `UpdateChecker` is skipped there).
- `PrivacyInfo.xcprivacy` declares `NSPrivacyTracking: false`, no collected data types, and only two accessed API types with reasons: `NSPrivacyAccessedAPICategoryFileTimestamp` (`C617.1` — file watching) and `NSPrivacyAccessedAPICategoryUserDefaults` (`CA92.1` — settings persistence).
- The user-facing privacy doc is `docs/privacy.html`. Any new outbound call must update `PrivacyInfo.xcprivacy`, `docs/privacy.html`, and the App Store privacy nutrition label in `MAS_METADATA.md`.

---

*Integration audit: 2026-05-20*
