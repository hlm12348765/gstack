# Architecture

本文档解释 gstack **为什么** 要这样构建。关于安装和命令，请看 `CLAUDE.md`。关于贡献流程，请看 `CONTRIBUTING.md`。

## 核心想法

gstack 给 Claude Code 提供了一个持久化浏览器，以及一组带有明确立场的工作流 skills。浏览器是最难的部分，其余几乎都只是 Markdown。

关键洞察是：AI 代理如果要和浏览器交互，就必须同时具备 **亚秒级延迟** 和 **持久化状态**。如果每条命令都要冷启动浏览器，你每次工具调用都要等 3-5 秒。如果浏览器在命令之间死掉，你会丢失 cookies、标签页和登录会话。因此 gstack 运行的是一个长生命周期的 Chromium daemon，而 CLI 通过 localhost HTTP 与它通信。

```
Claude Code                     gstack
─────────                      ──────
                               ┌──────────────────────┐
  Tool call: $B snapshot -i    │  CLI（编译后的二进制） │
  ─────────────────────────→   │  • 读取状态文件        │
                               │  • POST /command      │
                               │    到 localhost:PORT  │
                               └──────────┬───────────┘
                                          │ HTTP
                               ┌──────────▼───────────┐
                               │  Server (Bun.serve)  │
                               │  • 分发命令           │
                               │  • 与 Chromium 通信   │
                               │  • 返回纯文本         │
                               └──────────┬───────────┘
                                          │ CDP
                               ┌──────────▼───────────┐
                               │ Chromium（headless） │
                               │ • 持久化标签页        │
                               │ • cookies 会延续     │
                               │ • 30 分钟空闲超时    │
                               └──────────────────────┘
```

第一次调用会启动整套东西（约 3 秒）。之后每次调用：约 100-200ms。

## 为什么选择 Bun

Node.js 也能做。但在这里，Bun 更好，原因有四点：

1. **编译后的二进制。** `bun build --compile` 可以产出一个约 58MB 的单文件可执行程序。运行时不需要 `node_modules`，不需要 `npx`，不需要配置 PATH。二进制直接就能运行。这一点很重要，因为 gstack 安装在 `~/.claude/skills/` 里，而用户不会期待自己要管理一个 Node.js 项目。

2. **原生 SQLite。** Cookie 解密需要直接读取 Chromium 的 SQLite cookie 数据库。Bun 内建 `new Database()`，不需要 `better-sqlite3`、不需要原生 addon 编译、不需要 gyp。少一层跨机器出问题的东西。

3. **原生 TypeScript。** 开发时服务端直接通过 `bun run server.ts` 运行。不需要编译步骤，不需要 `ts-node`，也没有一堆要调试的 source maps。编译后的二进制用于部署；源码文件用于开发。

4. **内建 HTTP 服务端。** `Bun.serve()` 快、简单，不需要 Express 或 Fastify。整个服务端大概也就 10 条路由。引入框架反而是额外负担。

瓶颈始终是 Chromium，不是 CLI，也不是服务端。Bun 的启动速度（编译后二进制约 1ms，而 Node 约 100ms）当然不错，但这不是我们选择它的核心原因。核心原因是：编译后二进制，以及原生 SQLite。

## Daemon 模型

### 为什么不为每条命令都单独启动一个浏览器？

Playwright 启动 Chromium 大约要 2-3 秒。对于单次截图来说，这还行。但对于一个包含 20+ 条命令的 QA 会话来说，这意味着 40+ 秒的浏览器启动开销。更糟的是：命令之间所有状态都会消失。Cookies、localStorage、登录会话、已打开的标签页，全都没了。

Daemon 模型意味着：

- **持久化状态。** 登录一次，就一直保持登录。开一个标签页，它就一直开着。localStorage 会在命令之间保留。
- **亚秒级命令。** 第一次调用之后，每条命令都只是一个 HTTP POST。包括 Chromium 工作在内的往返时间约 100-200ms。
- **自动生命周期管理。** 首次使用自动启动，空闲 30 分钟自动关闭。无需手动管理进程。

### 状态文件

服务端会写入 `.gstack/browse.json`（通过 tmp + rename 原子写入，权限为 0o600）：

```json
{ "pid": 12345, "port": 34567, "token": "uuid-v4", "startedAt": "...", "binaryVersion": "abc123" }
```

CLI 通过读取这个文件来定位服务端。如果文件不存在、已过期，或者 PID 已死亡，CLI 就会启动一个新的服务端。

### 端口选择

