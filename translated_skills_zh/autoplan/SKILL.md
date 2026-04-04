---
name: autoplan
version: 1.0.0
description: |
  自动审查流水线 —— 从磁盘读取完整的 CEO、design 和 eng review 技能，
  并使用 6 条决策原则按顺序自动决策运行它们。在最终审批关卡中呈现品味类决策
  （接近的方案、边界范围、codex 分歧）。一个命令，输出完整审查后的计划。
  当被要求“自动审查”“autoplan”“运行所有审查”“自动审查这个计划”
  或“替我做决定”时使用。
  当用户有一个计划文件，并且希望运行完整的审查流程而不想回答 15-30 个中间问题时，
  主动建议使用。
benefits-from: [office-hours]
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - WebSearch
  - AskUserQuestion
---
<!-- 从 SKILL.md.tmpl 自动生成 —— 请勿直接编辑 -->
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
echo '{"skill":"autoplan","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","repo":"'$(basename "$(git rev-parse --show-toplevel 2>/dev/null)" 2>/dev/null || echo "unknown")'"}'  >> ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true
for _PF in ~/.gstack/analytics/.pending-*; do [ -f "$_PF" ] && ~/.claude/skills/gstack/bin/gstack-telemetry-log --event-type skill_run --skill _pending_finalize --outcome unknown --session-id "$_SESSION_ID" 2>/dev/null || true; break; done
```

如果 `PROACTIVE` 为 `"false"`，不要主动建议 gstack 技能，只有在用户明确要求时才调用。
用户已选择退出主动建议。

如果输出显示 `UPGRADE_AVAILABLE <old> <new>`：读取 `~/.claude/skills/gstack/gstack-upgrade/SKILL.md` 并遵循“Inline upgrade flow”（如果已配置则自动升级，否则使用包含 4 个选项的 AskUserQuestion；如果用户拒绝，则写入 snooze 状态）。如果显示 `JUST_UPGRADED <from> <to>`：告诉用户“正在运行 gstack v{to}（刚刚已更新！）”并继续。

如果 `LAKE_INTRO` 为 `no`：继续之前，先介绍 Completeness Principle。
告诉用户：“gstack 遵循 **Boil the Lake** 原则 —— 当 AI 让边际成本接近于零时，始终把事情完整做完。了解更多：https://garryslist.org/posts/boil-the-ocean”
然后主动提出是否在默认浏览器中打开这篇文章：

```bash
open https://garryslist.org/posts/boil-the-ocean
touch ~/.gstack/.completeness-intro-seen
```

只有在用户同意时才运行 `open`。始终运行 `touch` 以标记为已查看。此操作只会发生一次。

如果 `TEL_PROMPTED` 为 `no` 且 `LAKE_INTRO` 为 `yes`：在完成 lake 介绍之后，
询问用户是否启用 telemetry。使用 AskUserQuestion：

> 帮助 gstack 变得更好！Community mode 会共享使用数据（你使用了哪些技能、耗时多久、
> 崩溃信息），并使用稳定设备 ID，以便我们跟踪趋势并更快修复问题。
> 绝不会发送代码、文件路径或仓库名称。
> 可随时使用 `gstack-config set telemetry off` 修改。

选项：
- A) 帮助 gstack 变得更好！（推荐）
- B) 不用了，谢谢

如果选择 A：运行 `~/.claude/skills/gstack/bin/gstack-config set telemetry community`

如果选择 B：继续追问一个 AskUserQuestion：

> 那 anonymous mode 怎么样？我们只会知道*有人*使用了 gstack —— 没有唯一 ID，
> 无法关联会话。只有一个计数器，帮助我们知道是否真的有人在使用。

选项：
- A) 可以，anonymous 就行
- B) 不用了，彻底关闭

如果 B→A：运行 `~/.claude/skills/gstack/bin/gstack-config set telemetry anonymous`
如果 B→B：运行 `~/.claude/skills/gstack/bin/gstack-config set telemetry off`

始终运行：
```bash
touch ~/.gstack/.telemetry-prompted
```

此操作只会发生一次。如果 `TEL_PROMPTED` 为 `yes`，则完全跳过此步骤。

## AskUserQuestion 格式

**每次调用 AskUserQuestion 时都必须遵循以下结构：**
1. **重新锚定：** 说明项目、当前分支（使用前言中打印的 `_BRANCH` 值——不要使用对话历史或 gitStatus 中的任何分支），以及当前计划/任务。（1-2 句话）
2. **简化说明：** 用一个聪明的 16 岁青少年也能理解的通俗英语解释问题。不要使用原始函数名、内部术语或实现细节。使用具体示例和类比。说明它“做什么”，而不是“它叫什么”。
3. **给出推荐：** `RECOMMENDATION: Choose [X] because [一句话理由]` —— 始终优先选择完整方案而不是捷径（见 Completeness Principle）。为每个选项包含 `Completeness: X/10`。标尺说明：10 = 完整实现（所有边界情况、完整覆盖），7 = 覆盖主路径但跳过部分边缘情况，3 = 推迟大量工作的捷径。如果两个选项都在 8+，选择更高者；如果其中一个 ≤5，明确标出。
4. **选项：** 使用字母选项：`A) ... B) ... C) ...` —— 当选项涉及工作量时，同时展示两个量级：`(human: ~X / CC: ~Y)`

默认用户已经有 20 分钟没看这个窗口，而且没有打开代码。如果你必须重新阅读源码才能理解你自己的解释，那就说明它太复杂了。

各个技能的说明可以在这个基础规则之上增加额外格式要求。

## Completeness Principle —— Boil the Lake

AI 辅助编程让完整性的边际成本接近于零。当你给出选项时：

- 如果选项 A 是完整实现（完整对齐、处理所有边界情况、100% 覆盖），而选项 B 是只节省少量工作的捷径 —— **始终推荐 A**。使用 CC+gstack 时，80 行和 150 行之间的差异没有意义。当“完整”只多花几分钟时，“差不多就行”是错误直觉。
- **Lake 与 ocean：** “lake” 是可以煮沸的 —— 某个模块 100% 测试覆盖、完整功能实现、处理所有边界情况、完整错误路径。“ocean” 则不是 —— 从零重写整个系统、为无法控制的依赖添加特性、跨多个季度的平台迁移。建议“煮沸” lakes。把 oceans 标记为超出范围。
- **估算工作量时，** 始终同时展示两个量级：人工团队时间和 CC+gstack 时间。压缩比例会随任务类型变化 —— 使用以下参考：

| Task type | Human team | CC+gstack | Compression |
|-----------|-----------|-----------|-------------|
| Boilerplate / scaffolding | 2 days | 15 min | ~100x |
| Test writing | 1 day | 15 min | ~50x |
| Feature implementation | 1 week | 30 min | ~30x |
| Bug fix + regression test | 4 hours | 15 min | ~20x |
| Architecture / design | 2 days | 4 hours | ~5x |
| Research / exploration | 1 day | 3 hours | ~3x |

- 这条原则适用于测试覆盖、错误处理、文档、边界情况和功能完整性。不要为了“节省时间”而跳过最后 10% —— 在 AI 帮助下，这 10% 只需要几秒。

**反模式 —— 不要这样做：**
- 错误：`Choose B — it covers 90% of the value with less code.`（如果 A 只多 70 行，就选 A。）
- 错误：`We can skip edge case handling to save time.`（使用 CC 时，边界情况处理只需要几分钟。）
- 错误：`Let's defer test coverage to a follow-up PR.`（测试是最便宜、最该彻底完成的 lake。）
- 错误：只引用人工团队工作量：`This would take 2 weeks.`（应该说：`2 weeks human / ~1 hour CC.`）

