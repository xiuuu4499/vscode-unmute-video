# vscode-unmute-video — Project Conventions

## Project Identity

This is a **VS Code extension** (`publisher: yutabee`, id: `unmute-video`) that
plays MP4/MOV/M4V/WebM files inside the editor with audio. The core problem it
solves: VS Code's Chromium webview cannot decode AAC, so the extension plays the
video element **muted** and simultaneously plays a separately-extracted MP3 (via
ffmpeg) in sync as the audible track. WebM files carry Vorbis/Opus (natively
decodable) and skip the ffmpeg path entirely.

---

## Tech Stack

| Concern | Tool |
|---|---|
| Language | TypeScript (strict mode) |
| Host compile target | CommonJS / ES2020, `tsc -p ./` → `out/` |
| Webview compile target | ES modules / ES2020, type-checked via `tsconfig.webview.json`, bundled via **esbuild** → `media/player.js` |
| Linting | ESLint 10 + `typescript-eslint` (flat config `eslint.config.mjs`) |
| Testing | Node built-in `node:test` runner; test files in `test/*.test.js` (compiled JS, not TS) |
| Packaging | `vsce package --no-dependencies` |
| Publishing | Automated by GitHub Actions on `vX.Y.Z` tag push |

---

## Source Layout

```
src/
  extension.ts            # Extension entry point (activate/deactivate)
  editor/
    playerEditorProvider.ts        # CustomReadonlyEditorProvider; wires webview ↔ host
    audioExtractionController.ts   # Manages async ffmpeg extraction lifecycle per editor
  server/
    streamServer.ts       # Loopback HTTP server; token-based file streaming with Range support
  media/
    audio.ts              # ffmpeg discovery, extraction, cache management (NO vscode import)
    mediaFormat.ts        # VIDEO_EXTENSIONS constant; isNativeAudioFormat()
    subtitles.ts          # Sidecar subtitle discovery (.srt/.vtt); srtToVtt()
  shared/
    protocol.ts           # HostToWebview / WebviewToHost discriminated unions — SINGLE SOURCE OF TRUTH
    preferences.ts        # Preferences interface, clampPreferences(), ALLOWED_RATES
    config.ts             # resolveSeekStep(), FRAME_STEP_SECONDS
    resume.ts             # resumeKey(), shouldResume()
  webview/
    main.ts               # Webview entry point: message dispatch + event wiring
    playerController.ts   # All playback logic (play/pause/seek/loop/sync/subtitles)
    sync.ts               # Pure drift-correction policy (driftAction, canResumeAudio — no DOM)
    dom.ts                # Typed DOM element references via `els` object
    seekbar.ts            # Seekbar drag/scrub logic
    bufferingOverlay.ts   # Debounced buffering spinner (clock-injectable, testable)
    abLoop.ts             # A/B loop state machine
    status.ts             # Status bar show/clear/flash
    util.ts               # Pure helpers (clamp, formatTime, latestBufferedEnd)
media/
  player.html             # Webview HTML template ({{CSP}}, {{NONCE}}, {{STYLE}}, {{SCRIPT}} tokens)
  player.css              # Webview stylesheet
  player.js               # Generated — esbuild output of src/webview/main.ts (do NOT edit)
test/                     # Node built-in test files (run against compiled out/)
test-support/             # Shared helpers: tmp dir, ffmpeg discovery, fake ffmpeg, MP3 detection
```

---

## Key Architectural Invariants

1. **`src/media/audio.ts` must NEVER import `vscode`** — it is tested directly
   against a real ffmpeg binary outside the VS Code host process.

2. **`src/shared/protocol.ts` is the single source of truth** for the
   `postMessage` boundary. Both the host (`playerEditorProvider.ts`) and the
   webview (`main.ts`) must import types from here. Adding a new message always
   means updating this file first.

3. **`src/webview/sync.ts` is a pure module** — no DOM, no `vscode`, only
   math. It is unit-tested in isolation.

4. **`StreamServer` uses opaque hex tokens** — callers register an absolute fs
   path and receive a token. The server NEVER serves an arbitrary path; only
   registered tokens can be resolved. Ref-counting keeps a token alive across
   multiple editors opening the same file.

5. **One `StreamServer` instance is shared across all open editors.** It is
   started in `activate()` and its disposal is registered with
   `context.subscriptions`.

