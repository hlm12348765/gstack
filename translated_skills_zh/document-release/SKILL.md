---
name: document-release
version: 1.0.0
description: |
  发布后的文档更新。读取所有项目文档，交叉比对
  diff，更新 README/ARCHITECTURE/CONTRIBUTING/CLAUDE.md 以匹配已发布内容，
  润色 CHANGELOG 的表达风格，清理 TODOS，并可选择性地提升 VERSION。在被要求
  “update the docs”、“sync documentation”或“post-ship docs”时使用。
  在 PR 合并后或代码发布后主动建议使用。
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - AskUserQuestion
---
<!-- AUTO-GENERATED from SKILL.md.tmpl — 不要直接编辑 -->
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
echo '{"skill":"document-release","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","repo":"'$(basename "$(git rev-parse --show-toplevel 2>/dev/null)" 2>/dev/null || echo "unknown")'"}'  >> ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true
for _PF in ~/.gstack/analytics/.pending-*; do [ -f "$_PF" ] && ~/.claude/skills/gstack/bin/gstack-telemetry-log --event-type skill_run --skill _pending_finalize --outcome unknown --session-id "$_SESSION_ID" 2>/dev/null || true; break; done
```

如果 `PROACTIVE` 是 `"false"`，不要主动建议 gstack skills，只有在用户明确提出时才调用。
用户已选择退出主动建议。

如果输出显示 `UPGRADE_AVAILABLE <old> <new>`：读取 `~/.claude/skills/gstack/gstack-upgrade/SKILL.md` 并遵循“Inline upgrade flow”（如果已配置则自动升级，否则使用含 4 个选项的 AskUserQuestion，如果用户拒绝则写入 snooze 状态）。如果是 `JUST_UPGRADED <from> <to>`：告诉用户“Running gstack v{to} (just updated!)”，然后继续。

如果 `LAKE_INTRO` 是 `no`：继续之前，先介绍 Completeness Principle。
告诉用户：“gstack 遵循 **Boil the Lake** 原则，也就是当 AI 让边际成本几乎为零时，始终把事情完整做完。阅读更多：https://garryslist.org/posts/boil-the-ocean”
然后询问是否要在默认浏览器中打开这篇文章：

```bash
open https://garryslist.org/posts/boil-the-ocean
touch ~/.gstack/.completeness-intro-seen
```

只有在用户同意时才运行 `open`。始终运行 `touch` 以标记为已读。这个流程只会发生一次。

如果 `TEL_PROMPTED` 是 `no` 且 `LAKE_INTRO` 是 `yes`：在处理完 lake 介绍后，
向用户询问 telemetry。使用 AskUserQuestion：

> 帮助 gstack 变得更好！社区模式会共享使用数据（你使用了哪些技能、耗时多久、
> 崩溃信息），并附带稳定的设备 ID，这样我们就能跟踪趋势并更快修复问题。
> 永远不会发送代码、文件路径或仓库名称。
> 可随时使用 `gstack-config set telemetry off` 修改。

选项：
- A) 帮助 gstack 变得更好！（推荐）
- B) 不，谢谢

如果选 A：运行 `~/.claude/skills/gstack/bin/gstack-config set telemetry community`

如果选 B：继续使用 AskUserQuestion 追问：

> 那匿名模式怎么样？我们只会知道*有人*使用了 gstack，不会有唯一 ID，
> 也无法关联不同会话。只有一个计数器，用来帮助我们了解是否真的有人在使用。

选项：
- A) 可以，匿名模式没问题
- B) 不，谢谢，完全关闭

如果 B→A：运行 `~/.claude/skills/gstack/bin/gstack-config set telemetry anonymous`
如果 B→B：运行 `~/.claude/skills/gstack/bin/gstack-config set telemetry off`

始终运行：
```bash
touch ~/.gstack/.telemetry-prompted
```

这个流程只会发生一次。如果 `TEL_PROMPTED` 是 `yes`，则完全跳过。

## AskUserQuestion 格式

**每次调用 AskUserQuestion 时都必须遵循以下结构：**
1. **重新锚定：** 说明项目、当前分支（使用前言中打印出的 `_BRANCH` 值，而不是会话历史或 gitStatus 中的任何分支），以及当前计划/任务。（1-2 句话）
2. **简化说明：** 用一个聪明的 16 岁学生也能理解的自然语言解释问题。不要出现原始函数名、内部术语或实现细节。使用具体示例和类比。说明“它是做什么的”，而不是“它叫什么”。
3. **给出建议：** `RECOMMENDATION: Choose [X] because [one-line reason]`，始终优先推荐完整选项而不是捷径（见 Completeness Principle）。为每个选项都附上 `Completeness: X/10`。参考标准：10 = 完整实现（覆盖所有边界情况、完全覆盖），7 = 覆盖主路径但跳过部分边界情况，3 = 推迟大量工作的捷径。如果两个选项都在 8 以上，选择更高的；如果其中一个 ≤5，要明确标出。
4. **选项：** 使用字母编号：`A) ... B) ... C) ...`。当选项涉及工作量时，同时展示两种尺度：`(human: ~X / CC: ~Y)`

假设用户已经 20 分钟没看这个窗口了，而且也没有打开代码。如果你需要读源码才能理解你自己的说明，那它就太复杂了。

各技能的说明可以在这个基础规则之上附加额外格式要求。

## Completeness Principle — Boil the Lake

AI 辅助编码让“完整性”的边际成本几乎为零。当你给出选项时：

- 如果选项 A 是完整实现（完全一致、覆盖所有边界情况、100% 覆盖），而选项 B 是节省少量工作量的捷径，**始终推荐 A**。在 CC+gstack 下，80 行和 150 行之间的差距没有意义。当“完整”只多花几分钟时，“差不多够用”就是错误直觉。
- **Lake 与 ocean：** “lake” 是可以煮沸的，也就是单个模块 100% 测试覆盖、完整功能实现、处理所有边界情况、覆盖完整错误路径。“ocean” 则不是，例如从零重写整个系统、给你无法控制的依赖添加功能、跨多个季度的平台迁移。推荐煮沸 lakes。把 oceans 标记为超出范围。
- **估算工作量时**，始终同时展示两种尺度：人工团队时间和 CC+gstack 时间。压缩比会随任务类型变化，参考如下：

| Task type | Human team | CC+gstack | Compression |
|-----------|-----------|-----------|-------------|
| Boilerplate / scaffolding | 2 days | 15 min | ~100x |
| Test writing | 1 day | 15 min | ~50x |
| Feature implementation | 1 week | 30 min | ~30x |
| Bug fix + regression test | 4 hours | 15 min | ~20x |
| Architecture / design | 2 days | 4 hours | ~5x |
| Research / exploration | 1 day | 3 hours | ~3x |

- 这个原则适用于测试覆盖、错误处理、文档、边界情况和功能完整性。不要为了“节省时间”而跳过最后 10%，因为有了 AI，这 10% 的成本只需要几秒钟。

**反模式——不要这样做：**
- BAD: "Choose B — it covers 90% of the value with less code."（如果 A 只多 70 行，就选 A。）
- BAD: "We can skip edge case handling to save time."（有了 CC，处理边界情况只需几分钟。）
- BAD: "Let's defer test coverage to a follow-up PR."（测试是最便宜、最值得煮沸的 lake。）
- BAD: 只引用人工团队工作量："This would take 2 weeks."（应该说："2 weeks human / ~1 hour CC."）

## Repo Ownership Mode — See Something, Say Something

前言中的 `REPO_MODE` 告诉你这个仓库里的问题由谁负责：

- **`solo`** — 一个人完成了 80% 以上的工作。他拥有所有内容。当你注意到当前分支改动之外的问题时（测试失败、弃用警告、安全通告、lint 错误、死代码、环境问题），**要主动调查并提出修复**。单人开发者是唯一会修它的人。默认直接行动。
- **`collaborative`** — 有多位活跃贡献者。当你注意到当前分支改动之外的问题时，**通过 AskUserQuestion 提醒出来**，因为那可能是其他人的职责。默认先询问，而不是直接修复。
- **`unknown`** — 按 collaborative 处理（更安全的默认值，先问再修）。

**See Something, Say Something：** 在任何工作流步骤中，只要你注意到看起来不对的地方，而不仅仅是测试失败，都要简短指出。用一句话说明：你注意到了什么，以及它会带来什么影响。在 solo 模式下，再补一句“Want me to fix it?”。在 collaborative 模式下，只提示然后继续。

不要让你注意到的问题悄悄溜走。这个机制的重点就是主动沟通。

## Search Before Building

在构建基础设施、不熟悉的模式，或任何运行时可能已有内建能力的东西之前，**先搜索。** 完整理念请阅读 `~/.claude/skills/gstack/ETHOS.md`。

**三层知识：**
- **Layer 1**（成熟可靠，已进入分发版）。不要重复造轮子。但检查的成本几乎为零，而偶尔对“成熟做法”提出质疑，正是产生卓见的地方。
- **Layer 2**（新且流行，应搜索这些）。但要审慎：人类容易陷入狂热。搜索结果只是你思考的输入，不是答案。
- **Layer 3**（第一性原理，高于一切）。从对具体问题的推理中得出的原创观察。它最有价值。

**Eureka 时刻：** 当第一性原理推理揭示传统观念是错误的时，要明确指出：
"EUREKA: Everyone does X because [assumption]. But [evidence] shows this is wrong. Y is better because [reasoning]."

记录 eureka 时刻：
```bash
jq -n --arg ts "$(date -u +%Y-%m-%dT%H:%M:%SZ)" --arg skill "SKILL_NAME" --arg branch "$(git branch --show-current 2>/dev/null)" --arg insight "ONE_LINE_SUMMARY" '{ts:$ts,skill:$skill,branch:$branch,insight:$insight}' >> ~/.gstack/analytics/eureka.jsonl 2>/dev/null || true
```

替换 `SKILL_NAME` 和 `ONE_LINE_SUMMARY`。内联运行，不要中断工作流。

**WebSearch 兜底：** 如果 WebSearch 不可用，跳过搜索步骤，并说明：“Search unavailable — proceeding with in-distribution knowledge only.”

## Contributor Mode

如果 `_CONTRIB` 是 `true`：你处于 **contributor mode**。你既是 gstack 用户，也在帮助它变得更好。

**在每个主要工作流步骤结束时**（不是每条命令之后），回顾一下你使用过的 gstack 工具。给这次体验打 0 到 10 分。如果不是 10 分，想想原因。如果存在明显、可执行的 bug，或者某个由 gstack 代码或 skill markdown 本可以做得更好的、有见地且值得记录的问题，就提交一份 field report。也许我们的贡献者会帮助我们做得更好！

**评分标准——这就是门槛：** 例如，`$B js "await fetch(...)"` 过去会报错 `SyntaxError: await is only valid in async functions`，因为 gstack 没有把表达式包进 async 上下文。问题虽小，但这个输入是合理的，而 gstack 本应处理好它，这类问题就值得提交。比这更不重要的情况，忽略即可。

**不值得提交：** 用户应用本身的 bug、访问用户 URL 时的网络错误、用户站点上的认证失败、用户自己的 JS 逻辑错误。

**提交方式：** 写入 `~/.gstack/contributor-logs/{slug}.md`，并包含**下面所有章节**（不要截断，必须包含直到 Date/Version 页脚的全部内容）：

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
{paste the actual error or unexpected output here}
```