## Repo Ownership Mode —— 发现问题就要说

前言中的 `REPO_MODE` 表示这个仓库中的问题由谁负责：

- **`solo`** —— 一个人完成 80% 以上的工作。他负责所有事情。当你注意到当前分支变更之外的问题（测试失败、弃用警告、安全通告、lint 错误、死代码、环境问题）时，**主动调查并提出修复**。因为只有这位独立开发者会去修。默认直接行动。
- **`collaborative`** —— 有多位活跃贡献者。当你注意到当前分支变更之外的问题时，**通过 AskUserQuestion 提示出来** —— 这可能是别人的职责。默认先问，不直接修。
- **`unknown`** —— 按 collaborative 处理（更安全的默认方式 —— 先询问再修复）。

**发现问题就要说：** 无论在任何工作流步骤中，只要你注意到看起来有问题的地方 —— 不只是测试失败 —— 都要简要指出。用一句话说明：你注意到了什么，以及它的影响。在 solo 模式下，接着问“要我修吗？”；在 collaborative 模式下，只提示并继续。

绝不要让已注意到的问题悄悄掠过。核心目的就是主动沟通。

## 先搜索再构建

在构建基础设施、不熟悉的模式，或任何运行时可能已内建的能力之前 —— **先搜索。** 阅读 `~/.claude/skills/gstack/ETHOS.md` 以了解完整理念。

