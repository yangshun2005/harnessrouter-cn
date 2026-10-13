# 流式传输

**统一 Harness 协议，版本 `2026-10-04`**

Agent 任务耗时以分钟计。没有流，产品只能显示一个加载动画然后干等。本章定义服务器如何在任务执行期间上报进度。

## 1. 开启流

在 `POST /v1/responses` 上设置 `"stream": true`。服务器必须返回：

```http
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
```

每个事件是一条 Server-Sent Events 消息，其 `data` 为一个 JSON 对象：

```
data: {"type":"response.created","sequence_number":0,"response":{…}}

data: {"type":"response.output_text.delta","sequence_number":7,"item_id":"msg_1","output_index":0,"content_index":0,"delta":"Sum"}

data: {"type":"response.completed","sequence_number":42,"response":{…}}

```

规则：

- 每个事件必须携带 `type` 和 `sequence_number`。
- 同一条流内，`sequence_number` 必须从 `0` 开始，每个事件恰好递增 1。客户端因此能检测到事件丢失，而不是默默渲染出一段空白。
- 流必须以恰好一个终止事件结束（§4）。
- 服务器不得先缓冲整条流再发送。最后一次性发出的流不是流，客户端无法区分它和卡死。
- 位于代理之后的服务器必须禁用响应缓冲。这是 UHP 服务器最常见的部署错误；它的表现和 harness 变慢一模一样。

## 2. 事件词汇表

事件描述*发生了什么*，从不描述如何显示。

### 2.1 响应生命周期

| 事件 | 时机 |
|---|---|
| `response.created` | 任务已被接受。携带初始 `response`，其 `status: in_progress`。 |
| `response.in_progress` | 工作已开始。 |
| `response.completed` | 终止——成功。携带最终 `response`。 |
| `response.incomplete` | 终止——预算中止了工作。携带最终 `response`。 |
| `response.failed` | 终止——失败。携带最终 `response`，其 `error` 非空。 |

### 2.2 输出项

| 事件 | 时机 |
|---|---|
| `response.output_item.added` | 新项开始。携带 `output_index` 和一个空壳 `item`。 |
| `response.output_item.done` | 该项已定稿。携带完整的 `item`。 |

### 2.3 助手文本

| 事件 | 载荷 |
|---|---|
| `response.content_part.added` | 一个空的 `output_text` 部分开启 |
| `response.output_text.delta` | `delta`：下一段文本片段 |
| `response.output_text.done` | `text`：该部分的完整文本 |
| `response.content_part.done` | `part`：已完成的部分 |
| `response.output_text.annotation.added` | `annotation`：构建产物引用（见 [Files](files.md)） |

文本增量是片段，不是行，也不是 token。客户端必须按 `sequence_number` 顺序拼接，且不得假设任何特定的分块方式。

### 2.4 推理

| 事件 | 载荷 |
|---|---|
| `response.reasoning_summary_part.added` | 一个空的摘要部分开启 |
| `response.reasoning_summary_text.delta` | `delta`：下一段摘要片段 |
| `response.reasoning_summary_part.done` | `part`：已完成的摘要部分 |

推理是*摘要*，不是逐字的思维链。服务器不得被要求暴露原始模型推理，客户端必须将推理视为可选——许多 harness 与模型的组合不产生推理，要求推理的客户端会在这些情况下出错。

### 2.5 工具使用

| 事件 | 载荷 |
|---|---|
| `response.function_call_arguments.delta` | `delta`：JSON 参数的片段 |
| `response.function_call_arguments.done` | `arguments`：完整的 JSON 字符串 |

工具调用本身以类型为 `function_call` 的 `output_item` 到达，其结果以 `function_call_output` 到达，通过 `call_id` 匹配。

### 2.6 错误

| 事件 | 载荷 |
|---|---|
| `error` | `code`、`message`、`param`——未终止任务的错误 |

`error` 事件之后必须跟一个终止事件。发出 `error` 后没有终止事件就停止的流是畸形的：客户端无法判断是任务死了还是连接断了。

## 3. 顺序保证

服务器必须保证：

1. `response.created` 是第一个事件。
2. 恰好一个终止事件是最后一个。
3. 对任意项，`output_item.added` 先于所有引用它的事件，`output_item.done` 后于所有引用它的事件。
4. `sequence_number` 严格单调且无缺口。

服务器不得保证、客户端也不得假设：

- 项按添加顺序完成（一个长时间的工具调用可能在后续文本之后才完成）；
- 文本以任何特定分块大小到达；
- 任何可选事件类型一定会出现。

## 4. 终止事件

`response.completed`、`response.incomplete`、`response.failed` 中恰好一个。每个都携带完整的最终 `response` 对象——错过中间事件的客户端可以仅凭这一个事件渲染结果。这种冗余是刻意的：流中途断连应付出延迟代价，而非正确性代价。

被取消的任务以 `response.failed` 终止，其 response 对象中携带 `status: "cancelled"`。以 status 字段为准，而非事件名。

## 5. 重连

断连不得中止任务。工作在服务器端继续。

断连后要跟进任务，客户端重新读取：

```http
GET /v1/responses/{response_id}
```

或者，要获取 harness 会话的实时事件流：

```http
GET /v1/harnesses/{harness_id}/events
```

服务器可以额外支持 SSE `Last-Event-ID` 续传。若支持，必须从给定 `sequence_number` *之后*的事件恢复，且不得重放客户端已见过的事件。

> 客户端应将流视为优化，将存储的响应视为事实来源。反过来写的产品——流是结果存在的唯一地方——每次负载均衡器回收连接都会丢失工作。

## 6. 非流式

使用 `"stream": false`（默认值）时，服务器在任务到达终止状态时返回单个响应对象，其 `output` 数组与流式本应组装出的一致。相同输入下，两条路径必须产生相同结果。流式与非流式组装出不同 `output` 的服务器，等于把自己的协议实现了两遍，其中一遍是错的。