## What would make this a 10
{one sentence: what gstack should have done differently}

**Date:** {YYYY-MM-DD} | **Version:** {gstack version} | **Skill:** /{skill}
```

Slug：小写、连字符、最长 60 个字符（例如 `browse-js-no-await`）。如果文件已存在则跳过。每个会话最多 3 份报告。内联提交后继续，不要中断工作流。告诉用户：“Filed gstack field report: {title}”

## Completion Status Protocol

完成技能工作流时，使用以下状态之一进行报告：
- **DONE** — 所有步骤均已成功完成。每项结论都提供了证据。
- **DONE_WITH_CONCERNS** — 已完成，但存在用户应知晓的问题。列出每个问题。
- **BLOCKED** — 无法继续。说明阻塞点以及已尝试的内容。
- **NEEDS_CONTEXT** — 缺少继续所需的信息。明确说明你需要什么。

### Escalation

你随时都可以停下来并说“这对我来说太难了”或“我对这个结果没有信心”。

糟糕的工作比不做更糟。升级处理不会让你受罚。
- 如果你已经尝试一个任务 3 次仍未成功，停止并升级处理。
- 如果你对某个安全敏感的改动不确定，停止并升级处理。
- 如果工作范围超出了你能验证的程度，停止并升级处理。

升级格式：
```
STATUS: BLOCKED | NEEDS_CONTEXT
REASON: [1-2 sentences]
ATTEMPTED: [what you tried]
RECOMMENDATION: [what the user should do next]
```

## Telemetry（最后运行）

在技能工作流完成后（成功、报错或中止），记录 telemetry 事件。
从本文件 YAML frontmatter 中的 `name:` 字段确定技能名称。
根据工作流结果确定 outcome（正常完成为 success，失败为 error，
用户中断为 abort）。

**PLAN MODE 例外——始终运行：** 这个命令会把 telemetry 写入
`~/.gstack/analytics/`（用户配置目录，而不是项目文件）。技能前言
已经写入同一个目录，这里采用的是同样的模式。
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
如果无法确定 outcome，则使用 `"unknown"`。该命令在后台运行，
永远不会阻塞用户。

## Plan Status Footer

当你处于 plan mode 并且即将调用 ExitPlanMode 时：

1. 检查计划文件是否已经包含 `## GSTACK REVIEW REPORT` 章节。
2. 如果**已经有**，则跳过（某个 review skill 已经写入了更丰富的报告）。
3. 如果**没有**，则运行以下命令：

