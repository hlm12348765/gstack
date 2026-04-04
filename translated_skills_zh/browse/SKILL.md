---
name: browse
version: 1.1.0
description: |
  用于 QA 测试和站点自测的高速无头浏览器。可导航到任意 URL，与
  元素交互，验证页面状态，对操作前后进行 diff，截取带标注的截图，检查
  响应式布局，测试表单和上传，处理对话框，并断言元素状态。
  每条命令约 ~100ms。当你需要测试某个功能、验证一次部署、自测一条
  用户流程，或带证据提交一个缺陷时使用它。当被要求“open in browser”、“test the
  site”、“take a screenshot”或“dogfood this”时使用。
allowed-tools:
  - Bash
  - Read
  - AskUserQuestion

---
<!-- 从 SKILL.md.tmpl 自动生成 —— 请勿直接编辑 -->
<!-- 重新生成：bun run gen:skill-docs -->

## 前导步骤（先运行）

```bash
_UPD=$(~/.claude/skills/gstack/bin/gstack-update-check 2>/dev/null || .claude/skills/gstack/bin/gstack-update-check 2>/dev/null || true)
[ -n "$_UPD" ] && echo "$_UPD" || true
mkdir -p ~/.gstack/sessions
touch ~/.gstack/sessions/"$PPID"
_SESSIONS=$(find ~/.gstack/sessions -mmin -120 -type f 2>/dev/null | wc -l | tr -d ' ')
find ~/.gstack/sessions -mmin +120 -type f -delete 2>/dev/null || true
_CONTRIB=$(~/.claude/skills/gstack/bin/gstack-config get gstack_contributor 2>/dev/null || true)
_PROACTIVE=$(~/.claude/skills/gstack/bin/gstack-config get proactive 2>/dev/null || echo "true")
_BRANCH=$(git branch --show-current 2>/dev/null || echo "unknown")
echo "BRANCH: $_BRANCH"
echo "PROACTIVE: $_PROACTIVE"
source <(~/.claude/skills/gstack/bin/gstack-repo-mode 2>/dev/null) || true
REPO_MODE=${REPO_MODE:-unknown}
echo "REPO_MODE: $REPO_MODE"
_LAKE_SEEN=$([ -f ~/.gstack/.completeness-intro-seen ] && echo "yes" || echo "no")
echo "LAKE_INTRO: $_LAKE_SEEN"
_TEL=$(~/.claude/skills/gstack/bin/gstack-config get telemetry 2>/dev/null || true)
_TEL_PROMPTED=$([ -f ~/.gstack/.telemetry-prompted ] && echo "yes" || echo "no")
_TEL_START=$(date +%s)
_SESSION_ID="$$-$(date +%s)"
echo "TELEMETRY: ${_TEL:-off}"
echo "TEL_PROMPTED: $_TEL_PROMPTED"
mkdir -p ~/.gstack/analytics
echo '{"skill":"browse","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","repo":"'$(basename "$(git rev-parse --show-toplevel 2>/dev/null)" 2>/dev/null || echo "unknown")'"}'  >> ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true
for _PF in ~/.gstack/analytics/.pending-*; do [ -f "$_PF" ] && ~/.claude/skills/gstack/bin/gstack-telemetry-log --event-type skill_run --skill _pending_finalize --outcome unknown --session-id "$_SESSION_ID" 2>/dev/null || true; break; done
```

如果 `PROACTIVE` 是 `"false"`，不要主动建议 gstack 技能，只能在用户明确要求时调用。
这是用户选择退出主动建议的设置。

如果输出显示 `UPGRADE_AVAILABLE <old> <new>`：读取 `~/.claude/skills/gstack/gstack-upgrade/SKILL.md` 并遵循 “Inline upgrade flow”（如果已配置则自动升级，否则使用 AskUserQuestion 提供 4 个选项，如果用户拒绝则写入 snooze 状态）。如果显示 `JUST_UPGRADED <from> <to>`：告诉用户 “Running gstack v{to} (just updated!)” 然后继续。

如果 `LAKE_INTRO` 是 `no`：继续之前，先介绍 Completeness Principle。
告诉用户：“gstack 遵循 **Boil the Lake** 原则：当 AI 让边际成本接近于零时，始终把事情完整做完。阅读更多：https://garryslist.org/posts/boil-the-ocean”
然后询问是否要在其默认浏览器中打开这篇文章：

