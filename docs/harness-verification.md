# 验证 harness

能回答的 harness 不等于能用的 harness。它必须在你选定的连接上、在你指定的模型上运行，在模型切换和沙箱回收之间保持对话，并向读者展示它实际产出的内容。本页说明 harness 必须证明什么、每项声明如何度量、规则在代码中的位置，使变更可以被检验而非被相信。

工具是 [`scripts/support-matrix`](../scripts/support-matrix/README.md)；该 README 说明如何运行。本页讲它判定什么以及为什么。

## 度量内容

对每个 harness 及其菜单提供的每个模型，一个会话运行五个场景：

| 场景 | 证明什么 |
|---|---|
| 首轮 | harness 启动，使用指定模型，并作答 |
| 追问 | 会话继续，而非重新开始 |
| 切换 | 会话中途切换模型，在新模型上运行并保持上下文，然后切回 |
| 产物 | 任务必须产出的文件存在于轮次记录中，并展示给读者 |
| 回收 | 沙箱被有意释放，之后的追问仍能回忆起第一条消息 |

有一个基座不是编码代理。`systemone` 在类型化动作空间上运行决策循环，而非在工作区上运行 CLI，因此产物场景的提示对它不适用；它由自己的循环（其包的测试、UHP 一致性及其基准）以及此处的首轮、追问、切换和回收场景验证。参见 docs/support-matrix-notes.md 中的 "systemone"。

## 决定一行的规则

一个"已完成"的场景本身不等于通过。四条规则将一次运行转为判定，每条都在代码中强制执行，而非靠肉眼。

**1. 它在你说的地方运行。** 由被测连接之外的连接服务的轮次是发现项，绝不是通过。为运行设置 `EXPECT_CONNECTION`；每个轮次记录自身的连接戳记与之比对，因此由两个连接服务的一对是可见的。
在 `scripts/support-matrix/run.mjs` 中强制执行，由 `render.py` 报告。

**2. 它运行了你要求的内容。** 服务的模型不是所请求的 id 即为替换，是发现项。唯一例外是提供商对同一模型的自身命名：聚合器的厂商前缀（`anthropic/claude-fable-5`）或提供商的日期或版本后缀（`claude-haiku-4-5-20251001`）是同一模型，在行中注明并计入。任何前缀下不同的系列、编号或层级（`gemini-3.8-flash` 对应 `google/gemini-3-flash-preview`，`gemini-2.5-flash` 对应 `gemini-2.5-flash-lite`）是另一模型，仍为发现项。
在 `scripts/support-matrix/samemodel.py` 中强制执行，由 `test_samemodel.py` 固定。

服务模型来自何处是 CLI 的选择，有些则不上报：goose 仅在其从不使用的提供商格式上上报，cline 和 qwen 也不上报。这曾使该规则对这三个不可执行——三个 harness 上的一半标准。现在不再是。这些后端上的每个轮次都经由回环中继，且提供商自身的应答会命名 `model`，因此中继在字节流经时读取它，运行器将其戳记在 CLI 未标注的结果上（`runner/server.py` 中的 `_served_model_in` 和 `_relay_served_model`，由 `runner/tests/test_relay_served_model.py` 固定）；自行上报的 CLI 保留其上报。因此对新 harness 要问的不是"这个 CLI 是否上报服务模型"，而是"它是否经由中继"——若是，则无论 CLI 是否配合，规则 2 都适用。

**3. 读者所见即所存。** 产物行要求渲染的文件卡片就是该轮次存储的文件：相同名称、相同数量。仅询问某张卡片是否带有预期名称，曾让一个渲染两次的文件作为产物通过了数月。
在 `scripts/support-matrix/run.mjs` 中强制执行。

**4. 目录不承诺 harness 做不到的事。** 提供商仅在 Responses API 上服务的模型，不会在只支持 chat/completions 的 harness 上提供，因为在那里列出它就是一个发送即失败的选择器行。
由 `gateway/tests/test_catalog_chat_only_backends.py` 强制执行。

## 自定义 harness 维度

五个场景度量路由。它们不涉及用户实际构建的配置，而这正是本产品自身承诺所在：你自己的技能、携带你自己的脚本，以及对运行时自带工具的控制。一个能在所有模型上作答却忽略你所写技能的 harness 是不合格的。

