# l2a — Link to Audio

> CLI tool to convert YouTube links or search keywords into audio files, written in pure Go.

## Features

- Convert YouTube URLs or search keywords to audio files
- Support multiple audio formats: MP3, M4A, FLAC, WAV, OPUS
- Progress bar during download
- Resume interrupted downloads
- Intelligent stream selection (best quality available)

## Installation

### From source

```bash
git clone https://github.com/yourusername/link2audio.git
cd link2audio
make build
sudo make install
```

### Prerequisites

- Go 1.21+
- ffmpeg (must be installed)

### Install ffmpeg

**macOS:**
```bash
brew install ffmpeg
```

**Ubuntu/Debian:**
```bash
sudo apt install ffmpeg
```

**Windows:**
Download from [ffmpeg.org](https://ffmpeg.org/download.html) and add to PATH.

## Usage

```bash
l2a                           # Interactive mode
l2a -k 3                      # Show top 3 search results
l2a -o ~/Music               # Custom output directory
l2a -k 5 -o ~/Music          # Combine flags
```

### Interactive flow

```
$ l2a

> Paste YouTube link or enter search keyword: lo-fi study music

  Searching...

  [1] Lo-Fi Hip Hop Radio 📻   — ChilledCow          — LIVE
  [2] lofi hip hop mix         — Lofi Girl            — 1:02:34
  [3] Study With Me 4 Hours    — The Studi            — 4:00:01

> Select video [1-3]: 2

> Select format:
  [1] mp3   — lossy, smallest (~3-5 MB/min)
  [2] m4a   — lossy, better quality (~4-6 MB/min)
  [3] flac  — lossless, larger (~20-30 MB/min)
  [4] wav   — lossless, uncompressed (~50 MB/min)
  [5] opus  — lossy, smallest & high quality (~2-3 MB/min)

> Select format [1-5]: 1

  Downloading...  [=========>    ] 73%

  Saved: ~/Music/lofi-hip-hop-mix.mp3
```

### Direct URL mode

```bash
$ l2a
> Paste YouTube link or enter search keyword: https://youtu.be/dQw4w9WgXcQ

  Selected: Rick Astley - Never Gonna Give You Up
  ...
```

## Configuration

| Flag | Default | Description |
|------|---------|-------------|
| `-o, --output` | `~/Music` | Output directory |
| `-k, --top` | `5` | Number of search results |

## Supported Formats

| Format | Codec | Size (per minute) | Use case |
|--------|-------|------------------|----------|
| MP3 | libmp3lame | ~3-5 MB | Universal compatibility |
| M4A | AAC | ~4-6 MB | Apple devices, better quality |
| FLAC | FLAC | ~20-30 MB | Lossless archiving |
| WAV | PCM | ~50 MB | Maximum quality, no compression |
| OPUS | libopus | ~2-3 MB | Smallest size, streaming |

## Architecture

```
input.go → [URL?] → innertube/client.go → download.go → ffmpeg.go
         → [keyword?] → search.go → [user selects] → innertube/client.go ...
```

See [docs/implementation-plan.md](docs/implementation-plan.md) for detailed technical specification.

## Troubleshooting

| Issue | Solution |
|-------|----------|
| "ffmpeg not found" | Install ffmpeg: `brew install ffmpeg` (macOS) or `sudo apt install ffmpeg` (Linux) |
| "Video unavailable" | Video may be geo-restricted or removed |
| "Signature decrypt failed" | Try again later (YouTube updates frequently) |
| "Network error" | Check your internet connection |

## License

MIT