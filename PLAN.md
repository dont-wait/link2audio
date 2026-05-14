# l2a — Implementation Plan

> CLI tool to convert YouTube links or search keywords into audio files, written in pure Go.

---

## Overview

- **Repo**: `link2audio`
- **Language**: Go 1.21+
- **Binary**: `l2a`
- **External dependency**: ffmpeg (runtime)

---

## M1 — CLI Entry Point & Input Module

### Goal
Cobra CLI accepts input, distinguishes URL vs keyword, prints to terminal.

### Technologies

| Component | Choice | Reason |
|---|---|---|
| CLI framework | `github.com/spf13/cobra` | Standard, stable, has flag/command structure |
| Input reading | `bufio.Scanner` + `fmt.Fscan` | No external libraries needed |

### Structure

```
cmd/
├── main.go          # Cobra setup, flags: -o, -k
internal/
├── input/
│   └── input.go     # Read(), ParseVideoID(), InputType enum
```

### Implementation notes

**input.go**
```go
InputType: URL or Keyword
Read(): reads stdin, classifies via regex/prefix check
ParseVideoID(): handle formats:
  - youtube.com/watch?v=ID
  - youtu.be/ID
  - youtube.com/watch?v=ID&t=120
  - m.youtube.com/watch?v=ID
```

**Expected output**
```
$ l2a -k 3
Paste YouTube link or enter search keyword: lo-fi study
→ classified as: [KEYWORD] "lo-fi study"
$ l2a
Paste YouTube link or enter search keyword: https://youtu.be/dQw4w9WgXcQ
→ classified as: [URL] videoID="dQw4w9WgXcQ"
```

---

## M2 — Search Module

### Goal
Call YouTube search API, display top K results, user selects one.

### Technologies

| Component | Choice | Reason |
|---|---|---|
| HTTP client | `net/http` (stdlib) | No external libraries needed |
| JSON parsing | `encoding/json` (stdlib) | Go stdlib is powerful enough |
| Progress UI | ANSI escape codes (self-implemented) | Avoid dependency for simple progress bar |

### Implementation notes

**API endpoint:**
```
POST https://www.youtube.com/youtubei/v1/search
Content-Type: application/json

{
  "context": {
    "client": {
      "clientName": "WEB",
      "clientVersion": "2.20231121.01.00"
    }
  },
  "query": "<keyword>"
}
```

**Response parsing path:**
```
contents.twoColumnSearchResultsRenderer.primaryContents.sectionListRenderer.contents[].itemSectionRenderer.contents[].videoRenderer
```

**Fields to extract:**
- `videoId` → Video.ID
- `title.runs[0].text` → Video.Title
- `ownerText.runs[0].text` → Video.Channel
- `lengthText.simpleText` → Video.Duration ("1:02:34")
- `badges[0].metadataBadgeRenderer.label` → check "LIVE"

**Expected output:**
```
Paste YouTube link or enter search keyword: lo-fi study music
  Searching...

  [1] Lo-Fi Hip Hop Radio 📻   — ChilledCow          — LIVE
  [2] lofi hip hop mix         — Lofi Girl            — 1:02:34
  [3] Study With Me 4 Hours    — The Studi            — 4:00:01

  Select video [1-3]: 2
```

### Error handling
- Network timeout → retry 1 time with exponential backoff (500ms, 1s)
- Parse JSON fail → return error with raw response for debugging
- Empty results → print "No results found"

---

## M3 — Innertube Player Client

### Goal
Fetch stream URLs from YouTube for a specific videoId.

### Technologies

| Component | Choice | Reason |
|---|---|---|
| HTTP client | `net/http` (stdlib) | Consistent with M2 |
| JSON parsing | `encoding/json` (stdlib) + manual parse | AdaptiveFormats has deep nested structure |

### Implementation notes

**API endpoint (ANDROID client — fewer sig requirements than WEB):**
```
POST https://www.youtube.com/youtubei/v1/player?key=AIzaSyAO_FJ2SlqU8Q4STEHLcyilTtj-1Q6R2uI
Content-Type: application/json

{
  "videoId": "<videoID>",
  "context": {
    "client": {
      "clientName": "ANDROID",
      "clientVersion": "17.31.35",
      "androidSdkVersion": 30
    }
  }
}
```

**Parsing streamingData.adaptiveFormats:**
```go
type AudioStream struct {
    URL         string
    MimeType    string  // "audio/webm; codecs=\"opus\""
    Bitrate     int
    ContentLen  int64
    Cipher      string  // signatureCipher URL-encoded
    Signature   string  // direct signature
}
```

**SelectBestStream logic:**
1. Filter audio streams only (mimeType starts with "audio/")
2. Sort by bitrate descending
3. Prefer order: opus > mp4a > vorbis
4. Skip streams with Cipher (not yet decrypted) → pass to crypto module

**Expected output:**
- Get stream URL for videos without sig cipher
- Return error for videos with sig cipher (for M6 to handle)