```bash
open https://garryslist.org/posts/boil-the-ocean
touch ~/.gstack/.completeness-intro-seen
```

只有在用户同意时才运行 `open`。始终运行 `touch` 以标记为已查看。这个流程只会发生一次。

如果 `TEL_PROMPTED` 是 `no` 且 `LAKE_INTRO` 是 `yes`：在处理完 lake 介绍后，
使用 AskUserQuestion 询问用户是否启用 telemetry：

> 帮助 gstack 变得更好！Community mode 会共享使用数据（你使用了哪些技能、耗时多久、
> 崩溃信息），并附带一个稳定的设备 ID，这样我们就能跟踪趋势并更快修复缺陷。
> 绝不会发送代码、文件路径或仓库名称。
> 你可以随时通过 `gstack-config set telemetry off` 修改。

选项：
- A) Help gstack get better!（推荐）
- B) 不，谢谢

如果选择 A：运行 `~/.claude/skills/gstack/bin/gstack-config set telemetry community`

如果选择 B：继续用 AskUserQuestion 追问：

> 那 anonymous mode 呢？我们只会知道有 *某个人* 使用了 gstack，不会有唯一 ID，
> 也无法关联各次会话。只有一个计数器，用来帮助我们确认是否真的有人在使用。

选项：
- A) 可以，匿名就行
- B) 不，谢谢，完全关闭

如果 B→A：运行 `~/.claude/skills/gstack/bin/gstack-config set telemetry anonymous`
如果 B→B：运行 `~/.claude/skills/gstack/bin/gstack-config set telemetry off`

始终运行：
```bash
touch ~/.gstack/.telemetry-prompted
```

这个流程只会发生一次。如果 `TEL_PROMPTED` 是 `yes`，则完全跳过。

## AskUserQuestion 格式

**每次调用 AskUserQuestion 都必须遵循以下结构：**
1. **重新锚定：** 说明项目、当前分支（使用前导步骤打印出的 `_BRANCH` 值，而不是对话历史或 gitStatus 中的任何分支）以及当前计划/任务。（1-2 句话）
2. **简化说明：** 用一个聪明的 16 岁孩子也能听懂的普通英语解释问题。不要出现原始函数名、内部术语或实现细节。使用具体示例和类比。描述它“做什么”，而不是“它叫什么”。
3. **给出建议：** `RECOMMENDATION: Choose [X] because [one-line reason]`，始终优先选择完整方案而不是捷径（见 Completeness Principle）。为每个选项附上 `Completeness: X/10`。校准标准：10 = 完整实现（所有边界情况、完整覆盖），7 = 覆盖主路径但跳过部分边界情况，3 = 推迟大量工作的捷径。如果两个选项都在 8 以上，选更高的；如果有一个 ≤5，要明确标出。
4. **选项：** 使用字母编号选项：`A) ... B) ... C) ...`。当选项涉及工作量时，同时显示两种尺度：`(human: ~X / CC: ~Y)`

默认用户已经有 20 分钟没看这个窗口，而且没有打开代码。如果你需要重新阅读源码才能理解自己的解释，那说明解释太复杂了。

各技能说明可以在这个基础规则之上增加额外的格式要求。

## Completeness Principle — Boil the Lake

AI 辅助编程让“完整性”的边际成本接近于零。当你提供选项时：

- 如果选项 A 是完整实现（完整一致性、所有边界情况、100% 覆盖），而选项 B 是节省少量工作量的捷径，**始终推荐 A**。对于 CC+gstack 来说，80 行和 150 行之间的差别没有意义。当“完整”只多花几分钟时，“够用就行”是错误直觉。
- **lake 与 ocean：** “lake” 是可以煮沸的，例如一个模块 100% 的测试覆盖、一个功能的完整实现、处理所有边界情况、完整错误路径。“ocean” 则不是，例如从零重写整个系统、给你无法控制的依赖添加功能、跨度多个季度的平台迁移。建议煮沸 lakes。将 oceans 标记为超出范围。
- **估算工作量时，**始终同时给出两种尺度：人工团队时间和 CC+gstack 时间。压缩比会随任务类型变化，参考如下：

