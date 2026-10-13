# 实现客户端

UHP [客户端](versions/2026-09-12/architecture.md#1-roles)发现服务器运行的 harness、提交任务、接收进度事件并取回结果文件。它只通过 HTTP 使用 UHP，无需知道服务器背后运行哪个 harness。本指南讲流程，[架构章节](versions/2026-09-12/architecture.md#1-roles)讲定义。

另一个角色——在契约背后运行 harness 的服务器——见[实现服务器](SERVING.md)。

## 客户端做什么

客户端是产品、CLI、CI 任务或另一个想完成工作的 agent。它不运行 harness，而是驱动运行 harness 的服务器。客户端、服务器和 harness 的完整定义见[架构章节](versions/2026-09-12/architecture.md#1-roles)——本指南讲流程，不讲定义。

集成刻意保持轻量。任务接口形似 responses API，因此大部分工作就是普通 HTTP 加读取事件流。

## 生成客户端

不要手写类型。每个版本都在[Schema 章节](versions/2026-09-12/schema.md)发布 OpenAPI 3.1 和 JSON Schema 2020-12——据此为你的语言生成带类型的客户端，请求和响应结构就免费到手了。

## 集成步骤

1. **发现目录。** 询问服务器运行哪些 harness、各自接受哪些模型。客户端按服务器分配的 id 选择*已配置 harness*——基础 harness 加其配置。[Harness 章节](versions/2026-09-12/harnesses.md)。
2. **提交任务。** 一个工作单元：输入进，结果出。指定已配置 harness 并发送请求。[任务章节](versions/2026-09-12/tasks.md)。
3. **消费流。** 读取事件流以跟踪任务运行进度；连接断开时重连——协议定义了如何在不丢事件的情况下恢复。[流式章节](versions/2026-09-12/streaming.md)。
4. **在会话中继续。** 要从任务中断处接着做，就延续会话，而不是冷启动。[会话章节](versions/2026-09-12/sessions.md)。
5. **取走构建产物。** 下载运行产生的文件。[文件章节](versions/2026-09-12/files.md)。
6. **取消与处理错误。** 干净地停止工作，并从文档化的错误分类中读取失败原因，而不是靠状态码猜测。[错误章节](versions/2026-09-12/errors.md)。

## 最小端到端形态

提交一个任务并读取其结果的客户端只涉及核心接口：

    POST /v1/tasks            → 针对已配置 harness 提交
    GET  /v1/tasks/{id}/events → 流式跟踪进度直至完成
    GET  /v1/tasks/{id}        → 读取结果
    GET  /v1/files/{id}        → 下载构建产物

从 [Schema](versions/2026-09-12/schema.md) 生成确切的请求和响应体；[一致性测试套件](conformance/README.md)展示了同样的调用在真实服务器上运行。

## 下一步

服务器角色见[实现服务器](SERVING.md)。规范未回答的问题请到 [GitHub discussions](https://github.com/HarnessRouter/harnessrouter/discussions)。