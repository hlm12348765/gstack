---
name: retro
version: 2.0.0
description: |
  每周工程复盘。分析提交历史、工作模式
  以及代码质量指标，并保留历史记录与趋势跟踪。
  具备团队感知能力：按个人拆分贡献，包含表扬点与成长空间。
  当被要求执行“weekly retro”、“what did we ship”或“engineering retrospective”时使用。
  在工作周或冲刺结束时主动建议使用。
allowed-tools:
  - Bash
  - Read
  - Write
  - Glob
  - AskUserQuestion
---
<!-- AUTO-GENERATED from SKILL.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->

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
echo '{"skill":"retro","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","repo":"'$(basename "$(git rev-parse --show-toplevel 2>/dev/null)" 2>/dev/null || echo "unknown")'"}'  >> ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true
for _PF in ~/.gstack/analytics/.pending-*; do [ -f "$_PF" ] && ~/.claude/skills/gstack/bin/gstack-telemetry-log --event-type skill_run --skill _pending_finalize --outcome unknown --session-id "$_SESSION_ID" 2>/dev/null || true; break; done
```

如果 `PROACTIVE` 是 `"false"`，不要主动建议 gstack skills，只有在用户明确请求时才调用它们。用户已选择退出主动建议。

如果输出显示 `UPGRADE_AVAILABLE <old> <new>`：读取 `~/.claude/skills/gstack/gstack-upgrade/SKILL.md` 并遵循“Inline upgrade flow”（如果已配置则自动升级，否则使用 AskUserQuestion 提供 4 个选项；如果用户拒绝，则写入 snooze 状态）。如果显示 `JUST_UPGRADED <from> <to>`：告诉用户“Running gstack v{to} (just updated!)”，然后继续。

如果 `LAKE_INTRO` 是 `no`：继续之前，先介绍 Completeness Principle。
告诉用户：“gstack 遵循 **Boil the Lake** 原则：当 AI 让边际成本接近于零时，就应始终把事情完整做好。阅读更多：https://garryslist.org/posts/boil-the-ocean”
然后询问是否要在他们的默认浏览器中打开这篇文章：

```bash
open https://garryslist.org/posts/boil-the-ocean
touch ~/.gstack/.completeness-intro-seen
```

只有在用户说同意时才运行 `open`。始终运行 `touch` 以标记为已查看。这只会发生一次。

如果 `TEL_PROMPTED` 是 `no` 且 `LAKE_INTRO` 是 `yes`：在处理完 lake intro 之后，
询问用户是否启用 telemetry。使用 AskUserQuestion：

> 帮助 gstack 变得更好！Community mode 会共享使用数据（你使用了哪些技能、耗时多久、
> 崩溃信息），并附带稳定的设备 ID，以便我们跟踪趋势并更快修复问题。
> 绝不会发送代码、文件路径或仓库名称。
> 可随时使用 `gstack-config set telemetry off` 更改。

选项：
- A) Help gstack get better! (recommended)
- B) No thanks

如果选择 A：运行 `~/.claude/skills/gstack/bin/gstack-config set telemetry community`

如果选择 B：继续追问一个 AskUserQuestion：

> 那 anonymous mode 呢？我们只会知道*有人*使用了 gstack，不会有唯一 ID，
> 也无法关联各次会话。只有一个计数器，帮助我们了解是否真的有人在使用。

选项：
- A) Sure, anonymous is fine
- B) No thanks, fully off

如果 B→A：运行 `~/.claude/skills/gstack/bin/gstack-config set telemetry anonymous`
如果 B→B：运行 `~/.claude/skills/gstack/bin/gstack-config set telemetry off`

始终运行：
```bash
touch ~/.gstack/.telemetry-prompted
```

这只会发生一次。如果 `TEL_PROMPTED` 是 `yes`，则完全跳过此步骤。

## AskUserQuestion 格式

**每次调用 AskUserQuestion 时都必须遵循此结构：**
1. **重新落地：** 说明项目、当前分支（使用前言中打印出的 `_BRANCH` 值，而不是对话历史或 gitStatus 中的任何分支），以及当前计划/任务。（1-2 句）
2. **简化说明：** 用一个聪明的 16 岁少年也能理解的自然英语解释问题。不要使用原始函数名、内部术语或实现细节。使用具体示例和类比。说明它“做什么”，而不是“它叫什么”。
3. **给出建议：** `RECOMMENDATION: Choose [X] because [one-line reason]`，始终优先推荐完整选项而不是捷径（见 Completeness Principle）。为每个选项都附上 `Completeness: X/10`。标定标准：10 = 完整实现（所有边界情况、完整覆盖），7 = 覆盖主路径但跳过部分边界情况，3 = 推迟大量工作的捷径。如果两个选项都在 8 以上，选更高的；如果有一个 ≤5，要明确标出。
4. **选项：** 使用字母选项：`A) ... B) ... C) ...`。当选项涉及投入时，同时显示两个量级：`(human: ~X / CC: ~Y)`

假设用户已经有 20 分钟没看这个窗口了，而且代码也没打开。如果你的解释需要先读源码才能明白，那就说明太复杂了。

各技能说明可以在这个基础格式之上再增加额外的格式规则。

## Completeness Principle — Boil the Lake

AI 辅助编码让“完整性”的边际成本接近于零。当你给出选项时：

- 如果 Option A 是完整实现（完全对齐、覆盖所有边界情况、100% 覆盖），而 Option B 是只节省少量工作量的捷径，**始终推荐 A**。在 CC+gstack 下，80 行和 150 行之间的差异毫无意义。当“完整”只多花几分钟时，“差不多就行”的直觉是错误的。
- **Lake vs. ocean：** “lake” 是可以煮沸的，例如一个模块的 100% 测试覆盖、完整功能实现、处理所有边界情况、完整错误路径。“ocean” 则不行，例如从零重写整个系统、为你无法控制的依赖新增功能、跨多个季度的平台迁移。要推荐把 lake 做完。对于 ocean，要标明超出范围。
- **估算工作量时**，始终同时给出两个量级：人工团队时间和 CC+gstack 时间。压缩比会随任务类型而变化，参考如下：

| 任务类型 | 人工团队 | CC+gstack | 压缩比 |
|-----------|-----------|-----------|-------------|
| Boilerplate / scaffolding | 2 天 | 15 分钟 | ~100x |
| Test writing | 1 天 | 15 分钟 | ~50x |
| Feature implementation | 1 周 | 30 分钟 | ~30x |
| Bug fix + regression test | 4 小时 | 15 分钟 | ~20x |
| Architecture / design | 2 天 | 4 小时 | ~5x |
| Research / exploration | 1 天 | 3 小时 | ~3x |

- 这一原则适用于测试覆盖、错误处理、文档、边界情况和功能完整性。不要为了“省时间”而跳过最后 10%，有了 AI，这 10% 只需要几秒。

**反模式 —— 不要这样做：**
- BAD: "Choose B — it covers 90% of the value with less code."（如果 A 只多 70 行，就应该选 A。）
- BAD: "We can skip edge case handling to save time."（使用 CC 时，处理边界情况只需几分钟。）
- BAD: "Let's defer test coverage to a follow-up PR."（测试是最便宜、最值得一口气做完的 lake。）
- BAD: 只引用人工团队投入："This would take 2 weeks."（应该说："2 weeks human / ~1 hour CC."）

## Repo Ownership Mode — See Something, Say Something

前言中的 `REPO_MODE` 告诉你这个仓库中的问题由谁负责：

- **`solo`** —— 一个人完成了 80% 以上的工作。他对所有内容负责。当你注意到当前分支变更之外的问题时（测试失败、弃用警告、安全公告、lint 错误、死代码、环境问题），**要调查并主动提出修复**。这个 solo 开发者是唯一会修它的人。默认直接行动。
- **`collaborative`** —— 有多个活跃贡献者。当你注意到当前分支变更之外的问题时，**通过 AskUserQuestion 提醒**，因为那可能是别人的职责。默认先问，不直接修。
- **`unknown`** —— 按 collaborative 处理（更安全的默认方式：先询问再处理）。

**See Something, Say Something：** 在任何工作流步骤中，只要你发现看起来不对劲的地方，不仅限于测试失败，都要简要指出。用一句话说明：你发现了什么，以及它的影响。在 solo 模式下，接一句“Want me to fix it?”。在 collaborative 模式下，只提醒然后继续。

绝不要让你注意到的问题悄无声息地被略过。这个机制的意义就在于主动沟通。

## Search Before Building

在构建基础设施、不熟悉的模式，或任何运行时可能已有内建能力的东西之前，**先搜索。** 阅读 `~/.claude/skills/gstack/ETHOS.md` 了解完整理念。

**三层知识：**
- **Layer 1**（久经验证，已在发行版中存在）。不要重复造轮子。但检查的成本几乎为零，而偶尔质疑这些“既有正确答案”，正是灵感出现的地方。
- **Layer 2**（新且流行，应优先搜索）。但要仔细审视：人类容易陷入狂热。搜索结果只是思考输入，不是答案。
- **Layer 3**（第一性原理，高于一切）。从对具体问题的推理中得出的原创观察。价值最高。

**Eureka moment：** 当基于第一性原理的推理表明传统认知是错的时，要明确命名：
"EUREKA: Everyone does X because [assumption]. But [evidence] shows this is wrong. Y is better because [reasoning]."

记录 eureka moments：
```bash
jq -n --arg ts "$(date -u +%Y-%m-%dT%H:%M:%SZ)" --arg skill "SKILL_NAME" --arg branch "$(git branch --show-current 2>/dev/null)" --arg insight "ONE_LINE_SUMMARY" '{ts:$ts,skill:$skill,branch:$branch,insight:$insight}' >> ~/.gstack/analytics/eureka.jsonl 2>/dev/null || true
```

替换 `SKILL_NAME` 和 `ONE_LINE_SUMMARY`。内联执行，不要中断工作流。

**WebSearch 回退：** 如果 WebSearch 不可用，跳过搜索步骤，并注明：“Search unavailable — proceeding with in-distribution knowledge only.”

## Contributor Mode

如果 `_CONTRIB` 是 `true`：你处于 **contributor mode**。你既是 gstack 用户，也会帮助它变得更好。

**在每个主要工作流步骤结束时**（不是每一条命令后），回顾你使用过的 gstack 工具。为这次体验打 0 到 10 分。如果不是 10 分，想想为什么。如果存在一个明显、可执行的 bug，或者某个有洞见、很有意思的地方本可以由 gstack 代码或 skill markdown 做得更好，就提交一份 field report。也许我们的 contributor 会帮助我们改进！

**评分基准 —— 这是门槛：** 例如，`$B js "await fetch(...)"` 以前会报错：`SyntaxError: await is only valid in async functions`，因为 gstack 没有把表达式包在 async 上下文里。问题虽小，但这个输入是合理的，gstack 本该正确处理，这就是值得提交的那类问题。比这影响更小的情况，忽略即可。

**不值得提交：** 用户应用自身的 bug、访问用户 URL 的网络错误、用户站点的认证失败、用户自己的 JS 逻辑 bug。

**提交方式：** 写入 `~/.gstack/contributor-logs/{slug}.md`，并包含**下面所有章节**（不要截断，必须包含直到 Date/Version footer 的每一部分）：

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
{一句话：gstack 本应如何做得更好}

**Date:** {YYYY-MM-DD} | **Version:** {gstack version} | **Skill:** /{skill}
```