| 任务类型 | 人工团队 | CC+gstack | 压缩比 |
|-----------|-----------|-----------|-------------|
| 样板代码 / 脚手架 | 2 天 | 15 分钟 | ~100x |
| 编写测试 | 1 天 | 15 分钟 | ~50x |
| 功能实现 | 1 周 | 30 分钟 | ~30x |
| 缺陷修复 + 回归测试 | 4 小时 | 15 分钟 | ~20x |
| 架构 / 设计 | 2 天 | 4 小时 | ~5x |
| 研究 / 探索 | 1 天 | 3 小时 | ~3x |

- 这个原则适用于测试覆盖、错误处理、文档、边界情况和功能完整性。不要为了“节省时间”而跳过最后 10%：有了 AI，这 10% 的成本只是几秒钟。

**反模式 —— 不要这样做：**
- 错误示例：“选择 B，它用更少的代码覆盖了 90% 的价值。”（如果 A 只多 70 行，就选 A。）
- 错误示例：“我们可以跳过边界情况处理来节省时间。”（有了 CC，处理边界情况只需要几分钟。）
- 错误示例：“我们把测试覆盖推迟到后续 PR 再做。”（测试是最便宜、最应该一次做完的 lake。）
- 错误示例：只引用人工团队工作量：“这要花 2 周。”（应该说：“人工 2 周 / CC 约 1 小时。”）

## Repo Ownership Mode — 发现问题就说出来

前导步骤中的 `REPO_MODE` 表示这个仓库里的问题由谁负责：

- **`solo`** —— 一个人完成 80% 以上的工作。他拥有所有内容。当你注意到当前分支变更之外的问题时（测试失败、弃用警告、安全公告、lint 错误、死代码、环境问题），**要主动调查并提出修复**。这个单人开发者是唯一会修它的人。默认直接行动。
- **`collaborative`** —— 有多个活跃贡献者。当你注意到当前分支变更之外的问题时，**通过 AskUserQuestion 标记出来**，因为那可能是别人的职责。默认先询问，不要直接修。
- **`unknown`** —— 按 collaborative 处理（更安全的默认值：先问再修）。

**发现问题就说出来：** 无论在哪个工作流步骤中，只要你发现看起来不对的地方，不只是测试失败，都要简短指出。用一句话说明：你发现了什么，以及它的影响。在 solo 模式下，接着问 “Want me to fix it?”。在 collaborative 模式下，只需指出然后继续。

绝不要对已发现的问题视而不见。这条规则的核心就是主动沟通。

## 先搜索，再构建

在构建基础设施、不熟悉的模式，或任何运行时可能内置支持的东西之前，**先搜索。** 完整理念请阅读 `~/.claude/skills/gstack/ETHOS.md`。

**三层知识：**
- **Layer 1**（久经验证，已在分发中）。不要重复造轮子。但检查的成本接近于零，而偶尔对这些“久经验证的做法”提出质疑，正是产生卓见的地方。
- **Layer 2**（新且流行，要搜索这些）。但要审慎：人类容易陷入狂热。搜索结果只是你思考的输入，不是答案。
- **Layer 3**（第一性原理，高于一切）。基于对具体问题的推理得出的原创观察。这是最有价值的一层。

**Eureka 时刻：** 当基于第一性原理的推理揭示出传统认知是错的时，要明确说出来：
“EUREKA: Everyone does X because [assumption]. But [evidence] shows this is wrong. Y is better because [reasoning].”

记录 eureka 时刻：
```bash
jq -n --arg ts "$(date -u +%Y-%m-%dT%H:%M:%SZ)" --arg skill "SKILL_NAME" --arg branch "$(git branch --show-current 2>/dev/null)" --arg insight "ONE_LINE_SUMMARY" '{ts:$ts,skill:$skill,branch:$branch,insight:$insight}' >> ~/.gstack/analytics/eureka.jsonl 2>/dev/null || true
```
将 `SKILL_NAME` 和 `ONE_LINE_SUMMARY` 替换为实际内容。内联运行，不要中断工作流。

**WebSearch 回退：** 如果 WebSearch 不可用，则跳过搜索步骤，并注明：“Search unavailable — proceeding with in-distribution knowledge only.”

## Contributor Mode

如果 `_CONTRIB` 是 `true`：你处于 **contributor mode**。你既是 gstack 用户，也会帮助它变得更好。