随机在 10000-60000 之间选取端口（冲突时最多重试 5 次）。这意味着 10 个 Conductor workspaces 可以各自运行自己的 browse daemon，而不需要任何配置，也不会发生端口冲突。旧方案（扫描 9400-9409）在多工作区场景下经常出问题。

### 版本自动重启

构建时会把 `git rev-parse HEAD` 写入 `browse/dist/.version`。每次 CLI 调用时，如果当前二进制版本与正在运行的服务端记录的 `binaryVersion` 不一致，CLI 就会杀掉旧服务端并启动新服务端。这样可以彻底消除“二进制已经过时但服务还没重启”这一类 bug——只要重新构建，下一条命令就会自动切换到新版本。

## 安全模型

### 仅限 localhost

HTTP 服务只绑定 `localhost`，而不是 `0.0.0.0`，因此无法从外网访问。

### Bearer token 认证

每个服务端会话都会生成一个随机 UUID token，并以 0o600 权限写入状态文件（仅所有者可读）。每个 HTTP 请求都必须包含 `Authorization: Bearer <token>`。如果 token 不匹配，服务端会返回 401。

这可以防止同一台机器上的其他进程访问你的 browse server。cookie picker UI（`/cookie-picker`）和健康检查（`/health`）不受此限制——它们仅在 localhost 暴露，也不执行命令。

### Cookie 安全

Cookies 是 gstack 处理的最敏感数据。设计如下：

1. **访问 Keychain 必须得到用户批准。** 每个浏览器第一次导入 cookie 时，都会触发 macOS Keychain 弹窗。用户必须点击 “Allow” 或 “Always Allow”。gstack 从不会静默读取凭据。

2. **解密在进程内完成。** Cookie 值会在内存中被解密（PBKDF2 + AES-128-CBC），然后加载到 Playwright context 中，绝不会以明文形式写入磁盘。cookie picker UI 也不会显示 cookie 值，只会显示域名和数量。

3. **数据库只读。** gstack 会先把 Chromium 的 cookie DB 复制到一个临时文件（避免与正在运行的浏览器产生 SQLite 锁冲突），然后以只读方式打开。它不会修改你的真实浏览器 cookie 数据库。

4. **密钥缓存是会话级的。** Keychain 密码以及派生出的 AES key 只会在服务端生命周期内缓存于内存中。一旦服务端关闭（空闲超时或显式停止），缓存就会消失。

5. **日志中不记录 cookie 值。** console、network 和 dialog 日志绝不会包含 cookie 值。`cookies` 命令输出的是 cookie 元数据（domain、name、expiry），值会被截断。

### 防止 shell injection

浏览器注册表（Comet、Chrome、Arc、Brave、Edge）是硬编码的。数据库路径由已知常量构造，而不是来自用户输入。Keychain 访问使用带显式参数数组的 `Bun.spawn()`，而不是通过 shell 字符串插值。

## Ref 系统

Refs（`@e1`、`@e2`、`@c1`）让代理能够在不写 CSS selector 或 XPath 的情况下定位页面元素。

### 工作原理

```
1. Agent 执行：$B snapshot -i
2. Server 调用 Playwright 的 page.accessibility.snapshot()
3. 解析器遍历 ARIA tree，依次分配 refs：@e1, @e2, @e3...
4. 对每个 ref，构建一个 Playwright Locator：getByRole(role, { name }).nth(index)
5. 在 BrowserManager 实例上保存 Map<string, RefEntry>（role + name + Locator）
6. 把标注后的树以纯文本返回

之后：
7. Agent 执行：$B click @e3
8. Server 解析 @e3 → Locator → locator.click()
```

### 为什么使用 Locator，而不是修改 DOM

最显然的做法，是往 DOM 里注入 `data-ref="@e1"` 属性。但这会在以下场景里出问题：

- **CSP（Content Security Policy）。** 许多生产环境站点会阻止脚本修改 DOM。
- **React/Vue/Svelte hydration。** 框架在 reconciliation 时可能会把注入属性移除。
- **Shadow DOM。** 从外部无法直接进入 shadow roots。

Playwright Locator 存在于 DOM 之外。它依赖 accessibility tree（由 Chromium 内部维护）和 `getByRole()` 查询。无需改 DOM，不受 CSP 限制，也不会被框架清理。

### Ref 生命周期

导航时 refs 会被清空（监听主 frame 的 `framenavigated` 事件）。这就是正确行为——导航之后，所有 locators 都已失效。代理必须重新执行 `snapshot` 获取新的 refs。这是有意设计的：过期 refs 应当大声失败，而不是误点错误元素。

### Ref 过期检测

