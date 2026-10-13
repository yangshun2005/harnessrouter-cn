# 会话

**统一 Harness 协议，版本 `2026-08-11`**

会话让第二个任务比第一个更省事：对话还在，工作目录也在。本章定义如何继续会话、查看会话，以及停止会话中的工作。

## 1. 继续会话

用 `previous_response_id` 发送下一个任务：

```json
{
  "input": "Now add tests for the function you just wrote.",
  "previous_response_id": "resp_a1b2c3"
}
```

服务器必须：

- 在同一会话中运行新任务，使用同一工作目录及其文件；
- 向 Harness 提供此前任务的对话上下文；
- 使用同一已配置的 Harness；
- 报告相同的 `metadata.session_id`。

同一会话中各任务的 `model` 可以不同。服务器必须支持按任务切换模型，因为后续任务换用更便宜的模型是常见且合理的做法。

若 `previous_response_id` 指向未知响应，服务器必须返回 `404` 和 `code: "response_not_found"`。若所指会话已过期，返回 `404` 和 `code: "session_expired"`——客户端可以从头重试第一种情况，但不应重试第二种。

> **为什么用响应 id 而非会话 id 串联？**
> 因为响应 id 是客户端已经持有的：它来自刚运行完的任务。用它串联还能精确指向对话中的某个点，为服务器日后从更早的响应分支留下空间，而无需改变请求结构。

## 2. 列出会话

一致性等级 **Extended**。

```http
GET /v1/sessions?limit=20&cursor=&harness=chrn_…
```

```json
{
  "sessions": [
    {
      "id": "hsess7e78…",
      "object": "session",
      "harness_id": "chrn_…",
      "title": "Summarise README.md",
      "status": "completed",
      "created_at": 1786400000,
      "updated_at": 1786400240
    }
  ],
  "next_cursor": null
}
```

分页基于游标。服务器必须在最后一页返回 `next_cursor: null`，且不得要求客户端通过收到比请求更少的条目来判断结束——只要某页恰好满页，这种启发式判断就会出错。

## 3. 查看会话

```http
GET /v1/sessions/{session_id}
GET /v1/sessions/{session_id}/turns
```

`/turns` 返回会话按序排列的任务历史，客户端可据此重建未存储的对话记录。每个 turn 都标明其响应 id，客户端可据此获取任意 turn 的完整响应。

`turns` 中每一项必须至少包含：

| 字段 | 类型 | 含义 |
|---|---|---|
| `id` | string | 该 turn 的响应 id，可用于 `GET /v1/responses/{id}`。 |
| `status` | string | 该 turn 的状态，采用[任务 §1](tasks.md)的响应状态词汇。 |

且在服务器持有时应包含：`user`（string，用户消息）、`assistant`（string，助手最终文本）、`tools`（array，该 turn 的工具调用）、`files`（array，该 turn 产生的文件）。服务器可以添加字段；客户端必须忽略不理解的字段。此前这是一个未声明结构的对象，导致任何一致性检查都无法断言状态码以外的内容，每个重建对话记录的客户端都只能按某一实现的习惯来写。

## 4. 取消

两种范围，刻意区分：

```http
POST /v1/responses/{response_id}/cancel     # stop this task
POST /v1/sessions/{session_id}/cancel       # stop whatever is running in this session
```

语义：

- 取消是请求，不保证立即生效。服务器必须尽快停止工作，并必须达到终态。
- 被取消的任务必须以 `status: "cancelled"` 结束，绝不能是 `failed`。
- 取消前产生的输出必须保留。
- 取消已处于终态的任务必须成功且不改变任何东西。客户端在连接中断后重试取消，不应因已成功过而收到错误。
- 取消不得删除会话。对话仍可继续。

即使 Harness 需要更长时间收尾，服务器也应在一秒内响应取消。客户端通常是用户界面，一个三十秒毫无反应的停止按钮看起来就是坏了。

## 5. 会话共享

一致性等级 **Full**。服务器可以允许客户端发布会话的只读视图。

```http
POST   /v1/sessions/{session_id}/share      # publish (a body is OPTIONAL; no body means publish)
GET    /v1/sessions/{session_id}/share      # read the share back
DELETE /v1/sessions/{session_id}/share      # revoke
```

若实现，`POST` 和 `GET` 返回的 share 对象必须包含：

| 字段 | 类型 | 含义 |
|---|---|---|
| `id` | string | 该 share 的标识。 |
| `url` | string | 共享视图的提供地址。可以是相对路径，此时相对于调用方已在使用的 base URL 解析——代理后的服务器无法知道自己的公网源，而基于 base 的相对路径能经受任何前置层。 |

`object` 应为 `"session.share"`。服务器可以携带额外字段（参考实现还返回 `enabled` 开关并接受 `{"enabled": bool}` 请求体）；客户端必须忽略不理解的字段。第二次 `POST` 可以返回已有 share 或新建一个，但撤销必须覆盖为该会话创建的所有链接——只撤销最新的会让运维以为会话已私有，而有人仍持有可用链接。

规则不变：

- 共享视图必须只读——不得允许继续、取消或上传；
- 对已发布 `url` 的 `GET` 必须让持有链接者无需其他凭证即可访问——只有创建者能打开的视图等于没发布给任何人；
- `DELETE` 之后，以及会话本身被删除后（§6），已发布的 `url` 必须停止解析（404 或 410）；
- share 的 id 不得作为 API 其余部分的凭证；
- 服务器不得通过该视图暴露提供商凭证、令牌或他人的数据。

这些此前是一致性测试套件只能靠探测发现的行为（端点和对象都没有声明结构，可移植的检查只能猜测路径，猜不中就跳过）。现根据参考实现展示的结构和 R 系列度量将其编入规范；测试套件现在会断言这些行为。

## 6. 删除

```http
DELETE /v1/sessions/{session_id}
```

删除会话及其存储的历史：对话记录、trace，以及会话任务写入的工作文件夹。此后该会话必须不计入服务器执行的任何存储或内存配额，因为删除正是客户端腾出空间的方式。服务器必须返回 `2xx`，且之后 `GET /v1/sessions/{session_id}` 必须返回 `404`。

`DELETE /v1/traces/{session_id}` 是同一操作的旧路径。它早于会话词汇；服务器可以继续提供（参考实现如此，用同一处理函数），但上述路径才是协议命名的、客户端应使用的路径。

服务器必须先取消会话中所有进行中的任务，且必须使会话之后不可读。删除是取消与删除合理耦合的唯一场景，否则运行中的任务会写入已无归属的存储。