**在每个主要工作流步骤结束时**（不是每执行一条命令之后），回顾你刚使用的 gstack 工具。给体验打 0 到 10 分。如果不是 10 分，思考原因。如果存在一个明显且可执行的缺陷，或者有一个有洞察力、值得记录的点，说明 gstack 代码或技能 markdown 本可以做得更好，就提交一份 field report。也许我们的贡献者会帮助我们继续改进！

**评分标准 —— 这就是门槛：** 例如，`$B js "await fetch(...)"` 过去会因 `SyntaxError: await is only valid in async functions` 而失败，因为 gstack 没有把表达式包裹在 async 上下文中。问题虽小，但输入是合理的，gstack 本应处理好，这类问题就值得提交。

**不值得提交的内容：** 用户应用自身的缺陷、访问用户 URL 的网络错误、用户站点的认证失败、用户自己的 JS 逻辑错误。

**提交方式：** 将 `~/.gstack/contributor-logs/{slug}.md` 写成如下**完整结构**（不要截断，必须包含从标题到 Date/Version 页脚的所有部分）：

```
# {Title}

Hey gstack team — ran into this while using /{skill-name}:

**What I was trying to do:** {what the user/agent was attempting}
**What happened instead:** {what actually happened}
**My rating:** {0-10} — {one sentence on why it wasn't a 10}

## Steps to reproduce
1. {step}

## Raw output
```
{在这里粘贴实际错误或异常输出}
```

## What would make this a 10
{一句话：gstack 本应做出怎样不同的处理}

**Date:** {YYYY-MM-DD} | **Version:** {gstack version} | **Skill:** /{skill}
```

Slug：小写、使用连字符、最长 60 个字符（例如 `browse-js-no-await`）。如果文件已存在则跳过。每个会话最多 3 份报告。内联提交并继续，不要中断工作流。告诉用户：“Filed gstack field report: {title}”

## Completion Status Protocol

完成一个技能工作流时，使用以下状态之一报告结果：
- **DONE** —— 所有步骤都已成功完成。每一项结论都提供了证据。
- **DONE_WITH_CONCERNS** —— 已完成，但存在用户应当知晓的问题。逐项列出每个 concern。
- **BLOCKED** —— 无法继续。说明阻塞原因，以及你已经尝试过什么。
- **NEEDS_CONTEXT** —— 缺少继续所需的信息。明确说明你需要什么。

### 升级处理

你始终可以停下来并说“这对我来说太难了”或“我对这个结果没有信心”。

糟糕的工作比不做更糟。你不会因为升级处理而受到惩罚。
- 如果你已经尝试同一个任务 3 次仍未成功，停止并升级处理。
- 如果你对安全敏感的变更没有把握，停止并升级处理。
- 如果工作范围超出了你能够验证的程度，停止并升级处理。

升级处理格式：
```
STATUS: BLOCKED | NEEDS_CONTEXT
REASON: [1-2 句话]
ATTEMPTED: [你已尝试的内容]
RECOMMENDATION: [用户下一步应做什么]
```

## Telemetry（最后运行）

在技能工作流结束后（成功、错误或中止），记录 telemetry 事件。
从本文件 YAML frontmatter 中的 `name:` 字段确定技能名称。
根据工作流结果确定 outcome（正常完成则为 success，
失败则为 error，用户中断则为 abort）。

**PLAN MODE 例外 —— 必须运行：** 此命令会将 telemetry 写入
`~/.gstack/analytics/`（用户配置目录，而不是项目文件）。技能前导步骤
已经写入了同一个目录；这是同一种模式。
跳过这条命令会丢失会话时长和结果数据。

运行以下 bash：

```bash
_TEL_END=$(date +%s)
_TEL_DUR=$(( _TEL_END - _TEL_START ))
rm -f ~/.gstack/analytics/.pending-"$_SESSION_ID" 2>/dev/null || true
~/.claude/skills/gstack/bin/gstack-telemetry-log \
  --skill "SKILL_NAME" --duration "$_TEL_DUR" --outcome "OUTCOME" \
  --used-browse "USED_BROWSE" --session-id "$_SESSION_ID" 2>/dev/null &
```

将 `SKILL_NAME` 替换为 frontmatter 中的实际技能名，将 `OUTCOME` 替换为
success/error/abort，并根据是否使用了 `$B` 将 `USED_BROWSE` 设为 true/false。
如果无法确定 outcome，就使用 `"unknown"`。这条命令在后台运行，
不会阻塞用户。

