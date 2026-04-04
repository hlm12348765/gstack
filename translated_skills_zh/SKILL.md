---
name: gstack
version: 1.1.0
description: |
  用于 QA 测试和站点内部试用的快速无头浏览器。可导航页面、与元素交互、
  验证状态、对比前后差异、截取带标注的截图、测试响应式布局、
  表单、上传、对话框，并收集 bug 证据。适用于被要求打开或
  测试网站、验证部署、内部试用某个用户流程，或提交带截图的 bug 时。
  还可按阶段建议相邻的 gstack 技能：头脑风暴 /office-hours；策略
  /plan-ceo-review；架构 /plan-eng-review；设计 /plan-design-review 或
  /design-consultation；自动审查 /autoplan；调试 /investigate；QA /qa；代码审查
  /review；视觉审计 /design-review；发布 /ship；文档 /document-release；复盘
  /retro；第二意见 /codex；生产安全 /careful 或 /guard；范围内编辑 /freeze 或
  /unfreeze；gstack 升级 /gstack-upgrade。如果用户选择不接收建议，请停止
  并运行 gstack-config set proactive false；如果他们重新选择接收，请运行 gstack-config set
  proactive true。
allowed-tools:
  - Bash
  - Read
  - AskUserQuestion

---
<!-- AUTO-GENERATED from SKILL.md.tmpl — do not edit directly -->
<!-- 重新生成：bun run gen:skill-docs -->

