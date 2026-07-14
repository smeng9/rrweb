# WebSocket recording in `rrweb-plugin-network-record`

**Date:** 2026-07-13
**Status:** Approved design

## Problem

The network recording plugin (`@rrweb/rrweb-plugin-network-record`) records HTTP-style
traffic only: passive `navigation`/`resource` performance timing, plus opt-in
fetch and XHR headers/bodies. WebSocket connections are invisible — neither the
connect/disconnect lifecycle nor the messages flowing over a socket are captured.
This design adds WebSocket recording to the plugin.

## Goals

- Record WebSocket connection lifecycle: `open`, `close` (with code/reason/wasClean),
  and `error`.
- Record individual messages (both directions) with metadata, and optionally
  their payloads.
- Keep volume under control for high-frequency sockets (throttled batching +
  payload size cap).
- Zero behavior change for existing users who do not opt in.
- No changes required to the replay plugin.

## Non-goals

- Reconstructing / replaying live WebSocket traffic against a real server.
- Recording binary payload contents (binary messages are recorded as metadata
  only by default).
- Any new transport or storage mechanism — data flows through the existing
  `NetworkData` plugin-event pipeline.

## Architecture

A new `initWebSocketObserver(cb, win, options)` is added alongside
`initXhrObserver` and `initFetchObserver`, and wired into `initNetworkObserver`.
It follows the existing monkey-patch pattern:

- Patch the `win.WebSocket` **constructor** via `@rrweb/utils` `patch()`.
- For each constructed socket instance:
  - Assign a stable, incrementing `socketId`.
  - Attach listeners for `open`, `close`, `error`, `message`.
  - Wrap the instance's `send()` to capture outgoing messages. The original
    `send()` is always invoked, even if capture throws.
- Events for a socket are collected into a buffer and **flushed on a throttle**
  (default 500ms), each flush emitting one `NetworkData` callback.

The observer is only initialised when `'websocket'` is present in
`options.initiatorTypes`. It is a no-op when `win.WebSocket` is undefined
(SSR / worker contexts).

### Data flow

```
page WebSocket activity
  -> patched WebSocket ctor + wrapped send() + event listeners
  -> per-socket event buffer
  -> throttled flush -> networkCallback({ requests: [ { initiatorType:'websocket', socketId, name, events } ] })
  -> transformRequestFn -> plugin event (EventType.Plugin, PLUGIN_NAME)
  -> replay plugin forwards NetworkData verbatim -> consumer onNetworkData
```

A single long-lived socket produces **multiple** `NetworkData` callbacks over its
lifetime — one per throttle flush — each tagged with the same `socketId`.
Consumers accumulate events by `socketId`.

## Data model (`@rrweb/types`)

### `NetworkInitiatorType`

Add `'websocket'` to the union.

### `NetworkRequest`

Add two optional, backwards-compatible fields:

```ts
socketId?: number;           // stable per-connection id; correlates flushed batches
events?: WebSocketEvent[];   // lifecycle + message events for this connection
```

For a WebSocket connection: `initiatorType: 'websocket'`, `name` = socket URL,
`startTime` = connect time (`performance.now()` rounded), `endTime` = close time
(present once the socket closes). `method`/`status`/`requestBody`/etc. are unused
for websocket requests.

### New `WebSocketEvent` type

```ts
export type WebSocketEvent = {
  type: 'open' | 'close' | 'error' | 'message';
  timestamp: number;            // performance.now() rounded

  // message-only
  dir?: 'sent' | 'received';
  size?: number;                // payload byte length
  body?: NetworkBody;           // only when recordBody is enabled; may be truncated
  binary?: boolean;             // true for Blob/ArrayBuffer/ArrayBufferView payloads
  truncated?: boolean;          // true when body was truncated to maxSize

  // close-only
  code?: number;
  reason?: string;
  wasClean?: boolean;
};
```

## Options (`NetworkRecordOptions`)

### Enabling

Including `'websocket'` in `initiatorTypes` enables WebSocket recording. It is
**not** in `defaultNetworkOptions.initiatorTypes`, so existing users are
unaffected until they opt in. When enabled with no further options, only
lifecycle events (`open`/`close`/`error`) are recorded.

### New option

```ts
recordWebSocketMessages?:
  | boolean
  | { throttleMs?: number; maxSize?: number };
```

- Default `false` — no per-message recording (lifecycle only).
- Truthy — per-message **metadata** (`dir`, `size`, `timestamp`, `binary`) is
  captured.
- `throttleMs` — flush/batch interval for buffered socket events. Default `500`.
- `maxSize` — payload truncation cap in bytes. Default `1024`.

### Payload capture

Message payload **contents** (`body`) require **both** `recordWebSocketMessages`
(truthy) **and** the existing `recordBody` flag. This keeps payloads opt-in and
consistent with fetch/XHR. String payloads are captured (truncated to `maxSize`,
setting `truncated: true`). Binary payloads (`Blob`, `ArrayBuffer`,
`ArrayBufferView`) are recorded as metadata only (`binary: true`, no `body`) —
matching the existing "cannot read binary body" behavior of `readXhrBody`.

`defaultNetworkOptions` gains `recordWebSocketMessages: false`.

## Error handling

- All patching and per-event capture is wrapped in try/catch; failures degrade to
  no-op and never surface to the page.
- The wrapped `send()` always calls the original implementation regardless of
  capture success.
- No-op when `win.WebSocket` is undefined.
- On teardown, the returned `listenerHandler` restores the original `WebSocket`
  constructor and flushes any pending buffered events.

## Testing (`test/index.test.ts`)

Add a `MockWebSocket` (following the `MockXMLHttpRequest` pattern) and cases:

1. Inert when `'websocket'` is absent from `initiatorTypes` (constructor not patched / no events).
2. `open` event recorded with correct `name` and `startTime`.
3. `close` event recorded with `code`, `reason`, `wasClean`, and `endTime` on the request.
4. `error` event recorded.
5. Sent vs received messages captured with correct `dir` and `size` when
   `recordWebSocketMessages` is on.
6. Message payloads captured only when `recordBody` is also enabled.
7. Binary payload → `binary: true`, no `body`.
8. Payload longer than `maxSize` → `truncated: true` and body length capped.
9. Events for one socket share a `socketId`; multiple flushes correlate.
10. Throttle batching: multiple messages within `throttleMs` arrive in one
    `NetworkData` flush.

Plus a type-level assertion that `'websocket'` is a valid `NetworkInitiatorType`.

## Scope / files touched

- `packages/types/src/index.ts` — `NetworkInitiatorType`, `NetworkRequest`,
  new `WebSocketEvent`, `NetworkRecordOptions`.
- `packages/plugins/rrweb-plugin-network-record/src/index.ts` — new
  `initWebSocketObserver`, wiring in `initNetworkObserver`, default option,
  re-export of `WebSocketEvent`.
- `packages/plugins/rrweb-plugin-network-record/test/index.test.ts` — tests.
- `packages/plugins/rrweb-plugin-network-replay/src/index.ts` — no code change
  expected; verify pass-through.
- Changeset entry.

## Backwards compatibility

All new type fields are optional; `NetworkData`/`NetworkRequest` consumers keep
compiling. Default options leave WebSocket recording off. The replay plugin
forwards the extended `NetworkData` unchanged.