\`\`\`bash
~/.claude/skills/gstack/bin/gstack-review-read
\`\`\`

然后在计划文件末尾写入一个 `## GSTACK REVIEW REPORT` 章节：

- 如果输出中包含 review 条目（`---CONFIG---` 之前的 JSONL 行）：按
  review skills 使用的相同格式，输出标准报告表，包括 runs/status/findings。
- 如果输出是 `NO_REVIEWS` 或为空：写入以下占位表：

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

**PLAN MODE 例外——始终运行：** 这会写入计划文件，而计划文件是 plan mode 下
唯一允许编辑的文件。计划文件中的 review 报告是计划实时状态的一部分。

## 第 0 步：检测基准分支

确定这个 PR 的目标分支。后续所有步骤中都将其作为“基准分支”使用。

1. 检查当前分支是否已经有 PR：
   `gh pr view --json baseRefName -q .baseRefName`
   如果此命令成功，使用输出的分支名作为基准分支。

2. 如果还没有 PR（命令失败），检测仓库默认分支：
   `gh repo view --json defaultBranchRef -q .defaultBranchRef.name`

3. 如果两个命令都失败，则回退到 `main`。

打印检测到的基准分支名称。在后续所有 `git diff`、`git log`、
`git fetch`、`git merge` 和 `gh pr create` 命令中，把说明里出现的“the base branch”
替换为检测到的分支名。

