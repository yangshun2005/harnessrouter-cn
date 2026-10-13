# 环境

**统一 Harness 协议，版本 `2026-09-28`**

会话的工作目录（[架构 §3](architecture.md#3-object-model)）是为一次对话而生的：创建时为空，跨轮次保留，agent 需要的所有东西在每个需要它的会话中重新放入。这对任务自身的文件是合适的形态，对项目则不合适：一条内容流水线，带着脚本、素材、模板和上百个已安装的包，被许多人每天打开多次。装进工作目录意味着每个会话装一次、每个会话上传一次，而会话的检查点不会把其中任何东西带过去。

**环境**就是把那个项目当作一个对象：它的文件和已安装的依赖，构建一次，被每个指名它的会话读取。会话在固定路径上看到它，只读，紧挨着自己的工作目录；工作目录保持可写、私有，且是构建产物的唯一来源。

本章是环境子协议。它是可选的：服务器用 `environments` 能力声明它，未实现它的服务器在每个类别上都合规。

## 1. 环境是什么

```
/env/content-studio          环境：完整、只读、共享     （本章）
  scripts/  assets/  data/   项目的文件，按所有者放入的样子
  .venv/  node_modules/      它的依赖，构建时安装一次
<working directory>          会话自己的：可写、私有、有检查点   （Files 章）
```

三条规则让它成为一个东西，而不是一个共享文件夹：

- **它是构建出来的，不是复制来的。** 所有者放入文件（[§3](#3-files)）；构建（[§4](#4-builds-and-versions)）把它们快照成一个版本，并用服务器自己的工具链安装项目清单声明的内容。随文件一起到来的依赖目录不是会话拿到的东西；构建安装的才是。在笔记本上创建的 virtualenv 在容器里跑不起来，而构建正是让"无需重装"这个承诺成立的东西。
- **会话读取活动版本，且不能写入它。** 同一环境上的两个会话共享它的每一个字节，但不共享彼此的工作目录。服务器必须拒绝来自会话的对环境的写入：该路径是只读的，而不只是被描述为只读。
- **路径是固定的。** 服务器告诉客户端环境挂载在哪里（`mount`），也告诉 agent 同样的信息，这样项目的脚本可以从任何工作目录用绝对路径运行，其输出落在用户能看到的地方。

> **为什么不让会话直接在项目里启动？**
> 一个*就是*项目的工作目录，写时复制，读起来很好，直到两个事实相撞：构建产物和项目文件成了同一个目录，服务器再也说不清哪些文件是任务产生的；而且每次写入都必须由某种机制（overlay 文件系统、每会话副本）分层，不是每个沙箱都有。两条路径，其中一条只读，让两个答案都保持简单：任务产生的东西在工作目录里，项目本身留在环境里。

## 2. 环境对象

```json
{
  "id": "henv_2f1c9d4e8a7b4c3d9e6f5a4b3c2d1e0f",
  "object": "environment",
  "name": "Content Studio",
  "slug": "content-studio",
  "description": "The teaser pipeline: episodes in, teasers and thumbnails out.",
  "entry": "python3 run.py --episode <id>",
  "status": "ready",
  "mount": "/env/content-studio",
  "version": 3,
  "latestVersion": 3,
  "files": { "count": 16, "bytes": 48213 },
  "packages": [ { "manager": "pip", "name": "moviepy", "version": "2.1.1" },
                { "manager": "npm", "name": "sharp", "version": "0.33.5" } ],
  "versions": [ { "version": 3, "status": "ready", "started_at": 1790570000, "finished_at": 1790570094, "packages": 10 } ],
  "build": { "version": 3, "status": "ready", "started_at": 1790570000, "finished_at": 1790570094, "error": "" },
  "createdAt": 1790560000000,
  "updatedAt": 1790570094000
}
```

| 字段 | 类型 | 写入方 | 含义 |
|---|---|---|---|
| `id` | string | 服务器 | `henv_` 前缀（[架构 §3](architecture.md#3-object-model)） |
| `name` | string | 客户端 | 人类可读；必须以字母或数字开头 |
| `slug` | string | 服务器 | 会话看到的路径段，创建时由名称派生。它永不改变：会话的指令会指名它 |
| `description` | string | 客户端 | |
| `entry` | string | 客户端 | 项目如何运行，用所有者的话说。原样重复给 agent（[§6](#6-what-a-session-sees)） |
| `status` | string | 服务器 | `empty`（什么都没构建）、`building`（有构建在运行，无论哪个版本是活动的：任务继续读取活动版本）、`ready`（有版本是活动的）、`failed`（最新构建失败且没有活动版本） |
| `mount` | string | 服务器 | 会话在哪里找到它，只读。参考服务器上是 `/env/<slug>`；服务器可以选另一个根，但必须在环境的整个生命周期内保持不变 |
| `version` | integer 或 null | 服务器 | 活动构建，即会话读取的那个 |
| `latestVersion` | integer 或 null | 服务器 | 最新构建，无论是否活动 |
| `files` | object | 服务器 | 源的大小：所有者放入的内容，在任何构建之前 |
| `packages` | array | 服务器 | 活动构建安装的内容，按包管理器报告的样子：`manager`、`name`、`version` |
| `declared` | object | 客户端 | 所有者声明的内容，按管理器分组，作为构建在项目清单旁安装的 spec 字符串：`pip`（`moviepy==2.1.1`）、`npm`（`sharp@0.33.5`）、`apt`（`ffmpeg`）；回读时带每个 spec 的 `name` 和 `version` |
| `runtime` | object | 客户端 | `python`：构建用来创建虚拟环境的解释器，取自 `GET /v1/environments/runtimes` 之一；为空表示服务器默认 |
| `versions` | array | 服务器 | 每次构建：`version`、`status`、时间、`error`、大小 |
| `build` | object 或 null | 服务器 | 最新构建的状态，运行时的 `stage` |

环境的范围划分与 harness 相同：调用方只能看到自己范围内的环境（[架构 §5](architecture.md#5-authentication)），harness 只能指名同一范围内的环境。

```http
POST   /v1/environments                     创建（name、description、entry）
GET    /v1/environments                     列表
GET    /v1/environments/{id}                单个，带其版本
PUT    /v1/environments/{id}                重命名；description；entry
DELETE /v1/environments/{id}                删除所有版本
```

`POST` 拒绝无法产生路径段的名称，返回 `environment_invalid`；拒绝另一个环境已占用的 slug，返回 `environment_exists`：挂载路径是一个位置，两个环境不能都在里面。`DELETE` 在有构建运行时拒绝（`environment_busy`）。指名了已删除环境的 harness 保留该引用；它的下一个任务被拒绝，返回 `environment_not_found`（[§5](#5-attaching-an-environment)）。

## 3. 文件

源按路径编辑，方式和插件的包一样（[插件 §2.1](plugins.md#21-files)），但每个请求一个文件，因为项目比包大：

```http
GET    /v1/environments/{id}/files                 树：每个文件和目录，带大小
GET    /v1/environments/{id}/files/{path}          单个文件的字节，带其内容类型
PUT    /v1/environments/{id}/files/{path}          写入一个文件（请求体是它的字节；其目录会被创建）
POST   /v1/environments/{id}/directories           创建空目录：{"path": "assets/brand"}
DELETE /v1/environments/{id}/files/{path}          删除一个文件或目录树
POST   /v1/environments/{id}/import                一次性导入整个项目
```

- `path` 相对于环境的根。服务器必须拒绝逃出根的路径，或指名常规文件或目录之外的任何东西，返回 `environment_invalid`。
- 服务器必须按发送的原样保留字节，必须拒绝过大的文件，返回 `file_too_large`（`detail.max_bytes` 给出限制），而不是截断它。
- 写入源在下次构建之前不会改变会话看到的任何东西（[§4](#4-builds-and-versions)）。

### 3.1 导入

项目作为一个整体到来。`POST /v1/environments/{id}/import` 接受一个归档（zip 或 tar，作为请求体，内容类型为除 `application/json` 外的任意值）或一个指名 git 仓库的 JSON 体：

```json
{ "git": { "url": "https://github.com/example/content-studio", "ref": "main" }, "replace": true }
```

树按归档或仓库中的原样保留。包裹一切的单层目录（仓库归档的形态）会被剥掉，所以项目的根就是环境的根。不应落地的成员、链接和逃出根的路径会被丢弃并计入 `skipped`，绝不写入。`replace` 先清空源；不带它则导入覆盖在现有内容之上。归档中的依赖目录（`node_modules`、`.venv`）不会被复制进构建；构建从清单安装（[§4](#4-builds-and-versions)）。

## 4. 构建与版本

```http
GET  /v1/environments/runtimes                          这里能用什么构建：Python 次版本、Node 主版本、操作系统
GET  /v1/environments/packages/check?manager=&spec=     管理器的 registry 对某个 spec 的说法，在任何构建之前（可选）
POST /v1/environments/{id}/build                        开始构建：下一个版本
GET  /v1/environments/{id}/builds/{version}             其记录：状态、日志、包、大小
GET  /v1/environments/{id}/versions                     每次构建，以及哪个是活动的
POST /v1/environments/{id}/versions/{version}/activate  让一个已完成的构建成为活动版本
```

构建把源快照成一个新版本，并安装项目声明的内容：

| 根目录的清单 | 构建运行什么 |
|---|---|
| 声明的 `pip` | 同一个虚拟环境（服务器有时用 `runtime.python` 创建），然后对每个 spec 执行 `pip install` |
| 声明的 `npm` | 对每个 spec 执行 `npm install` 到 `node_modules` |
| 声明的 `apt` | 镜像缺少的具名包和依赖，由 apt 下载并用 dpkg 解包到版本内的 `apt/`，会话在 `PATH` 和 `LD_LIBRARY_PATH` 上获得它；不向服务器安装任何东西 |
| `requirements.txt` | 在 `.venv` 创建 Python 虚拟环境，然后 `pip install -r requirements.txt` |
| `pyproject.toml` | 同一个虚拟环境，然后 `pip install .` |
| `package.json` | 有 lockfile 时 `npm ci`，否则 `npm install`，到 `node_modules` |
| `setup.sh` | 最后运行，在版本的目录中，用于清单无法表达的内容 |

服务器可以支持其他清单，但必须记录它运行哪些。安装发生在服务器自己的工具链中，即会话运行的那个，所以构建安装的东西在会话运行的地方能跑。构建之后不再向版本写入任何东西：该层被设为只读并保持如此。

`POST …/build` 立即返回版本号；`…/builds/{version}` 的记录先是 `building`，然后 `ready` 或 `failed`，带安装器自己的输出作为 `log`，失败原因作为 `error`。构建期间，记录用 `stage` 说明正在运行哪一步（参考服务器上：`copying files`、`python packages`、`node packages`、`system packages`、`setup.sh`、`finishing`），并带上到目前为止的日志，这样客户端可以显示进度而不是转圈；服务器必须在每步结束时写入记录，而不只在构建结束时。与此同时环境的 `status` 是 `building`，即使有版本是活动的，其上的任务继续读取活动版本。每个环境一次只运行一个构建（`environment_busy`）。构建成功时它成为活动版本：下一个启动的会话读取它。更早启动的会话在其本轮剩余时间内保持启动时的版本；服务器应让它后续轮次也保持该版本，且必须说明某轮读取了哪个版本（[§6](#6-what-a-session-sees)）。

失败的构建不改变任何东西：之前的活动版本保持活动，`status` 只有在完全没有活动版本时才说 `failed`。回滚是对更早的已完成构建执行 `POST …/versions/{n}/activate`；不重建任何东西，从未完成的版本不能被激活（`environment_not_ready`）。

服务器可以用管理器 registry 对某个 spec 在任何构建之前的说法来回答 `GET /v1/environments/packages/check?manager=&spec=`：名称是否存在、其最新版本、以及某个精确 pin 是否已发布（`EnvironmentPackageCheck`：`manager`、`name`、`spec`、`exists` 为 true、false，或无法询问 registry 时为 null、`latest`、已发布 pin 的 `version`、文字形式的 `error`）。提供它的客户端在输入处就拒绝拼写错误，而不是几分钟后让构建失败；客户端不得要求它，没有它的服务器回答 `404`。参考服务器询问 PyPI、npm registry 和自己的 apt 列表。

> **为什么用版本而不是原地重建？**
> 原地重建会改变会话当时正在读取的层，失败的构建会让它们什么都没有。版本让构建从会话的角度看是原子的：旧层完整直到新层完整，切换只是一个指针。

## 5. 附加环境

harness 指名其任务读取的环境，任务可以指名另一个：

```json
{ "name": "Teaser cutter", "base": "codex", "environment": "henv_2f1c9d4e8a7b4c3d9e6f5a4b3c2d1e0f" }
```

```json
{ "input": "Render the teaser for episode 12.",
  "metadata": { "harness_id": "chrn_…", "environment": "henv_2f1c9d4e8a7b4c3d9e6f5a4b3c2d1e0f" } }
```

- harness 上的 `environment`（[Harness §2](harnesses.md#2-the-harness-object)）是该 harness 每个任务的默认值；任务上的 `metadata.environment` 为该任务覆盖它。响应把任务读取的环境报告为 `metadata.environment`，紧挨着 `metadata.session_id`。

> **为什么任务的环境在 `metadata` 里而不是顶层字段？**
> 原因和 harness 一样（[任务 §1.2](tasks.md#12-selecting-the-harness)）：任务接口是一个 Responses 请求，而 `metadata` 是该接口为调用方提供的上下文定义的扩展点。harness 和会话是本协议自己的对象，所以在它们上面 `environment` 是普通字段；任务请求不是，所以在它上面环境随 harness 一起走。每个 Responses SDK 今天都能发送它。
- 服务器必须拒绝指名调用方范围之外环境的 harness 写入，或完全没有环境时，返回 `environment_not_found`。
- 服务器必须在**任务开始之前**拒绝环境不是 `ready` 的任务：不存在时 `environment_not_found`，什么都没构建时 `environment_not_ready`。任务因缺少项目而在几分钟后失败，正是这条规则要防止的结果。
- `GET /v1/environments/{id}/harnesses` 列出指名它的 harness，这样所有者能看到重建会影响什么。

## 6. 会话看到什么

对于带环境运行的任务，服务器必须：

1. 在 agent 启动前，把活动版本以只读方式挂载到 `mount`；
2. 保持会话的工作目录原样：可写、会话私有、其构建产物的来源（[文件 §2](files.md#2-getting-artifacts-out)），检查点方式不变。环境永远不是检查点的一部分；
3. 让环境已安装的依赖成为 agent shell 运行的依赖：参考服务器上 `PATH` 以版本的 `.venv/bin` 和 `node_modules/.bin` 开头，`PYTHONPATH` 和 `NODE_PATH` 以环境开头，`PROJECT_ROOT` 指名 `mount`；
4. 在指令中告诉 agent 项目在哪里、它是只读且共享的、其依赖已安装、在哪里写入，以及所有者给出时的 `entry`；
5. 记录该轮读取了哪个环境和版本：响应带 `metadata.environment`，会话对象带 `environment`（[会话 §3](sessions.md#3-inspecting-a-session)），该轮的记录带版本。

同一环境上的两个会话必须不能读取彼此的工作目录，也不能写入环境。会话必须不能读取其任务未指名环境，也不能枚举服务器持有的环境。服务器如何强制这一点是它自己的事：带只读挂载的每会话容器、每会话用户加对共享目录无写权限的每环境组，或做成只读的副本。客户端可以依赖的是结果。

## 7. 发现

```json
{ "capabilities": { "environments": true } }
```

报告 `environments: true` 的服务器实现本章的每个端点。未实现本章的报告 `false` 或省略它，且必须以 `404` 回答这些端点，绝不部分实现。

## 8. 错误

| 代码 | 状态 | 何时 |
|---|---|---|
| `environment_not_found` | 404 | 调用方范围内没有该环境；在 harness 写入或任务上，指被引用的那个