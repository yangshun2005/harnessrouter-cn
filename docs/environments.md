# 环境

项目的文件及其已安装的依赖，构建一次，之后每个引用它的任务会话都读取同一份。这是自托管服务器（0.26 中发布）和托管服务（将按同一契约构建）的正式设计；线上契约是 UHP `2026-09-28` 的[环境章节](../protocol/versions/2026-09-28/environments.md)。

## 它解决的问题

会话的工作目录是为一次对话而生的。它从空开始，每轮结束后打包成 tarball 做检查点（排除依赖目录），下一轮前恢复。任务需要的一切都上传进去，安装的一切下次都要重装。对于一个项目型任务——带脚本、资源、模板和上百个包的内容流水线，每天被多个人打开多次——这种形态是错的（Spiral，SPI-001 到 SPI-004）：每个会话都要重新付上传和安装的代价，两个会话也无法共享同一份副本。

## 会话看到的契约

```
/env/content-studio            环境：完整、只读、所有会话共享
  scripts/ assets/ data/ …     项目文件，按所有者放入时的样子
  .venv/  node_modules/        构建环境时安装一次
/data/workspaces/<sid>         会话工作目录：可写、私有、做检查点，
                               产物的唯一来源（不变）
PROJECT_ROOT=/env/content-studio
PATH=/env/content-studio/.venv/bin:/env/content-studio/node_modules/.bin:…
PYTHONPATH=/env/content-studio   NODE_PATH=/env/content-studio/node_modules:…
```

Agent 的指令文件带一个"项目环境"章节（runner/environments.py，`doc_section`）：项目在哪、它是只读且共享的、依赖已安装、输出写到工作目录、脚本用绝对路径运行，以及所有者提供时的 `entry` 行。Richard 的决定（2026-09-28）：固定的只读路径加私有 cwd，而不是项目内的写时复制工作目录；理由见该章节的第一条注记。

## 对象与操作

一个环境（`henv_…`）像 harness 一样属于某个组织和某个工作区。它有**源**（所有者编辑的文件：逐个上传、导入归档或 git 仓库、在控制台编辑）、**版本**（构建：复制源，然后从 `requirements.txt` / `pyproject.toml` / `package.json` 安装 `.venv` / `node_modules`，然后跑 `setup.sh`，然后把目录树设为只读），以及一个**活跃**版本，即会话读取的那个。构建成功即成为活跃版本；失败则什么都不变；回滚就是指针指回更早的构建。harness 指定其任务读取的环境；任务也可以指定另一个。任务所在环境没有任何构建时，任务在开始前就被拒绝（`environment_not_ready`）。

路由：`/v1/environments`、`…/{id}`、`…/{id}/files[/{path}]`、`…/{id}/import`、`…/{id}/build`、`…/{id}/builds/{n}`、`…/{id}/versions[/{n}/activate]`、`…/{id}/harnesses`；harness 对象上和 `POST /v1/responses` 上的 `environment`；会话对象上的 `environment`。

## 自托管实现（本仓库）

- **记录**：网关（`gateway/app.py`，"Environments" 一节）：一个 `Environment` 顶点，带组织、工作区、名称、slug、描述、entry、状态、活跃/最新版本、版本列表和活跃构建的包，读取时从 runner 刷新。
- **字节**：runner（`runner/environments.py`）拥有 `/data/environments/<id>/`：`source/`（可编辑，`0700`）、`versions/<n>/`（一次构建，`root:<env group>` `0750/0640`，含 `.hr-build.json`：状态、日志、包、大小）、`active` → `versions/<n>`。挂载点 `/env/<slug>` 是指向 `active` 的链接，位于容器 rootfs 上，重启后惰性重建。构建在 runner 的线程中运行，用镜像自带的 `python3` 和 `npm`，缓存在临时目录，墙钟超时 `HR_ENV_BUILD_TIMEOUT`（1800 秒）。源中的依赖目录从不复制。
- **只读，且只被自己的会话读取**：写墙（`HR_SESSION_UIDS`）：agent 以会话的 uid 运行，层属于 root，所以写入会以 EACCES 失败。每个环境有自己的组（`GID_BASE` 60000 + id 的哈希；环境目录的组**就是**记录，正如工作区的所有者就是会话 uid）。源是 `0700`，仅 runner 可访问；构建后的版本是 `0750/0640`，组可读；agent 进程只在其 harness 指定该环境的那一轮加入该组（`extra_groups`），所以一个会话只能读被赋予的那一个环境，读不到别的。`/data/environments` 和 `/env` 是 `0751`：可穿越、不可列举，所以会话无法枚举其余环境。pip、npm 和 `setup.sh` 会运行包自身的代码，所以构建以环境的身份（`uid = gid = 环境的编号`）运行它们，该身份只拥有正在构建的层，别的都不拥有；apt-get 和 `dpkg -x` 不运行包代码，仍以 runner 身份运行。分组之前的存储会在启动时迁移一次（`secure_store`）。没有写墙时（开发机），这些权限模式对任何非 root agent 仍然成立；root 是操作者自己的选择。2026-09-28 在 hr-test 上实测：分组之前，任何会话都能列举并读取每个环境的源和层；之后，会话只能读自己的，读别的会得到 EACCES。
- **构建期间**：runner 在每一步结束时写 `.hr-build.json`，含 `stage`（复制文件、python 包、node 包、系统包、setup.sh、收尾）和目前的日志；页面每 3 秒轮询，显示一条带阶段、已用秒数和日志尾部的条带。添加时调用 `GET /v1/environments/packages/check`（runner `check_package`：PyPI JSON、npm registry、本机 apt 列表），拒绝不存在的名称或从未发布过的版本锁定；一行显示已安装版本，或"安装中"，或"保存时安装"并附 registry 报告的版本。
- **轮次**：网关在分配任何资源之前解析环境（请求上的 `metadata.environment`，否则用 harness 的），把 `{id, slug, entry}` 传给 runner，在会话顶点上盖 `environment`，在轮次记录上盖版本。runner 在写 agent 文档前解析挂载点（此处 409 会在任何进程启动前让该轮失败），然后设置变量。
- **检查点**不变：环境在工作区之外，所以永远不会进入检查点。

