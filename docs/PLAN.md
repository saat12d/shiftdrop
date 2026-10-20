# ShiftDrop — full plan

## Context

Build a native macOS menu-bar utility: drag a file from Finder, hold **Shift**, and a radial wheel appears under the cursor. Drop the file on a wedge to convert it (PNG / JPG / HEIC / WEBP / TIFF / BMP / GIF / PDF / TXT) or run a tool (crop, compress, strip metadata, merge PDFs). Output is written next to the original. Then share it with friends as a free download from GitHub.

Decisions made with the user:
- **Distribution:** free, no Apple Developer account. Ad-hoc signed DMG + zip on GitHub Releases.
- **v1 scope:** images + PDF + tools, plus **PDF merge**. No video/audio/DOCX in v1.
- **Public GitHub repo.**
- **Swift + full Xcode.** AppKit for windowing and drag plumbing, SwiftUI for the wheel and crop UI.

Starting point: `shiftdrop/` is empty. Machine: macOS 15.5, Command Line Tools only (Swift 6.0.3), Homebrew present, no Xcode, no ffmpeg/ImageMagick.

## Verified facts that shape the design

| Fact | Consequence |
|---|---|
| Global `NSEvent` monitors for **mouse** events (`leftMouseDragged`, `leftMouseUp`) need no permission; only key events need Accessibility | Detect drags from mouse events, never from key events |
| `NSEvent.modifierFlags` and `NSEvent.mouseLocation` need no permission | Read Shift state by polling inside the drag handler |
| `NSPasteboard(name: .drag)` is readable; Finder drags carry `public.file-url` | Detect "a file drag is in progress" via drag-pasteboard `changeCount` |
| A pre-created non-activating `NSPanel` registered for `.fileURL` receives drops mid-drag | The wheel is a panel created at launch and shown on demand |
| ImageIO on 15.5 **writes** PNG, JPEG, GIF, TIFF, BMP, HEIC; **WebP and AVIF are read-only** | WebP output needs bundled libwebp (SwiftPM); AVIF output is out of v1 |
| PDFKit extracts text, renders pages, and can insert pages across documents | PDF→TXT, PDF→images, images→PDF, PDF merge all free of dependencies |
| Sequoia: Control-click → Open no longer bypasses Gatekeeper; users need System Settings → Privacy & Security → **Open Anyway**. Stripping `com.apple.quarantine` skips it entirely. Apple silicon refuses fully unsigned binaries | Ship ad-hoc signed; provide clear Open Anyway instructions **and** a one-line install script that strips quarantine |
| Homebrew casks apply quarantine and `--no-quarantine` is gone | A tap is optional polish, not the main install path |
| Sparkle under ad-hoc signing is awkward (library validation, quarantine on updates) | v1 update check = ping GitHub Releases API and open the release page |

## Product spec (v1)

**Trigger.** Start dragging one or more files in any app. Hold Shift. Wheel appears centred under the cursor. Release Shift or end the drag → wheel hides. Drop on a wedge → action runs.

**Wheel.** Two pages, flipped by hovering the centre hub for ~0.5 s (keys cannot be used without Accessibility). Centre hub shows file name + size and a page indicator. Wedges are **context-aware**: only actions valid for *all* dragged items appear.

| Dragged | Convert page | Tools page |
|---|---|---|
| Image (png/jpg/heic/tiff/bmp/gif/webp/avif) | PNG, JPG, WEBP, HEIC, TIFF, BMP, GIF, PDF | Crop, Compress, Strip metadata |
| PDF | PNG, JPG (one image per page), TXT | Compress, Strip metadata |
| 2+ PDFs | — | **Merge PDFs** (Finder selection order) |
| Mix of images + PDFs | PDF (combine all into one) | — |
| Anything else | wheel shows "No actions for this file" | |

**Output.** Written next to the source, never overwriting: `Photo.jpg` → `Photo.png`; collisions → `Photo 2.png`. Tools use suffixes: `Photo Cropped.jpg`, `Photo Compressed.jpg`, `Photo (no metadata).jpg`, `Merged.pdf`. Multi-page PDF → images go in a folder `Photo pages/`. Completion shows a small toast with "Reveal in Finder"; failures show the error in the toast.