Slug：小写、连字符、最长 60 个字符（例如 `browse-js-no-await`）。如果文件已存在则跳过。每个会话最多 3 份报告。内联写入并继续，不要中断工作流。告诉用户：“Filed gstack field report: {title}”

## Completion Status Protocol

完成一个 skill 工作流时，使用以下状态之一报告结果：
- **DONE** —— 所有步骤均已成功完成。每项结论都提供了证据。
- **DONE_WITH_CONCERNS** —— 已完成，但存在用户需要知道的问题。逐项列出每个 concern。
- **BLOCKED** —— 无法继续。说明阻塞点以及已经尝试过什么。
- **NEEDS_CONTEXT** —— 缺少继续所需的信息。明确说明你需要什么。

### Escalation

在任何时候，都可以停下来并说“这对我来说太难了”或“我对这个结果没有信心”。

糟糕的工作比不做更差。你不会因为升级处理而受到惩罚。
- 如果你已经尝试某个任务 3 次仍未成功，停止并升级处理。
- 如果你对某个安全敏感变更不确定，停止并升级处理。
- 如果工作范围超出你能验证的程度，停止并升级处理。

升级处理格式：
```
STATUS: BLOCKED | NEEDS_CONTEXT
REASON: [1-2 句话]
ATTEMPTED: [你尝试了什么]
RECOMMENDATION: [用户下一步应该做什么]
```

## Telemetry（最后运行）

在 skill 工作流完成后（成功、报错或中止），记录 telemetry 事件。
从本文件 YAML frontmatter 中的 `name:` 字段确定 skill 名称。
根据工作流结果确定 outcome（正常完成则为 success，
失败则为 error，用户中断则为 abort）。

**PLAN MODE EXCEPTION — ALWAYS RUN：** 此命令会将 telemetry 写入
`~/.gstack/analytics/`（用户配置目录，而不是项目文件）。skill 的
前言已经写入过同一目录，这里沿用的是同一种模式。
跳过这个命令会丢失会话时长和 outcome 数据。

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
绝不会阻塞用户。

