---
title: Reconnection and Error Handling
description: Understand fetchEventSource's retry strategy, last-event-id resume, and Page Visibility behavior.
outline: deep
---

# Reconnection and Error Handling

When an error occurs, `fetchEventSource` does not silently retry **a few times and then give up**; instead it hands full control to you via `onerror` to decide what happens next.

## Default Retry Behavior

- The default retry interval is **1000ms** (`DefaultRetryInterval` in the source).
- If there is no `onerror`, or `onerror` returns `undefined` / `null`, all errors are treated as **retriable** and will automatically reconnect after `retryInterval` milliseconds.

## Controlling the Retry Interval

Return a **number of milliseconds** from `onerror` to specify the next reconnection interval:

```ts
fetchEventSource('/api/sse', {
  onerror(err) {
    return 5000; // reconnect after 5 seconds
  }
});
```

The server can also update the default interval via the `retry:` field. That value is used as the fallback (the `retryInterval`) when `onerror` does not return a specific number.

## Stopping Reconnection

**Rethrow** an exception inside `onerror` to stop the whole operation, after which the Promise from `fetchEventSource` is rejected:

```ts
class FatalError extends Error {}

fetchEventSource('/api/sse', {
  onerror(err) {
    if (err instanceof FatalError) {
      throw err; // stop reconnection, Promise rejects
    }
    // otherwise retry automatically
  }
});
```

## Custom `onopen`

The default `onopen` only validates that the response `Content-Type` starts with `text/event-stream`. If you want to make a secondary judgment on errors returned by a gateway (such as nginx 4xx/5xx), you can override `onopen` and throw a retriable / non-retriable error:

```ts
class RetriableError extends Error {}
class FatalError extends Error {}

fetchEventSource('/api/sse', {
  async onopen(response) {
    if (response.ok && response.headers.get('content-type') === EventStreamContentType) {
      return; // OK
    } else if (response.status >= 400 && response.status < 500 && response.status !== 429) {
      throw new FatalError();
    } else {
      throw new RetriableError();
    }
  },
  onerror(err) {
    if (err instanceof FatalError) throw err;
  }
});
```

## Handling Unexpected Closes (`onclose`)

When the server ends the response but it is not the close you expected, you can throw inside `onclose` to enter the `onerror` reconnection flow:

```ts
fetchEventSource('/api/sse', {
  onclose() {
    throw new RetriableError();
  },
  onerror(err) {
    // handle the retry
  }
});
```

## `last-event-id` Resume

The library records the value when it receives an `id:` field, and sends it back to the server as the `last-event-id` request header on the next reconnect, enabling resumption from a breakpoint:

- When a non-empty `id` is received, the `last-event-id` header is set.
- If the server sends an **empty `id` field**, the `last-event-id` header is no longer sent (i.e., the resume position is cleared).

## Page Visibility

By default (when `openWhenHidden` is not enabled), in a browser environment, when the document becomes hidden (e.g., the user minimizes the window), `fetchEventSource` will **abort the current request**; when the document becomes visible again, it will **automatically rebuild the connection** and resume with `last-event-id`. This reduces unnecessary server connection usage.

If you want to keep the connection alive while the page is hidden, you can set:

```ts
fetchEventSource('/api/sse', {
  openWhenHidden: true
});
```

## Related Pages

- [Quick Start](../getting-started.md)
- [API Reference](../api/index.md)
- [index](../index.md)
