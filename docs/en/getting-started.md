---
title: Quick Start
description: Minimal examples of installing and running @ai-zen/node-fetch-event-source, plus usage of POST, custom headers, and AbortController.
outline: deep
---

# Quick Start

## Installation

```bash
npm install @ai-zen/node-fetch-event-source
```

## Minimal Example

`fetchEventSource` continuously consumes the server event stream in the background and hands each event to `onmessage`:

```ts
import { fetchEventSource } from '@ai-zen/node-fetch-event-source';

await fetchEventSource('/api/sse', {
  onmessage(ev) {
    console.log(ev.data);
  }
});
```

`onmessage` is triggered for **all** events, including those with a custom `event` field (unlike the browser's built-in `EventSource.onmessage`, which only fires for the default `message` event).

## Sending a Request Body and Custom Headers

Since it uses `fetch` under the hood, you can pass every parameter supported by `RequestInit`:

```ts
import { fetchEventSource } from '@ai-zen/node-fetch-event-source';

await fetchEventSource('/api/sse', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({ foo: 'bar' })
});
```

> Note: `headers` only supports the `Record<string, string>` format (not compatible with a `Headers` instance).

## Cancelling with AbortController

Pass a `signal` to abort the connection when needed. When the `signal` fires an `abort`, resources are released and the Promise returned by `fetchEventSource` is resolved (it will not enter the `onerror` retry flow).

```ts
import { fetchEventSource } from '@ai-zen/node-fetch-event-source';

const ctrl = new AbortController();

await fetchEventSource('/api/sse', {
  signal: ctrl.signal,
  onmessage(ev) {
    console.log(ev.data);
  }
});

// Cancel the request at any time:
// ctrl.abort();
```

## Better Error Handling

The following example distinguishes between "retriable" and "fatal" errors: client-side 4xx (except 429) are usually non-retriable, while server errors or unexpected closes trigger a retry.

```ts
class RetriableError extends Error {}
class FatalError extends Error {}

fetchEventSource('/api/sse', {
  async onopen(response) {
    if (response.ok && response.headers.get('content-type') === EventStreamContentType) {
      return; // OK
    } else if (response.status >= 400 && response.status < 500 && response.status !== 429) {
      throw new FatalError(); // client-side error, do not retry
    } else {
      throw new RetriableError(); // otherwise retry
    }
  },
  onmessage(msg) {
    if (msg.event === 'FatalError') {
      throw new FatalError(msg.data);
    }
  },
  onclose() {
    throw new RetriableError(); // server closed unexpectedly, retry
  },
  onerror(err) {
    if (err instanceof FatalError) {
      throw err; // rethrow to stop the whole operation
    }
    // otherwise do nothing to retry automatically, or return a specific interval in milliseconds
  }
});
```

For more on reconnection and `onerror` semantics, see [Reconnection and Error Handling](./guides/reconnection.md).

## Next Steps

- For the complete set of options and type definitions, see [API Reference](./api/index.md).
- For an overview, environment requirements, and core concepts, see [index](./index.md).
