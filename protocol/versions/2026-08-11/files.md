# 文件

**统一 Harness 协议，版本 `2026-08-11`**

只能返回文本的 agent 只是个聊天机器人。Harness 会编辑文件、撰写报告、生成图表。本章定义如何传入文件、取出构建产物。

一致性等级 **Extended**。

## 1. 传入文件

### 1.1 内联

小文件可将字节内联为 data URL：

```json
{
  "input": [
    { "role": "user", "content": [
        { "type": "input_text", "text": "Summarise this." },
        { "type": "input_file", "filename": "q3.pdf",
          "file_data": "data:application/pdf;base64,JVBERi0xLjQK…" }
    ]}
  ]
}
```

### 1.2 通过上传

更大的文件先上传一次，再用 id 引用：

```http
POST /v1/files
Content-Type: multipart/form-data
```

```json
{ "id": "file_abc123", "object": "file", "filename": "q3.pdf", "bytes": 184320, "created_at": 1786400000 }
```

```json
{ "type": "input_file", "file_id": "file_abc123" }
```

服务器必须同时支持两种形式。4 KB 的文件用内联可省一次往返；40 MB 的文件用上传可避免每次重试都做 base64 膨胀。服务器必须说明其大小限制，超限上传必须返回 `413` 和 `code: "file_too_large"`，不得截断——静默截断的输入会产出自信却错误的答案。

## 2. 取出构建产物

agent 产出的文件成为会话容器的构建产物，在两处报告。

### 2.1 在响应中

构建产物以 assistant 消息上的注解形式出现：

```json
{
  "type": "message",
  "content": [
    { "type": "output_text", "text": "I've written the report.",
      "annotations": [
        { "type": "container_file_citation",
          "container_id": "cntr_…",
          "file_id": "file_…",
          "filename": "report.md",
          "download_url": "https://server/v1/containers/cntr_…/files/file_…/content",
          "start_index": 0, "end_index": 24 }
      ]
    }
  ]
}
```

只渲染文本的客户端仍能显示正确答案；读取注解的客户端可提供文件下载。这正是预期的分层——构建产物是对消息的补充，绝不替代消息。

### 2.2 通过列出会话

```http
GET /v1/sessions/{session_id}/files
```

```json
{ "files": [ { "id": "file_…", "container_id": "cntr_…", "filename": "report.md",
               "bytes": 2048, "created_at": 1786400240 } ] }
```

服务器必须列出会话的每个构建产物，包括更早任务产生的文件。若只列最近任务，多步会话的早期输出将无法访问。

## 3. 下载

```http
GET /v1/containers/{container_id}/files/{file_id}/content
```

返回原始字节，`Content-Type` 为文件自身的类型。服务器不得将字节包进 JSON。

服务器应发送带原始文件名的 `Content-Disposition`，且必须设置 `X-Content-Type-Options: nosniff`。构建产物是受攻击者影响的内容——agent 可能被诱导写入文件——若不带这些响应头，构建产物就会变成针对客户端自身源的存储型 XSS。

### 3.1 预览渲染

服务器可提供文档格式的渲染预览：

```http
GET /v1/containers/{container_id}/files/{file_id}/pdf
```

若服务器未实现转换，必须返回 `501` 和 `code: "preview_unavailable"`，而不是损坏或空的 PDF。若转换失败，必须返回 `502` 和 `code: "preview_failed"`。客户端据此可区分"此服务器从不预览"与"此文件无法转换"，只有后者值得重试。

## 4. 批量下载

```http
GET /v1/sessions/{session_id}/files/archive
```

将会话的所有构建产物作为单个归档返回。服务器应实现此端点：产出四十个文件的会话不应需要四十次请求。

## 5. 保留与范围

- 构建产物随其会话存在。删除会话必须使其构建产物不可访问。
- 服务器必须将文件访问限定在所属主体内。来自其他主体的 `file_id` 必须返回 `404`，绝不返回字节。
- 服务器不得让 `container_id` / `file_id` 组合越出其容器。通过构建产物 id 进行路径遍历是 UHP 实现中最可能出现的严重漏洞，一致性测试套件会对此探测。