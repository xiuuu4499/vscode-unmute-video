---
name: write-a-test
description: |
  Guide for writing tests in the vscode-unmute-video project. Use this skill
  when asked to add, fix, or extend tests. Covers the Node built-in test runner
  conventions, test-support helpers, and how to handle ffmpeg-dependent tests.
---

# Writing Tests

All tests use the **Node.js built-in `node:test` runner** — no jest, mocha, or
vitest. Tests live in `test/*.test.js` and run against compiled JS in `out/`.

---

## Running tests

```bash
npm test              # compiles (pretest) then runs node --test
npm run compile       # compile only (host + webview)
npm run compile:host  # host only (usually sufficient before running tests)
```

---

## Test file anatomy

```js
'use strict';

const { test, before, after } = require('node:test');
const assert = require('node:assert/strict');

// Import from compiled output, not src/:
const { MyClass } = require('../out/path/to/module.js');
// Shared test helpers:
const { makeTempDir, createCleanup } = require('../test-support/tmp.js');
const { discoverFfmpeg, makeAacMp4 } = require('../test-support/ffmpeg.js');
// Or all at once:
const { makeTempDir, createCleanup, discoverFfmpeg } = require('../test-support');
```

---

## Available test-support helpers

### `test-support/tmp.js`

| Helper | Description |
|---|---|
| `makeTempDir(prefix)` | Creates a temp dir under `os.tmpdir()` and returns its path |
| `createCleanup()` | Returns `{ track(path), run() }` — collects paths to delete at the end |

### `test-support/ffmpeg.js`

| Helper | Description |
|---|---|
| `discoverFfmpeg()` | Returns a path to a working ffmpeg binary, or `null` |
| `makeAacMp4(ffmpeg, dir)` | Creates a minimal AAC-encoded MP4 in `dir`, returns its path |
| `looksLikeMp3(path)` | Checks for ID3 or MPEG frame sync bytes |

### `test-support/fakeFfmpeg.js`

Provides a scriptable fake `ffmpeg` binary (a Node.js script) that can be
configured to succeed, fail, or produce specific output — useful for unit tests
that must not depend on a real ffmpeg installation.

---

## Patterns by test type

### Pure-logic unit tests (no ffmpeg, no vscode)

```js
test('driftAction: within soft band -> none', () => {
    const result = driftAction(1.0, 1.0, 1.0);
    assert.deepEqual(result, { kind: 'none' });
});
```

### StreamServer integration tests

Use the `freshServer(t)` pattern:

```js
async function freshServer(t) {
    const server = new StreamServer();
    await server.start();
    t.after(() => server.dispose());
    return server;
}

test('my server test', async (t) => {
    const server = await freshServer(t);
    const port = server.getPort();
    // ... make HTTP requests
});
```

For HTTP requests against the server:
```js
function request(port, opts = {}) { /* see test/streamServer.test.js */ }
function requestToken(port, token, opts = {}) { return request(port, { ...opts, pathname: `/${token}` }); }
```

### Tests that require a real ffmpeg

Always self-skip when ffmpeg is absent:

```js
let ffmpeg = null;
const cleanup = createCleanup();

before(async () => {
    ffmpeg = await discoverFfmpeg();
    // ... set up files if ffmpeg found
});

after(() => cleanup.run());

test('my ffmpeg test', async (t) => {
    if (!ffmpeg) { t.skip('ffmpeg not available'); return; }
    // ... test using ffmpeg
});
```

### package.json manifest tests

```js
test('package.json: my invariant', () => {
    const pkg = JSON.parse(fs.readFileSync(path.join(__dirname, '..', 'package.json'), 'utf8'));
    assert.equal(pkg.some.property, 'expectedValue');
});
```

---

## Assert style

Always `require('node:assert/strict')` — prefer strict equality everywhere:

```js
assert.equal(actual, expected);          // ===
assert.deepEqual(actual, expected);      // deep equality
assert.ok(condition, 'message');
assert.doesNotThrow(() => fn());
await assert.rejects(() => promise, (err) => { ... return true; });
```

---

## Cleanup discipline

- Use `createCleanup()` and call `cleanup.run()` in `after()` for temp files.
- Use `t.after(() => server.dispose())` for per-test resource cleanup.
- For global teardown: `test.after(() => { fs.rmSync(TMP_ROOT, { recursive: true, force: true }); })`.

---

## Naming conventions

- File: `test/<featureName>.test.js`
- Test names: plain English, describe observable behaviour, include `REGRESSION:` prefix for bug-fix tests.
- Helper functions local to a test file: `camelCase`, short names (`freshServer`, `makeTempFile`, `request`).
