# 错误

**统一 Harness 协议，版本 `2026-10-04`**

客户端的错误处理，取决于服务端的错误报告。本章定义一个信封、一组封闭的错误码，以及哪些失败值得重试。

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
| `param` | string \| null | 是 | 出错字段的点分路径，若有 |
| `detail` | object \| null | 是 | 结构化附加上下文，或 `null` |

规则：

- `message` 必须可安全展示给用户。不得包含凭据、内部主机名、文件路径或堆栈跟踪。
- 若 §3 中有适用的 `code`，必须使用该值。服务端可为本规范未覆盖的情况定义额外错误码，但必须加厂商前缀（`acme_…`）作为命名空间，以免与本规范的未来版本冲突。
- 服务端不得在 `200` 响应中内嵌错误。*任务*失败以 HTTP `200` 加 `status: "failed"` 报告，因为请求本身成功了；*请求*失败才是带此信封的非 2xx。二者是不同事件，不得混为一谈。

> **兼容性说明。** 服务端可在 `error` 之外额外包含顶层 `detail` 字符串，供针对早于本信封的实现编写的客户端使用。该字段已弃用，不含 `error.message` 之外的信息，将在未来版本移除。

## 2. 错误类型

| `type` | HTTP | 含义 |
|---|---|---|
| `invalid_request_error` | 400、404、409、413、422 | 请求有误。原样重试仍会失败。 |
| `authentication_error` | 401 | 凭据缺失、格式错误或未知。 |
| `permission_error` | 403 | 已认证，但无权执行此操作。 |
| `rate_limit_error` | 429 | 请求过多，或配额耗尽。 |
| `harness_error` | 200（失败响应中） | Harness 运行了但失败。 |
| `server_error` | 500、502、503、504 | 服务端失败。重试可能成功。 |

## 3. 错误码

### 3.1 请求与路由

