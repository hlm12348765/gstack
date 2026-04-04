---
name: ship
version: 1.0.0
description: |
  Ship 工作流：检测并合并基线分支，运行测试，审查 diff，提升 VERSION，更新 CHANGELOG，提交，推送，创建 PR。当用户要求“ship”、“deploy”、“push to main”、“create a PR”或“merge and push”时使用。
  当用户表示代码已准备就绪或询问部署相关问题时，主动建议使用。
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - Agent
  - AskUserQuestion
  - WebSearch
---
<!-- AUTO-GENERATED from SKILL.md.tmpl — do not edit directly -->
<!-- 从 SKILL.md.tmpl 自动生成 —— 不要直接编辑 -->
<!-- Regenerate: bun run gen:skill-docs -->
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
echo '{"skill":"ship","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","repo":"'$(basename "$(git rev-parse --show-toplevel 2>/dev/null)" 2>/dev/null || echo "unknown")'"}'  >> ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true
for _PF in ~/.gstack/analytics/.pending-*; do [ -f "$_PF" ] && ~/.claude/skills/gstack/bin/gstack-telemetry-log --event-type skill_run --skill _pending_finalize --outcome unknown --session-id "$_SESSION_ID" 2>/dev/null || true; break; done
```

如果 `PROACTIVE` 是 `"false"`，不要主动建议 gstack skills，只有在用户明确要求时才调用它们。用户已选择退出主动建议。

如果输出显示 `UPGRADE_AVAILABLE <old> <new>`：读取 `~/.claude/skills/gstack/gstack-upgrade/SKILL.md` 并遵循 “Inline upgrade flow”（如果已配置则自动升级，否则使用 AskUserQuestion 提供 4 个选项；如果用户拒绝，则写入 snooze 状态）。如果显示 `JUST_UPGRADED <from> <to>`：告诉用户 `"Running gstack v{to} (just updated!)"`，然后继续。

如果 `LAKE_INTRO` 是 `no`：继续之前，先介绍 Completeness Principle。告诉用户：`"gstack follows the **Boil the Lake** principle — always do the complete thing when AI makes the marginal cost near-zero. Read more: https://garryslist.org/posts/boil-the-ocean"`
然后询问是否要在他们的默认浏览器中打开这篇文章：

```bash
open https://garryslist.org/posts/boil-the-ocean
touch ~/.gstack/.completeness-intro-seen
```

只有在用户表示同意时才运行 `open`。始终运行 `touch` 以标记为已查看。这只会发生一次。

如果 `TEL_PROMPTED` 是 `no` 且 `LAKE_INTRO` 是 `yes`：在 lake intro 处理完之后，向用户询问 telemetry。使用 AskUserQuestion：

> 帮助 gstack 变得更好！社区模式会共享使用数据（你使用了哪些 skills、耗时多久、崩溃信息），并附带稳定的设备 ID，这样我们可以跟踪趋势并更快修复 bug。
> 绝不会发送代码、文件路径或仓库名称。
> 可随时使用 `gstack-config set telemetry off` 修改。

选项：
- A) 帮助 gstack 变得更好！（推荐）
- B) 不，谢谢

如果选 A：运行 `~/.claude/skills/gstack/bin/gstack-config set telemetry community`

如果选 B：继续使用 AskUserQuestion 追问：

> 那匿名模式怎么样？我们只会知道*有人*使用了 gstack，没有唯一 ID，
> 也无法关联不同会话。只是一个计数器，用来帮助我们知道是否真的有人在使用。

选项：
- A) 可以，匿名模式没问题
- B) 不，谢谢，完全关闭

如果 B→A：运行 `~/.claude/skills/gstack/bin/gstack-config set telemetry anonymous`
如果 B→B：运行 `~/.claude/skills/gstack/bin/gstack-config set telemetry off`

始终运行：
```bash
touch ~/.gstack/.telemetry-prompted
```

这只会发生一次。如果 `TEL_PROMPTED` 是 `yes`，则完全跳过此步骤。

## AskUserQuestion 格式

**每次调用 AskUserQuestion 都必须遵循此结构：**
1. **重新锚定：** 说明项目、当前分支（使用前言打印出的 `_BRANCH` 值，而不是会话历史或 gitStatus 中的任何分支），以及当前计划/任务。（1-2 句）
2. **简化：** 用一个聪明的 16 岁青少年也能理解的自然语言解释问题。不要出现原始函数名、内部术语或实现细节。使用具体示例和类比。描述它“做什么”，而不是“叫什么”。
3. **建议：** `RECOMMENDATION: Choose [X] because [one-line reason]` —— 始终优先推荐完整方案而不是捷径（见 Completeness Principle）。为每个选项都包含 `Completeness: X/10`。校准标准：10 = 完整实现（所有边界情况，全面覆盖），7 = 覆盖主要路径但跳过一些边缘情况，3 = 推迟大量工作的捷径。如果两个选项都 ≥8，选更高的；如果有一个 ≤5，要明确标出。
4. **选项：** 使用字母编号：`A) ... B) ... C) ...` —— 当某个选项涉及工作量时，同时显示两个尺度：`(human: ~X / CC: ~Y)`

假设用户已经 20 分钟没看这个窗口，而且也没有打开代码。如果你需要再去读源代码才能解释清楚自己的说明，那就说明这段说明太复杂了。

每个 skill 的专属说明可以在此基础上添加额外格式规则。

## Completeness Principle — Boil the Lake

AI 辅助编码让“做完整”的边际成本接近于零。当你展示选项时：

- 如果选项 A 是完整实现（完整一致性、所有边界情况、100% 覆盖），而选项 B 是只节省少量工作量的捷径，**始终推荐 A**。对 CC+gstack 来说，80 行和 150 行之间的差别毫无意义。当“完整”只多花几分钟时，“差不多就行”就是错误直觉。
- **Lake vs. ocean：** “lake” 是可以煮沸的，例如某个模块的 100% 测试覆盖、完整功能实现、处理所有边界情况、完整错误路径。“ocean” 则不是，例如从零重写整个系统、给无法控制的依赖增加功能、持续多个季度的平台迁移。推荐煮沸 lake。对于 ocean，要明确指出超出范围。
- **估算工作量时，**始终展示两个尺度：人工团队时间和 CC+gstack 时间。压缩比会因任务类型而异，参考如下：

| Task type | Human team | CC+gstack | Compression |
|-----------|-----------|-----------|-------------|
| Boilerplate / scaffolding | 2 days | 15 min | ~100x |
| Test writing | 1 day | 15 min | ~50x |
| Feature implementation | 1 week | 30 min | ~30x |
| Bug fix + regression test | 4 hours | 15 min | ~20x |
| Architecture / design | 2 days | 4 hours | ~5x |
| Research / exploration | 1 day | 3 hours | ~3x |

- 这个原则适用于测试覆盖、错误处理、文档、边界情况和功能完整性。不要为了“省时间”跳过最后 10% 的工作；在 AI 的帮助下，这 10% 只需要几秒钟。

**反模式 —— 不要这样做：**
- 错误：`"Choose B — it covers 90% of the value with less code."`（如果 A 只多 70 行，就选 A。）
- 错误：`"We can skip edge case handling to save time."`（在 CC 的帮助下，处理边界情况只需要几分钟。）
- 错误：`"Let's defer test coverage to a follow-up PR."`（测试是最便宜、最该煮沸的 lake。）
- 错误：只引用人工团队工作量：`"This would take 2 weeks."`（应该说：`"2 weeks human / ~1 hour CC."`）

## Repo Ownership Mode — See Something, Say Something

前言中的 `REPO_MODE` 会告诉你这个仓库中的问题归谁负责：

- **`solo`** —— 一个人完成 80% 以上的工作。他负责所有事情。当你注意到当前分支变更之外的问题（测试失败、弃用警告、安全告警、lint 错误、死代码、环境问题）时，**主动调查并主动提出修复**。这个单人开发者是唯一会修复它的人。默认直接行动。
- **`collaborative`** —— 有多个活跃贡献者。当你注意到当前分支变更之外的问题时，**通过 AskUserQuestion 提示出来**，因为这可能是别人的职责。默认先询问，而不是直接修复。
- **`unknown`** —— 按 collaborative 处理（更安全的默认行为：先询问再修复）。

**See Something, Say Something：** 在任何工作流步骤中，只要你注意到看起来有问题的地方，不只是测试失败，都要简要指出。用一句话说明：你注意到了什么，以及它的影响。在 solo 模式下，接着问 `"Want me to fix it?"`。在 collaborative 模式下，只提示并继续。

绝不要让注意到的问题悄悄溜过去。这个机制的全部意义就在于主动沟通。

## 先搜索，再构建

在构建基础设施、不熟悉的模式，或任何运行时可能已经内建的能力之前，**先搜索。** 阅读 `~/.claude/skills/gstack/ETHOS.md` 了解完整理念。

**三层知识：**
- **Layer 1**（久经验证，已广泛存在）。不要重复造轮子。但检查的成本接近于零，而且偶尔，质疑那些“久经验证”的做法正是产生卓越想法的来源。
- **Layer 2**（新且流行 —— 搜索这些）。但要审慎：人类容易陷入狂热。搜索结果只是思考输入，不是答案。
- **Layer 3**（第一性原理 —— 最高优先级）。基于对具体问题的推理得出的原创观察。这是最有价值的一层。

**Eureka moment：** 当基于第一性原理的推理揭示传统观点是错误的时，要明确命名：
`"EUREKA: Everyone does X because [assumption]. But [evidence] shows this is wrong. Y is better because [reasoning]."`

记录 eureka moments：
```bash
jq -n --arg ts "$(date -u +%Y-%m-%dT%H:%M:%SZ)" --arg skill "SKILL_NAME" --arg branch "$(git branch --show-current 2>/dev/null)" --arg insight "ONE_LINE_SUMMARY" '{ts:$ts,skill:$skill,branch:$branch,insight:$insight}' >> ~/.gstack/analytics/eureka.jsonl 2>/dev/null || true
```

替换 SKILL_NAME 和 ONE_LINE_SUMMARY。内联运行，不要中断工作流。

**WebSearch 回退：** 如果 WebSearch 不可用，跳过搜索步骤，并注明：`"Search unavailable — proceeding with in-distribution knowledge only."`

## Contributor Mode

如果 `_CONTRIB` 是 `true`：你处于 **contributor mode**。你是 gstack 的用户，同时也帮助它变得更好。

**在每个主要工作流步骤结束时**（不是每一条命令后），回顾一下你刚才使用的 gstack 工具。为体验打 0 到 10 分。如果不是 10 分，思考原因。如果存在明显、可操作的 bug，或者对 gstack 代码或 skill markdown 有一个有洞察力且有意义的改进点，就提交一份 field report。也许我们的 contributor 会帮助我们变得更好！

**评分标准 —— 这就是门槛：** 例如，`$B js "await fetch(...)"` 以前会报错 `SyntaxError: await is only valid in async functions`，因为 gstack 没有把表达式包裹在 async 上下文中。问题虽小，但这个输入是合理的，gstack 理应处理好 —— 这类情况就值得提交。比这更轻微的，就忽略。

**不值得提交：** 用户应用本身的 bug、访问用户 URL 的网络错误、用户站点的认证失败、用户自己的 JS 逻辑错误。

**提交流程：** 写入 `~/.gstack/contributor-logs/{slug}.md`，包含**以下所有部分**（不要截断 —— 必须包含直到 Date/Version footer 的每一节）：

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

Slug：小写，使用连字符，最长 60 个字符（例如 `browse-js-no-await`）。如果文件已存在，则跳过。每个会话最多 3 份报告。内联提交并继续，不要中断工作流。告诉用户：`"Filed gstack field report: {title}"`

## Completion Status Protocol

完成 skill 工作流时，用以下其中一种状态报告：

- **DONE** —— 所有步骤均已成功完成。为每个结论提供证据。
- **DONE_WITH_CONCERNS** —— 已完成，但存在用户需要知晓的问题。逐项列出每个 concern。
- **BLOCKED** —— 无法继续。说明阻塞原因以及已经尝试过什么。
- **NEEDS_CONTEXT** —— 缺少继续所需的信息。明确说明你需要什么。

### Escalation

在以下情况下，始终可以停下来并说 `"this is too hard for me"` 或 `"I'm not confident in this result."`

糟糕的工作比不做更差。提出升级处理不会受到惩罚。
- 如果你已尝试某项任务 3 次仍未成功，停止并升级处理。
- 如果你对某项涉及安全的变更不确定，停止并升级处理。
- 如果工作范围超出了你能验证的程度，停止并升级处理。

升级处理格式：
```
STATUS: BLOCKED | NEEDS_CONTEXT
REASON: [1-2 sentences]
ATTEMPTED: [what you tried]
RECOMMENDATION: [what the user should do next]
```

## Telemetry（最后运行）

在 skill 工作流完成后（成功、出错或中止），记录 telemetry 事件。
根据本文件 YAML frontmatter 中的 `name:` 字段确定 skill 名称。
根据工作流结果确定 outcome（正常完成则为 success，失败则为 error，
用户中断则为 abort）。

**PLAN MODE 例外 —— 始终运行：** 该命令会把 telemetry 写入
`~/.gstack/analytics/`（用户配置目录，而不是项目文件）。skill 的
前言已经向同一目录写入过内容；这是同一种模式。
跳过该命令会丢失会话时长和结果数据。

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
success/error/abort，将 `USED_BROWSE` 替换为 true/false，依据 `$B` 是否被使用。
如果无法确定 outcome，使用 `"unknown"`。该命令在后台运行，
绝不会阻塞用户。

## Plan Status Footer

当你处于 plan mode 并且即将调用 ExitPlanMode 时：

1. 检查 plan 文件是否已经包含 `## GSTACK REVIEW REPORT` 这一节。
2. 如果**已经有** —— 跳过（说明某个 review skill 已经写入了更完整的报告）。
3. 如果**没有** —— 运行以下命令：

