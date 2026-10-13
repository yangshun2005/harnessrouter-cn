# Schema

**统一 Harness 协议，版本 `2026-10-04`**

机器可读定义在结构上具有规范性。当本文与 Schema 对某个对象的形状描述不一致时，以 Schema 为准；当二者对行为描述不一致时，以本文为准，因为行为无法用 JSON Schema 表达。

## 文件

| 文件 | 格式 | 用途 |
|---|---|---|
| [`uhp-2026-10-04.openapi.yaml`](../../schema/uhp-2026-10-04.openapi.yaml) | OpenAPI 3.1 | 生成客户端和服务端；浏览 API |
| [`uhp-2026-10-04.schema.json`](../../schema/uhp-2026-10-04.schema.json) | JSON Schema 2020-12 | 校验对象和流式事件 |

两者均以文件名区分版本。已发布的版本不可变：修复 Schema 意味着发布新版本，绝不能修改客户端可能已据此生成代码的版本。

## 生成客户端

```bash
# TypeScript
npx openapi-typescript protocol/schema/uhp-2026-10-04.openapi.yaml -o uhp.d.ts

# Python
openapi-python-client generate --path protocol/schema/uhp-2026-10-04.openapi.yaml

# Go
oapi-codegen -package uhp protocol/schema/uhp-2026-10-04.openapi.yaml > uhp.gen.go
```

## 校验事件

每个流式事件都依据 `Event` 定义校验，该定义是以 `type` 为判别字段的可辨识联合：

```python
import json, jsonschema

schema = json.load(open("protocol/schema/uhp-2026-10-04.schema.json"))
validator = jsonschema.Draft202012Validator(
    {"$ref": "#/$defs/Event", **schema})

for line in stream:
    if line.startswith("data: "):
        validator.validate(json.loads(line[6:]))
```

一致性测试套件正是对运行中的服务端执行上述校验，因此参考实现无法满足的 Schema 变更会在 CI 中失败，而不会发布出去。

## 插件清单

`PluginManifest` 逐字段对应 Agent Plugins 1.0.0 清单 Schema，但在一处更宽松：它允许额外属性，因为 UHP 客户端必须忽略不认识的字段（[版本管理](../../VERSIONING.md)），而后续 Agent Plugins 版本会新增一些字段。清单的权威 Schema 是其自身 `$schema` 所指定的那个；UHP 服务端据此校验，此处的定义仅描述客户端可以依赖的形状（[插件 §2.2](plugins.md#22-what-the-server-derives)）。

## 扩展点

服务端可以在 Schema 允许额外属性的任何位置添加字段：

- 请求或响应上的 `metadata` —— 客户端和服务端上下文的预期存放位置。
- 错误上的 `detail` —— 特定失败的结构化上下文。
- 额外的输出项类型，以及额外的事件类型。

服务端不得重新定义已指定字段的含义，也不得添加必填字段：针对此版本编写的客户端必须继续可用。厂商特定字段应带有厂商前缀，以免与后续版本的规范发生冲突。