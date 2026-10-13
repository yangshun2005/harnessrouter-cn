# 错误

**统一 Harness 协议，版本 `2026-09-12`**

客户端的错误处理水平，取决于服务端的错误报告。本章定义一个信封、一组封闭的错误码，以及哪些失败值得重试。

## 1. 错误信封

每个非 2xx 响应都必须包含以下响应体：

```json
{
  "error": {
    "type": "invalid_request_error",
    "code": "harness_not_found",
    "message": "No harness with id 'chrn_deadbeef'.",
    "param": "metadata.harness_id",
    "detail": null
  }
}
```

| 字段 | 类型 | 必填 | 含义 |
|---|---|---|---|
| `type` | string | 是 | 大类；见 §2 |
| `code` | string | 是 | 具体的、机器可读的原因；见 §3 |
| `message` | string | 是 | 人类可读，一句话，不含堆栈跟踪 |
| `param` | string \| null | 是 | 出错字段的点分路径（如果有） |
| `detail` | object \| null | 是 | 结构化的额外上下文，或 `null` |

规则：

- `message` 必须可以安全地展示给用户。不得包含凭据、内部主机名、文件路径或堆栈跟踪。
- 当 §3 中有适用的 `code` 时，必须使用 §3 中的值。服务端可以为本规范未覆盖的情况定义额外错误码，但必须加厂商前缀（`acme_…`）作为命名空间，以免与本规范的未来版本冲突。
- 服务端不得返回 `200` 并在其中携带错误。*任务*的失败以 `status: "failed"` 和 HTTP `200` 的响应报告，因为请求本身成功了；*请求*的失败则是带此信封的非 2xx 响应。这是两类不同的事件，不得混为一谈。

> **兼容性说明。** 服务端可以额外在 `error` 旁边包含一个顶层 `detail` 字符串，供针对早于本信封的实现编写的客户端使用。该字段已弃用，不携带 `error.message` 之外的信息，将在未来版本中移除。

## 2. 错误类型

| `type` | HTTP | 含义 |
|---|---|---|
| `invalid_request_error` | 400, 404, 409, 413, 422 | 请求有误。原样重试仍会失败。 |
| `authentication_error` | 401 | 凭据缺失、格式错误或未知。 |
| `permission_error` | 403 | 已认证，但不允许执行此操作。 |
| `rate_limit_error` | 429 | 请求过多，或配额已耗尽。 |
| `harness_error` | 200（在失败响应中） | Harness 运行了但失败了。 |
| `server_error` | 500, 502, 503, 504 | 服务端失败。重试可能成功。 |

## 3. 错误码

### 3.1 请求与路由

| 错误码 | HTTP | 含义 |
|---|---|---|
| `unsupported_protocol_version` | 400 | `UHP-Version` 指定的版本此服务端无法提供。`detail.supported` 列出可提供的版本。 |
| `invalid_input` | 400 | 响应体无法解析，或某字段类型错误。 |
| `harness_not_found` | 404 | 调用方范围内无此 harness。 |
| `response_not_found` | 404 | 调用方范围内无此响应。 |
| `session_not_found` | 404 | 调用方范围内无此会话。 |
| `plugin_not_found` | 404 | 该 harness 没有此名称的插件。 |
| `file_not_found` | 404 | 调用方范围内无此文件。 |
| `session_expired` | 404 | 会话曾存在，但已超过保留期。 |
| `harness_mismatch` | 409 | 任务指定的 harness 与其所延续的会话不一致。 |
| `session_busy` | 409 | 此会话中已有任务在运行。 |
| `plugin_conflict` | 409 | 两个插件同名，或组件名在 harness 与其插件之间冲突。`detail.component`、`detail.name` 和 `detail.between` 指明具体情况。 |
| `file_too_large` | 413 | 上传超过服务端限制。`detail.max_bytes` 给出限制值。 |
| `model_unavailable` | 422 | 此 harness 无法提供所请求的模型。 |
| `unsupported_base` | 422 | 此服务端不支持所请求的 harness 基础。 |
| `plugin_invalid` | 422 | 该包不是有效的 Agent Plugins 包：无清单、清单存在致命错误、路径逃逸出根目录，或 `name` 与清单矛盾。`detail.path` 和 `detail.reason` 指明位置。 |
| `unsupported_plugin_schema` | 422 | 清单的 `$schema` 指定的 Agent Plugins 版本此服务端不安装。`detail.supported` 列出可安装的版本。 |
| `unsupported_transport` | 422 | 插件声明的 MCP 服务器，其传输方式无法在此 harness 基础上运行。`detail.transport` 和 `detail.base` 指明具体情况。 |

### 3.2 认证与限制

| 错误码 | HTTP | 含义 |
|---|---|---|
| `missing_credential` | 401 | 无 bearer token。 |
| `invalid_credential` | 401 | token 格式错误、未知或已撤销。 |
| `insufficient_scope` | 403 | 凭据无法执行此操作。 |
| `rate_limited` | 429 | 请放慢速度。应设置 `Retry-After`。 |
| `quota_exhausted` | 429 | 预算已用尽；立即重试无济于事。 |

### 3.3 执行

| 错误码 | 出现位置 | 含义 |
|---|---|---|
| `harness_error` | 失败响应 | Harness 运行了但无法完成工作。 |
| `harness_unavailable` | 503 | 当前无容量运行此 harness。 |
| `provider_error` | 失败响应 | 上游模型提供方拒绝或失败。 |
| `timeout` | 失败响应 | 任务超过其墙钟时间预算。 |
| `cancelled` | 已取消响应 | 客户端取消了它。 |
| `preview_unavailable` | 501 | 此服务端不渲染预览。 |
| `preview_failed` | 502 | 此文件的转换失败。 |

## 4. 重试

| 情况 | 重试？ |
|---|---|
| `server_error`（500, 502, 503, 504） | 是，采用指数退避加抖动 |
| `rate_limited` | 是，在 `Retry-After` 之后 |
| `quota_exhausted` | 否——配额不变则无济于事 |
| `session_busy` | 是，待进行中的任务进入终态后 |
| 任何 `invalid_request_error` | 否——修正请求 |
| `authentication_error` | 否——修正凭据 |
| 已进入 `failed` 的任务 | 仅凭客户端自行判断；它已消耗真实资源 |

**`POST /v1/responses` 的重试必须携带 `Idempotency-Key`。** 否则，超时后的重试会再次运行任务——而第一次可能仍在运行，编辑同一批文件。这是 UHP 客户端可能犯的最具破坏性的错误；见[任务 §6](tasks.md)。

## 5. 超时

客户端应设置宽松的超时。Agent 任务通常运行数分钟；30 秒的 HTTP 超时会取消健康的工作。

- 流式：不设总超时。改用不活动超时——服务端应至少每 30 秒发出一个注释行（`: keep-alive`），以便客户端区分工作中的 agent 与已断开的连接。
- 非流式：至少为任务的 `timeout_seconds`，外加余量。
- 放弃的客户端不得假定任务已停止。它并未停止。请显式取消。