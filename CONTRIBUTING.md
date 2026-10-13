# 贡献指南

欢迎贡献，也欢迎提问。

本项目由两个相互关联的开源层组成：统一 Harness 协议（UHP）——一套关于产品如何与 agent harness 协作的开放标准，以及本仓库——该标准的一个可用实现。两者都接受贡献：规范、一致性测试套件、harness 支持、实现以及文档。

## 验证 harness

能应答的 harness 不等于能正常工作的 harness。在 harness 支持的改动发布前，必须经过测量：每个 harness 和模型五个场景，外加一个自带技能和工具策略的自定义 harness，由代码中强制执行的规则判定，而非肉眼判断。[docs/harness-verification.md](docs/harness-verification.md) 说明了必须证明什么、每项声明如何测量、规则位于何处。在新增 harness 或修改其路由方式之前请先阅读该文档，并据此检查改动，而不是凭一次成功的对话轮次。

## 各类事项的去处

- 提案和 bug 报告：GitHub Issues。
- 讨论：HarnessRouter Discord 社区，https://discord.gg/nPcbwqVPb2
- 安全漏洞：按 [SECURITY.md](SECURITY.md) 私下报告，不要发在公开 Issue 中。

## 重大改动：先描述，再动手

对于任何超出小修复的改动，请先开一个 Issue。在编写实现之前，描述你要解决的问题、提议的改动及其预期影响。维护者认可方向后，由你或维护者来实现。

小而明显的修复，例如拼写错误或原因和修复都很明确的 bug，可以直接提 Pull Request。

## 协议改动

统一 Harness 协议有更严格的流程，因为规范、参考实现和一致性测试套件必须保持同步。如果你的改动涉及协议，请遵循 [protocol/GOVERNANCE.md](protocol/GOVERNANCE.md)：以 Issue 形式提交 UHP 增强提案（UEP），打上 `uep` 标签，包含问题、提案、兼容性和替代方案四部分。维护者在 10 个工作日内回复。被接受的 UEP 以一个 Pull Request 的形式发布，同时更新规范、Schema、参考实现、一致性测试和变更日志。

## Pull Request

每个 Pull Request 只解决一个问题。请求审查前：若存在对应 Issue 请链接；行为变更需新增或更新测试；公共 API、协议、配置或用户工作流变更需更新文档；说明兼容性、安全性或授权方面的影响；并确认没有夹带凭据、生成的依赖或无关文件。

### 描述 bug 修复

修复 bug 的 Pull Request 按以下四个标题描述，无论是否已有对应 Issue：

```markdown
## The bug
What goes wrong and how to see it: the version, the steps or the input, and what comes back.

## Expected behavior
What should happen instead.

## The fix
What the cause is and what you changed.

## Verification
The steps you ran and what each one showed, written so a reviewer can repeat them: the test you
added, the commands, and the result before and after the fix.
```

审查者先读这段描述再看 diff，并在合并前复现验证。一个在修复前失败、修复后通过的测试是最有力的验证；无法编写测试时，说明你手动运行了什么、看到了什么。

## 开发检查

运行与你改动区域对应的检查。

控制台：

```bash
cd ui
npm ci
npm run type-check
npm test
npm run build
```

网关：

```bash
python -m pytest gateway/tests
```

涉及容器层面的改动，在请求审查前先构建镜像并走一遍文档中记录的自托管流程。

## 许可证

提交贡献即表示你同意你的贡献以 Apache License 2.0 授权，与本仓库的许可证相同。

## 改动如何进入 main

`main` 只接受 Pull Request：一人批准，所有检查通过，任何人都不例外，包括管理员。Pull Request 不能由作者本人批准，因此通过维护者账号工作的 agent 需改用 `open-pr` 工作流提交 Pull Request，该工作流以 `github-actions[bot]` 身份创建；维护者随后像审查其他改动一样审查并批准。

```bash
gh workflow run open-pr.yml -f branch=<branch> -f title="<title>" -f body="<body>"
git commit --allow-empty -m "ci: start checks" && git push   # a bot-opened PR needs one push to run CI
```