SPA 可以在不触发 `framenavigated` 的情况下修改 DOM（例如 React router 跳转、tab 切换、modal 打开）。这会让 refs 虽然 URL 没变，却已经过期。为了解决这个问题，`resolveRef()` 在使用 ref 前会先执行异步 `count()` 检查：

```
resolveRef(@e3) → entry = refMap.get("e3")
                → count = await entry.locator.count()
                → if count === 0: throw "Ref @e3 is stale — element no longer exists. Run 'snapshot' to get fresh refs."
                → if count > 0: return { locator }
```

这样会快速失败（约 5ms 开销），而不是让 Playwright 对一个不存在的元素等到 30 秒 action timeout。`RefEntry` 还会保存 `role` 和 `name` 元数据，因此错误消息可以告诉代理这个元素原本是什么。

### Cursor-interactive refs（`@c`）

`-C` flag 会找出那些可点击但不在 ARIA tree 中的元素——例如带 `cursor: pointer` 的元素、带 `onclick` 属性的元素，或自定义 `tabindex`。这类元素会使用单独的命名空间，分配为 `@c1`、`@c2` 等 refs。这样可以捕获那些框架把它们渲染成 `<div>`，但实际上充当按钮的自定义组件。

## 日志架构

三组 ring buffers（每组 50,000 条，O(1) push）：

```
Browser events → CircularBuffer（内存中）→ 异步刷盘到 .gstack/*.log
```

Console messages、network requests 和 dialog events 各自拥有独立 buffer。刷盘每 1 秒发生一次——服务端只会把上次刷盘之后新增的内容 append 到文件中。这意味着：

- HTTP 请求处理不会被磁盘 I/O 阻塞
- 服务端崩溃后日志仍然保留（最多丢失 1 秒的数据）
- 内存是有界的（50K 条 × 3 个 buffers）
- 磁盘文件是 append-only 的，可被外部工具读取

`console`、`network` 和 `dialog` 命令读取的是内存 buffers，而不是磁盘。磁盘文件用于事后排障。

## SKILL.md 模板系统

### 问题是什么

`SKILL.md` 文件告诉 Claude 如何使用 browse 命令。如果文档里写了一个不存在的 flag，或者漏掉了新增命令，代理就会直接撞错。手工维护的文档必然会与代码漂移。

### 解决方案

```
SKILL.md.tmpl          （人工编写的 prose + placeholders）
       ↓
gen-skill-docs.ts      （读取源码元数据）
       ↓
SKILL.md               （已提交，自动生成的 sections）
```

模板里保留那些需要人工判断的 workflows、tips 和 examples。placeholders 则在构建时由源码填充：

| Placeholder | 来源 | 生成内容 |
|-------------|------|----------|
| `{{COMMAND_REFERENCE}}` | `commands.ts` | 按类别整理的命令表 |
| `{{SNAPSHOT_FLAGS}}` | `snapshot.ts` | 含示例的 flag 参考 |
| `{{PREAMBLE}}` | `gen-skill-docs.ts` | 启动块：更新检查、会话跟踪、贡献者模式、AskUserQuestion 格式 |
| `{{BROWSE_SETUP}}` | `gen-skill-docs.ts` | 二进制定位与安装说明 |
| `{{BASE_BRANCH_DETECT}}` | `gen-skill-docs.ts` | 面向 PR 的 skills（ship、review、qa、plan-ceo-review）的动态 base branch 检测 |
| `{{QA_METHODOLOGY}}` | `gen-skill-docs.ts` | `/qa` 和 `/qa-only` 共享的 QA 方法块 |
| `{{DESIGN_METHODOLOGY}}` | `gen-skill-docs.ts` | `/plan-design-review` 和 `/design-review` 共享的设计审计方法块 |
| `{{REVIEW_DASHBOARD}}` | `gen-skill-docs.ts` | `/ship` 起飞前的 Review Readiness Dashboard |
| `{{TEST_BOOTSTRAP}}` | `gen-skill-docs.ts` | `/qa`、`/ship`、`/design-review` 的测试框架检测、初始化与 CI/CD 设置 |

这在结构上是健全的：如果某条命令存在于代码里，它就会出现在文档中；如果它不存在，它就不可能出现在文档中。

### Preamble

每个 skill 都以一个 `{{PREAMBLE}}` 块开头，并在 skill 自身逻辑之前执行。它在一个 bash 命令中处理五件事：

