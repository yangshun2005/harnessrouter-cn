# 变更日志

Unified Harness Protocol 的所有重要变更。

## 一致性测试套件 2026.10.4.post9 (2026-10-06)

协议、Schema 或合规服务器获得的任何判定均无变化。

一项检查在失败消息中引用了服务器的回答，却无法读取它。十八条消息——由九项 Environments 检查中的八项和全部十项 Memories 检查（在检查本身或其共享的辅助函数中）触达——由 `r.text` 构建，而客户端结果有 `body`、`json` 和 `header()`，却没有 `text`。消息仅在断言失败时才会被求值，因此对合规服务器不会暴露任何问题。面对真正拒绝调用的服务器，该检查抛出 `AttributeError`，测试套件将其记为 ERROR，"检查本身出错了"，而服务器关于拒绝原因的说明从未进入报告：一次真正的不合规被读成了测试套件的缺陷。现在结果有了 `text`，即解码后的正文且永不抛出异常，这些检查会以状态码和服务器的原话 FAIL。由 @Jiaqiiii523 报告并修复（#411），附带一项测试，它读取这些检查并拒绝结果不具备的任何属性；该测试现在还跟踪从辅助函数返回的回答和直接从调用读取的回答，两者合计占检查所读回答的三分之一。

报告会标明生成它的测试套件。`suite_version` 是版本的第二个副本，自 `2026.9.12.post4` 以来从未变动，因此 `2026.10.4` 到 `2026.10.4.post8` 期间写的每份报告都带有那个更旧的名字。现在版本只在包中写一次，构建从那里读取。`reports/` 下发布的报告没有一份是在该窗口期内写的。

## 2026-10-04（同日修补）

对 `2026-09-28` 的增量：在上一版本下有效的每个请求和对象在此版本下依然有效。

