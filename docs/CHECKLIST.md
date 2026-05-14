# Implementation Checklist

## M1 — CLI Entry Point & Input Module

- [ ] Add `github.com/spf13/cobra` to go.mod
- [ ] Create `cmd/main.go` with cobra root command
- [ ] Implement flags: `-o, --output` (default `~/Music`), `-k, --top` (default `5`)
- [ ] Create `internal/input/input.go`
- [ ] Implement `InputType` enum (URL, Keyword)
- [ ] Implement `Read()` function to read stdin
- [ ] Implement `ParseVideoID()` for YouTube URL formats:
  - [ ] `youtube.com/watch?v=ID`
  - [ ] `youtu.be/ID`
  - [ ] `youtube.com/watch?v=ID&t=120`
  - [ ] `m.youtube.com/watch?v=ID`
- [ ] Implement URL vs keyword classification logic
- [ ] Add unit tests for `ParseVideoID()`
- [ ] Test: `l2a -k 3` with keyword input
- [ ] Test: `l2a` with YouTube URL input

## M2 — Search Module & UI

### UI Components
- [ ] Create `internal/ui/ui.go`
- [ ] Implement `PrintResults(videos []Video)` function
- [ ] Implement `PrintFormats()` function
- [ ] Implement `PromptSelect(prompt string, max int) (int, error)`
- [ ] Create `internal/ui/spinner.go`
- [ ] Implement `Spinner(label string) func()` with stop mechanism
- [ ] Create `internal/ui/progress.go`
- [ ] Implement `ProgressBar(p Progress)` with ANSI escape codes

### Search Module
- [ ] Create `internal/search/search.go`
- [ ] Define `Video` struct (ID, Title, Channel, Duration, IsLive)
- [ ] Implement `Search(keyword string, limit int) ([]Video, error)`
- [ ] Implement Innertube search API call (POST to `/youtubei/v1/search`)
- [ ] Parse JSON response from `videoRenderer`
- [ ] Handle LIVE badge detection
- [ ] Add retry logic with exponential backoff (500ms, 1s)
- [ ] Handle empty results case
- [ ] Add unit tests for search parsing
- [ ] Test: search for "lo-fi study music"

## M3 — Innertube Player Client

- [ ] Create `internal/innertube/types.go`
- [ ] Define `AudioStream` struct
- [ ] Define `PlayerResponse` struct
- [ ] Create `internal/innertube/client.go`
- [ ] Implement `FetchPlayer(videoID string) (PlayerResponse, error)`
- [ ] Use ANDROID client for fewer signature requirements
- [ ] Parse `streamingData.adaptiveFormats[]` for audio streams
- [ ] Filter audio streams (mimeType starts with "audio/")
- [ ] Implement `SelectBestStream(streams []AudioStream) AudioStream`
- [ ] Sort by bitrate, prefer opus > mp4a > vorbis
- [ ] Handle cipher streams (pass to crypto module later)
- [ ] Add error handling for geo-restricted videos
- [ ] Add unit tests with mock responses
- [ ] Test: fetch stream for known video ID

## M4 — Download Module

- [ ] Create `internal/download/download.go`
- [ ] Define `Progress` struct (Downloaded, Total, Percent)
- [ ] Implement `Download(url string, destPath string, onProgress func(Progress)) error`
- [ ] Set required headers (User-Agent, Referer, Origin)
- [ ] Implement resume support:
  - [ ] Check existing file size
  - [ ] HEAD request for Content-Length
  - [ ] Range header for partial download
- [ ] Implement 32KB chunk buffer reading
- [ ] Write to file during download
- [ ] Call progress callback on each chunk
- [ ] Handle HTTP 403 (need signature decrypt)
- [ ] Handle HTTP 404 (video unavailable)
- [ ] Implement retry with resume on connection reset
- [ ] Output to `~/.cache/l2a/tmp/<videoID>.<ext>`
- [ ] Add unit tests with mock HTTP server
- [ ] Test: download a small audio file

## M5 — Convert Module (ffmpeg)

- [ ] Create `internal/convert/ffmpeg.go`
- [ ] Define `Format` type (mp3, m4a, flac, wav, opus)
- [ ] Implement `CheckFFmpeg() error` function
- [ ] Verify ffmpeg installation on startup
- [ ] Implement `SanitizeFilename(title string) string`
  - [ ] Remove `/ \ : * ? " < > |`
  - [ ] Replace multiple spaces with single space
  - [ ] Trim whitespace
  - [ ] Limit to 200 characters