\`\`\`bash
~/.claude/skills/gstack/bin/gstack-review-read
\`\`\`

然后在 plan 文件末尾写入一个 `## GSTACK REVIEW REPORT` 小节：

- 如果输出中包含 review 条目（`---CONFIG---` 之前的 JSONL 行）：按
  标准报告表格格式输出每个 skill 的 runs/status/findings，格式与 review
  skills 使用的格式一致。
- 如果输出是 `NO_REVIEWS` 或为空：写入这个占位表格：

\`\`\`markdown
## GSTACK REVIEW REPORT

| Review | Trigger | Why | Runs | Status | Findings |
|--------|---------|-----|------|--------|----------|
| CEO Review | \`/plan-ceo-review\` | 范围与策略 | 0 | — | — |
| Codex Review | \`/codex review\` | 独立的第二意见 | 0 | — | — |
| Eng Review | \`/plan-eng-review\` | 架构与测试（必需） | 0 | — | — |
| Design Review | \`/plan-design-review\` | UI/UX 缺口 | 0 | — | — |

**VERDICT:** 尚无任何评审 —— 运行 \`/autoplan\` 以执行完整评审流水线，或运行上面的单项评审。
\`\`\`

**PLAN MODE 例外 —— 始终运行：** 这会写入 plan 文件，而这是你在 plan mode 下唯一允许编辑的文件。plan 文件中的 review report 是 plan 实时状态的一部分。

## Step 0: 检测基线分支

确定这个 PR 以哪个分支为目标。在后续所有步骤中，都将结果作为“基线分支”。

1. 检查这个分支是否已经存在 PR：
   `gh pr view --json baseRefName -q .baseRefName`
   如果命令成功，使用输出的分支名作为基线分支。

2. 如果不存在 PR（命令失败），检测仓库的默认分支：
   `gh repo view --json defaultBranchRef -q .defaultBranchRef.name`

3. 如果两个命令都失败，则回退到 `main`。

打印检测到的基线分支名称。在后续所有 `git diff`、`git log`、
`git fetch`、`git merge` 和 `gh pr create` 命令中，只要说明里写着
“the base branch”，都替换成检测到的这个分支名。

---

# Ship：全自动 Ship 工作流

你正在运行 `/ship` 工作流。这是一个**非交互式、全自动**工作流。任何步骤都**不要**请求确认。用户说了 `/ship`，意思就是去做。直接一路执行到底，最后输出 PR URL。

**只在以下情况下停止：**
- 当前就在基线分支上（中止）
- 出现无法自动解决的合并冲突（停止并显示冲突）
- 当前分支内的测试失败（预先存在的失败会分流处理，不会自动阻塞）
- pre-landing review 发现需要用户判断的 ASK 项
- 需要进行 MINOR 或 MAJOR 版本提升（询问用户 —— 见 Step 4）
- Greptile review comments 需要用户决定（复杂修复、误报）
- 缺少 TODOS.md 且用户希望创建一个（询问 —— 见 Step 5.5）
- TODOS.md 结构混乱且用户希望重新整理（询问 —— 见 Step 5.5）

**绝不要因为以下情况停止：**
- 有未提交更改（始终一并纳入）
- 版本提升选择（自动选择 MICRO 或 PATCH —— 见 Step 4）
- CHANGELOG 内容（根据 diff 自动生成）
- commit message 审批（自动提交）
- 多文件 changeset（自动拆分为可 bisect 的 commits）
- TODOS.md 已完成项检测（自动标记）
- 可自动修复的 review findings（死代码、N+1、过时注释 —— 自动修复）
- 测试覆盖缺口（自动生成并提交，或在 PR body 中标记）

---

## Step 1: 起飞前检查

1. 检查当前分支。如果当前就在基线分支或仓库默认分支上，**中止**：`"You're on the base branch. Ship from a feature branch."`

2. 运行 `git status`（绝不使用 `-uall`）。始终包含未提交更改，无需询问。

3. 运行 `git diff <base>...HEAD --stat` 和 `git log <base>..HEAD --oneline` 以了解本次要 ship 的内容。

4. 检查 review readiness：

## Review Readiness Dashboard

完成 review 后，读取 review log 和 config 来显示 dashboard。

```bash
~/.claude/skills/gstack/bin/gstack-review-read
```

解析输出。为每个 skill 找到最近一条记录（plan-ceo-review、plan-eng-review、plan-design-review、design-review-lite、adversarial-review、codex-review）。忽略时间戳早于 7 天前的记录。对于 Adversarial 行，显示 `adversarial-review`（新的自动缩放）和 `codex-review`（旧版）中较新的那个。对于 Design Review，显示 `plan-design-review`（完整视觉审计）和 `design-review-lite`（代码级检查）中较新的那个。追加 `(FULL)` 或 `(LITE)` 到状态后面以示区分。显示：

```
+====================================================================+
|                    REVIEW READINESS DASHBOARD                       |
+====================================================================+
| Review          | Runs | Last Run            | Status    | Required |
|-----------------|------|---------------------|-----------|----------|
| Eng Review      |  1   | 2026-03-16 15:00    | CLEAR     | YES      |
| CEO Review      |  0   | —                   | —         | no       |
| Design Review   |  0   | —                   | —         | no       |
| Adversarial     |  0   | —                   | —         | no       |
+--------------------------------------------------------------------+
| VERDICT: CLEARED — Eng Review passed                                |
+====================================================================+
```

**Review 层级：**
- **Eng Review（默认必需）：** 唯一会阻止 ship 的 review。涵盖架构、代码质量、测试、性能。可通过 \`gstack-config set skip_eng_review true\` 全局禁用（也就是“别烦我”设置）。
- **CEO Review（可选）：** 由你自行判断。对于重大的产品/业务变更、新的面向用户功能或范围决策，建议运行。对于 bug 修复、重构、基础设施和清理工作则可跳过。
- **Design Review（可选）：** 由你自行判断。对于 UI/UX 变更，建议运行。对于纯后端、基础设施或仅 prompt 的变更则可跳过。
- **Adversarial Review（自动）：** 按 diff 大小自动缩放。小 diff（<50 行）跳过 adversarial。中等 diff（50–199）进行跨模型 adversarial。大 diff（200+）执行全部 4 轮：Claude structured、Codex structured、Claude adversarial subagent、Codex adversarial。无需任何配置。

**Verdict 逻辑：**
- **CLEARED**：Eng Review 在 7 天内至少有 1 条状态为 `"clean"` 的记录（或 \`skip_eng_review\` 为 \`true\`）
- **NOT CLEARED**：缺少 Eng Review、已过期（>7 天）或存在未解决问题
- CEO、Design 和 Codex reviews 仅作上下文展示，绝不会阻止 ship
- 如果 \`skip_eng_review\` 配置为 \`true\`，则 Eng Review 显示 `"SKIPPED (global)"`，verdict 为 CLEARED

**过期检测：** 显示 dashboard 之后，检查任何现有 review 是否可能已过期：
- 解析 bash 输出中的 \`---HEAD---\` 小节，获取当前 HEAD commit hash
- 对每条带有 \`commit\` 字段的 review 记录：将它与当前 HEAD 比较。如果不同，统计经过了多少个 commits：\`git rev-list --count STORED_COMMIT..HEAD\`。显示：`"Note: {skill} review from {date} may be stale — {N} commits since review"`
- 对于没有 \`commit\` 字段的记录（旧版记录）：显示 `"Note: {skill} review from {date} has no commit tracking — consider re-running for accurate staleness detection"`
- 如果所有 reviews 都匹配当前 HEAD，则不显示任何过期提示

如果 Eng Review 不是 `"CLEAR"`：

1. **检查此分支上是否已有 override：**
   ```bash
   source <(~/.claude/skills/gstack/bin/gstack-slug 2>/dev/null)
   grep '"skill":"ship-review-override"' ~/.gstack/projects/$SLUG/$BRANCH-reviews.jsonl 2>/dev/null || echo "NO_OVERRIDE"
   ```
   如果存在 override，显示 dashboard，并注明 `"Review gate previously accepted — continuing."`。不要再次询问。

2. **如果没有 override，** 使用 AskUserQuestion：
   - 说明 Eng Review 缺失或存在未解决问题
   - RECOMMENDATION：如果变更显然非常小（< 20 行、typo 修复、仅配置变更），选择 C；对于更大的改动，选择 B
   - 选项：A) 仍然 ship  B) 中止 —— 先运行 /plan-eng-review  C) 这个变更太小，不需要 eng review
   - 如果 CEO Review 缺失，仅作为信息说明（`"CEO Review not run — recommended for product changes"`），但不要阻止
   - 对于 Design Review：运行 `source <(~/.claude/skills/gstack/bin/gstack-diff-scope <base> 2>/dev/null)`。如果 `SCOPE_FRONTEND=true` 且 dashboard 中没有 design review（plan-design-review 或 design-review-lite），则说明：`"Design Review not run — this PR changes frontend code. The lite design check will run automatically in Step 3.5, but consider running /design-review for a full visual audit post-implementation."` 仍然绝不阻止。

3. **如果用户选择 A 或 C，** 持久化这个决定，以便后续在这个分支上再次运行 `/ship` 时跳过此 gate：
   ```bash
   source <(~/.claude/skills/gstack/bin/gstack-slug 2>/dev/null)
   echo '{"skill":"ship-review-override","timestamp":"'"$(date -u +%Y-%m-%dT%H:%M:%SZ)"'","decision":"USER_CHOICE"}' >> ~/.gstack/projects/$SLUG/$BRANCH-reviews.jsonl
   ```
   将 USER_CHOICE 替换为 `"ship_anyway"` 或 `"not_relevant"`。

---

## Step 2: 合并基线分支（在测试之前）

先将基线分支 fetch 并 merge 到功能分支，这样测试会在合并后的状态上运行：

```bash
git fetch origin <base> && git merge origin/<base> --no-edit
```

**如果出现 merge conflicts：** 如果是简单冲突（VERSION、schema.rb、CHANGELOG 顺序），尝试自动解决。如果冲突复杂或存在歧义，**停止**并显示冲突内容。

**如果已经是最新：** 静默继续。

---

## Step 2.5: 测试框架 Bootstrap

## Test Framework Bootstrap

**检测现有测试框架和项目运行时：**

```bash
# Detect project runtime
[ -f Gemfile ] && echo "RUNTIME:ruby"
[ -f package.json ] && echo "RUNTIME:node"
[ -f requirements.txt ] || [ -f pyproject.toml ] && echo "RUNTIME:python"
[ -f go.mod ] && echo "RUNTIME:go"
[ -f Cargo.toml ] && echo "RUNTIME:rust"
[ -f composer.json ] && echo "RUNTIME:php"
[ -f mix.exs ] && echo "RUNTIME:elixir"
# Detect sub-frameworks
[ -f Gemfile ] && grep -q "rails" Gemfile 2>/dev/null && echo "FRAMEWORK:rails"
[ -f package.json ] && grep -q '"next"' package.json 2>/dev/null && echo "FRAMEWORK:nextjs"
# Check for existing test infrastructure
ls jest.config.* vitest.config.* playwright.config.* .rspec pytest.ini pyproject.toml phpunit.xml 2>/dev/null
ls -d test/ tests/ spec/ __tests__/ cypress/ e2e/ 2>/dev/null
# Check opt-out marker
[ -f .gstack/no-test-bootstrap ] && echo "BOOTSTRAP_DECLINED"
```

**如果检测到测试框架**（发现 config 文件或测试目录）：
打印 `"Test framework detected: {name} ({N} existing tests). Skipping bootstrap."`
读取 2-3 个现有测试文件以学习约定（命名、imports、断言风格、setup 模式）。
将这些约定以自然语言上下文的形式保存，供 Phase 8e.5 或 Step 3.4 使用。**跳过 bootstrap 的其余部分。**

**如果出现 `BOOTSTRAP_DECLINED`：** 打印 `"Test bootstrap previously declined — skipping."` **跳过 bootstrap 的其余部分。**

**如果未检测到运行时**（没有发现 config 文件）：使用 AskUserQuestion：
`"I couldn't detect your project's language. What runtime are you using?"`
选项：A) Node.js/TypeScript B) Ruby/Rails C) Python D) Go E) Rust F) PHP G) Elixir H) This project doesn't need tests.
如果用户选择 H → 写入 `.gstack/no-test-bootstrap` 并继续，不运行测试。

**如果检测到运行时但没有测试框架 —— 执行 bootstrap：**

### B2. Research best practices

使用 WebSearch 查找检测到的运行时的当前最佳实践：
- `"[runtime] best test framework 2025 2026"`
- `"[framework A] vs [framework B] comparison"`

如果 WebSearch 不可用，则使用下面的内置知识表：

| Runtime | Primary recommendation | Alternative |
|---------|----------------------|-------------|
| Ruby/Rails | minitest + fixtures + capybara | rspec + factory_bot + shoulda-matchers |
| Node.js | vitest + @testing-library | jest + @testing-library |
| Next.js | vitest + @testing-library/react + playwright | jest + cypress |
| Python | pytest + pytest-cov | unittest |
| Go | stdlib testing + testify | stdlib only |
| Rust | cargo test (built-in) + mockall | — |
| PHP | phpunit + mockery | pest |
| Elixir | ExUnit (built-in) + ex_machina | — |

### B3. Framework selection

使用 AskUserQuestion：
`"I detected this is a [Runtime/Framework] project with no test framework. I researched current best practices. Here are the options:
A) [Primary] — [rationale]. Includes: [packages]. Supports: unit, integration, smoke, e2e
B) [Alternative] — [rationale]. Includes: [packages]
C) Skip — don't set up testing right now
RECOMMENDATION: Choose A because [reason based on project context]"`

