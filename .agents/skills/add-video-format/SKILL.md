---
name: add-video-format
description: |
  Step-by-step guide for adding a new video/audio container format to the
  vscode-unmute-video extension. Use this skill when asked to support a new
  file type (e.g. MKV, AVI, TS, FLV).
---

# Adding a New Video Format

Adding a new container format touches several places. Follow these steps in
order to keep all surfaces in sync.

---

## Step 1 — Determine audio support

Decide whether VS Code's Chromium webview can decode the audio codec natively:
- **Vorbis / Opus** (e.g. `.webm`) → native, no ffmpeg needed.
- **AAC / MP3 / etc.** in any other container → requires ffmpeg extraction.

---

## Step 2 — `src/media/mediaFormat.ts`

This is the **single source of truth** for which extensions the extension opens.

```typescript
// Add the new extension (lowercase) to the tuple:
export const VIDEO_EXTENSIONS = ['.mp4', '.mov', '.m4v', '.webm', '.mkv'] as const;

// If the new format has natively-decodable audio, add it here:
export function isNativeAudioFormat(fsPath: string): boolean {
    const ext = path.extname(fsPath).toLowerCase();
    return ext === '.webm'; // add more here if applicable
}
```

---

## Step 3 — `package.json` — `customEditors.selector`

Add a case-insensitive pair of `filenamePattern` entries:

```json
{
    "filenamePattern": "**/*.{mkv,MKV}"
}
```

Also add the lowercase extension to `keywords` for discoverability.

---

## Step 4 — `src/server/streamServer.ts` — `contentTypeFor()`

Map the new extension to its MIME type:

```typescript
case '.mkv':
    return 'video/x-matroska';
```

If the webview's `<video>` element cannot play the container directly, you may
need to re-mux to a supported container via ffmpeg instead of just extracting
audio.

---

## Step 5 — Update `src/media/subtitles.ts` if needed

`sidecarCandidates()` derives sidecar subtitle paths from the video path. If
the new format has format-specific subtitle conventions, update it there.

---

## Step 6 — Tests

1. Add a Content-Type test in `test/streamServer.test.js`:
   ```js
   test('Content-Type by extension: .mkv -> video/x-matroska', async (t) => { ... });
   ```
2. Add a `mediaFormat.test.js` case if `isNativeAudioFormat` changes.
3. If adding a real ffmpeg-extracted format, add a case to `test/audio.test.js`
   (uses real ffmpeg, self-skips if absent).

---

## Step 7 — README & CHANGELOG

- Update the README features table / supported formats list.
- Add an entry under `## [Unreleased]` in `CHANGELOG.md` using the
  `### Added` subsection.

---

## Checklist

- [ ] `VIDEO_EXTENSIONS` in `mediaFormat.ts`
- [ ] `isNativeAudioFormat()` updated if native audio
- [ ] `package.json` → `customEditors.selector` (both cases)
- [ ] `package.json` → `keywords` (lowercase extension)
- [ ] `streamServer.ts` → `contentTypeFor()` MIME mapping
- [ ] `test/streamServer.test.js` → Content-Type test
- [ ] `test/mediaFormat.test.js` updated
- [ ] `README.md` supported formats list updated
- [ ] `CHANGELOG.md` `[Unreleased]` entry added