---

# Document Release：发布后的文档更新

你正在运行 `/document-release` 工作流。这个流程运行在 **`/ship` 之后**
（代码已提交，PR 已存在或即将创建）但 **在 PR 合并之前**。你的任务是确保项目中的每个文档文件
都是准确、最新的，并且采用友好、面向用户的表达风格。

这个流程大部分是自动化的。明显的事实性更新请直接处理。只有在遇到风险较高或主观性较强的决定时才停下来询问。

**只在以下情况停下：**
- 有风险或存疑的文档变更（叙述、理念、安全、删除、大规模重写）
- VERSION 提升决策（如果尚未提升）
- 需要新增的 TODOS 条目
- 文档之间存在叙述性矛盾（而非事实性矛盾）

**以下情况不要停下：**
- 能从 diff 明确看出的事实性修正
- 向表格/列表中添加条目
- 更新路径、数量、版本号
- 修复过期的交叉引用
- CHANGELOG 表达风格润色（小幅措辞调整）
- 将 TODOS 标记为已完成
- 文档之间的事实性不一致（例如版本号不一致）

**绝对不要做：**
- 覆盖、替换或重新生成 CHANGELOG 条目，只能润色措辞，保留全部内容
- 未经询问就提升 VERSION，版本变更始终使用 AskUserQuestion
- 在 CHANGELOG.md 上使用 `Write` 工具，始终使用带精确 `old_string` 匹配的 `Edit`

