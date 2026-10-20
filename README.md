# ShiftDrop

Drag a file. Hold **Shift**. Drop it on what you want it to become.

ShiftDrop is a small macOS menu-bar app. While you are dragging a file from Finder (or anywhere else), hold Shift and a radial wheel appears under your cursor. Drop the file on a wedge to convert it or run a tool. The result is saved next to the original; nothing is ever overwritten.

> Status: in development. Nothing to download yet.

## What it can do (v1)

| You drag | Convert to | Tools |
|---|---|---|
| An image (PNG, JPG, HEIC, TIFF, BMP, GIF, WebP, AVIF) | PNG, JPG, WebP, HEIC, TIFF, BMP, GIF, PDF | Crop, Compress, Strip metadata |
| A PDF | PNG or JPG (one per page), TXT | Compress, Strip metadata |
| Two or more PDFs | | Merge into one PDF |
| Images and PDFs together | One combined PDF | |

Multiple files at once are fine. The wheel only shows actions that work for everything you are dragging.

## Install

Coming with the first release. There will be two ways:

1. **Download the DMG** from the Releases page and drag ShiftDrop to Applications. Because ShiftDrop is a free, unsigned app, macOS will warn you the first time. On macOS 15 open **System Settings → Privacy & Security**, scroll down, and click **Open Anyway**. You only do this once.
2. **One-line install** that downloads the latest release, puts it in Applications, and clears the warning:

   ```bash
   curl -fsSL https://raw.githubusercontent.com/saat12d/shiftdrop/main/scripts/install.sh | bash
   ```

ShiftDrop asks for no permissions. The first time it saves into your Desktop, Documents or Downloads, macOS shows its normal folder-access prompt.

## Why does macOS warn about it?

Apple charges 99 USD a year to notarize apps. ShiftDrop is a free hobby project, so it is not notarized. The source is all here; build it yourself if you prefer.

## Building from source

Requirements: macOS 14 or later, Xcode 16 or later, [XcodeGen](https://github.com/yonaskolb/XcodeGen).

```bash
brew install xcodegen
xcodegen generate
open ShiftDrop.xcodeproj
```

See [docs/PLAN.md](docs/PLAN.md) for the design and roadmap.

## License

MIT. See [LICENSE](LICENSE).