[`scripts/support-matrix/custom-harness.mjs`](../scripts/support-matrix/custom-harness.mjs) 为每个基座创建一个 harness，携带一个技能包，其 `SKILL.md` 指示代理运行随附的脚本，并关闭一个继承的工具。它运行一个轮次，然后删除该 harness。每个基座一个轮次，而非每个模型：这度量 harness 携带什么，而非轮次路由到哪里。

一行仅当以下五项全部成立才通过，每项均从服务器存储的内容读取：

| 声明 | 如何证明 |
|---|---|
| 技能已存储在 harness 上 | 读取 harness 时返回 |
| 工具策略已存储 | 被禁用的工具也返回 |
| 技能包到达了代理 | 应答携带一个仅存在于脚本内的令牌 |
| 脚本确实运行了 | 脚本写入的文件在该轮次产出的文件中 |
| 工具策略生效 | 该轮次的工具调用中不出现被禁用的工具。它禁用的是该基座自身的工具，来自 `custom-harness.mjs` 的 `DISABLED_TOOL` 表：在运行时中匹配不到任何东西的工具 id 会在传递途中被丢弃，什么也不禁用，因此单一固定 id 曾使该声明在二十个基座中的十一个上空洞地成立。没有任务可舍弃工具的基座（aider，其目录只有一个 Shell）会说明该策略在此不可测，而非通过 |
| MCP 服务器已存储并被调用 | 读取时返回，且第二个轮次的工具调用命名它。调用由其工具名命名，但 omp 除外，它将 MCP 作为对 `xd://mcp__<server>_<tool>` 的 `write` 分派：那里由调用的参数命名它 |

令牌每次运行生成，绝不出现在 `SKILL.md` 或提示中，因此携带它的应答来自技能包而非模型的想象，而写入的文件区分了运行过的脚本与仅被读取的脚本。

harness 的另一类工具是 MCP 服务器，同一 harness 声明一个。自包含实例仅托管数据库和媒体服务器，一个需要数据库，另一个每次调用花费真金白银，因此这一半指向一个无需密钥、无需账户的公共 MCP 服务器，且同一会话中的第二个轮次必须调用它。

它评判的是调用，而非应答：公共服务器的文字不归我们固定，但工具调用是轮次记录中的事实。各后端对这些调用的命名不同，有些记录服务器和工具（`deepwiki.read_wiki_structure`），有些仅记录使用了 MCP 工具（`mcp`），由于 harness 恰好声明一个服务器，两种形态都能识别它。

第三方是依赖，因此按依赖对待。运行前探测一次该服务器；若不可达则跳过 MCP 这一半，行中说明原因，因为公共服务宕机不是关于本产品的证据。`MCP_URL=off` 直接跳过，用于无出网的实例，任何其他 URL 覆盖默认值。

## 系列巡览

一次对话，一个交付物，依次遍历每个模型系列。首轮构建一个小交付物（一页 `.pptx`）；每个后续轮次将编排器的模型切换到下一个系列（OpenAI、Anthropic、Google、xAI、Meta、DeepSeek、Moonshot、Qwen、Zhipu、Mistral、StepFun、Tencent、NVIDIA，各一个模型），并要求对同一演示文稿再做一处修改。轮次完成且演示文稿再次产出即为通过；实例服务的每个系列都通过则巡览通过。harness 的任何方面都不得依赖于上一轮由哪个模型作答：会话文件、恢复、对轮次结束方式的判定。

它的存在是因为矩阵的五个场景未能在 2026-09-27 捕捉到一个人用五轮发现的问题：CheetahClaws 在 muse-spark-1.1 上的一个完整作答的轮次被报告为失败，因为该模型较小的上下文使 CLI 在轮次中途压缩历史，而驱动器按该历史中的位置判定轮次。每个基座在其列发布前运行巡览，改变自身历史读取方式的基座重新运行。

`scripts/support-matrix/family-tour.mjs` 通过控制台驱动它（`HARNESS=<base>`；`SID=` 继续现有对话，这是重测某人自己失败会话的方式），并为每个系列写一条记录：模型、服务模型、状态、秒数、该轮次产出的文件，以及失败时的失败文本。注释中列出实例、连接及每个系列的结果。

## 运行本身的规则

