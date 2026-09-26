[![dshfind](https://dshfind.com/api/card/lehhair/dsh-html-artifact?lang=zh)](https://dshfind.com/zh/plugins/lehhair/dsh-html-artifact?ref=badge)

# dsh-html-artifact

HTML 实时渲染插件：模型通过专门的 `artifact` 工具创建/修补 HTML 文档，GUI 在沙箱 iframe 里**实时渲染**。核心区别：模型**打 patch**（`old_string`/`new_string`，和文件 `edit` 工具同一套语义），而不是每次重发整个 HTML —— 只传增量，预览原地更新。

纯扩展实现，**不改 harness core**：

- 服务端：`artifact` 工具（create / patch / read / destroy / list），per-session 注册表；`card: 'artifact'` 渲染意图通过 `presentResult` 走线（core 的 union 不认识这个 card，返回值做了边界强转，客户端未知 card 自动降级 generic）。
- 客户端：`tool.call.toolview` keyed slot 注册 `artifact` 行的原子视图，沙箱 iframe（CSP `default-src 'none'` + `sandbox="allow-scripts"` 不透明源 + 主题变量 + 尺寸桥 + 存储 shim），同一 artifact id 的后续 patch 调用原地更新预览。

## Fork 说明（dsh 0.1.5-rc.3 兼容版）

上游发布版是针对 rc.5 时期的 harness master 构建的，其客户端代码 require `@deepseek-ai/dsh-client-runtime/client`（`resultView` 字段同源）——这些在**任何已发布的 dsh 版本**（含 0.1.5-rc.3 与 0.1.7-rc.2）的客户端模块表里都不存在，加载必然报 `missed the module table`。

本 fork 把插件移植到 **dsh 0.1.5-rc.3（npm latest 稳定版）**：

- 客户端外部依赖收敛到 rc.3 的 SPA 种子表（`react` 全家、`cordis`、`dsh-client-store`、`dsh-client-ui-slots`、`dsh-client-ui-primitives`、`dsh-client-ui-dockkit`），不再依赖任何未发布的 runtime 包，`client.inject` 随之为空；
- `conversationContextKey`、`RunningToolCall`、`ToolCallBlock`、`ChatNodeViewProps`、`ChatNodeDataMap` 等改从 rc.3 的 `dsh-client-ui-conversation` / `dsh-client-ui-chat` 取用；
- 流式事件 `assistant/chunk` → `assistant/live-chunk`；会话服务 `conversationEvents` → `uiConversation`（`ctx.uiConversation.events.register`）；
- 结算工具节点的卡片视图从 `block.resultView` 改读 rc.3 的 `block.meta`（宿主侧 `output.presentationMeta` 机制不变）；
- CI 构建锁定 `dsh-v0.1.5-rc.3` tag，发布产物与 rc.3 运行时逐位对应。

安装（替换上游链接即可）：

```sh
dsh plugin --profile web add "https://github.com/Weihong-Liu/dsh-html-artifact/releases/latest/download/dsh-external-dsh-html-artifact.tgz"
```

## 安装

发布版走 GitHub Releases 的 tarball（`dsh plugin` 安装后自动加入 profile 层栈），`releases/latest` 永远指向最新版本，链接不用随版本改动：

```sh
dsh plugin --profile web add "https://github.com/lehhair/dsh-html-artifact/releases/latest/download/dsh-external-dsh-html-artifact.tgz"
```

然后重启 dsh web 服务，刷新页面。**不要**用 `dsh plugin add github:lehhair/dsh-html-artifact` 安装源码（仓库的 `lib/` 被 gitignore，源码安装没有构建产物）。

> ⚠️ 升级注意：pnpm 会按 URL 缓存 tarball——同一 `latest` 链接在新版本发布后可能命中旧缓存。装到旧版时先 `dsh plugin --profile web remove @dsh-external/dsh-html-artifact` 再重新安装（必要时 `pnpm store prune`）。

## 用法

模型侧（工具描述已内置）：

1. `artifact create { title?, html? }` —— 创建，GUI 立即渲染沙箱预览。
2. `artifact patch { id, old_string, new_string, replace_all? }` —— 对已有 artifact 打补丁，只发变更；每个 patch 调用结算后预览原地更新，会话里显示一行旧/新对比。
3. `artifact read { id }` / `artifact destroy { id }` / `artifact list` —— 读取源码 / 关闭 / 枚举。

安全边界：预览在 `srcdoc` iframe 中，CSP `default-src 'none'`（脚本/样式允许内联，外部 https/http 资源允许），iframe 无 `allow-same-origin`（不透明源），artifact 脚本无法触达宿主文档；localStorage/sessionStorage 被内存 shim 替代。

## 交互数据提交

预览底部 **Submit interaction** 按钮会把用户在 artifact 内的交互数据注入当前会话（`Agent.followup`，agent 立即读取分析）。收集三层：

1. **显式协议（推荐，游戏/计数器等内部状态）**：artifact 脚本把值得提交的状态写成 JSON 值并随状态变化更新：
   ```js
   window.__dshArtifactData = { score: currentScore, moves: moves }
   ```
   工具描述里已内置该指令，模型生成带内部状态的 artifact 时会自动写。
2. **表单控件（自动）**：`input` / `textarea` / `select` / `checkbox` / `radio` 的当前值按 name/id 收集。
3. **按钮点击（自动）**：点击过的按钮（`data-artifact-action` 优先，否则按钮文本）与次数自动记录。

## 开发

```sh
pnpm install        # devDeps 是 link: 到 ../dsh2026/deepseek-harness 的本地 checkout
pnpm run check      # typecheck + test + build
```

测试：`tests/registry.spec.ts`（服务端纯逻辑：替换语义、字节上限、截断、错误分类）+ `tests/artifact-row.spec.tsx`（客户端渲染：沙箱 iframe、patch 对比、跨调用实时更新、destroy/list/read 分支）。

CI（`.github/workflows/build-release.yml`）：发布时 checkout 固定 ref 的 harness 构建后打包 tarball 上传到 Release。注意：**同 URL 重建 release 会命中 pnpm 的 URL 缓存**，出新版本务必 bump version。