1. **Update check** —— 调用 `gstack-update-check`，如果有新版本可升级就提示。
2. **Session tracking** —— touch `~/.gstack/sessions/$PPID`，并统计活跃会话数（过去 2 小时内被修改的文件）。当同时运行 3+ 个会话时，所有 skills 会进入 “ELI16 mode”——每个问题都会重新帮用户锚定上下文，因为此时用户往往在多个窗口间切换。
3. **Contributor mode** —— 从配置中读取 `gstack_contributor`。为 true 时，如果 gstack 本身行为异常，代理会把简短 field reports 写到 `~/.gstack/contributor-logs/`。
4. **AskUserQuestion format** —— 统一格式：context、question、`RECOMMENDATION: Choose X because ___`、字母选项。所有 skills 保持一致。
5. **Search Before Building** —— 在构建基础设施或不熟悉的模式之前，先搜索。知识分三层：tried-and-true（Layer 1）、new-and-popular（Layer 2）、first-principles（Layer 3）。如果通过 first-principles reasoning 发现传统共识是错的，代理要明确点出这个 “eureka moment” 并记录下来。完整的 builder philosophy 见 `ETHOS.md`。

### 为什么提交生成结果，而不是运行时再生成？

原因有三：

1. **Claude 会在 skill 加载时读取 `SKILL.md`。** 用户执行 `/browse` 时不存在额外的构建步骤。因此文件必须事先存在，而且必须是正确的。
2. **CI 可以校验新鲜度。** `gen:skill-docs --dry-run` + `git diff --exit-code` 可以在合并前发现过期文档。
3. **Git blame 有意义。** 你可以看到一条命令是在什么时候、哪个 commit 中加入的。

### 模板测试分层

| Tier | 内容 | 成本 | 速度 |
|------|------|------|------|
| 1 — Static validation | 解析 `SKILL.md` 中每条 `$B` 命令，并与注册表比对校验 | 免费 | <2s |
| 2 — E2E via `claude -p` | 启动真实 Claude 会话，运行每个 skill，检查是否报错 | ~$3.85 | ~20min |
| 3 — LLM-as-judge | 由 Sonnet 为文档的清晰度 / 完整度 / 可执行性打分 | ~$0.15 | ~30s |

Tier 1 会在每次 `bun test` 时运行。Tier 2 + 3 则通过 `EVALS=1` 控制。核心思路是：免费抓住 95% 的问题，只有涉及判断的部分才调用 LLM。

## 命令分发

命令按副作用分类：

- **READ**（text、html、links、console、cookies 等）：不修改状态。可安全重试。返回页面状态。
- **WRITE**（goto、click、fill、press 等）：修改页面状态。不是幂等操作。
- **META**（snapshot、screenshot、tabs、chain 等）：服务端级操作，不完全适合归入 read / write。

这不只是为了组织结构。服务端也用它进行分发：

```typescript
if (READ_COMMANDS.has(cmd))  → handleReadCommand(cmd, args, bm)
if (WRITE_COMMANDS.has(cmd)) → handleWriteCommand(cmd, args, bm)
if (META_COMMANDS.has(cmd))  → handleMetaCommand(cmd, args, bm, shutdown)
```

`help` 命令会返回这三组命令，因此代理可以自行发现可用命令。

## 错误哲学

错误信息是写给 AI 代理看的，不是写给人看的。每条错误都必须可执行：

- “Element not found” → “Element not found or not interactable. Run `snapshot -i` to see available elements.”
- “Selector matched multiple elements” → “Selector matched multiple elements. Use @refs from `snapshot` instead.”
- 超时 → “Navigation timed out after 30s. The page may be slow or the URL may be wrong.”

Playwright 原生错误会通过 `wrapError()` 改写，去掉内部堆栈，并添加下一步建议。代理读完错误后，不需要人类介入就应该知道该怎么做。

### 崩溃恢复

服务端不会尝试自愈。如果 Chromium 崩溃（`browser.on('disconnected')`），服务端会立即退出。CLI 会在下一条命令时检测到服务死亡并自动重启。相比尝试重新接回一个半死不活的浏览器进程，这种方案更简单也更可靠。

## E2E 测试基础设施

### Session runner（`test/helpers/session-runner.ts`）

E2E 测试会以完全独立的子进程方式启动 `claude -p`——而不是通过 Agent SDK，因为 Agent SDK 不能嵌套运行在 Claude Code 会话里。runner 的流程如下：

1. 把 prompt 写到临时文件（避免 shell 转义问题）
2. 启动 `sh -c 'cat prompt | claude -p --output-format stream-json --verbose'`
3. 从 stdout 流式读取 NDJSON，以获得实时进度
4. 与一个可配置 timeout 竞速
5. 将完整 NDJSON transcript 解析为结构化结果