## 前言（先运行）

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
echo '{"skill":"gstack","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","repo":"'$(basename "$(git rev-parse --show-toplevel 2>/dev/null)" 2>/dev/null || echo "unknown")'"}'  >> ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true
for _PF in ~/.gstack/analytics/.pending-*; do [ -f "$_PF" ] && ~/.claude/skills/gstack/bin/gstack-telemetry-log --event-type skill_run --skill _pending_finalize --outcome unknown --session-id "$_SESSION_ID" 2>/dev/null || true; break; done
```

如果 `PROACTIVE` 是 `"false"`，不要主动建议 gstack 技能，只有在用户明确要求时才调用
它们。用户已选择退出主动建议。

如果输出显示 `UPGRADE_AVAILABLE <old> <new>`：读取 `~/.claude/skills/gstack/gstack-upgrade/SKILL.md` 并遵循“Inline upgrade flow”（如果已配置则自动升级，否则使用带 4 个选项的 AskUserQuestion，若用户拒绝则写入 snooze 状态）。如果是 `JUST_UPGRADED <from> <to>`：告诉用户“Running gstack v{to} (just updated!)”，然后继续。

如果 `LAKE_INTRO` 是 `no`：继续前，先介绍 Completeness Principle。
告诉用户：“gstack 遵循 **Boil the Lake** 原则：当 AI 让边际成本几乎为零时，就始终把事情完整做好。
了解更多：https://garryslist.org/posts/boil-the-ocean”
然后询问是否要在他们的默认浏览器中打开这篇文章：

```bash
open https://garryslist.org/posts/boil-the-ocean
touch ~/.gstack/.completeness-intro-seen
```

只有在用户表示同意时才运行 `open`。始终运行 `touch` 以标记为已查看。此流程只会发生一次。

如果 `TEL_PROMPTED` 是 `no` 且 `LAKE_INTRO` 是 `yes`：在处理完 lake 介绍后，
询问用户是否启用 telemetry。使用 AskUserQuestion：

> 帮助 gstack 做得更好！Community mode 会共享使用数据（你使用了哪些技能、耗时多久、
> 崩溃信息），并附带稳定的设备 ID，以便我们跟踪趋势并更快修复 bug。
> 绝不会发送代码、文件路径或仓库名称。
> 可随时通过 `gstack-config set telemetry off` 更改。

选项：
- A) 帮助 gstack 做得更好！（推荐）
- B) 不，谢谢

如果选 A：运行 `~/.claude/skills/gstack/bin/gstack-config set telemetry community`

如果选 B：继续使用 AskUserQuestion 追问：

> 那匿名模式怎么样？我们只会知道*有人*使用了 gstack，不含唯一 ID，
> 也无法关联不同会话。只是一个计数器，帮助我们知道是否真的有人在使用。

选项：
- A) 可以，匿名模式没问题
- B) 不，谢谢，完全关闭

如果 B→A：运行 `~/.claude/skills/gstack/bin/gstack-config set telemetry anonymous`
如果 B→B：运行 `~/.claude/skills/gstack/bin/gstack-config set telemetry off`

始终运行：
```bash
touch ~/.gstack/.telemetry-prompted
```

此流程只会发生一次。如果 `TEL_PROMPTED` 是 `yes`，则完全跳过。

## AskUserQuestion 格式

**每次调用 AskUserQuestion 时都必须遵循此结构：**
1. **重新锚定：** 说明项目、当前分支（使用前言中打印的 `_BRANCH` 值，而不是会话历史或 gitStatus 中的任何分支），以及当前计划/任务。（1-2 句）
2. **简化说明：** 用一个聪明的 16 岁少年也能理解的朴素英语解释问题。不要出现原始函数名、内部术语或实现细节。使用具体示例和类比。说明它“做什么”，不要说它“叫什么”。
3. **给出建议：** `RECOMMENDATION: Choose [X] because [one-line reason]`，始终优先推荐完整方案而不是捷径（见 Completeness Principle）。为每个选项都附上 `Completeness: X/10`。评分校准：10 = 完整实现（所有边界情况、完整覆盖），7 = 覆盖主路径但跳过部分边界情况，3 = 推迟大量工作的捷径。如果两个选项都在 8+，选更高的；如果某个选项 ≤5，要明确提示。
4. **选项：** 用字母列出选项：`A) ... B) ... C) ...`；当某个选项涉及工作量时，同时显示两个量级：`(human: ~X / CC: ~Y)`

假设用户已经 20 分钟没看这个窗口了，而且当前并没有打开代码。如果你的解释连你自己都需要去读源码才能理解，那就说明过于复杂。

每个技能的专属说明可以在此基线之上再增加额外的格式规则。

## Completeness Principle — Boil the Lake

AI 辅助编码让“做完整”的边际成本几乎为零。当你提供选项时：

- 如果选项 A 是完整实现（完全对齐、覆盖所有边界情况、100% 覆盖），而选项 B 是只节省少量工作量的捷径，**始终推荐 A**。借助 CC+gstack，80 行和 150 行之间的差别毫无意义。当“完整”只多花几分钟时，“差不多就行”是错误直觉。
- **Lake vs. ocean：** “lake” 是可以煮开的，例如模块的 100% 测试覆盖、功能的完整实现、处理所有边界情况、覆盖完整错误路径。“ocean” 则不是，例如从零重写整个系统、给无法控制的依赖添加功能、跨度数个季度的平台迁移。建议煮开 lakes。对 oceans 要标记为超出范围。
- **估算工作量时**，始终同时给出两个量级：人工团队时间和 CC+gstack 时间。不同任务类型的压缩比不同，可参考下表：

| Task type | Human team | CC+gstack | Compression |
|-----------|-----------|-----------|-------------|
| Boilerplate / scaffolding | 2 days | 15 min | ~100x |
| Test writing | 1 day | 15 min | ~50x |
| Feature implementation | 1 week | 30 min | ~30x |
| Bug fix + regression test | 4 hours | 15 min | ~20x |
| Architecture / design | 2 days | 4 hours | ~5x |
| Research / exploration | 1 day | 3 hours | ~3x |

- 这一原则适用于测试覆盖、错误处理、文档、边界情况和功能完整性。不要为了“省时间”而跳过最后 10%，在 AI 的帮助下，这 10% 只需要几秒钟。

**反模式，不要这样做：**
- 错误：”Choose B — it covers 90% of the value with less code.“（如果 A 只多 70 行，就应该选 A。）
- 错误：”We can skip edge case handling to save time.“（借助 CC，边界情况处理只需要几分钟。）
- 错误：”Let's defer test coverage to a follow-up PR.“（测试是最便宜、最该一口气做完的 lake。）
- 错误：只引用人工团队工作量：”This would take 2 weeks.“（应写成：”2 weeks human / ~1 hour CC.“）

## Repo Ownership Mode — See Something, Say Something

前言中的 `REPO_MODE` 会告诉你这个仓库里的问题由谁负责：

- **`solo`**：一个人完成了 80% 以上的工作。他们负责一切。当你注意到当前分支改动范围之外的问题时（测试失败、弃用警告、安全告警、lint 错误、死代码、环境问题），**要主动调查并提出修复**。这个单人开发者是唯一会修这些问题的人。默认直接行动。
- **`collaborative`**：有多个活跃贡献者。当你注意到当前分支改动范围之外的问题时，**通过 AskUserQuestion 提示出来**，因为那可能是别人的职责。默认先询问，不直接修。
- **`unknown`**：按 collaborative 处理（更安全的默认值，先问再修）。

**See Something, Say Something：** 无论在哪个工作流步骤中，只要你注意到看起来不对的东西，不仅限于测试失败，都要简要指出。用一句话说明：你注意到了什么，以及它有什么影响。在 solo 模式下，后接 “Want me to fix it?”。在 collaborative 模式下，只需指出后继续。

不要让已注意到的问题悄悄溜过去。主动沟通正是这个机制的核心。

## Search Before Building

在构建基础设施、不熟悉的模式，或任何运行时可能已内建的能力之前，**先搜索。**
完整理念请阅读 `~/.claude/skills/gstack/ETHOS.md`。

**三层知识：**
- **Layer 1**（经过验证、已在发行版中使用）。不要重复造轮子。但检查的成本几乎为零，而偶尔，质疑这些既有做法正是产生卓越想法的起点。
- **Layer 2**（新且流行，应该搜索这些）。但要审慎看待：人类会受到潮流狂热影响。搜索结果只是你思考的输入，不是答案。
- **Layer 3**（第一性原理，最应珍视）。通过针对具体问题的推理得出的原创观察。这是最有价值的一层。

**Eureka moment：** 当基于第一性原理的推理表明传统观点是错的时，要明确指出：
“EUREKA: Everyone does X because [assumption]. But [evidence] shows this is wrong. Y is better because [reasoning].”

记录 eureka moments：
```bash
jq -n --arg ts "$(date -u +%Y-%m-%dT%H:%M:%SZ)" --arg skill "SKILL_NAME" --arg branch "$(git branch --show-current 2>/dev/null)" --arg insight "ONE_LINE_SUMMARY" '{ts:$ts,skill:$skill,branch:$branch,insight:$insight}' >> ~/.gstack/analytics/eureka.jsonl 2>/dev/null || true
```

将 SKILL_NAME 和 ONE_LINE_SUMMARY 替换为实际值。内联运行，不要中断工作流。

**WebSearch 兜底：** 如果 WebSearch 不可用，则跳过搜索步骤，并说明：“Search unavailable — proceeding with in-distribution knowledge only.”

## Contributor Mode

如果 `_CONTRIB` 是 `true`：你处于 **contributor mode**。你既是 gstack 用户，也在帮助它变得更好。

**在每个主要工作流步骤结束时**（不是每一条命令后），回顾你使用过的 gstack 工具，并为体验打 0 到 10 分。如果不是 10 分，思考原因。如果存在明显、可执行的 bug，或者是某个值得记录的、对 gstack 代码或技能 markdown 改进很有价值的见解，就提交一份 field report。也许我们的贡献者会帮助我们变得更好！

**评分校准，这才是门槛：** 例如，`$B js "await fetch(...)"` 过去会因 `SyntaxError: await is only valid in async functions` 而失败，因为 gstack 没有把表达式包裹在 async 上下文中。这虽是个小问题，但输入本身合理，gstack 本应正确处理，这类问题就值得提报。比这影响更小的，忽略即可。

**不值得提报：** 用户应用本身的 bug、访问用户 URL 时的网络错误、用户站点上的认证失败、用户自己的 JS 逻辑错误。

**提交方式：** 写入 `~/.gstack/contributor-logs/{slug}.md`，并包含**以下所有章节**（不要截断，必须包含直到 Date/Version 页脚的每一节）：

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
{一句话说明：gstack 本应如何做得更好}

**Date:** {YYYY-MM-DD} | **Version:** {gstack version} | **Skill:** /{skill}
```

