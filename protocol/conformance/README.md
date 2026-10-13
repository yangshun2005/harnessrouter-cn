# UHP 一致性测试套件

**通过本套件，才叫"符合 UHP"。** 不是实现了端点，不是自我评估，也不是通过了大部分。套件就在本仓库中，通过 HTTP 对任何服务器运行，任何人都可以对任何实现运行它。

## 运行方式

```bash
pip install "git+https://github.com/HarnessRouter/harnessrouter.git#subdirectory=protocol/conformance"

uhp-conformance \
  --base-url https://your-uhp-server \
  --api-key "$UHP_API_KEY" \
  --class full \
  --json report.json
```

安装的包自带其 UHP 版本的 Schema；无需检出仓库即可校验响应，若包因故丢失 Schema，所有依赖 Schema 的检查会报告为套件自身的错误，而非跳过。开发时，请在仓库根目录运行 `pip install -e protocol/conformance`。

`--harness-id` 接受 harness 的 id（HarnessRouter 上为 `chrn_…`），而非其名称：匹配不到任何 harness 的 id 会在第一个任务前终止运行；当未指定 `--model` 时，指定了名称但没有默认模型的 harness 同样如此。任务运行的 harness 和模型会在第一个检查前打印。

| 选项 | 含义 |
|---|---|
| `--base-url` | 服务器根地址。套件自行追加 `/v1/...`。 |
| `--api-key` | Bearer token，或设置 `UHP_API_KEY`。 |
| `--class` | `core`、`extended` 或 `full`。累积式——`full` 运行全部。 |
| `--harness-id` | 针对指定 harness 运行任务，而非列表中的第一个。 |
| `--model` | 使用指定模型运行任务。当 `--harness-id` 指定的 harness 没有默认模型时必须提供：套件会提示并在第一个任务前停止，而不是逐个等待任务超时。 |
| `--task-timeout` | 单个 agent 任务的超时秒数。默认 300。 |
| `--plugin-mcp-url` | 插件检查指向插件的可访问 streamable-HTTP MCP 服务器，或设置 `UHP_PLUGIN_MCP_URL`。默认为套件自带 fixture 的公开副本（见下文）。 |
| `--only` | 逗号分隔的检查 id，用于针对单个失败迭代。子集会被报告为部分运行，而非某类一致性声明。 |
| `--json` | 写出机器可读的报告。 |
| `--plain` | 无 ANSI 颜色，用于 CI 日志。 |

`--only` 保留退出码约定以便调试：`0` 表示所选检查未失败或报错。它不认证所请求的类别。决定是否发布类别声明时，请使用 JSON 的 `conformant` 和 `coverage` 字段，或终端摘要。

退出码为 `0` 表示无失败，否则为 `1`，因此可直接接入 CI。

**套件会运行真实的 agent 任务。** 它会发送约七个任务，消耗真实的模型 token 并耗时几分钟。这是刻意的：值得捕获的缺陷——永不刷新的流、永不终止的取消、无法下载的构建产物——对任何只检查 Schema 的东西都是不可见的。

## 检查内容

三个类别共 76 项检查。

| 类别 | 检查数 | 覆盖范围 |
|---|---|---|
| **Core** | 40 | 发现、版本协商、认证、错误封装、harness、模型、任务执行（流式与非流式）、事件流、会话、取消、保留请求字段 |
| **Extended** | +8 | 会话列表与检查、文件输入、构建产物、下载头、路径遍历探测 |
| **Full** | +26 | Harness 创建 / 更新 / 删除、拒绝不支持的基础、技能文件夹往返、MCP 与禁用工具持久化、会话共享、插件（包往返、派生、拒绝、导出，以及通过插件服务器发起的工具调用） |

每项检查都标明其所执行的规范章节，因此失败会指向被违反的句子，而非测试名称。

其中几项值得特别说明，因为它们能捕获 Schema 检查永远捕获不到的问题：