## 托管服务如何实现

2026-09-28 与托管会话一起评审；修正意见来自他们。托管服务在自定义容器池（`harness-sessions-e2`：1 vCPU、2 GiB、约 4 GiB 临时磁盘，1800 秒冷却后容器被回收）中，每个 Azure Container Apps 动态会话跑一个会话。没有池主机可以放缓存，没有可给会话的卷或绑定挂载，沙箱中也不能持有 blob 凭据。同一契约这样构建：

1. **记录。** HR 租户上同样的 `Environment` 顶点，在写它的网关滚动之前用种子脚本注册（标签和边），如同计费标签：种子之前写入会 400。同样的路由、同样的对象，`mount` 仍是 `/env/<slug>`。
2. **源。** 每个文件一个 blob（`environments/<id>/source/<path>`），外加一个小索引 blob 存目录树，所以控制台的编辑器一次读写一个文件。构建时先从它们物化出 `environments/<id>/versions/<n>/source.tgz`，所以构建的输入是一个不可变对象：可复现、一次下载。
3. **构建 = 沙箱任务，绝不是网关线程。** 网关副本没有工具链、没有磁盘，还可能重启。构建从池中取一个沙箱，标识符为 `env-<id>-<n>`（不含冒号；字符集是池的），策略门在沙箱分配**之前**运行（额度不足和计划拒绝；构建对 Free 计划的并发算作一个任务），"每个环境同时只允许一个构建"的锁和构建状态作为带 TTL 的租约存在控制存储中，由掉队清扫清理，和轮次完全一样，所以副本重启不会遗留孤儿。沙箱内运行同一个 `runner/environments.py::_build`（两侧都调用的那一个函数），输入是 `source.tgz`，墙钟 1800 秒。构建作为组织下的 agent 工作计量，有自己的收据行（"Environment build"，`agent.active_second`，应用每任务成本上限），不是新的账本单位。
4. **blob 访问归网关，形态统一。** 沙箱永远看不到账户密钥。对于层，网关签发按 blob、分钟级有效期的 SAS（构建输出为只写，轮次输入为只读），在构建或轮次 body 中交给对方，所以几百 MB 的层不会经过没有磁盘的网关副本。构建用其写 SAS 写 `layer.tgz`（zstd）和 `.hr-build.json`（状态、日志、包、大小）；轮次的水合用自己的读 SAS 下载 `layer.tgz`。源文件和索引只由 API 写入（网关自己流式写入，如同流式写检查点）。
5. **挂载 = 水合时解压。** 这就是托管设计，不是第一版切片：当一轮指定环境时，`/hydrate` 下载活跃版本的 `layer.tgz` 并解压到 `/env/<slug>`，每个会话生命周期一次（容器存活到冷却回收，所以该会话后续轮次发现它已就位），解压后 root 所有、755/644，不做 `chmod -R`；agent 以镜像的 agent uid（10001）运行，与自托管服务器用的是同一道墙。轮次记录带 `env_hydrate_ms`，让成本可见。如果将来需要跨会话共享层，ACA 原生的形态是会话池把层烤进镜像，那是另一个产品决定；不承诺绑定挂载。
6. **隔离。** 容器就是边界：一个会话完全看不到另一个的容器，一个容器只持有其轮次指定的环境（水合只下载那一个层，不下载别的），这就是分组模型的结果，只是没有分组；层属于 root，agent 不是；源和层只由 API 和构建的写 SAS 写入。构建在自己的任务容器中运行，绝不在会话的容器中。
7. **版本、回滚、保留。** 顶点上的 `active_version` 是指针；网关在轮次 body 中传 `version`，使水合确定，并在会话上盖它启动时的版本，该会话后续轮次沿用。保留活跃版本加最近三个；环境删除时其 blob 一并删除；层计入 `/internal/storage-usage`。
8. **大小。** 临时磁盘还要放工作区，所以层上限是它的一部分，按计划而定，构建时根据记录的大小强制执行：超过上限的层构建失败，日志中带大小。导入 512 MiB 和文件 64 MiB 仍是可通过环境变量调整的上限；把数字写进计划前，先在会话中用 `df` 测量磁盘。

## 本切片不包含

- 环境级系统包（apt）：镜像提供 `ffmpeg`、LibreOffice、Playwright；需要更多的项目在 `setup.sh` 中说明，它以构建的权限运行。
- 超出组织工作区范围的跨工作区共享；环境市场。
- 从会话内部受控更新（SPI-006）：会话不能写环境；"把这些文件提升到下一版本"的流程是后续设计，带评审和版本管理。