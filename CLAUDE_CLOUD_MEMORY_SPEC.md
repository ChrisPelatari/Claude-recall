# Claude Recall Feature: Claude.ai Cloud Memory Sync

Spec for surfacing the user's claude.ai cloud memory inside Claude Recall, on the same sidebar that already lists their local `CLAUDE.md` files.

## Why this matters

Today, Claude Recall shows everything **except** the one place Claude actually persists context about the user across all of their conversations — claude.ai's server-side "Memory" feature. That memory is plain Markdown, ~1-4 KB per user, regenerated nightly, and user-editable. It's the missing piece.

Once we ship this, Claude Recall's positioning sharpens from "viewer for local memory files" → **"single pane of glass for everything Claude remembers about you, local and cloud."**

## Reverse-engineered API (claude.ai web)

The claude.ai web app uses a Cloudflare-fronted REST API under `https://claude.ai/api/`. Auth is **session cookies** — no separate token, no OAuth. Confirmed endpoints:

### 1. Memory settings (read-only check)

```http
GET https://claude.ai/api/organizations/{org_id}/memory/settings
→ 200 application/json
{
  "enabled_saffron":        true,    // master memory toggle
  "enabled_saffron_search": true,    // "Search and reference chats"
  "enabled_melange":        null     // unrelated feature flag
}
```

`"saffron"` is Anthropic's internal codename for the consumer memory feature. The `enabled_*` flags should be checked before attempting to read memory.

### 2. Account-level memory (the main blob)

```http
GET https://claude.ai/api/organizations/{org_id}/memory
→ 200 application/json
{
  "memory":     "<markdown string, typically 1–4 KB>",
  "controls":   null,
  "updated_at": "2026-05-01T..." (ISO 8601)
}
```

- `memory` is the **summary** of everything Claude knows about the user. Plain Markdown.
- Regenerated nightly by Claude (per the modal copy: *"This summary is regenerated each night"*).
- User can edit it in the claude.ai modal — write-back path is **not** at this endpoint (OPTIONS responds `Allow: GET`). Write probably goes through a separate `PATCH /memory/entries/{id}` or `POST /memory/edit` — not yet reverse-engineered. Read-only is enough for v1.

### 3. Project-level memory (likely, not yet confirmed)

claude.ai's modal calls out that **projects have their own memory**, separate from the account blob. By URL convention, this is almost certainly:

```http
GET https://claude.ai/api/organizations/{org_id}/projects/{project_id}/memory
```

Worth confirming when a project is opened in claude.ai while DevTools is attached.

## Authentication on macOS

Three options for a native Swift app:

| Approach | Notes |
|---|---|
| **A. WKWebView one-time login** | Embed a `WKWebView`, navigate to `claude.ai/login`. User logs in once. App captures cookies from the WKWebView's `WKHTTPCookieStore`. Subsequent API calls go through `URLSession` with those cookies. **Recommended.** Persists across launches via `WKWebsiteDataStore.default()`. |
| **B. ASWebAuthenticationSession** | Same UX but blocks the rest of the app during login; less control over the cookie jar. Worse fit for a settings/sync use case. |
| **C. User-supplied org_id + cookie header** | Power-user mode: a hidden "advanced" sheet where user pastes their org UUID + `sessionKey` cookie value. Useful as a fallback for users who don't want to embed a browser. |

Default to **A**. Surface **C** behind a debug-menu flag for issue reporters.

## Discovering `org_id`

Two ways:

1. **`GET /api/bootstrap`** — claude.ai's session bootstrap, returns the current user + their default organization. Confirm structure by sniffing. (Has been the pattern across multiple Anthropic web apps.)
2. **Scrape from a known page** — after login, navigate WKWebView to `claude.ai/settings/capabilities`, evaluate JS to read the org id from a known global (`window.__NEXT_DATA__` or similar).

Prefer #1 if the endpoint is clean.

## UX in Claude Recall

### Sidebar

Add a new top-level sidebar group `Claude.ai (Cloud)` that contains:

- `Account memory` — opens the account-level Markdown blob in the existing rendering view, read-only. Shows the `Updated 17 days ago` timestamp from `updated_at`.
- `Projects/` — expand to list project-scoped memories (lazy-fetch on click). Each project's memory renders like a file.

The cloud group lives next to the existing local groups (Claude Code, Codex, Cursor, etc.) so the user's mental model is *"every place Claude remembers about me, in one tree."*

### First-launch

If the user has never logged in:

> **Connect your claude.ai account**
> Pull your account-level memory and project memories into Claude Recall. Read-only — Claude Recall never writes to your claude.ai memory. [Connect…] [Skip]

Click → WKWebView modal → user logs in once → modal closes → memories appear in sidebar.

### Privacy

- Memory text is **fetched on demand** and held only in-memory; **never persisted to disk** unless the user explicitly chooses "Save snapshot to local CLAUDE.md."
- Cookies stored only in the system keychain (`WKHTTPCookieStore` does this by default for the default data store).
- No analytics on memory content, ever.

### Refresh

- Manual `⌘R` triggers a refetch.
- Automatic refresh on app foreground if the cached `updated_at` is older than 6 hours.

## v1 vs v2

**v1 (this sprint):**
- Read-only account memory via WKWebView-captured cookies
- Sidebar entry, Markdown render
- "Save snapshot to local file" → writes to `~/.claude-recall/cloud-memory.md` for offline reference
- ~1 week of Swift work, mostly in `AppState.swift` / `AISource.swift` / new `CloudMemoryService.swift`

**v2 (later):**
- Project-level memory tree
- Write-back: edit in Claude Recall → PATCH to claude.ai (after reverse-engineering the write endpoint)
- Bidirectional sync local `CLAUDE.md` ↔ claude.ai memory (with conflict prompts)

## Risks

- **ToS:** claude.ai's TOS forbid automated scraping in general, but reading **your own** memory through their **published UI's own API** is squarely "personal data access," not scraping. We'll add a clear "no third-party services involved — your cookies stay on your device" note in the about dialog.
- **Endpoint drift:** Anthropic could rename/restructure these endpoints (especially the `saffron` codename). Mitigation: ship feature flags, fail soft to a "claude.ai memory unavailable" empty state, ship updates fast.
- **Two-factor / SSO flows:** WKWebView handles them natively; tested in prior Anthropic web flows.

## Marketing angle

This single feature unlocks the Claude Recall README headline:

> **The only viewer for both your local AND your claude.ai cloud memory.**

No competitor in the Claude Code / claude.ai tooling space does this. It's also the kind of "wait, I needed that" feature that gets word-of-mouth — exactly the marketing the project has been lacking.
