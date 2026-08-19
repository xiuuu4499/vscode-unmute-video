---
name: add-webview-message
description: |
  Step-by-step guide for adding a new postMessage type to the host↔webview
  protocol in vscode-unmute-video. Use this skill whenever you need to send a
  new kind of data or command between the extension host and the webview.
---

# Adding a New Webview Message

The `postMessage` boundary is governed by discriminated unions in
`src/shared/protocol.ts`, which is the **single source of truth**. Both the
extension host and the webview bundle import from this module.

---

## Step 1 — Update `src/shared/protocol.ts`

Add a new variant to the correct union:

**Host → Webview** (extension host sends to webview):
```typescript
export type HostToWebview =
    | ...existing variants...
    | { type: 'myNewMessage'; someField: string; count: number };
```

**Webview → Host** (webview sends to extension host):
```typescript
export type WebviewToHost =
    | ...existing variants...
    | { type: 'myNewAction'; payload: string };
```

Use `import type` only — this module emits no runtime JS.

---

## Step 2 — Handle the message on the receiving end

### If **host → webview**: handle in `src/webview/main.ts`

```typescript
// In the window.addEventListener('message', ...) switch:
case 'myNewMessage':
    controller.handleMyNewMessage(msg.someField, msg.count);
    break;
```

### If **webview → host**: handle in `src/editor/playerEditorProvider.ts`

```typescript
// In the messageListener switch:
case 'myNewAction': {
    const text = typeof message.payload === 'string' ? message.payload : '';
    // ... handle it
    break;
}
```

For actions that call into VS Code APIs (opening dialogs, writing clipboard,
etc.), add the action name to the `WebviewAction` union in `protocol.ts` and
handle it in the `case 'action':` switch via an explicit allowlist. Log unknown
names with `console.warn` instead of silently ignoring.

---

## Step 3 — Send the message from the originating end

### From the host (in `playerEditorProvider.ts`):
```typescript
// The `post` helper is already typed to HostToWebview:
post({ type: 'myNewMessage', someField: 'hello', count: 42 });
```

### From the webview (in `main.ts` or `playerController.ts`):
```typescript
// `postMessage` is typed to WebviewToHost:
this.postMessage({ type: 'myNewAction', payload: 'data' });
```

---

## Step 4 — If data needs to survive panel hides/shows

State restored after `retainContextWhenHidden` cycles is re-sent from the host
on `'ready'`. If your new message carries state the webview needs on startup,
ensure the host re-sends it in the `'ready'` handler in `resolveCustomEditor()`.

---

## Step 5 — Tests

- If adding a **host → webview** message that triggers observable behavior in
  `PlayerController`, add a unit test in the relevant `test/*.test.js`.
- If adding a **webview → host** message that triggers host behavior, write an
  integration test or update `test/audioExtractionController.test.js` as
  appropriate.

---

## Checklist

- [ ] New union member added to `protocol.ts` (`HostToWebview` or `WebviewToHost`)
- [ ] Message handled in the receiver (`main.ts` or `playerEditorProvider.ts`)
- [ ] Message sent from the sender
- [ ] Re-sent on `'ready'` if it carries startup state
- [ ] Tests updated
