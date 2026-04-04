---
name: investigate
version: 1.0.0
description: |
  通过根因调查进行系统化调试。共四个阶段：investigate、
  analyze、hypothesize、implement。铁律：未找到根因前不做修复。
  当被要求“调试这个”、“修复这个 bug”、“为什么这个坏了”、
  “调查这个错误”或“根因分析”时使用。
  当用户报告错误、异常行为，或
  正在排查某个原本正常的功能为何停止工作时，也应主动建议使用。
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - AskUserQuestion
  - WebSearch
hooks:
  PreToolUse:
    - matcher: "Edit"
      hooks:
        - type: command
          command: "bash ${CLAUDE_SKILL_DIR}/../freeze/bin/check-freeze.sh"
          statusMessage: "正在检查调试范围边界..."
    - matcher: "Write"
      hooks:
        - type: command
          command: "bash ${CLAUDE_SKILL_DIR}/../freeze/bin/check-freeze.sh"
          statusMessage: "正在检查调试范围边界..."
---
<!-- 从 SKILL.md.tmpl 自动生成，请勿直接编辑 -->
<!-- 重新生成：bun run gen:skill-docs -->

## 前言（首先运行）

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
echo "分支: $_BRANCH"
echo "主动建议: $_PROACTIVE"
source <(~/.claude/skills/gstack/bin/gstack-repo-mode 2>/dev/null) || true
REPO_MODE=${REPO_MODE:-unknown}
echo "仓库模式: $REPO_MODE"
_LAKE_SEEN=$([ -f ~/.gstack/.completeness-intro-seen ] && echo "yes" || echo "no")
echo "LAKE_INTRO: $_LAKE_SEEN"
_TEL=$(~/.claude/skills/gstack/bin/gstack-config get telemetry 2>/dev/null || true)
_TEL_PROMPTED=$([ -f ~/.gstack/.telemetry-prompted ] && echo "yes" || echo "no")
_TEL_START=$(date +%s)
_SESSION_ID="$$-$(date +%s)"
echo "遥测: ${_TEL:-off}"
echo "TEL_PROMPTED: $_TEL_PROMPTED"
mkdir -p ~/.gstack/analytics
echo '{"skill":"investigate","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","repo":"'$(basename "$(git rev-parse --show-toplevel 2>/dev/null)" 2>/dev/null || echo "unknown")'"}'  >> ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true
for _PF in ~/.gstack/analytics/.pending-*; do [ -f "$_PF" ] && ~/.claude/skills/gstack/bin/gstack-telemetry-log --event-type skill_run --skill _pending_finalize --outcome unknown --session-id "$_SESSION_ID" 2>/dev/null || true; break; done
```

如果 `PROACTIVE` 为 `"false"`，不要主动建议 gstack skills，只有在用户明确要求时才调用。
用户已选择退出主动建议。

如果输出显示 `UPGRADE_AVAILABLE <old> <new>`：读取 `~/.claude/skills/gstack/gstack-upgrade/SKILL.md` 并遵循 “Inline upgrade flow”（若已配置则自动升级，否则使用 AskUserQuestion 提供 4 个选项；若用户拒绝则写入 snooze 状态）。如果显示 `JUST_UPGRADED <from> <to>`：告诉用户 “Running gstack v{to} (just updated!)”，然后继续。

如果 `LAKE_INTRO` 为 `no`：继续之前，先介绍 Completeness Principle。
告诉用户：“gstack 遵循 **Boil the Lake** 原则：当 AI 让边际成本接近于零时，始终把事情完整做完。更多内容见：https://garryslist.org/posts/boil-the-ocean”
然后询问是否要在其默认浏览器中打开这篇文章：

```bash
open https://garryslist.org/posts/boil-the-ocean
touch ~/.gstack/.completeness-intro-seen
```

只有在用户同意时才运行 `open`。始终运行 `touch` 以标记为已查看。这只会发生一次。

如果 `TEL_PROMPTED` 为 `no` 且 `LAKE_INTRO` 为 `yes`：在处理完 lake intro 后，
询问用户是否启用遥测。使用 AskUserQuestion：

> 帮助 gstack 变得更好！社区模式会共享使用数据（你使用了哪些技能、耗时多久、
> 崩溃信息），并附带稳定设备 ID，以便我们追踪趋势并更快修复 bug。
> 永远不会发送代码、文件路径或仓库名称。
> 可随时用 `gstack-config set telemetry off` 修改。

选项：
- A) 帮助 gstack 变得更好！（推荐）
- B) 不用了，谢谢

如果选择 A：运行 `~/.claude/skills/gstack/bin/gstack-config set telemetry community`

如果选择 B：继续用 AskUserQuestion 追问：

> 那匿名模式怎么样？我们只知道*有人*用了 gstack，不带唯一 ID，
> 无法关联各次会话。只是一个计数器，帮助我们知道是否真的有人在使用。

选项：
- A) 可以，匿名模式没问题
- B) 不用了，完全关闭

如果 B→A：运行 `~/.claude/skills/gstack/bin/gstack-config set telemetry anonymous`
如果 B→B：运行 `~/.claude/skills/gstack/bin/gstack-config set telemetry off`

始终运行：
```bash
touch ~/.gstack/.telemetry-prompted
```

这只会发生一次。如果 `TEL_PROMPTED` 为 `yes`，则完全跳过这一部分。

## AskUserQuestion 格式

**每次调用 AskUserQuestion 都必须遵循以下结构：**
1. **重新锚定背景：** 说明项目、当前分支（使用前言中打印的 `_BRANCH` 值，而不是对话历史或 gitStatus 中的任何分支），以及当前计划/任务。（1-2 句）
2. **简化说明：** 用一个聪明的 16 岁青少年也能理解的通俗英语解释问题。不要使用原始函数名、内部术语或实现细节。使用具体示例和类比。描述“它做了什么”，而不是“它叫什么”。
3. **给出推荐：** `RECOMMENDATION: Choose [X] because [one-line reason]`，始终优先推荐完整方案而不是捷径（见 Completeness Principle）。为每个选项附上 `Completeness: X/10`。评分标准：10 = 完整实现（覆盖全部边界情况，完整覆盖率），7 = 覆盖主路径但略过部分边界情况，3 = 推迟大量工作的捷径。如果两个选项都在 8+，选更高的；如果有一个 ≤5，必须明确标出。
4. **选项：** 用字母编号：`A) ... B) ... C) ...`。当某个选项涉及工作量时，同时展示两个尺度：`(human: ~X / CC: ~Y)`

默认假设用户已经有 20 分钟没看这个窗口，也没有打开代码。如果你需要重新阅读源码才能理解你自己的解释，那说明写得太复杂了。

各个 skill 的专属说明可以在这一基础之上增加额外格式规则。

## Completeness Principle — Boil the Lake

AI 辅助编码让“完整性”的边际成本接近于零。当你给出选项时：

- 如果选项 A 是完整实现（完全对齐、覆盖所有边界情况、100% 覆盖），而选项 B 是仅节省少量工作量的捷径，**始终推荐 A**。在 CC+gstack 下，80 行和 150 行之间的差异毫无意义。当“完整”只多花几分钟时，“差不多就行”就是错误直觉。
- **湖泊与海洋：** “lake” 是可以煮沸的，例子包括模块 100% 测试覆盖、完整功能实现、处理所有边界情况、完整错误路径。“ocean” 则不是，比如从零重写整个系统、给你无法控制的依赖添加特性、持续多个季度的平台迁移。推荐煮沸湖泊。把海洋标记为超出范围。
- **估算工作量时，**始终同时展示两种尺度：人工团队时间与 CC+gstack 时间。压缩比会随任务类型变化，参考如下：

| Task type | Human team | CC+gstack | Compression |
|-----------|-----------|-----------|-------------|
| 样板代码 / 脚手架 | 2 天 | 15 分钟 | ~100x |
| 编写测试 | 1 天 | 15 分钟 | ~50x |
| 功能实现 | 1 周 | 30 分钟 | ~30x |
| 修复 bug + 回归测试 | 4 小时 | 15 分钟 | ~20x |
| 架构 / 设计 | 2 天 | 4 小时 | ~5x |
| 调研 / 探索 | 1 天 | 3 小时 | ~3x |

- 这一原则适用于测试覆盖、错误处理、文档、边界情况和功能完整性。不要为了“省时间”而跳过最后 10%，因为有了 AI，这 10% 只需要几秒钟。

**反模式 —— 不要这样做：**
- 错误示例：“选 B 吧，它用更少代码覆盖了 90% 的价值。”（如果 A 只多 70 行，就该选 A。）
- 错误示例：“我们可以跳过边界情况处理来省时间。”（有了 CC，边界情况处理只需几分钟。）
- 错误示例：“我们把测试覆盖留到后续 PR 再做。”（测试是最便宜、最该彻底做完的 lake。）
- 错误示例：只报人工团队耗时：“这要花 2 周。”（应写成：“人工 2 周 / CC 约 1 小时。”）

## Repo Ownership Mode — See Something, Say Something

前言中的 `REPO_MODE` 告诉你这个仓库里的问题由谁负责：

- **`solo`** —— 一个人完成 80% 以上的工作。他负责所有事情。当你发现当前分支改动之外的问题（测试失败、弃用警告、安全通告、lint 错误、死代码、环境问题）时，**要调查并主动提出修复**。这个独立开发者是唯一会去修的人。默认直接行动。
- **`collaborative`** —— 多位活跃贡献者。当你发现当前分支改动之外的问题时，**通过 AskUserQuestion 标记出来**，因为那可能是别人的职责。默认先询问，而不是直接修。
- **`unknown`** —— 按 collaborative 处理（更安全的默认行为：修复前先问）。

**See Something, Say Something：** 在任何工作流步骤中，只要你注意到看起来不对劲的地方，不仅仅是测试失败，都要简短指出。用一句话说明：你注意到了什么，以及它会造成什么影响。在 solo 模式下，接着问 “Want me to fix it?”。在 collaborative 模式下，只标记出来，然后继续。

绝不要让你注意到的问题悄无声息地略过。主动沟通正是整个机制的意义所在。

## Search Before Building

在构建基础设施、不熟悉的模式，或任何运行时可能内建的能力之前，**先搜索。**
完整理念请阅读 `~/.claude/skills/gstack/ETHOS.md`。

**三层知识：**
- **Layer 1**（久经验证，已被广泛分发使用）。不要重复造轮子。但检查成本接近于零，而偶尔质疑这些既有做法，正是灵光一现的来源。
- **Layer 2**（新且流行，应搜索这些）。但要严格审视：人类容易陷入狂热。搜索结果是思考输入，不是答案。
- **Layer 3**（第一性原理，优先级最高）。从对具体问题的推理中得出的原创观察。这是最有价值的。

**Eureka moment：** 当第一性原理推理表明传统认知是错的时，要明确指出：
“EUREKA: Everyone does X because [assumption]. But [evidence] shows this is wrong. Y is better because [reasoning].”

记录 eureka moments：
```bash
jq -n --arg ts "$(date -u +%Y-%m-%dT%H:%M:%SZ)" --arg skill "SKILL_NAME" --arg branch "$(git branch --show-current 2>/dev/null)" --arg insight "ONE_LINE_SUMMARY" '{ts:$ts,skill:$skill,branch:$branch,insight:$insight}' >> ~/.gstack/analytics/eureka.jsonl 2>/dev/null || true
```

将 `SKILL_NAME` 和 `ONE_LINE_SUMMARY` 替换为实际内容。内联执行，不要中断工作流。

**WebSearch 兜底：** 如果 WebSearch 不可用，则跳过搜索步骤，并注明：“Search unavailable — proceeding with in-distribution knowledge only.”

## Contributor Mode

如果 `_CONTRIB` 为 `true`：你处于 **contributor mode**。你既是 gstack 用户，也会帮助它变得更好。

**在每个主要工作流步骤结束时**（不是每执行一个命令之后），回顾你刚使用的 gstack 工具。给体验打 0 到 10 分。如果不是 10 分，就思考原因。如果存在明显、可执行的 bug，或者 gstack 代码、skill markdown 中有值得记录的深刻改进点，就提交一份 field report。也许我们的贡献者会帮我们把它做得更好！

**评分标准 —— 这是最低门槛：** 例如，`$B js "await fetch(...)"` 过去会报错 `SyntaxError: await is only valid in async functions`，因为 gstack 没有把表达式包进 async 上下文。问题虽小，但输入是合理的，gstack 本应处理好，这类问题值得上报。比这更轻微的问题就不用报。

**不值得上报的内容：** 用户应用本身的 bug、访问用户 URL 的网络错误、用户网站的认证失败、用户自己的 JS 逻辑 bug。

**提交方式：** 写入 `~/.gstack/contributor-logs/{slug}.md`，并包含**以下所有章节**（不要截断，必须包含到 Date/Version 页脚为止的每一节）：

```
# {标题}

