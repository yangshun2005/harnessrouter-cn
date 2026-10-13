# 统一 Harness 协议 — 规范 `2026-10-04`

将完整 agent harness 作为共享基础设施运行的一项开放标准。

本文件是规范性规范。关于 UHP 是什么以及为何存在，请从[协议 README](../../README.md) 开始阅读。

## 章节

| # | 章节 | 定义内容 |
|---|---|---|
| 1 | [架构](architecture.md) | 角色、一致性类别、对象模型、传输、认证、设计原则 |
| 2 | [生命周期](lifecycle.md) | 版本协商、能力发现、任务与会话状态、并发 |
| 3 | [Harness](harnesses.md) | 发现、选择、配置和管理 harness；模型与可用性 |
| 4 | [插件](plugins.md) | 以 Agent Plugins 格式打包的工具与技能，安装到 harness 中；组合；导出 |
| 5 | [环境](environments.md) | 项目的文件与已安装依赖，构建一次，在引用它的每个会话中只读；构建、版本、挂载 |
| 6 | [记忆](memories.md) | 比会话更持久的记忆：按节点授予访问权限的记忆树、记录、召回、提供方，以及 harness 何时访问它们 |
| 7 | [任务](tasks.md) | 运行工作、响应对象、模型替换、幂等性 |
| 8 | [流式传输](streaming.md) | 事件词汇表、顺序保证、重连 |
| 9 | [会话](sessions.md) | 继续、检查、取消、共享、删除 |
| 10 | [文件](files.md) | 文件输入、构建产物、下载、保留与作用域 |
| 11 | [错误](errors.md) | 错误封装、错误码、重试规则、超时 |
| 12 | [安全](security.md) | 凭据、对象作用域、构建产物、注入、耗尽、插件、环境、记忆、错误卫生 |
| 13 | [Schema](schema.md) | 机器可读定义及如何从中生成 |

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
| `GET` | `/v1/harnesses/{id}/skills/{name}/files` | Full | [Harness](harnesses.md) |
| `GET` | `/v1/harnesses/{id}/plugins/{name}/files` | Full, `plugins` | [插件](plugins.md) |
| `GET` | `/v1/harnesses/{id}/plugin` | Full, `plugins` | [插件](plugins.md) |
| `POST` | `/v1/sessions/{id}/share` | Full | [会话](sessions.md) |
| `GET` | `/v1/sessions/{id}/share` | Full | [会话](sessions.md) |
| `DELETE` | `/v1/sessions/{id}` | Full | [会话](sessions.md) |
| `DELETE` | `/v1/traces/{id}` | Full | [会话](sessions.md)（旧路径，同一操作） |
| `GET` | `/v1/environments` | Full, 能力 `environments` | [环境](environments.md) |
| `POST` | `/v1/environments` | Full, 能力 `environments` | [环境](environments.md) |
| `GET` `PUT` `DELETE` | `/v1/environments/{id}` | Full, 能力 `environments` | [环境](environments.md) |
| `GET` | `/v1/environments/{id}/files` | Full, 能力 `environments` | [环境](environments.md) |
| `GET` `PUT` `DELETE` | `/v1/environments/{id}/files/{path}` | Full, 能力 `environments` | [环境](environments.md) |
| `POST` | `/v1/environments/{id}/directories` | Full, 能力 `environments` | [环境](environments.md) |
| `POST` | `/v1/environments/{id}/import` | Full, 能力 `environments` | [环境](environments.md) |
| `GET` | `/v1/environments/runtimes` | Full, 能力 `environments` | [环境](environments.md) |
| `POST` | `/v1/environments/{id}/build` | Full, 能力 `environments` | [环境](environments.md) |
| `GET` | `/v1/environments/{id}/builds/{version}` | Full, 能力 `environments` | [环境](environments.md) |
| `GET` | `/v1/environments/{id}/versions` | Full, 能力 `environments` | [环境](environments.md) |
| `POST` | `/v1/environments/{id}/versions/{version}/activate` | Full, 能力 `environments` | [环境](environments.md) |
| `GET` | `/v1/environments/{id}/harnesses` | Full, 能力 `environments` | [环境](environments.md) |
| `GET` `POST` | `/v1/memories` | Full, 能力 `memories` | [记忆](memories.md) |
| `GET` `PUT` `DELETE` | `/v1/memories/{id}` | Full, 能力 `memories` | [记忆](memories.md) |
| `GET` `POST` | `/v1/memories/{id}/grants` | Full, 能力 `memories` | [记忆](memories.md) |
| `DELETE` | `/v1/memories/{id}/grants/{grant_id}` | Full, 能力 `memories` | [记忆](memories.md) |
| `POST` | `/v1/memories/{id}/observe` | Full, 能力 `memories` | [记忆](memories.md) |
| `GET` | `/v1/memories/{id}/jobs/{job}` | Full, 能力 `memories` | [记忆](memories.md) |
| `GET` `POST` | `/v1/memories/{id}/records` | Full, 能力 `memories` | [记忆](memories.md) |
| `GET` `PATCH` `DELETE` | `/v1/memories/{id}/records/{rid}` | Full, 能力 `memories` | [记忆](memories.md) |
| `GET` | `/v1/memories/{id}/records/{rid}/history` | Full, 能力 `memories` | [记忆](memories.md) |
| `GET` | `/v1/memories/{id}/records/{rid}/content/{index}` | Full, 能力 `memories` | [记忆](memories.md) |
| `POST` | `/v1/memories/{id}/records/{rid}/operations/{name}` | Full, 能力 `memories` | [记忆](memories.md) |
| `POST` | `/v1/memories/{id}/recall` | Full, 能力 `memories` | [记忆](memories.md) |
| `POST` | `/v1/memories/{id}/graph` | Full, 能力 `memories` | [记忆](memories.md) |
| `POST` | `/v1/memories/{id}/erase` | Full, 能力 `memories` | [记忆](memories.md) |
| `GET` | `/v1/memories/{id}/queries` | Full, 能力 `memories` | [记忆](memories.md) |
| `PUT` `POST` | `/v1/memories/{id}/queries/{name}` | Full, 能力 `memories` | [记忆](memories.md) |
| `POST` | `/v1/memories/{id}/query` | Full, 能力 `memories` | [记忆](memories.md) |
| `GET` `POST` | `/v1/memories/{id}/snapshots` | Full, 能力 `memories` | [记忆](memories.md) |
| `GET` `POST` | `/v1/memories/{id}/consolidations` | Full, 能力 `memories` | [记忆](memories.md) |
| `GET` | `/v1/memories/{id}/consolidations/{run}` | Full, 能力 `memories` | [记忆](memories.md) |
| `GET` | `/v1/memories/{id}/consolidations/{run}/changes` | Full, 能力 `memories` | [记忆](memories.md) |
| `POST` | `/v1/memories/{id}/consolidations/{run}/revert` | Full, 能力 `memories` | [记忆](memories.md) |
| `GET` | `/v1/memories/providers` | Full, 能力 `memories` | [记忆](memories.md) |
| `GET` | `/v1/memories/types` | Full, 能力 `memories` | [记忆](memories.md) |
| `GET` `PUT` | `/v1/harnesses/{id}/memories` | Full, 能力 `memories` | [记忆](memories.md) |