---

## M4 — Download Module with Progress

### Goal
Download audio stream to temp file, display progress bar.

### Technologies

| Component | Choice | Reason |
|---|---|---|
| HTTP client | `net/http` (stdlib) | Consistent |
| Progress bar | ANSI escape codes (self-implemented) | Lightweight, no dependency |
| Chunk size | 32KB buffer | Balance between memory and performance |

### Implementation notes

**Required headers:**
```
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36
Referer: https://www.youtube.com/
Origin: https://www.youtube.com
```

**Download logic:**
```go
// 1. Check file exists, get current size (resume support)
// 2. HEAD request to get Content-Length
// 3. GET with Range header
// 4. Write to file, update progress callback
```

**Progress bar format:**
```
Downloading...  [==========>       ] 78%  (12.3 MB / 15.8 MB)
```

**Output path:**
```
~/.cache/l2a/tmp/<videoID>.<ext>  // .webm or .m4a depending on mimeType
```

**Error cases:**
- HTTP 403 → video needs signature decrypt (pass to M6)
- HTTP 404 → video unavailable
- Connection reset → retry with resume

---

## M5 — Convert Module (ffmpeg)

### Goal
Convert temp file to user-selected format (mp3, m4a, flac, wav, opus).

### Technologies

| Component | Choice | Reason |
|---|---|---|
| ffmpeg bindings | `exec.Command` | No native ffmpeg binding in Go, subprocess is best |
| Quality settings | Built-in presets | Optimize size vs quality |

### FFmpeg commands

| Format | Command | Quality |
|---|---|---|
| mp3 | `ffmpeg -i input -vn -acodec libmp3lame -q:a 2 output.mp3` | ~3-5 MB/min |
| m4a | `ffmpeg -i input -vn -acodec aac -b:a 192k output.m4a` | ~4-6 MB/min |
| flac | `ffmpeg -i input -vn -acodec flac output.flac` | ~20-30 MB/min |
| wav | `ffmpeg -i input -vn -acodec pcm_s16le output.wav` | ~50 MB/min |
| opus | `ffmpeg -i input -vn -acodec libopus -b:a 128k output.opus` | ~2-3 MB/min |

### SanitizeFilename logic
- Remove: `/ \ : * ? " < > |`
- Replace multiple spaces with single space
- Trim leading/trailing whitespace
- Max length: 200 chars

### Output path
```
<outputDir>/<sanitized_title>.<format>
Default: ~/Music/<sanitized_title>.mp3
```

### FFmpeg check
Check ffmpeg exists and has correct codecs:
```bash
ffmpeg -version | grep -E "libmp3lame|aac|flac|opus"
```

**Expected output:**
```
  Converting... ✓
  Saved: ~/Music/lofi-hip-hop-mix.mp3
```

---

## M6 — Crypto / Signature Decryption

### Goal
Decrypt YouTube signature to get actual downloadable URL.

### Technologies

| Component | Choice | Reason |
|---|---|---|
| Regex | `regexp` (stdlib) | Extract functions from JS |
| String manipulation | `strings`, `strconv` (stdlib) | Transform operations |
| Caching | os filesystem | `~/.cache/l2a/player_*.js` |

### Signature cipher format

`signatureCipher` is URL-encoded string containing:
```
s=<encoded_signature>&sp=sig&url=<base_url>
```

### Decryption algorithm

**Transform functions** (from `base.js`):

| Function | JS equivalent | Go implementation |
|---|---|---|
| reverse | `a.reverse()` | `reverse(s string) string` |
| splice | `a.splice(0, n)` | `splice(s string, n int) string` |
| swap | `a[0]=a[n%len]; a[n]=a[0]` | `swap(s string, n int) string` |

**Algorithm extraction from JS:**
1. Find `set("signature", a.reverse)` or `sig||...` pattern
2. Extract helper object name (e.g., `DE` in `DE.AJ(a,15)`)
3. Find `var DE={(...)};` block
4. Parse each function: `AJ:function(a){a.reverse()}`

**Transform operations parsing:**
```go
// From JS: mw.p7(a,34);mw.qM(a,13);
// Parse to: [{op:"splice", arg:34}, {op:"swap", arg:13}]
// Apply sequentially to signature string
```

### Player JS caching

```go
// Cache key: hash of playerJSURL (e.g., en_US-vflz7mN68)
// Path: ~/.cache/l2a/player_<hash>.js
// TTL: 24 hours (YouTube changes player JS frequently)
```

### n-param throttling

If stream URL has `n=` param that's throttled:
1. Find function to transform `n` in player JS (pattern: `n=...split("")`)
2. Apply same transformation → replace `n` param

### Reference implementations

- **kkdai/youtube**: `decipher.go` — Go implementation, read source to understand transform map extraction
- **yt-dlp**: `extractor/youtube.py` function `_decrypt_signature` and `_get_download_url`
- **pytube/cipher.py**: Python implementation, read to understand transform map parsing

