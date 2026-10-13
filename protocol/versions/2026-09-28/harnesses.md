# Harness

**统一 Harness 协议，版本 `2026-09-28`**

客户端在发送任务前，需要知道哪些 Harness 能运行任务、以及能用哪些模型。本章定义发现机制、Harness 对象，以及在一致性等级 **Full** 下如何创建和修改 Harness。

## 1. 发现 Harness

```http
GET /v1/harnesses
```

```json
{
  "harnesses": [
    {
      "id": "chrn_08dae611630d467ab3e67ed792570ae5",
      "object": "harness",
      "name": "Research agent",
      "base": "claude-code",
      "baseLabel": "Claude Code",
      "defaultModel": "claude-sonnet-4.6",
      "systemPrompt": "",
      "mcpServers": [],
      "skills": [],
      "plugins": [],
      "disabledTools": [],
      "maxStep": null,
      "timeoutSeconds": null,
      "createdAt": 1786403298205
    }
  ]
}
```

服务端 MUST 只返回调用方作用域内的 Harness。列表 MAY 为空——没有配置任何 Harness 的服务端是合法的，客户端 MUST 处理这种情况，不能假定索引 `0` 一定存在。

```http
GET /v1/harnesses/{harness_id}
```

返回单个 Harness 对象，或返回 `404` 及 `code: "harness_not_found"`。

## 2. Harness 对象

| 字段 | 类型 | 必填 | 含义 |
|---|---|---|---|
| `id` | string | 是 | 以 `chrn_` 为前缀 |
| `object` | string | 是 | 始终为 `harness` |
| `name` | string | 是 | 人类可读的标签，不是标识符 |
| `base` | string | 是 | Harness 运行时类型：`codex`、`claude-code`、`hermes` 等 |
| `baseLabel` | string | 否 | `base` 的显示名称 |
| `defaultModel` | string | 否 | 任务未指定 `model` 时使用的模型 |
| `systemPrompt` | string | 否 | 附加的常驻指令 |
| `mcpServers` | array | 否 | 挂载到该 Harness 的 MCP 服务器 |
| `skills` | array | 否 | 该 Harness 可用的技能 |
| `plugins` | array | 否 | 已安装的插件，每个插件是一组工具和技能；见 [Plugins](plugins.md) |
| `environment` | string | 否 | 该 Harness 的任务读取的环境，以只读方式挂载；需要 `environments` 能力。见 [Environments](environments.md) |
| `disabledTools` | string[] | 否 | 对智能体禁用的工具 |
| `maxStep` | integer \| null | 否 | 默认步数预算 |
| `timeoutSeconds` | integer \| null | 否 | 默认墙钟时间预算 |
| `createdAt` | integer | 是 | Unix 毫秒时间戳 |

本规范不枚举 `base` 的取值。服务端 MAY 支持本文档从未听说过的 base，客户端 MUST 将 `base` 视为不透明字符串——否则每发布一个 Harness 就得修订协议，而这正是 UHP 要消除的耦合。

## 3. 模型

```http
GET /v1/models
```

```json
{
  "backends": {
    "claude": {
      "default": "claude-sonnet-4.6",
      "models": [
        { "id": "claude-sonnet-4.6", "label": "claude-sonnet-4.6", "backend": "claude",
          "available": true, "default": true }
      ]
    }
  }
}
```

```http
GET /v1/harnesses/{harness_id}/models
```

```json
{
  "harness_id": "chrn_…",
  "backend": "claude",
  "default": "claude-sonnet-4.6",
  "fallback": "claude-sonnet-4.6",
  "models": [ { "id": "claude-opus-5", "available": true, "default": false } ]
}
```

### 3.1 `available` 是一种承诺

`available: true` 表示服务端**此刻**能为该 Harness 提供该模型——凭据存在，且提供商可访问。`available: false` 表示对该模型的请求不会按所请求的方式运行。

服务端 MUST 计算 `available`，而不是直接断言。把模型列为可用、随后任务失败，对客户端来说是最糟的结果：它向用户展示了一个选项，用户选了它，工作却在用户已经投入之后失败。

客户端 SHOULD 将不可用的模型显示为禁用，而不是隐藏，这样用户能看到某个模型存在但未配置，而不是疑惑它为什么不见了。

## 4. 工具与技能

配置好的 Harness 携带三样东西，决定其智能体能做什么。三者都是 Harness 对象的一部分，设置方式与 Harness 本身相同。

### 4.1 MCP 服务器

```json
{ "mcpServers": [
    { "name": "vault", "url": "https://mcp.example.com/mcp", "transport": "http", "enabled": true }
] }
```

| 字段 | 必填 | 含义 |
|---|---|---|
| `name` | 是 | 向智能体标识该服务器；会被清理为 CLI 安全的标识符 |
| `url` | 是 | 端点 |
| `transport` | 否 | `http`（Streamable HTTP，默认）或 `sse` |
| `enabled` | 否 | 缺省或 `true` 表示启用 |
| `headers` | 否 | 额外的请求头 |
| `auth` | 否 | Bearer token，或由服务端解析的服务端引用 |