`plugins` 列中的条目仅由发现文档报告 `plugins` 能力的服务器提供；该能力在每个类别中都是可选的。

`memories` 条目仅由报告 `memories` 能力的服务器提供，该能力在每个类别中同样可选；在该能力内，命名查询与自由查询、类型、记录可携带的媒体、快照、整合运行与擦除均按提供方声明（[记忆 §10.2](memories.md#102-the-capability-document)）。

## 本版本的变化

本版本是对 `2026-08-11` 的增量补充：当时有效的每个请求和对象现在依然有效，针对上一版本编写的客户端在提供本版本的服务器上仍可正常工作。本版本新增了[插件](plugins.md)章节，包含插件内的 `stdio` MCP 服务器、`plugins` 能力以及五个错误码。服务器可以用同一代码路径同时提供两个版本。[变更日志](../../CHANGELOG.md)列出了完整清单。

## 一致性

当服务器在某个类别通过[一致性测试套件](../../conformance/)时，即在该类别上符合规范。除此之外都不算一致性声明——自我评估不算，实现了端点不算，通过"大多数"测试也不算。

```bash
pip install -e protocol/conformance
uhp-conformance --base-url https://your-server --api-key "$KEY" --class extended
```

该测试套件是本规范的一部分。若套件与本说明文字不一致，则其中一方存在缺陷，必须通过修改错误的一方来解决——绝不能放任其不一致。

## 约定

- MUST / SHOULD / MAY 按 [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) 和 [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174) 的定义使用。
- JSON 示例仅作说明；[schema](../../schema/) 是结构的权威来源。
- 字段名在任务接口上为 `snake_case`（继承自 Responses 兼容结构），在 harness 对象上为 `camelCase`。这一不一致确实存在，在此明确指出而非隐藏，并予以保留，因为修改任一方都会破坏现有客户端，而收益仅是外观上的。未来的主版本 SHOULD 统一二者。