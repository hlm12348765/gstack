# Browser — 技术细节

本文档介绍 gstack 无头浏览器的命令参考和内部实现。

## 命令参考

| 分类 | Commands | 用途 |
|------|----------|------|
| Navigate | `goto`, `back`, `forward`, `reload`, `url` | 到达某个页面 |
| Read | `text`, `html`, `links`, `forms`, `accessibility` | 提取内容 |
| Snapshot | `snapshot [-i] [-c] [-d N] [-s sel] [-D] [-a] [-o] [-C]` | 获取 refs、diff、标注 |
| Interact | `click`, `fill`, `select`, `hover`, `type`, `press`, `scroll`, `wait`, `viewport`, `upload` | 操作页面 |
| Inspect | `js`, `eval`, `css`, `attrs`, `is`, `console`, `network`, `dialog`, `cookies`, `storage`, `perf` | 调试与验证 |
| Visual | `screenshot [--viewport] [--clip x,y,w,h] [sel\|@ref] [path]`, `pdf`, `responsive` | 看见 Claude 所看到的内容 |
| Compare | `diff <url1> <url2>` | 比较不同环境之间的差异 |
| Dialogs | `dialog-accept [text]`, `dialog-dismiss` | 控制 alert/confirm/prompt |
| Tabs | `tabs`, `tab`, `newtab`, `closetab` | 多页面工作流 |
| Cookies | `cookie-import`, `cookie-import-browser` | 从文件或真实浏览器导入 cookies |
| Multi-step | `chain`（从 stdin 读取 JSON） | 一次调用批量执行多个命令 |
| Handoff | `handoff [reason]`, `resume` | 切换到可见 Chrome 供用户接管 |

所有 selector 参数都支持 CSS selectors、`snapshot` 之后的 `@e` refs，
以及 `snapshot -C` 之后的 `@c` refs。总计 50+ 条命令，外加 cookie 导入。

## 工作原理