**Expected output:**
- Stream URL without sig → M3/M4 handles immediately
- Stream URL with sig → M6 decrypts, passes new URL to M4

---

## M7 — Polish & Error Handling

### Goal
All edge cases, UX improvements, Vietnamese error messages.

### Features

| Feature | Implementation |
|---|---|
| Resume download | HEAD request check existing file size, Range header |
| Cache player JS | Filesystem cache with TTL |
| Error messages VN | Error wrapper with i18n-style messages |
| Retry logic | Exponential backoff: 500ms, 1s, 2s, max 3 attempts |
| Cleanup | Delete tmp file after conversion completes |

### Error messages mapping

```go
var ErrorMessages = map[string]string{
    "ffmpeg_not_found":      "ffmpeg chưa được cài. Cài bằng: brew install ffmpeg",
    "network_error":         "Không thể kết nối. Kiểm tra kết nối mạng.",
    "geo_restricted":        "Video này không khả dụng ở khu vực của bạn.",
    "video_unavailable":     "Không tìm thấy video. Kiểm tra lại đường dẫn.",
    "disk_full":             "Không đủ dung lượng. Cần thêm ít nhất %d MB.",
    "sig_decrypt_failed":    "Không thể giải mã stream URL. Thử lại sau.",
    "signature_not_found":   "Video không có stream audio.",
}
```

### Retry logic

```go
func withRetry(ctx context.Context, fn func() error, maxAttempts int) error {
    for i := 0; i < maxAttempts; i++ {
        if err := fn(); err == nil {
            return nil
        }
        time.Sleep(time.Duration(1<<uint(i)) * 500 * time.Millisecond)
    }
    return ErrMaxRetriesExceeded
}
```

---

## Dependencies Summary

### Go modules (go.mod)

```go
module link2audio

go 1.21

require (
    github.com/spf13/cobra v1.8.1  // CLI framework
)
```

### External runtime

| Tool | Version | Purpose |
|---|---|---|
| ffmpeg | latest | Audio conversion |
| - | - | No other Go dependencies |

---

## New File Structure

```
l2a/
├── cmd/
│   └── main.go               # Entry point, cobra setup
│
├── internal/
│   ├── input/
│   │   └── input.go          # Read stdin, distinguish URL/keyword
│   │
│   ├── search/
│   │   └── search.go         # Call YouTube search API
│   │
│   ├── innertube/
│   │   ├── client.go         # Call /player endpoint
│   │   └── types.go          # PlayerResponse, AudioStream types
│   │
│   ├── crypto/
│   │   ├── sig.go            # Decrypt signature cipher
│   │   ├── transforms.go     # reverse, splice, swap operations
│   │   └── js.go             # Parse player JS, extract functions
│   │
│   ├── download/
│   │   └── download.go       # HTTP download with progress
│   │
│   ├── convert/
│   │   └── ffmpeg.go         # Convert to final format
│   │
│   └── ui/
│       ├── ui.go             # Print functions
│       ├── progress.go       # Progress bar
│       └── spinner.go        # Spinner animation
│
├── go.mod
├── Makefile                  # build, install, clean targets
└── README.md
```

---

## Implementation Order

```
M1 (1-2 days)
  ├─ cmd/main.go (cobra setup)
  └─ internal/input/input.go

M2 (2-3 days)
  ├─ internal/ui/ui.go, spinner.go, progress.go
  └─ internal/search/search.go

M3 (2-3 days)
  ├─ internal/innertube/types.go
  └─ internal/innertube/client.go

M4 (2 days)
  └─ internal/download/download.go

M5 (1-2 days)
  └─ internal/convert/ffmpeg.go

M6 (3-5 days) ← hardest, YouTube changes frequently
  ├─ internal/crypto/js.go
  ├─ internal/crypto/transforms.go
  └─ internal/crypto/sig.go

M7 (1-2 days)
  └─ Error handling, retry, cleanup, polish
```

**Total estimated: 12-20 days** (depending on YouTube internals experience)

---

## Key References

| Resource | URL | Purpose |
|---|---|---|
| kkdai/youtube | github.com/kkdai/youtube | Go implementation, sig decryption |
| yt-dlp | github.com/yt-dlp/yt-dlp | Most complete reference |
| YouTube.js | github.com/LuanRT/YouTube.js | Innertube API docs |
| pytube/cipher.py | github.com/pytube/pytube | Transform function parsing |

---

## Important Warnings

1. **YouTube changes frequently**: Player JS and signature algorithm can change at any time. M6 may need frequent updates.

2. **Geo-restriction**: Some videos are not available in all regions. Handle gracefully with error message.

3. **Rate limiting**: Calling Innertube API too many times may result in temporary ban. Implement backoff.

4. **Copyright**: Tool should only be used for content you have the right to download. Spec does not encourage copyright violations.

5. **ffmpeg requirement**: User must install ffmpeg first. Check and guide installation if not present.
