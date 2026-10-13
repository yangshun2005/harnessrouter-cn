# 统一 Harness 协议 — 规范 `2026-09-28`

将完整 Agent Harness 作为共享基础设施运行的一项开放标准。

本文件是规范性规范。若想了解 UHP 是什么以及为何存在，请从[协议 README](../../README.md)开始。

## 章节

| # | 章节 | 定义内容 |
|---|---|---|
| 1 | [架构](architecture.md) | 角色、一致性类别、对象模型、传输、认证、设计原则 |
| 2 | [生命周期](lifecycle.md) | 版本协商、能力发现、任务与会话状态、并发 |
| 3 | [Harness](harnesses.md) | 发现、选择、配置和管理 Harness；模型与可用性 |
| 4 | [插件](plugins.md) | 工具与技能包，采用 Agent Plugins 格式，安装到 Harness 中；组合；导出 |
| 5 | [环境](environments.md) | 项目的文件与已安装依赖，构建一次，在每个引用它的会话中只读；构建、版本、挂载 |
| 6 | [任务](tasks.md) | 运行工作、响应对象、模型替换、幂等性 |
| 7 | [流式传输](streaming.md) | 事件词汇表、顺序保证、重连 |
| 8 | [会话](sessions.md) | 继续、检查、取消、共享、删除 |
| 9 | [文件](files.md) | 文件输入、构建产物、下载、保留与作用域 |
| 10 | [错误](errors.md) | 错误封装、错误码、重试规则、超时 |
| 11 | [安全](security.md) | 凭据、对象作用域、构建产物、注入、资源耗尽、插件、环境、错误信息卫生 |
| 12 | [Schema](schema.md) | 机器可读定义及如何据此生成 |

## 端点汇总

| 方法 | 路径 | 类别 | 章节 |
|---|---|---|---|
| `GET` | `/v1/uhp` | Core | [生命周期](lifecycle.md) |
| `GET` | `/v1/harnesses` | Core | [Harness](harnesses.md) |
| `GET` | `/v1/harnesses/{id}` | Core | [Harness](harnesses.md) |
| `GET` | `/v1/models` | Core | [Harness](harnesses.md) |
| `GET` | `/v1/harnesses/{id}/models` | Core | [Harness](harnesses.md) |
| `POST` | `/v1/responses` | Core | [任务](tasks.md) |
| `GET` | `/v1/responses/{id}` | Core | [任务](tasks.md) |
| `GET` | `/v1/responses/{id}/input_items` | Core | [任务](tasks.md) |
| `POST` | `/v1/responses/{id}/cancel` | Core | [会话](sessions.md) |
| `DELETE` | `/v1/responses/{id}` | Core | [任务](tasks.md) |
| `POST` | `/v1/sessions/{id}/cancel` | Core | [会话](sessions.md) |
| `GET` | `/v1/sessions` | Extended | [会话](sessions.md) |
| `GET` | `/v1/sessions/{id}` | Extended | [会话](sessions.md) |
| `GET` | `/v1/sessions/{id}/turns` | Extended | [会话](sessions.md) |
| `POST` | `/v1/files` | Extended | [文件](files.md) |
| `GET` | `/v1/sessions/{id}/files` | Extended | [文件](files.md) |
| `GET` | `/v1/sessions/{id}/files/archive` | Extended | [文件](files.md) |
| `GET` | `/v1/containers/{cid}/files/{fid}/content` | Extended | [文件](files.md) |
| `GET` | `/v1/containers/{cid}/files/{fid}/pdf` | Extended | [文件](files.md) |
| `POST` | `/v1/harnesses` | Full | [Harness](harnesses.md) |
| `PUT` | `/v1/harnesses/{id}` | Full | [Harness](harnesses.md) |
| `DELETE` | `/v1/harnesses/{id}` | Full | [Harness](harnesses.md) |
| `GET` | `/v1/harnesses/{id}/skills/{name}/files` | Full | [Harness](harnesses.md) |
| `GET` | `/v1/harnesses/{id}/plugins/{name}/files` | Full, `plugins` | [插件](plugins.md) |
| `GET` | `/v1/harnesses/{id}/plugin` | Full, `plugins` | [插件](plugins.md) |
| `POST` | `/v1/sessions/{id}/share` | Full | [会话](sessions.md) |
| `GET` | `/v1/sessions/{id}/share` | Full | [会话](sessions.md) |
| `DELETE` | `/v1/sessions/{id}` | Full | [会话](sessions.md) |
| `DELETE` | `/v1/traces/{id}` | Full | [会话](sessions.md)（旧路径，同一操作） |
| `GET` | `/v1/environments` | Full, capability `environments` | [环境](environments.md) |
| `POST` | `/v1/environments` | Full, capability `environments` | [环境](environments.md) |
| `GET` `PUT` `DELETE` | `/v1/environments/{id}` | Full, capability `environments` | [环境](environments.md) |
| `GET` | `/v1/environments/{id}/files` | Full, capability `environments` | [环境](environments.md) |
| `GET` `PUT` `DELETE` | `/v1/environments/{id}/files/{path}` | Full, capability `environments` | [环境](environments.md) |
| `POST` | `/v1/environments/{id}/directories` | Full, capability `environments` | [环境](environments.md) |
| `POST` | `/v1/environments/{id}/import` | Full, capability `environments` | [环境](environments.md) |
| `GET` | `/v1/environments/runtimes` | Full, capability `environments` | [环境](environments.md) |
| `POST` | `/v1/environments/{id}/build` | Full, capability `environments` | [环境](environments.md) |
| `GET` | `/v1/environments/{id}/builds/{version}` | Full, capability `environments` | [环境](environments.md) |
| `GET` | `/v1/environments/{id}/versions` | Full, capability `environments` | [环境](environments.md) |
| `POST` | `/v1/environments/{id}/versions/{version}/activate` | Full, capability `environments` | [环境](environments.md) |
| `GET` | `/v1/environments/{id}/harnesses` | Full, capability `environments` | [环境](environments.md) |

`plugins` 列中的端点仅由发现文档中报告 `plugins` 能力的服务器提供；该能力在任何类别下都是可选的。

## 本版本的变化

本版本是对 `2026-08-11` 的增量更新：当时有效的每个请求和对象现在依然有效，针对上一版本编写的客户端在提供本版本的服务器上仍可正常工作。本版本新增了[插件](plugins.md)章节，包含插件内的 `stdio` MCP 服务器、`plugins` 能力，以及五个错误码。服务器可以用同一套代码路径同时提供两个版本。完整列表见[变更日志](../../CHANGELOG.md)。

## 一致性

服务器在某个类别下通过该类别的一致性测试套件时，即在该类别下具备一致性。除此之外任何东西都不构成一致性声明——自我评估不算，实现了端点不算，通过"大多数"测试也不算。

```bash
pip install -e protocol/conformance
uhp-conformance --base-url https://your-server --api-key "$KEY" --class extended
```

测试套件是本规范的一部分。如果测试套件与本说明文字不一致，说明其中一方有 bug，必须通过修改错误的一方来解决——绝不能放任二者不一致。

## 约定

- MUST / SHOULD / MAY 的含义见 [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) 和 [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174)。
- JSON 示例仅作说明；结构以 [schema](../../schema/) 为准。
- 任务接口上的字段名为 `snake_case`（继承自 Responses 兼容的结构），而 harness 对象上为 `camelCase`。这一不一致是真实存在的，在此明确指出而非隐藏，保留它是因为改动任一方都会为了表面上的收益而破坏现有客户端。未来的主版本 SHOULD 将其统一。