gstack 的浏览器是一个编译后的 CLI 二进制，它通过 HTTP 与一个持久化的本地 Chromium daemon 通信。
CLI 是一个薄客户端：读取状态文件、发送命令、把响应打印到 stdout。
真正的工作由服务端通过 [Playwright](https://playwright.dev/) 完成。

```
┌─────────────────────────────────────────────────────────────────┐
│  Claude Code                                                    │
│                                                                 │
│  "browse goto https://staging.myapp.com"                        │
│       │                                                         │
│       ▼                                                         │
│  ┌──────────┐    HTTP POST     ┌──────────────┐                 │
│  │ browse   │ ──────────────── │ Bun HTTP     │                 │
│  │ CLI      │  localhost:rand  │ server       │                 │
│  │          │  Bearer token    │              │                 │
│  │ 编译后的 │ ◄──────────────  │  Playwright  │──── Chromium    │
│  │ 二进制   │  纯文本响应      │  API calls   │    (headless)   │
│  └──────────┘                  └──────────────┘                 │
│   启动约 1ms                   持久化 daemon                    │
│                                 首次调用自动启动                │
│                                 空闲 30 分钟后自动停止          │
└─────────────────────────────────────────────────────────────────┘
```

### 生命周期

1. **首次调用：** CLI 检查项目根目录中的 `.gstack/browse.json`，看是否已有运行中的服务端。若没有，则在后台启动 `bun run browse/src/server.ts`。服务端通过 Playwright 启动 headless Chromium，随机选择端口（10000-60000），生成 bearer token，写入状态文件，并开始接收 HTTP 请求。整个过程约 3 秒。

2. **后续调用：** CLI 读取状态文件，带着 bearer token 发送 HTTP POST，并打印响应。往返约 100-200ms。

3. **空闲关闭：** 30 分钟无命令后，服务端自动关闭并清理状态文件。下次调用会自动重启。

4. **崩溃恢复：** 如果 Chromium 崩溃，服务端会立即退出（不做自愈，不掩盖失败）。CLI 会在下一次调用时检测到服务已死亡，并启动一个新的实例。

### 核心组件

```
browse/
├── src/
│   ├── cli.ts              # 薄客户端：读取状态文件、发送 HTTP、打印响应
│   ├── server.ts           # Bun.serve HTTP 服务：把命令路由给 Playwright
│   ├── browser-manager.ts  # Chromium 生命周期：启动、标签页、ref map、崩溃处理
│   ├── snapshot.ts         # Accessibility tree → @ref 分配 → Locator map + diff/annotate/-C
│   ├── read-commands.ts    # 非变更命令（text、html、links、js、css、is、dialog 等）
│   ├── write-commands.ts   # 变更命令（click、fill、select、upload、dialog-accept 等）
│   ├── meta-commands.ts    # 服务端管理、chain、diff、snapshot 路由
│   ├── cookie-import-browser.ts  # 从真实 Chromium 浏览器解密并导入 cookies
│   ├── cookie-picker-routes.ts   # 交互式 cookie picker UI 的 HTTP 路由
│   ├── cookie-picker-ui.ts       # 自包含的 HTML/CSS/JS cookie picker
│   └── buffers.ts          # CircularBuffer<T> + console/network/dialog 捕获
├── test/                   # 集成测试与 HTML fixtures
└── dist/
    └── browse              # 编译后的二进制（约 58MB，Bun --compile）
```

### Snapshot 系统

浏览器最关键的创新，是基于 ref 的元素选择机制，它建立在 Playwright 的 accessibility tree API 之上：

1. `page.locator(scope).ariaSnapshot()` 返回一个类似 YAML 的 accessibility tree
2. snapshot parser 为每个元素分配 ref（`@e1`, `@e2`, ...）
3. 对每个 ref，构建一个 Playwright `Locator`（使用 `getByRole` + nth-child）
4. ref 到 Locator 的映射保存在 `BrowserManager` 上
5. 后续像 `click @e3` 这样的命令，会查出对应 Locator 并调用 `locator.click()`

不改 DOM。不注入脚本。完全使用 Playwright 原生的 accessibility API。

**Ref 过期检测：** SPA 可以在不导航的情况下变更 DOM（React router、tab 切换、modal 打开等）。这时，从之前 `snapshot` 收集到的 refs 可能指向已经不存在的元素。为解决这个问题，`resolveRef()` 会在使用 ref 之前异步执行一次 `count()` 检查：如果元素数量为 0，就立即抛错，并提示代理重新执行 `snapshot`。这样会快速失败（约 5ms），而不是等 Playwright 的 30 秒 action timeout。

**扩展 snapshot 功能：**
- `--diff`（`-D`）：把每次 snapshot 存为 baseline。下一次 `-D` 调用时，返回一个 unified diff，显示发生了什么变化。可用于验证点击、填写等动作是否真的生效。
- `--annotate`（`-a`）：在每个 ref 的 bounding box 上临时注入 overlay div，拍摄带 ref 标签的截图，然后移除 overlay。用 `-o <path>` 指定输出路径。
- `--cursor-interactive`（`-C`）：通过 `page.evaluate` 扫描不在 ARIA tree 中、但可交互的元素（如带 `cursor:pointer`、`onclick`、`tabindex>=0` 的 div），并分配 `@c1`, `@c2`... refs，使用可复现的 `nth-child` CSS selectors。用于捕获 ARIA tree 漏掉但用户仍然能点击的元素。

### Screenshot 模式

`screenshot` 命令支持四种模式：

| 模式 | 语法 | Playwright API |
|------|------|----------------|
| 整页（默认） | `screenshot [path]` | `page.screenshot({ fullPage: true })` |
| 仅 viewport | `screenshot --viewport [path]` | `page.screenshot({ fullPage: false })` |
| 元素裁剪 | `screenshot "#sel" [path]` 或 `screenshot @e3 [path]` | `locator.screenshot()` |
| 区域裁剪 | `screenshot --clip x,y,w,h [path]` | `page.screenshot({ clip })` |

元素裁剪支持 CSS selectors（`.class`、`#id`、`[attr]`）和来自 `snapshot` 的 `@e` / `@c` refs。
自动识别规则：`@e` / `@c` 前缀表示 ref，`.` / `#` / `[` 前缀表示 CSS selector，`--` 前缀表示 flag，其余则视为输出路径。

互斥规则：`--clip` + selector 以及 `--viewport` + `--clip` 都会报错。未知 flag（例如 `--bogus`）也会报错。

### 身份验证

每个服务端会话都会生成一个随机 UUID 作为 bearer token。该 token 会以 chmod 600 的权限写入状态文件（`.gstack/browse.json`）。每个 HTTP 请求都必须包含 `Authorization: Bearer <token>`。这样可以防止同一台机器上的其他进程控制你的浏览器。

### Console、network 和 dialog 捕获

服务端会挂载 Playwright 的 `page.on('console')`、`page.on('response')` 和 `page.on('dialog')` 事件。所有条目都保存在容量为 50,000 的 O(1) circular buffer 中，并通过 `Bun.write()` 异步刷盘：

- Console: `.gstack/browse-console.log`
- Network: `.gstack/browse-network.log`
- Dialog: `.gstack/browse-dialog.log`

`console`、`network` 和 `dialog` 命令读取的是内存中的 buffers，而不是磁盘。

### 用户接管（handoff）

当 headless 浏览器无法继续（CAPTCHA、MFA、复杂登录）时，`handoff` 会打开一个可见的 Chrome 窗口，停留在完全相同的页面上，并保留所有 cookies、localStorage 和标签页。用户手动完成问题处理后，`resume` 会把控制权交还给代理，并附带一份新的 snapshot。

```bash
$B handoff "卡在登录页的 CAPTCHA"   # 打开可见 Chrome
# 用户解决 CAPTCHA...
$B resume                            # 返回 headless，并生成新的 snapshot
```

浏览器在连续 3 次失败后会自动建议 `handoff`。切换前后状态完整保留，不需要重新登录。

### Dialog 处理

Dialogs（alert、confirm、prompt）默认会自动接受，以防浏览器卡死。`dialog-accept` 和 `dialog-dismiss` 可用于控制该行为。对于 prompt，`dialog-accept <text>` 会提供响应文本。所有 dialogs 都会记录到 dialog buffer 中，包含类型、消息和采取的动作。

### JavaScript 执行（`js` 和 `eval`）

`js` 用于执行单个表达式，`eval` 用于执行一个 JS 文件。两者都支持 `await`，凡是包含 `await` 的表达式都会被自动包进异步上下文：

```bash
$B js "await fetch('/api/data').then(r => r.json())"  # 可用
$B js "document.title"                                # 也可用（不需要包裹）
$B eval my-script.js                                  # 文件里有 await 也可用
```

对于 `eval` 文件，单行文件会直接返回表达式值。多行文件在使用 `await` 时需要显式 `return`。仅包含 “await” 文本的注释不会触发自动包裹。

### 多工作区支持

每个 workspace 都有自己隔离的浏览器实例，拥有独立的 Chromium 进程、标签页、cookies 和日志。状态保存在项目根目录下的 `.gstack/` 中（通过 `git rev-parse --show-toplevel` 检测）。

| Workspace | State file | Port |
|-----------|------------|------|
| `/code/project-a` | `/code/project-a/.gstack/browse.json` | 随机（10000-60000） |
| `/code/project-b` | `/code/project-b/.gstack/browse.json` | 随机（10000-60000） |

没有端口冲突。没有共享状态。每个项目完全隔离。

### 环境变量

| Variable | Default | 说明 |
|----------|---------|------|
| `BROWSE_PORT` | 0（随机 10000-60000） | HTTP 服务端固定端口（调试覆盖） |
| `BROWSE_IDLE_TIMEOUT` | 1800000（30 分钟） | 空闲关闭超时，单位毫秒 |
| `BROWSE_STATE_FILE` | `.gstack/browse.json` | 状态文件路径（CLI 会传给服务端） |
| `BROWSE_SERVER_SCRIPT` | 自动检测 | `server.ts` 路径 |

### 性能

| Tool | 首次调用 | 后续调用 | 每次调用的上下文开销 |
|------|----------|----------|----------------------|
| Chrome MCP | ~5s | ~2-5s | ~2000 tokens（schema + protocol） |
| Playwright MCP | ~3s | ~1-3s | ~1500 tokens（schema + protocol） |
| **gstack browse** | **~3s** | **~100-200ms** | **0 tokens**（纯文本 stdout） |

上下文开销的差异会迅速累积。在一个 20 条命令的浏览器会话中，MCP 工具单在协议封装上就会烧掉 30,000-40,000 tokens。gstack 为零。

### 为什么选择 CLI，而不是 MCP？

MCP（Model Context Protocol）很适合远程服务，但对本地浏览器自动化来说，它只增加了纯开销：

- **上下文膨胀：** 每次 MCP 调用都包含完整的 JSON schemas 和协议封装。一个简单的“读取页面文本”，上下文 token 成本会变成应有值的 10 倍。
- **连接脆弱：** 持久化的 WebSocket/stdio 连接会断，重连也不稳定。
- **不必要的抽象：** Claude Code 已经自带 Bash 工具。一个把结果打印到 stdout 的 CLI，就是最简单的接口。

gstack 跳过了这些层。编译后的二进制。纯文本输入，纯文本输出。没有协议，没有 schema，没有连接管理。

## 致谢

浏览器自动化层建立在 Microsoft 的 [Playwright](https://playwright.dev/) 之上。
Playwright 的 accessibility tree API、locator system 和 headless Chromium 管理能力，
让 ref-based interaction 成为可能。snapshot 系统——给 accessibility tree 节点分配 `@ref`
标签，并把它们映射回 Playwright Locators——完全建立在 Playwright 的原语之上。
感谢 Playwright 团队打下如此扎实的基础。

## 开发

### 前置条件

- [Bun](https://bun.sh/) v1.0+
- Playwright 的 Chromium（`bun install` 会自动安装）

### 快速开始

```bash
bun install              # 安装依赖 + Playwright Chromium
bun test                 # 运行集成测试（约 3 秒）
bun run dev <cmd>        # 从源码运行 CLI（不编译）
bun run build            # 编译到 browse/dist/browse
```

### 开发模式 vs 编译后的二进制

开发时，使用 `bun run dev`，而不是编译后的二进制。它会直接通过 Bun 运行 `browse/src/cli.ts`，因此无需编译即可即时反馈：

```bash
bun run dev goto https://example.com
bun run dev text
bun run dev snapshot -i
bun run dev click @e3
```

编译后的二进制（`bun run build`）只在分发时需要。它会使用 Bun 的 `--compile` flag，在 `browse/dist/browse` 生成一个约 58MB 的单文件可执行程序。

### 运行测试

```bash
bun test                         # 运行所有测试
bun test browse/test/commands              # 只运行命令集成测试
bun test browse/test/snapshot              # 只运行 snapshot 测试
bun test browse/test/cookie-import-browser # 只运行 cookie import 单元测试
```

测试会启动一个本地 HTTP 服务（`browse/test/test-server.ts`），从 `browse/test/fixtures/` 提供 HTML fixtures，然后针对这些页面执行 CLI 命令。总计 203 个测试，分布在 3 个文件中，总耗时约 15 秒。

### 源码地图

| File | 角色 |
|------|------|
| `browse/src/cli.ts` | 入口。读取 `.gstack/browse.json`，向服务端发 HTTP，并打印响应。 |
| `browse/src/server.ts` | Bun HTTP 服务端。把命令路由到合适的 handler。管理 idle timeout。 |
| `browse/src/browser-manager.ts` | Chromium 生命周期：启动、tab 管理、ref map、崩溃检测。 |
| `browse/src/snapshot.ts` | 解析 accessibility tree，分配 `@e` / `@c` refs，构建 Locator map。处理 `--diff`、`--annotate`、`-C`。 |
| `browse/src/read-commands.ts` | 非变更命令：`text`、`html`、`links`、`js`、`css`、`is`、`dialog`、`forms` 等。导出 `getCleanText()`。 |
| `browse/src/write-commands.ts` | 变更命令：`goto`、`click`、`fill`、`upload`、`dialog-accept`、`useragent`（带 context 重建）等。 |
| `browse/src/meta-commands.ts` | 服务端管理、chain 路由、diff（通过 `getCleanText` 复用 DRY）、snapshot 委派。 |
| `browse/src/cookie-import-browser.ts` | 通过 macOS Keychain + PBKDF2/AES-128-CBC 解密 Chromium cookies。自动检测已安装浏览器。 |
| `browse/src/cookie-picker-routes.ts` | `/cookie-picker/*` 的 HTTP 路由——浏览器列表、domain 搜索、导入、移除。 |
| `browse/src/cookie-picker-ui.ts` | 交互式 cookie picker 的自包含 HTML 生成器（暗色主题，无框架）。 |
| `browse/src/buffers.ts` | `CircularBuffer<T>`（O(1) ring buffer）+ console/network/dialog 捕获与异步刷盘。 |

### 部署到当前正在使用的 skill

当前启用的 skill 位于 `~/.claude/skills/gstack/`。完成修改后：

1. 推送你的分支
2. 在 skill 目录中拉取更新：`cd ~/.claude/skills/gstack && git pull`
3. 重新构建：`cd ~/.claude/skills/gstack && bun run build`

或者直接复制二进制：`cp browse/dist/browse ~/.claude/skills/gstack/browse/dist/browse`

### 添加新命令

1. 在 `read-commands.ts`（非变更）或 `write-commands.ts`（变更）中添加 handler
2. 在 `server.ts` 中注册路由
3. 在 `browse/test/commands.test.ts` 中添加测试用例，如有需要也添加 HTML fixture
4. 运行 `bun test` 验证
5. 运行 `bun run build` 编译