**三层知识：**
- **Layer 1**（经过验证 —— 已广泛分发）。不要重复造轮子。但检查的成本几乎为零，而偶尔质疑这些“久经验证”的东西，正是卓越洞察出现的地方。
- **Layer 2**（新且流行 —— 搜索这些）。但要仔细审视：人类容易陷入潮流狂热。搜索结果只是你思考的输入，不是答案。
- **Layer 3**（第一性原理 —— 最高优先级）。基于对具体问题推理得出的原创观察。这是最有价值的。

**Eureka 时刻：** 当第一性原理推理揭示传统认知是错的时，把它明确写出来：
`EUREKA: Everyone does X because [assumption]. But [evidence] shows this is wrong. Y is better because [reasoning].`

记录 eureka 时刻：
```bash
jq -n --arg ts "$(date -u +%Y-%m-%dT%H:%M:%SZ)" --arg skill "SKILL_NAME" --arg branch "$(git branch --show-current 2>/dev/null)" --arg insight "ONE_LINE_SUMMARY" '{ts:$ts,skill:$skill,branch:$branch,insight:$insight}' >> ~/.gstack/analytics/eureka.jsonl 2>/dev/null || true
```

替换 SKILL_NAME 和 ONE_LINE_SUMMARY。内联运行 —— 不要中断工作流。

**WebSearch 回退：** 如果 WebSearch 不可用，跳过搜索步骤，并注明：`Search unavailable — proceeding with in-distribution knowledge only.`

## Contributor Mode

如果 `_CONTRIB` 为 `true`：你处于 **contributor mode**。你既是 gstack 用户，也会帮助它变得更好。

**在每个主要工作流步骤结束时**（不是每执行一个命令后），回顾一下你使用过的 gstack 工具。给这次体验打 0 到 10 分。如果不是 10 分，想想原因。如果存在明显、可操作的 bug，或者有有洞察力、值得记录的地方，说明 gstack 代码或 skill markdown 本可以做得更好 —— 提交一份 field report。也许我们的贡献者会帮我们做得更好！

**评分校准 —— 标准如下：** 例如，`$B js "await fetch(...)"` 过去会报错 `SyntaxError: await is only valid in async functions`，因为 gstack 没有把表达式包装到 async 上下文中。问题虽小，但输入是合理的，gstack 理应处理好 —— 这类问题就值得提交。比这更无关紧要的问题，忽略。

**不值得提交：** 用户应用本身的 bug、访问用户 URL 的网络错误、用户站点的鉴权失败、用户自己的 JS 逻辑错误。

**提交方式：** 写入 `~/.gstack/contributor-logs/{slug}.md`，并包含**以下所有章节**（不要截断 —— 必须包含直到 Date/Version 页脚的每一节）：

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
{一句话：gstack 本应如何做得不同}

**Date:** {YYYY-MM-DD} | **Version:** {gstack version} | **Skill:** /{skill}
```

Slug：小写、连字符、最长 60 个字符（例如 `browse-js-no-await`）。如果文件已存在则跳过。每个会话最多 3 份报告。内联写入并继续 —— 不要中断工作流。告诉用户：`Filed gstack field report: {title}`

## Completion Status Protocol

完成一个技能工作流时，使用以下状态之一进行报告：
- **DONE** —— 所有步骤均已成功完成。每项声明都提供了证据。
- **DONE_WITH_CONCERNS** —— 已完成，但存在用户应知晓的问题。列出每一项顾虑。
- **BLOCKED** —— 无法继续。说明阻塞点以及已尝试的操作。
- **NEEDS_CONTEXT** —— 缺少继续所需的信息。准确说明你需要什么。

### 升级处理

始终可以停下来并说“这个对我来说太难了”或“我对这个结果没有信心”。

糟糕的工作比不做更糟。你不会因为升级处理而受到惩罚。
- 如果你已尝试某个任务 3 次仍未成功，停止并升级处理。
- 如果你对一个涉及安全敏感的变更不确定，停止并升级处理。
- 如果工作范围超出你能验证的程度，停止并升级处理。

升级处理格式：
```
STATUS: BLOCKED | NEEDS_CONTEXT
REASON: [1-2 sentences]
ATTEMPTED: [what you tried]
RECOMMENDATION: [what the user should do next]
```

## Telemetry（最后运行）

在技能工作流完成后（成功、出错或中止），记录 telemetry 事件。
从本文件 YAML frontmatter 中的 `name:` 字段确定技能名称。
从工作流结果确定 outcome（正常完成则为 success，
失败则为 error，用户中断则为 abort）。

**PLAN MODE 例外 —— 必须始终运行：** 此命令会将 telemetry 写入
`~/.gstack/analytics/`（用户配置目录，而不是项目文件）。技能前言已经写入同一目录 —— 这是同一种模式。
跳过这个命令会丢失会话时长和结果数据。

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
success/error/abort，并根据是否使用了 `$B` 将 `USED_BROWSE` 替换为 true/false。
如果无法确定 outcome，使用 `"unknown"`。该命令在后台运行，
永远不会阻塞用户。

## Plan Status Footer

当你处于 plan mode 并准备调用 ExitPlanMode 时：

1. 检查计划文件是否已经有 `## GSTACK REVIEW REPORT` 章节。
2. 如果**有** —— 跳过（说明某个 review 技能已经写入了更丰富的报告）。
3. 如果**没有** —— 运行以下命令：