## Plan Status Footer

当你处于 plan mode 并即将调用 ExitPlanMode 时：

1. 检查计划文件是否已经包含 `## GSTACK REVIEW REPORT` 小节。
2. 如果**有**，则跳过（说明 review skill 已经写入了更丰富的报告）。
3. 如果**没有**，则运行以下命令：

\`\`\`bash
~/.claude/skills/gstack/bin/gstack-review-read
\`\`\`

然后在计划文件末尾写入一个 `## GSTACK REVIEW REPORT` 小节：

- 如果输出包含 review 条目（`---CONFIG---` 之前的 JSONL 行）：按
  review skills 使用的相同格式，写出标准报告表，包含每个 skill 的 runs/status/findings。
- 如果输出是 `NO_REVIEWS` 或为空：写入以下占位表：

\`\`\`markdown
## GSTACK REVIEW REPORT

| Review | Trigger | Why | Runs | Status | Findings |
|--------|---------|-----|------|--------|----------|
| CEO Review | \`/plan-ceo-review\` | 范围与策略 | 0 | — | — |
| Codex Review | \`/codex review\` | 独立的第二意见 | 0 | — | — |
| Eng Review | \`/plan-eng-review\` | 架构与测试（必需） | 0 | — | — |
| Design Review | \`/plan-design-review\` | UI/UX 缺口 | 0 | — | — |

**VERDICT:** 还没有任何 Review —— 运行 \`/autoplan\` 获取完整 review 流水线，或单独运行上述某个 review。
\`\`\`

**PLAN MODE EXCEPTION — ALWAYS RUN：** 这会写入计划文件，而计划文件是你在 plan mode 中唯一允许编辑的文件。计划文件中的 review report 是计划动态状态的一部分。

## 检测默认分支

在收集数据之前，检测仓库的默认分支名称：
`gh repo view --json defaultBranchRef -q .defaultBranchRef.name`

如果失败，则回退到 `main`。凡是下面说明中提到 `origin/<default>` 的地方，都使用检测到的名称。

---

# /retro — 每周工程复盘

生成一份全面的工程复盘，分析提交历史、工作模式和代码质量指标。具备团队感知能力：识别运行命令的用户，然后分析每位贡献者，并给出按个人拆分的表扬和成长机会。为使用 Claude Code 作为生产力倍增器的资深 IC/CTO 级建设者而设计。

## 用户可调用
当用户输入 `/retro` 时，运行此 skill。

## 参数
- `/retro` —— 默认：最近 7 天
- `/retro 24h` —— 最近 24 小时
- `/retro 14d` —— 最近 14 天
- `/retro 30d` —— 最近 30 天
- `/retro compare` —— 比较当前时间窗与前一个等长时间窗
- `/retro compare 14d` —— 使用显式时间窗进行比较
- `/retro global` —— 跨所有 AI 编码工具的跨项目复盘（默认 7d）
- `/retro global 14d` —— 使用显式时间窗的跨项目复盘

## 说明

解析参数以确定时间窗口。如果没有提供参数，默认使用 7 天。所有时间都应以用户的**本地时区**报告（使用系统默认值，不要设置 `TZ`）。

**按午夜对齐的窗口：** 对于天（`d`）和周（`w`）单位，计算本地午夜的绝对起始日期，而不是相对字符串。例如，如果今天是 2026-03-18，窗口为 7 天：起始日期就是 2026-03-11。对 git log 查询使用 `--since="2026-03-11T00:00:00"`，显式的 `T00:00:00` 后缀可确保 git 从午夜开始。否则，git 会使用当前墙钟时间（例如，晚上 11 点执行 `--since="2026-03-11"`，含义就是晚上 11 点，而不是午夜）。对于周单位，将其乘以 7 得到天数（例如 `2w` = 往前 14 天）。对于小时（`h`）单位，使用 `--since="N hours ago"`，因为午夜对齐不适用于不足一天的窗口。

**参数校验：** 如果参数不匹配“数字 + `d`、`h` 或 `w`”，也不是单词 `compare`（可选后跟一个窗口），或单词 `global`（可选后跟一个窗口），则显示以下用法并停止：
```
Usage: /retro [window | compare | global]
  /retro              — 最近 7 天（默认）
  /retro 24h          — 最近 24 小时
  /retro 14d          — 最近 14 天
  /retro 30d          — 最近 30 天
  /retro compare      — 比较当前周期与前一周期
  /retro compare 14d  — 使用显式窗口进行比较
  /retro global       — 跨所有 AI 工具的跨项目复盘（默认 7d）
  /retro global 14d   — 使用显式窗口的跨项目复盘
```

**如果第一个参数是 `global`：** 跳过普通的仓库范围复盘（步骤 1-14）。改为遵循本文末尾的 **Global Retrospective** 流程。可选的第二个参数是时间窗口（默认 7d）。此模式**不要求**位于 git 仓库内部。

### 步骤 1：收集原始数据

首先，抓取 origin 并识别当前用户：
```bash
git fetch origin <default> --quiet
# 识别是谁在运行这次 retro
git config user.name
git config user.email
```

`git config user.name` 返回的名字就是**“你”**，也就是正在阅读这份复盘的人。所有其他作者都是团队成员。使用这一点来组织叙述：“你的”提交与队友的贡献。

并行运行所有这些 git 命令（它们彼此独立）：

```bash
# 1. 窗口内的所有提交，包含时间戳、主题、hash、AUTHOR、变更文件、插入、删除
git log origin/<default> --since="<window>" --format="%H|%aN|%ae|%ai|%s" --shortstat

# 2. 按提交统计测试代码与总代码 LOC，并带作者
#    每个提交块以 COMMIT:<hash>|<author> 开始，后跟 numstat 行。
#    将测试文件（匹配 test/|spec/|__tests__/）与生产文件分开。
git log origin/<default> --since="<window>" --format="COMMIT:%H|%aN" --numstat

# 3. 用于会话检测和按小时分布的提交时间戳（带作者）
git log origin/<default> --since="<window>" --format="%at|%aN|%ai|%s" | sort -n

# 4. 变更最频繁的文件（热点分析）
git log origin/<default> --since="<window>" --format="" --name-only | grep -v '^$' | sort | uniq -c | sort -rn

# 5. 从提交消息中提取 PR 编号（提取 #NNN 模式）
git log origin/<default> --since="<window>" --format="%s" | grep -oE '#[0-9]+' | sed 's/^#//' | sort -n | uniq | sed 's/^/#/'

# 6. 按作者拆分的文件热点（谁在改什么）
git log origin/<default> --since="<window>" --format="AUTHOR:%aN" --name-only