- **S-04** — `sequence_number` 无间隙且单调。没有它，客户端无法区分丢失的事件和单纯跳号的服务器。
- **S-09** — 流是渐进的。它测量事件到达时间的分布，并让缓冲全部内容、最后一次性刷新的服务器失败。这是最常见的 UHP 部署错误，通常是代理设置问题，除非有东西测量它，否则与慢速 agent 无法区分。
- **C-03** — 运行中的任务确实会停止。它启动长时间工作、取消它，并等待终态。一个返回 `200` 却让 agent 继续运行的取消端点能通过其他所有类型的测试。
- **X-07** — 构建产物下载时带 `X-Content-Type-Options: nosniff`。构建产物是攻击者可影响的内容；不带此头提供，构建产物就是对客户端自身源的存储型 XSS。
- **X-08** — 构建产物 id 不会遍历出其容器。探测 `../` 及其百分号编码形式。
- **X-09** — `POST /v1/files` 本身。X-05 内联发送文件，因此上传端点彻底失败的服务器仍能通过文件章节：HarnessRouter CE 0.17.3 对每次上传都返回 500，却是绿的。X-09 上传、按 Schema 和发送时的字节数校验文件对象，并从任务中引用该 id。
- **P-02/P-06** — 插件包是派生的，不是复制的，且 harness 导出为可再次安装的包。P-02 让自身 `mcpServers` 吸收插件服务器的服务器失败，这个错误会让每次读后写都安装两次；P-06 让携带操作员凭据的导出失败，也让未携带却不说明的导出失败。两者在报告 `plugins` 能力为 false 的服务器上均跳过而非失败，因为该章节是可选的。
- **P-11** — 唯一运行任务的插件检查。它安装一个插件，其唯一服务器是运行所指向的 fixture，要求 agent 用 nonce 调用 fixture 的工具，并期望返回 fixture 的答案（agent 不调用就无法产生的 nonce 函数）。P-02 到 P-10 读取服务器派生并拒绝的内容；只有这一项展示 agent 在运行时到达插件的服务器。
- **T-08/T-09/T-10** — 保留字段 `tools` 和 `include` 被接受，并报告为已忽略。两半都重要：在此之前，套件两个字段都不发送，因此直接拒绝它们的服务器和静默对其采取行动的服务器得分与正确服务器相同。T-10 覆盖第三种错误，即服务器报告请求从未发送的字段。
- **R-01…R-08** — 会话共享，主要是一组拒绝。Sessions §5 要求共享视图只读、可撤销且不含凭据，而仅打开链接并读取对话的检查，会通过一个同时允许该链接继续任务、取消运行或上传到工作目录的服务器。§5 将共享设为可选，因此这些在不实现它的服务器上跳过而非失败。由于 §5 命名了共享对象的 `url` 和铸造端点上的 `DELETE`，`R-01`、`R-02` 和 `R-06` 断言两者而非发现它们——它们过去以缺失句子为原因跳过。`tests/test_session_sharing_checks.py` 针对一个故意错误的服务器逐个缺陷运行该系列，因此每个检查都已知会在其描述的缺陷上失败。
- **R-08** — 无 body 的 `POST` 会发布。该系列通过一个辅助函数铸造，该函数用 `{"enabled": true}` 重试 400/422，因为只讲这种方言的服务器否则会跳过上述全部七项检查——因此重试使该系列可移植，也正是它隐藏了这句话：仅支持开关的服务器凭借 §5 不要求任何人接受的请求通过 `R-01`…`R-07`，却拒绝它确实要求的那个。`R-08` 是唯一不重试的检查。其桩缺陷 `bodyless_rejected` 正是本系列存在之前参考实现的样子。
- **F-03/F-04** — 技能是一个文件夹，且能在无关编辑后存活。只存储 `SKILL.md` 的服务器，或在 harness 重命名时清空 bundle 的服务器，能通过其他所有检查：配置看起来仍然正确，损失只在之后表现为 agent 行为异常。

## 插件 fixture

每项插件检查都将插件指向一个 MCP 服务器，主机允许在配置时拒绝无法到达的服务器并记录在 `skipped` 中。直到 2026.9.12.post3，该服务器都是无法解析的占位符，因此在此类主机上，检查测量的是拒绝而非插件。套件现在自带所需的服务器：

```bash
pip install "uhp-conformance[fixture]"
uhp-conformance-fixture --host 0.0.0.0 --port 8080      # serves http://<host>:8080/mcp
uhp-conformance --base-url https://your-uhp-server --api-key "$UHP_API_KEY" --class full \
  --plugin-mcp-url https://fixture.example.com/mcp
```

