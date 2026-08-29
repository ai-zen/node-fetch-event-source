---
title: API 参考
description: fetchEventSource 函数签名与配置项、EventSourceMessage 类型、EventStreamContentType 常量。
outline: deep
---

# API 参考

## 导出

包入口 `@ai-zen/node-fetch-event-source` 公开导出以下符号：

| 导出 | 类型 | 说明 |
|------|------|------|
| `fetchEventSource` | 函数 | 消费 SSE 事件流的主入口函数 |
| `FetchEventSourceInit` | 接口 | `fetchEventSource` 的配置项类型 |
| `EventStreamContentType` | 常量 | `'text/event-stream'` |
| `EventSourceMessage` | 接口 | 单条事件消息的形状 |

## `fetchEventSource(input, init)`

```ts
function fetchEventSource(
  input: RequestInfo,
  init: FetchEventSourceInit
): Promise<void>
```

- **`input`**：目标 URL，或一个 `Request` 对象（`RequestInfo`）。
- **返回**：`Promise<void>`。当连接自然关闭、被 `signal` 中止，或抛出致命错误被 `reject` 时 resolve；若在 `onerror` 中重新抛出错误，则该 Promise 会被 reject。
- `fetchEventSource` 正常关闭（服务端结束流、`onclose` 无异常）或 `signal` 中止时会 resolve。

## `FetchEventSourceInit`

继承自 [`RequestInit`](https://developer.mozilla.org/en-US/docs/Web/API/RequestInit)，因此 `method`、`body`、`credentials`、`mode`、`cache` 等字段均可用。本库额外声明/覆盖的字段如下：

| 字段 | 类型 | 说明 |
|------|------|------|
| `headers` | `Record<string, string>` | 请求头。**仅支持普通对象格式**，不兼容 `Headers` 实例。若未设置 `accept`，会默认填充为 `EventStreamContentType`。 |
| `signal` | `AbortSignal` | 来自 `RequestInit`。中止时会释放资源并 resolve。 |
| `onopen` | `(response: Response) => Promise<void>` | 收到响应时调用，用于校验响应是否符合预期（不匹配则抛出）。缺省时使用默认校验：`Content-Type` 需以 `text/event-stream` 开头。 |
| `onmessage` | `(ev: EventSourceMessage) => void` | 每收到一条完整事件消息时调用。**所有**事件都会触发，包括自定义 `event` 字段。 |
| `onclose` | `() => void` | 响应流结束时调用。若不希望服务端关闭连接，可在此抛出异常以触发重连。 |
| `onerror` | `(err: any) => number \| null \| undefined \| void` | 任意错误（请求、解析、回调内抛出的异常等）时调用，用于控制重试策略：返回毫秒数作为下次重连间隔；若重新抛出异常则停止整个操作。未指定或返回 `undefined`/`null` 时视为可重试，默认按 1000ms（或服务端 `retry:` 指定值）重试。 |
| `openWhenHidden` | `boolean` | 若为 `true`，页面隐藏时仍保持连接。默认情况下，`fetchEventSource` 会在页面隐藏（document 不可见）时关闭连接，并在重新可见时自动携带 `last-event-id` 重连。 |
| `fetch` | `typeof fetch` | 自定义 `fetch` 实现。缺省时优先使用全局 `fetch`，否则回退到 `cross-fetch`。 |

> 说明：`onerror` 返回 `null` 与返回 `undefined`/`void` 语义一致（都回退到当前 `retryInterval`）。要实现「不再重试」，需要在 `onerror` 中**重新抛出**异常，被捕获后 Promise 会 reject。

## `EventStreamContentType`

常量字符串 `'text/event-stream'`，用于在 `onopen` 中比对响应 `Content-Type`，或在自定义头中设置 `accept`。

## `EventSourceMessage`

单条事件消息的形状，字段如下：

```ts
interface EventSourceMessage {
  /** 事件 ID，用于设置 EventSource 对象的 lastEventId */
  id: string;
  /** 描述事件类型的字符串 */
  event: string;
  /** 事件数据 */
  data: string;
  /** 重连间隔（毫秒）；仅当服务端发送 retry 字段时存在 */
  retry?: number;
}
```

字段语义遵循 [Event Stream 解释规范](https://html.spec.whatwg.org/multipage/server-sent-events.html#event-stream-interpretation)：

- `id`、`event`、`data` 在未出现时初始化为**空字符串**。
- 多条 `data:` 行会被拼接为单字符串，用 `\n` 连接（如 `Foo`、`Bar`、空行、`Baz` 会得到 `'Foo\nBar\n\nBaz'`）。
- `retry:` 只接受整数，非整数会被忽略。

## 内部解析函数（非公开 API）

`src/parse.ts` 导出了 `getBytes`、`getLines`、`getMessages` 等底层函数，但它们**并未**从包入口（`index.ts`）重新导出，因此不属于公开 API，可能随版本变化，请勿依赖。

## 相关页面

- [快速开始](../getting-started.md)
- [重连与错误处理](../guides/reconnection.md)
- [index](../index.md)
