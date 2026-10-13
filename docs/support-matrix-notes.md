# 支持矩阵说明，自托管实例，2026-09-06

[support-matrix.md](support-matrix.md) 中的表格由 `scripts/support-matrix` 针对自托管测试实例（单容器、owner 信任、runner 与网关同机）生成，每次一个提供商，三个 worker 各带一个 harness（免费层密钥用一个 worker），每个 harness × 模型对跑五个场景：首轮、追问、列内模型切换、产物、回收。失败的行重跑一次，备注中保留首次结果；未从托管运行继承任何内容。

## 版本

本次运行从 v0.13.5 开始，到 v0.13.13 结束。其间每个版本都源于下述某项发现，并在下一列之前通过实时轮次门控部署到实例：0.13.5（本地 blob 存储按前缀列出）、0.13.6（读缓存，"refused"一词并非密钥拒绝）、0.13.7（同一账号下 Codex 历史保持完整，已完成的轮次释放其进程句柄）、0.13.8（Google 密钥可保存，qwen 移除 gpt-5.3-codex，通过 URL 重新打开的任务保留其模型，broker 重发 Google 请求时去掉被拒字段，自托管沙箱通过 loopback 访问 broker）、0.13.9（opencode 的 base 携带 /v1）、0.13.10（relay 的 base 携带其 API 版本）、0.13.11（claude CLI 将其剥离）、0.13.12（无法恢复的检查点中止该轮次）、0.13.13（owner 信任像 broker 一样规范化 Azure base）。

## 四个新 base 只提供它们能可靠运行的内容（2026-10-12，0.33.0-rc.5 至 rc.11，审查实例）

此后的规则，对每个 base 适用：base 只提供经测量在每个维度都能可靠运行的模型，而一个属于模型自身且重复出现的失败（在其复测中，或在另一个 base 的同一维度中）会使该模型从该 base 下架。由产品引起的失败则修复后重新测量，绝不用于下架或保留模型。

**测量内容，按 base 和模型。** 每个 base 能触及的每个模型都跑了两遍五个控制台场景（0.33.0-batch3.3 和 batch3.7，七个连接）以及家族巡览。然后每个留在 base 上的模型通过审查实例的 API 运行：插件技能、stdio 上的 MCP 工具、自由格式行工具、SSE 和 streamable HTTP（仅当模型请求了所服务的内容时该行才通过）、浏览器插件（导航和点击，答案需指明页面），以及一个带印章的单页 deck，该印章任何其他轮次都无法生成，下载、打开并检查印章。每个首次失败都复测一次。

**每个 base 提供的内容。** mimo 47、jcode 50、opensquilla 51、nanobot 51。每个提供的模型在发布构建上都通过了上述每个维度，仅有一个例外如下所述。

**下架的模型及其表现**（相同文本位于 gateway/app.py 中 `_NOT_OFFERED` 上方）：
- mimo：claude-opus-4.7（拒绝场景提示词，两次）、llama-3.3-70b（答非所问）、llama-4-maverick（将工具调用写成文本，两次）、qwen3.8-flash（混淆回复）、gpt-5.6-luna（超出轮次限制）、hunyuan-4-preview（巡览中无 deck）、nemotron-3-super、kimi-k3（工具调用写成文本）、mistral-medium-3.5（将 `<use_skill>officecli</use_skill>` 写成文本，无 deck，三次中两次）、gemini-3.5-flash 和 gemini-3.6-flash（空白 deck 且轮次结束，各两次）、gpt-5.4-mini（被要求使用指定的 MCP 工具时却运行插件的技能）。
- jcode：grok-build-0.1 和 hunyuan-4-preview（工具循环至上限）、kimi-k3（无文件，两次）、gemini-3-flash-preview（首轮退化）、nemotron-3-super、llama-3.3-70b（将 deck.pptx 写成一个不是 deck 的文件，两次）。
- opensquilla：claude-opus-4.7（拒绝提示词，两次）、grok-4.20 和 llama-3.3-70b（无文件）、hunyuan-4-preview（答非所问）、nemotron-3-super。
- nanobot：grok-build-0.1（exec 循环）、kimi-k3 和 llama-3.3-70b（无文件）、grok-4.5（流停滞）、nemotron-3-super。
- nemotron-3-super，在全部四个上：不遵循技能（mimo 两次、jcode、opensquilla），且制作的 deck 缺少所要求的文本或没有 deck（mimo、jcode、nanobot）。