**Crop** opens a window (as in the reference screenshot): image preview, drag handles, aspect-ratio presets (Free, 1:1, 16:9, 9:16, 4:3, 3:4), W/H pixel fields, Reset, Apply. Images only in v1.

**Compress.** Images: re-encode as JPEG/HEIC at quality 0.6 (same format kept where lossy, PNG → downscale to ≤2048px + PNG). PDF: Quartz "Reduce File Size" filter via `QuartzFilter`. Toast shows before→after size.

**Strip metadata.** Images: rewrite via ImageIO with EXIF/GPS/IPTC/XMP removed. PDF: clear `documentAttributes`.

**Menu bar.** Status item with: Enabled toggle, Open at Login (`SMAppService`), Settings (output folder: next to file / Downloads; JPEG quality; wheel size), Check for Updates, Quit. `LSUIElement = true` (no Dock icon). First-launch onboarding window shows an animated "drag + hold Shift" hint.

**Permissions.** None requested. First write into Desktop/Documents/Downloads triggers macOS's standard folder-access prompt once.

**Deployment target:** macOS 14.0.

## Architecture

```
shiftdrop/
  project.yml                 # XcodeGen spec → ShiftDrop.xcodeproj (generated, gitignored)
  ShiftDrop/
    App/            AppDelegate, StatusItemController, SettingsView, OnboardingWindow, UpdateChecker
    DragWatcher/    DragMonitor (global mouse monitor + drag pasteboard + Shift polling), DragSession model
    Wheel/          WheelPanel (NSPanel), WheelDropView (NSView drag destination, wedge hit-testing),
                    WheelView (SwiftUI drawing), WheelPage/Wedge models, ActionResolver (files → wedges)
    Actions/        Action protocol; ImageConvert, PDFToImages, PDFToText, ImagesToPDF, MergePDF,
                    Compress, StripMetadata, Crop; OutputNamer; ActionRunner (async, progress, errors)
    Tools/Crop/     CropWindow, CropView (SwiftUI), CropModel
    UI/             Toast panel, shared styles
    Resources/      Assets.xcassets (icon), Info.plist, ShiftDrop.entitlements (empty), Localizable
  ShiftDropTests/   unit tests for OutputNamer, ActionResolver, each Action with fixture files
  Fixtures/         tiny sample png/jpg/heic/webp/pdf files
  scripts/          build.sh (xcodebuild → ad-hoc codesign → zip + DMG), install.sh (friends' one-liner)
  .github/workflows/release.yml   # on tag v*: build, package, upload to GitHub Release
  README.md, LICENSE (MIT), CHANGELOG.md
```

Key mechanics:
- **DragMonitor:** `addGlobalMonitorForEvents(.leftMouseDragged)`; on each event, if drag pasteboard `changeCount` changed since last `leftMouseUp` → a drag is live; read file URLs once; if `NSEvent.modifierFlags.contains(.shift)` and wheel hidden → show at `NSEvent.mouseLocation`. On `leftMouseUp` or Shift released → hide. Also add a *local* monitor so drags over our own panel keep updating. Debounce so the wheel does not flicker.
- **WheelPanel:** created at launch; `.nonactivatingPanel`, `.borderless`, `level = .statusBar`, transparent background, `collectionBehavior = [.canJoinAllSpaces, .fullScreenAuxiliary, .transient]`, `hidesOnDeactivate = false`. Shown with `orderFrontRegardless()`. `registerForDraggedTypes([.fileURL])` is called after the view is in the window. Hit-testing in `draggingUpdated` converts cursor → polar coords → wedge index, published to the SwiftUI view for highlight.
- **Action protocol:** `id`, `title`, `symbol`, `appliesTo(items: [DragItem]) -> Bool`, `run(items, options) async throws -> [URL]`. `ActionResolver` builds the two pages from the registry. Runner executes on a background task, posts progress to the toast.
- **WebP encoding:** SwiftPM dependency on `ainame/Swift-WebP` (libwebp). Decoding of WebP/AVIF uses ImageIO.
- **Multi-file drags:** every action receives `[DragItem]`; converters map over items; MergePDF requires ≥2 PDFs.

## Milestones