\`\`\`bash
~/.claude/skills/gstack/bin/gstack-review-read
\`\`\`

然后把 `## GSTACK REVIEW REPORT` 章节写到计划文件末尾：

- 如果输出包含 review 条目（`---CONFIG---` 之前的 JSONL 行）：按
  标准报告表格格式输出每个 skill 的 runs/status/findings，格式与 review
  技能使用的格式相同。
- 如果输出为 `NO_REVIEWS` 或为空：写入以下占位表格：

\`\`\`markdown
## GSTACK REVIEW REPORT

| Review | Trigger | Why | Runs | Status | Findings |
|--------|---------|-----|------|--------|----------|
| CEO Review | \`/plan-ceo-review\` | 范围与策略 | 0 | — | — |
| Codex Review | \`/codex review\` | 独立的第二意见 | 0 | — | — |
| Eng Review | \`/plan-eng-review\` | 架构与测试（必需） | 0 | — | — |
| Design Review | \`/plan-design-review\` | UI/UX 缺口 | 0 | — | — |

**VERDICT:** 还没有任何审查 —— 运行 \`/autoplan\` 以执行完整审查流水线，或运行上面的单项审查。
\`\`\`

**PLAN MODE 例外 —— 必须始终运行：** 这会写入计划文件，而这是在 plan mode 中唯一允许编辑的
文件。计划文件中的 review 报告是计划动态状态的一部分。

## 第 0 步：检测 base branch

确定此 PR 目标指向哪个分支。后续所有步骤都将该结果作为“base branch”。

1. 检查当前分支是否已存在 PR：
   `gh pr view --json baseRefName -q .baseRefName`
   如果成功，使用打印出的分支名作为 base branch。

2. 如果不存在 PR（命令失败），检测仓库默认分支：
   `gh repo view --json defaultBranchRef -q .defaultBranchRef.name`

3. 如果这两个命令都失败，则回退到 `main`。

打印检测到的 base branch 名称。在后续每一个 `git diff`、`git log`、
`git fetch`、`git merge` 和 `gh pr create` 命令中，将说明里出现的“the base branch”
替换为检测到的实际分支名。

---

## 前置技能建议

如果上面的 design doc 检查打印出 `"No design doc found"`，在继续之前先建议使用前置技能。

通过 AskUserQuestion 对用户说：

> `"No design doc found for this branch. \`/office-hours\` produces a structured problem
> statement, premise challenge, and explored alternatives — it gives this review much
> sharper input to work with. Takes about 10 minutes. The design doc is per-feature,
> not per-product — it captures the thinking behind this specific change."`

选项：
- A) 先运行 /office-hours（在另一个窗口中运行，然后回来）
- B) 跳过 —— 继续标准审查

如果他们跳过：`No worries — standard review. If you ever want sharper input, try
/office-hours first next time.` 然后正常继续。本次会话中不要再次提出。

# /autoplan —— 自动审查流水线

一个命令。输入粗略计划，输出完整审查后的计划。

`/autoplan` 会从磁盘读取完整的 CEO、design 和 eng review 技能文件，并按完整深度执行
它们 —— 与手动逐个运行技能相比，严谨程度、章节内容和方法论完全相同。唯一的区别是：
中间的 AskUserQuestion 调用会依据下面的 6 条原则自动决策。品味类决策（理性的人可能会有分歧的地方）
会在最终审批关卡中呈现出来。

---

## 6 条决策原则

这些规则会自动回答所有中间问题：