---

## 第 1 步：预检与 Diff 分析

1. 检查当前分支。如果当前就在基准分支上，**中止**：“You're on the base branch. Run from a feature branch.”

2. 收集变更上下文：

```bash
git diff <base>...HEAD --stat
```

```bash
git log <base>..HEAD --oneline
```

```bash
git diff <base>...HEAD --name-only
```

3. 发现仓库中的所有文档文件：

```bash
find . -maxdepth 2 -name "*.md" -not -path "./.git/*" -not -path "./node_modules/*" -not -path "./.gstack/*" -not -path "./.context/*" | sort
```

4. 将变更归类为与文档相关的类别：
   - **新功能** — 新文件、新命令、新技能、新能力
   - **行为变更** — 修改过的服务、更新后的 API、配置变更
   - **已移除功能** — 删除的文件、移除的命令
   - **基础设施** — 构建系统、测试基础设施、CI

5. 输出一段简短摘要：“Analyzing N files changed across M commits. Found K documentation files to review.”

---

## 第 2 步：逐文件文档审计

读取每个文档文件，并与 diff 交叉比对。使用以下通用启发式规则
（根据你所在的项目灵活调整，这些规则不是 gstack 专用的）：

**README.md：**
- 它是否描述了 diff 中可见的全部功能和能力？
- 安装/设置说明是否与这些变更保持一致？
- 示例、演示和用法描述是否仍然有效？
- 故障排查步骤是否仍然准确？

**ARCHITECTURE.md：**
- ASCII 图和组件说明是否与当前代码一致？
- 设计决策和“为什么这样做”的解释是否仍然准确？
- 保守处理，只有在 diff 明确否定现有内容时才更新。架构文档
  描述的是不太可能频繁变化的内容。

**CONTRIBUTING.md — 新贡献者冒烟测试：**
- 以全新贡献者的身份逐步检查设置说明。
- 列出的命令是否准确？每一步是否都能成功？
- 测试分层说明是否与当前测试基础设施一致？
- 工作流说明（开发环境设置、contributor mode 等）是否仍然最新？
- 标记任何会让首次贡献者失败或困惑的地方。

**CLAUDE.md / 项目说明：**
- 项目结构章节是否与实际文件树一致？
- 列出的命令和脚本是否准确？
- 构建/测试说明是否与 package.json（或同类文件）中的内容一致？

**任何其他 .md 文件：**
- 阅读文件，判断它的目的和目标读者。
- 与 diff 交叉比对，检查它是否与文件中的任何说法相矛盾。

对于每个文件，将所需更新归类为：

- **Auto-update** — diff 明确支持的事实性修正：向
  表格中添加条目、更新文件路径、修正计数、更新项目结构树。
- **Ask user** — 叙述性变更、删除章节、安全模型变更、大规模重写
  （单个章节超过约 10 行）、相关性不明确、添加全新章节。

---

## 第 3 步：应用自动更新

使用 Edit 工具直接完成所有明确、事实性的更新。

对于每个被修改的文件，输出一行摘要，说明**具体改了什么**，而不只是
“Updated README.md”，而应类似 “README.md: added /new-skill to skills table, updated skill count
from 9 to 10.”

**绝不自动更新：**
- README 引言或项目定位
- ARCHITECTURE 中的理念或设计依据
- 安全模型描述
- 不要从任何文档中移除整个章节

---

## 第 4 步：询问高风险/存疑的变更

对于第 2 步中识别出的每一项高风险或存疑更新，使用 AskUserQuestion，并包含：
- 上下文：项目名称、分支、正在审查哪个文档文件、当前审查内容
- 具体的文档决策点
- `RECOMMENDATION: Choose [X] because [one-line reason]`
- 选项中包含 C) Skip — 保持原样

每次获得答复后，立即应用获准的更改。

---

## 第 5 步：CHANGELOG 表达风格润色

**关键——绝不能覆盖 CHANGELOG 条目。**

这一步是润色表达风格，不是重写、替换或重新生成 CHANGELOG 内容。

曾经真实发生过一次事故：某个 agent 在本该保留现有 CHANGELOG 条目的情况下替换了它们。
这个 skill **绝对不能**这样做。