## Plan Status Footer

当你处于 plan mode 并且即将调用 ExitPlanMode 时：

1. 检查计划文件是否已经有 `## GSTACK REVIEW REPORT` 段落。
2. 如果**有**，则跳过（说明已有 review 技能写入了更完整的报告）。
3. 如果**没有**，则运行以下命令：

```bash
~/.claude/skills/gstack/bin/gstack-review-read
```

然后在计划文件末尾写入一个 `## GSTACK REVIEW REPORT` 段落：

- 如果输出中包含 review 条目（`---CONFIG---` 之前的 JSONL 行）：按标准报告表格格式写入，包含每个技能的 runs/status/findings，格式与 review 技能使用的格式一致。
- 如果输出是 `NO_REVIEWS` 或为空：写入以下占位表格：

```markdown
## GSTACK REVIEW REPORT

| Review | Trigger | Why | Runs | Status | Findings |
|--------|---------|-----|------|--------|----------|
| CEO Review | `/plan-ceo-review` | 范围与策略 | 0 | — | — |
| Codex Review | `/codex review` | 独立的第二意见 | 0 | — | — |
| Eng Review | `/plan-eng-review` | 架构与测试（必需） | 0 | — | — |
| Design Review | `/plan-design-review` | UI/UX 缺口 | 0 | — | — |

**VERDICT:** 尚无 REVIEW —— 运行 `/autoplan` 以启用完整 review 流水线，或运行上面的单独 review。
```

**PLAN MODE 例外 —— 必须运行：** 这会写入计划文件，而计划文件是 plan mode 下唯一允许编辑的文件。计划文件中的 review report 是计划实时状态的一部分。

# browse：QA 测试与自测

持久化无头 Chromium。首次调用会自动启动（约 3 秒），之后每条命令约 100ms。
状态会在调用之间保持（cookies、标签页、登录会话）。

## SETUP（在任何 browse 命令之前先运行此检查）

```bash
_ROOT=$(git rev-parse --show-toplevel 2>/dev/null)
B=""
[ -n "$_ROOT" ] && [ -x "$_ROOT/.claude/skills/gstack/browse/dist/browse" ] && B="$_ROOT/.claude/skills/gstack/browse/dist/browse"
[ -z "$B" ] && B=~/.claude/skills/gstack/browse/dist/browse
if [ -x "$B" ]; then
  echo "READY: $B"
else
  echo "NEEDS_SETUP"
fi
```

如果显示 `NEEDS_SETUP`：
1. 告诉用户：“gstack browse 需要一次性构建（约 10 秒）。可以继续吗？” 然后停止并等待。
2. 运行：`cd <SKILL_DIR> && ./setup`
3. 如果未安装 `bun`：运行 `curl -fsSL https://bun.sh/install | bash`

## 核心 QA 模式

### 1. 验证页面是否正确加载
```bash
$B goto https://yourapp.com
$B text                          # 内容是否加载？
$B console                       # 是否有 JS 错误？
$B network                       # 是否有失败请求？
$B is visible ".main-content"    # 关键元素是否存在？
```

### 2. 测试用户流程
```bash
$B goto https://app.com/login
$B snapshot -i                   # 查看所有可交互元素
$B fill @e3 "user@test.com"
$B fill @e4 "password"
$B click @e5                     # 提交
$B snapshot -D                   # diff：提交后发生了什么变化？
$B is visible ".dashboard"       # 成功状态是否存在？
```

### 3. 验证某个操作是否生效
```bash
$B snapshot                      # 基线
$B click @e3                     # 执行某个操作
$B snapshot -D                   # 统一 diff 精确显示发生了哪些变化
```

### 4. 为缺陷报告提供可视化证据
```bash
$B snapshot -i -a -o /tmp/annotated.png   # 带标签的截图
$B screenshot /tmp/bug.png                # 普通截图
$B console                                # 错误日志
```

### 5. 找出所有可点击元素（包括非 ARIA 元素）
```bash
$B snapshot -C                   # 查找带 cursor:pointer、onclick、tabindex 的 div
$B click @c1                     # 与它们交互
```

### 6. 断言元素状态
```bash
$B is visible ".modal"
$B is enabled "#submit-btn"
$B is disabled "#submit-btn"
$B is checked "#agree-checkbox"
$B is editable "#name-field"
$B is focused "#search-input"
$B js "document.body.textContent.includes('Success')"
```

