# Agarthian Media Tools
![logo](src-tauri/icons/Square150x150Logo.png)

AMT is a desktop application for Downloading and compressing media files to under 10MB using advanced encoding techniques. It features a drag-and-drop interface for easy file selection, with options enabled by default that copy compressed outputs directly to the clipboard.

## Features

- Under 10MB video optimization with modern codecs.
- Video downloading from most major platforms.
- Drag and drop files or for compression.
- Progress indicators during processing.
- Clipboard integration for instant sharing.
- Notification system for completion and errors.

The app processes files efficiently in the background, leveraging optimized pipelines for high-quality compression.

## Quick Start

<img src="/static/ui.png" hspace="11rem" width="358rem" >

- Download the latest release from the [Releases page](https://github.com/Aaron2599/AMT/releases).
- Install the app on your system (available for Windows).
- Launch AMT and drag a video file into the interface or paste a URL to download files.
- Wait for compression to complete, then access your result from the clipboard or file explorer.

No additional setup is required.

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