**测量发现的产品缺陷，均已修复并重新测量**（全部在本版本中，共十一个）：
- jcode 的工具调用到达控制台时没有参数：runner 在 `tool_start` 时发送每个调用，并丢弃了携带参数的 `tool_input` 片段。现在保持至完成；一个控制台检查（scripts/support-matrix/ui-tool-rows.mjs）打开 Bash 行并找到命令。
- 在繁忙的单容器实例上等待槽位的轮次先读作 `failed`，然后 `completed`：孤儿清理将没有心跳的 `starting` 会话误认为已死。接受时写入会盖上心跳时间戳，`_turn_slot` 在轮次等待期间续期。
- mimo 可以触及轮次连接之外的提供商：模型的 `actor` 子代理命名为 `xiaomi/mimo-v2.5`，MiMo 的服务拒绝了密钥，已完成的 deck 轮次失败为"你的密钥被拒绝"。opencode 和 mimo 配置现在设置 `enabled_providers: ["hr"]`。
- mimo 的 23 个内置技能与 harness 的竞争：其 pptx/docx/xlsx/pdf-official 需要 python-pptx 等，此处的沙箱都无法安装。`MIMOCODE_DISABLE_BUILTIN_SKILLS=1`。
- jcode 自带的 `browser` 需要 Firefox 扩展桥接，在沙箱中从不连接；选择它的模型循环了两小时（459 次工具调用）。在每个 jcode 轮次上关闭，连同 `macos_computer_use`、`gmail` 和 `maintainer_feedback`。
- 代理在会话首轮提交的文件从未被交回（收集游标设置在该轮次的收集时，即提交之后）。现在设置在轮次开始时。
- 工作区内的 LibreOffice 配置文件（模型通过渲染来检查其 deck）填满了轮次交回的 25 个文件并把 deck 挤了出去。配置文件的 `user/` 树不是交付物。gpt-6.1-sol 曾因此短暂从 opensquilla 下架，现已重新提供并通过。
- opensquilla 的工具行在轮次运行时没有参数（其进度事件不携带参数）。runner 的 relay 现在按调用 id 保留模型发送的每个调用的参数，行打开时带有它们；转录在轮次结束时填充 relay 未看到的任何调用。在轮次中途于控制台检查（scripts/support-matrix/ui-tool-rows-live.mjs）。
- opensquilla 在 GPT 模型上用中文、法语或西班牙语回答英文提示词（gpt-5.4：8 次全为中文）：其自身系统提示词的三条回复语言规则。runner 将它们从安装的模板中移除；英文提示词随后得到英文回答（8 次全对），中文提示词得到中文回答。其 `cron` 工具（其调度器在一次性轮次中从不运行）也已关闭。
- 每个 opensquilla 答案都重放了此前的对话：恢复会话的转录包含每个轮次，轮次结束时的重放读取了全部内容。现在从本轮次的提示词开始，套件的追问会判定重复首轮的答案为失败。
- 提供商拒绝的 hermes 轮次只显示"↻ Resumed session …"：relay 将提供商的拒绝保留在路由上，当 hermes 未说其他内容时，轮次的错误即为该拒绝。

**例外。** nemotron-3.5-lightning 是这些 base 在审查实例上通过 OpenRouter 触及的唯一提供的模型，而该实例的 OpenRouter 密钥在 rc.9 重新测量期间达到支出上限：其 deck 在那里无法运行。它在 rc.7 上通过了每个维度。

## Claude Haiku 5.5（2026-10-07）

