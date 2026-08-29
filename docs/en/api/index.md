---
title: API Reference
description: The fetchEventSource function signature and options, the EventSourceMessage type, and the EventStreamContentType constant.
outline: deep
---

# API Reference

## Exports

The package entry `@ai-zen/node-fetch-event-source` publicly exports the following symbols:

| Export | Type | Description |
|------|------|------|
| `fetchEventSource` | function | The main entry function for consuming SSE event streams |
| `FetchEventSourceInit` | interface | The options type for `fetchEventSource` |
| `EventStreamContentType` | constant | `'text/event-stream'` |
| `EventSourceMessage` | interface | The shape of a single event message |

## `fetchEventSource(input, init)`

```ts
function fetchEventSource(
  input: RequestInfo,
  init: FetchEventSourceInit
): Promise<void>
```

- **`input`**: the target URL, or a `Request` object (`RequestInfo`).
- **Returns**: `Promise<void>`. It resolves when the connection closes naturally, is aborted by `signal`, or when a fatal error is thrown and rejected; if an error is rethrown inside `onerror`, the Promise is rejected.
- `fetchEventSource` resolves on a normal close (the server ends the stream and `onclose` does not throw) or when `signal` aborts.

## `FetchEventSourceInit`

It extends [`RequestInit`](https://developer.mozilla.org/en-US/docs/Web/API/RequestInit), so fields such as `method`, `body`, `credentials`, `mode`, `cache` are all available. The fields additionally declared/overridden by this library are as follows:

| Field | Type | Description |
|------|------|------|
| `headers` | `Record<string, string>` | The request headers. **Only supports a plain object format**, not a `Headers` instance. If `accept` is not set, it is defaulted to `EventStreamContentType`. |
| `signal` | `AbortSignal` | From `RequestInit`. When aborted, resources are released and it resolves. |
| `onopen` | `(response: Response) => Promise<void>` | Called when a response is received, used to validate that the response matches expectations (throw if it does not). When absent, the default validation is used: the `Content-Type` must start with `text/event-stream`. |
| `onmessage` | `(ev: EventSourceMessage) => void` | Called for each complete event message received. **All** events trigger it, including those with a custom `event` field. |
| `onclose` | `() => void` | Called when the response stream ends. If you do not want the server to close the connection, you can throw here to trigger a reconnection. |
| `onerror` | `(err: any) => number \| null \| undefined \| void` | Called for any error (request, parsing, exceptions thrown in callbacks, etc.) to control the retry strategy: return a number in milliseconds as the next reconnection interval; rethrow an exception to stop the whole operation. When unspecified or returning `undefined`/`null`, it is treated as retriable and retries at the default 1000ms (or the server-specified `retry:` value). |
| `openWhenHidden` | `boolean` | If `true`, keeps the connection open when the page is hidden. By default, `fetchEventSource` closes the connection when the document is hidden (not visible) and automatically reconnects with `last-event-id` when it becomes visible again. |
| `fetch` | `typeof fetch` | A custom `fetch` implementation. When absent, it prefers the global `fetch`, otherwise falls back to `cross-fetch`. |

> Note: Returning `null` from `onerror` has the same semantics as returning `undefined`/`void` (both fall back to the current `retryInterval`). To achieve "do not retry", you must **rethrow** an exception inside `onerror`, after which the Promise is rejected.

## `EventStreamContentType`

The constant string `'text/event-stream'`, used to compare the response `Content-Type` in `onopen`, or to set the `accept` header in custom headers.

## `EventSourceMessage`

The shape of a single event message, with the following fields:

```ts
interface EventSourceMessage {
  /** The event ID, used to set the EventSource object's lastEventId */
  id: string;
  /** A string identifying the type of event described */
  event: string;
  /** The event data */
  data: string;
  /** The reconnection interval (in milliseconds); only present when the server sends a retry field */
  retry?: number;
}
```

The field semantics follow the [Event Stream interpretation](https://html.spec.whatwg.org/multipage/server-sent-events.html#event-stream-interpretation):

- `id`, `event`, `data` are initialized to an **empty string** when absent.
- Multiple `data:` lines are concatenated into a single string joined by `\n` (e.g., `Foo`, `Bar`, blank line, `Baz` yields `'Foo\nBar\n\nBaz'`).
- `retry:` only accepts integers; non-integers are ignored.

## Internal Parsing Functions (Non-Public API)

`src/parse.ts` exports low-level functions such as `getBytes`, `getLines`, `getMessages`, but they are **not** re-exported from the package entry (`index.ts`), so they are not part of the public API and may change between versions. Please do not depend on them.

## Related Pages

- [Quick Start](../getting-started.md)
- [Reconnection and Error Handling](../guides/reconnection.md)
- [index](../index.md)
