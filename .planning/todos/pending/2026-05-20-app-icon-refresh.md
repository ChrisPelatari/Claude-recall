---
created: 2026-05-20T06:25:55Z
title: App icon refresh for Claude Recall fork
area: ui
files:
  - ClaudeRecall/Sources/Resources/Assets.xcassets/AppIcon.appiconset/icon.png
  - ClaudeRecall/Sources/Resources/Assets.xcassets/AppIcon.appiconset/AppIcon.png
  - ClaudeRecall/Sources/Resources/Assets.xcassets/AppIcon.appiconset/Contents.json
  - home.png
  - README.md
---

## Problem

After the rename from AI Memory Reader → Claude Recall (commit `2104b8c`), the app still ships the upstream-designed icon and the upstream-styled `home.png` screenshot in the README. The product display name now says "Claude Recall" but the icon and hero image still look like upstream's branding — a visual mismatch that gets worse the more the fork diverges.

Personal-use project, so there are no marketing/brand-system constraints. Goal is simply a 1024×1024 PNG that *looks like* Claude Recall rather than the inherited upstream brand. Note also that `nvwalj/ai-memory-reader` retains copyright on the existing icon under GPL-3.0 — a fresh original icon avoids any visual-asset-reuse ambiguity even though the GPL permits redistribution.

## Solution

TBD. Open questions for whoever picks this up:

1. **Design direction.** Default: ride on Anthropic's "Claude" association since the name implies it. The Claude API style guide has guidance for third-party apps that reference Claude. A minimalist mark (e.g. a stylized C with a clock/recall motif, or a memory-ribbon glyph) on a flat tinted background tends to read well at all macOS icon sizes.
2. **Tool.** SF Symbols + a vector editor (Figma, Sketch, Affinity Designer, or even Inkscape) → export 1024×1024 PNG. Or generate via image AI (Nano Banana / DALL-E / Ideogram) and clean up in a vector tool.
3. **Implementation steps** once the PNG exists:
   - Replace `ClaudeRecall/Sources/Resources/Assets.xcassets/AppIcon.appiconset/icon.png` and `AppIcon.png` with the new 1024 master
   - Verify `Contents.json` in that appiconset still references the correct filename (it may; the previous setup used a single 1024 source and let Xcode 14+ auto-derive smaller sizes)
   - `xcodegen generate` is not required (resources only)
   - Rebuild: `xcodebuild -project ClaudeRecall.xcodeproj -scheme ClaudeRecall -destination 'platform=macOS,arch=arm64' -configuration Debug build`
   - Confirm new icon shows in `Claude Recall.app` (Finder may cache; `killall Finder` if needed)
4. **`home.png`** — repo-root screenshot used by README hero. Capture a fresh screenshot of the running app under the new branding once the icon lands. Recommended capture size: 2× the displayed width in the README (≈ 2000–2400px wide on a Retina Mac).
5. **Optional**: add a 16/32/128/256/512/1024 set explicitly rather than relying on auto-derive — gives sharper small-size rendering. Use [`iconutil`](https://developer.apple.com/library/archive/documentation/GraphicsAnimation/Conceptual/HighResolutionOSX/Optimizing/Optimizing.html#//apple_ref/doc/uid/TP40012302-CH7-SW2) or any icon generator.

Defer until Chris is ready to spend design time. This is cosmetic, not blocking any functionality.
