# Harness 支持矩阵

每列的运行说明见 [support-matrix-notes.md](support-matrix-notes.md)。

场景：首轮、同一会话内的追问、会话中途切换模型、产物（任务必须生成的文件）、回收（故意释放沙箱，随后追问必须回忆起第一条消息）。pass = 按预期运行并作答，FAIL = 失败（原因见说明），n/a = 未运行。

## 提供方：agentzero-default

| Harness | 模型 | 首轮 | 追问 | 切换 | 产物 | 回收 | 服务方 | 说明 |
|---|---|---|---|---|---|---|---|---|
| agentzero | claude-fable-5 | FAIL | n/a | n/a | n/a | n/a | My TokenRouter | first：本轮失败：提供方拒绝了请求（finish_reason content_filter）；重测一次；首次尝试：first 本轮失败：提供方拒绝了请求（finish_reason content_filter；缓存占输入的 0% |
| agentzero | claude-fable-5-1 | FAIL | n/a | n/a | n/a | n/a | My TokenRouter | first：本轮失败：提供方拒绝了请求（finish_reason content_filter）；重测一次；首次尝试：first 本轮失败：提供方拒绝了请求（finish_reason content_filter；缓存占输入的 0% |
| agentzero | claude-haiku-4.5 | pass | pass | pass (gpt-6.1-sol) | pass | pass | Anthropic, OpenRouter | 无提示缓存读取（见下方发现）；以 claude-haiku-4-5-20251001 提供（提供方对同一模型的别名） |
| agentzero | claude-opus-4.7 | pass | pass | pass (gpt-6.1-sol) | pass | pass | My TokenRouter, OpenRouter | 无提示缓存读取（见下方发现）；以 claude-opus-4-7 提供（提供方对同一模型的别名） |
| agentzero | claude-opus-4.8 | pass | pass | pass (gpt-6.1-sol) | pass | pass | My TokenRouter, OpenRouter | 无提示缓存读取（见下方发现）；以 claude-opus-4-8 提供（提供方对同一模型的别名） |
| agentzero | claude-opus-5 | pass | pass | pass (gpt-6.1-sol) | FAIL | FAIL | My TokenRouter, OpenRouter | artifact：本轮失败：提供方拒绝了请求（finish_reason content_filter）；recycle：本轮失败：提供方拒绝了请求（finish_reason content_filter）；重测一次；首次尝试：artifact 本轮失败：提供方拒绝了请求（finish_reason content_filter；recycle 本轮失败：提供方拒绝了请求（finish_reason content_filter；缓存占输入的 0% |
| agentzero | claude-opus-5.5 | pass | pass | pass (gpt-6.1-sol) | pass | pass | Vercel AI Gateway, OpenRouter | 无提示缓存读取（见下方发现）；以 anthropic/claude-opus-5.5 提供（提供方对同一模型的别名） |
| agentzero | claude-sonnet-4.6 | pass | pass | pass (gpt-6.1-sol) | pass | pass | Anthropic, OpenRouter | 无提示缓存读取（见下方发现）；以 claude-sonnet-4-6 提供（提供方对同一模型的别名） |
| agentzero | claude-sonnet-5 | pass | pass | pass (gpt-6.1-sol) | pass | pass | My TokenRouter, OpenRouter | 无提示缓存读取（见下方发现） |
| agentzero | claude-sonnet-5.5 | pass | pass | pass (gpt-6.1-sol) | pass | pass | My TokenRouter, OpenRouter | 无提示缓存读取（见下方发现）；以 claude-sonnet-5-5 提供（提供方对同一模型的别名） |
| agentzero | claude-sonnet-5.5-direct | pass | pass | pass (gpt-6.1-sol) | pass | pass | Anthropic, OpenRouter | 无提示缓存读取（见下方发现）；以 claude-sonnet-5-5 提供（见下方发现） |
| agentzero | deepseek-v4-flash | pass | pass | pass (gpt-6.1-sol) | pass | pass | My TokenRouter, OpenRouter | 以 deepseek-v4.1-flash 提供（见下方发现）；缓存占输入的 63% |
| agentzero | deepseek-v4-pro | pass | pass | pass (gpt-6.1-sol) | pass | pass | My TokenRouter, OpenRouter | 缓存占输入的 59% |
| agentzero | deepseek-v4.1-flash | pass | pass | pass (gpt-6.1-sol) | pass | pass | My TokenRouter, OpenRouter | 缓存占输入的 63% |
| agentzero | gemini-3-flash-preview | pass | pass | pass (gpt-6.1-sol) | pass | FAIL | My TokenRouter, OpenRouter | recycle：未带 M1-gemini-3-flash-preview 作答：What exact word did I ask you to reply with in my very first message of this task? Reply with ju；重测一次；首次尝试：recycle 未带 M1-gemini-3-flash-preview 作答：What exact word did I ask you to rep；缓存占输入的 29% |
| agentzero | gemini-3.1-flash-lite | pass | pass | pass (gpt-6.1-sol) | pass | pass | Vercel AI Gateway, OpenRouter | 以 google/gemini-3.1-flash-lite 提供（提供方对同一模型的别名）；缓存占输入的 44% |
| agentzero | gemini-3.1-pro-preview | pass | pass | pass (gpt-6.1-sol) | pass | pass | My TokenRouter, OpenRouter | 缓存占输入的 29% |
| agentzero | gemini-3.5-flash | pass | pass | pass (gpt-6.1-sol) | pass | pass | My TokenRouter, OpenRouter | 缓存占输入的 44% |
| agentzero | gemini-3.5-flash-lite | pass | pass | pass (gpt-6.1-sol) | pass | pass | My TokenRouter, OpenRouter | 缓存占输入的 44% |
| agentzero | gemini-3.6-flash | pass | pass | pass (gpt-6.1-sol) | pass | pass | My TokenRouter, OpenRouter | 缓存占输入的 19% |
| agentzero | gemini-3.7-flash | pass | pass | pass (gpt-6.1-sol) | pass | pass | My TokenRouter, OpenRouter | 缓存占输入的 29% |
| agentzero | gemini-3.8-flash | pass | pass | pass (gpt-6.1-sol) | pass | pass | My TokenRouter, OpenRouter | 缓存占输入的 29% |
| agentzero | glm-5.3 | pass | pass | pass (gpt-6.1-sol) | pass | pass | My TokenRouter, OpenRouter | 缓存占输入的 71% |
| agentzero | glm-5.3-flash | pass | pass | pass (gpt-6.1-sol) | pass | pass | My TokenRouter, OpenRouter | 缓存占输入的 46% |
| agentzero | gpt-5.2 | pass | pass | pass (gpt-6.1-sol) | pass | pass | Custom OpenAI Chat, OpenRouter | 以 gpt-5.2-2025-12-11 提供（提供方对同一模型的别名）；缓存占输入的 60% |
| agentzero | gpt-5.4 | pass | pass | pass (gpt-6.1-sol) | pass | pass | My TokenRouter, OpenRouter | 以 gpt-5.4-2026-03-05 提供（提供方对同一模型的别名）；缓存占输入的 58% |
| agentzero | gpt-5.4-mini | pass | pass | pass (gpt-6.1-sol) | pass | pass | Custom OpenAI Chat, OpenRouter | 以 gpt-5.4-mini-2026-03-17 提供（提供方对同一模型的别名）；缓存占输入的 58% |
| agentzero | gpt-5.5 | pass | pass | pass (gpt-6.1-sol) | pass | pass | Azure OpenAI E2, OpenRouter | 以 gpt-5.5-2026-04-24 提供（提供方对同一模型的别名）；缓存占输入的 56% |
| agentzero | gpt-5.6-luna | pass | pass | pass (gpt-6.1-sol) | pass | pass | My TokenRouter, OpenRouter | 缓存占输入的 61% |
| agentzero | gpt-5.6-sol | pass | pass | pass (gpt-6.1-sol) | pass | pass | My TokenRouter, OpenRouter | 缓存占输入的 62% |
| agentzero | gpt-5.6-terra | pass | pass | pass (gpt-6.1-sol) | pass | pass | My TokenRouter, OpenRouter | 缓存占输入的 66% |
| agentzero | gpt-6-luna | pass | pass | pass (gpt-6.1-sol) | pass | pass | My TokenRouter, OpenRouter | 缓存占输入的 61% |
| agentzero | gpt-6-sol | pass | pass | pass (gpt-6.1-sol) | pass | pass | My TokenRouter, OpenRouter | 缓存占输入的 66% |
| agentzero | gpt-6.1-sol | pass | pass | pass (gpt-6-sol) | pass | pass | OpenRouter, My TokenRouter | 以 openai/gpt-6.1-sol 提供（提供方对同一模型的别名）；缓存占输入的 62% |
| agentzero | grok-4.20 | pass | pass | pass (gpt-6.1-sol) | pass | pass | My TokenRouter, OpenRouter | 以 x-ai/grok-4.20 提供（提供方对同一模型的别名）；缓存占输入的 47% |
| agentzero | grok-4.3 | pass | pass | pass (gpt-6.1-sol) | pass | pass | My TokenRouter, OpenRouter | 以 x-ai/grok-4.3 提供（提供方对同一模型的别名）；缓存占输入的 17% |
| agentzero | grok-4.5 | pass | pass | pass (gpt-6.1-sol) | pass | pass | My TokenRouter, OpenRouter | 缓存占输入的 34% |
| agentzero | grok-4.6 | pass | pass | pass (gpt-6.1-sol) | pass | pass | My TokenRouter, OpenRouter | 缓存占输入的 6% |
| agentzero | grok-build-0.1 | pass | pass | pass (gpt-6.1-sol) | pass | pass | My TokenRouter, OpenRouter | 以 x-ai/grok-build-0.1 提供（提供方对同一模型的别名）；缓存占输入的 32% |
| agentzero | hunyuan-4-preview | pass | pass | pass (gpt-6.1-sol) | pass | pass | My TokenRouter, OpenRouter | 以 tencent/hy4-preview 提供（提供方对同一模型的别名）；缓存占输入的 55% |
| agentzero | kimi-k2.7-code | pass | pass | pass (gpt-6.1-sol) | pass | pass | My TokenRouter, OpenRouter | 缓存占输入的 62% |
| agentzero | kimi-k3 | pass | pass | pass (gpt-6.1-sol) | pass | pass | My TokenRouter, OpenRouter | 缓存占输入的 0% |
| agentzero | llama-3.3-70b | pass | pass | pass (gpt-6.1-sol) | pass | pass | OpenRouter | 以 meta-llama/llama-3.3-70b-instruct 提供（提供方对同一模型的别名）；缓存占输入的 32% |
| agentzero | mistral-medium-3.5 | pass | pass | pass (gpt-6.1-sol) | pass | pass | My TokenRouter, OpenRouter | 以 mistralai/mistral-medium-3-5 提供（提供方对同一模型的别名）；缓存占输入的 63% |
| agentzero | muse-glimmer-30b | pass | pass | pass (gpt-6.1-sol) | pass | pass | Vercel AI Gateway, OpenRouter | 以 meta/muse-glimmer-30b 提供（提供方对同一模型的别名）；重测一次；首次尝试：artifact 本轮失败：HandledException: Agent stopped after 5 consecutive unusable mo；缓存占输入的 60% |
| agentzero | muse-spark-1.1 | pass | pass | pass (gpt-6.1-sol) | pass | pass | Vercel AI Gateway, OpenRouter | 以 meta/muse-spark-1.1 提供（提供方对同一模型的别名）；缓存占输入的 66% |
| agentzero | muse-spark-1.2 | pass | pass | pass (gpt-6.1-sol) | pass | pass | Vercel AI Gateway, OpenRouter | 以 meta/muse-spark-1.2 提供（提供方对同一模型的别名）；缓存占输入的 67% |
| agentzero | muse-spark-1.3 | pass | pass | pass (gpt-6.1-sol) | pass | pass | Vercel AI Gateway, OpenRouter | 以 meta/muse-spark-1.3 提供（提供方对同一模型的别名）；缓存占输入的 0% |
| agentzero | nemotron-3-super | pass | pass | pass (gpt-6.1-sol) | pass | pass | My TokenRouter, OpenRouter | 以 nvidia/nemotron-3-super-120b-a12b 提供（提供方对同一模型的别名）；缓存占输入的 0% |
| agentzero | nemotron-3.5-lightning | pass | pass | pass (gpt-6.1-sol) | pass | pass | My TokenRouter, OpenRouter | 以 nvidia/nemotron-3.5-lightning 提供（提供方对同一模型的别名）；缓存占输入的 15% |
| agentzero | qwen3.7-max | pass | pass | pass (gpt-6.1-sol) | pass | pass | My TokenRouter, OpenRouter | 缓存占输入的 57% |
| agentzero | qwen3.7-plus | pass | pass | pass (gpt-6.1-sol) | pass | pass | My TokenRouter, OpenRouter | 缓存占输入的 47% |
| agentzero | qwen3.8-27b | pass | pass | pass (gpt-6.1-sol) | pass | pass | Vercel AI Gateway, OpenRouter | 以 alibaba/qwen3.8-27b 提供（提供方对同一模型的别名）；缓存占输入的 51% |
| agentzero | qwen3.8-flash | pass | pass | pass (gpt-6.1-sol) | pass | pass | My TokenRouter, OpenRouter | 缓存占输入的 64% |
| agentzero | qwen3.8-max | pass | pass | pass (gpt-6.1-sol) | pass | pass | My TokenRouter, OpenRouter | 缓存占输入的 62% |
| agentzero | step-3.7-flash | pass | pass | pass (gpt-6.1-sol) | pass | pass | My TokenRouter, OpenRouter | 以 stepfun/step-3.7-flash 提供（提供方对同一模型的别名）；缓存占输入的 60% |