`parseNDJSON()` 函数是纯函数——无 I/O、无副作用，因此可以独立测试。

### 可观测性数据流

```
  skill-e2e-*.test.ts
        │
        │ 生成 runId，并把 testName + runId 传给每次调用
        │
  ┌─────┼──────────────────────────────┐
  │     │                              │
  │  runSkillTest()              evalCollector
  │  (session-runner.ts)         (eval-store.ts)
  │     │                              │
  │  每次工具调用：              每次 addTest()：
  │  ┌──┼──────────┐              savePartial()
  │  │  │          │                   │
  │  ▼  ▼          ▼                   ▼
  │ [HB] [PL]    [NJ]          _partial-e2e.json
  │  │    │        │             （原子覆盖）
  │  │    │        │
  │  ▼    ▼        ▼
  │ e2e-  prog-  {name}
  │ live  ress   .ndjson
  │ .json .log
  │
  │  失败时：
  │  {name}-failure.json
  │
  │  所有文件都在 ~/.gstack-dev/
  │  Run 目录：e2e-runs/{runId}/
  │
  │         eval-watch.ts
  │              │
  │        ┌─────┴─────┐
  │     读取 HB      读取 partial
  │        └─────┬─────┘
  │              ▼
  │        渲染 dashboard
  │        （超过 10 分钟无更新则告警）
```

**所有权拆分：** session-runner 负责 heartbeat（当前测试状态），eval-store 负责 partial results（已完成测试状态）。watcher 同时读取两者。两个组件彼此互不感知——它们只通过文件系统共享数据。

**一切都不应因为可观测性失败而失败：** 所有可观测性 I/O 都包在 try/catch 里。写文件失败永远不应导致测试失败。真正的 truth source 是测试本身；可观测性只是 best-effort。

**机器可读诊断：** 每条测试结果都包含 `exit_reason`（success、timeout、error_max_turns、error_api、exit_code_N）、`timeout_at_turn` 和 `last_tool_call`。这样就可以做类似下面的 `jq` 查询：

```bash
jq '.tests[] | select(.exit_reason == "timeout") | .last_tool_call' ~/.gstack-dev/evals/_partial-e2e.json
```

### Eval 持久化（`test/helpers/eval-store.ts`）

`EvalCollector` 会累计测试结果，并以两种方式写盘：

1. **增量写入：** `savePartial()` 会在每个测试结束后写入 `_partial-e2e.json`（原子写：写 `.tmp`，再 `fs.renameSync`）。即使进程被杀也能保留结果。
2. **最终写入：** `finalize()` 会写出一个带时间戳的 eval 文件（例如 `e2e-20260314-143022.json`）。partial 文件不会被清理——它会与最终文件一起保留下来，作为可观测性数据。

`eval:compare` 用于比较两次 eval 运行。`eval:summary` 用于汇总 `~/.gstack-dev/evals/` 下所有运行的统计信息。

### 测试分层

| Tier | 内容 | 成本 | 速度 |
|------|------|------|------|
| 1 — Static validation | 解析 `$B` 命令、与注册表校验、可观测性单元测试 | 免费 | <5s |
| 2 — E2E via `claude -p` | 启动真实 Claude 会话，运行每个 skill，并扫描错误 | ~$3.85 | ~20min |
| 3 — LLM-as-judge | 由 Sonnet 为文档的清晰度 / 完整度 / 可执行性打分 | ~$0.15 | ~30s |

Tier 1 会在每次 `bun test` 时运行。Tier 2 + 3 则通过 `EVALS=1` 控制。核心思路是：免费抓住 95% 的问题，把 LLM 留给判断型问题和集成测试。

## 有意不做的部分

- **不做 WebSocket streaming。** HTTP request/response 更简单，可用 curl 调试，而且速度已经足够。Streaming 只会为了边际收益增加复杂度。
- **不做 MCP protocol。** MCP 每次请求都会引入 JSON schema 开销，还要求持久连接。纯 HTTP + 纯文本输出对 tokens 更轻，也更容易调试。
- **不做多用户支持。** 每个 workspace 一个 server，一个用户。token 认证只是 defense-in-depth，不是多租户设计。
- **不做 Windows/Linux cookie 解密。** 当前只支持 macOS Keychain。Linux（GNOME Keyring / kwallet）和 Windows（DPAPI）在架构上可行，但尚未实现。
- **不做 iframe 支持。** Playwright 可以处理 iframes，但 ref 系统还不能跨 frame 工作。这是目前最常被提到的缺失能力。