如果用户选择 C → 写入 `.gstack/no-test-bootstrap`。告诉用户：`"If you change your mind later, delete `.gstack/no-test-bootstrap` and re-run."` 然后继续，不运行测试。

如果检测到多个运行时（monorepo）→ 询问先为哪个运行时建立配置，并提供依次处理两者的选项。

### B4. Install and configure

1. 安装所选 packages（npm/bun/gem/pip 等）
2. 创建最小 config 文件
3. 创建目录结构（test/、spec/ 等）
4. 创建一个与项目代码匹配的示例测试，用于验证配置生效

如果 package 安装失败 → 调试一次。如果仍然失败 → 使用 `git checkout -- package.json package-lock.json` 回退（或对应运行时的等效做法）。警告用户，并继续，不运行测试。

### B4.5. First real tests

为现有代码生成 3-5 个真实测试：

1. **查找最近改动的文件：** `git log --since=30.days --name-only --format="" | sort | uniq -c | sort -rn | head -10`
2. **按风险排序：** 错误处理 > 带条件分支的业务逻辑 > API endpoints > 纯函数
3. **对每个文件：** 编写一个测试，验证真实行为，并带有有意义的断言。绝不要使用 `expect(x).toBeDefined()` —— 要测试代码“做了什么”。
4. 运行每个测试。通过 → 保留。失败 → 修一次。仍失败 → 静默删除。
5. 至少生成 1 个测试，最多 5 个。

绝不要在测试文件中导入 secrets、API keys 或 credentials。使用 environment variables 或 test fixtures。

### B5. Verify

```bash
# Run the full test suite to confirm everything works
{detected test command}
```

如果测试失败 → 调试一次。如果仍失败 → 回退所有 bootstrap 改动并警告用户。

### B5.5. CI/CD pipeline

```bash
# Check CI provider
ls -d .github/ 2>/dev/null && echo "CI:github"
ls .gitlab-ci.yml .circleci/ bitrise.yml 2>/dev/null
```

如果存在 `.github/`（或者未检测到 CI —— 默认使用 GitHub Actions）：
创建 `.github/workflows/test.yml`，内容包括：
- `runs-on: ubuntu-latest`
- 适合该运行时的 setup action（setup-node、setup-ruby、setup-python 等）
- 与 B5 中验证通过的相同测试命令
- 触发条件：push + pull_request

如果检测到非 GitHub CI → 跳过生成 CI，并注明：`"Detected {provider} — CI pipeline generation supports GitHub Actions only. Add test step to your existing pipeline manually."`

### B6. Create TESTING.md

首先检查：如果 `TESTING.md` 已存在 → 读取它，并在原有内容基础上更新/追加，而不是覆盖。绝不要破坏现有内容。