**2026-10-04 的补丁，同一版本。** 记录的 `content` 是有序的部件列表，在每次读取时始终如此（[Memories §4.3](versions/2026-10-04/memories.md#43-content)）。该章当天早上首次发布时说 `content` 是文本或一个文件引用，这无法在其所示内容旁放一张图片，也无法容纳含文件的一轮对话。定义了两类部件：`text` 和 `file`；模态是文件部件的媒体类型，而非部件的种类，因此新增一种模态无需在此改动，任何其他种类都以 `x.` 为前缀并原样携带。文件部件可以携带代表它的文字（`text`，配合 `text_source`），这正是按文字召回所找到的；部件可以携带说出它的 `role`，因此一轮对话也以同样的形态被观察。在写入时，字符串仍是一个文本部件的简写。文件部件的字节在 `GET /v1/memories/{id}/records/{rid}/content/{index}` 读取（原为 `/content`）。提供者的能力文档说明它在 `content` 中保留什么（`media`、`bytes`、`describes`），取代 `files` 标志，写入提供者不保留的媒体类型会以 `memory_unsupported` 被拒绝，绝不会存储时不带其文件。Schema：`MemoryContentPart`；`MemoryRecord.content` 是数组。测试套件 `2026.10.4.post1`：ME-02 期望该列表，ME-09 演练部件、顺序、拒绝，以及一个文件部件连同其描述被保留并逐字节读回，或按提供者声明被拒绝。没有任何实现是针对当天早上的文本发布的。

**2026-10-04 的第二个补丁，同一版本。** 记忆的 `id` 是不透明的。Schema 曾要求 `hmem_` 前缀，而一个记忆本就是它已维护的树（自己的 workspace、自己的文件夹）的服务器，只能通过第二个 id 空间和两者之间的查找来满足。`hmem_` 仍是自行铸造 id 的服务器的前缀。Schema：该模式已从 `Memory.id`、路径参数和 harness 的条目中移除。测试套件 `2026.10.4.post2`：ME-01 不再断言该前缀。

**2026-10-04 的第三个补丁，同一版本。** `recall` 搜索该记忆**及其下调用者可读的一切**；当天早上的文本让它作用于单个记忆。搜索是调用者找到某物存放位置的方式，因此每个结果现在携带 `memory`（`id`、`name`）：即出发的位置。它绝不搜索祖先。`depth` 缩小范围（`0` 表示仅该记忆；它过去是扩大范围，默认一个记忆）。服务器可以限制一个问题覆盖的记忆数量，并在 `degraded` 中说明。`list`、`get`、`history` 和查询仍作用于单个记忆。提供者能力 `recall.max_depth` 已移除：子树由服务器自行组合。Schema：`MemoryRecall.results[].memory` 为必填。测试套件 `2026.10.4.post3`：ME-03 重写。

**2026-10-04 的第四个补丁，同一版本。** 对记忆执行操作的主体使用统一词汇。授权的 principal 曾是"一个不透明的带类型标识符"，其种类由各服务器自选，而记录的 `written_by` 仅以示例（`harness`、`consolidator`）命名其种类；两个实现已分别将人标记为 `member` 和 `user`。现在四种种类同时服务于两处：`member`、`harness`、`group`（持有授权，从不写入）和 `provider`（写入，从不持有授权）；服务器自己的种类以 `x.` 为前缀。`consolidator` 已移除：一次整合的记录由 `provider` 写入，旁边带 `consolidation_id`，而派生自观察的记录携带 `on_behalf_of`。凭证不是一种种类。Schema：`MemoryWriter`，以及授权 `principal` 上的模式。测试套件 `2026.10.4.post4`：ME-02 检查写入者的种类。

**2026-10-04 的第五个补丁，同一版本。** 人和 agent 是同一种类，访问只有一种机制。第四个补丁将 `member`（人）和 `harness`（agent）列为两种种类；它们是同一种，即 `member`，而一个 member 是两者中的哪一个，是其 `type`（`human`、`agent`），这是没有任何规则依赖的属性。由此，harness 不再携带一个附带 `access` 的已附加记忆列表来缩小授权：agent 像人一样，在其被授予的记忆上触达被授予的内容。`/v1/harnesses/{id}/memories` 读取 agent 的 principal 及其被授予的内容，并设置两项：`default_memory_id` 和 `observe`。agent 仅在有人添加它时才加入组。种类现为 `member`、`group`，以及仅对写入者的 `provider`。Schema：`HarnessMemories` 重塑，`HarnessMemorySettings` 新增，`HarnessMemoryEntry` 移除。测试套件 `2026.10.4.post5`。

**2026-10-04 的第六个补丁，同一版本。** 一张图。实体及它们之间的关系不是记录之外的第二套结构：实体是一条记录（新核心类型 `entity`），引用是一条边，而有话要说的关系是一条 `fact`，它引用其 `subject` 和 `object`，因此它像任何记录一样有写入者、时间、历史和来源。定义了五个引用名称（`derived_from`、`part_of`、`about`、`subject`、`object`）。一次读取 `POST /v1/memories/{id}/graph`，围绕一条记录或针对一个记忆，将记录作为节点、引用作为边返回，对每个提供者都相同；提供者在 `graph.entities`（`derived`、`stated`、`none`）中声明其实体来自何处。此外：agent 的组成员资格遵循与人的相同规则（第五个补丁禁止服务器自行将 agent 放入组，这在每个新人都加入一个组的情况下并非平等地位）。Schema：`MemoryGraph`、`MemoryGraphRequest`、`MemoryRecordRef`。测试套件 `2026.10.4.post6`：ME-10。

该章的开篇、§1、agent 工具列表（`memory_graph`）和 §14 随后与上述六个补丁对齐；无规则变化。

**2026-10-04 的第七个补丁，同一版本。** 记录可以携带 `title`：一行命名它的文字，与它所说的内容分开（文档的标题、实体的名称、事实的陈述）。它是可选的，至多 300 个字符，与内容一起被搜索匹配，并像任何其他字段一样被修订。带标题的记录可以有空内容。`entity` 的名称即其标题。Schema：`MemoryRecord.title`。测试套件 `2026.10.4.post7`：ME-02 检查标题与内容分开读回且可单独存在。

不带 `around` 的图读取从该记忆**及其下方内容**的记录开始，即搜索所具有的范围；第六个补丁的文本说仅该记忆，这对记录全保存在其子节点中的记忆会返回空。

一条记录至多命名一个 `subject` 和至多一个 `object`；其中任一出现第二个会以 `memory_invalid`（422）被拒绝。之所以说明，是因为两个实现必须就此达成一致：引用网关保留了这样一条记录，而图引擎拒绝了它。

另一个团队基于本章构建的第一个服务器有两处与参考网关读法不同，都是本章的错，现均已说明一次：`supersedes` 是一个版本所替换版本的 NUMBER（示例显示的是记录 id，而 Schema 两者都允许）；以及读者不可跟随的引用是 `{ record_id, available: false }`，不带 `memory_id`（本章返回了记忆的 id，这告诉读者该记忆存在）。

**跟随来源的记录（§5.4），由第二个服务器发现。** 一个将文档显示为每节一条记录的服务器，不能让某一节通过记忆被修订或遗忘：文档的下一次保存会撤销它。本章对此无说法，因此该服务器的拒绝未定义，而一个被保留的文件也被同样的拒绝所困。现在：此类记录携带 `follows`（其来源的 `kind`、`id`、`name`），对其的 `revise` 和 `forget` 会以 `memory_unsupported` 被拒绝，并说明来源在何处更改，客户端两者都不提供。某人保留的文件不是此类记录：其标题和其行是其写入者的陈述，它像任何其他记录一样被修订和遗忘；遗忘它会把文件从读取中移除，擦除它会移除字节。Schema：`MemoryRecord.follows`，可选。

**调用者被授予了什么（`GET /v1/memories?granted=true`），由第二个服务器发现。** 不带 `parent` 的列表说明调用者从何处进入树，而没有任何东西说明在其下方何处它可以做更多。一个运行 agent 而其记忆由另一个服务器维护的服务器，从入口列表构建 agent 的工具，而一个读取某记忆并写入其某个子节点的 agent 没有被提供可写入的工具。`granted=true` 列出调用者可读但其父节点不可见的每个记忆，以及那些其下可做之事与父节点不同的每个记忆，并附上其在每个记忆上的权限。它以调用者可做什么来陈述，而非以访问如何维护来陈述，因此一个通过角色让成员进入的服务器，与一个按记忆维护授权的服务器回答相同。§9 将其命名为 agent 工具所构建自的东西。一致性：ME-01 询问它。

- **Memories**（[Memories](versions/2026-10-04/memories.md)），Harness Memories 子协议，在 `memories` 能力之后的每个类别中都是可选的。记忆是树中的命名节点：它持有记录，并可以有子记忆。授权给予 principal 对某节点的权限（`read`、`write`、`create`、`delete`），该节点的后代继承它，直到 `restricted` 节点；调用者不可读的内容，其回答与不存在的内容完全相同。
- 记录以两种方式写入（`observe`：原始片段，提供者决定派生什么，并可以带作业回答 `202`；`remember`：一条如所述的记录），以两种方式移除（`forget`：关闭，保留痕迹；`erase`：内容消失，附上无法触达内容的列表）。变更追加一个版本；`history`、`as_of` 和命名快照读取过去。
- `recall` 作用于单个记忆，按含义、文字和字段，并回答调用者可读的父节点和子节点，因此 agent 自行沿树上下行走。提供者未做的事在 `degraded` 中说明；`abstain` 说明没有返回内容回答了该问题。命名查询，以及提供者提供时的自由查询，以提供者自己的语言运行，通过语句无法撤销的手段限制在调用者可读的范围内。
- 服务器不保留自己的记忆：记录由服务器所连接的**提供者**维护，在一个契约之后，每个提供者在其能力文档（`GET /v1/memories/providers`）中说明它做什么：隔离、派生、信号、历史、擦除、查询、类型。
- harness 在 `PUT /v1/harnesses/{id}/memories` 处附加到记忆；附加即授权。任务在 `metadata.memory` 中再命名一个。会话在四个时刻触达记忆：预热、agent 的工具、一轮之后的观察，以及整合，其运行可见、有界且可逆。
- 扩展记录类型携带自己的 Schema 和操作，用于不只是文本的资源。
- Schema 中的新对象：`Memory`、`MemoryRecord`、`MemoryRecall`、`MemoryGrant`、`MemoryProvider`、`MemoryConsolidation` 及其请求；`/v1/memories` 和 `/v1/harnesses/{id}/memories` 下的 23 条路径。新错误码：`memory_not_found`、`memory_forbidden`、`memory_record_not_found`、`memories_not_attached`、`memory_invalid`、`memory_unsupported`、`memory_busy`、`memory_unavailable`。
- 一致性测试套件 `2026.10.4`：ME-01 到 ME-08 在报告该能力并连接了提供者的服务器上驱动树、记录、召回和提供者的能力文档；没有该能力的服务器跳过它们。它们在一个以托管记忆服务为提供者的参考服务器构建上通过，8 项全过。测试套件以单个凭证运行，因此 principal 之间的隔离尚未被它检查。

## 2026-09-28（2026-09-29 修补）

对 `2026-09-12` 的增量：在上一版本下有效的每个请求和对象在此版本下依然有效。

**2026-09-29 的补丁，同一版本。** 任务侧对环境的引用是 `metadata.environment`，而非顶层请求字段：任务表面是一个 Responses 请求，而 `metadata` 是其扩展点，`harness_id` 已经在那里传递（[Tasks §1.2](versions/2026-09-28/tasks.md#12-selecting-the-harness)、[Environments §5](versions/2026-09-28/environments.md#5-attaching-an-environment)）。响应在 `session_id` 旁报告 `metadata.environment`。harness 和 session 对象将 `environment` 保留为普通字段。已发布的章节曾将该字段放在顶层一天；Schema、一致性测试套件（`2026.9.28.post1`：EN-05 在 `metadata` 中发送它，EN-07 断言响应回显；`post2`：EN-02 比较返回的字节，`post1` 无法读取）和参考服务器（0.26.7）一起更改。参考服务器在 `post2` 上通过整个类别，84 项全过。

**2026-09-29 的第二个补丁，同一版本。** 构建的记录说明哪个步骤正在运行（`stage`），并在运行时携带其日志，服务器必须在步骤结束时写入它，以便客户端显示进度（[Environments §4](versions/2026-09-28/environments.md#4-builds-and-versions)）。一个可选的 `GET /v1/environments/packages/check` 在任何构建之前回答管理器的镜像仓库对某个 spec 的说法（`EnvironmentPackageCheck`），以便客户端在输入处拒绝拼写错误。构建运行时环境的 `status` 为 `building`，无论哪个版本处于活动状态；任务在此期间继续读取活动版本。测试套件 `2026.9.28.post3`：EN-09 在提供该检查的服务器上演练它；参考服务器 0.26.10。

- **Environments**（[Environments](versions/2026-09-28/environments.md)），Environments 子协议，在 `environments` 能力之后的每个类别中都是可选的。环境是一个项目的文件和已安装依赖，构建一次并以只读方式挂载在命名它的每个会话中的固定路径，与会话自己的可写工作目录并列。该章定义了对象（`henv_`）、按路径的文件和整项目导入、带活动指针和回滚的构建与版本、harness 对象上的 `environment` 字段和任务上的 `metadata.environment`、会话所见（挂载、变量、指令、记录），以及错误。
- session 对象携带 `environment`（[Sessions §3](versions/2026-09-28/sessions.md#3-inspecting-a-session)）。
- 新错误码：`environment_not_found`、`environment_not_ready`、`environment_busy`、`environment_exists`、`environment_invalid`、`environment_unavailable`。
- 一致性测试套件 `2026.9.28`：检查 EN-01 到 EN-08（EN-07 运行一个读取环境并尝试写入它的任务），在没有该能力的服务器上跳过。

## 一致性测试套件 2026.9.12.post4 (2026-09-27)

- 公共 fixture 在 `https://uhp-fixture.harnessrouter.ai/mcp` 回答，这是测试套件的新默认值；post3 指向容器自己的地址。

## 一致性测试套件