# 7. 按作者统计提交数（快速摘要）
git shortlog origin/<default> --since="<window>" -sn --no-merges

# 8. Greptile 分诊历史（如果可用）
cat ~/.gstack/greptile-history.md 2>/dev/null || true

# 9. TODOS.md backlog（如果可用）
cat TODOS.md 2>/dev/null || true

# 10. 测试文件数量
find . -name '*.test.*' -o -name '*.spec.*' -o -name '*_test.*' -o -name '*_spec.*' 2>/dev/null | grep -v node_modules | wc -l

# 11. 窗口内的回归测试提交
git log origin/<default> --since="<window>" --oneline --grep="test(qa):" --grep="test(design):" --grep="test: coverage"

# 12. gstack skill 使用 telemetry（如果可用）
cat ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true

# 12. 窗口内发生变更的测试文件
git log origin/<default> --since="<window>" --format="" --name-only | grep -E '\.(test|spec)\.' | sort -u | wc -l
```

### 步骤 2：计算指标

计算并在摘要表中展示以下指标：

| 指标 | 值 |
|--------|-------|
| 提交到 main 的次数 | N |
| 贡献者数量 | N |
| 合并的 PR 数 | N |
| 总插入行数 | N |
| 总删除行数 | N |
| 净新增 LOC | N |
| 测试 LOC（插入） | N |
| 测试 LOC 占比 | N% |
| 版本范围 | vX.Y.Z.W → vX.Y.Z.W |
| 活跃天数 | N |
| 检测到的会话数 | N |
| 平均每会话小时 LOC | N |
| Greptile 信号 | N%（Y 次捕获，Z 次误报） |
| 测试健康度 | 测试总数 N · 本周期新增 M · 回归测试 K |

然后立刻在下方展示一个**按作者的排行榜**：

```
Contributor         Commits   +/-          Top area
You (garry)              32   +2400/-300   browse/
alice                    12   +800/-150    app/services/
bob                       3   +120/-40     tests/
```

按提交数降序排序。当前用户（来自 `git config user.name`）始终显示在第一位，标记为 “You (name)”。

**Greptile 信号（如果历史存在）：** 读取 `~/.gstack/greptile-history.md`（步骤 1 的第 8 条命令已获取）。按日期过滤出位于本次复盘时间窗口内的条目。按类型统计：`fix`、`fp`、`already-fixed`。计算信号比率：`(fix + already-fixed) / (fix + already-fixed + fp)`。如果窗口内没有条目，或文件不存在，则跳过 Greptile 指标行。无法解析的行静默跳过。

**Backlog Health（如果 `TODOS.md` 存在）：** 读取 `TODOS.md`（步骤 1 的第 9 条命令已获取）。计算：
- 未完成 TODO 总数（排除 `## Completed` 小节中的条目）
- P0/P1 数量（关键/紧急项）
- P2 数量（重要项）
- 本周期已完成条目数（Completed 小节中日期落在复盘窗口内的条目）
- 本周期新增条目数（交叉参考 git log 中窗口内修改 `TODOS.md` 的提交）

在指标表中加入：
```
| Backlog Health | N 个未完成（X 个 P0/P1，Y 个 P2）· 本周期完成 Z 个 |
```

如果 `TODOS.md` 不存在，则跳过 Backlog Health 行。

**Skill Usage（如果 analytics 存在）：** 如果 `~/.gstack/analytics/skill-usage.jsonl` 存在，则读取。按 `ts` 字段过滤出位于复盘窗口内的条目。将 skill 激活（无 `event` 字段）与 hook 触发（`event: "hook_fire"`）分开。按 skill 名称聚合。展示为：

```
| Skill Usage | /ship(12) /qa(8) /review(5) · 3 次 safety hook 触发 |
```

如果 JSONL 文件不存在，或窗口内没有条目，则跳过 Skill Usage 行。

**Eureka Moments（如果有记录）：** 如果 `~/.gstack/analytics/eureka.jsonl` 存在，则读取。按 `ts` 字段过滤出位于复盘窗口内的条目。对于每个 eureka moment，展示标记它的 skill、分支以及一句话总结该洞见。展示为：

```
| Eureka Moments | 本周期 2 次 |
```

如果存在这些记录，则列出：
```
  EUREKA /office-hours (branch: garrytan/auth-rethink): "Session tokens 不需要服务端存储 —— 浏览器 crypto API 让客户端 JWT 校验成为可行方案"
  EUREKA /plan-eng-review (branch: garrytan/cache-layer): "这里不需要 Redis —— Bun 内建的 LRU cache 足以处理这类负载"
```

如果 JSONL 文件不存在，或窗口内没有条目，则跳过 Eureka Moments 行。

### 步骤 3：提交时间分布

使用本地时间的条形图显示按小时分布的直方图：

```
Hour  Commits  ████████████████
 00:    4      ████
 07:    5      █████
 ...
```

识别并指出：
- 峰值时段
- 冷区时段
- 模式是双峰（早晨/晚上）还是连续分布
- 深夜编码簇（晚上 10 点之后）

### 步骤 4：工作会话检测

使用连续两次提交之间 **45 分钟间隔** 作为阈值来检测会话。对每个会话报告：
- 开始/结束时间（Pacific）
- 提交数
- 持续时长（分钟）

对会话进行分类：
- **Deep sessions**（50 分钟以上）
- **Medium sessions**（20-50 分钟）
- **Micro sessions**（少于 20 分钟，通常是单提交的即做即走）

计算：
- 总活跃编码时长（所有会话时长之和）
- 平均会话长度
- 每小时活跃时间的 LOC

### 步骤 5：提交类型拆分

按 conventional commit 前缀分类（feat/fix/refactor/test/chore/docs）。以百分比条形图展示：

```
feat:     20  (40%)  ████████████████████
fix:      27  (54%)  ███████████████████████████
refactor:  2  ( 4%)  ██
```

如果 fix 占比超过 50%，要明确指出，这表明存在“快速发布、快速修补”的模式，可能意味着评审存在缺口。

### 步骤 6：热点分析

显示变更次数最多的前 10 个文件。标出：
- 被修改 5 次以上的文件（变更热点）
- 热点列表中的测试文件与生产文件
- VERSION/CHANGELOG 的出现频率（版本纪律指标）

### 步骤 7：PR 规模分布

根据提交 diff 估算 PR 规模，并分桶：
- **Small**（<100 LOC）
- **Medium**（100-500 LOC）
- **Large**（500-1500 LOC）
- **XL**（1500+ LOC）

### 步骤 8：Focus Score + Ship of the Week

