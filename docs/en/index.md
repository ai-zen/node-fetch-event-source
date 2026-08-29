---
title: node-fetch-event-source
description: A Fetch API based SSE event stream consumer, providing more complete and controllable request and reconnection for server-sent events.
outline: deep
---

# node-fetch-event-source

`@ai-zen/node-fetch-event-source` (source repo `node-fetch-event-source`) is a [Server-Sent Events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events) (SSE) event stream consumer built on the [Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API). It is forked from [Azure/fetch-event-source](https://github.com/Azure/fetch-event-source) and is fully compatible with the [Event Stream format](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events#Event_stream_format).

Compared with the browser's built-in [`EventSource`](https://developer.mozilla.org/en-US/docs/Web/API/EventSource), `fetchEventSource` exposes more low-level capabilities:

- You can use any request method / headers / body, plus all the other capabilities of `fetch()` (`credentials`, `mode`, `cache`, etc.).
- You can provide a custom `fetch` implementation (e.g., `cross-fetch` in Node).
- You can access the `Response` object before parsing the event stream, for secondary validation of gateway or upstream errors.
- When the connection is interrupted or an error occurs, you have full control over the retry strategy, instead of the browser silently retrying a few times and then giving up.

## Environment Requirements

- Runtime: Node.js (TypeScript compiles to **ES2017**) or all mainstream evergreen browsers (Chrome, Firefox, Safari, Edge).
- Dependency: `cross-fetch` (`^4.0.0`), used as a fallback when a global `fetch` is unavailable.
- Compatibility note: legacy Edge (< 79) requires a `TextDecoder` polyfill, for example:

```js
require('fast-text-encoding');
```

## Installation

```bash
npm install @ai-zen/node-fetch-event-source
```

You can also use pnpm or yarn:

```bash
pnpm add @ai-zen/node-fetch-event-source
```

## Quick Example

```ts
import { fetchEventSource } from '@ai-zen/node-fetch-event-source';

await fetchEventSource('/api/sse', {
  onmessage(ev) {
    console.log(ev.data);
  }
});
```

For a more complete usage (POST, custom headers, AbortController, and error handling), see [Quick Start](./getting-started.md).

## Core Concepts

- **`fetchEventSource(input, init)`**: the entry function of this library, returns `Promise<void>`, resolves when the connection closes or is aborted.
- **`EventStreamContentType`**: the `'text/event-stream'` constant; the default `onopen` validates the response `Content-Type` using `startsWith`.
- **`EventSourceMessage`**: the shape of a single event message, containing `id`, `event`, `data`, and an optional `retry`.
- **Callback pipeline**: `onopen` → read bytes → `getLines` splits → `getMessages` assembles → `onmessage`; `onclose` / `onerror` handle the finalization and reconnection.
- **Reconnection**: the default retry interval is 1000ms; it can be updated via the server's `retry:` field; the retry interval is controlled by the return value of `onerror`, and rethrowing an exception stops the process. See [Reconnection and Error Handling](./guides/reconnection.md).

## Reference

- [Quick Start](./getting-started.md) — Installation, minimal example, POST/Abort example.
- [API Reference](./api/index.md) — `fetchEventSource` options, `EventSourceMessage` type and constants.
- [Reconnection and Error Handling](./guides/reconnection.md) — Retry strategy, `last-event-id`, and page visibility behavior.
