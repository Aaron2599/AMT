# AMT (Agarthean Media Tools) README

AMT is a desktop application for Downloading and compressing media files to under 10MB using advanced encoding techniques. It features a drag-and-drop interface for easy file selection, with options enabled by default that copy compressed outputs directly to the clipboard.

## Features

- Drag and drop files or for compression.
- Under 10MB video optimization with modern codecs.
- Real-time progress indicators during processing.
- Clipboard integration for seamless results sharing.
- Notification system for completion and errors.

The app processes files efficiently in the background, leveraging optimized FFmpeg pipelines for high-quality compression.

## Quick Start

1. Download the latest release from the [Releases page](https://github.com/Aaron2599/AMT/releases).
2. Install the app on your system (available for Windows).
3. Launch AMT and drag a video file into the interface or paste a URL to download files.
4. Wait for compression to complete, then access your result from the clipboard or file explorer.

No additional setup required; it's built for instant use in media workflows.

## Releases

### v0.1.0-beta (Latest)

- Improved compression algorithms for better file size reduction without quality loss.
- Added URL download support for remote media.
- Enhanced UI with dark theme and notifications.
- Bug fixes for large file handling.
  Download: [v0.1.0-beta](https://github.com/Aaron2599/AMT/releases/tag/v0.1.0-beta)


### v0.0.1-alpha

- Initial drag-and-drop compression interface.
- Basic file processing and clipboard export.
- Core media optimization engine.
  Download: [v0.0.1-alpha](https://github.com/Aaron2599/AMT/releases/tag/v0.0.1-alpha)

These releases mark progressive improvements, starting from core functionality in alpha to polished beta features.

## Building from Source

AMT is built with modern web technologies including Tauri for desktop packaging, SvelteKit for the frontend.

### Prerequisites

- Bun
- Rust
- Tauri CLI

### Setup

```
git clone https://github.com/Aaron2599/AMT.git
cd AMT
bun i
bun run tauri dev  # For development
bun run tauri build  # For production release
```

The build process compiles the Svelte frontend and bundles it into a native executable using Tauri's webview.

## Technical Details

Under the hood, AMT uses FFmpeg-compatible pipelines for video compression (SVT-AV1) and optimizes parameters for your media type. Svelte 5 powers the reactive UI.

## Contributing

- Fork the repo and create a feature branch.
- Submit pull requests with tests.
- Focus areas: expansion to general audio, video, image compression, image support, UI enhancements & features, performance tweaks, quality & speed sliders.

Issues and PRs welcome for optimization enthusiasts.

## License

See LICENSE file for details.