1. **选择完整性** —— 一次把整件事做好。选择能覆盖更多边界情况的方案。
2. **煮沸 lakes** —— 修复影响半径内的所有内容（本计划修改的文件 + 直接导入方）。对于处于影响半径内且 CC 工作量 < 1 天（< 5 个文件、无新基础设施）的扩展，自动批准。
3. **务实** —— 如果两个选项修复的是同一件事，选更干净的那个。花 5 秒做决定，而不是 5 分钟。
4. **DRY** —— 如果重复了现有功能？拒绝。复用已有内容。
5. **明确优于巧妙** —— 10 行的直白修复优于 200 行抽象。选择新贡献者能在 30 秒内读明白的方案。
6. **偏向行动** —— 合并 > 审查循环 > 陈旧拖延。提出顾虑，但不要阻塞。

**冲突解决（按上下文决定的平局打破规则）：**
- **CEO 阶段：** P1（完整性）+ P2（煮沸 lakes）优先级最高。
- **Eng 阶段：** P5（明确）+ P3（务实）优先级最高。
- **Design 阶段：** P5（明确）+ P1（完整性）优先级最高。

---

## 决策分类

每个自动决策都会被归类：

**Mechanical** —— 只有一个明显正确的答案。静默自动决策。
示例：运行 codex（始终是）、运行 evals（始终是）、对一个完整计划缩小范围（始终否）。

**Taste** —— 理性的人可能会有不同意见。自动决策并给出推荐，但会在最终关卡中呈现。主要有三种自然来源：
1. **接近的方案** —— 排名前两位都可行，但权衡不同。
2. **边界范围** —— 在影响半径内，但涉及 3-5 个文件，或影响半径不明确。
3. **Codex 分歧** —— codex 给出了不同建议，而且理由成立。

---

## “自动决策”是什么意思

自动决策是用这 6 条原则替代“用户的判断”。它**不会**替代“分析”。
从已加载技能文件中的每个章节，仍然必须按与交互版本相同的深度执行。改变的只有
谁来回答 AskUserQuestion：由你依据 6 条原则回答，而不是由用户回答。

**你仍然必须：**
- 阅读每个章节引用的实际代码、diff 和文件
- 产出该章节要求的每一项输出（图表、表格、注册表、产物）
- 识别该章节设计用来发现的每一个问题
- 使用 6 条原则对每个问题做决策（而不是询问用户）
- 在审计轨迹中记录每一个决策
- 将所有必需产物写入磁盘

**你绝不能：**
- 把一个审查章节压缩成表格中的一行总结
- 在不展示你检查了什么的情况下写“no issues found”
- 因为“it doesn't apply”而跳过某个章节，却不说明你检查了什么以及为什么
- 用总结代替要求的输出（例如写“architecture looks good”，
  而不是该章节要求的 ASCII dependency graph）

“No issues found” 可以作为一个章节的有效输出 —— 但前提是已经完成分析。
要说明你检查了什么，以及为什么没有标记任何问题（至少 1-2 句话）。
对于未在跳过列表中的章节，“Skipped” 永远不是有效输出。

---

## 阶段 0：接收 + Restore Point

### 第 1 步：捕获 restore point

在做任何事情之前，将计划文件的当前状态保存到一个外部文件中：

```bash
source <(~/.claude/skills/gstack/bin/gstack-slug 2>/dev/null) && mkdir -p ~/.gstack/projects/$SLUG
BRANCH=$(git rev-parse --abbrev-ref HEAD 2>/dev/null | tr '/' '-')
DATETIME=$(date +%Y%m%d-%H%M%S)
echo "RESTORE_PATH=$HOME/.gstack/projects/$SLUG/${BRANCH}-autoplan-restore-${DATETIME}.md"
```

将计划文件的完整内容写入 restore path，并使用以下头部：
```
# /autoplan Restore Point
Captured: [timestamp] | Branch: [branch] | Commit: [short hash]

## Re-run Instructions
1. 将下面的 “Original Plan State” 复制回你的计划文件
2. 调用 /autoplan

## Original Plan State
[计划文件的逐字内容]
```

然后在计划文件最前面加上一行 HTML 注释：
`<!-- /autoplan restore point: [RESTORE_PATH] -->`

### 第 2 步：读取上下文

- 读取 CLAUDE.md、TODOS.md、git log -30、相对于 base branch 的 git diff --stat
- 发现 design docs：`ls -t ~/.gstack/projects/$SLUG/*-design-*.md 2>/dev/null | head -1`
- 检测 UI 范围：在计划中 grep 视图/渲染相关术语（component、screen、form、
  button、modal、layout、dashboard、sidebar、nav、dialog）。要求匹配 2 次以上。排除
  误报（单独出现的 "page"、缩写中的 "UI"）。

### 第 3 步：从磁盘加载技能文件

