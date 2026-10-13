# 实现一个服务器

如何将 agent harness 接入 Unified Harness Protocol —— 服务器必须响应什么、按什么顺序构建、如何验证其可用。本指南是[规范](versions/2026-09-12/index.md)的实用配套文档；两者不一致时，以规范为准。

## 什么是 UHP 服务器

UHP 服务器是任何 HTTP 服务器，只要它用规范中的响应来应答规范中的请求即可。它可以在容器、子进程、队列或他人的基础设施上运行 agent。线格式不要求托管服务、账号、许可证密钥或回连。

服务器有两种形态，契约完全相同：

- **runner** 将现有 harness —— Codex、Claude Code、Hermes —— 接入契约，并将其作为目录对外公布。
- **原生 harness** 直接使用 UHP：单个 harness，直接提供服务，中间没有适配器。如果你维护一个 harness，这种形态能让所有 UHP 客户端一次性都成为你的客户端。

## 契约，按构建顺序

按一致性测试套件测试的顺序构建，因为每个阶段都可独立测试：

1. **发现** —— 公布你运行的 harness 及其接受的模型。[Harnesses 章节](versions/2026-09-12/harnesses.md)。
2. **任务** —— 接受一个工作单元，运行它，返回结果。任务接口刻意设计成 responses API 的形态，再加上 harness 需要而模型端点没有概念的部分。[Tasks 章节](versions/2026-09-12/tasks.md)。
3. **流式传输** —— 长任务运行时报告进度，并在连接断开后存活。[Streaming 章节](versions/2026-09-12/streaming.md)。
4. **会话** —— 让任务从上一个任务中断处继续。[Sessions 章节](versions/2026-09-12/sessions.md)。
5. **文件** —— 接收文件，交回构建产物。[Files 章节](versions/2026-09-12/files.md)。
6. **取消与错误** —— 干净地停止运行中的工作，并按文档化的分类体系报错。[Errors 章节](versions/2026-09-12/errors.md)。

[Schema 章节](versions/2026-09-12/schema.md)为每个版本发布 OpenAPI 3.1 和 JSON Schema 2020-12 —— 生成你的类型，而不是手写。

## 验证

从第一天起就对服务器运行一致性测试套件；它被设计为开发工具，而非期末考试：

    pip install -e protocol/conformance
    uhp-conformance --base-url https://your-server --api-key "$KEY" --class core

类别是累积的：`core` 是每个服务器都必须应答的契约，`extended` 和 `full` 在其之上增加接口。`--only` 可在修复时重跑单个失败的检查，每个失败都会引用它所违反的规范条文。套件运行真实的 agent 任务 —— 这是刻意的，因为值得捕获的缺陷只有在真实工作运行时才会出现。

## 上架

当套件在任一类别通过后，通过 Pull Request 将你的服务器添加到[示例](IMPLEMENTATIONS.md)页面，并附上报告 JSON 作为证据。上架不是认证，页面也如此说明 —— 但一行带有可复现报告的记录，是每个生态条目的起点。

规范未回答的问题请发到 [GitHub discussions](https://github.com/HarnessRouter/harnessrouter/discussions)；你需要的变更请提交到 [governance](GOVERNANCE.md)，先写文字提案。