**Focus score：** 计算触及单个变更最多的顶层目录（例如 `app/services/`、`app/views/`）的提交占比。分数越高 = 工作越聚焦深入。分数越低 = 上下文切换越分散。报告格式如下：“Focus score: 62% (app/services/)”

**Ship of the week：** 自动识别窗口内 LOC 最高的单个 PR。重点展示：
- PR 编号和标题
- 变更的 LOC
- 为什么它重要（根据提交消息和触及文件推断）

### 步骤 9：团队成员分析

对每位贡献者（包括当前用户）计算：

1. **提交数和 LOC** —— 总提交数、插入、删除、净 LOC
2. **关注领域** —— 他们最常触及的目录/文件（前 3）
3. **提交类型混合** —— 他们个人的 feat/fix/refactor/test 拆分
4. **会话模式** —— 他们通常何时编码（峰值时段）、会话数
5. **测试纪律** —— 他们个人的测试 LOC 占比
6. **最大 ship** —— 他们在窗口内影响力最大的单个提交或 PR

**对于当前用户（“You”）：** 这一部分要写得最深入。包含 solo retro 中的所有细节：会话分析、时间模式、focus score。使用第二人称表述：“Your peak hours...”、“Your biggest ship...”

**对于每位队友：** 用 2-3 句话说明他们做了什么，以及他们的工作模式。然后：

- **Praise**（1-2 个具体点）：必须锚定在真实提交上。不要写“great work”，而要准确说出好在哪里。示例：“在 3 个高聚焦会话中完成了整个 auth middleware 重写，并带有 45% 的测试覆盖”，“每个 PR 都控制在 200 LOC 以内，拆分纪律很好。”
- **Opportunity for growth**（1 个具体点）：作为进阶建议，而不是批评。必须锚定真实数据。示例：“这周测试占比只有 12%，在 payment module 变得更复杂之前补上测试覆盖会很值得”，“同一个文件上出现 5 个 fix 提交，说明原始 PR 可能需要一次额外评审。”

**如果只有一个贡献者（solo repo）：** 跳过团队拆分，按原样继续，复盘就是个人复盘。

**如果存在 Co-Authored-By trailers：** 解析提交消息中的 `Co-Authored-By:` 行。除主要作者外，也把这些作者计入该提交的贡献。记录 AI co-authors（例如 `noreply@anthropic.com`），但不要把他们纳入团队成员；改为将 “AI-assisted commits” 作为单独指标统计。

### 步骤 10：按周趋势（如果窗口 >= 14d）

如果时间窗口为 14 天或更长，将其拆分为按周分桶并展示趋势：
- 每周提交数（总数及按作者拆分）
- 每周 LOC
- 每周测试占比
- 每周 fix 占比
- 每周会话数

### 步骤 11：连续记录跟踪

统计从今天往回、连续每天至少有 1 次提交到 `origin/<default>` 的天数。同时跟踪团队连续记录和个人连续记录：

```bash
# 团队连续记录：所有唯一提交日期（本地时间）—— 无硬性截止
git log origin/<default> --format="%ad" --date=format:"%Y-%m-%d" | sort -u

# 个人连续记录：仅当前用户的提交
git log origin/<default> --author="<user_name>" --format="%ad" --date=format:"%Y-%m-%d" | sort -u
```

从今天往回计数：有多少连续天数至少有一次提交？这个查询会遍历完整历史，因此任意长度的连续记录都能准确报告。显示这两项：
- “Team shipping streak: 47 consecutive days”
- “Your shipping streak: 32 consecutive days”

### 步骤 12：加载历史并比较

在保存新快照之前，检查是否存在之前的 retro 历史：

```bash
ls -t .context/retros/*.json 2>/dev/null
```

**如果存在之前的 retros：** 使用 Read tool 加载最新的一份。计算关键指标的变化量，并加入一个 **Trends vs Last Retro** 小节：
```
                    Last        Now         Delta
Test ratio:         22%    →    41%         ↑19pp
Sessions:           10     →    14          ↑4
LOC/hour:           200    →    350         ↑75%
Fix ratio:          54%    →    30%         ↓24pp (improving)
Commits:            32     →    47          ↑47%
Deep sessions:      3      →    5           ↑2
```

**如果不存在之前的 retros：** 跳过比较小节，并追加：“First retro recorded — run again next week to see trends.”

### 步骤 13：保存 Retro 历史

在计算完所有指标之后（包括 streak），并加载完用于比较的历史之后，保存一个 JSON 快照：

```bash
mkdir -p .context/retros
```

确定今天的下一个序号（将 `$(date +%Y-%m-%d)` 替换为实际日期）：
```bash
# 统计今天已有多少个 retro，以得到下一个序号
today=$(date +%Y-%m-%d)
existing=$(ls .context/retros/${today}-*.json 2>/dev/null | wc -l | tr -d ' ')
next=$((existing + 1))
# 保存为 .context/retros/${today}-${next}.json
```

使用 Write tool 按以下 schema 保存 JSON 文件：
```json
{
  "date": "2026-03-08",
  "window": "7d",
  "metrics": {
    "commits": 47,
    "contributors": 3,
    "prs_merged": 12,
    "insertions": 3200,
    "deletions": 800,
    "net_loc": 2400,
    "test_loc": 1300,
    "test_ratio": 0.41,
    "active_days": 6,
    "sessions": 14,
    "deep_sessions": 5,
    "avg_session_minutes": 42,
    "loc_per_session_hour": 350,
    "feat_pct": 0.40,
    "fix_pct": 0.30,
    "peak_hour": 22,
    "ai_assisted_commits": 32
  },
  "authors": {
    "Garry Tan": { "commits": 32, "insertions": 2400, "deletions": 300, "test_ratio": 0.41, "top_area": "browse/" },
    "Alice": { "commits": 12, "insertions": 800, "deletions": 150, "test_ratio": 0.35, "top_area": "app/services/" }
  },
  "version_range": ["1.16.0.0", "1.16.1.0"],
  "streak_days": 47,
  "tweetable": "Week of Mar 1: 47 commits (3 contributors), 3.2k LOC, 38% tests, 12 PRs, peak: 10pm",
  "greptile": {
    "fixes": 3,
    "fps": 1,
    "already_fixed": 2,
    "signal_pct": 83
  }
}
```

**注意：** 只有当 `~/.gstack/greptile-history.md` 存在且窗口内有条目时，才包含 `greptile` 字段。只有当 `TODOS.md` 存在时，才包含 `backlog` 字段。只有在找到测试文件时（第 10 条命令返回 > 0），才包含 `test_health` 字段。如果某项没有数据，则完全省略该字段。

在存在测试文件时，将测试健康度数据写入 JSON：
```json
  "test_health": {
    "total_test_files": 47,
    "tests_added_this_period": 5,
    "regression_test_commits": 3,
    "test_files_changed": 8
  }
```

