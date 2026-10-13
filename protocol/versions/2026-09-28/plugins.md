# 插件

**统一 Harness 协议，版本 `2026-09-28`**

Harness 携带其 agent 可用的工具和技能（[Harness §4](harnesses.md#4-tools-and-skills)）。在此版本之前，它们只能一次附加一个，附加到一个 harness。如果操作者想在第二个 harness 或另一个产品中使用相同的三个 MCP 服务器和两个技能，就必须重新输入。**插件**就是将这些相同的组件作为一个具名、带版本的包携带，可安装到任何 UHP 服务器上的任何 harness，以及任何其他读取相同包格式的客户端。

本章是 Harness 插件子协议。它是可选的：服务器通过 `plugins` 能力声明它，未实现它的服务器在所有一致性等级下都合规。

## 1. 插件是什么

UHP 不定义包格式。UHP 插件**就是**一个 [Agent Plugins](https://agent-plugins.org/specification) 包，版本 1.0.0：一个根目录下带有 `plugin.json` 清单的文件夹，MCP 服务器在 `mcp.json` 中声明，技能位于 `skills/` 下，每个技能是一个带有 `SKILL.md` 的 [Agent Skills](https://agentskills.io/specification) 文件夹。

Agent Plugins 包装 Agent Skills 而不重新定义它们：它规定技能在包中的位置，并将技能自身的格式交由 Agent Skills 规范定义。本章在上一层做同样的事。它规定包如何通过 UHP 传输、如何绑定到 harness，以及托管服务器对它负有什么义务。关于包内容的一切都交由 Agent Plugins 定义。

```
Agent Skills     SKILL.md 及其文件夹                    一个技能是什么
Agent Plugins    plugin.json + mcp.json + skills/      技能和工具如何打包
UHP Plugins      本章                                   包如何安装到托管 harness
```

插件携带三样东西，而 harness 对象已经为其中两样预留了位置：

| 部分 | 在包中的位置 | 在 harness 中变成什么 |
|---|---|---|
| 元数据 | `plugin.json` | 插件的身份：`name`、`version`、`author`、`license` 等 |
| 工具 | `mcp.json` | harness 的 MCP 服务器（[Harness §4.1](harnesses.md#41-mcp-servers)） |
| 技能 | `skills/<name>/SKILL.md` | harness 的技能（[Harness §4.2](harnesses.md#42-skills)） |

> **为什么采用一种格式而不是定义一种？**
> 包格式的价值恰好等于它能安装的地方数量。Agent Plugins 由来自主要 agent 客户端的指导委员会维护，因此为其中之一编写的插件可以原样安装到 UHP harness，而从 UHP harness 导出的插件（[§5](#5-exporting-a-harness-as-a-plugin)）也可以安装到它们中。UHP 专属格式需要在每个边界处转换，而转换正是内容丢失的地方。

## 2. 插件对象

```json
{
  "name": "contract-review",
  "enabled": true,
  "manifest": {
    "$schema": "https://agent-plugins.org/schemas/1.0.0/plugin.schema.json",
    "name": "contract-review",
    "version": "1.2.0",
    "description": "Clause extraction and a risk checklist for commercial agreements.",
    "author": { "name": "Example Legal Tools" },
    "license": "MIT"
  },
  "mcpServers": [
    { "name": "clauses", "transport": "http", "url": "https://mcp.example.com/clauses", "enabled": true },
    { "name": "redline", "transport": "stdio", "command": "./bin/redline",
      "args": ["--data", "${PLUGIN_DATA}"], "enabled": true }
  ],
  "skills": [
    { "name": "risk-checklist", "description": "Turn an agreement into a JSON risk checklist." }
  ],
  "skipped": [],
  "files": [
    { "path": "plugin.json", "content": "{ … }" },
    { "path": "mcp.json", "content": "{ … }" },
    { "path": "skills/risk-checklist/SKILL.md", "content": "---\nname: risk-checklist\n…" },
    { "path": "skills/risk-checklist/references/clauses.md", "content": "…" },
    { "path": "bin/redline", "content_b64": "f0VMRgIB…" }
  ]
}
```

| 字段 | 类型 | 由谁写入 | 含义 |
|---|---|---|---|
| `name` | string | 任一方 | 清单的 `name`；在 harness 的插件中唯一 |
| `enabled` | boolean | 客户端 | 缺失或 `true` 表示已安装且激活 |
| `files` | array | 客户端 | 包，以相对于插件根目录的文件表示 |
| `blob` | string | 服务器 | 服务器另行存储的包的句柄 |
| `manifest` | object | 服务器 | 解析后的 `plugin.json` |
| `mcpServers` | array | 服务器 | `mcp.json` 声明的服务器，以插件 MCP 服务器对象（`PluginMcpServer`）表示 |
| `skills` | array | 服务器 | `skills/` 下找到的技能：各含 `name` 和 `description` |
| `skipped` | array | 服务器 | 服务器找到但无法加载的组件，各含原因 |

服务器写入的字段在每次写入时从包派生。客户端可以原样回传它们，这正是客户端 `PUT` 它所读取内容时的做法，而服务器必须重新计算它们，而不是存储所发送的内容。客户端不得依赖修改它们：在插件上编辑 `mcpServers` 不会改变任何东西，因为包是来源，`mcp.json` 才是服务器的所在。

### 2.1 文件

规则与技能已遵循的规则相同（[Harness §4.2](harnesses.md#42-skills)）：

- `files[].path` 相对于插件根目录，且必须支持嵌套目录。服务器必须以 `plugin_invalid` 拒绝逃逸根目录的路径，或指向非常规文件的路径。
- 文本通过 `content` 传输；二进制通过 `content_b64`。服务器必须逐字节保留两者。
- 包必须在其根目录包含 `plugin.json`。服务器必须在配置时而非运行时拒绝不含它的包。
- 服务器必须在 agent 可读取的位置物化**整个包**。插件的技能携带引用和脚本，其 stdio 服务器携带它们启动的可执行文件。
- 服务器可以另行存储包，并返回 `blob` 代替 `files`。此时它必须从 `GET /v1/harnesses/{harness_id}/plugins/{plugin_name}/files`（[§3.1](#31-reading-the-files)）返回完整的文件列表。
- 服务器应限制包大小，且必须以 `file_too_large` 拒绝过大的包（`detail.max_bytes` 说明限制），而不是截断它。

### 2.2 服务器派生的内容

每次写入时，服务器像 Agent Plugins 客户端一样读取包，并在对象上记录所找到的内容。

**`manifest`。** 服务器必须根据 Agent Plugins §5 的规则，针对其 `$schema` 指定的版本验证 `plugin.json`。服务器不支持的 `$schema` 以 `unsupported_plugin_schema` 拒绝；`detail.supported` 列出它支持的标识符。未知的顶层字段，或非对象的 `extensions`，不是致命错误：该字段被忽略并记录在 `skipped` 中。任何其他违规都是致命的，写入以 `plugin_invalid` 拒绝。

**`mcpServers`。** 从 `mcp.json` 读取，按 Agent Plugins §7.2 所述。每个有效的服务器条目变成一个插件 MCP 服务器对象，映射如下。无效条目记录在 `skipped` 中，其余仍加载。如果 `mcp.json` 本身无效，或其 `$schema` 指定的 Agent Plugins 版本与 `plugin.json` 不同，则其中的每个服务器都以该原因跳过。

| `mcp.json` `type` | UHP `transport` | 携带的字段 |
|---|---|---|
| `streamable-http` | `http` | `url`、`headers` |
| `sse` | `sse` | `url`、`headers` |
| `stdio` | `stdio` | `command`、`args`、`env`、`cwd` |

`${PLUGIN_ROOT}` 和 `${PLUGIN_DATA}` 按原样报告。服务器不得在返回给客户端的内容中展开它们：展开结果是服务器沙箱内的文件系统路径，与客户端无关，且每轮都会变化。每个派生服务器上的 `enabled` 始终为 `true`；插件自身的 `enabled` 才是将它们一起关闭的开关。

插件 MCP 服务器对象是 [Harness §4.1](harnesses.md#41-mcp-servers) 的 harness MCP 服务器对象加上 `stdio` 传输，以 `command`、`args`、`env` 和 `cwd` 代替 `url`、`headers` 和 `auth`。`command` 是一个可执行令牌，绝不是 shell 字符串：一个裸名称，或以 `./` 开头的插件相对路径。

> **为什么 `stdio` 只在插件内声明？**
> 进程需要运行的位置和保存状态的位置，而插件正是同时提供两者的对象：`PLUGIN_ROOT` 和 `PLUGIN_DATA` 由包定义，而非 harness。harness 自身列表上的 stdio 服务器需要为它并不拥有的根目录另设一套规则，且无法导出，因为其可执行文件不会随之传输。因此 harness 自身列表保持上一版本的样子，仅远程服务器，而进程只差两个文件：一个 `plugin.json` 和一个 `mcp.json` 就构成一个完整插件。这使上一版本定义的每个对象逐字节不变，从而让服务器从一条代码路径同时服务两个版本（[§4.1](#41-the-direct-fields-are-unchanged)）。

**`skills`。** `skills/` 的每个直接子目录，若包含名为 `SKILL.md` 的常规文件，就是一个技能，按 Agent Plugins §7.1。服务器不得搜索更深层。每个技能必须符合 Agent Skills，包括其 frontmatter `name` 等于目录名的规则；不符合的技能记录在 `skipped` 中，其余仍加载。派生条目携带 frontmatter 的 `name` 和 `description`。

**`skipped`。** 每个条目是 `{ "path": …, "reason": … }`，其中 `path` 定位包中的组件（`skills/broken`、`mcp.json#/mcpServers/redline`、`plugin.json#/vendorField`），`reason` 是一句话。服务器必须在每个插件对象上返回 `skipped`，无跳过时为空，以便客户端区分"无跳过"和"服务器不报告跳过"。

> **为什么记录损坏的组件而不是拒绝？**
> 两条规则在此交汇。[Harness §4.2](harnesses.md#42-skills) 在配置时拒绝没有 `SKILL.md` 的技能，因为一个被存储后在运行时被静默忽略的包是用户能收到的最难排查的故障。Agent Plugins §11.3 要求包内相反的做法：隔离到单个组件的故障不得阻止其余部分加载，因为包是由安装者以外的人编写的，一个过时条目不应拖垮整个插件。两者在各自的情况下都是对的。调和方式正是 [Tasks §1.1](tasks.md#11-request-fields) 已为保留字段所做的：允许忽略，不允许静默忽略。包安装，未加载的内容写在对象上，配置 harness 的人可以看到。清单是例外。没有有效的清单就没有可在其中隔离故障的插件，因此致命的清单错误拒绝写入。

## 3. 安装插件

插件是 harness 对象的一部分，按其其他配置的方式设置（[Harness §5](harnesses.md#5-managing-harnesses)）：一致性等级 **Full**，且 `plugins` 能力必须为 `true`。

```http
POST /v1/harnesses
```

```json
{
  "name": "Contract Review Agent",
  "base": "claude-code",
  "plugins": [
    { "files": [
        { "path": "plugin.json", "content": "{\"$schema\":\"https://agent-plugins.org/schemas/1.0.0/plugin.schema.json\",\"name\":\"contract-review\"}" },
        { "path": "skills/risk-checklist/SKILL.md", "content": "---\nname: risk-checklist\ndescription: …\n---\n…" }
    ] }
  ]
}
```

- 写入时的 `name` 是可选的。若存在，它必须等于清单的 `name`；否则写入以 `plugin_invalid` 拒绝。清单是名称的来源，因此安装时重命名的插件仍是其作者发布的插件。
- 一个 harness 中两个同名插件必须以 `plugin_conflict` 拒绝。
- 被拒绝的写入必须使 harness 保持不变。插件要么整体安装，要么完全不安装。
- `PUT /v1/harnesses/{harness_id}` 替换 `plugins`，如同替换其他每个可变字段。通过 `GET` 和 `PUT` 往返 harness 不得丢失包内容；发送回 `{ "name": "contract-review", "blob": "…" }` 的客户端保留它所读取的包。
- `enabled: false` 使插件保持安装但惰性。其中任何内容都不物化，其 MCP 服务器也不被联系（[Harness §4.1](harnesses.md#41-mcp-servers)）。

### 3.1 读取文件

```http
GET /v1/harnesses/{harness_id}/plugins/{plugin_name}/files
```

```json
{ "files": [ { "path": "plugin.json", "content": "…" }, { "path": "bin/redline", "content_b64": "…" } ] }
```

完整的包，逐字节，无论服务器是否另行存储它。当 harness 没有该名称的插件时，返回 `404` 和 `plugin_not_found`。

## 4. 组合：agent 得到什么

harness 的**有效配置**是其 agent 运行所用的配置。服务器每轮从 harness 对象计算它：

- 有效 MCP 服务器是 harness 启用的 `mcpServers`，加上每个启用插件的 `mcpServers`；
- 有效技能是 harness 启用的 `skills`，加上每个启用插件的 `skills`；
- `disabledTools` 对它们一视同仁地适用。

### 4.1 直接字段不变

harness 上的 `mcpServers` 和 `skills` 保持与上一版本完全相同的含义。它们报告客户端在那里写入的内容，仅此而已。插件的组件报告在插件上，绝不复制到直接字段中。

没有插件的 harness 就是本版本之前的样子，`plugins` 为空数组。今天存在的每个 harness 在本版本下都是有效 harness，针对上一版本编写的每个客户端都继续工作，依据 [Versioning](../../VERSIONING.md) 中的客户端规则：它忽略 `plugins`，看到一个工具比 agent 实际拥有的更少的 harness，而这已经是它对 base 自行提供的工具所看到的情况。

反之亦然。本版本定义的任何内容都不改变上一版本定义的对象形状，因此实现插件的服务器从一条代码路径服务两个版本：它在 `versions` 中列出两者，以相同的对象应答 `2026-08-11` 请求，而 `plugins` 字段只是该版本客户端忽略的又一个字段。未实现插件的服务器继续服务 `2026-08-11`，报告 `plugins` 能力为 `false`，并在本版本的测试套件下于所有一致性等级都合规；插件检查跳过，而跳过绝不是通过。

> **为什么不在 `mcpServers` 中报告有效集合？**
> 因为往返。[Harness §4.2](harnesses.md#42-skills) 正是围绕客户端读取 harness 并在无关编辑后 `PUT` 回去这一情形设计的。如果 `mcpServers` 携带插件派生的服务器，该 `PUT` 会把它们写入直接列表，harness 随后会携带每个插件服务器两次：一次来自插件，一次作为自身的。卸载插件会留下其服务器。因此直接字段报告写入它们的内容，想要有效集合的客户端取并集，本节使其成为一行计算，其中不含任何服务器专属规则。

### 4.2 一个命名空间，在配置时检查

在一个 harness 内，MCP 服务器名称必须在直接列表和每个启用插件之间唯一，技能名称同理。会造成冲突的写入必须以 `409` 和 `plugin_conflict` 拒绝；`detail.component` 为 `mcp_server` 或 `skill`，`detail.name` 是冲突的名称，`detail.between` 命名两个所有者，各为 `harness` 或插件名。

> **为什么拒绝而不是加前缀？**
> agent 看到一个扁平命名空间：一个 MCP 配置，一个技能目录。服务器能否为插件的组件加上插件名前缀取决于 harness base，而对技能则不能：Agent Skills 要求技能的 `name` 等于其目录名，因此重命名目录会破坏技能。协议无法在每个 base 上保持的规则就不是规则。因此歧义在配置时被拒绝，那里有人在场解决它，而不是在运行时被静默解决，那里无人。禁用任一方即可清除冲突。

### 4.3 直接字段就是没有名字的插件

harness 的直接 `mcpServers` 和 `skills` 恰好是插件的内容：工具和技能，harness 代替清单。这不是类比。[§5](#5-exporting-a-harness-as-a-plugin) 使它成为一个操作。

## 5. 将 harness 导出为插件

```http
GET /v1/harnesses/{harness_id}/plugin
```

返回一个插件对象，其 `files` 是从 harness 的直接配置构建的 Agent Plugins 包。结果可通过原样传给 `plugins` 安装到另一个 harness，或通过将其文件写入磁盘安装到任何其他 Agent Plugins 客户端。

| 包中 | 构建自 |
|---|---|
| `plugin.json` | Agent Plugins 1.0.0