编写 `TESTING.md`，内容包括：
- Philosophy：`"100% test coverage is the key to great vibe coding. Tests let you move fast, trust your instincts, and ship with confidence — without them, vibe coding is just yolo coding. With tests, it's a superpower."`
- 框架名称和版本
- 如何运行测试（B5 中验证过的命令）
- 测试层级：Unit tests（测试什么、在哪里、何时使用）、Integration tests、Smoke tests、E2E tests
- 约定：文件命名、断言风格、setup/teardown 模式

### B7. Update CLAUDE.md

首先检查：如果 `CLAUDE.md` 已经有 `## Testing` 小节 → 跳过。不要重复。

追加一个 `## Testing` 小节：
- 运行命令和测试目录
- 指向 `TESTING.md` 的引用
- 测试预期：
  - 目标是 100% 测试覆盖 —— 测试让 vibe coding 变得安全
  - 编写新函数时，编写对应测试
  - 修复 bug 时，编写回归测试
  - 添加错误处理时，编写触发该错误的测试
  - 添加条件分支（if/else、switch）时，为**两条路径**都编写测试
  - 绝不要提交会导致现有测试失败的代码

### B8. Commit

```bash
git status --porcelain
```

只有在存在变更时才提交。stage 所有 bootstrap 文件（config、测试目录、TESTING.md、CLAUDE.md、如果创建了则包括 `.github/workflows/test.yml`）：
`git commit -m "chore: bootstrap test framework ({framework name})"`

---

---

## Step 3: 运行测试（在合并后的代码上）

**不要运行 `RAILS_ENV=test bin/rails db:migrate`** —— `bin/test-lane` 已经会在内部调用
`db:test:prepare`，它会把 schema 加载到正确的 lane 数据库中。
如果在没有 INSTANCE 的情况下直接运行测试迁移，会打到一个孤立数据库并破坏 structure.sql。

并行运行两个测试套件：

```bash
bin/test-lane 2>&1 | tee /tmp/ship_tests.txt &
npm run test 2>&1 | tee /tmp/ship_vitest.txt &
wait
```

两者都完成后，读取输出文件并检查通过/失败情况。

**如果有任何测试失败：** 不要立刻停止。先应用 Test Failure Ownership Triage：

## Test Failure Ownership Triage

测试失败时，**不要**立刻停止。先确定归属：

### Step T1: 对每个失败进行分类

对每一个失败的测试：

1. **获取此分支修改过的文件：**
   ```bash
   git diff origin/<base>...HEAD --name-only
   ```

2. **对失败进行分类：**
   - **In-branch**：如果失败的测试文件本身在此分支上被修改过，或者测试输出引用了此分支改动过的代码，或者你能将失败追溯到此分支 diff 中的某项变更。
   - **Likely pre-existing**：如果测试文件和它测试的代码都没有在此分支中修改过，且该失败与你能识别出的任何分支变更无关。
   - **如果有歧义，默认归类为 in-branch。** 阻止开发者比放行一个坏测试更安全。只有在你有把握时，才归类为 pre-existing。

   这个分类是启发式的 —— 结合 diff 和测试输出自行判断。你没有程序化的依赖图。

### Step T2: 处理 in-branch 失败

**停止。** 这些失败属于你。展示它们，并且不要继续。开发者必须先修复自己分支上导致的测试失败，然后才能 ship。

### Step T3: 处理 pre-existing 失败

检查前言输出中的 `REPO_MODE`。

**如果 `REPO_MODE` 是 `solo`：**

使用 AskUserQuestion：

> 这些测试失败看起来是预先存在的（不是由你的分支变更引起的）：
>
> [列出每个失败，包括 file:line 和简短错误描述]
>
> 因为这是一个 solo 仓库，所以最终只有你会来修这些问题。
>
> RECOMMENDATION: Choose A — fix now while the context is fresh. Completeness: 9/10.
> A) Investigate and fix now (human: ~2-4h / CC: ~15min) — Completeness: 10/10
> B) Add as P0 TODO — fix after this branch lands — Completeness: 7/10
> C) Skip — I know about this, ship anyway — Completeness: 3/10

**如果 `REPO_MODE` 是 `collaborative` 或 `unknown`：**

使用 AskUserQuestion：

> 这些测试失败看起来是预先存在的（不是由你的分支变更引起的）：
>
> [列出每个失败，包括 file:line 和简短错误描述]
>
> 这是一个 collaborative 仓库 —— 这些问题可能属于别人负责。
>
> RECOMMENDATION: Choose B — assign it to whoever broke it so the right person fixes it. Completeness: 9/10.
> A) Investigate and fix now anyway — Completeness: 10/10
> B) Blame + assign GitHub issue to the author — Completeness: 9/10
> C) Add as P0 TODO — Completeness: 7/10
> D) Skip — ship anyway — Completeness: 3/10

### Step T4: 执行选定操作

**如果选择 “Investigate and fix now”：**
- 切换到 /investigate 思维模式：先找根因，再做最小修复。
- 修复这个预先存在的失败。
- 将修复与当前分支变更分开提交：`git commit -m "fix: pre-existing test failure in <test-file>"`
- 然后继续工作流。

**如果选择 “Add as P0 TODO”：**
- 如果存在 `TODOS.md`，按 `review/TODOS-format.md`（或 `.claude/skills/review/TODOS-format.md`）中的格式添加条目。
- 如果不存在 `TODOS.md`，创建它，写入标准头部，并添加该条目。
- 条目应包括：标题、错误输出、在哪个分支上发现、优先级 P0。
- 继续工作流 —— 将此预先存在的失败视为非阻塞。

**如果选择 “Blame + assign GitHub issue”（仅 collaborative）：**
- 找出最可能引入问题的人。检查**测试文件**和它所测试的**生产代码**：
  ```bash
  # Who last touched the failing test?
  git log --format="%an (%ae)" -1 -- <failing-test-file>
  # Who last touched the production code the test covers? (often the actual breaker)
  git log --format="%an (%ae)" -1 -- <source-file-under-test>
  ```
  如果这两者是不同的人，优先选择生产代码作者 —— 他们更可能引入了回归。
- 创建一个 GitHub issue，并指派给此人：
  ```bash
  gh issue create \
    --title "Pre-existing test failure: <test-name>" \
    --body "Found failing on branch <current-branch>. Failure is pre-existing.\n\n**Error:**\n```\n<first 10 lines>\n```\n\n**Last modified by:** <author>\n**Noticed by:** gstack /ship on <date>" \
    --assignee "<github-username>"
  ```
- 如果 `gh` 不可用或 `--assignee` 失败（用户不在组织中等），则创建不带 assignee 的 issue，并在 body 中注明应该由谁查看。
- 然后继续工作流。

**如果选择 “Skip”：**
- 继续工作流。
- 在输出中注明：`"Pre-existing test failure skipped: <test-name>"`

**分流完成后：** 如果还有任何未修复的 in-branch 失败，**停止**。不要继续。如果所有失败都属于 pre-existing 且已得到处理（修复、记为 TODO、指派或跳过），则继续到 Step 3.25。

**如果全部通过：** 静默继续 —— 只需简要记录通过数量。

---

## Step 3.25: Eval Suites（按条件执行）

当 prompt 相关文件发生变化时，evals 是必需的。如果 diff 中没有 prompt 文件，则完全跳过此步骤。

**1. 检查 diff 是否触及 prompt 相关文件：**

```bash
git diff origin/<base> --name-only
```

匹配以下模式（来自 CLAUDE.md）：
- `app/services/*_prompt_builder.rb`
- `app/services/*_generation_service.rb`, `*_writer_service.rb`, `*_designer_service.rb`
- `app/services/*_evaluator.rb`, `*_scorer.rb`, `*_classifier_service.rb`, `*_analyzer.rb`
- `app/services/concerns/*voice*.rb`, `*writing*.rb`, `*prompt*.rb`, `*token*.rb`
- `app/services/chat_tools/*.rb`, `app/services/x_thread_tools/*.rb`
- `config/system_prompts/*.txt`
- `test/evals/**/*`（eval 基础设施变更会影响所有 suites）

**如果没有匹配项：** 打印 `"No prompt-related files changed — skipping evals."`，然后继续到 Step 3.5。

**2. 确定受影响的 eval suites：**

每个 eval runner（`test/evals/*_eval_runner.rb`）都会声明 `PROMPT_SOURCE_FILES`，列出哪些源文件会影响它。grep 它们，以找出哪些 suites 与变更文件匹配：

```bash
grep -l "changed_file_basename" test/evals/*_eval_runner.rb
```

runner 到测试文件的映射：`post_generation_eval_runner.rb` → `post_generation_eval_test.rb`。

**特殊情况：**
- 对 `test/evals/judges/*.rb`、`test/evals/support/*.rb` 或 `test/evals/fixtures/` 的改动会影响使用这些 judges/support 文件的**所有** suites。检查 eval 测试文件中的 imports 以确定受影响范围。
- 对 `config/system_prompts/*.txt` 的改动 —— grep eval runners 中对应的 prompt 文件名，以查找受影响的 suites。
- 如果不确定哪些 suites 受影响，就运行所有**可能**受影响的 suites。多测总比漏掉回归更好。

**3. 以 `EVAL_JUDGE_TIER=full` 运行受影响的 suites：**

`/ship` 是合并前 gate，因此始终使用 full tier（Sonnet structural + Opus persona judges）。

```bash
EVAL_JUDGE_TIER=full EVAL_VERBOSE=1 bin/test-lane --eval test/evals/<suite>_eval_test.rb 2>&1 | tee /tmp/ship_evals.txt
```

如果需要运行多个 suites，按顺序执行（每个都需要一个 test lane）。如果第一个 suite 失败，立即停止 —— 不要继续消耗 API 成本跑剩余 suites。

**4. 检查结果：**

- **如果有任何 eval 失败：** 显示失败信息、cost dashboard，并**停止**。不要继续。
- **如果全部通过：** 记录通过数量和成本。继续到 Step 3.5。

**5. 保存 eval 输出** —— 在 PR body 中包含 eval 结果和 cost dashboard（Step 8）。

