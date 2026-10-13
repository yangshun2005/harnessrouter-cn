# 统一 Harness 协议 — 规范 `2026-08-11`

将完整 agent harness 作为共享基础设施运行的一项开放标准。

本文件是规范性规范。关于 UHP 是什么以及为何存在，请从[协议 README](../../README.md) 开始阅读。

## 章节

| # | 章节 | 定义内容 |
|---|---|---|
| 1 | [架构](architecture.md) | 角色、一致性类别、对象模型、传输、认证、设计原则 |
| 2 | [生命周期](lifecycle.md) | 版本协商、能力发现、任务与会话状态、并发 |
| 3 | [Harness](harnesses.md) | 发现、选择、配置和管理 harness；模型与可用性 |
| 4 | [任务](tasks.md) | 运行工作、响应对象、模型替换、幂等性 |
| 5 | [流式传输](streaming.md) | 事件词汇表、顺序保证、重连 |
| 6 | [会话](sessions.md) | 继续、检查、取消、共享、删除 |
| 7 | [文件](files.md) | 文件输入、构建产物、下载、保留与作用域 |
| 8 | [错误](errors.md) | 错误封装、错误码、重试规则、超时 |
| 9 | [安全](security.md) | 凭据、对象作用域、构建产物、注入、资源耗尽、错误卫生 |
| 10 | [Schema](schema.md) | 机器可读定义及如何从中生成 |

## 端点摘要

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
| `POST` | `/v1/sessions/{id}/share` | Full | [会话](sessions.md) |
| `GET` | `/v1/sessions/{id}/share` | Full | [会话](sessions.md) |
| `DELETE` | `/v1/sessions/{id}` | Full | [会话](sessions.md) |
| `DELETE` | `/v1/traces/{id}` | Full | [会话](sessions.md)（旧路径，同一操作） |

## 一致性

服务器通过某一类别的一致性测试套件，即在该类别上符合规范。除此之外都不算一致性声明——自我评估不算，实现了端点不算，通过"大部分"测试也不算。

```bash
pip install -e protocol/conformance
uhp-conformance --base-url https://your-server --api-key "$KEY" --class extended
```

该测试套件是本规范的一部分。如果套件与本文描述不一致，说明其中一方有 bug，必须修改错误的一方来解决——绝不能放任不一致。

## 约定

- MUST / SHOULD / MAY 的含义遵循 [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) 和 [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174)。
- JSON 示例仅作说明；结构以 [schema](../../schema/) 为准。
- 任务接口上的字段名为 `snake_case`（继承自 Responses 兼容结构），harness 对象上为 `camelCase`。这一不一致确实存在，在此明确指出而非隐藏，保留它是因为改动任何一方都会为了表面美观而破坏现有客户端。未来的主版本 SHOULD 统一二者。