56 对，262 次场景运行中 257 次通过；2 对由其他连接或作为其他模型提供，属于发现，不计入。

发现，即由被测连接之外的连接提供、或作为所请求 id 之外的模型提供的配对：

- agentzero x claude-haiku-4.5：42994 个输入 token 无提示缓存读取（每次调用都按全价付费）
- agentzero x claude-opus-4.7：55588 个输入 token 无提示缓存读取（每次调用都按全价付费）
- agentzero x claude-opus-4.8：55260 个输入 token 无提示缓存读取（每次调用都按全价付费）
- agentzero x claude-opus-5.5：55502 个输入 token 无提示缓存读取（每次调用都按全价付费）
- agentzero x claude-sonnet-4.6：42617 个输入 token 无提示缓存读取（每次调用都按全价付费）
- agentzero x claude-sonnet-5：55459 个输入 token 无提示缓存读取（每次调用都按全价付费）
- agentzero x claude-sonnet-5.5：55454 个输入 token 无提示缓存读取（每次调用都按全价付费）
- agentzero x claude-sonnet-5.5-direct：以 claude-sonnet-5-5 提供（CLI 报告其运行的模型）
- agentzero x claude-sonnet-5.5-direct：55461 个输入 token 无提示缓存读取（每次调用都按全价付费）
- agentzero x deepseek-v4-flash：以 deepseek-v4.1-flash 提供（CLI 报告其运行的模型）

## 提供方：anthropic

| Harness | 模型 | 首轮 | 追问 | 切换 | 产物 | 回收 | 服务方 | 说明 |
|---|---|---|---|---|---|---|---|---|
| agentzero | claude-haiku-5.5 | pass | pass | pass (claude-haiku