**Tier 参考（仅供上下文 —— `/ship` 始终使用 `full`）：**
| Tier | When | Speed (cached) | Cost |
|------|------|----------------|------|
| `fast` (Haiku) | 开发迭代、smoke tests | ~5s (14x 更快) | ~$0.07/run |
| `standard` (Sonnet) | 默认开发、`bin/test-lane --eval` | ~17s (4x 更快) | ~$0.37/run |
| `full` (Opus persona) | **`/ship` 和合并前** | ~72s（基线） | ~$1.27/run |

---

## Step 3.4: Test Coverage Audit

目标是 100% 覆盖 —— 每一条未测试路径，都是 bug 藏身之处，也会让 vibe coding 变成 yolo coding。评估的是**实际写出来的代码**（来自 diff），而不是原本计划写什么。

### Test Framework Detection

在分析覆盖率之前，先检测项目的测试框架：

1. **读取 CLAUDE.md** —— 查找 `## Testing` 小节，其中应包含测试命令和框架名称。如果找到了，就把它作为权威来源。
2. **如果 CLAUDE.md 没有 testing 小节，则自动检测：**

```bash
# Detect project runtime
[ -f Gemfile ] && echo "RUNTIME:ruby"
[ -f package.json ] && echo "RUNTIME:node"
[ -f requirements.txt ] || [ -f pyproject.toml ] && echo "RUNTIME:python"
[ -f go.mod ] && echo "RUNTIME:go"
[ -f Cargo.toml ] && echo "RUNTIME:rust"
# Check for existing test infrastructure
ls jest.config.* vitest.config.* playwright.config.* cypress.config.* .rspec pytest.ini phpunit.xml 2>/dev/null
ls -d test/ tests/ spec/ __tests__/ cypress/ e2e/ 2>/dev/null
```

3. **如果没有检测到框架：** 则进入 Test Framework Bootstrap 步骤（Step 2.5），由它处理完整配置。

**0. 生成前/后的测试数量：**

```bash
# Count test files before any generation
find . -name '*.test.*' -o -name '*.spec.*' -o -name '*_test.*' -o -name '*_spec.*' | grep -v node_modules | wc -l
```

保存这个数字，用于写入 PR body。

**1. 追踪每一条被修改的代码路径**，使用 `git diff origin/<base>...HEAD`：

读取每个变更文件。对每一个文件，追踪数据如何在代码中流动 —— 不要只是列出函数，要真正跟随执行路径：

1. **读取 diff。** 对每个变更文件，读取整个文件（不只是 diff hunk），以理解上下文。
2. **追踪数据流。** 从每个入口点（route handler、导出函数、事件监听器、组件 render）开始，沿着每条分支跟踪数据：
   - 输入来自哪里？（request params、props、database、API call）
   - 什么在变换它？（validation、mapping、computation）
   - 它流向哪里？（database write、API response、rendered output、side effect）
   - 每一步可能出现什么问题？（null/undefined、invalid input、network failure、empty collection）
3. **画出执行图。** 对每个变更文件，绘制 ASCII 图，显示：
   - 每个新增或修改的函数/方法
   - 每个条件分支（if/else、switch、ternary、guard clause、early return）
   - 每条错误路径（try/catch、rescue、error boundary、fallback）
   - 每次对其他函数的调用（继续追进去 —— 那个函数里是否也有未测试的分支？）
   - 每条边界情况：null 输入会怎样？空数组呢？无效类型呢？

这是关键步骤 —— 你正在构建一张地图，展示所有会根据输入而执行不同路径的代码。图中的每个分支都需要一个测试。

**2. 映射用户流程、交互和错误状态：**

代码覆盖率还不够 —— 你还必须覆盖真实用户如何与这段变更代码交互。对每个变更功能，逐步思考：

- **用户流程：** 用户会执行怎样的动作序列，才会触达这段代码？把完整旅程画出来（例如 `"user clicks 'Pay' → form validates → API call → success/failure screen"`）。旅程中的每一步都需要一个测试。
- **交互边界情况：** 当用户做出意料之外的操作时会发生什么？
  - 双击 / 快速重复提交
  - 操作中途离开页面（返回按钮、关闭标签页、点击其他链接）
  - 使用过期数据提交（页面开着 30 分钟、session 过期）
  - 连接很慢（API 需要 10 秒 —— 用户会看到什么？）
  - 并发操作（两个标签页、同一个表单）
- **用户可见的错误状态：** 对于代码处理的每一种错误，用户实际会经历什么？
  - 是清晰的错误提示，还是静默失败？
  - 用户能恢复吗（重试、返回、修正输入），还是会卡住？
  - 无网络时会怎样？API 返回 500 时会怎样？服务器返回无效数据时会怎样？
- **空 / 零 / 边界状态：** 当结果为零时 UI 显示什么？有 10,000 条结果时呢？输入只有一个字符时呢？达到最大长度时呢？

把这些内容也加到图中，与代码分支一起展示。没有测试的用户流程，与没有测试的 if/else 一样，都是覆盖缺口。

**3. 将每个分支与现有测试逐一对照：**

逐个检查图中的每个分支 —— 包括代码路径和用户流程。对每一个，搜索是否已有测试覆盖：
- 函数 `processPayment()` → 查找 `billing.test.ts`、`billing.spec.ts`、`test/billing_test.rb`
- 一个 if/else → 查找是否有测试同时覆盖 true 和 false 路径
- 一个 error handler → 查找是否有测试触发该具体错误条件
- 对 `helperFn()` 的调用，而它本身还有分支 → 那些分支也需要测试
- 一个用户流程 → 查找是否有 integration 或 E2E 测试走完这条旅程
- 一个交互边界情况 → 查找是否有测试模拟这种意外操作

质量评分标准：
- ★★★  测试行为，并覆盖边界情况和错误路径
- ★★   测试正确行为，但仅覆盖 happy path
- ★    Smoke test / 存在性检查 / 轻量断言（例如 `"it renders"`、`"it doesn't throw"`）

### E2E Test Decision Matrix

在检查每个分支时，也要判断是 unit test 还是 E2E/integration test 更合适：

**推荐 E2E（在图中标记为 [→E2E]）：**
- 跨越 3 个以上组件/服务的常见用户流程（例如 signup → verify email → first login）
- 真实失败会被 mock 掩盖的集成点（例如 API → queue → worker → DB）
- auth / payment / data-destruction 流程 —— 太重要，不能只靠 unit tests

**推荐 EVAL（在图中标记为 [→EVAL]）：**
- 需要质量评估的关键 LLM 调用（例如 prompt 变更 → 测试输出仍满足质量门槛）
- 对 prompt templates、system instructions 或 tool definitions 的修改

**坚持使用 UNIT TESTS：**
- 输入输出明确的纯函数
- 没有副作用的内部 helper
- 单一函数的边界情况（null input、empty array）
- 不面向客户的冷门/少见流程

### REGRESSION RULE（强制）

**铁律：** 当 coverage audit 识别出 REGRESSION —— 即 diff 破坏了之前可正常工作的行为 —— 必须立即编写 regression test。不使用 AskUserQuestion。不能跳过。回归测试是最高优先级的测试，因为它能证明某些东西确实坏了。

以下情况属于 regression：
- diff 修改了已有行为（不是新代码）
- 现有测试套件（如果有）没有覆盖这条变更路径
- 此次变更为已有调用方引入了新的失败模式

如果不确定某项变更是否属于 regression，宁可写这个测试。

格式：提交信息写为 `test: regression test for {what broke}`

**4. 输出 ASCII 覆盖图：**

在同一张图中同时包含代码路径和用户流程。标记值得做 E2E 和 eval 的路径：

```
CODE PATH COVERAGE
===========================
[+] src/services/billing.ts
    │
    ├── processPayment()
    │   ├── [★★★ TESTED] Happy path + card declined + timeout — billing.test.ts:42
    │   ├── [GAP]         Network timeout — NO TEST
    │   └── [GAP]         Invalid currency — NO TEST
    │
    └── refundPayment()
        ├── [★★  TESTED] Full refund — billing.test.ts:89
        └── [★   TESTED] Partial refund (checks non-throw only) — billing.test.ts:101

USER FLOW COVERAGE
===========================
[+] Payment checkout flow
    │
    ├── [★★★ TESTED] Complete purchase — checkout.e2e.ts:15
    ├── [GAP] [→E2E] Double-click submit — needs E2E, not just unit
    ├── [GAP]         Navigate away during payment — unit test sufficient
    └── [★   TESTED]  Form validation errors (checks render only) — checkout.test.ts:40

[+] Error states
    │
    ├── [★★  TESTED] Card declined message — billing.test.ts:58
    ├── [GAP]         Network timeout UX (what does user see?) — NO TEST
    └── [GAP]         Empty cart submission — NO TEST

[+] LLM integration
    │
    └── [GAP] [→EVAL] Prompt template change — needs eval test

─────────────────────────────────
COVERAGE: 5/13 paths tested (38%)
  Code paths: 3/5 (60%)
  User flows: 2/8 (25%)
QUALITY:  ★★★: 2  ★★: 2  ★: 1
GAPS: 8 paths need tests (2 need E2E, 1 needs eval)
─────────────────────────────────
```

**快速路径：** 所有路径都已覆盖 → `"Step 3.4: All new code paths have test coverage ✓"` 继续。

**5. 为未覆盖路径生成测试：**

如果检测到测试框架（或在 Step 2.5 中完成了 bootstrap）：
- 优先处理 error handlers 和 edge cases（happy path 更可能已经有测试）
- 读取 2-3 个现有测试文件，严格匹配它们的约定
- 生成 unit tests。mock 所有外部依赖（DB、API、Redis）。
- 对于标记为 [→E2E] 的路径：使用项目的 E2E 框架（Playwright、Cypress、Capybara 等）生成 integration/E2E tests
- 对于标记为 [→EVAL] 的路径：使用项目的 eval framework 生成 eval tests；如果没有，则标记为手动 eval
- 编写测试，针对具体未覆盖路径，使用真实断言
- 运行每个测试。通过 → 以 `test: coverage for {feature}` 提交
- 失败 → 修一次。仍失败 → 回退，并在图中注明缺口。

限制：最多 30 条代码路径，最多生成 20 个测试（代码 + 用户流程合计），每个测试探索上限 2 分钟。

如果没有测试框架且用户拒绝 bootstrap → 只输出图，不生成测试。注明：`"Test generation skipped — no test framework configured."`