在 `TODOS.md` 存在时，将 backlog 数据写入 JSON：
```json
  "backlog": {
    "total_open": 28,
    "p0_p1": 2,
    "p2": 8,
    "completed_this_period": 3,
    "added_this_period": 1
  }
```

### 步骤 14：撰写叙述

输出结构如下：

---

**可发推摘要**（第一行，放在最前面）：
```
Week of Mar 1: 47 commits (3 contributors), 3.2k LOC, 38% tests, 12 PRs, peak: 10pm | Streak: 47d
```

## Engineering Retro: [日期范围]

### Summary Table
（来自步骤 2）

### Trends vs Last Retro
（来自步骤 11，保存前加载；首次 retro 时跳过）

### Time & Session Patterns
（来自步骤 3-4）

叙述应解释团队整体模式意味着什么：
- 最高产的时间段是什么，背后驱动因素是什么
- 会话随着时间是在变长还是变短
- 估算每天的活跃编码小时数（团队总量）
- 值得注意的模式：团队成员是在同一时间编码，还是分时段接力？

### Shipping Velocity
（来自步骤 5-7）

叙述应覆盖：
- 提交类型混合及其揭示的信息
- PR 规模分布及其对发布节奏的揭示
- fix-chain 检测（同一子系统上的连续 fix 提交序列）
- 版本升级纪律

### Code Quality Signals
- 测试 LOC 占比趋势
- 热点分析（是否总是同一批文件反复变动？）
- Greptile 信号比率及趋势（如果历史存在）：“Greptile: X% signal (Y valid catches, Z false positives)”

### Test Health
- 测试文件总数：N（来自命令 10）
- 本周期新增测试：M（来自命令 12 —— 发生变更的测试文件）
- 回归测试提交：列出命令 11 中的 `test(qa):`、`test(design):` 和 `test: coverage` 提交
- 如果存在之前的 retro 且包含 `test_health`：显示变化量 “Test count: {last} → {now} (+{delta})”
- 如果测试占比 < 20%：标记为成长空间 —— “100% test coverage is the goal. Tests make vibe coding safe.”

### Focus & Highlights
（来自步骤 8）
- Focus score 及其解读
- Ship of the week 重点提示

### Your Week（个人深挖）
（来自步骤 9，仅针对当前用户）

这是用户最关心的部分。包括：
- 他们个人的提交数、LOC、测试占比
- 他们的会话模式与峰值时段
- 他们的关注领域
- 他们最大的 ship
- **What you did well**（2-3 个锚定在提交上的具体点）
- **Where to level up**（1-2 个具体、可执行的建议）

### Team Breakdown
（来自步骤 9，针对每位队友；solo repo 时跳过）

对于每位队友（按提交数降序排序），写一个小节：

#### [Name]
- **What they shipped**：2-3 句话说明他们的贡献、关注领域和提交模式
- **Praise**：1-2 个他们做得好的具体点，必须锚定真实提交。要真诚，像你在 1:1 里真的会说的话。示例：
  - “用 3 个小而易审的 PR 清理了整个 auth module —— 这是教科书级的拆分”
  - “为每个新 endpoint 都加了集成测试，而不只是 happy path”
  - “修复了导致 dashboard 加载 2 秒的 N+1 查询问题”
- **Opportunity for growth**：1 条具体、建设性的建议。要体现为投入建议，而不是批评。示例：
  - “payment module 的测试覆盖率只有 8%，在下一个功能叠加之前值得先投入补齐”
  - “大多数提交都集中在一次爆发中，分散到全天可能有助于减少上下文切换疲劳”
  - “所有提交都发生在凌晨 1-4 点，长期来看，可持续的节奏对代码质量很重要”

**AI collaboration note：** 如果很多提交带有 `Co-Authored-By` 的 AI trailers（例如 Claude、Copilot），则将 AI-assisted commit 百分比作为团队指标说明。中性表述即可 —— “N% of commits were AI-assisted” —— 不做价值判断。

### Top 3 Team Wins
识别窗口内整个团队发布的 3 件影响力最大的事情。对每一项：
- 它是什么
- 谁发布的
- 为什么重要（产品/架构影响）

### 3 Things to Improve
必须具体、可执行，并锚定在真实提交上。混合个人与团队层面的建议。措辞用 “to get even better, the team could...”

### 3 Habits for Next Week
小而实际、可落地。每条都必须是少于 5 分钟就能开始采用的做法。至少有一条应面向团队（例如 “review each other's PRs same-day”）。

### Week-over-Week Trends
（如适用，来自步骤 10）

---

## Global Retrospective Mode

当用户运行 `/retro global`（或 `/retro global 14d`）时，执行此流程，而不是仓库范围的步骤 1-14。此模式可以在任意目录运行，**不要求**位于 git 仓库内部。

### Global Step 1：计算时间窗口

与普通 retro 相同的午夜对齐逻辑。默认 7d。`global` 之后的第二个参数是窗口（例如 `14d`、`30d`、`24h`）。

### Global Step 2：运行 discovery

使用如下回退链来定位并运行 discovery 脚本：

```bash
DISCOVER_BIN=""
[ -x ~/.claude/skills/gstack/bin/gstack-global-discover ] && DISCOVER_BIN=~/.claude/skills/gstack/bin/gstack-global-discover
[ -z "$DISCOVER_BIN" ] && [ -x .claude/skills/gstack/bin/gstack-global-discover ] && DISCOVER_BIN=.claude/skills/gstack/bin/gstack-global-discover
[ -z "$DISCOVER_BIN" ] && which gstack-global-discover >/dev/null 2>&1 && DISCOVER_BIN=$(which gstack-global-discover)
[ -z "$DISCOVER_BIN" ] && [ -f bin/gstack-global-discover.ts ] && DISCOVER_BIN="bun run bin/gstack-global-discover.ts"
echo "DISCOVER_BIN: $DISCOVER_BIN"
```

如果没有找到任何可执行文件，告诉用户：“Discovery script not found. Run `bun run build` in the gstack directory to compile it.” 然后停止。

运行 discovery：
```bash
$DISCOVER_BIN --since "<window>" --format json 2>/tmp/gstack-discover-stderr
```

从 `/tmp/gstack-discover-stderr` 读取 stderr 输出，用于诊断信息。解析 stdout 上的 JSON 输出。

如果 `total_sessions` 为 0，则输出：“No AI coding sessions found in the last <window>. Try a longer window: `/retro global 30d`” 并停止。

### Global Step 3：对每个发现的仓库运行 git log

对 discovery JSON 中 `repos` 数组里的每个仓库，找到 `paths[]` 中第一个有效路径（目录存在且包含 `.git/`）。如果没有有效路径，则跳过该仓库并注明。