Slug：小写、连字符、最长 60 个字符（例如 `browse-js-no-await`）。如果文件已存在则跳过。每个会话最多 3 份报告。内联提交并继续，不要中断工作流。告诉用户：“Filed gstack field report: {title}”

## Completion Status Protocol

完成技能工作流时，使用以下状态之一报告：

- **DONE**：所有步骤均已成功完成。每项结论都提供了证据。
- **DONE_WITH_CONCERNS**：已完成，但有用户应知晓的问题。逐条列出每个 concern。
- **BLOCKED**：无法继续。说明阻塞原因以及已尝试的内容。
- **NEEDS_CONTEXT**：缺少继续所需的信息。明确说明你需要什么。

### Escalation

在任何时候，停下来并说“this is too hard for me”或“I'm not confident in this result.” 都是可以接受的。

糟糕的工作比没有工作更糟。升级处理不会受到惩罚。
- 如果你已尝试同一任务 3 次仍未成功，停止并升级处理。
- 如果你对安全敏感的变更不确定，停止并升级处理。
- 如果工作范围超出你能验证的程度，停止并升级处理。

升级处理格式：
```
STATUS: BLOCKED | NEEDS_CONTEXT
REASON: [1-2 句]
ATTEMPTED: [你已尝试的内容]
RECOMMENDATION: [用户下一步应做什么]
```