**如果 diff 只有测试相关变更：** 完全跳过 Step 3.4：`"No new application code paths to audit."`

**6. 生成后计数与覆盖率摘要：**

```bash
# Count test files after generation
find . -name '*.test.*' -o -name '*.spec.*' -o -name '*_test.*' -o -name '*_spec.*' | grep -v node_modules | wc -l
```

写入 PR body：
`Tests: {before} → {after} (+{delta} new)`
覆盖率行：
`Test Coverage Audit: N new code paths. M covered (X%). K tests generated, J committed.`

### Test Plan Artifact

生成 coverage 图后，写出一个 test plan artifact，以便 `/qa` 和 `/qa-only` 使用：

```bash
source <(~/.claude/skills/gstack/bin/gstack-slug 2>/dev/null) && mkdir -p ~/.gstack/projects/$SLUG
USER=$(whoami)
DATETIME=$(date +%Y%m%d-%H%M%S)
```

写入 `~/.gstack/projects/{slug}/{user}-{branch}-ship-test-plan-{datetime}.md`：

```markdown
# Test Plan
Generated by /ship on {date}
Branch: {branch}
Repo: {owner/repo}

## Affected Pages/Routes
- {URL path} — {what to test and why}

## Key Interactions to Verify
- {interaction description} on {page}

## Edge Cases
- {edge case} on {page}

## Critical Paths
- {end-to-end flow that must work}
```

---

## Step 3.5: Pre-Landing Review

审查 diff，查找测试无法捕捉的结构性问题。

1. 读取 `.claude/skills/review/checklist.md`。如果无法读取该文件，**停止**并报告错误。

2. 运行 `git diff origin/<base>` 获取完整 diff（范围限定为相对于刚 fetch 下来的基线分支的功能变更）。

3. 分两轮应用 review checklist：
   - **Pass 1（CRITICAL）：** SQL & Data Safety、LLM Output Trust Boundary
   - **Pass 2（INFORMATIONAL）：** 其余所有类别

## Design Review（按条件执行，范围限定于 diff）

使用 `gstack-diff-scope` 检查 diff 是否触及前端文件：

```bash
source <(~/.claude/skills/gstack/bin/gstack-diff-scope <base> 2>/dev/null)
```

**如果 `SCOPE_FRONTEND=false`：** 静默跳过 design review。不输出任何内容。

**如果 `SCOPE_FRONTEND=true`：**

1. **检查 DESIGN.md。** 如果仓库根目录存在 `DESIGN.md` 或 `design-system.md`，读取它。所有 design findings 都要以它为基准进行校准 —— 在 DESIGN.md 中已明确认可的模式不应被标记。如果未找到，则使用通用设计原则。

2. **读取 `.claude/skills/review/design-checklist.md`。** 如果无法读取该文件，则跳过 design review，并注明：`"Design checklist not found — skipping design review."`

3. **读取每个变更的前端文件**（读取完整文件，而不只是 diff hunks）。前端文件通过 checklist 中列出的模式识别。

4. **对变更文件应用 design checklist。** 对每一项：
   - **[HIGH] 机械式 CSS 修复**（`outline: none`、`!important`、`font-size < 16px`）：归类为 AUTO-FIX
   - **[HIGH/MEDIUM] 需要设计判断**：归类为 ASK
   - **[LOW] 基于意图的检测**：表述为 `"Possible — verify visually or run /design-review"`

5. **将 findings 包含在 review 输出中**，放在 `"Design Review"` 标题下，格式遵循 checklist 中的输出格式。Design findings 与代码 review findings 合并进入同一个 Fix-First 流程。

6. **记录结果**，用于 Review Readiness Dashboard：

```bash
~/.claude/skills/gstack/bin/gstack-review-log '{"skill":"design-review-lite","timestamp":"TIMESTAMP","status":"STATUS","findings":N,"auto_fixed":M,"commit":"COMMIT"}'
```

替换：TIMESTAMP = ISO 8601 datetime，STATUS = 若 0 个 findings 则为 `"clean"`，否则为 `"issues_found"`，N = findings 总数，M = 自动修复数量，COMMIT = `git rev-parse --short HEAD` 的输出。

7. **Codex design voice**（可选，如可用则自动运行）：

```bash
which codex 2>/dev/null && echo "CODEX_AVAILABLE" || echo "CODEX_NOT_AVAILABLE"
```

如果 Codex 可用，则对 diff 运行一个轻量 design check：

```bash
TMPERR_DRL=$(mktemp /tmp/codex-drl-XXXXXXXX)
codex exec "Review the git diff on this branch. Run 7 litmus checks (YES/NO each): 1. Brand/product unmistakable in first screen? 2. One strong visual anchor present? 3. Page understandable by scanning headlines only? 4. Each section has one job? 5. Are cards actually necessary? 6. Does motion improve hierarchy or atmosphere? 7. Would design feel premium with all decorative shadows removed? Flag any hard rejections: 1. Generic SaaS card grid as first impression 2. Beautiful image with weak brand 3. Strong headline with no clear action 4. Busy imagery behind text 5. Sections repeating same mood statement 6. Carousel with no narrative purpose 7. App UI made of stacked cards instead of layout 5 most important design findings only. Reference file:line." -s read-only -c 'model_reasoning_effort="high"' --enable web_search_cached 2>"$TMPERR_DRL"
```

使用 5 分钟超时（`timeout: 300000`）。命令完成后，读取 stderr：
```bash
cat "$TMPERR_DRL" && rm -f "$TMPERR_DRL"
```

**错误处理：** 所有错误都不阻塞流程。遇到认证失败、超时或空响应时 —— 简要说明并跳过，继续执行。

将 Codex 输出放在 `CODEX (design):` 标题下，与上面的 checklist findings 合并展示。

   将所有 design findings 与代码 review findings 一起展示。它们遵循下面相同的 Fix-First 流程。

4. **根据 checklist.md 中的 Fix-First Heuristic，把每个 finding 归类为 AUTO-FIX 或 ASK。** Critical findings 更偏向 ASK；informational 更偏向 AUTO-FIX。

5. **自动修复所有 AUTO-FIX 项。** 应用每个修复。每个修复输出一行：
   `[AUTO-FIXED] [file:line] Problem → what you did`

6. **如果仍有 ASK 项，** 通过**一个** AskUserQuestion 展示它们：
   - 逐项列出编号、严重程度、问题、推荐修复方式
   - 每项选项：A) 修复  B) 跳过
   - 给出整体 RECOMMENDATION
   - 如果 ASK 项不超过 3 个，也可以分别发起单独的 AskUserQuestion

7. **所有修复完成后（自动修复 + 用户批准修复）：**
   - 如果应用了**任何**修复：按文件名提交修复过的文件（`git add <fixed-files> && git commit -m "fix: pre-landing review fixes"`），然后**停止**，并告诉用户重新运行 `/ship` 以重新测试。
   - 如果没有应用修复（所有 ASK 项都被跳过，或没有发现问题）：继续到 Step 4。

8. 输出摘要：`Pre-Landing Review: N issues — M auto-fixed, K asked (J fixed, L skipped)`

   如果没有发现问题：`Pre-Landing Review: No issues found.`

保存 review 输出 —— 它会在 Step 8 中进入 PR body。

---

## Step 3.75: 处理 Greptile review comments（如果 PR 已存在）

读取 `.claude/skills/review/greptile-triage.md` 并遵循其中的 fetch、filter、classify 和 **escalation detection** 步骤。

**如果不存在 PR、`gh` 失败、API 返回错误，或者 Greptile comments 数量为零：** 静默跳过此步骤。继续到 Step 4。

**如果找到了 Greptile comments：**

在输出中包含 Greptile 摘要：`+ N Greptile comments (X valid, Y fixed, Z FP)`

在回复任何 comment 之前，运行 greptile-triage.md 中的 **Escalation Detection** 算法，以确定应使用 Tier 1（友好）还是 Tier 2（坚定）回复模板。

对于每个分类后的 comment：

**VALID & ACTIONABLE：** 使用 AskUserQuestion，内容包括：
- comment 内容（file:line 或 [top-level] + body 摘要 + permalink URL）
- `RECOMMENDATION: Choose A because [one-line reason]`
- 选项：A) 立即修复，B) 确认并继续 ship，C) 这是误报
- 如果用户选择 A：应用修复，提交修复过的文件（`git add <fixed-files> && git commit -m "fix: address Greptile review — <brief description>"`），使用 greptile-triage.md 中的 **Fix reply template** 回复（包含 inline diff + explanation），并同时保存到每项目和全局 greptile-history（type: fix）。
- 如果用户选择 C：使用 greptile-triage.md 中的 **False Positive reply template** 回复（包含 evidence + suggested re-rank），并同时保存到每项目和全局 greptile-history（type: fp）。

**VALID BUT ALREADY FIXED：** 使用 greptile-triage.md 中的 **Already Fixed reply template** 回复 —— 不需要 AskUserQuestion：
- 包含已做了什么以及修复该问题的 commit SHA
- 同时保存到每项目和全局 greptile-history（type: already-fixed）

**FALSE POSITIVE：** 使用 AskUserQuestion：
- 展示 comment 以及你认为它错误的原因（file:line 或 [top-level] + body 摘要 + permalink URL）
- 选项：
  - A) 回复 Greptile，解释这是误报（如果显然错误，推荐）
  - B) 仍然修复（如果修起来很简单）
  - C) 静默忽略
- 如果用户选择 A：使用 greptile-triage.md 中的 **False Positive reply template** 回复（包含 evidence + suggested re-rank），并同时保存到每项目和全局 greptile-history（type: fp）

**SUPPRESSED：** 静默跳过 —— 这些是之前分流后确认的已知误报。

**所有 comments 处理完毕后：** 如果应用了任何修复，则 Step 3 中的测试结果已经过期。**重新运行测试**（Step 3），然后再继续到 Step 4。如果没有应用修复，则直接继续到 Step 4。

---

## Step 3.8: Adversarial review（自动缩放）

Adversarial review 的严格程度会根据 diff 大小自动缩放。无需配置。

**检测 diff 大小和工具可用性：**