- [ ] Implement `Convert(inputPath string, outputPath string, format Format) error`
  - [ ] MP3: `-vn -acodec libmp3lame -q:a 2`
  - [ ] M4A: `-vn -acodec aac -b:a 192k`
  - [ ] FLAC: `-vn -acodec flac`
  - [ ] WAV: `-vn -acodec pcm_s16le`
  - [ ] OPUS: `-vn -acodec libopus -b:a 128k`
- [ ] Add conversion progress indicator
- [ ] Handle ffmpeg errors with user-friendly messages
- [ ] Add unit tests with actual ffmpeg (if available)
- [ ] Test: convert webm to mp3

## M6 — Crypto / Signature Decryption

### Player JS Handling
- [ ] Create `internal/crypto/js.go`
- [ ] Implement `FetchPlayerJS(playerJSURL string) (string, error)`
- [ ] Implement player JS caching to `~/.cache/l2a/player_<hash>.js`
- [ ] Implement cache TTL (24 hours)
- [ ] Implement `ExtractDecryptFunc(jsContent string) (string, error)`
  - [ ] Find `set("signature", a.reverse)` or `sig||...` pattern
  - [ ] Extract helper object name
  - [ ] Find `var DE={(...)};` block
  - [ ] Parse transform functions

### Transform Functions
- [ ] Create `internal/crypto/transforms.go`
- [ ] Implement `reverse(s string) string`
- [ ] Implement `splice(s string, n int) string`
- [ ] Implement `swap(s string, n int) string`
- [ ] Implement `ApplyTransforms(sig string, ops []Op) string`

### Signature Decryption
- [ ] Create `internal/crypto/sig.go`
- [ ] Implement `DecryptSignature(sigCipher string, playerJSURL string) (string, error)`
- [ ] Parse `signatureCipher` URL (s=, sp=, url=)
- [ ] Apply transform operations to signature
- [ ] Append `&sig=` to base URL
- [ ] Implement `n` param throttling (if needed)

### Integration
- [ ] Integrate crypto module with innertube client
- [ ] Test signature decryption with known video
- [ ] Handle edge cases (missing functions, etc.)

## M7 — Polish & Error Handling

### Error Messages
- [ ] Create `internal/errors/errors.go`
- [ ] Define all error types with Vietnamese messages:
  - [ ] ffmpeg_not_found
  - [ ] network_error
  - [ ] geo_restricted
  - [ ] video_unavailable
  - [ ] disk_full
  - [ ] sig_decrypt_failed
  - [ ] signature_not_found
- [ ] Implement error wrapper with context

### Retry Logic
- [ ] Implement `withRetry(ctx context.Context, fn func() error, maxAttempts int) error`
- [ ] Use exponential backoff (500ms, 1s, 2s)
- [ ] Max 3 attempts
- [ ] Apply to network requests

### Cleanup
- [ ] Delete temp files after successful conversion
- [ ] Handle cleanup on SIGINT/SIGTERM
- [ ] Log cleanup errors (non-fatal)

### UX Polish
- [ ] Clear terminal output between steps
- [ ] Colored output for errors (red)
- [ ] Colored output for success (green)
- [ ] Spinner during network operations
- [ ] Progress bar during download

### Testing
- [ ] Integration test: full flow with search
- [ ] Integration test: full flow with direct URL
- [ ] Test error scenarios

## Build & Release

- [ ] Create `Makefile` with targets:
  - [ ] `make build` — build binary
  - [ ] `make install` — install to $PATH
  - [ ] `make clean` — remove build artifacts
  - [ ] `make test` — run tests
  - [ ] `make lint` — run linter
- [ ] Create `.gitignore`
- [ ] Verify `go build` succeeds
- [ ] Test binary on clean system (if possible)

## Documentation

- [ ] Update README.md with final usage
- [ ] Add troubleshooting section
- [ ] Document all flags
- [ ] Add architecture diagram
- [ ] Update plan.md with any changes

---

## Progress Tracking

| Milestone | Status | Notes |
|-----------|--------|-------|
| M1 | Not started | |
| M2 | Not started | |
| M3 | Not started | |
| M4 | Not started | |
| M5 | Not started | |
| M6 | Not started | Hardest, YouTube changes |
| M7 | Not started | |
| Build & Release | Not started | |

---

## Dependencies

### Go Modules
- `github.com/spf13/cobra v1.8.1`

### External Runtime
- `ffmpeg` (latest)

### No External Dependencies For
- HTTP client (stdlib)
- JSON parsing (stdlib)
- Regex (stdlib)
- File I/O (stdlib)