# @deepseek-ai/dsh-client-ui-token-viewer

[English](README.md) | 中文

Token 消耗展示插件，浏览器半区：两个只读界面，数据来自 host 侧已算好的 token-meter 会话投影（`tokenUsage`、`contextPressure`、`contextBreakdown`），因此本插件不拥有领域 store、刷新链或事件监听。

- **`TokenDock`** 注册于 `conversation.input.dock`（order 20，位于 Goal 之后）。展示当前会话的消耗——计费输入（未缓存 + 缓存读 + 缓存写）、输出、缓存命中率，以及近似上下文占用率（`projectedTokens / contextWindow`，带迷你进度条）。悬停气泡给出完整计费明细。在提供方上报用量之前不渲染任何内容。
- **`SidebarTokenPanel`** 注册于 `sidebar.workspaces.header`——由 ui-sidebar 外壳声明在工作区浏览区上方的一个插槽。它汇总所有会话行 `projectionValues` 中的 `tokenUsage`——计费输入、输出、缓存命中率，以及上报了用量的会话数——因此是整实例总量而非单会话读数。在任一会话上报用量之前不渲染；侧边栏收起（`wide === false`）时也不渲染。

`/client` 导出为插件主体（`apply`/`inject`）与组合后的 props 类型。

## 模型体验

无。这两个界面是对 host 已算好的投影值的纯展示；不添加提示词内容、工具、消息或提供方请求。

#### KV Cache 影响

无；本插件既不组装也不发送提供方请求。

## 已知限制与暂缓事项

- **启发式近似**——缓存命中率与上下文占用率继承 token-meter 固定的「4 字符 ≈ 1 token」密度估计（凡提供方未计费的内容都按此计价）；CJK 文本与 JSON schema 会被系统性低估。占用率是面向用户的参考数字，不是计费或门控输入（见 token-meter README）。
- **侧边栏卡片依赖 ui-sidebar 的 header 插槽**——只有外壳声明 `sidebar.workspaces.header` 时才渲染；若组合层替换了不带该插槽的 ui-sidebar，卡片会静默消失，而 dock 条仍正常。