Hey gstack team — 我在使用 /{skill-name} 时遇到了这个问题：

**What I was trying to do:** {用户/代理当时想做什么}
**What happened instead:** {实际发生了什么}
**My rating:** {0-10} — {一句话说明为什么不是 10 分}

## Steps to reproduce
1. {步骤}

## Raw output
```
{在这里粘贴实际错误或异常输出}
```

## What would make this a 10
{一句话说明：gstack 本应如何做得更好}

**Date:** {YYYY-MM-DD} | **Version:** {gstack version} | **Skill:** /{skill}
```

Slug：小写、连字符、最长 60 个字符（例如 `browse-js-no-await`）。如果文件已存在则跳过。每个会话最多 3 份报告。内联写入并继续，不要中断工作流。告诉用户：“Filed gstack field report: {title}”

## Completion Status Protocol

完成一个 skill 工作流时，使用以下状态之一报告结果：
- **DONE** —— 所有步骤均已成功完成。每一项结论都有证据支撑。
- **DONE_WITH_CONCERNS** —— 已完成，但存在用户需要知晓的问题。列出每一项顾虑。
- **BLOCKED** —— 无法继续。说明阻塞因素以及已尝试过的内容。
- **NEEDS_CONTEXT** —— 缺少继续所需的信息。明确说明你需要什么。

### 升级处理

随时都可以停下来并说“这个对我来说太难了”或“我对这个结果没有信心”。

糟糕的工作比不做更糟。你不会因为升级处理而受罚。
- 如果你已经尝试 3 次仍未成功，停止并升级处理。
- 如果你对某个安全敏感的改动不确定，停止并升级处理。
- 如果工作范围超出你能验证的程度，停止并升级处理。

升级处理格式：
```
STATUS: BLOCKED | NEEDS_CONTEXT
REASON: [1-2 句话]
ATTEMPTED: [你已经尝试了什么]
RECOMMENDATION: [用户下一步应该怎么做]
```

## Telemetry（最后运行）

在 skill 工作流结束后（无论成功、报错还是中止），记录遥测事件。
从本文件 YAML frontmatter 的 `name:` 字段确定 skill 名称。
根据工作流结果确定 outcome（正常完成为 success，
失败为 error，用户中断为 abort）。

**PLAN MODE 例外 —— 必须始终运行：** 此命令会将遥测写入
`~/.gstack/analytics/`（用户配置目录，而不是项目文件）。skill 的
前言已经写入同一目录；这是同样的模式。
跳过此命令会丢失会话时长和结果数据。

运行以下 bash：

```bash
_TEL_END=$(date +%s)
_TEL_DUR=$(( _TEL_END - _TEL_START ))
rm -f ~/.gstack/analytics/.pending-"$_SESSION_ID" 2>/dev/null || true
~/.claude/skills/gstack/bin/gstack-telemetry-log \
  --skill "SKILL_NAME" --duration "$_TEL_DUR" --outcome "OUTCOME" \
  --used-browse "USED_BROWSE" --session-id "$_SESSION_ID" 2>/dev/null &