它是一个 streamable-HTTP MCP 服务器，带两个工具：`uhp_echo`，其答案是输入的函数（前缀 `UHP-FIXTURE:` 加反转文本），P-11 借此证明 agent 调用了它；以及 `uhp_time`。运行默认使用同一代码的公开副本，位于 `https://uhp-fixture.harnessrouter.ai/mcp`（包中的 `DEFAULT_PLUGIN_MCP_URL`；`fixture/Dockerfile` 是其构建方式），任何有出站 HTTPS 的主机都可访问。
被测主机若其 agent 无法访问公共互联网，则需要一个可访问的副本，地址通过 `--plugin-mcp-url` 传入。

## 结果

| 结果 | 含义 |
|---|---|
| **PASS** | 观察到了要求的行为。 |
| **FAIL** | 观察到行为错误。一致性缺陷。 |
| **SKIP** | 检查无法运行，原因已记录。 |
| **ERROR** | 检查本身出错——套件的 bug。 |

**跳过永远不是通过。** 报告单独列出跳过并重申它们未被验证，因为一个把未运行检查藏在绿色摘要背后的套件，正是一个套件开始对它存在所要确立的东西撒谎的方式。

JSON 报告也遵守同一规则，因为它是作为一致性声明证据发布的构建产物，其读者往往不是运行它的人：

- `conformant` 仅当所请求类别要求的每项检查都运行并通过时才为 `true`。跳过或未选中的检查使其为 `false`。
- `conformant_with_skips` 仅当每项要求的检查都运行且无失败或报错时才为 `true`；实际跳过项在 `skipped_not_verified` 中枚举。部分 `--only` 运行将其设为 `false`，因为未选中的检查未被尝试。
- `coverage` 报告所有要求的检查是否运行、其计数，以及未运行或出现多次的 id。部分运行可以通过所有选中的检查，却仍不作出一致性声明。`highest_class_passed` 只计入完全覆盖且通过的类别。
- `suite_version` 和 `generated_at`（UTC）将报告绑定到套件修订版和产生它的时刻。缺少这些字段的报告早于套件 `2026.8.11.post1`；更早运行的已检入报告属于那种旧形态。

## 已测量的实现

<!-- measured-implementations -->

此处的行是证据，不是证书：行中链接的报告由本仓库的 `conformance-measure` 工作流针对所命名的实现生成，并通过任何人都可阅读的 Pull Request 落地。徽章在每次站点构建时从该报告生成，仅存在于完全通过且无跳过的类别；它只说明类别、服务器所服务的协议版本，别无其他。未通过的实现没有徽章。

要被测量，请开一个 Issue 或在你的列表 Pull Request 中评论，提供可访问的服务器根地址及要使用的凭据，或安装并启动服务器的命令，维护者会派发工作流。自带密钥的主机也可被测量：套件的 `--header` 标志发送你服务器想要的头，密钥由测量者支付。每次运行约花费六个小型模型任务。每次发布时重新测量：最新报告胜出，旧报告作为历史保留。

```bash
# What the workflow runs, for a local or self-hosted target
uhp-conformance --base-url http://127.0.0.1:8787 --api-key "$KEY" --class core --plain \
  --label "My Server 1.2.0" --json report.json
# For a host that expects the caller's model key in a header of its own
uhp-conformance --base-url https://uhp.example.com --api-key "$KEY" --class core \
  --header "X-Provider-Api-Key: $MODEL_KEY" --json report.json
```

## 参考实现结果

<!-- conformance-recorded-run:start — regenerated by .github/workflows/conformance-remeasure.yml; do not edit by hand -->
HarnessRouter Community Edition 0.17.3，本仓库中的参考实现，于 2026-09-15 针对实时实例运行（套件 2026.9.12）：

```
  Summary
    74/74 passed · 0 failed · 0 skipped · 0 errored
    CONFORMANT — UHP 2026-09-12 (full)
```
<!-- conformance-recorded-run:end -->

两个新系列都直接针对参考实现、在自托管实例上实时测量，且都在通过前测量到真实缺陷：