使用 Read 工具读取每个文件：
- `~/.claude/skills/gstack/plan-ceo-review/SKILL.md`
- `~/.claude/skills/gstack/plan-design-review/SKILL.md`（仅在检测到 UI 范围时）
- `~/.claude/skills/gstack/plan-eng-review/SKILL.md`

**章节跳过列表 —— 在跟随已加载技能文件时，跳过以下章节
（这些内容已由 /autoplan 处理）：**
- Preamble（run first）
- AskUserQuestion Format
- Completeness Principle — Boil the Lake
- Search Before Building
- Contributor Mode
- Completion Status Protocol
- Telemetry（run last）
- Step 0: Detect base branch
- Review Readiness Dashboard
- Plan File Review Report
- Prerequisite Skill Offer (BENEFITS_FROM)

只遵循 review 专属的方法论、章节和必需输出。

输出：`Here's what I'm working with: [plan summary]. UI scope: [yes/no].
Loaded review skills from disk. Starting full review pipeline with auto-decisions.`

---

## 阶段 1：CEO Review（策略与范围）

遵循 plan-ceo-review/SKILL.md —— 所有章节，完整深度执行。
覆盖规则：每一个 AskUserQuestion → 使用 6 条原则自动决策。

**覆盖规则：**
- 模式选择：SELECTIVE EXPANSION
- 前提：接受合理前提（P6），只质疑明显错误的前提
- **GATE: Present premises to user for confirmation** —— 这是唯一**不**自动决策的 AskUserQuestion。
  前提需要人类判断。
- 备选方案：选择完整性最高的（P1）。若持平，选最简单的（P5）。
  如果前两名接近 → 标记为 TASTE DECISION。
- 范围扩展：在影响半径内且 CC <1d → 批准（P2）。超出影响半径 → 延后到 TODOS.md（P3）。
  重复内容 → 拒绝（P4）。边界情况（3-5 个文件）→ 标记为 TASTE DECISION。
- 所有 10 个 review 章节：全部完整执行，对每个问题自动决策，并记录每一个决策。

**必需执行清单（CEO）：**

第 0 步（0A-0F）—— 运行每个子步骤并产出：
- 0A：前提挑战，明确点名前提并进行评估
- 0B：现有代码借力图（子问题 → 现有代码）
- 0C：理想终态图（CURRENT → THIS PLAN → 12-MONTH IDEAL）
- 0C-bis：实现备选方案表（2-3 种方法，含工作量/风险/优缺点）
- 0D：按模式进行分析，并记录范围决策
- 0E：时间维度审查（HOUR 1 → HOUR 6+）
- 0F：模式选择确认

第 1-10 节 —— 对于**每一节**，运行已加载技能文件中的评估标准：
- 有发现的章节：完整分析、对每个问题自动决策、记录到审计轨迹
- 没有发现的章节：用 1-2 句话说明检查了什么，以及为什么没有标记任何问题。
  绝不要把一个章节压缩成表格中的章节名一行。
- 第 11 节（Design）：仅在阶段 0 检测到 UI 范围时运行

**阶段 1 的强制输出：**
- “NOT in scope” 章节，列出延后事项及理由
- “What already exists” 章节，将子问题映射到现有代码
- Error & Rescue Registry 表格（来自第 2 节）
- Failure Modes Registry 表格（来自 review 章节）
- Dream state delta（本计划与 12 个月理想状态之间的差距）
- Completion Summary（CEO skill 的完整总结表）

---

## 阶段 2：Design Review（条件执行 —— 若无 UI 范围则跳过）

遵循 plan-design-review/SKILL.md —— 全部 7 个维度，完整深度执行。
覆盖规则：每一个 AskUserQuestion → 使用 6 条原则自动决策。

**覆盖规则：**
- 关注区域：所有相关维度（P1）
- 结构性问题（缺失状态、层级损坏）：自动修复（P5）
- 美学/品味问题：标记为 TASTE DECISION
- 设计系统对齐：如果存在 DESIGN.md 且修复方式明显，则自动修复

---

## 阶段 3：Eng Review + Codex

遵循 plan-eng-review/SKILL.md —— 所有章节，完整深度执行。
覆盖规则：每一个 AskUserQuestion → 使用 6 条原则自动决策。

**覆盖规则：**
- 范围质疑：绝不缩小（P2）
- Codex review：如可用则始终运行（P6）
  命令：`codex exec "Review this plan for architectural issues, missing edge cases, and hidden complexity. Be adversarial. File: <plan_path>" -s read-only --enable web_search_cached`
  超时：10 分钟，然后继续并写明 `Codex timed out — single-reviewer mode`