**规则：**
1. 先完整读取整个 CHANGELOG.md。理解其中已有内容。
2. 只能修改现有条目内部的措辞。绝不能删除、重排或替换条目。
3. 绝不能从头重新生成 CHANGELOG 条目。该条目由 `/ship` 根据
   实际 diff 和提交历史写入，它才是真实来源。你是在润色表达，不是在重写历史。
4. 如果某个条目看起来有误或不完整，使用 AskUserQuestion，不要静默修正。
5. 使用带精确 `old_string` 匹配的 Edit 工具，绝不能用 Write 覆盖 CHANGELOG.md。

**如果这个分支没有修改 CHANGELOG：** 跳过此步骤。

**如果这个分支修改了 CHANGELOG，** 按表达风格审查条目：

- **Sell test：** 用户读完每条 bullet 后会不会觉得“哦，这不错，我想试试”？如果不会，
  那就重写措辞（不是重写内容）。
- 先写出用户现在**能做什么**，而不是实现细节。
- 用 “You can now...” 而不是 “Refactored the...”
- 标记并重写任何读起来像提交说明的条目。
- 内部/贡献者变更应放在单独的 “### For contributors” 小节中。
- 小幅表达风格调整可自动修正。若重写会改变原意，则使用 AskUserQuestion。

---

## 第 6 步：跨文档一致性与可发现性检查

在逐个审查完每个文件之后，进行一次跨文档一致性检查：

1. README 中的功能/能力列表是否与 CLAUDE.md（或项目说明）描述的一致？
2. ARCHITECTURE 中的组件列表是否与 CONTRIBUTING 中的项目结构描述一致？
3. CHANGELOG 中的最新版本是否与 VERSION 文件一致？
4. **可发现性：** 每个文档文件是否都可以从 README.md 或 CLAUDE.md 进入？如果
   存在 ARCHITECTURE.md，但 README 和 CLAUDE.md 都没有链接到它，就要标记出来。每个文档
   都应能从这两个入口文件中的一个被发现。
5. 标记文档之间的任何矛盾。对明确的事实性不一致自动修复（例如
   版本号不匹配）。对叙述性矛盾使用 AskUserQuestion。

---

## 第 7 步：清理 TODOS.md

这是对 `/ship` 第 5.5 步的第二轮补充检查。读取 `review/TODOS-format.md`（如果
存在），将其视为 TODO 条目格式的规范来源。

如果不存在 TODOS.md，跳过此步骤。

1. **已完成但尚未标记的条目：** 将 diff 与未完成的 TODO 条目交叉比对。如果某个
   TODO 明确已被本分支改动完成，则将它移动到 Completed 区域，
   并标注 `**Completed:** vX.Y.Z.W (YYYY-MM-DD)`。要保守，只有在 diff 中有明确
   证据时才标记完成。

2. **需要更新描述的条目：** 如果某个 TODO 引用了已被
   大幅修改的文件或组件，那么它的描述可能已经过时。使用 AskUserQuestion 确认
   这个 TODO 应该更新、完成，还是保持原样。

3. **新的延后工作：** 检查 diff 中的 `TODO`、`FIXME`、`HACK` 和 `XXX` 注释。对于
   每一个代表有意义的延后工作（而不是琐碎的行内注释）的项，使用
   AskUserQuestion 询问是否应将其记录到 TODOS.md 中。

---

## 第 8 步：VERSION 提升询问

**关键——未询问前绝不要提升 VERSION。**

1. **如果 VERSION 不存在：** 静默跳过。

2. 检查当前分支是否已经修改过 VERSION：

```bash
git diff <base>...HEAD -- VERSION
```

3. **如果 VERSION 尚未提升：** 使用 AskUserQuestion：
   - RECOMMENDATION: Choose C (Skip) because docs-only changes rarely warrant a version bump
   - A) 提升 PATCH（X.Y.Z+1）— 如果文档变更与代码变更一起发布
   - B) 提升 MINOR（X.Y+1.0）— 如果这是一次重要的独立发布
   - C) Skip — 不需要提升版本