- `R-01`…`R-07` 全部通过。针对当时的参考实现，七项全部跳过（其共享方言想要 `{"enabled": true}` 且不发布视图 URL），而这些跳过背后藏着一个真实的 Sessions §6 违规：会话的共享链接在 `DELETE /v1/traces/{id}` 之后仍继续提供完整对话，修复前已实时验证。
- `T-08`–`T-10` 通过，其中 `T-09` 针对当时的参考实现失败：`tools` 喂给了幂等哈希，却没有任何东西报告它被忽略——§1.1 现在禁止的静默丢弃。

一个只发布快乐路径检查的套件，两次都会称该服务器符合一致性。

`R-08` 是之后添加的，来自关闭那两项的 PR 在其正文中自己指出而非留待发现的缺口：*"body 是 OPTIONAL；无 body 意味着发布" 尚无专门检查。* 它针对参考实现和独立的 Go 实现（后者 `full` 63/63，零跳过）均通过。两个服务器目前都无法让它失败——这正是该检查的理由，而非反对它的理由：两个实现碰巧一致，正是规范句子本应不再成立的东西，而两棵树中都没有任何东西会注意到其中一个停止一致。

套件是针对该实现开发的，这正是规范说一致性由套件定义而非由实现定义的原因：参考实现做的任何套件不要求的事，都是 HarnessRouter 行为，而非 UHP 要求，其他实现可以自由地以不同方式做。

编写和运行套件发现了四个真实缺陷，三个在参考实现中，一个在套件本身：

1. 完全没有能力发现——客户端只能猜测或从 `404` 中学习。
2. 线上没有协议版本。
3. 失败返回裸的人类字符串，因此客户端必须匹配散文才能决定是否重试。
4. `POST /v1/harnesses` 接受了服务器无法运行的基础，将失败推迟到第一个任务——在客户端已提交之后。首次完整运行时被 **F-02** 捕获。

针对实时 agent 测试工具和技能表面又发现了三个，任何配置级检查都看不到：

5. **Hermes 完全无法使用 HTTP MCP 服务器。** 镜像安装了未固定版本的 `mcp` SDK，解析到的版本移除了 Hermes 用于门控 HTTP 传输的符号。为该后端配置的每个远程 MCP 服务器都被静默丢弃，agent 回复"我无法访问该工具"——与模型拒绝无法区分。SDK 现已固定版本、启动时验证，并在已有损坏版本的卷上就地修复。
6. **Hermes 完全忽略 `disabledTools`。** Claude 用硬标志强制执行，Codex 将其作为指令接收；Hermes 两者都没有，因此禁用了工具的操作员既无强制执行也无警告。
7. **MCP URL 策略是建议性的。** 它只在控制台的"测试连接"按钮上运行，因此控制台拒绝的 URL，回合随后仍会连接。现在它是一个函数，在配置时和运行时都应用。
8. **Claude 的硬阻止是空操作，而产品声称它是硬的。** 禁用的工具作为 `--disallowedTools` 传入，这属于权限提示系统——而自主运行传入 `--dangerously-skip-permissions`，会关闭该系统。标志被接受并忽略，因此禁用了 `Bash` 的操作员看着 agent 运行 `Bash`。限制现在作为拒绝规则写入运行时的设置文件，跳过权限标志无法覆盖。针对实时 agent 双向验证：启用工具时它运行命令；禁用时 agent 报告没有该工具且未运行。

### 这些检查不确立什么

套件验证 `disabledTools` **持久化**，而非**强制执行**。强制执行刻意不检查，因为 §4.3 允许指令级强制执行，而遵守指令的 agent 在 HTTP 上与阻止工具的运行时无法区分——因此行为检查会在模型恰好遵守时通过一个阻止无效的服务器。缺陷 8 是一个空操作阻止，行为检查在大多数运行中会称其符合一致性。强制执行针对运行时验证，通过禁用工具并断言 agent 从不调用它，它属于实现自己的测试套件而非此处。

## 添加检查

检查是断言一项要求的函数：

```python
@check("T-08", "Tasks reject an empty input", "core", f"{SPEC}/tasks.md#2-input")
def t08(ctx):
    r = ctx.client.post("/v1/responses", body={"input": ""})
    assert r.status == 400, f"empty input returned HTTP {r.status}, expected 400"
```

好检查的规则：

- **断言规范所说的，而非参考实现所做的。** 规范允许余地之处，检查也必须允许——否则套件强制执行一个实现的偏好，其他每个实现都会因不同而非因错误失败。