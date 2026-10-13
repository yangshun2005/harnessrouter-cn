# 生命周期

**统一 Harness 协议，版本 `2026-09-12`**

本章定义客户端与服务器如何就协议版本达成一致、客户端如何获知服务器能力，以及任务从提交到结果所经历的状态。

## 1. 版本协商

UHP 版本是日期：`YYYY-MM-DD`，即版本发布当天。日期可排序，不会像语义化版本那样暗示一个无法兑现的兼容性承诺，还能让实现的年代一目了然。兼容性规则见 [VERSIONING.md](../../VERSIONING.md)。

客户端可以声明其编写时所针对的版本：

```http
UHP-Version: 2026-09-12
```

- 若缺少该请求头，服务器必须假定使用自身默认版本，并必须在响应头中注明该版本。省略请求头的客户端是在请求服务器代为选择，那就得到服务器的选择。
- 若该请求头指定的版本服务器支持，服务器必须在该请求中遵循它。
- 若该请求头指定的版本服务器不支持，服务器必须以 `400` 和 `code: "unsupported_protocol_version"` 使请求失败，且错误的 `detail` 必须列出其支持的版本。服务器不得静默地提供其他版本——客户端请求了它能解析的版本，就不应收到它无法解析的版本。

每个响应（包括错误）都必须携带实际使用的版本：

```http
UHP-Version: 2026-09-12
```

## 2. 能力发现

```http
GET /v1/uhp
```

无论是否认证——服务器必须无需 bearer token 即可提供此端点，因为客户端需要先知道自己在与一个 UHP 服务器通信，才能合理地出示凭据。该文档不得包含任何与主体相关的信息。

```json
{
  "object": "uhp.discovery",
  "protocol": "uhp",
  "versions": ["2026-09-12", "2026-08-11"],
  "default_version": "2026-09-12",
  "conformance_class": "full",
  "capabilities": {
    "streaming": true,
    "sessions": true,
    "cancellation": true,
    "files_input": true,
    "files_output": true,
    "session_listing": true,
    "harness_management": true,
    "session_sharing": true,
    "idempotency": true,
    "plugins": true
  },
  "plugin_schemas": ["https://agent-plugins.org/schemas/1.0.0/plugin.schema.json"],
  "implementation": { "name": "HarnessRouter Community Edition", "version": "0.3.0" }
}
```

| 字段 | 类型 | 必需 | 含义 |
|---|---|---|---|
| `object` | string | 是 | 始终为 `uhp.discovery` |
| `protocol` | string | 是 | 始终为 `uhp`——让客户端区分 UHP 与仿冒者 |
| `versions` | string[] | 是 | 此服务器可提供的所有版本；必须非空 |
| `default_version` | string | 是 | 客户端未发送 `UHP-Version` 时使用；必须出现在 `versions` 中 |
| `conformance_class` | string | 是 | `core`、`extended` 或 `full` |
| `capabilities` | object | 是 | 具名布尔值；见下文 |
| `plugin_schemas` | string[] | 当 `plugins` 为 true 时 | 服务器安装的 Agent Plugins 清单 Schema；见 [Plugins §7](plugins.md#7-discovery) |
| `implementation` | object | 否 | 自由格式的标识信息，用于调试和缺陷报告 |

服务器对未实现的能力必须报告 `false`，而非省略，这样客户端才能区分"不支持"与"服务器早于该字段"。客户端必须将缺失的能力键视为 `false`。

`conformance_class` 必须与 `capabilities` 一致：声称 `extended` 的服务器必须将 `files_input`、`files_output` 和 `session_listing` 报告为 `true`。一致性测试套件会检查这一点，因为与能力列表相矛盾的类别声明会向客户端传达两种不同的信息。

## 3. 任务生命周期

任务经历以下状态。响应对象上的 `status` 携带当前状态。

```
                    ┌──────────────┐
   POST /responses  │              │
   ────────────────▶│ in_progress  │
                    │              │
                    └──────┬───────┘
                           │
        ┌──────────────────┼──────────────────┬───────────────────┐
        ▼                  ▼                  ▼                   ▼
   ┌──────────┐      ┌──────────┐      ┌────────────┐      ┌───────────┐
   │completed │      │  failed  │      │ incomplete │      │ cancelled │
   └──────────┘      └──────────┘      └────────────┘      └───────────┘
```

| 状态 | 含义 | 终态 |
|---|---|---|
| `in_progress` | 已接受并正在运行 | 否 |
| `completed` | Harness 完成工作并产出结果 | 是 |
| `failed` | 任务无法完成；`error` 说明原因 | 是 |
| `incomplete` | Harness 因预算——步数上限或时间上限——而停止，输出不完整 | 是 |
| `cancelled` | 客户端取消了它；可能存在部分输出 | 是 |

规则：

- 服务器不得从终态转出。一旦客户端看到 `completed`，之后对该响应的读取必须返回 `completed`。
- 当预算导致工作中止时必须使用 `incomplete`，且不得用于错误。这一区别对客户端很重要：`incomplete` 通常值得继续，`failed` 通常不值得。
- `cancelled` 的任务必须报告 `cancelled`，而非 `failed`。请求停止的客户端并未遭遇错误。
- 终态响应必须保留其变为终态之前产生的所有输出。因任务后来失败而丢弃部分工作，会毁掉查明问题所在的唯一证据。

## 4. 会话生命周期

```
首个任务 ──▶ 隐式创建会话 ──▶ 会话活跃 ──▶ 删除（客户端）或过期（服务器策略）
                                        ▲    │
                                        └────┘
                              每个后续任务都会延长它
```

- 当链中首个任务运行时，服务器创建会话。其 id 必须在响应的 `metadata.session_id` 中报告。
- 会话必须在链中各任务之间保留：对话上下文、工作目录及其文件，以及所配置的 harness。
- 会话不得被指定了不同已配置 harness 的任务延长。服务器必须以 `409` 和 `code: "harness_mismatch"` 使此类请求失败，而非静默启动新会话——用不同的 agent 继续对话就是另一场对话，静默进行会丢失客户端以为仍持有的工作。
- 会话过期由服务器策略决定。会使会话过期的服务器在续接时必须报告 `404` 和 `code: "session_expired"`，以便与 `session_not_found` 区分。

## 5. 并发

- 服务器必须接受*不同*会话中的并发任务。
- 服务器不得在*同一*会话中并发运行两个任务：一个会话只有一个工作目录和一场对话，两个 agent 同时写入两者不是一种已定义的状态。对繁忙会话的第二个任务必须以 `409` 和 `code: "session_busy"` 失败。
- 收到 `session_busy` 的客户端应等待进行中的任务到达终态后重试。服务器在能估算时应在错误 `detail` 中包含 `retry_after_ms`，不能估算时必须省略，而非猜测。