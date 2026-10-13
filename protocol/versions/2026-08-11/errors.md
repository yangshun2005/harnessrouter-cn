# 错误

**统一 Harness 协议，版本 `2026-08-11`**

客户端的错误处理，取决于服务端的错误报告。本章定义一个信封、一组封闭的代码，以及哪些失败值得重试。

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
| `code` | string | 是 | 具体、机器可读的原因；见 §3 |
| `message` | string | 是 | 人类可读，一句话，不含堆栈跟踪 |
| `param` | string \| null | 是 | 出错字段的点分路径（若有） |
| `detail` | object \| null | 是 | 结构化附加上下文，或 `null` |

规则：

- `message` 必须可安全展示给用户。不得包含凭据、内部主机名、文件路径或堆栈跟踪。
- 若 §3 中有适用的 `code`，必须使用其中的值。服务端可为本规范未覆盖的情况定义额外代码，但必须加厂商前缀（`acme_…`）作为命名空间，以免与本规范的未来版本冲突。
- 服务端不得返回内含错误的 `200`。*任务*失败以 `status: "failed"` 且 HTTP `200` 的响应报告，因为请求本身成功了；*请求*失败则是带此信封的非 2xx。这是两类不同事件，不得混为一谈。

> **兼容性说明。** 服务端可额外在 `error` 旁包含顶层 `detail` 字符串，供针对早于本信封的实现编写的客户端使用。该字段已弃用，不携带 `error.message` 之外的信息，将在未来版本中移除。

## 2. 错误类型

| `type` | HTTP | 含义 |
|---|---|---|
| `invalid_request_error` | 400、404、409、413、422 | 请求有误。原样重试仍会失败。 |
| `authentication_error` | 401 | 凭据缺失、格式错误或未知。 |
| `permission_error` | 403 | 已认证，但无权执行此操作。 |
| `rate_limit_error` | 429 | 请求过多，或配额耗尽。 |
| `harness_error` | 200（在失败响应中） | Harness 已运行并失败。 |
| `server_error` | 500、502、503、504 | 服务端失败。重试可能成功。 |

## 3. 错误代码

### 3.1 请求与路由

| 代码 | HTTP | 含义 |
|---|---|---|
| `unsupported_protocol_version` | 400 | `UHP-Version` 指定的版本此服务端无法提供。`detail.supported` 列出可提供的版本。 |
| `invalid_input` | 400 | 响应体无法解析，或字段类型错误。 |
| `harness_not_found` | 404 | 调用方范围内无此 Harness。 |
| `response_not_found` | 404 | 调用方范围内无此响应。 |
| `session_not_found` | 404 | 调用方范围内无此会话。 |
| `file_not_found` | 404 | 调用方范围内无此文件。 |
| `session_expired` | 404 | 会话曾存在，但已超过保留期。 |
| `harness_mismatch` | 409 | 任务指定的 Harness 与其所延续的会话不一致。 |
| `session_busy` | 409 | 此会话中已有任务在运行。 |
| `file_too_large` | 413 | 上传超过服务端限制。`detail.max_bytes` 给出上限。 |
| `model_unavailable` | 422 | 此 Harness 无法提供所请求的模型。 |
| `unsupported_base` | 422 | 此服务端不支持所请求的 Harness 基础。 |

### 3.2 认证与限制

| 代码 | HTTP | 含义 |
|---|---|---|
| `missing_credential` | 401 | 无 Bearer 令牌。 |
| `invalid_credential` | 401 | 令牌格式错误、未知或已吊销。 |
| `insufficient_scope` | 403 | 凭据无法执行此操作。 |
| `rate_limited` | 429 | 请放慢速度。应设置 `Retry-After`。 |
| `quota_exhausted` | 429 | 预算已用尽；立即重试无济于事。 |

### 3.3 执行

| 代码 | 出现位置 | 含义 |
|---|---|---|
| `harness_error` | 失败响应 | Harness 已运行，但无法完成工作。 |
| `harness_unavailable` | 503 | 当前无容量运行此 Harness。 |
| `provider_error` | 失败响应 | 上游模型提供商拒绝或失败。 |
| `timeout` | 失败响应 | 任务超出其墙钟时间预算。 |
| `cancelled` | 已取消响应 | 客户端取消了它。 |
| `preview_unavailable` | 501 | 此服务端不渲染预览。 |
| `preview_failed` | 502 | 此文件的转换失败。 |

## 4. 重试

| 情况 | 重试？ |
|---|---|
| `server_error`（500、502、503、504） | 是，采用指数退避加抖动 |
| `rate_limited` | 是，在 `Retry-After` 之后 |
| `quota_exhausted` | 否——配额不变则无济于事 |
| `session_busy` | 是，待进行中的任务进入终态后 |
| 任何 `invalid_request_error` | 否——修正请求 |
| `authentication_error` | 否——修正凭据 |
| 已进入 `failed` 的任务 | 仅凭客户端自行判断；它已消耗真实资源 |

**重试 `POST /v1/responses` 必须携带 `Idempotency-Key`。** 否则，超时后的重试会再次运行任务——而第一次可能仍在运行，编辑同一批文件。这是 UHP 客户端可能犯下的最具破坏性的错误；见 [Tasks §6](tasks.md)。

## 5. 超时

客户端应设置宽松的超时。Agent 任务通常运行数分钟；30 秒的 HTTP 超时会取消健康的工作。

- 流式：无总超时。改用不活动超时——服务端应至少每 30 秒发出一个注释行（`: keep-alive`），以便客户端区分工作中的 Agent 与已断开的连接。
- 非流式：至少为任务的 `timeout_seconds`，另加余量。
- 放弃的客户端不得假定任务已停止。它并未停止。请显式取消。