- 架构选择：明确优于巧妙（P5）。如果 codex 因有效理由不同意 → 标记为 TASTE DECISION。
- Evals：始终包含所有相关套件（P1）
- 测试计划：在 `~/.gstack/projects/$SLUG/{user}-{branch}-test-plan-{datetime}.md` 生成产物
- TODOS.md：收集阶段 1 中所有延后的范围扩展，并自动写入

**必需执行清单（Eng）：**

1. 第 0 步（Scope Challenge）：读取计划引用的实际代码。将每个
   子问题映射到现有代码。运行复杂度检查。产出具体发现。

2. 第 0.5 步（Codex）：如可用则运行。将完整输出置于 `CODEX SAYS` 标题下展示。

3. 第 1 节（Architecture）：生成 ASCII dependency graph，展示新组件
   及其与现有组件的关系。评估耦合、扩展性、安全性。

4. 第 2 节（Code Quality）：识别 DRY 违规、命名问题、复杂度问题。
   引用具体文件和模式。对每个发现自动决策。

5. **第 3 节（Test Review）—— 绝不跳过，也绝不压缩。**
   这一节要求阅读实际代码，不能凭记忆总结。
   - 读取 diff 或计划影响到的文件
   - 构建测试图：列出每一个新的 UX flow、data flow、codepath 和 branch
   - 对于图中的**每一项**：哪种测试会覆盖它？是否已存在？缺口在哪里？
   - 对于 LLM/prompt 变更：必须运行哪些 eval 套件？
   - 自动决策测试缺口意味着：识别缺口 → 决定是添加测试
     还是延后（附理由和所依据原则）→ 记录该决策。这**不**意味着
     跳过分析。
   - 将测试计划产物写入磁盘

6. 第 4 节（Performance）：评估 N+1 queries、内存、缓存、慢路径。

**阶段 3 的强制输出：**
- “NOT in scope” 章节
- “What already exists” 章节
- Architecture ASCII diagram（第 1 节）
- Test diagram，将 codepaths 映射到覆盖情况（第 3 节）
- 将测试计划产物写入磁盘（第 3 节）
- Failure modes registry，并标记关键缺口
- Completion Summary（Eng skill 的完整总结）
- TODOS.md 更新（从所有阶段收集）

---

## 决策审计轨迹

在每次自动决策之后，使用 Edit 向计划文件追加一行：

```markdown
<!-- AUTONOMOUS DECISION LOG -->
## Decision Audit Trail

| # | Phase | Decision | Principle | Rationale | Rejected |
|---|-------|----------|-----------|-----------|----------|
```

每个决策按增量写入一行（通过 Edit）。这样审计内容会保存在磁盘上，
而不会累计在对话上下文中。

---

## 关卡前验证

在呈现 Final Approval Gate 之前，验证所需输出是否确实已经生成。
检查计划文件和对话中的每一项。

**阶段 1（CEO）输出：**
- [ ] 已完成前提挑战，并明确点名前提（而不只是写 “premises accepted”）
- [ ] 所有适用的 review 章节都有发现，或者明确写出“已检查 X，未标记任何问题”
- [ ] 已生成 Error & Rescue Registry 表格（或注明不适用并给出原因）
- [ ] 已生成 Failure Modes Registry 表格（或注明不适用并给出原因）
- [ ] 已写入 “NOT in scope” 章节
- [ ] 已写入 “What already exists” 章节
- [ ] 已写入 Dream state delta
- [ ] 已生成 Completion Summary

**阶段 2（Design）输出 —— 仅在检测到 UI 范围时：**
- [ ] 已评估全部 7 个维度并给出分数
- [ ] 已识别问题并完成自动决策

**阶段 3（Eng）输出：**
- [ ] 已完成基于实际代码分析的范围质疑（而不只是写 “scope is fine”）
- [ ] 已生成 Architecture ASCII diagram
- [ ] 已生成将 codepaths 映射到测试覆盖的 Test diagram
- [ ] 已将测试计划产物写入 `~/.gstack/projects/$SLUG/`
- [ ] 已写入 “NOT in scope” 章节
- [ ] 已写入 “What already exists” 章节
- [ ] 已生成 failure modes registry，并包含关键缺口评估
- [ ] 已生成 Completion Summary

**审计轨迹：**
- [ ] Decision Audit Trail 中每个自动决策至少有一行记录（不能为空）

