# Harness 基准测试

一行的测量方式及判定规则见 [scripts/benchmark/README.md](../scripts/benchmark/README.md)。一个任务即一个会话；由测试包自带的评分器判定；所有数字均读取自回合记录。

## 提供商：deepseek，测试包：spreadsheetbench

| Harness | 模型 | 任务数 | 已解决 | 奖励 | 总耗时（累计） | 耗时中位数 | 工具调用 | 失败调用 | 新增输入 | 缓存输入 | 输出 | 服务方 | 备注 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| cline | deepseek-v4.1-flash | 47 | 36 (77%) | 0.77 | 6882 s | 108 s | 736 | 121 (16%) | 1.20M | 19.74M | 1.21M | deepseek | 2 次运行测试包无法评分（任务未通过其自带评分器），已排除；50 次运行中 50 次未报告服务模型（该处规则 2 无法验证）；1 次运行属于发现项，不计入 |
| dsh | deepseek-v4.1-flash | 46 | 36 (78%) | 0.78 | 6506 s | 100 s | 721 | 113 (16%) | 761k | 12.32M | 752k | deepseek | 2 次运行测试包无法评分（任务未通过其自带评分器），已排除；50 次运行中 50 次未报告服务模型（该处规则 2 无法验证）；2 次运行属于发现项，不计入 |
| opencode | deepseek-v4.1-flash | 48 | 39 (81%) | 0.81 | 5515 s | 75 s | 563 | 67 (12%) | 666k | 11.64M | 709k | deepseek | 2 次运行测试包无法评分（任务未通过其自带评分器），已排除；50 次运行中 1 次未报告服务模型（该处规则 2 无法验证） |
| pi | deepseek-v4.1-flash | 47 | 40 (85%) | 0.85 | 3605 s | 48 s | 526 | 89 (17%) | 484k | 7.53M | 546k | deepseek | 2 次运行测试包无法评分（任务未通过其自带评分器），已排除；50 次运行中 1 次未报告服务模型（该处规则 2 无法验证）；1 次运行属于发现项，不计入 |

发现项，即由其他连接、以其他模型提供服务，或访问了网络、在工作区外查找任务的运行：仅列出，不计分。

- cline x deepseek-v4.1-flash 290-1：访问了网络：fetch_web_content；fetch_web_content
- dsh x deepseek-v4.1-flash 79-7：在工作区外查找任务：bash: grep -rl "Where the answer goes" /opt/harnessrouter /data/agent-tools 2>/dev/null | head -20
- dsh x deepseek-v4.1-flash 73-45：在工作区外查找任务：bash: ls /data/agent-tools/dsh-venv/lib/python3.12/site-packages/deepseek_harness_runtime/ 2>/dev/null; echo "---"; find
- pi x deepseek-v4.1-flash 73-45：在工作区外查找任务：bash: cd /data/workspaces/hsesscb99ac8612e64459b6aa2b6b071fe991 && find / -name '*.xlsx' 2>/dev/null | grep -v site-pack

## 运行说明

- SpreadsheetBench Verified，dataset.json 的前五十个任务；deepseek-v4.1-flash 通过一个自定义 OpenAI 格式连接，所有 harness 均禁用网络工具（opencode 和 pi 为硬开关，cline 和 dsh 为指令）；WORKERS=2，TASK_CAP_S=900。
- 自托管 CE：pi 和 dsh 使用 0.18.0（2026-09-18）；cline 使用 0.18.4（2026-09-19），该版本包含 #209，因此其输入 token 为新增输入；opencode 使用 0.18.4 并应用了 #216（2026-09-19），因此其输出 token 包含推理内容。早先的两列数据已搁置：cline 使用 0.18.0（50 中 36，输入按总量报告为 18.87M）和 #216 之前的 opencode（48 中 37，输出报告为 81k 可见文本）。
- 两个版本的控制台代理都会在五分钟时切断同步回合（当时 #214 处于开启状态）；20 次运行超过该时限，在会话列表中找到并从记录中评分。
- 任务 130-9 和 283-32 未通过其自带评分器（一个工作表名含逗号；一个整列范围），已从所有行中排除。