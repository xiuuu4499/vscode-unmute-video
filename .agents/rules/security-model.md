# Security & Trust Model

## Overview

This extension has explicit, well-tested security hardening. When making any
change that touches file paths, external processes, network access, or the
webview sandbox, consult this rule first.

---

## ffmpeg Path Hardening (`src/media/audio.ts`)

The `unmuteVideo.ffmpegPath` setting is **machine-scoped** (`"scope": "machine"`
in `package.json`). This prevents workspace-level overrides.

Additionally, `resolveFfmpegOverride()` enforces a belt-and-suspenders guard:
- Rejects empty, whitespace-only, or relative paths.
- Rejects any absolute path that resolves **inside** any workspace root.
- Uses `fs.realpathSync.native()` to canonicalize symlinks before the boundary
  check, preventing symlink-based bypasses.
- Uses case-insensitive comparison on macOS and Windows.

**Never skip `resolveFfmpegOverride()`** when reading the ffmpeg path from
configuration. The test suite in `test/securityHardening.test.js` covers these
invariants explicitly.

---

## StreamServer Security (`src/server/streamServer.ts`)

- Binds to `127.0.0.1` only — never `0.0.0.0`.
- **Token-only serving:** callers register a path and receive an opaque 32-char
  hex token; the server NEVER resolves an arbitrary fs path from a URL.
- **DNS-rebinding guard:** any request whose `Host` header does not exactly
  match `127.0.0.1:<port>` receives a `403`. This is validated in
  `test/streamServer.test.js`.
- Only `GET` and `HEAD` methods are accepted; all others receive `405`.

---

## Webview CSP

The Content Security Policy assembled in `PlayerEditorProvider.buildHtml()` is:
```
default-src 'none';
img-src <cspSource> data:;
style-src <cspSource>;
font-src <cspSource>;
script-src 'nonce-<perPanelNonce>';
media-src http://127.0.0.1:<port> blob:;
```

- **`default-src 'none'`** — nothing is allowed unless explicitly listed.
- **Scripts require a per-panel nonce** — inline scripts cannot be injected.
- **Media allowed only from loopback** — the webview cannot load arbitrary URLs.
- The nonce is regenerated per panel using `crypto.randomBytes(16)`.

When modifying `media/player.html` or `buildHtml()`, never relax these rules
without explicit justification and a corresponding CHANGELOG security entry.

---

## Workspace Trust

- In **untrusted workspaces**, ffmpeg extraction is disabled (audio stays muted).
  The webview shows a "Trust workspace" prompt; the host sends `{ type: 'audioUntrusted' }`.
- Extraction is started only after `vscode.workspace.onDidGrantWorkspaceTrust` fires.
- The extension declares `"untrustedWorkspaces": { "supported": "limited" }` in
  `package.json` — this must be kept in sync with actual behavior.

---

## Audio Cache Safety

- Cache dir: `os.tmpdir()/unmute-video-cache/` with mode `0o700`.
- Extracted files have mode `0o600`.
- Filenames are derived from a content-addressed MD5 hash of `path\0size\0mtime` —
  not from the video filename directly (no path injection in filenames).
- Atomic rename: ffmpeg writes to a `.part` temp file and renames on success.
  A killed process never leaves a reusable partial file.
- 7-day TTL pruning at activation (`pruneAudioCache()`).

---

## What to do when adding security-sensitive changes

1. Add or update a test in `test/securityHardening.test.js`.
2. Add a `### Security` entry to `CHANGELOG.md → [Unreleased]`.
3. If the change affects the `package.json` manifest (scope, capabilities,
   trust), update the manifest test assertions in `securityHardening.test.js`.