4. **如果 VERSION 已经提升：** 不要静默跳过。相反，检查这次提升
   是否仍然覆盖了当前分支变更的完整范围：

   a. 读取当前 VERSION 对应的 CHANGELOG 条目。它描述了哪些功能？
   b. 读取完整 diff（`git diff <base>...HEAD --stat` 和 `git diff <base>...HEAD --name-only`）。
      是否存在未在当前版本 CHANGELOG 条目中提到的重要变更（新功能、新技能、新命令、重大重构）？
   c. **如果 CHANGELOG 条目覆盖了全部内容：** 跳过，并输出 “VERSION: Already bumped to
      vX.Y.Z, covers all changes.”
   d. **如果存在重要但未覆盖的变更：** 使用 AskUserQuestion，说明
      当前版本覆盖了什么，以及新增了什么，然后询问：
      - RECOMMENDATION: Choose A because the new changes warrant their own version
      - A) 提升到下一个 patch（X.Y.Z+1）— 让这些新变更拥有自己的版本
      - B) 保持当前版本 — 将新变更加入现有 CHANGELOG 条目
      - C) Skip — 保持当前版本不变，稍后处理

   关键洞见是：如果 VERSION 提升原本是为“feature A”设置的，那么在“feature B”
   足够重要、理应拥有独立版本条目的情况下，不应让它被静默吸收到同一个版本里。

---

## 第 9 步：提交与输出

**先做空变更检查：** 运行 `git status`（绝不要使用 `-uall`）。如果之前各步骤中
没有任何文档文件被修改，输出 “All documentation is up to date.”，然后直接退出，不要提交。

**提交：**

1. 按文件名暂存已修改的文档文件（绝不要使用 `git add -A` 或 `git add .`）。
2. 创建单个提交：

```bash
git commit -m "$(cat <<'EOF'
docs: update project documentation for vX.Y.Z.W

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
EOF
)"
```

3. 推送到当前分支：

```bash
git push
```

**PR 正文更新（幂等、竞态安全）：**

1. 将现有 PR 正文读入一个带 PID 唯一性的临时文件：

```bash
gh pr view --json body -q .body > /tmp/gstack-pr-body-$$.md
```

2. 如果临时文件中已经包含 `## Documentation` 小节，则替换该小节为
   更新后的内容。如果不包含，则在末尾追加一个 `## Documentation` 小节。

3. Documentation 小节应包含一个 **文档 diff 预览**。对于每个已修改文件，
   说明具体改了什么（例如：“README.md: added /document-release to skills
   table, updated skill count from 9 to 10”）。

4. 将更新后的正文写回：

```bash
gh pr edit --body-file /tmp/gstack-pr-body-$$.md
```

5. 清理临时文件：

```bash
rm -f /tmp/gstack-pr-body-$$.md
```

6. 如果 `gh pr view` 失败（不存在 PR）：跳过，并输出 “No PR found — skipping body update.”
7. 如果 `gh pr edit` 失败：警告 “Could not update PR body — documentation changes are in the
   commit.”，然后继续。

**结构化文档健康摘要（最终输出）：**

输出一个便于扫读的摘要，展示每个文档文件的状态：

```
Documentation health:
  README.md       [status] ([details])
  ARCHITECTURE.md [status] ([details])
  CONTRIBUTING.md [status] ([details])
  CHANGELOG.md    [status] ([details])
  TODOS.md        [status] ([details])
  VERSION         [status] ([details])
```

其中 status 必须是以下之一：
- Updated — 附带修改内容描述
- Current — 不需要修改
- Voice polished — 仅调整了措辞
- Not bumped — 用户选择跳过
- Already bumped — 版本已由 /ship 设置
- Skipped — 文件不存在

---

## 重要规则

- **先读后改。** 修改文件之前始终先完整读取文件内容。
- **绝不覆盖 CHANGELOG。** 只能润色措辞。绝不能删除、替换或重新生成条目。
- **绝不静默提升 VERSION。** 必须始终询问。即使已经提升过，也要检查它是否覆盖了变更的完整范围。
- **明确说明改了什么。** 每次编辑都要有一行摘要。
- **使用通用启发式规则，而不是项目专用规则。** 审计检查应适用于任何仓库。
- **可发现性很重要。** 每个文档文件都应能从 README 或 CLAUDE.md 到达。
- **表达风格：友好、面向用户、不过分晦涩。** 像是在向一个
  没看过代码的聪明人解释问题。