6. **Webview CSP is strict:** `default-src 'none'`; media allowed only from
   `http://127.0.0.1:<port>` and `blob:`; scripts require a per-panel nonce.

7. **DNS-rebinding guard:** `StreamServer` rejects any request whose `Host`
   header does not match `127.0.0.1:<port>` exactly.

8. **`ffmpegPath` is `machine`-scoped** — workspace-level override is rejected
   by both the VS Code manifest and the `resolveFfmpegOverride()` guard that
   rejects paths inside workspace roots (ACE mitigation).

9. **Audio cache lives in `os.tmpdir()/unmute-video-cache/`** — MP3 files are
   named by an MD5 of `<input path>\0<size>\0<mtime>`, so editing a file
   in-place forces re-extraction. Files older than 7 days are pruned at
   activation.

10. **Extracted MP3 files use atomic rename** — ffmpeg writes to a `.part` temp
    file then renames to the final name on success; a killed extraction never
    leaves a reusable partial file.

---

## TypeScript Style

- **Strict mode everywhere** (`"strict": true`). No `any` without a comment.
- **`import type`** for type-only imports (never emit a runtime import for a
  type boundary crossing between host and webview).
- **No barrel `index.ts` files.** Import specific modules directly.
- **Class members** are declared with visibility modifiers (`private`, `public`,
  `readonly`). Prefer `readonly` for injected dependencies.
- **`void` operator** on fire-and-forget `Promise`s: `void vscode.window.showErrorMessage(...)`.
- **Error handling:** catch blocks either rethrow, surface a user message, or
  use `/* ignore */` with a comment explaining why silence is correct.
- **`/* nothing to clean up */`** as the conventional comment for no-op
  `dispose()` bodies.

---

## Testing Conventions

- Tests live in `test/*.test.js` and run against compiled JS in `out/`.
- Test runner: `node --test` (Node.js built-in, no jest/mocha/vitest).
- Each test file imports from `'node:test'` and `'node:assert/strict'`.
- Tests that require ffmpeg **self-skip** when the binary is absent:
  `if (!ffmpeg) { t.skip('ffmpeg not available'); return; }`.
- **`test-support/`** provides shared helpers:
  - `tmp.js` — `makeTempDir()`, `createCleanup()`
  - `ffmpeg.js` — `discoverFfmpeg()`, `makeAacMp4()`
  - `fakeFfmpeg.js` — a scriptable fake ffmpeg binary for unit tests
  - `index.js` — re-exports all of the above
- The `freshServer(t)` helper pattern: create a `StreamServer`, start it,
  register `t.after(() => server.dispose())` for cleanup, and return the
  started instance.
- Security/regression tests assert `package.json` manifest properties directly
  (scope, capabilities, extensionKind).

---

## Build Workflow

```bash
# Full build (host + webview)
npm run compile

# Host only (tsc)
npm run compile:host

# Webview only (type-check then esbuild bundle)
npm run compile:webview

# Run all tests (compiles first via pretest)
npm test

# Watch host changes
npm run watch

# Watch webview changes (two terminals)
npm run watch:webview          # esbuild watch
npm run watch:webview:types    # tsc type-check watch

# Package for publishing
npm run package

# Clean generated artifacts
npm run clean
```

---

## Release Process

See [`docs/PUBLISHING.md`](../docs/PUBLISHING.md). Summary:
1. Bump `version` in `package.json`.
2. Update `CHANGELOG.md` — move `[Unreleased]` into `[X.Y.Z] - YYYY-MM-DD`.
3. PR → CI green → merge to `main`.
4. `git tag vX.Y.Z && git push origin vX.Y.Z` triggers the GitHub Actions
   release workflow, which publishes to VS Code Marketplace and Open VSX.

---

## CHANGELOG Format

Follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) strictly:
- `## [Unreleased]` section is always present at the top.
- Use subsections: `### Added`, `### Changed`, `### Fixed`, `### Security`,
  `### Packaging`, `### Removed`.
- Link references at the bottom of the file.

---

## What Ships in the .vsix

Compiled `out/` + `media/` (including generated `media/player.js`) + `images/`
icon + `README.md` + `CHANGELOG.md` + `LICENSE.md`.

Source, tests, CI config, `node_modules/`, and local dot-files are excluded via
`.vscodeignore`. CI verifies `media/player.js` is present in the `.vsix`.