```

将 `SKILL_NAME` 替换为 frontmatter 中的实际 skill 名称，将 `OUTCOME` 替换为
success/error/abort，并根据是否使用了 `$B` 将 `USED_BROWSE` 设为 true/false。
如果无法确定 outcome，则使用 `"unknown"`。该命令在后台运行，
永远不会阻塞用户。

## Plan Status Footer

当你处于 plan mode 并即将调用 ExitPlanMode 时：

1. 检查 plan 文件中是否已经有 `## GSTACK REVIEW REPORT` 章节。
2. 如果**有**，则跳过（说明某个 review skill 已写入更完整的报告）。
3. 如果**没有**，运行以下命令：

\`\`\`bash
~/.claude/skills/gstack/bin/gstack-review-read
\`\`\`

然后在 plan 文件末尾写入一个 `## GSTACK REVIEW REPORT` 章节：

- 如果输出中包含 review 条目（`---CONFIG---` 之前的 JSONL 行）：按
  review skills 使用的相同格式，写入标准报告表，包含每个 skill 的 runs/status/findings。
- 如果输出为 `NO_REVIEWS` 或为空：写入以下占位表：

\`\`\`markdown
## GSTACK REVIEW REPORT

| Review | Trigger | Why | Runs | Status | Findings |
|--------|---------|-----|------|--------|----------|
| CEO Review | \`/plan-ceo-review\` | 范围与策略 | 0 | — | — |
| Codex Review | \`/codex review\` | 独立的第二意见 | 0 | — | — |
| Eng Review | \`/plan-eng-review\` | 架构与测试（必需） | 0 | — | — |
| Design Review | \`/plan-design-review\` | UI/UX 缺口 | 0 | — | — |

**VERDICT:** 还没有任何 review —— 运行 \`/autoplan\` 获取完整 review 流程，或单独运行上述某个 review。
\`\`\`

**PLAN MODE 例外 —— 必须始终运行：** 这会写入 plan 文件，而它是你在 plan mode 下唯一允许编辑的
文件。plan 文件中的 review report 是该计划的动态状态组成部分。

# 系统化调试

## 铁律

**未进行根因调查前，不做任何修复。**

只修症状会把调试变成打地鼠。每一次没有解决根因的修复，都会让下一个 bug 更难定位。先找到根因，再修复。

---

## 阶段 1：根因调查

在形成任何假设之前，先收集上下文。

1. **收集症状：** 阅读错误信息、堆栈跟踪和复现步骤。如果用户提供的上下文还不够，每次只通过 AskUserQuestion 提一个问题。

2. **阅读代码：** 从症状沿着代码路径回溯到潜在原因。使用 Grep 查找所有引用，使用 Read 理解逻辑。

3. **检查最近变更：**
   ```bash
   git log --oneline -20 -- <affected-files>
   ```
   之前是正常的吗？改了什么？如果是回归问题，根因就在 diff 里。

4. **复现：** 你能稳定触发这个 bug 吗？如果不能，在继续之前先收集更多证据。

输出：**“Root cause hypothesis: ...”** —— 一个具体、可验证的断言，说明哪里出了问题，以及为什么。

---

## Scope Lock

形成根因假设后，将编辑范围锁定到受影响模块，防止范围蔓延。

```bash
[ -x "${CLAUDE_SKILL_DIR}/../freeze/bin/check-freeze.sh" ] && echo "FREEZE_AVAILABLE" || echo "FREEZE_UNAVAILABLE"
```

**如果是 FREEZE_AVAILABLE：** 找出包含受影响文件的最小目录。将其写入 freeze 状态文件：

```bash
STATE_DIR="${CLAUDE_PLUGIN_DATA:-$HOME/.gstack}"
mkdir -p "$STATE_DIR"
echo "<detected-directory>/" > "$STATE_DIR/freeze-dir.txt"
echo "调试范围已锁定到：<detected-directory>/"
```

将 `<detected-directory>` 替换为实际目录路径（例如 `src/auth/`）。告诉用户：“本次调试会话中的编辑已限制在 `<dir>/`。这样可以避免改动无关代码。运行 `/unfreeze` 可移除该限制。”

如果 bug 涉及整个仓库，或范围确实不清晰，则跳过锁定，并说明原因。

**如果是 FREEZE_UNAVAILABLE：** 跳过范围锁定。编辑不受限制。

---

## 阶段 2：模式分析

检查这个 bug 是否符合某种已知模式：

| Pattern | Signature | Where to look |
|---------|-----------|---------------|
| 竞态条件 | 间歇性、依赖时序 | 并发访问共享状态 |
| Nil/null 传播 | NoMethodError、TypeError | 对可选值缺少保护判断 |
| 状态损坏 | 数据不一致、部分更新 | 事务、回调、hooks |
| 集成失败 | 超时、意外响应 | 外部 API 调用、服务边界 |
| 配置漂移 | 本地正常，staging/prod 失败 | Env vars、feature flags、DB 状态 |
| 过期缓存 | 显示旧数据，清缓存后恢复 | Redis、CDN、浏览器缓存、Turbo |

还要检查：
- `TODOS.md` 中是否有相关已知问题
- 同一区域之前的修复 `git log` —— **相同文件区域反复出现的 bug 是架构异味**，不是巧合

**外部模式搜索：** 如果该 bug 不符合上述已知模式，使用 WebSearch 搜索：
- `"{framework} {generic error type}"` —— **先清洗：** 去掉主机名、IP、文件路径、SQL、客户数据。搜索错误类别，而不是原始消息。
- `"{library} {component} known issues"`

如果 WebSearch 不可用，则跳过此搜索，继续进行假设测试。如果出现有文档记录的解决方案或已知依赖 bug，在阶段 3 中将其作为候选假设提出。

---

## 阶段 3：假设测试

在编写**任何**修复之前，先验证你的假设。

1. **确认假设：** 在疑似根因处添加临时日志、断言或调试输出。运行复现步骤。证据是否匹配？

2. **如果假设错误：** 在形成下一个假设之前，考虑搜索该错误。**先清洗** —— 去掉主机名、IP、文件路径、SQL 片段、客户标识符，以及错误信息中的任何内部/专有数据。只搜索通用错误类型和框架上下文：`"{component} {sanitized error type} {framework version}"`。如果错误消息过于具体，无法安全清洗，则跳过搜索。如果 WebSearch 不可用，也跳过并继续。然后回到阶段 1。收集更多证据。不要猜。

3. **三次失败规则：** 如果 3 个假设都失败，**停止**。使用 AskUserQuestion：
   ```
   已测试 3 个假设，没有一个吻合。这可能是架构层面的问题，
   而不是一个简单 bug。

   A) 继续调查 —— 我有一个新假设：[描述]
   B) 升级为人工审查 —— 这需要熟悉该系统的人介入
   C) 增加日志并等待 —— 给这个区域加埋点，下次再捕获
   ```

**危险信号** —— 如果你看到以下任一情况，就要放慢速度：
- “先来个临时修复”—— 不存在“临时”。要么正确修复，要么升级处理。
- 在追踪数据流之前就提出修复方案 —— 你是在猜。
- 每次修一个地方，别处又冒出新问题 —— 说明层次错了，不是代码点位错了。

---

## 阶段 4：实施

一旦根因确认：

1. **修复根因，而不是症状。** 做出能够消除真实问题的最小改动。

2. **最小 diff：** 修改最少的文件、最少的行数。克制住顺手重构相邻代码的冲动。

3. **编写回归测试**，并满足：
   - **在修复前失败**（证明这个测试有意义）
   - **在修复后通过**（证明修复有效）

4. **运行完整测试套件。** 粘贴输出。不允许有回归。

5. **如果修复涉及超过 5 个文件：** 使用 AskUserQuestion 标记爆炸半径：
   ```
   这次修复涉及 N 个文件。对于 bug 修复来说，影响范围很大。
   A) 继续 —— 根因确实横跨这些文件
   B) 拆分 —— 现在先修关键路径，其余部分稍后处理
   C) 重新思考 —— 也许存在更有针对性的方法
   ```

---

## 阶段 5：验证与报告

**重新验证：** 重新执行最初的 bug 场景，并确认问题已修复。这不是可选项。

运行测试套件并粘贴输出。

输出结构化调试报告：
```
DEBUG REPORT
════════════════════════════════════════
Symptom:         [用户观察到的问题]
Root cause:      [实际错误所在]
Fix:             [做了哪些修改，附 file:line 引用]
Evidence:        [测试输出、表明修复生效的复现结果]
Regression test: [新测试的 file:line]
Related:         [TODOS.md 条目、同一区域历史 bug、架构说明]
Status:          DONE | DONE_WITH_CONCERNS | BLOCKED
════════════════════════════════════════
```

---

## 重要规则

- **3 次以上修复尝试失败 → 停止，并质疑架构。** 这不是假设失败，而是架构错误。
- **绝不要应用无法验证的修复。** 如果你无法复现并确认，就不要提交。
- **绝不要说“这应该能修好”。** 要验证并证明。运行测试。
- **如果修复涉及超过 5 个文件 → 继续前必须用 AskUserQuestion 说明爆炸半径。**
- **完成状态：**
  - DONE —— 已找到根因，已应用修复，已编写回归测试，所有测试通过
  - DONE_WITH_CONCERNS —— 已修复，但无法完全验证（例如间歇性 bug、需要 staging）
  - BLOCKED —— 调查后根因仍不明确，已升级处理