在 0.31.12 中与 Haiku 4.5 一同添加，位于每个提供 Haiku 4.5 的 base 上（aider 除外），每个提供商使用自己的 id，读取自当日供应商列表：Anthropic `claude-haiku-5-5`，OpenRouter 和 Vercel `anthropic/claude-haiku-5.5`（OpenRouter 以 `anthropic/claude-haiku-5.5-20261007` 提供，同一模型），Bedrock `us.anthropic.claude-haiku-5-5`（来自 Anthropic 的 Bedrock id 和其他 5.5 模型的推理配置文件形式；未测量 Bedrock 连接）。TokenRouter 和 llmtr 未列出它，因此在 `_TOKENROUTER_NO_CHANNEL` 中。同一次读取在 TokenRouter 和 llmtr 的列表上发现了 Opus 5.5；两者现在都路由它。

其价格按提示词长度设定（Anthropic 的定价页）：最多 100,000 提示词 token 时，输入 $0.10、缓存读取 $0.01、缓存写入 $0.125 和 $0.20、输出 $0.50 每百万 token；超过 100,000 时各五倍。此实例不保留自己的价格表。

通过侧容器的控制台测量（0.31.11 加此更改），每个连接一列，Haiku 4.5 作为切换伙伴：

- Anthropic：16 个 base 中 15 个通过全部五个场景。aider 在回收场景失败（回收后它回答了一个从未被要求的词），三次中两次，因此 Haiku 5.5 被排除在 aider 的目录之外。Claude Code 在镜像固定的 CLI（2.1.280）上运行它；无版本提升。
- Vercel AI Gateway：17 个 base 中 17 个通过全部五个。
- OpenRouter：16 个 base 中 16 个通过全部五个（Claude Code 无法使用 OpenRouter 连接）。
- 与 Haiku 5.5 无关、在同一 base 上对 Haiku 4.5 出现的发现：通过 Anthropic 连接，agentzero、aider、cline、grok、kimi、minimax 和 qwen 不读取提示词缓存（它们不要求 Anthropic 缓存，因此每次调用都支付完整输入价格）；cline 和 dsh 不报告所服务的模型，因此同模型规则无法判定其轮次。

思考。探针（`scripts/support-matrix/thinking/probe.py`）在 Vercel 和 OpenRouter 上，chat 和 Messages：每个级别都被接受，包括 `none`（OpenRouter 拒绝关闭 Opus 和 Sonnet 5.5，但不拒绝此模型），high 和 xhigh 比 medium 思考更多；low 在探针问题上思考与 none 一样少。在 Messages 上，OpenRouter 按名称拒绝它的 `thinking: between_tools`，两条路由都接受 `disabled`，因此 runner 现在只为 Opus 和 Sonnet 5.5 发送 `between_tools`。OpenRouter 上的思考列（提供商计算思考 token）：hermes、pi、opencode 和 goose 通过（none 不花费，low 少于 high）。通过 Anthropic 连接（不报告思考计数），hermes 通过，Claude Code 和 opencode 记录每个级别都已应用，仅未通过输出 token 替代检查（自适应 low 花费与 none 大致相同），pi 不记录任何级别已应用；pi 在那里对 Sonnet 5.5 也是如此，因此那是 Anthropic 连接上的 pi，而非此模型。同一探针发现通过 TokenRouter 的 Opus 5.5 拒绝所有关闭思考的方式（`disabled`、`between_tools`、预算），因此那里不为其提供 `none`。

## Claude Sonnet 5.5（2026-09-30）

在 0.26.18 中与 Sonnet 5 一同添加，位于每个提供 Sonnet 5 的 base 上，每个提供商使用自己的 id（Anthropic `claude-sonnet-5-5`，Bedrock `us.anthropic.claude-sonnet-5-5`，OpenRouter、Vercel、TokenRouter 和 llmtr `anthropic/claude-sonnet-5.5`，均读取自当日供应商列表）。在 hr-test（0.26.18-rc.1）上通过控制台测量，使用实例的 TokenRouter 连接，即有效模型映射发送它的位置：

