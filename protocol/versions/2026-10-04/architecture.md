# 架构

**统一 Harness 协议，版本 `2026-10-04`**

本规范中的关键词 MUST、MUST NOT、REQUIRED、SHALL、SHALL NOT、SHOULD、SHOULD NOT、RECOMMENDED、MAY 和 OPTIONAL，仅在以全大写形式出现时，按 [RFC 2119][rfc2119] 和 [RFC 8174][rfc8174] 的定义解释。

[rfc2119]: https://www.rfc-editor.org/rfc/rfc2119
[rfc8174]: https://www.rfc-editor.org/rfc/rfc8174

## 1. 角色

```
┌──────────┐   UHP over HTTP   ┌──────────┐   implementation-defined   ┌──────────┐
│  Client  │ ────────────────▶ │  Server  │ ─────────────────────────▶ │ Harness  │
└──────────┘                   └──────────┘                            └──────────┘
```

**客户端（Client）**——想要完成工作的应用：产品后端、CLI、CI 任务、另一个 agent。客户端只说 UHP。它 MUST NOT 被要求知道服务器背后运行的是哪个 harness 实现，也不知道服务器如何执行它。

**服务器（Server）**——实现本规范。它接受任务，驱动一个或多个 harness，并用此处定义的词汇报告进度和结果。关于它*如何*运行 harness 的一切——容器、子进程、队列、远程 worker——都不在本规范范围内，且 MUST NOT 泄漏到线格式中。

**Harness**——一个完整的 agent 运行时，有自己的循环、工具和会话状态。UHP 不规定 harness 本身；它规定客户端如何通过服务器驱动 harness。Harness 由 **base** 标识：一个稳定的字符串，如 `codex`、`claude-code` 或 `hermes`。

**已配置的 harness（configured harness）**是 base 加配置——默认模型、系统提示、工具限制、技能、MCP 服务器、步数和时间预算。它是客户端发送工作时选择的单位，由服务器分配的 id 寻址。

> **为什么是已配置的 harness，而不只是 base？**
> 同一个 base 在不同系统提示、不同模型或不同工具集下行为差异很大。把配置变成一等、可寻址的对象，意味着产品可以改变其 agent 的行为而无需重新部署后端，也意味着两个产品可以共享一个服务器而不共享行为。

## 2. 一致性类别

服务器声明它实现哪个类别。每个类别是累积的：**Extended** 包含 **Core**，**Full** 包含 **Extended**。

| 类别 | 此类别服务器 MUST 实现 |
|---|---|
| **Core** | 能力发现、harness 发现、非流式和流式任务执行、会话延续、取消，以及错误模型。 |
| **Extended** | Core，加上文件输入、构建产物检索，以及会话列表/检查。 |
| **Full** | Extended，加上 harness 生命周期管理（创建、更新、删除）和会话共享。 |

客户端 MUST NOT 在未检查能力发现（见[生命周期 §2](lifecycle.md)）的情况下假设任何高于 **Core** 的能力。服务器 MUST NOT 宣传它未实现的能力——宣传是承诺，信任它却收到 `404` 的客户端被骗了，且无法恢复。

[一致性测试套件](../../conformance/)分别测试每个类别并按类别报告。在某类别通过的服务器 MAY 自称"UHP <版本> 一致性（<类别>）"，版本以其发现文档作为 `default_version` 提供的为准，测试套件会报告该版本。"一致性"一词的任何其他用法都无意义。

## 3. 对象模型

七种对象类型。每一种都带一个 `object` 字段标明其类型，以及一个带类型区分前缀的 `id`，使标识符指向什么从不含糊。

