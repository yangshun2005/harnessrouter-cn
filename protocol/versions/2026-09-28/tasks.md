# 任务

**统一 Harness 协议，版本 `2026-09-28`**

任务是一个工作单元：输入进去，结果出来。它是大多数客户端花费全部时间的端点。

## 1. 运行任务

```http
POST /v1/responses
Authorization: Bearer <token>
Content-Type: application/json
```

```json
{
  "input": "Summarise README.md in three bullets.",
  "model": "claude-sonnet-4.6",
  "metadata": { "harness_id": "chrn_08dae611630d467ab3e67ed792570ae5" },
  "stream": true
}
```

### 1.1 请求字段

| 字段 | 类型 | 必填 | 含义 |
|---|---|---|---|
| `input` | string \| item[] | 是 | 工作内容。裸字符串是一条用户消息的简写。见 §2。 |
| `model` | string | 否 | 规范模型 id。省略表示使用 harness 的默认值。 |
| `metadata` | object | 否 | 客户端元数据。`harness_id` 选择已配置的 harness。见 §1.2。 |
| `stream` | boolean | 否 | `true` 流式返回 Server-Sent Events；默认 `false` 返回单个 JSON 对象。 |
| `previous_response_id` | string | 否 | 延续产生该响应的会话。见 [Sessions](sessions.md)。 |
| `instructions` | string | 否 | 仅针对此任务的额外系统指导。 |
| `store` | boolean | 否 | 服务器是否保留响应以供后续读取。默认 `true`。 |
| `max_output_tokens` | integer | 否 | 生成 token 的上限。 |
| `max_step` | integer | 否 | 此任务 agent 步数（工具调用轮次）的上限。 |
| `timeout_seconds` | integer | 否 | 此任务的挂钟时间预算。 |
| `environment` | string | 否 | 此任务读取的环境，覆盖 harness 的环境；任务开始前以 `environment_not_found` 或 `environment_not_ready` 拒绝。需要 `environments` 能力（[Environments §5](environments.md#5-attaching-an-environment)）。 |
| `tools` | array | 否 | **保留并忽略。** 接受，但从不执行。见 §1.4。 |
| `include` | string[] | 否 | **保留并忽略。** 接受，但从不执行。见 §1.4。 |
| `background` | boolean | 否 | 任务被接受后立即返回；用事件端点跟进。 |

服务器必须忽略它不理解的请求字段，而不是拒绝请求。客户端不得依赖未知字段产生效果。

忽略必须是可观察的。当服务器未对请求携带的字段采取行动时，它必须在响应的 `metadata.ignored_fields` 中列出该字段——一个请求字段名数组，顺序不限。这与 §1.3 中的模型替换是同一原则：服务器没有按请求字面执行的地方，响应要说明。被静默忽略的字段与被遵守的字段无法区分，客户端无法判断得到的是哪种。

`tools` 和 `include` 是保留并忽略的（§1.4），因此只要请求携带它们，服务器就必须在 `metadata.ignored_fields` 中列出。服务器不得因请求携带其中任一字段而拒绝请求。

`max_step` 和 `timeout_seconds` 是预算，不是精确性保证：服务器必须在预算处或之后停止任务，必须报告 `incomplete`，且不得为被截断的工作报告 `completed`。

### 1.2 选择 harness

已配置的 harness 由 `metadata.harness_id` 选择：

```json
{ "metadata": { "harness_id": "chrn_…" } }
```

- 如果 `harness_id` 缺失，服务器必须使用默认 harness，并必须在响应 `metadata` 中报告使用了哪一个。
- 如果 `harness_id` 指向未知 harness，或调用者范围之外的 harness，服务器必须以 `404` 和 `code: "harness_not_found"` 失败。
- 如果 `previous_response_id` 和 `harness_id` 同时存在且不一致，服务器必须以 `409` 和 `code: "harness_mismatch"` 失败（见 [Lifecycle §4](lifecycle.md)）。

> **为什么 harness 在 `metadata` 中而不是顶层字段？**
> 因为任务接口刻意保持 Responses 兼容（见 [README](../../README.md)），而 `metadata` 正是该接口已定义的、用于调用者提供上下文的扩展点。顶层 `harness` 字段会是同一想法的第二套冲突约定，且每个现有 Responses SDK 都必须打补丁才能发送它。

### 1.3 模型选择与替换

`model` 指定规范模型 id，而非提供商特定的 id。规范 id 跨提供商稳定，因此无论服务器直接访问模型还是通过聚合器，同一请求都能工作。

如果请求的模型无法为所选 harness 提供服务，服务器不得静默失败，也不得假装运行了所请求的模型。它必须且只能执行以下之一：

1. **失败**，以 `422` 和 `code: "model_unavailable"`；或
2. **替换**为 harness 的授权默认模型，并在响应中记录替换：

```json
{
  "model": "gpt-5.4",
  "metadata": {
    "requested_model": "gpt-5.6-sol",
    "model_fallback": true,
    "model_fallback_reason": "model 'gpt-5.6-sol' is not available for this harness's backend"
  }
}
```

因此客户端始终可以通过比较 `model` 与 `metadata.requested_model` 来回答"我请求的模型实际运行了吗？"

> 这条规则存在是因为缺失它代价高昂。静默替换的服务器会让下游所有测量都出错——基准测试、成本归因、质量比较——而客户端无法检测。报告替换只花两个字段。

### 1.4 保留字段：`tools` 和 `include`

两个字段都随本版本保持兼容的 OpenAI Responses 线格式而来。二者都不是为 UHP 设计的，在此也从未有定义语义。它们是**保留的**：服务器接受它们，从不执行，并在 `metadata.ignored_fields` 中报告（§1.1）。

这是决定而非遗留缺口，因此实现者可以不再把它们当作未完成工作，客户端作者也可以不再基于它们构建。

**`tools` 不能表示它在 Responses API 中的含义。** 在那里，该字段存在是因为*客户端*执行工具：模型发出 `function_call`，客户端运行它，并在下一个请求中返回 `function_call_output` 作为输入。UHP 把这两者都放在 `output` 中（§3.1）。harness 自己调用并执行工具，并将其作为可观测性报告。工具结果没有输入路径，因此 `tools` 暗示的循环无法由合规服务器完成——不是因为实现不成熟，而是因为对象模型没有返回段的位置。

**`tools` 也不能表示"此任务的 MCP 服务器"，这是另一种可能的解读。** [Harnesses](harnesses.md) §4.1 已在 harness 上指定 MCP 服务器，含字段表和关键语义：禁用的条目不得被联系，不可达的服务器不得使任务失败。请求级形式会是同一行为的第二套机制。

更重要的是，它会成为**提权原语**。如果请求可以附加 MCP 服务器，任何持有 API 密钥的人都能把 agent 指向自选端点，而 agent 以 harness 的凭据和工作区访问权限在服务端执行工具。harness 所有者和 API 调用者通常是不同方——产品配置 harness，其终端用户驱动任务。请求级声明会静默地把能力决策从责任方转移到任意调用者。已配置的 harness 是一等对象，正是为了让该决策由承担后果的人一次性、审慎地做出。

底层的持久规则，适用范围超出此字段：**收窄是安全的，放宽是提权。** 未来版本可以合理地添加类型化的每请求工具*禁用*列表。它绝不应添加每请求*授予*。

**`include` 没有词汇表。** 它是 `array of string`，没有枚举值，因此服务器识别的任何字符串都是该服务器自己命名的。识别某个值等于发明没有其他实现共享的词汇表。

**agent 如何获得工具：** 在 harness 上配置它们。MCP 服务器见 [Harnesses](harnesses.md) §4.1，技能见 §4.2。

移除是下一版本的问题。本版本仅做加法，不能删除字段，因此这里诚实的形式是直说这两个字段什么都不做。

## 2. 输入

`input` 是字符串或条目数组。

```json
{ "input": "Summarise README.md" }
```

```json
{
  "input": [
    { "role": "user", "content": [
        { "type": "input_text", "text": "Summarise this report." },
        { "type": "input_file", "filename": "q3.pdf", "file_data": "data:application/pdf;base64,…" }
    ]}
  ]
}
```

| 条目类型 | 用途 |
|---|---|
| `input_text` | 文本 |
| `input_file` | 文件，以数据 URL 内联于 `file_data`，或通过先前上传的 `file_id` |
| `input_image` | 图像，内联或通过 `file_id` |

**Core** 一致性等级的服务器必须接受 `input_text`。**Extended** 等级的服务器还必须接受 `input_file` 和 `input_image`。见 [Files](files.md)。

## 3. 响应对象

```json
{
  "id": "resp_a1b2c3",
  "object": "response",
  "created_at": 1786400000,
  "status": "completed",
  "error": null,
  "incomplete_details": null,
  "previous_response_id": null,
  "model": "claude-sonnet-4.6",
  "output": [
    { "id": "rs_1", "type": "reasoning", "summary": [ { "type": "summary_text", "text": "…" } ], "status": "completed" },
    { "id": "fc_1", "type": "function_call", "call_id": "call_1", "name": "read_file",
      "arguments": "{\"path\":\"README.md\"}", "status": "completed" },
    { "id": "fco_1", "type": "function_call_output", "call_id": "call_1", "output": "…", "status": "completed" },
    { "id": "msg_1", "type": "message", "role": "assistant", "status": "completed",
      "content": [ { "type": "output_text", "text": "- …\n- …\n- …", "annotations": [] } ] }
  ],
  "store": true,
  "usage": { "input_tokens": 5120, "output_tokens": 240, "total_tokens": 5360 },
  "metadata": { "session_id": "hsess7e78…" }
}
```

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `id` | string | 是 | `resp_` 前缀 |
| `object` | string | 是 | 始终为 `response` |
| `created_at` | integer | 是 | Unix 秒 |
| `status` | string | 是 | 见 [Lifecycle §3](lifecycle.md) |
| `error` | object \| null | 是 | 仅当 `status` 为 `failed` 时存在且非 null |
| `output` | item[] | 是 | 有序；未产生任何内容的任务可以为空 |
| `model` | string | 是 | 实际运行的模型 |
| `usage` | object \| null | 是 | 服务器无法核算用量时为 `null`——绝不用伪造的零 |
| `metadata` | object | 是 | 必须包含 `session_id`；替换字段见 §1.3 |
| `previous_response_id` | string \| null | 是 | 此响应延续的响应（如有） |
| `store` | boolean | 是 | 此响应是否被保留 |

### 3.1 输出条目

| `type` | 含义 |
|---|---|
| `message` | 助手文本，在 `content[].text` 中，`annotations` 用于构建产物 |
| `reasoning` | agent 的摘要思考，在 `summary[].text` 中 |
| `function_call` | agent 调用的工具，`arguments` 为 JSON 字符串 |
| `function_call_output` | 该工具调用的结果，通过 `call_id` 匹配 |

客户端必须容忍它不认识的输出条目类型，且不得假设数组给定顺序之外的任何顺序。只渲染 `message` 条目并忽略其余的客户端是有效客户端。

## 4. 读回任务

```http
GET /v1/responses/{response_id}
```

返回相同的响应对象。保留了响应（`store: true`）的服务器必须在完成后返回它；对于以 `store: false` 创建的响应，服务器可以返回 `404` 和 `code: "response_not_found"`。

```http
GET /v1/responses/{response_id}/input_items
```

返回任务创建时的输入，供需要在不自行存储的情况下重建记录的客户端使用。

## 5. 取消

```http
POST /v1/responses/{response_id}/cancel
```

见 [Sessions §4](sessions.md)。

## 6. 幂等性

客户端可以发送幂等键：

```http
Idempotency-Key: <client-generated-unique-string>
```

声明 `idempotency` 的服务器对于重复的键必须：

- 返回**第一个**请求的结果，且
- **不**启动第二次执行。

如果第一个请求仍在运行，服务器必须等待它并返回其结果，而不是返回部分结果或冲突。agent 任务昂贵且有副作用：运行两次工作的重试比慢答案更糟。

幂等键应至少保留 24 小时。

## 7. 删除

```http
DELETE /v1/responses/{response_id}
```

删除已存储的响应。服务器不得让此操作取消正在运行的任务——取消和删除是不同的意图，混淆它们意味着客户端无法在不停止工作的情况下清理历史。