### 7. 测试响应式布局
```bash
$B responsive /tmp/layout        # mobile + tablet + desktop 截图
$B viewport 375x812              # 或设置指定 viewport
$B screenshot /tmp/mobile.png
```

### 8. 测试文件上传
```bash
$B upload "#file-input" /path/to/file.pdf
$B is visible ".upload-success"
```

### 9. 测试对话框
```bash
$B dialog-accept "yes"           # 预先设置处理器
$B click "#delete-button"        # 触发对话框
$B dialog                        # 查看出现了什么
$B snapshot -D                   # 验证删除是否发生
```

### 10. 比较不同环境
```bash
$B diff https://staging.app.com https://prod.app.com
```

### 11. 向用户展示截图
在执行 `$B screenshot`、`$B snapshot -a -o` 或 `$B responsive` 之后，始终对输出的 PNG 使用 Read 工具，这样用户才能看到它们。否则，截图对用户是不可见的。

## 用户交接

当你遇到无头模式下无法处理的情况时（CAPTCHA、复杂认证、多因素
登录），将控制权交给用户：

```bash
# 1. 在当前页面打开一个可见的 Chrome
$B handoff "Stuck on CAPTCHA at login page"

# 2. 告诉用户发生了什么（通过 AskUserQuestion）
#    “我已经在登录页面打开了 Chrome。请完成 CAPTCHA，
#     然后在完成后告诉我。”

# 3. 当用户说“done”后，重新 snapshot 并继续
$B resume
```

**何时使用 handoff：**
- CAPTCHA 或机器人检测
- 多因素认证（短信、认证器应用）
- 需要用户交互的 OAuth 流程
- AI 连续尝试 3 次后仍无法处理的复杂交互

浏览器会在交接过程中保留全部状态（cookies、localStorage、标签页）。
执行 `resume` 后，你会拿到用户停留位置的最新 snapshot。

## Snapshot 标志

snapshot 是你理解和操作页面的主要工具。

```
-i        --interactive           仅显示可交互元素（按钮、链接、输入框），带 @e 引用
-c        --compact               紧凑模式（不显示空结构节点）
-d <N>    --depth                 限制树深度（0 = 仅根节点，默认：不限制）
-s <sel>  --selector              限定到 CSS selector 范围
-D        --diff                  与上一次 snapshot 做统一 diff（首次调用会存储基线）
-a        --annotate              生成带红色覆盖框和引用标签的标注截图
-o <path> --output                标注截图输出路径（默认：<temp>/browse-annotated.png）
-C        --cursor-interactive    光标可交互元素（@c 引用，例如带 pointer、onclick 的 div）
```

所有标志都可以自由组合。`-o` 只有在同时使用 `-a` 时才生效。
示例：`$B snapshot -i -a -C -o /tmp/annotated.png`

**引用编号：** @e 引用按树顺序依次分配（@e1、@e2、...）。
`-C` 生成的 @c 引用单独编号（@c1、@c2、...）。

执行 snapshot 后，可以在任意命令中把 @refs 当作 selector 使用：
```bash
$B click @e3       $B fill @e4 "value"     $B hover @e1
$B html @e2        $B css @e5 "color"      $B attrs @e6
$B click @c1       # 光标可交互引用（来自 -C）
```

**输出格式：** 带缩进的可访问性树，包含 @ref ID，每行一个元素。
```
  @e1 [heading] "Welcome" [level=1]
  @e2 [textbox] "Email"
  @e3 [button] "Submit"
```

导航后引用会失效，因此在执行 `goto` 之后要重新运行 `snapshot`。

## 完整命令列表

### Navigation
| Command | Description |
|---------|-------------|
| `back` | 历史记录后退 |
| `forward` | 历史记录前进 |
| `goto <url>` | 导航到 URL |
| `reload` | 重新加载页面 |
| `url` | 打印当前 URL |

### Reading
| Command | Description |
|---------|-------------|
| `accessibility` | 完整 ARIA 树 |
| `forms` | 以 JSON 形式输出表单字段 |
| `html [selector]` | selector 的 innerHTML（未找到时抛错）；若未给 selector，则返回整页 HTML |
| `links` | 以 “text → href” 形式列出所有链接 |
| `text` | 清理后的页面文本 |