该列表只携带远程服务器，与上一版本完全一致。以进程方式运行的 MCP 服务器（`stdio`）在插件中声明，插件为它提供了运行根目录和保存状态的目录（[Plugins §2.2](plugins.md#22-what-the-server-derives)）。

服务端 MUST 在一轮中只连接**已启用**的条目。已禁用的条目 MUST NOT 被联系——不能先连接再隐藏，那样仍会向该端点的运营者泄露本轮的存在。

无法访问的 MCP 服务器 MUST NOT 导致任务失败。本轮 MUST 在没有这些工具的情况下继续，因为第三方端点宕机不是丢失用户工作的理由。服务端 SHOULD 让降级在运行中可见，而不是悄无声息。

> 服务端 MUST NOT 宣传自己无法兑现的 MCP 支持。如果其背后的 Harness 运行时无法使用所配置的传输方式，那就是安装损坏；只在工作区内的日志文件里报告，对用户而言与模型拒绝使用该工具无法区分。

### 4.2 技能

技能是一个**文件夹**，不是单个文件：

```json
{ "skills": [
    { "name": "vault-manual", "enabled": true, "files": [
        { "path": "SKILL.md",            "content": "---\nname: vault-manual\n---\n…" },
        { "path": "references/codes.md", "content": "…" },
        { "path": "scripts/helper.sh",   "content": "#!/bin/sh\n…" },
        { "path": "assets/logo.png",     "content_b64": "iVBORw0KGgo…" }
    ]}
] }
```

- `files[].path` 相对于技能自身的文件夹，MUST 支持嵌套目录。服务端 MUST 拒绝逃逸出该文件夹的路径。
- 文本放在 `content` 中，二进制放在 `content_b64` 中。服务端 MUST 逐字节保留两者。
- 一个包 MUST 包含 `SKILL.md`；服务端 MUST 在配置时而非运行时拒绝不含它的包。
- 服务端 MUST 将**整个文件夹**落到智能体可读取的位置。只落 `SKILL.md` 会破坏所有携带引用、脚本或数据的技能——而大多数非平凡技能都是如此。
- `enabled: false` 会抑制某个技能，包括从 base 继承来的技能。

服务端 MAY 将大型包离线存储，并且 MUST 通过以下接口返回完整文件列表：

```http
GET /v1/harnesses/{harness_id}/skills/{skill_id}/files
```

Harness 经 `GET` 和 `PUT` 往返 MUST NOT 丢失技能内容。这正是值得针对性设计的失败：一次无关的编辑——重命名 Harness——悄悄清空了一个技能文件夹，而用户直到几周后智能体行为异常才会发现它不见了。

### 4.3 禁用的工具

```json
{ "disabledTools": ["WebSearch"] }
```

名称来自 Harness base 自身的工具目录。执行方式因运行时而异，服务端 MUST NOT 夸大其词：

- 运行时支持按工具限制时，服务端 MUST 作为硬性阻断执行。
- 不支持时，服务端 MUST 仍将限制传达给智能体——作为常驻指令——且 MUST NOT 悄悄丢弃。丢弃是最糟的结果：运营者以为某工具已关闭，实际并没有。

需要保证阻断的客户端 SHOULD 确认 Harness base 支持阻断，而不是假定 `disabledTools` 总是硬性的。

### 4.4 插件

```json
{ "plugins": [ { "name": "contract-review", "enabled": true, "blob": "plg_…" } ] }
```

插件是一个 [Agent Plugins](https://agent-plugins.org/specification) 包：`plugin.json`、`mcp.json` 和 `skills/`，作为一个文件夹整体携带并整体安装。其服务器和技能在每一轮加入 Harness 自身的服务器和技能；Harness 自身的 `mcpServers` 和 `skills` 只报告写入其中的内容，绝不吸收插件的。对象、组合规则、将 Harness 导出为包，以及托管服务端对运行中插件的义务，见 [Plugins 章节](plugins.md)。能力名为 `plugins`；不具备该能力的服务端在所有等级下都合规。

## 5. 管理 Harness

仅限一致性等级 **Full**。客户端 MUST 先检查 `harness_management` 能力。

### 5.1 创建

```http
POST /v1/harnesses
```

```json
{ "name": "Research agent", "base": "claude-code", "default_model": "claude-sonnet-4.6" }
```

返回创建的 Harness 对象。`base` 为必填，且 MUST 是服务端支持的取值；否则返回 `422` 及 `code: "unsupported_base"`。

### 5.2 更新

```http
PUT /v1/harnesses/{harness_id}
```

替换可变配置。服务端 MUST NOT 更改 `id`、`base` 或 `createdAt`。更改现有 Harness 的 base 会悄悄改变所有已挂载会话的行为；想要不同 base 的客户端 MUST 创建另一个 Harness。

### 5.3 删除

```http
DELETE /v1/harnesses/{harness_id}
```

服务端 MUST NOT 删除使用过该 Harness 的会话或响应。配置变更后就消失的历史无法审计。

## 6. 选择 Harness

非规范性内容，但这是每个客户端都要面对的问题：

| 如果任务是…… | 考虑 |
|---|---|
| 在仓库中编辑代码 | 编码专用的 base（`codex`、`claude-code`） |
| 长周期、多工具的工作 | 具有明确步数预算和强工具使用能力的 base |
| 低成本、大批量分类 | 任意 base 上最小的模型 |

UHP 的意义在于让这一选择保持可逆。支持 UHP 的产品只需改一次配置就能更换 base 或模型，也能用同一输入对两个 Harness 做 A/B 测试，而无需第二套集成。