- 13 个 base 中 13 个通过全部五个场景（首轮、追问、模型切换、产物、回收）：claude-code、cline、dsh、hermes、opencode、pi、qwen、aider、kimi、openhands、cheetahclaws、goose、omp。Claude Code 在镜像固定的 CLI（2.1.280）上运行它；无版本提升。
- claude-code 上的家族巡览：一个 deck 对话从 base 的默认模型交给 Sonnet 5.5 再交给 Sonnet 5，两轮都完成并产出了 deck；默认列表的其他家族在 Claude Code 上不提供，因此按此跳过。
- 关于切换伙伴而非 Sonnet 5.5 的发现：当列的伙伴强制为 `claude-opus-5` 时，omp 在 TokenRouter 上的切换轮次失败，带有提供商自己的句子（"Claude Code 2.1.257 不支持此模型；需要 2.1.280 或更新版本"），与 Opus 5.5 在 2026-09-22 遇到的同一渠道限制相同。以 Sonnet 5 作为伙伴复测，omp 通过全部五个。Sonnet 5.5 本身在每个 base 上都得到服务。
- TokenRouter 对 `claude-sonnet-5.5` 的请求报告带连字符形式的所服务 id（`claude-sonnet-5-5`）；runner 将其标记为按字符串替换，且是同一模型。

## 列

- **tokenrouter**（实例自有的 TokenRouter 集成，26 个模型）：169 对，832/840。opencode 与 gemini-3.6-flash 因其工具 schema 被拒（`Unknown name "$schema"`）；qwen 与 gpt-5.3-codex 回答文本轮次但工具轮次失败（chat/completions harness 上的仅 Responses 模型；qwen 自 0.13.8 起不再列出它）；Codex 按设计拒绝在另一模型之后使用 gpt-5.3-codex。
- **vercel**（实例的 Vercel AI Gateway 集成，31 个模型）：174 对，857/859。Hermes 因 Vercel 为其声明的 32k 上下文窗口（低于其 64k 最低要求）拒绝 kimi-k2.7-code 和 ling-3.0-flash。
- **azure-openai**（实例自有的 Azure 集成，user-interview 资源，8 个部署）：55 对，267/271，在添加 gpt-5.3-codex 部署并在 0.13.7 发布 Codex 同账号规则之后。在此之前，Codex 无法在 Azure 上模型切换后继续线程（一个部署生成的推理项无法被另一个解析）。剩余：gpt-5.3-codex 的 Codex 家族规则；qwen 对仅 Responses 模型的 chat/completions 调用。
- **openai**（组织密钥，8 个模型）：55 对，267/273。剩余：同样的两个按设计行，以及 gpt-5.6-luna 在回收后用第二条消息的词而非第一条回答（其历史完好）。
- **anthropic**（组织密钥，7 个模型）：49 对，245/245（opencode 的 haiku 首轮一次回答了能力简介，下次尝试回答了词），在该列发现的三项修复之后：opencode 的 Anthropic 客户端需要 base 携带 /v1（0.13.9），承载 cline 和 qwen 的 loopback relay 同样需要（0.13.10），claude CLI 需要再次剥离它（0.13.11）。以任一方式存储的 base 现在服务每个 harness。
- **openrouter**（新密钥，31 个模型）：134 对，669/670。qwen 与 gpt-5.6-luna 在回收后未用第一个词回答。该密钥当天晚些时候达到支出上限；约 13:00Z 之后的任何 OpenRouter 失败是密钥问题，不是产品问题。
- **azure-e2**（捆绑包的 agentstudio-oai-e2 资源，8 个部署）：54 对，267/270，在 base 以 /openai/v1 存储之后。以裸门户端点存储时，文本轮次回答而每个工具轮次都是"Resource not found"：broker 的规范化器在 owner 信任中未运行（0.13.13）。剩余：Codex 在 gpt-5.5 之后也拒绝 gpt-5.3-codex，因此该模型从此不接受切换伙伴；gpt-5.6-luna 的回收答案未命中。
- **google**（免费层 AI Studio 密钥，gemini-3.6-flash）：未测量任何行。每日配额在托管运行和本次运行之间用完（opencode 和 hermes 上 429）；pi