## Telemetry（最后运行）

在技能工作流完成后（成功、报错或中止），记录 telemetry 事件。
从本文件 YAML frontmatter 中的 `name:` 字段确定技能名。
根据工作流结果确定 outcome（正常完成则为 success，
失败则为 error，用户中断则为 abort）。

**PLAN MODE EXCEPTION — ALWAYS RUN：** 该命令会将 telemetry 写入
`~/.gstack/analytics/`（用户配置目录，而不是项目文件）。技能前言已经写入了同一目录，
这是同一种模式。跳过该命令会丢失会话时长和结果数据。

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
success/error/abort，并将 `USED_BROWSE` 替换为 true/false，依据是否使用了 `$B`。
如果无法确定 outcome，则使用 "unknown"。此命令在后台运行，
绝不会阻塞用户。

## Plan Status Footer

当你处于 plan mode 并且即将调用 ExitPlanMode 时：

1. 检查计划文件中是否已有 `## GSTACK REVIEW REPORT` 小节。
2. 如果**有**，跳过（说明某个 review 技能已经写入了更丰富的报告）。
3. 如果**没有**，运行以下命令：

\`\`\`bash
~/.claude/skills/gstack/bin/gstack-review-read
\`\`\`

然后在计划文件末尾写入 `## GSTACK REVIEW REPORT` 小节：

- 如果输出中包含 review 条目（`---CONFIG---` 之前的 JSONL 行）：按 review 技能使用的相同格式，写入标准报告表，包含每个技能的 runs/status/findings。
- 如果输出是 `NO_REVIEWS` 或为空：写入以下占位表格：

\`\`\`markdown
## GSTACK REVIEW REPORT

| Review | Trigger | Why | Runs | Status | Findings |
|--------|---------|-----|------|--------|----------|
| CEO Review | \`/plan-ceo-review\` | Scope & strategy | 0 | — | — |
| Codex Review | \`/codex review\` | Independent 2nd opinion | 0 | — | — |
| Eng Review | \`/plan-eng-review\` | Architecture & tests (required) | 0 | — | — |
| Design Review | \`/plan-design-review\` | UI/UX gaps | 0 | — | — |

**VERDICT:** NO REVIEWS YET — run \`/autoplan\` for full review pipeline, or individual reviews above.
\`\`\`

**PLAN MODE EXCEPTION — ALWAYS RUN：** 这会写入计划文件，而计划文件正是你在 plan mode 中唯一允许编辑的文件。计划文件中的 review report 是计划持续状态的一部分。

如果 `PROACTIVE` 是 `false`：本次会话中不要主动建议其他 gstack 技能。
只运行用户明确调用的技能。此偏好会通过
`gstack-config` 跨会话持久保存。

# gstack browse：QA 测试与内部试用

持久化无头 Chromium。首次调用会自动启动（约 3 秒），之后每条命令约 100-200 毫秒。
空闲 30 分钟后自动关闭。状态会在调用间保留（cookies、标签页、会话）。

## SETUP（在任何 browse 命令之前运行此检查）

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

如果是 `NEEDS_SETUP`：
1. 告诉用户：“gstack browse 需要一次性构建（约 10 秒）。可以继续吗？” 然后停止并等待。
2. 运行：`cd <SKILL_DIR> && ./setup`
3. 如果未安装 `bun`：`curl -fsSL https://bun.sh/install | bash`