| 错误码 | HTTP | 含义 |
|---|---|---|
| `unsupported_protocol_version` | 400 | `UHP-Version` 指定的版本此服务端无法提供。`detail.supported` 列出可提供的版本。 |
| `invalid_input` | 400 | 响应体无法解析，或字段类型错误。 |
| `harness_not_found` | 404 | 调用方范围内无此 Harness。 |
| `response_not_found` | 404 | 调用方范围内无此响应。 |
| `session_not_found` | 404 | 调用方范围内无此会话。 |
| `plugin_not_found` | 404 | 该 Harness 无此名称的插件。 |
| `file_not_found` | 404 | 调用方范围内无此文件。 |
| `session_expired` | 404 | 会话曾存在，但已超过保留期。 |
| `harness_mismatch` | 409 | 任务指定的 Harness 与其所延续的会话不一致。 |
| `session_busy` | 409 | 此会话中已有任务在运行。 |
| `plugin_conflict` | 409 | 两个插件同名，或组件名在 Harness 与其插件间冲突。`detail.component`、`detail.name` 和 `detail.between` 指明具体冲突。 |
| `file_too_large` | 413 | 上传超过服务端限制。`detail.max_bytes` 给出限制值。 |
| `model_unavailable` | 422 | 请求的模型无法为此 Harness 提供。 |
| `unsupported_base` | 422 | 此服务端不支持请求的 Harness 基础。 |
| `environment_not_found` | 404 | 调用方范围内无此环境，出现在读取、Harness 写入或任务中（[环境 §8](environments.md#8-errors)）。 |
| `environment_not_ready` | 409 | 在无活跃版本的环境上执行任务，或激活未完成的构建。 |
| `environment_busy` | 409 | 构建正在运行：第二次构建或删除必须等待。 |
| `environment_exists` | 409 | 该名称的 slug 已是另一环境的挂载路径。 |
| `environment_invalid` | 422 | 名称不是路径段、文件路径逃逸出根目录，或导入既非归档也非仓库。 |
| `environment_unavailable` | 502 | 服务端的构建或存储后端未响应。 |
| `memory_not_found` | 404 | 无此记忆，或调用方对其无权限：二者是同一答案（[记忆 §12](memories.md#12-errors)）。 |
| `memory_forbidden` | 403 | 调用方能看到该记忆，但缺少操作所需的权限。 |
| `memory_record_not_found` | 404 | 此记忆中无此记录。 |
| `memories_not_attached` | 404 | 智能体未持有任何记忆：未授予其任何记忆。 |
| `memory_invalid` | 422 | 父级会形成环、两个默认条目、无内容的记录、参数不匹配的查询。 |
| `memory_unsupported` | 422 | 记忆的提供方完全未实现该操作。部分完成的答案在 `degraded` 中报告，而非此错误码。 |
| `memory_busy` | 409 | 整合运行时执行移动或删除。 |
| `memory_unavailable` | 502 或 503 | 记忆的提供方未响应，或未连接。 |
| `plugin_invalid` | 422 | 该包不是有效的 Agent Plugins 包：无清单、清单有致命错误、路径逃逸出根目录，或 `name` 与清单矛盾。`detail.path` 和 `detail.reason` 指明位置。 |
| `unsupported_plugin_schema` | 422 | 清单的 `$schema` 指定的 Agent Plugins 版本此服务端不安装。`detail.supported` 列出可安装的版本。 |
| `unsupported_transport` | 422 | 插件声明的 MCP 服务器，其传输无法在此 Harness 基础上运行。`detail.transport` 和 `detail.base` 指明具体值。 |

### 3.2 认证与限制

| 错误码 | HTTP | 含义 |
|---|---|---|
| `missing_credential` | 401 | 无 Bearer 令牌。 |
| `invalid_credential` | 401 | 令牌格式错误、未知或已撤销。 |
| `insufficient_scope` | 403 | 凭据无法执行此操作。 |
| `rate_limited` | 429 | 请放慢速度。应设置 `Retry-After`。 |
| `quota_exhausted` | 429 | 预算已用尽；立即重试无济于事。 |

### 3.3 执行

| 错误码 | 出现位置 | 含义 |
|---|---|---|
| `harness_error` | 失败响应 | Harness 运行了但无法完成工作。 |
| `harness_unavailable` | 503 | 当前无容量运行此 Harness。 |
| `provider_error` | 失败响应 | 上游模型提供方拒绝或失败。 |
| `timeout` | 失败响应 | 任务超出其墙钟时间预算。 |
| `cancelled` | 已取消响应 | 客户端取消了它。 |
| `preview_unavailable` | 501 | 此服务端不渲染预览。 |
| `preview_failed` | 502 | 此文件的转换失败。 |

## 4. 重试

| 情况 | 重试？ |
|---|---|
| `server_error`（500、502、503、504） | 是，使用指数退避加抖动 |
| `rate_limited` | 是，在 `Retry-After` 之后 |
| `quota_exhausted` | 否——配额不变则无济于事 |
| `session_busy` | 是，待进行中的任务进入终态后 |
| 任何 `invalid_request_error` | 否——修正请求 |
| `authentication_error` | 否——修正凭据 |
| 已进入 `failed` 的任务 | 仅凭客户端自行判断；它消耗了真实资源 |

**重试 `POST /v1/responses` 必须携带 `Idempotency-Key`。** 否则，超时后的重试会再次运行任务——而第一次可能仍在运行，编辑同一批文件。这是 UHP 客户端可能犯的最具破坏性的错误；见 [任务 §6](tasks.md)。

## 5. 超时

客户端应设置宽松的超时。智能体任务通常运行数分钟；30 秒的 HTTP 超时会取消健康的工作。

- 流式：不设总超时。改用不活动超时——服务端应至少每 30 秒发送一行注释（`: keep-alive`），以便客户端区分工作中的智能体与已断开的连接。
- 非流式：至少为任务的 `timeout_seconds` 加余量。
- 放弃的客户端不得假定任务已停止。它并未停止。应显式取消。