**对于仅本地仓库**（`remote` 以 `local:` 开头）：跳过 `git fetch`，并使用本地默认分支。使用 `git log HEAD` 而不是 `git log origin/$DEFAULT`。

**对于有远程仓库的仓库：**

```bash
git -C <path> fetch origin --quiet 2>/dev/null
```

检测每个仓库的默认分支：先尝试 `git symbolic-ref refs/remotes/origin/HEAD`，然后检查常见分支名（`main`、`master`），最后回退到 `git rev-parse --abbrev-ref HEAD`。在下面命令中，将检测到的分支用作 `<default>`。

```bash
# 带统计信息的提交
git -C <path> log origin/$DEFAULT --since="<start_date>T00:00:00" --format="%H|%aN|%ai|%s" --shortstat

# 用于会话检测、连续记录和上下文切换的提交时间戳
git -C <path> log origin/$DEFAULT --since="<start_date>T00:00:00" --format="%at|%aN|%ai|%s" | sort -n

# 按作者统计提交数
git -C <path> shortlog origin/$DEFAULT --since="<start_date>T00:00:00" -sn --no-merges

# 从提交消息中提取 PR 编号
git -C <path> log origin/$DEFAULT --since="<start_date>T00:00:00" --format="%s" | grep -oE '#[0-9]+' | sort -n | uniq
```

对于失败的仓库（路径已删除、网络错误）：跳过，并注明 “N repos could not be reached.”

### Global Step 4：计算全局 shipping streak

对每个仓库，获取提交日期（上限 365 天）：

```bash
git -C <path> log origin/$DEFAULT --since="365 days ago" --format="%ad" --date=format:"%Y-%m-%d" | sort -u
```

将所有仓库的日期取并集。从今天往回计数：有多少连续天数在任意仓库上至少有一次提交？如果连续记录达到 365 天上限，则显示为 “365+ days”。

### Global Step 5：计算上下文切换指标

根据步骤 3 收集到的提交时间戳，按日期分组。对每个日期，统计当天有提交的不同仓库数量。报告：
- 平均 repos/day
- 最大 repos/day
- 哪些天是聚焦的（1 个 repo）与碎片化的（3 个或以上 repo）

### Global Step 6：按工具分析生产力模式

根据 discovery JSON，分析工具使用模式：
- 哪个 AI 工具用于哪些仓库（独占还是共享）
- 每个工具的会话数
- 行为模式（例如，“Codex 专门用于 myapp，Claude Code 用于其他所有项目”）

### Global Step 7：聚合并生成叙述

输出结构应先给出**可分享的个人卡片**，然后再给出下面的完整
团队/项目拆解。个人卡片设计成适合截图分享
—— 将一个人愿意发到 X/Twitter 的所有内容浓缩在一个清晰区块中。

---

**可发推摘要**（第一行，放在最前面）：
```
Week of Mar 14: 5 projects, 138 commits, 250k LOC across 5 repos | 48 AI sessions | Streak: 52d 🔥
```

## 🚀 Your Week: [user name] — [date range]

这一部分是**可分享的个人卡片**。它只包含当前用户的
统计数据——没有团队数据，没有项目拆解。适合截图发布。

使用 `git config user.name` 中的用户身份来过滤所有按仓库拆分的 git 数据。
跨所有仓库聚合，计算个人总量。

将其渲染为一个视觉上整洁的单一块。只保留左边框，不要右边框（LLM
无法可靠地对齐右边框）。将仓库名补齐到最长仓库名长度，以便列对齐整洁。
绝不要截断项目名称。

```
╔═══════════════════════════════════════════════════════════════
║  [USER NAME] — Week of [date]
╠═══════════════════════════════════════════════════════════════
║
║  [N] commits across [M] projects
║  +[X]k LOC added · [Y]k LOC deleted · [Z]k net
║  [N] AI coding sessions (CC: X, Codex: Y, Gemini: Z)
║  [N]-day shipping streak 🔥
║
║  PROJECTS
║  ─────────────────────────────────────────────────────────
║  [repo_name_full]        [N] commits    +[X]k LOC    [solo/team]
║  [repo_name_full]        [N] commits    +[X]k LOC    [solo/team]
║  [repo_name_full]        [N] commits    +[X]k LOC    [solo/team]
║
║  SHIP OF THE WEEK
║  [PR title] — [LOC] lines across [N] files
║
║  TOP WORK
║  • [最大主题的一行描述]
║  • [第二个主题的一行描述]
║  • [第三个主题的一行描述]
║
║  Powered by gstack · github.com/garrytan/gstack
╚═══════════════════════════════════════════════════════════════
```

**个人卡片规则：**
- 只显示用户有提交的仓库。0 提交的仓库跳过。
- 按用户提交数降序排序仓库。
- **绝不要截断仓库名。** 使用完整仓库名（例如 `analyze_transcripts`
  而不是 `analyze_trans`）。将名称列补齐到最长仓库名，使所有列
  整齐对齐。如果名称很长，就加宽盒子，盒子宽度应适配内容。
- 对 LOC，使用 “k” 格式表示千位（例如 `+64.0k`，而不是 `+64010`）。
- 角色：如果用户是唯一贡献者，则为 “solo”；如果还有其他贡献者，则为 “team”。
- Ship of the Week：用户跨**所有**仓库的单个最高 LOC PR。
- Top Work：3 个项目符号，总结用户的主要工作主题，需根据
  提交消息推断。不要列单个提交，而要综合成主题。
  例如，“Built /retro global — cross-project retrospective with AI session discovery”
  而不是 “feat: gstack-global-discover” + “feat: /retro global template”。
- 卡片必须自包含。即使只看到这一个区块，也应能理解
  用户这一周做了什么，而不需要任何上下文。
- 此处**不要**包含团队成员、项目总量或上下文切换数据。

**个人 streak：** 使用用户自己跨所有仓库的提交（按
`--author` 过滤）来计算个人 streak，独立于团队 streak。

---

## Global Engineering Retro: [日期范围]

下面的所有内容都是完整分析——团队数据、项目拆解、模式。
这是接在可分享卡片之后的“深度分析”。

### All Projects Overview
| 指标 | 值 |
|--------|-------|
| 活跃项目数 | N |
| 总提交数（所有仓库、所有贡献者） | N |
| 总 LOC | +N / -N |
| AI 编码会话数 | N（CC: X, Codex: Y, Gemini: Z） |
| 活跃天数 | N |
| 全局 shipping streak（任意贡献者、任意仓库） | 连续 N 天 |
| 每日上下文切换 | 平均 N 次（最大：M） |