## IMPORTANT

- 通过 Bash 使用已编译的二进制：`$B <command>`
- **绝不要**使用 `mcp__claude-in-chrome__*` 工具。它们又慢又不可靠。
- 浏览器会在调用间持久保留，cookies、登录会话和标签页都会延续。
- 对话框（alert/confirm/prompt）默认会自动接受，不会导致浏览器锁死。
- **显示截图：** 在 `$B screenshot`、`$B snapshot -a -o` 或 `$B responsive` 之后，始终使用 Read 工具读取输出的 PNG 文件，让用户能够看到它们。否则截图对用户不可见。

## QA 工作流

### 测试一个用户流程（登录、注册、结账等）

```bash
# 1. 前往页面
$B goto https://app.example.com/login

# 2. 查看哪些元素可交互
$B snapshot -i

# 3. 使用 refs 填写表单
$B fill @e3 "test@example.com"
$B fill @e4 "password123"
$B click @e5

# 4. 验证是否成功
$B snapshot -D              # diff 显示点击后发生了哪些变化
$B is visible ".dashboard"  # 断言 dashboard 已出现
$B screenshot /tmp/after-login.png
```

### 验证部署 / 检查生产环境

```bash
$B goto https://yourapp.com
$B text                          # 读取页面，是否成功加载？
$B console                       # 是否有 JS 错误？
$B network                       # 是否有失败请求？
$B js "document.title"           # 标题是否正确？
$B is visible ".hero-section"    # 关键元素是否存在？
$B screenshot /tmp/prod-check.png
```

### 从头到尾内部试用某个功能

```bash
# 导航到该功能
$B goto https://app.example.com/new-feature

# 截取带标注的截图，显示每个可交互元素及其标签
$B snapshot -i -a -o /tmp/feature-annotated.png

# 找出所有可点击项（包括带 cursor:pointer 的 div）
$B snapshot -C

# 走完整个流程
$B snapshot -i          # 基线
$B click @e3            # 交互
$B snapshot -D          # 有什么变化？（统一 diff）

# 检查元素状态
$B is visible ".success-toast"
$B is enabled "#next-step-btn"
$B is checked "#agree-checkbox"

# 交互后检查 console 中是否有错误
$B console
```

### 测试响应式布局

```bash
# 快速方式：分别截取 mobile/tablet/desktop 的 3 张截图
$B goto https://yourapp.com
$B responsive /tmp/layout

# 手动方式：指定 viewport
$B viewport 375x812     # iPhone
$B screenshot /tmp/mobile.png
$B viewport 1440x900    # Desktop
$B screenshot /tmp/desktop.png

# 元素截图（裁剪到指定元素）
$B screenshot "#hero-banner" /tmp/hero.png
$B snapshot -i
$B screenshot @e3 /tmp/button.png

# 区域裁剪
$B screenshot --clip 0,0,800,600 /tmp/above-fold.png

# 仅 viewport（不滚动）
$B screenshot --viewport /tmp/viewport.png
```

### 测试文件上传

```bash
$B goto https://app.example.com/upload
$B snapshot -i
$B upload @e3 /path/to/test-file.pdf
$B is visible ".upload-success"
$B screenshot /tmp/upload-result.png
```

### 测试带校验的表单

```bash
$B goto https://app.example.com/form
$B snapshot -i

# 空提交，检查是否出现校验错误
$B click @e10                        # 提交按钮
$B snapshot -D                       # diff 显示错误消息已出现
$B is visible ".error-message"

# 填写后再次提交
$B fill @e3 "valid input"
$B click @e10
$B snapshot -D                       # diff 显示错误消失，进入成功状态
```

### 测试对话框（删除确认、提示输入）

```bash
# 在触发前先设置对话框处理
$B dialog-accept              # 自动接受下一个 alert/confirm
$B click "#delete-button"     # 触发确认对话框
$B dialog                     # 查看出现了什么对话框
$B snapshot -D                # 验证该项已被删除

# 对于需要输入内容的 prompt
$B dialog-accept "my answer"  # 接受并填写文本
$B click "#rename-button"     # 触发 prompt
```

### 测试已认证页面（导入真实浏览器 cookies）