**M0 — Setup (half a day, mostly waiting on downloads)**
- Install Xcode from the App Store (or xcodes), `xcode-select` to it, `brew install xcodegen create-dmg`.
- `git init`, create public GitHub repo `shiftdrop`, MIT license, `.gitignore` for Xcode/DerivedData/generated xcodeproj.
- XcodeGen `project.yml`, empty menu-bar app that builds and shows a status item.

**M1 — Spike the risky part (1 day)**
- DragMonitor + bare WheelPanel: drag a file, hold Shift, a panel appears and logs `draggingEntered` / drop URLs. Prove no permission prompts on a clean user account. Verify behaviour over a full-screen Space and when the cursor is stationary when the panel appears (may need a 1-px nudge tolerance).
- Go/no-go gate: if drops are not received mid-drag, fall back to CGEventTap (needs Input Monitoring) before building anything else.

**M2 — Wheel + image conversions (2–3 days)**
- SwiftUI wheel with wedges, highlight, two pages, hub hover flip, context-aware wedges.
- ImageConvert (ImageIO + libwebp), ImagesToPDF, OutputNamer, toast with Reveal. Unit tests with fixtures.
- First usable build for yourself.

**M3 — PDF + tools (2–3 days)**
- PDFToImages, PDFToText, MergePDF, Compress (image + PDF), StripMetadata, Crop window.
- Error handling: unreadable files, read-only folders, iCloud-not-downloaded files.

**M4 — Polish (1–2 days)**
- App icon, onboarding window, Settings, Open at Login, Check for Updates (GitHub Releases API), Enabled toggle, Quit. Memory/CPU check while idle (monitor must be cheap).

**M5 — Ship (1 day)**
- `scripts/build.sh`: `xcodebuild -configuration Release`, `codesign --force --deep -s -` (ad-hoc), zip, `create-dmg` with drag-to-Applications background.
- `scripts/install.sh`: downloads latest release zip, unzips to `/Applications`, runs `xattr -dr com.apple.quarantine`, opens the app. Friends run one `curl … | bash` line.
- GitHub Actions `release.yml` on tag push: build, package, attach DMG + zip + checksums to the Release.
- README: GIF demo, "Install" with both paths (DMG + Open Anyway steps with screenshots for Sequoia; one-liner script), "Why does macOS warn?" explanation, supported conversions table.
- Tag `v0.1.0`, send friends the link.

**Later / optional:** Homebrew tap cask (`homebrew-tap` repo), Sparkle, draggable result chip (chain convert → tools like the reference video), Annotate/Redact/Add background, video/audio via AVFoundation, AVIF encode, landing page on Vercel, buy the $99 account if adoption grows (removes all Gatekeeper friction and enables Sparkle cleanly).

## Risks and mitigations

- **Gatekeeper friction for friends** is the biggest non-technical risk. Mitigation: install script + screenshot instructions; mention the $99 upgrade path.
- **Ad-hoc signature changes every build** → macOS folder-access prompts reappear for *you* after each rebuild (friends with one build are fine). Acceptable; note in CONTRIBUTING.
- **Wheel appearing under a stationary cursor** may not receive drag-enter until the cursor moves. Mitigation: show the wheel slightly offset or accept; test in M1.
- **Drag pasteboard false positives** (text drags, in-app drags): require `public.file-url` and at least one existing path.
- **Apple-silicon-only vs universal:** build universal (arm64 + x86_64) in CI so Intel friends work; libwebp supports both.
- **Large files / many pages:** run actions off the main thread with progress; cap PDF→image DPI at 150 by default.

## Verification

- Unit tests (`xcodebuild test`): OutputNamer collision rules; ActionResolver tables above; each Action round-trips fixture files and the output opens with ImageIO/PDFKit; MergePDF page count and order; StripMetadata leaves no EXIF/GPS keys.
- Manual checklist on your Mac: trigger from Finder, Desktop, and a Mail attachment; multi-file drag; full-screen Space; Shift released mid-drag hides wheel; drop outside the wheel does nothing; idle CPU ≈ 0% in Activity Monitor; no permission prompts beyond folder access.
- Fresh-machine test: new macOS user account (or a friend's Mac), download the DMG from the GitHub Release, follow README's Open Anyway steps; then test `install.sh` path separately. Both must end with a working wheel.
- CI green on the tag build; release assets downloadable and checksums match.
