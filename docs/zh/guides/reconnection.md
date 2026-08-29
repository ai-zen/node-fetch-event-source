---
title: 重连与错误处理
description: 了解 fetchEventSource 的重试策略、last-event-id 续传，以及页面可见性（Page Visibility）行为。
outline: deep
---

# 重连与错误处理

`fetchEventSource` 在错误发生时不会自行静默重试**有限次后放弃**，而是把控制权完全交给你，通过 `onerror` 决定下一步。

## 默认重试行为

- 默认重试间隔为 **1000ms**（源码中的 `DefaultRetryInterval`）。
- 若无 `onerror`，或 `onerror` 返回 `undefined` / `null`，所有错误都被视为**可重试**，将在 `retryInterval` 毫秒后自动重连。

## 控制重试间隔

`onerror` 返回一个**毫秒数**即可指定下次重连的间隔：

```ts
fetchEventSource('/api/sse', {
  onerror(err) {
    return 5000; // 5 秒后重连
  }
});
```

服务端也可通过 `retry:` 字段更新默认间隔。该值会被 `onerror` 未返回具体数字时的默认（`retryInterval`）使用。

## 停止重连

在 `onerror` 中**重新抛出**异常即可停止整个操作，此时 `fetchEventSource` 的 Promise 会 reject：

```ts
class FatalError extends Error {}

fetchEventSource('/api/sse', {
  onerror(err) {
    if (err instanceof FatalError) {
      throw err; // 停止重连，Promise reject
    }
    // 否则自动重试
  }
});
```

## 自定义 `onopen`

默认的 `onopen` 只校验响应 `Content-Type` 是否以 `text/event-stream` 开头。若想对网关返回的错误（如 nginx 的 4xx/5xx）做二次判断，可以覆盖 `onopen` 并抛出可重试 / 不可重试的错误：

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

## 处理意外关闭（`onclose`）

当服务端结束响应但并非你预期的关闭时，可在 `onclose` 中抛出异常以进入 `onerror` 重连流程：

```ts
fetchEventSource('/api/sse', {
  onclose() {
    throw new RetriableError();
  },
  onerror(err) {
    // 处理重试
  }
});
```

## `last-event-id` 续传

库会在收到 `id:` 字段时记录该值，并在下一次重连时以 `last-event-id` 请求头发送回服务端，从而支持断点续传：

- 收到非空 `id` 时，会设置 `last-event-id` 头。
- 若服务端发送**空的 `id` 字段**，`last-event-id` 头将不再发送（即清除续传位置）。

## 页面可见性（Page Visibility）

默认（`openWhenHidden` 未开启）时，在浏览器环境下，当文档变为隐藏（如用户最小化窗口）`fetchEventSource` 会**中止当前请求**；当文档重新可见时会**自动重建连接**，并携带 `last-event-id` 续传。这减少了不必要的服务器连接占用。

若希望在页面隐藏时仍保持连接，可设置：

```ts
fetchEventSource('/api/sse', {
  openWhenHidden: true
});
```

## 相关页面

- [快速开始](../getting-started.md)
- [API 参考](../api/index.md)
- [index](../index.md)