```bash
# 从你的真实浏览器导入 cookies（会打开交互式选择器）
$B cookie-import-browser

# 或直接导入指定域名
$B cookie-import-browser comet --domain .github.com

# 现在测试已认证页面
$B goto https://github.com/settings/profile
$B snapshot -i
$B screenshot /tmp/github-profile.png
```

### 比较两个页面 / 环境

```bash
$B diff https://staging.app.com https://prod.app.com
```

### 多步骤链式执行（适合长流程，更高效）

```bash
echo '[
  ["goto","https://app.example.com"],
  ["snapshot","-i"],
  ["fill","@e3","test@test.com"],
  ["fill","@e4","password"],
  ["click","@e5"],
  ["snapshot","-D"],
  ["screenshot","/tmp/result.png"]
]' | $B chain
```

## 快速断言模式

```bash
# 元素存在且可见
$B is visible ".modal"

# 按钮已启用/已禁用
$B is enabled "#submit-btn"
$B is disabled "#submit-btn"

# 复选框状态
$B is checked "#agree"

# 输入框可编辑
$B is editable "#name-field"

# 元素已获得焦点
$B is focused "#search-input"

# 页面包含文本
$B js "document.body.textContent.includes('Success')"

# 元素数量
$B js "document.querySelectorAll('.list-item').length"

# 指定属性值
$B attrs "#logo"    # 以 JSON 返回所有属性

# CSS 属性
$B css ".button" "background-color"
```

## Snapshot 系统

snapshot 是你理解和操作页面的主要工具。

```
-i        --interactive           仅显示可交互元素（按钮、链接、输入框），并带 @e refs
-c        --compact               紧凑模式（不显示空的结构节点）
-d <N>    --depth                 限制树深度（0 = 仅根节点，默认：不限制）
-s <sel>  --selector              限定到 CSS selector
-D        --diff                  与上一次 snapshot 进行统一 diff（首次调用会存储基线）
-a        --annotate              带红色覆盖框和 ref 标签的标注截图
-o <path> --output                标注截图输出路径（默认：<temp>/browse-annotated.png）
-C        --cursor-interactive    光标可交互元素（@c refs，例如带 pointer 或 onclick 的 div）
```

所有 flags 都可以自由组合。`-o` 仅在同时使用 `-a` 时生效。
示例：`$B snapshot -i -a -C -o /tmp/annotated.png`

**Ref 编号：** @e refs 按树顺序依次分配（@e1、@e2、...）。
来自 `-C` 的 @c refs 单独编号（@c1、@c2、...）。

执行 snapshot 后，可在任意命令中将 @refs 用作 selector：
```bash
$B click @e3       $B fill @e4 "value"     $B hover @e1
$B html @e2        $B css @e5 "color"      $B attrs @e6
$B click @c1       # 光标可交互 ref（来自 -C）
```

**输出格式：** 带 @ref ID 的缩进无障碍树，每行一个元素。
```
  @e1 [heading] "Welcome" [level=1]
  @e2 [textbox] "Email"
  @e3 [button] "Submit"
```

导航后 refs 会失效，执行 `goto` 后请重新运行 `snapshot`。

## 命令参考

### Navigation
| Command | Description |
|---------|-------------|
| `back` | 历史回退 |
| `forward` | 历史前进 |
| `goto <url>` | 导航到 URL |
| `reload` | 重新加载页面 |
| `url` | 打印当前 URL |

### Reading
| Command | Description |
|---------|-------------|
| `accessibility` | 完整 ARIA 树 |
| `forms` | 以 JSON 输出表单字段 |
| `html [selector]` | selector 的 innerHTML（未找到则抛错）；若未提供 selector，则返回完整页面 HTML |
| `links` | 以 “text → href” 形式列出所有链接 |
| `text` | 清理后的页面文本 |

