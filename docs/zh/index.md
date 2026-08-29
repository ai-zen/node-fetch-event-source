---
title: node-fetch-event-source
description: 基于 Fetch 的 SSE 事件流消费库，为 server-sent events 提供更完整、可控的请求与重连能力。
outline: deep
---

# node-fetch-event-source

`@ai-zen/node-fetch-event-source`（源码仓库 `node-fetch-event-source`）是一个基于 [Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API) 的 [Server-Sent Events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events)（SSE）事件流消费库。它派生于 [Azure/fetch-event-source](https://github.com/Azure/fetch-event-source)，并完全兼容 [Event Stream 格式](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events#Event_stream_format)。

相比浏览器内置的 [`EventSource`](https://developer.mozilla.org/en-US/docs/Web/API/EventSource)，`fetchEventSource` 暴露了更多底层能力：

- 可使用任意请求方法 / 请求头 / 请求体，以及 `fetch()` 的所有其他能力（`credentials`、`mode`、`cache` 等）。
- 可提供自定义 `fetch` 实现（如 Node 环境下的 `cross-fetch`）。
- 可在解析事件流之前访问 `Response` 对象，用于对网关或上游错误的二次校验。
- 当连接中断或出错时，可完全自定义重试策略，而不是由浏览器静默重试有限次后放弃。

## 环境要求

- 运行时：Node.js（TypeScript 编译目标为 **ES2017**）或所有主流的常青浏览器（Chrome、Firefox、Safari、Edge）。
- 依赖：`cross-fetch`（`^4.0.0`），在全局 `fetch` 不可用时作为回退实现。
- 兼容性提示：旧版 Edge（< 79）需要先 polyfill `TextDecoder`，例如：

```js
require('fast-text-encoding');
```

## 安装

```bash
npm install @ai-zen/node-fetch-event-source
```

使用 pnpm 或 yarn 亦可：

```bash
pnpm add @ai-zen/node-fetch-event-source
```

## 快速示例

```ts
import { fetchEventSource } from '@ai-zen/node-fetch-event-source';

await fetchEventSource('/api/sse', {
  onmessage(ev) {
    console.log(ev.data);
  }
});
```

更完整的用法（POST、自定义头、AbortController 与错误处理）见 [快速开始](./getting-started.md)。

## 核心概念

- **`fetchEventSource(input, init)`**：本库的入口函数，返回 `Promise<void>`，在连接关闭或被中止时 resolve。
- **`EventStreamContentType`**：`'text/event-stream'` 常量；默认 `onopen` 会用 `startsWith` 校验响应 `Content-Type`。
- **`EventSourceMessage`**：单条事件消息的形状，包含 `id`、`event`、`data` 与可选的 `retry`。
- **回调管线**：`onopen` → 逐字节读取 → `getLines` 切分 → `getMessages` 组装 → `onmessage`；`onclose` / `onerror` 负责收尾与重连。
- **重连**：默认重试间隔 1000ms；可通过服务端 `retry:` 字段更新；通过 `onerror` 的返回值控制重试间隔，抛出异常则停止。详见 [重连与错误处理](./guides/reconnection.md)。

## 参考

- [快速开始](./getting-started.md) — 安装、最小示例、POST/Abort 示例。
- [API 参考](./api/index.md) — `fetchEventSource` 配置项、`EventSourceMessage` 类型与常量。
- [重连与错误处理](./guides/reconnection.md) — 重试策略、`last-event-id` 与页面可见性行为。