如果上面任意一项缺失，返回并补齐缺失输出。最多尝试 2 次 —— 如果重试两次后仍缺失，
则带着警告继续进入关卡，并注明哪些项目尚未完成。不要无限循环。

---

## 阶段 4：最终审批关卡

**在这里停止，并向用户呈现最终状态。**

以消息形式呈现，然后使用 AskUserQuestion：

```
## /autoplan 审查完成

### 计划摘要
[1-3 句摘要]

### 已做决策：[N] 总计（[M] 自动决策，[K] 个需要你选择）

### 你的选择（品味类决策）
[对于每一个品味类决策：]
**Choice [N]: [title]**（来自 [phase]）
我建议选择 [X] —— [principle]。但 [Y] 也可行：
  [如果你选择 Y，会带来的下游影响，用 1 句话说明]

### 自动决策：[M] 个决策 [参见计划文件中的 Decision Audit Trail]

### 审查评分
- CEO: [summary]
- Design: [summary or "skipped, no UI scope"]
- Eng: [summary]
- Codex: [summary or "unavailable"]

### 已延后到 TODOS.md
[自动延后的事项及理由]
```

**认知负载管理：**
- 0 个品味类决策：跳过 “Your Choices” 章节
- 1-7 个品味类决策：使用平铺列表
- 8 个以上：按 phase 分组。增加警告：`This plan had unusually high ambiguity ([N] taste decisions). Review carefully.`

AskUserQuestion 选项：
- A) 按当前内容批准（接受所有推荐）
- B) 带覆盖项批准（说明要修改哪些品味类决策）
- C) 追问（询问任何具体决策）
- D) 修订（计划本身需要修改）
- E) 拒绝（重新开始）

**选项处理：**
- A: 标记为 APPROVED，写入 review logs，并建议使用 /ship
- B: 询问具体要覆盖哪些项，应用后重新呈现关卡
- C: 自由回答，然后重新呈现关卡
- D: 进行修改，并重新运行受影响阶段（scope→1B，design→2，test plan→3，arch→3）。最多 3 个循环。
- E: 重新开始

---

## 完成：写入 Review Logs

在批准后，写入 3 条独立的 review log 记录，这样 /ship 的 dashboard 才能识别：

```bash
COMMIT=$(git rev-parse --short HEAD 2>/dev/null)
TIMESTAMP=$(date -u +%Y-%m-%dT%H:%M:%SZ)

~/.claude/skills/gstack/bin/gstack-review-log '{"skill":"plan-ceo-review","timestamp":"'"$TIMESTAMP"'","status":"clean","unresolved":0,"critical_gaps":0,"mode":"SELECTIVE_EXPANSION","via":"autoplan","commit":"'"$COMMIT"'"}'

~/.claude/skills/gstack/bin/gstack-review-log '{"skill":"plan-eng-review","timestamp":"'"$TIMESTAMP"'","status":"clean","unresolved":0,"critical_gaps":0,"issues_found":0,"mode":"FULL_REVIEW","via":"autoplan","commit":"'"$COMMIT"'"}'
```

如果阶段 2 已运行（存在 UI 范围）：
```bash
~/.claude/skills/gstack/bin/gstack-review-log '{"skill":"plan-design-review","timestamp":"'"$TIMESTAMP"'","status":"clean","unresolved":0,"via":"autoplan","commit":"'"$COMMIT"'"}'
```

将字段值替换为审查中的实际统计结果。

建议下一步：准备好创建 PR 时使用 `/ship`。

---

## 重要规则

- **绝不终止。** 用户选择了 /autoplan。尊重这个选择。呈现所有品味类决策，绝不要改为交互式审查。
- **前提是唯一关卡。** 唯一一个不自动决策的 AskUserQuestion 是阶段 1 中的前提确认。
- **记录每一个决策。** 不允许静默自动决策。每一个选择都必须在审计轨迹中有一行。
- **完整深度就必须完整深度。** 不要压缩或跳过已加载技能文件中的章节（阶段 0 跳过列表中的章节除外）。“完整深度”的意思是：阅读该章节要求你阅读的代码，产出该章节要求的输出，识别每一个问题，并对每一个问题做出决策。用一句话总结一个章节，不叫“完整深度”—— 那叫跳过。如果你发现自己对任何 review 章节写的内容少于 3 句话，你很可能就是在压缩。
- **产物就是交付物。** 测试计划产物、failure modes registry、error/rescue table、ASCII diagrams —— 审查完成时，这些内容必须已经存在于磁盘或计划文件中。如果它们不存在，审查就是不完整的。
- **顺序执行。** CEO → Design → Eng。每个阶段都建立在前一个阶段之上。