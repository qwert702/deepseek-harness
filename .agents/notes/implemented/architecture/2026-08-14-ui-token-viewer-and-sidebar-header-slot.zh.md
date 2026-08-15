# Agent Note：Token 消耗展示与侧边栏 header 插槽

Status: implemented

[English](2026-08-14-ui-token-viewer-and-sidebar-header-slot.md) | 中文

## 问题

用户希望不离开 Web GUI 就能看到某个会话（或整个实例）消耗了多少 token。host 侧已经算好了这些数字：token-meter 把 `tokenUsage` / `contextPressure` / `contextBreakdown` 发布为按会话的投影，运行时又把每个会话的投影值发布到会话列表行上。缺的只是读取它们的界面——以及侧边栏里的一个挂载点：侧边栏外壳（`dsh-client-ui-sidebar`）只声明了 `sidebar.workspaces`（单槽）、`sidebar.settings` 与 `sidebar.footer.action`，工作区浏览区上方没有任何可供全局卡片挂载的插槽。

## 决策

新增客户端插件包 `@deepseek-ai/dsh-client-ui-token-viewer`，包含两个投影模式界面加一条 host 路由（无 store、无事件监听；唯一网络是余额读取）：

- `TokenDock` 注册进已有的 `conversation.input.dock` 列表（order 20），展示当前会话的计费输入、输出、缓存命中率与近似上下文占用率，悬停气泡给出完整计费明细。
- `SidebarTokenPanel` 注册进由侧边栏外壳新声明的 `sidebar.workspaces.header` 插槽（渲染在工作区浏览区上方），汇总所有会话行 `projectionValues` 中的 `tokenUsage`（计费输入、输出、缓存命中率、上报会话数）。侧边栏收起时隐藏。同时展示 DeepSeek 账号余额，并可展开为按会话明细列表：每个会话的计费输入/输出，按总量降序，点击行通过注入的 `sessions` 服务打开该会话。
- node 半区注册 `GET /api/billing/balance`：通过凭据服务解析 `DEEPSEEK_API_KEY` 凭据引用（与 LLM 适配器使用同一引用），代理 DeepSeek 的 `/user/balance`，只回余额数字——API key 永不离开服务器。余额行请求该路由；失败显示错误重试，未配置 key 时返回 503 `no-api-key`，卡片隐藏而非刷屏报错。

ui-sidebar 的改动是增量且最小的：一个子插槽声明（`kind: 'single', scope: 'root'`）、一个置于工作区区域上方的 `renderSlot('sidebar.workspaces.header', { wide })` 调用，以及对应的 `SlotMap` / owner props / `PropsRenderSlots` 契约条目。该插槽是平台缝隙而非 token-viewer 特例：任何应当位于浏览区域上方的全局卡片都可使用。

## 备选方案

**在 token-viewer 包内 fork 侧边栏外壳并禁用原行。** 本地可用，但会重复整个外壳 bundle、强制 profile 级 `disabled` 覆盖，并且上游不会接受：重复声明 `sidebar` 会抛错，fork 的 bundle 也会与官方外壳漂移。

**注册进 `sidebar.footer.action`（或 `sidebar.settings`）。** 两者都在侧边栏底部——用户要求的放置位置是工作区上方，且底部承载交互控件，卡片会显得拥挤。

**以更低优先级遮蔽 `sidebar.workspaces`。** 单槽只渲染一个条目；遮蔽会整体替换工作区浏览器，而不是加一张卡片。

**在客户端直接读 `ctx.tokenMeter.measure()`。** meter 服务属于 host 平面；浏览器通过会话投影 store 读取投影值，这正是这两个界面所消费的。

**从浏览器直接查询提供方余额。** API key 是服务端凭据；客户端调用会暴露它。host 路由通过凭据服务解析 key，只返回余额数字。

## 后果

用户无需离开 GUI 即可看到实时会话级与整实例级 token 消耗，且数据与压缩、占用率展示使用的 host 计算口径一致，另加提供方计费端点的账号余额。web profile 增加一行 `dsh.client` 与一个依赖；合并后需重新生成 `pnpm-lock.yaml`。包纯展示：不添加提示词内容、工具、消息或提供方请求，因此无模型或 KV 缓存影响（余额路由是服务端 fetch，不是模型调用）。余额为 DeepSeek 专属——路由调用 DeepSeek 的 `/user/balance`，且只展示 `balance_infos` 首项。侧边栏卡片依赖 header 插槽；若组合层替换了不带该插槽的 ui-sidebar，卡片会静默消失，而 dock 条仍正常（已记入 Known Limitations）。