- **一次一个提供商，隔离意味着删除。** 一列度量提供商实际能驱动什么，因此运行时组织仅持有该提供商的集成，别无其他。指向某集成的模型映射不是隔离：另一个集成仍可服务该轮次。
- **一次运行独占实例。** 列运行期间不要部署控制台或新镜像。实时轮次检查在 worker 轮次之间的间隙通过，因此"无轮次运行"不等于"无任何运行"，该间隙中的部署会杀死 worker 的会话并损失该列。
- **绝不从另一实例继承已验证列表。** 每个实例通过自己的路径和密钥到达提供商。
- **排除模型前重测裸 `incomplete`**，并在失败行上记录复现的提供商错误文本。
- **发现项绝不计为通过。** 表格单独报告发现项，并将这些对排除在场景计数之外。

## 添加 harness

新 harness 在以下位置注册，且在有度量列之前不算完成。每一处遗漏都会静默失败——harness 仍能构建、仍能作答，并在不声不响中丢失一项能力。唯一例外是控制台的后端联合类型，它会使类型检查失败，也是编译器唯一能替你发现的。

**运行器** — `runner/server.py`：

1. `BACKENDS`：其提供商、默认模型，以及将其输出转为网关存储形态的规范化器。
2. 构建其 argv 的 `turn()` 分派分支。
3. `_write_skills`：技能包必须落在何处，该 CLI 的加载器才能找到。优先 `.harness/` 下的路径：工作区根目录会被收集为产出文件，因此写在那里的 skills 文件夹每轮都会作为交付物交还给用户。
4. `_agent_doc_path`：`AGENTS.md` 或 `CLAUDE.md`——CLI 实际读取的那个。弄错会写入文件而代理永远看不到，工作区契约永远无法到达。
5. `_resume_lost`：无法继续对话的轮次如何说明，若该 CLI 会丢失。此处沉默会被读作一个已完成却忘了一切的轮次。
6. **该 CLI 如何报告提供商失败——检查，不要假设它有错误事件。** 有几个将其叙述为普通助手文本然后正常结束运行：claude 注入 `API Error: …`，goose 注入 `Ran into this error: …`。照字面读取，轮次以失败为答案完成，运行计为通过，该会话的下一轮从携带错误文字的历史作答。规范化器必须识别 CLI 的前缀，使轮次失败，并保留该句作为原因——`_CLAUDE_ERR_RE` 和 `_GOOSE_ERR_RE`。用测试固定前缀，并从固定的二进制（`strings`）获取，而非文档。
   harness 也可能完全不报告失败。openhands 的 agent-server 仅为非 `ConversationRunError` 的异常发布错误事件，假定运行已发出自己的——而从运行的错误处理中抛出的异常恰恰是无人发出的情况：状态翻转为 error，而线路上完全没有原因。后端写日志时，读取其尾部作为回退，并从中取出 EXCEPTION 行而非最后几行——一个几秒内死亡然后被重试至耗尽的轮次以 `SIGTERM … Shutting down` 结束，把它放进记录比沉默更糟，因为它读起来像个原因。
   成功的轮次也可能丢失已配置的 MCP 服务器。Goose 1.50.0 在 stderr 记录 `Failed to start extension 'name' ..., continuing without it`，然后发出成功的 stream-json 结果。运行器必须将其保留为 `system/mcp_unavailable`，以便网关能告诉用户哪些工具缺失。不要将扩展的原始 stderr 复制进该事件；它可能含凭据。
7. `CHECKPOINT_EXCLUDE` 和 `_git_ensure` 的忽略列表：CLI 写入的任何可能含凭据的文件。

**网关** — `gateway/app.py`：

8. `_MODEL_CATALOG`：它提供的 id，如实（见规则 4）。
9. `_BASE_CATALOG`：标签、系统提示、**以 CLI 自身工具名**表示的工具列表，以及 `tool_enforcement`——仅当运行时确实能扣留工具时为 `hard`，依据 UHP §4.3，该节禁止夸大和低估。匹配不到任何东西的工具 id 会在传递途中被丢弃，什么也不禁用，而控制台将其报告为关闭。
10. `_INTEGRATION_WIRING`：哪些提供商类型能驱动它。
11. `_CUSTOM_FORMAT_BACKENDS`：它实际能说哪些自定义端点格式。

`_HID_PREFIX_RE`、`_backend_of_builtin` 和 `_backend_of_harness` 派生自 `_BASE_CATALOG`，无需编辑——它们曾是手写列表，各自静默遗漏了一个基座。