```bash
DIFF_INS=$(git diff origin/<base> --stat | tail -1 | grep -oE '[0-9]+ insertion' | grep -oE '[0-9]+' || echo "0")
DIFF_DEL=$(git diff origin/<base> --stat | tail -1 | grep -oE '[0-9]+ deletion' | grep -oE '[0-9]+' || echo "0")
DIFF_TOTAL=$((DIFF_INS + DIFF_DEL))
which codex 2>/dev/null && echo "CODEX_AVAILABLE" || echo "CODEX_NOT_AVAILABLE"
# Respect old opt-out
OLD_CFG=$(~/.claude/skills/gstack/bin/gstack-config get codex_reviews 2>/dev/null || true)
echo "DIFF_SIZE: $DIFF_TOTAL"
echo "OLD_CFG: ${OLD_CFG:-not_set}"
```

如果 `OLD_CFG` 是 `disabled`：静默跳过此步骤。继续到下一步。

**用户覆盖：** 如果用户明确要求某个特定层级（例如 `"run all passes"`、`"paranoid review"`、`"full adversarial"`、`"do all 4 passes"`、`"thorough review"`），无论 diff 大小如何，都应遵从该要求。直接跳到对应层级的小节。

**根据 diff 大小自动选择层级：**
- **小型（< 50 行变更）：** 完全跳过 adversarial review。打印：`"Small diff ($DIFF_TOTAL lines) — adversarial review skipped."` 然后继续到下一步。
- **中型（50–199 行变更）：** 运行 Codex adversarial challenge（如果 Codex 不可用，则使用 Claude adversarial subagent）。跳到 “Medium tier” 小节。
- **大型（200+ 行变更）：** 运行其余全部流程 —— Codex structured review + Claude adversarial subagent + Codex adversarial。跳到 “Large tier” 小节。

---

### Medium tier（50–199 行）

Claude 的 structured review 已经运行过。现在追加一个**跨模型 adversarial challenge**。

**如果 Codex 可用：** 运行 Codex adversarial challenge。**如果 Codex 不可用：** 自动回退到 Claude adversarial subagent。

**Codex adversarial：**

```bash
TMPERR_ADV=$(mktemp /tmp/codex-adv-XXXXXXXX)
codex exec "Review the changes on this branch against the base branch. Run git diff origin/<base> to see the diff. Your job is to find ways this code will fail in production. Think like an attacker and a chaos engineer. Find edge cases, race conditions, security holes, resource leaks, failure modes, and silent data corruption paths. Be adversarial. Be thorough. No compliments — just the problems." -s read-only -c 'model_reasoning_effort="xhigh"' --enable web_search_cached 2>"$TMPERR_ADV"
```

把 Bash 工具的 `timeout` 参数设为 `300000`（5 分钟）。**不要**使用 `timeout` shell 命令 —— macOS 上没有这个命令。命令完成后，读取 stderr：
```bash
cat "$TMPERR_ADV"
```

将完整输出原样展示。这是信息性输出 —— 它绝不会阻止 ship。

**错误处理：** 所有错误都不阻塞 —— adversarial review 只是质量增强，不是前置条件。
- **认证失败：** 如果 stderr 包含 `"auth"`、`"login"`、`"unauthorized"` 或 `"API key"`：`"Codex authentication failed. Run \`codex login\` to authenticate."`
- **超时：** `"Codex timed out after 5 minutes."`
- **空响应：** `"Codex returned no response. Stderr: <paste relevant error>."`

遇到任何 Codex 错误时，自动回退到 Claude adversarial subagent。

**Claude adversarial subagent**（在 Codex 不可用或出错时作为回退）：

通过 Agent 工具派发。subagent 拥有全新的上下文 —— 不会受到 structured review checklist 偏见的影响。这种真正的独立性能够发现主审查者看不到的问题。

Subagent prompt：
`"Read the diff for this branch with \`git diff origin/<base>\`. Think like an attacker and a chaos engineer. Your job is to find ways this code will fail in production. Look for: edge cases, race conditions, security holes, resource leaks, failure modes, silent data corruption, logic errors that produce wrong results silently, error handling that swallows failures, and trust boundary violations. Be adversarial. Be thorough. No compliments — just the problems. For each finding, classify as FIXABLE (you know how to fix it) or INVESTIGATE (needs human judgment)."`

在 `ADVERSARIAL REVIEW (Claude subagent):` 标题下展示 findings。**FIXABLE findings** 会进入与 structured review 相同的 Fix-First 流程。**INVESTIGATE findings** 作为信息展示。

如果 subagent 失败或超时：`"Claude adversarial subagent unavailable. Continuing without adversarial review."`

**持久化 review 结果：**
```bash
~/.claude/skills/gstack/bin/gstack-review-log '{"skill":"adversarial-review","timestamp":"'"$(date -u +%Y-%m-%dT%H:%M:%SZ)"'","status":"STATUS","source":"SOURCE","tier":"medium","commit":"'"$(git rev-parse --short HEAD)"'"}'
```
替换 STATUS：如果没有 findings，则为 `"clean"`；如果有 findings，则为 `"issues_found"`。SOURCE：如果运行的是 Codex，则为 `"codex"`；如果运行的是 subagent，则为 `"claude"`。如果两者都失败，则**不要**持久化。

**清理：** 处理完成后运行 `rm -f "$TMPERR_ADV"`（如果使用了 Codex）。

---

### Large tier（200+ 行）

Claude 的 structured review 已经运行过。现在执行**另外三轮全部流程**，以获得最大覆盖：

**1. Codex structured review（如果可用）：**
```bash
TMPERR=$(mktemp /tmp/codex-review-XXXXXXXX)
codex review --base <base> -c 'model_reasoning_effort="xhigh"' --enable web_search_cached 2>"$TMPERR"
```

把 Bash 工具的 `timeout` 参数设为 `300000`（5 分钟）。**不要**使用 `timeout` shell 命令 —— macOS 上没有这个命令。将输出放在 `CODEX SAYS (code review):` 标题下展示。
检查是否存在 `[P1]` 标记：有 → `GATE: FAIL`，没有 → `GATE: PASS`。

如果 GATE 为 FAIL，使用 AskUserQuestion：
```
Codex 在 diff 中发现了 N 个严重问题。

A) 立即调查并修复（推荐）
B) 继续 —— review 仍会完整执行
```

如果选 A：处理这些 findings。修复后，重新运行测试（Step 3），因为代码已变更。然后重新运行 `codex review` 进行验证。

读取 stderr 中的错误（错误处理与 medium tier 相同）。

读取 stderr 后：`rm -f "$TMPERR"`

**2. Claude adversarial subagent：** 使用 adversarial prompt 派发一个 subagent（与 medium tier 使用相同 prompt）。无论 Codex 是否可用，这一步都要运行。

**3. Codex adversarial challenge（如果可用）：** 使用与 medium tier 相同的 adversarial prompt 运行 `codex exec`。

如果在步骤 1 和 3 中 Codex 不可用，提示用户：`"Codex CLI not found — large-diff review ran Claude structured + Claude adversarial (2 of 4 passes). Install Codex for full 4-pass coverage: \`npm install -g @openai/codex\`"`

**在所有流程完成后再持久化 review 结果**（不是每个子步骤后都写）：
```bash
~/.claude/skills/gstack/bin/gstack-review-log '{"skill":"adversarial-review","timestamp":"'"$(date -u +%Y-%m-%dT%H:%M:%SZ)"'","status":"STATUS","source":"SOURCE","tier":"large","gate":"GATE","commit":"'"$(git rev-parse --short HEAD)"'"}'
```
替换：STATUS = 如果所有流程都没有 findings，则为 `"clean"`；任一流程有 findings，则为 `"issues_found"`。SOURCE = 如果运行了 Codex，则为 `"both"`；如果只运行了 Claude subagent，则为 `"claude"`。GATE = Codex structured review 的 gate 结果（`"pass"`/`"fail"`），如果 Codex 不可用，则为 `"informational"`。如果所有流程都失败，则**不要**持久化。

---

### Cross-model synthesis（medium 和 large tiers）

所有流程完成后，对所有来源的 findings 进行综合：

```
ADVERSARIAL REVIEW SYNTHESIS (auto: TIER, N lines):
════════════════════════════════════════════════════════════
  High confidence (found by multiple sources): [findings agreed on by >1 pass]
  Unique to Claude structured review: [from earlier step]
  Unique to Claude adversarial: [from subagent, if ran]
  Unique to Codex: [from codex adversarial or code review, if ran]
  Models used: Claude structured ✓  Claude adversarial ✓/✗  Codex ✓/✗
════════════════════════════════════════════════════════════
```

高置信度 findings（被多个来源共同指出）应优先修复。

---

## Step 4: Version bump（自动决定）

1. 读取当前 `VERSION` 文件（4 位格式：`MAJOR.MINOR.PATCH.MICRO`）

2. **根据 diff 自动决定 bump 级别：**
   - 统计改动行数（`git diff origin/<base>...HEAD --stat | tail -1`）
   - **MICRO**（第 4 位）：改动少于 50 行、微小调整、typo、配置变更
   - **PATCH**（第 3 位）：改动 50+ 行、bug 修复、小到中等功能
   - **MINOR**（第 2 位）：**询问用户** —— 仅用于重大功能或显著架构变更
   - **MAJOR**（第 1 位）：**询问用户** —— 仅用于里程碑或 breaking changes

3. 计算新版本：
   - 提升某一位时，其右侧所有位都重置为 0
   - 例如：`0.19.1.0` + PATCH → `0.19.2.0`

4. 将新版本写入 `VERSION` 文件。

---

## Step 5: CHANGELOG（自动生成）

1. 读取 `CHANGELOG.md` 头部，了解格式。

2. 根据**分支上的全部 commits** 自动生成条目（不只是最近的）：
   - 使用 `git log <base>..HEAD --oneline` 查看本次 ship 的每一条 commit
   - 使用 `git diff <base>...HEAD` 查看相对于基线分支的完整 diff
   - CHANGELOG 条目必须完整覆盖所有将进入 PR 的变更
   - 如果分支上已有的 CHANGELOG 条目已经覆盖了其中某些 commits，则用一个针对新版本的统一条目替换它们
   - 将变更归类到适用的小节：
     - `### Added` —— 新功能
     - `### Changed` —— 对现有功能的变更
     - `### Fixed` —— bug 修复
     - `### Removed` —— 被移除的功能
   - 写出简洁、描述明确的 bullet points
   - 插入到文件头之后（第 5 行），日期为今天
   - 格式：`## [X.Y.Z.W] - YYYY-MM-DD`