| 对象 | `object` 值 | Id 前缀 | 生命周期 |
|---|---|---|---|
| Harness | `harness` | `chrn_` | 直到删除 |
| Response | `response` | `resp_` | 按服务器策略保留 |
| Session | `session` | `hsess` | 直到删除 |
| File | `file` | `file_` | 随其容器 |
| Container | `container` | `cntr_` | 随其会话 |
| Environment | `environment` | `henv_` | 直到删除；可选，见[环境](environments.md) |
| Memory | `memory` | `hmem_`，或服务器映射的树的 id | 直到删除；可选，见[记忆](memories.md) |
| Event | *（无——事件带 `type`）* | — | 流式传输，可重放 |

它们的关系：

```
Harness ──┐
          ├──▶ Session ──┬──▶ Response ──▶ Response ──▶ …    (one per task, chained)
Model  ───┘              └──▶ Container ──▶ File, File, …    (artifacts)
```

- **response** 是一个任务：一个请求进，一个结果出。它是工作的原子单位。
- **session** 是一串共享对话上下文和工作目录的 response。会话由第一个任务隐式创建，通过 `previous_response_id` 延续。
- **container** 是会话的文件命名空间。agent 写入的文件成为其中的构建产物。
- **environment** 是项目的文件和已安装依赖，构建一次后以只读方式挂载到每个指定它的会话中；harness 指定一个作为其任务的默认环境。它从不属于容器：构建产物来自工作目录，环境是工作目录读取的内容（[环境](environments.md)）。
- **memory** 是比每个会话都持久的地方：树中的一个节点，持有记录，访问权限按节点授予并向下继承。agent 被授予记忆如同人被授予一样；记忆从不属于会话或容器（[记忆](memories.md)）。

> **为什么会话是隐式的？**
> 要求在做任何工作前先 `POST /sessions` 会增加一次往返、一个失败模式和一个要清理的对象——而大多数首个任务从不需要这个概念。只发送一次性任务的客户端永远学不到"会话"这个词；需要连续性的客户端通过引用它已有的 id 来获得。

## 4. 传输

- 所有请求 MUST 是 HTTP/1.1 或更高版本，且基于 TLS，回环开发除外，此时服务器 MAY 接受明文。
- 请求和响应体是 `application/json; charset=utf-8`，文件上传（`multipart/form-data`）和文件下载（文件自身的媒体类型）除外。
- 流式传输使用 Server-Sent Events（`text/event-stream`），如[流式传输](streaming.md)所述。
- 服务器 MUST 接受缺少 `Accept` 头的请求，并默认使用 JSON。

## 5. 认证

客户端用 bearer token 认证：

```http
Authorization: Bearer <token>
```

token 如何签发不在范围内。服务器 MUST 以 `401` 和[错误](errors.md)中的错误模型拒绝缺失、格式错误或未知的 token，且 MUST NOT 以让调用者能枚举有效 token 的方式区分"无此 token"和"token 在此不被允许"。

服务器 MUST 将每个对象限定在创建它的主体范围内。客户端 MUST NOT 能读取、延续、取消或删除另一主体的 response、session 或 file——包括通过猜测 id。对于调用者范围之外的对象，服务器 MUST 返回 `404` 而非 `403`，以免泄露 id 是否存在。

## 6. 设计原则

这些是规范对自身的要求。列入是因为它们解释了后续内容的形态，也因为未来违反其中一条的变更很可能是错误。

1. **客户端不应能分辨是哪个 harness 运行了工作**——除非主动询问。客户端能观察到的 harness 之间的每个差异都是泄漏，每个泄漏都是替换 harness 会破坏产品的地方。
2. **进度是事实的流，不是渲染。** 事件描述发生了什么——文本、工具调用、文件——从不描述如何显示。
3. **一个字段只表示一件事。** UHP 扩展借用表面之处，它添加字段；从不重定义一个。
4. **缺失不是空，空不是零。** 服务器不知道某个值就省略它。它 MUST NOT 编造占位符、估计值或零。
5. **失败是一等结果。** 每个失败都能在与成功相同的信封中报告，带机器可读的代码。客户端永远不必解析散文来找出该做什么。
6. **规范、参考实现和一致性测试套件同步演进。** 此处没有测试执行的句子是愿望，不是标准。