### Interaction
| Command | Description |
|---------|-------------|
| `click <sel>` | 点击元素 |
| `cookie <name>=<value>` | 在当前页面域上设置 cookie |
| `cookie-import <json>` | 从 JSON 文件导入 cookies |
| `cookie-import-browser [browser] [--domain d]` | 从 Comet、Chrome、Arc、Brave 或 Edge 导入 cookies（会打开选择器，或使用 --domain 直接导入） |
| `dialog-accept [text]` | 自动接受下一个 alert/confirm/prompt。可选 text 会作为 prompt 的响应发送 |
| `dialog-dismiss` | 自动关闭下一个对话框 |
| `fill <sel> <val>` | 填充输入框 |
| `header <name>:<value>` | 设置自定义请求头（冒号分隔，敏感值会自动打码） |
| `hover <sel>` | 悬停元素 |
| `press <key>` | 按键操作，如 Enter、Tab、Escape、ArrowUp/Down/Left/Right、Backspace、Delete、Home、End、PageUp、PageDown，或 Shift+Enter 这类组合键 |
| `scroll [sel]` | 将元素滚动到可视区域；若未给 selector，则滚动到页面底部 |
| `select <sel> <val>` | 按 value、label 或可见文本选择下拉选项 |
| `type <text>` | 在聚焦元素中输入 |
| `upload <sel> <file> [file2...]` | 上传文件 |
| `useragent <string>` | 设置 user agent |
| `viewport <WxH>` | 设置 viewport 尺寸 |
| `wait <sel|--networkidle|--load>` | 等待元素、网络空闲或页面加载完成（超时：15 秒） |

### Inspection
| Command | Description |
|---------|-------------|
| `attrs <sel|@ref>` | 以 JSON 形式输出元素属性 |
| `console [--clear|--errors]` | 控制台消息（`--errors` 仅筛选 error/warning） |
| `cookies` | 以 JSON 形式输出所有 cookies |
| `css <sel> <prop>` | 计算后的 CSS 值 |
| `dialog [--clear]` | 对话框消息 |
| `eval <file>` | 运行文件中的 JavaScript，并将结果作为字符串返回（路径必须位于 /tmp 或 cwd 下） |
| `is <prop> <sel>` | 状态检查（visible/hidden/enabled/disabled/checked/editable/focused） |
| `js <expr>` | 运行 JavaScript 表达式，并将结果作为字符串返回 |
| `network [--clear]` | 网络请求 |
| `perf` | 页面加载时序 |
| `storage [set k v]` | 以 JSON 读取所有 localStorage + sessionStorage，或使用 set <key> <value> 写入 localStorage |

### Visual
| Command | Description |
|---------|-------------|
| `diff <url1> <url2>` | 页面之间的文本 diff |
| `pdf [path]` | 保存为 PDF |
| `responsive [prefix]` | 分别在 mobile（375x812）、tablet（768x1024）、desktop（1280x720）下截图。保存为 `{prefix}-mobile.png` 等 |
| `screenshot [--viewport] [--clip x,y,w,h] [selector|@ref] [path]` | 保存截图（支持通过 CSS/@ref 裁剪元素，支持 `--clip` 区域，支持 `--viewport`） |

### Snapshot
| Command | Description |
|---------|-------------|
| `snapshot [flags]` | 输出带 @e 引用的可访问性树，用于元素选择。标志：`-i` 仅可交互，`-c` 紧凑，`-d N` 深度限制，`-s sel` 范围限定，`-D` 与上次做 diff，`-a` 标注截图，`-o path` 输出路径，`-C` 光标可交互 @c 引用 |

### Meta
| Command | Description |
|---------|-------------|
| `chain` | 从 JSON stdin 运行命令。格式：`[["cmd","arg1",...],...]` |

### Tabs
| Command | Description |
|---------|-------------|
| `closetab [id]` | 关闭标签页 |
| `newtab [url]` | 打开新标签页 |
| `tab <id>` | 切换到标签页 |
| `tabs` | 列出打开的标签页 |

### Server
| Command | Description |
|---------|-------------|
| `handoff [message]` | 在当前页面打开可见的 Chrome，供用户接管 |
| `restart` | 重启服务器 |
| `resume` | 用户接管后重新 snapshot，并将控制权交还给 AI |
| `status` | 健康检查 |
| `stop` | 关闭服务器 |