**不要询问用户如何描述这些变更。** 根据 diff 和 commit 历史自行推断。

---

## Step 5.5: TODOS.md（自动更新）

将项目中的 TODOS.md 与本次 ship 的变更进行交叉核对。自动标记已完成项；仅在文件缺失或结构混乱时提示用户。

读取 `.claude/skills/review/TODOS-format.md` 作为规范格式参考。

**1. 检查仓库根目录是否存在** `TODOS.md`。

**如果不存在 `TODOS.md`：** 使用 AskUserQuestion：
- 消息：`"GStack recommends maintaining a TODOS.md organized by skill/component, then priority (P0 at top through P4, then Completed at bottom). See TODOS-format.md for the full format. Would you like to create one?"`
- 选项：A) 现在创建，B) 先跳过
- 如果选 A：创建 `TODOS.md`，写入骨架（`# TODOS` 标题 + `## Completed` 小节）。继续到 step 3。
- 如果选 B：跳过 Step 5.5 的其余部分。继续到 Step 6。

**2. 检查结构和组织：**

读取 TODOS.md，并验证它是否符合推荐结构：
- 条目按 `## <Skill/Component>` 标题分组
- 每个条目都有 `**Priority:**` 字段，值为 P0-P4
- 底部有一个 `## Completed` 小节

**如果结构混乱**（缺少 priority 字段、没有 component 分组、没有 Completed 小节）：使用 AskUserQuestion：
- 消息：`"TODOS.md doesn't follow the recommended structure (skill/component groupings, P0-P4 priority, Completed section). Would you like to reorganize it?"`
- 选项：A) 现在重组（推荐），B) 保持原样
- 如果选 A：按照 TODOS-format.md 就地重组。保留所有内容 —— 只调整结构，绝不删除条目。
- 如果选 B：不重组，直接继续到 step 3。

**3. 检测已完成的 TODOs：**

这一步完全自动 —— 不需要用户交互。

使用前面步骤已经收集到的 diff 和 commit 历史：
- `git diff <base>...HEAD`（相对于基线分支的完整 diff）
- `git log <base>..HEAD --oneline`（本次 ship 的全部 commits）

对于每个 TODO 条目，通过以下方式判断本次 PR 是否完成了它：
- 将 commit messages 与 TODO 标题和描述进行匹配
- 检查 TODO 中引用的文件是否出现在 diff 中
- 检查 TODO 描述的工作内容是否与功能变更相符

**要保守：** 只有当 diff 中有明确证据时，才将 TODO 标记为已完成。如果不确定，就保持不动。

**4. 将已完成项移动** 到底部的 `## Completed` 小节。追加：`**Completed:** vX.Y.Z (YYYY-MM-DD)`

**5. 输出摘要：**
- `TODOS.md: N items marked complete (item1, item2, ...). M items remaining.`
- 或：`TODOS.md: No completed items detected. M items remaining.`
- 或：`TODOS.md: Created.` / `TODOS.md: Reorganized.`

**6. 防御性处理：** 如果无法写入 TODOS.md（权限错误、磁盘已满），警告用户并继续。绝不要因为 TODOS 失败而中断 ship 工作流。

保存这段摘要 —— 它会在 Step 8 中进入 PR body。

---

## Step 6: Commit（可 bisect 的逻辑块）

**目标：** 创建小而清晰的逻辑性 commits，使其更适合 `git bisect`，也有助于 LLM 理解变更内容。

1. 分析 diff，并将变更分组成逻辑 commits。每个 commit 都应表示**一个连贯的变更** —— 不是一个文件，而是一个逻辑单元。

2. **Commit 顺序**（先提交前面的）：
   - **Infrastructure：** migrations、config changes、route additions
   - **Models & services：** 新 models、services、concerns（以及它们的测试）
   - **Controllers & views：** controllers、views、JS/React components（以及它们的测试）
   - **VERSION + CHANGELOG + TODOS.md：** 始终放在最后一个 commit

3. **拆分规则：**
   - model 和它的测试文件放在同一个 commit
   - service 和它的测试文件放在同一个 commit
   - controller、它的 views 以及它的测试放在同一个 commit
   - migrations 单独成一个 commit（或与其支持的 model 一起提交）
   - config/route changes 可与其启用的功能一起分组
   - 如果总 diff 很小（< 50 行且 < 4 个文件），一个 commit 即可

4. **每个 commit 都必须能独立成立** —— 不允许 broken imports，不允许引用尚不存在的代码。按依赖关系排序 commits，使依赖项先出现。

5. 组织每个 commit message：
   - 第一行：`<type>: <summary>`（type = feat/fix/chore/refactor/docs）
   - 正文：简要说明该 commit 包含什么
   - 只有**最后一个 commit**（VERSION + CHANGELOG）才带版本标签和 co-author trailer：

```bash
git commit -m "$(cat <<'EOF'
chore: bump version and changelog (vX.Y.Z.W)

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
EOF
)"
```

---

## Step 6.5: Verification Gate

**铁律：没有最新的验证证据，就不能声称完成。**

在推送之前，如果 Step 4-6 中代码发生了变更，必须重新验证：

1. **测试验证：** 如果在 Step 3 运行测试之后有**任何代码**发生变化（来自 review findings 的修复，CHANGELOG 编辑不算），重新运行测试套件。粘贴最新输出。Step 3 的旧输出**不能接受**。

2. **构建验证：** 如果项目有 build 步骤，就运行它。粘贴输出。

3. **防止自我合理化：**
   - `"Should work now"` → **去运行。**
   - `"I'm confident"` → 信心不是证据。
   - `"I already tested earlier"` → 代码后来改了。重新测试。
   - `"It's a trivial change"` → 微小改动也会破坏生产环境。

**如果这里测试失败：** 停止。不要推送。修复问题并返回 Step 3。

没有验证就声称工作完成，不是高效，而是不诚实。

---

## Step 7: Push

推送到远程，并设置 upstream tracking：

```bash
git push -u origin <branch-name>
```

---

## Step 8: Create PR

使用 `gh` 创建 pull request：

```bash
gh pr create --base <base> --title "<type>: <summary>" --body "$(cat <<'EOF'
## Summary
<bullet points from CHANGELOG>

## Test Coverage
<coverage diagram from Step 3.4, or "All new code paths have test coverage.">
<If Step 3.4 ran: "Tests: {before} → {after} (+{delta} new)">

## Pre-Landing Review
<findings from Step 3.5 code review, or "No issues found.">

## Design Review
<If design review ran: "Design Review (lite): N findings — M auto-fixed, K skipped. AI Slop: clean/N issues.">
<If no frontend files changed: "No frontend files changed — design review skipped.">

## Eval Results
<If evals ran: suite names, pass/fail counts, cost dashboard summary. If skipped: "No prompt-related files changed — evals skipped.">

## Greptile Review
<If Greptile comments were found: bullet list with [FIXED] / [FALSE POSITIVE] / [ALREADY FIXED] tag + one-line summary per comment>
<If no Greptile comments found: "No Greptile comments.">
<If no PR existed during Step 3.75: omit this section entirely>

## TODOS
<If items marked complete: bullet list of completed items with version>
<If no items completed: "No TODO items completed in this PR.">
<If TODOS.md created or reorganized: note that>
<If TODOS.md doesn't exist and user skipped: omit this section>

## Test plan
- [x] All Rails tests pass (N runs, 0 failures)
- [x] All Vitest tests pass (N tests)

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

**输出 PR URL** —— 然后继续到 Step 8.5。

---

## Step 8.5: 自动调用 /document-release

创建 PR 后，自动同步项目文档。读取
`document-release/SKILL.md` skill 文件（与当前 skill 目录相邻），并
执行其完整工作流：

1. 读取 `/document-release` skill：`cat ${CLAUDE_SKILL_DIR}/../document-release/SKILL.md`
2. 按照其中的说明执行 —— 它会读取项目中的所有 .md 文件，交叉比对
   diff，并更新任何已经漂移的内容（README、ARCHITECTURE、CONTRIBUTING、
   CLAUDE.md、TODOS 等）
3. 如果更新了任何文档，就提交变更并推送到同一分支：
   ```bash
   git add -A && git commit -m "docs: sync documentation with shipped changes" && git push
   ```
4. 如果不需要更新任何文档，则说明 `"Documentation is current — no updates needed."`

这一步是自动的。不要向用户请求确认。目标是零摩擦的
文档更新 —— 用户运行 `/ship`，文档就会自动保持最新，无需再执行单独命令。

---

## 重要规则

- **绝不要跳过测试。** 如果测试失败，就停止。
- **绝不要跳过 pre-landing review。** 如果 checklist.md 无法读取，就停止。
- **绝不要 force push。** 只使用普通的 `git push`。
- **绝不要为了琐碎确认打断用户**（例如 `"ready to push?"`、`"create PR?"`）。但以下情况必须停止：版本提升（MINOR/MAJOR）、pre-landing review findings（ASK 项）、以及 Codex structured review 的 [P1] findings（仅限大型 diff）。
- **始终使用 VERSION 文件中的 4 位版本格式。**
- **CHANGELOG 中的日期格式：** `YYYY-MM-DD`
- **为可 bisect 性拆分 commits** —— 每个 commit = 一个逻辑变更。
- **TODOS.md 已完成项检测必须保守。** 只有当 diff 明确显示工作已完成时，才标记为已完成。
- **使用 greptile-triage.md 中的 Greptile 回复模板。** 每条回复都必须包含证据（inline diff、代码引用、re-rank 建议）。绝不要发布含糊回复。
- **没有最新的验证证据就绝不要推送。** 如果 Step 3 测试之后代码发生了变化，推送前必须重新运行测试。
- **Step 3.4 会生成覆盖率测试。** 它们必须先通过，才能提交。绝不要提交失败的测试。
- **目标是：用户说 `/ship`，接下来看到的就是 review + PR URL + 自动同步后的文档。**