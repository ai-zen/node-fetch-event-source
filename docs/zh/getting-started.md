---
title: 快速开始
description: 安装并运行 @ai-zen/node-fetch-event-source 的最小示例，以及 POST、自定义头和 AbortController 用法。
outline: deep
---

# 快速开始

## 安装

```bash
npm install @ai-zen/node-fetch-event-source
```

## 最小示例

`fetchEventSource` 会在后台持续消费服务端事件流，并把每条事件交给 `onmessage`：

```ts
import { fetchEventSource } from '@ai-zen/node-fetch-event-source';

await fetchEventSource('/api/sse', {
  onmessage(ev) {
    console.log(ev.data);
  }
});
```

`onmessage` 会被**所有**事件触发，包括带有自定义 `event` 字段的事件（这一点与浏览器内置 `EventSource.onmessage` 不同——后者只触发默认的 `message` 事件）。

## 发送请求体与自定义头

由于底层直接使用 `fetch`，你可以传入 `RequestInit` 支持的全部参数：

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

> 注意：`headers` 仅支持 `Record<string, string>` 格式（不兼容 `Headers` 实例）。

## 使用 AbortController 取消

传入 `signal` 即可在需要时中止连接。当 `signal` 触发 `abort` 时，资源会被释放，并且 `fetchEventSource` 返回的 Promise 会被 resolve（不会进入 `onerror` 重试流程）。

```ts
import { fetchEventSource } from '@ai-zen/node-fetch-event-source';

const ctrl = new AbortController();

await fetchEventSource('/api/sse', {
  signal: ctrl.signal,
  onmessage(ev) {
    console.log(ev.data);
  }
});

// 在任意时刻取消请求：
// ctrl.abort();
```

## 更好的错误处理

以下示例区分「可重试」与「致命」错误：客户端 4xx（除 429 外）通常不可重试，而服务端异常或意外关闭则触发重试。

```ts
class RetriableError extends Error {}
class FatalError extends Error {}

fetchEventSource('/api/sse', {
  async onopen(response) {
    if (response.ok && response.headers.get('content-type') === EventStreamContentType) {
      return; // OK
    } else if (response.status >= 400 && response.status < 500 && response.status !== 429) {
      throw new FatalError(); // 客户端错误，不重试
    } else {
      throw new RetriableError(); // 否则重试
    }
  },
  onmessage(msg) {
    if (msg.event === 'FatalError') {
      throw new FatalError(msg.data);
    }
  },
  onclose() {
    throw new RetriableError(); // 服务器意外关闭，重试
  },
  onerror(err) {
    if (err instanceof FatalError) {
      throw err; // 重新抛出以停止整个操作
    }
    // 否则不做任何事即自动重试，也可返回具体毫秒数指定间隔
  }
});
```

更多关于重连与 `onerror` 语义的说明见 [重连与错误处理](./guides/reconnection.md)。

## 下一步

- 完整的配置项与类型定义见 [API 参考](./api/index.md)。
- 概述、环境要求与核心概念见 [index](./index.md)。
