# 示例

Unified Harness Protocol 的示例实现，按各自承担的[角色](versions/2026-09-12/architecture.md#1-roles)分组。列表由社区通过 Pull Request 维护，**不代表认可或认证**——唯一有意义的合规声明是[通过合规测试套件](conformance/README.md)。

<div class="impl-actions">
<a class="impl-cta" href="#listing-criteria">添加实现</a>
<a class="impl-crit" href="#listing-criteria">列表标准</a>
</div>

## 示例服务器

服务器在契约背后运行一个或多个 harness。指南：[实现服务器](SERVING.md)。

<div class="impl-list">

<div class="impl-card" id="harnessrouter-ce">
<div class="impl-head"><a href="https://github.com/HarnessRouter/harnessrouter"><b>HarnessRouter Community Edition</b></a><span class="role role-server">服务器</span></div>
<p>开源运行器，将现有 harness——Codex、Claude Code、Hermes——置于 UHP 契约背后。HarnessRouter · Apache-2.0。</p>
<p class="impl-meta">UHP 2026-09-12</p>
</div>

<div class="impl-card" id="harnessrouter">
<div class="impl-head"><a href="https://harnessrouter.ai"><b>HarnessRouter</b></a><span class="role role-server">服务器</span></div>
<p>托管服务：十六个 harness 基础置于 UHP 契约背后，提供控制台、计费和共享会话，地址为 <code>https://api.harnessrouter.ai</code>。HarnessRouter · 商业产品，提供免费套餐。</p>
<p class="impl-meta">UHP 2026-09-12</p>
</div>

<div class="impl-card" id="superqode">
<div class="impl-head"><a href="https://github.com/SuperagenticAI/superqode"><b>SuperQode</b></a><span class="role role-server">服务器</span></div>
<p>Harness 工程框架，其自带 harness 通过 <code>superqode serve uhp</code> 原生支持 UHP，绑定一个已配置的 HarnessSpec。SuperagenticAI · Apache-2.0。</p>
<p class="impl-meta">UHP 2026-09-12</p>
</div>

</div>

## 示例客户端

客户端驱动 UHP 服务器。指南：[实现客户端](CONNECTING.md)。

<div class="impl-list">

<div class="impl-card" id="superqode-client">
<div class="impl-head"><a href="https://github.com/SuperagenticAI/superqode"><b>SuperQode</b></a><span class="role role-client">客户端</span></div>
<p>Harness 工程框架，通过 <code>superqode connect uhp</code> 连接到 UHP 服务器。SuperagenticAI · Apache-2.0。</p>
<p class="impl-meta">UHP 2026-09-12</p>
</div>

</div>

## 列表标准

列表是事实性条目，不是认证。在[仓库](https://github.com/HarnessRouter/harnessrouter)中针对本文件提交 Pull Request，添加一张卡片。要合并，条目需要：

1. 公开可用的发布版本——一个仓库或产品页面，而非公告。
2. 该实现所针对的 UHP 版本。
3. 可从所链接页面联系到的维护者联系方式。
4. 所承担的角色：**服务器**（在契约背后运行 harness）、**客户端**（驱动 UHP 服务器）、**SDK**（用于构建前两者的库），或组合。

只有当 UHP 为该实现发布可复现的测试证据后，列表上才会显示合规级别；在此之前，没有级别是正常的，并非对该条目的不利标记。证据是由 `conformance-measure` 工作流生成并归档到 `conformance/reports/<slug>/` 下的报告；为你的卡片设置 `id="<slug>"`，网站便会根据报告在其上显示测得的级别和徽章。如何获得测量，以及要嵌入的徽章，见[合规页面](conformance/README.md#measured-implementations)。

正在构建实现？从[实现服务器](SERVING.md)或[实现客户端](CONNECTING.md)开始。