### Interaction
| Command | Description |
|---------|-------------|
| `click <sel>` | 点击元素 |
| `cookie <name>=<value>` | 在当前页面域名下设置 cookie |
| `cookie-import <json>` | 从 JSON 文件导入 cookies |
| `cookie-import-browser [browser] [--domain d]` | 从 Comet、Chrome、Arc、Brave 或 Edge 导入 cookies（会打开选择器，或使用 --domain 直接导入） |
| `dialog-accept [text]` | 自动接受下一个 alert/confirm/prompt。可选 text 作为 prompt 的响应内容发送 |
| `dialog-dismiss` | 自动关闭下一个对话框 |
| `fill <sel> <val>` | 填充输入框 |
| `header <name>:<value>` | 设置自定义请求头（使用冒号分隔，敏感值会自动打码） |
| `hover <sel>` | 悬停元素 |
| `press <key>` | 按键，支持 Enter、Tab、Escape、ArrowUp/Down/Left/Right、Backspace、Delete、Home、End、PageUp、PageDown，以及 Shift+Enter 这类组合键 |
| `scroll [sel]` | 将元素滚动到可见区域；如果未提供 selector，则滚动到页面底部 |
| `select <sel> <val>` | 按 value、label 或可见文本选择下拉选项 |
| `type <text>` | 在当前聚焦元素中输入 |
| `upload <sel> <file> [file2...]` | 上传文件 |
| `useragent <string>` | 设置 user agent |
| `viewport <WxH>` | 设置 viewport 大小 |
| `wait <sel|--networkidle|--load>` | 等待元素出现、网络空闲或页面加载完成（超时：15 秒） |

### Inspection
| Command | Description |
|---------|-------------|
| `attrs <sel|@ref>` | 以 JSON 输出元素属性 |
| `console [--clear|--errors]` | console 消息（`--errors` 仅筛选 error/warning） |
| `cookies` | 以 JSON 输出所有 cookies |
| `css <sel> <prop>` | 计算后的 CSS 值 |
| `dialog [--clear]` | 对话框消息 |
| `eval <file>` | 运行文件中的 JavaScript，并以字符串返回结果（路径必须位于 /tmp 或 cwd 下） |
| `is <prop> <sel>` | 状态检查（visible/hidden/enabled/disabled/checked/editable/focused） |
| `js <expr>` | 运行 JavaScript 表达式，并以字符串返回结果 |
| `network [--clear]` | 网络请求 |
| `perf` | 页面加载时序 |
| `storage [set k v]` | 以 JSON 读取全部 localStorage + sessionStorage，或使用 set <key> <value> 写入 localStorage |

### Visual
| Command | Description |
|---------|-------------|
| `diff <url1> <url2>` | 页面之间的文本 diff |
| `pdf [path]` | 另存为 PDF |
| `responsive [prefix]` | 分别在 mobile（375x812）、tablet（768x1024）、desktop（1280x720）下截图。保存为 {prefix}-mobile.png 等 |
| `screenshot [--viewport] [--clip x,y,w,h] [selector|@ref] [path]` | 保存截图（支持通过 CSS/@ref 裁剪元素，支持 `--clip` 区域裁剪，支持 `--viewport`） |

### Snapshot
| Command | Description |
|---------|-------------|
| `snapshot [flags]` | 带 @e refs 的无障碍树，用于选择元素。Flags：-i 仅交互元素，-c 紧凑模式，-d N 深度限制，-s sel 作用域，-D 与上次比较差异，-a 标注截图，-o 输出路径，-C 光标可交互 @c refs |

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
| `tabs` | 列出所有打开的标签页 |

### Server
| Command | Description |
|---------|-------------|
| `handoff [message]` | 在当前页面打开可见的 Chrome，交由用户接管 |
| `restart` | 重启服务器 |
| `resume` | 在用户接管后重新 snapshot，并将控制权交还给 AI |
| `status` | 健康检查 |
| `stop` | 关闭服务器 |

## 提示

1. **导航一次，查询多次。** `goto` 会加载页面；之后 `text`、`js`、`screenshot` 都会立即作用于已加载页面。
2. **先用 `snapshot -i`。** 先看所有可交互元素，再按 ref 进行 click/fill。无需猜 CSS selector。
3. **用 `snapshot -D` 验证。** 基线 → 操作 → diff。精确查看发生了什么变化。
4. **用 `is` 做断言。** `is visible .modal` 比解析页面文本更快也更可靠。
5. **用 `snapshot -a` 保留证据。** 带标注的截图非常适合用于 bug 报告。
6. **棘手界面用 `snapshot -C`。** 它能找出无障碍树遗漏的可点击 div。
7. **操作后检查 `console`。** 捕获那些视觉上不明显的 JS 错误。
8. **长流程用 `chain`。** 单条命令完成，无需为每一步承担 CLI 开销。