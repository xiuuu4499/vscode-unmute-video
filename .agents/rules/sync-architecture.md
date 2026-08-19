# Audio/Video Sync Architecture

## The Core Problem

VS Code's Chromium webview cannot decode **AAC** audio (the codec used by
MP4/MOV/M4V). The `<video>` element plays video fine but is permanently muted.
To work around this, the extension:

1. Runs **ffmpeg on the extension host** to extract the audio track to MP3.
2. Registers the MP3 with `StreamServer` and sends an `audioSrc` URL to the
   webview.
3. The webview creates a hidden `<audio>` element and plays the MP3 in parallel
   with the muted `<video>`.
4. A drift-correction algorithm keeps the two elements in sync.

**WebM files** are exempt — Chromium can decode Vorbis/Opus natively, so WebM
plays unmuted from the `<video>` element with no ffmpeg needed.

---

## The Drift-Correction Algorithm (`src/webview/sync.ts`)

`driftAction(audioTime, videoTime, baseRate)` returns one of three actions:

| Condition | Action |
|---|---|
| `|drift| <= 0.1s` (soft band) | `{ kind: 'none' }` — do nothing |
| `0.1s < |drift| < 1.0s` | `{ kind: 'rate', playbackRate }` — nudge audio ±0.05 from base rate |
| `|drift| >= 1.0s` (hard band) | `{ kind: 'seek', to: videoTime }` — hard-seek audio to video clock |

**Why rate-nudge instead of always hard-seeking?**
Hard-seeking (`audio.currentTime = x`) flushes decode buffers and re-fetches
from the HTTP server — every frame would cause a dropout. Rate-nudging lets
audio gently drift back into sync without a seek. Only when drift is so large
that rate-nudging would take too long do we accept the seek cost.

This policy is **pure** (no DOM, no vscode), lives in `sync.ts`, and is fully
unit-tested in `test/sync.test.js`.

---

## Buffering Coordination (`src/webview/playerController.ts`)

When the **video** buffers:
- `video.waiting` / `video.stalled` → pause the audio element (so it doesn't
  run ahead into silence), show the buffering spinner.
- `video.playing` / `video.canplay` → resume audio, hide spinner.

When the **audio** buffers:
- `audio.waiting` / `audio.stalled` → hold the video paused (`waitingForAudio = true`),
  show the buffering spinner. The UI should read as "buffering", not "stopped".
- `audio.canplay` / `audio.playing` / `audio.seeked` → release the hold,
  resume the pair.

The `waitingForAudio` flag distinguishes a masked buffering-hold pause from a
genuine user-initiated pause. The `video.pause` event handler bails out early
when `waitingForAudio` is true.

---

## Audio Extraction (`src/media/audio.ts`)

The ffmpeg command:
```
ffmpeg -nostdin -i <input>
       -vn                        # drop video stream
       -af aresample=async=1:first_pts=0  # normalize audio start time to PTS 0
       -c:a libmp3lame
       -b:a 192k
       -f mp3
       -y <output.mp3>
```

The `-af aresample=async=1:first_pts=0` filter is **critical**: MP4/MOV files
commonly carry an edit-list priming delay (encoder delay) that offsets the
first audio sample from time 0. Without this filter, the extracted MP3 starts a
constant amount late (or early), which the drift-correction loop perceives as
a steady offset and fights against indefinitely.

**In-flight deduplication:** `inFlightExtractions` (a `Map<key, Promise<string>>`)
ensures that two editors opening the same file simultaneously share a single
ffmpeg process.

**Caching:** The output path is derived from `MD5(path + '\0' + size + '\0' + mtime)`,
so editing a video in place forces re-extraction. The cache lives in
`os.tmpdir()/unmute-video-cache/` and is pruned at 7-day TTL on activation.

---

## Protocol Flow (Happy Path)

```
Host                            Webview
 |                                 |
 |<---- { type: 'ready' } ---------|   (webview loaded)
 |                                 |
 |-- { type: 'init', ... } ------->|   (name, preferences, seekStep, audioPending=true)
 |-- { type: 'videoSrc', url } --->|   (video starts streaming immediately)
 |-- { type: 'subtitles', vtt } -->|   (if sidecar subtitle found)
 |                                 |
 |  [ffmpeg extraction runs...]    |
 |                                 |
 |-- { type: 'audioSrc', url } --->|   (MP3 URL; PlayerController.attachAudio())
 |                                 |
 |<-- { type: 'progress', time } --|   (every 5s while playing; saved to workspaceState)
 |<-- { type: 'savePreferences' }--|   (volume/mute/speed changes)
```

Error paths replace `audioSrc` with `audioNone` (no audio track), `audioError`
(extraction failed), or `audioUntrusted` (workspace not trusted).

---

## Key Invariants for Changes

- **Never assign `audio.currentTime` inside the `timeupdate` loop without
  checking `audio.seeking`** — that was the pre-sync.ts approach and caused
  re-seek storms.
- **Always clear `waitingForAudio` before a user-initiated pause** to prevent
  the `video.pause` event handler from treating the pause as a buffering event.
- **`canResumeAudio()` gates `audio.play()` calls** — don't call `play()` when
  `readyState < HAVE_FUTURE_DATA` or `seeking === true`; it will stall and then
  re-drift on recovery.
- **The drift-correction action is computed once per `video.timeupdate`**, not
  on every audio event.