**控制台**——注意只有第一项会被类型检查捕获：

12. `ui/src/lib/harness.ts`：`backend` 联合类型，以及 `OOB`，控制台以其标记和模型列出的内置 harness。
13. `ui/src/components/HarnessLogo.tsx`：其品牌标记，若有。映射到不存在的文件会渲染出损坏图像——比它回退到的通用字形更糟。
14. `ui/src/lib/harness.ts` 的 `instructionFileLabel`（由 `HarnessSettings.tsx` 调用）：指令文件标签，必须与 `_agent_doc_path` 一致，否则设置页会命名运行器不写的文件。它曾漂移——qwen 和 gemini 被标为 `AGENTS.md`，而运行器写的是 `QWEN.md` 和 `GEMINI.md`——因此现在是一张映射表，由 `gateway/tests/test_instruction_file_label.py` 与运行器的函数保持一致。

**安装与度量：**

15. `docker/entrypoint.sh`：`HR_BACKENDS`、`backend_bin`、安装器，及其在 `install_backends` 中的调用——首次启动时安装的 CLI，按其自身许可，精确固定并验证。固定标签不是验证：标签可被移动，发布资产可被重新上传。上游发布校验和处，获取并比对（`install_omp`）；不发布处，自行计算摘要并固定在版本旁（`install_goose`）。跳过检查的覆盖必须在日志中说明，而非悄悄跳过。
16. `scripts/support-matrix/custom-harness.mjs`，`BASES`：否则自定义 harness 维度根本不会度量此 harness。
17. `docs/support-matrix.md`，每个提供商一列，由套件产出，注释在 `docs/support-matrix-notes.md`。

其模型必须诚实：选择器提供的每个 id 都必须以自身运行，在不同模型上运行的轮次应失败而非悄悄成功。

### 远程运行时

远程运行时是其代理循环运行在此运行器不生成之处的一种 harness：通过 API 到达的托管会话，此处的轮次进程是它的 HTTP 客户端。Qoder Cloud Agent（`qoder`，`runner/qoder_driver.py`）是第一个。注册表条目声明它——`BACKENDS[...]["remote"] = True`——该标志是运行器据此区分以下差异的依据，因此第二个遵循相同形态而非重新发现。每轮契约不变：每轮一个进程、`__hr_init`、每个事件一行 `{"m","p"}`、`__hr_result`，且 `server.py` 不感知线路。变化的是每个本地后端免费获得的三样东西必须构建，且上述若干注册点没有对象。

**不适用，基座说明而非跳过：**

- *15，安装器。* 没有二进制。`docker/entrypoint.sh` 不为它安装任何东西，目录条目的注释说明运行时是远程的。
- *3 和 4，`_write_skills` 和 `_agent_doc_path` 作为加载器查找的位置。* 此机器上没有任何东西加载任何东西。技能暂存于 `.harness/skills/` 供驱动器发布（qoder：作为绑定在会话 Agent 上的 Skill 资源，按内容哈希复用）；harness 的指令随作业传递，成为远程 Agent 的系统提示（`_write_agent_doc` 不写任何东西，systemone 先例）。
- *插件的 stdio MCP 服务器。* 它们是此机器上的进程；远程运行时无法拨号它们。驱动器将每个报告为 `system/mcp_unavailable`，基座不为它们列出工具。URL 服务器在远程 Agent 上配置，其 bearer 传至厂商（qoder：Vault 凭据，每会话创建并随之删除）；基座的描述说明这一点，因为 harness 所有者的令牌随后存于厂商处。
- *环境*（`runner/environments.py`）。该层在此沙箱中以只读挂载；远程会话仅看到驱动器上传的内容。命名环境的轮次在任何进程启动前被拒绝（`turn()`，400），而非无环境运行。
- *7，凭据的 `CHECKPOINT_EXCLUDE`。* 驱动器不写任何凭据：它持有中继的占位符。它确实写的是 `.harness/<backend>/state.json`，即它在远程创建的所有东西的 id，该文件必须随检查点传递（另一沙箱上的继续和删除依赖它）。

**必须回答，每项在 PR 描述中且每项有测试：**

- *输入文件。* `_write_input_files` 将它们落在运行时看不到的工作区。驱动器上传已