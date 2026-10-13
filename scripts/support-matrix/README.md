# 支持矩阵套件

测试框架必须证明什么、以及如何判定一行，见
[docs/harness-verification.md](../../docs/harness-verification.md)。本文件讲如何运行。

以单一用户身份驱动控制台，对每个测试框架及其菜单提供的每个模型，在一个会话中运行五个
场景：首轮对话、追问、切换到该框架的另一个模型（再切回）、产物（任务必须生成的文件，
通过转录中的文件卡片检查）、以及回收（会话的沙箱通过内部回收路径被有意释放，随后追问
必须能回忆起第一条消息：历史在检查点往返后存活）。
结果按每个框架 x 模型一条 JSON 记录，包含每个场景的结果、耗时和原因；`fill-connection.py` 为每条记录
打上其会话实际使用的连接；`render.py` 将记录转换为 `docs/support-matrix.md`（末尾的 Browser 部分来自 docs/browser-column.json）。

```
export BASE=https://your-instance HR_USER=harnessrouter HR_PASS=... PROVIDER=tokenrouter
HARNESSES=claude-code,codex,opencode,pi RESULTS=results-A.json LOG=log-A.txt node run.mjs
HARNESSES=hermes,dsh,qwen,cline       RESULTS=results-B.json LOG=log-B.txt node run.mjs
HR_API_KEY=... python3 fill-connection.py results-A.json results-B.json
python3 render.py <(jq -s add results-A.json results-B.json) > ../../docs/support-matrix.md
```

需要 `playwright`（在 `run.mjs` 旁执行 `npm i playwright`，然后 `npx playwright install chromium`）。
可断点续跑：已记录的配对会被跳过，因此被终止的 worker 重新启动后会继续；记录中带有 `error`（运行器自身失败）的配对会重跑。要重跑失败的配对，
删除其记录后重新启动。`PROVIDER` 是表格的标签：每个 provider 运行一次，模型映射将该 provider 服务的
每个模型指向其集成（一次一个 provider，因为一个模型只有一个
集成）。`MODELS`（逗号列表）将运行限制
到某些模型，这是复现单个失败配对并开启追踪的方式。对使用自签名证书的实例，设置
`IGNORE_TLS=1`。

失败的行必须带有复现的 provider 错误文本；已验证的列表绝不从另一个实例继承，
因为每个实例通过自己的路径到达 provider。在排除某个模型前，先重测单纯的 `incomplete`。

## 家族巡览

一次对话，依次覆盖每个模型家族，针对一个交付物（见
docs/harness-verification.md 中的 "The family tour"）。通过控制台，与 `run.mjs` 一样：

```
export BASE=https://your-instance HR_USER=harnessrouter HR_PASS=...
HARNESS=cheetahclaws RESULTS=tour-cheetahclaws.json node family-tour.mjs
SID=hsess... HARNESS=cheetahclaws node family-tour.mjs     # retest a person's own conversation
```

`FAMILIES`（模型 id 的逗号列表，每个家族一个）覆盖默认列表；实例不服务的模型会被跳过并
记录为未提供。每个家族都通过时退出码为 0。

## 插件矩阵

`plugins/run-matrix.py` 在每个 base 上验证插件路径，一次一个 base：创建一个测试框架，
仅使用一个 Agent Plugins 包（`plugins/fixture/`：一个清单、一个技能（其 token 模型必须复述）、一个手写协议实现的 stdio MCP 服务器），
其直接列表上没有任何内容，每列运行一个真实任务（技能、stdio MCP、一个接受自由格式对象列表的 stdio 工具、SSE MCP、streamable-HTTP MCP、针对公共探针的远程两者），
且答案仅根据探针 token 判定。`rows` 列的 token 来自工具本身：
只有当传给它的 rows 带有其字段时它才作答，否则会说出它收到了什么，
因此 agent 对自己调用的陈述绝不是判据。

    python3 plugins/run-matrix.py --base-url https://your-instance/api/harness --api-key "$KEY" \
        --bases aider --out plugin-results.json

自 2026-09-18 起保留在仓库中，此前临时文件夹中的副本在评审中途被清空。

## 浏览器列

`plugs/browser.py` 在除 System One 外的每个 base 上通过 API 运行浏览器插件，
针对一个用平实语言描述的任务（打开 example.com，跟随其链接，报告你落在哪里），并根据追踪判定每个
base：浏览器导航并点击了，答案说出了它到达的页面，
浏览器会话被停止并计费。失败的 base 会重测一次。提示词不指定任何工具；
agent 依据的是其指令文件中测试框架的 Browser 部分。

    python3 plugs/browser.py --base-url https://<instance>/api/harness --api-key "$KEY" \
        --out browser.json --md browser.md            # every ready base except systemone
    python3 plugs/browser.py ... --bases codex,pi     # a few bases
    python3 plugs/browser.py ... --model gpt-5.4      # one model on every base

表格（`--md`）是 docs/support-matrix.md 的 Browser 部分；运行的发现会像其他每一列一样
写入 docs/support-matrix-notes.md。

## 思考列

`thinking/run-column.py` 在每个 base 上探究一轮对话所要求的级别是否会改变模型的思考量：
每个 base 一个测试框架，使用一个有级别的模型，同一任务分别在无级别以及
`none`、`low` 和 `high` 下运行，根据该轮自身的记录（`reasoning.applied`）和 provider 的
思考 token 计数（`usage.output_tokens_details.reasoning_tokens`）判定，在 provider 不提供计数时
以输出 token 代替。不读取 agent 对自己的陈述。

    python3 thinking/run-column.py --base-url https://<instance>/api/harness --api-key "$KEY" \
        --out thinking.json                           # every base, its first model with levels
    python3 thinking/run-column.py ... --bases codex,pi --model gpt-5.4 --repeat 2

`thinking/probe.py` 是发现 runner/reasoning.py 中级别的方式：在每个级别、以及每个 provider 自己的开关下
各问一个问题，直接打到 provider 的 API，读取其 usage。添加模型时
重新运行它；只有被观察到接受的级别才会进入表格。

    THINKING_PROBE_BASE=https://openrouter.ai/api/v1 THINKING_PROBE_KEY=... \
        python3 thinking/probe.py chat:openai/gpt-5.4 responses:openai/gpt-5.4 messages:anthropic/claude-haiku-4.5

## 环境列

`python3 environments/column.py --base-url ... --api-key ... --environment henv_...` 在一个已构建环境上
为每个 base 运行三个任务（pip、npm、apt），并根据追踪（无安装）、答案（包在环境下的路径）
和会话记录判定每个任务。见 docs/support-matrix-notes.md。