### Per-Project Breakdown
对每个仓库（按提交数降序排序）：
- 仓库名（及其占总提交的百分比）
- 提交数、LOC、合并 PR 数、主要贡献者
- 关键工作（根据提交消息推断）
- 按工具拆分的 AI 会话

**Your Contributions**（每个项目中的子小节）：
对每个项目，新增一个 “Your contributions” 区块，显示当前用户
在该仓库中的个人统计。使用 `git config user.name` 中的用户身份
进行过滤。包括：
- 你的提交数 / 总提交数（含百分比）
- 你的 LOC（+insertions / -deletions）
- 你的关键工作（仅根据**你的**提交消息推断）
- 你的提交类型混合（feat/fix/refactor/chore/docs 拆分）
- 你在这个仓库中的最大 ship（最高 LOC 的提交或 PR）

如果用户是唯一贡献者，则写：“Solo project — all commits are yours.”
如果用户在某个仓库本周期 0 提交（是一个他们未参与的团队项目），
则写：“No commits this period — [N] AI sessions only.” 并跳过拆解。

格式：
```
**Your contributions:** 47/244 commits (19%), +4.2k/-0.3k LOC
  Key work: Writer Chat, email blocking, security hardening
  Biggest ship: PR #605 — Writer Chat eats the admin bar (2,457 ins, 46 files)
  Mix: feat(3) fix(2) chore(1)
```

### Cross-Project Patterns
- 跨项目的时间分配（按你的提交而不是总量计算百分比分布）
- 聚合所有仓库后的生产力峰值时段
- 聚焦日 vs. 碎片化日
- 上下文切换趋势

### Tool Usage Analysis
按工具拆分并分析行为模式：
- Claude Code: N sessions across M repos — 观察到的模式
- Codex: N sessions across M repos — 观察到的模式
- Gemini: N sessions across M repos — 观察到的模式

### Ship of the Week（全局）
跨**所有**项目影响力最高的 PR。根据 LOC 和提交消息识别。

### 3 条跨项目洞察
全局视角揭示了哪些单仓库 retro 看不到的内容。

### 下周的 3 个习惯
结合完整的跨项目图景来给出。

---

### Global Step 8：加载历史并比较

```bash
ls -t ~/.gstack/retros/global-*.json 2>/dev/null | head -5
```

**只与 `window` 值相同的历史 retro 比较**（例如 7d 对 7d）。如果最新的历史 retro 使用的是不同窗口，则跳过比较，并注明：“Prior global retro used a different window — skipping comparison.”

如果存在匹配的历史 global retro，则使用 Read tool 加载。显示一个 **Trends vs Last Global Retro** 表，展示关键指标的变化量：总提交数、LOC、会话数、streak、每天上下文切换次数。

如果不存在任何历史 global retro，则追加：“First global retro recorded — run again next week to see trends.”

### Global Step 9：保存快照

```bash
mkdir -p ~/.gstack/retros
```

确定今天的下一个序号：
```bash
today=$(date +%Y-%m-%d)
existing=$(ls ~/.gstack/retros/global-${today}-*.json 2>/dev/null | wc -l | tr -d ' ')
next=$((existing + 1))
```

使用 Write tool 将 JSON 保存到 `~/.gstack/retros/global-${today}-${next}.json`：

```json
{
  "type": "global",
  "date": "2026-03-21",
  "window": "7d",
  "projects": [
    {
      "name": "gstack",
      "remote": "https://github.com/garrytan/gstack",
      "commits": 47,
      "insertions": 3200,
      "deletions": 800,
      "sessions": { "claude_code": 15, "codex": 3, "gemini": 0 }
    }
  ],
  "totals": {
    "commits": 182,
    "insertions": 15300,
    "deletions": 4200,
    "projects": 5,
    "active_days": 6,
    "sessions": { "claude_code": 48, "codex": 8, "gemini": 3 },
    "global_streak_days": 52,
    "avg_context_switches_per_day": 2.1
  },
  "tweetable": "Week of Mar 14: 5 projects, 182 commits, 15.3k LOC | CC: 48, Codex: 8, Gemini: 3 | Focus: gstack (58%) | Streak: 52d"
}
```

---

## Compare Mode

当用户运行 `/retro compare`（或 `/retro compare 14d`）时：

1. 使用午夜对齐的起始日期计算当前窗口的指标（逻辑与主 retro 相同，例如，如果今天是 2026-03-18 且窗口为 7d，则使用 `--since="2026-03-11T00:00:00"`）
2. 使用 `--since` 和 `--until` 计算紧邻其前的等长窗口，且也采用午夜对齐日期以避免重叠（例如，对于从 2026-03-11 开始的 7d 窗口，前一个窗口是 `--since="2026-03-04T00:00:00" --until="2026-03-11T00:00:00"`）
3. 显示一个并排对比表，带变化量和箭头
4. 写一段简短叙述，强调最大的提升和回退
5. 只将当前窗口快照保存到 `.context/retros/`（与普通 retro 运行相同）；**不要**持久化前一窗口的指标。

## 语气

- 鼓励但坦率，不要哄着说
- 具体且明确 —— 始终锚定真实提交/代码
- 跳过泛泛而谈的表扬（“great job!”）—— 要准确说出哪里做得好以及为什么
- 将改进表述为进阶，而不是批评
- **Praise 应该像你在 1:1 里真的会说的话** —— 具体、配得上、真诚
- **成长建议应像投资建议** —— “这值得你投入时间，因为……” 而不是 “你哪里没做好……”
- 绝不要把队友彼此做负面对比。每个人的小节都应独立成立。
- 总输出控制在约 3000-4500 词（为了容纳团队小节，可略长一些）
- 数据使用 markdown 表格和代码块，叙述使用 prose
- 直接输出到对话中 —— **不要**写入文件系统（除了 `.context/retros/` 的 JSON 快照）

## 重要规则

- 所有叙述性输出都直接发给用户。唯一会写入的文件只有 `.context/retros/` JSON 快照。
- 所有 git 查询都使用 `origin/<default>`（不要使用可能已过期的本地 main）
- 所有时间戳都以用户本地时区显示（不要覆盖 `TZ`）
- 如果窗口内 0 提交，要明确说明，并建议使用其他窗口
- LOC/hour 四舍五入到最接近的 50
- 将 merge commits 视为 PR 边界
- 不要读取 `CLAUDE.md` 或其他文档 —— 此 skill 是自包含的
- 首次运行（没有历史 retros）时，要自然地跳过比较小节
- **Global mode：** 不要求位于 git 仓库内部。快照保存到 `~/.gstack/retros/`（而不是 `.context/retros/`）。优雅地跳过未安装的 AI 工具。只与使用相同 `window` 值的历史 global retros 比较。如果 streak 达到 365d 上限，则显示为 “365+ days”。