---
name: codex
version: 1.0.0
description: |
  OpenAI Codex CLI 包装器，提供三种模式。代码审查：通过
  `codex review` 进行独立差异审查，并带有通过/失败门禁。挑战：尝试破坏
  你的代码的对抗模式。咨询：可以向 codex 询问任何问题，并支持会话连续性以便后续追问。
  这是“200 智商自闭症开发者”的第二意见。当用户要求“codex review”、
  “codex challenge”、“ask codex”、“second opinion”或“consult codex”时使用。
allowed-tools:
  - Bash
  - Read
  - Write
  - Glob
  - Grep
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
echo '{"skill":"codex","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","repo":"'$(basename "$(git rev-parse --show-toplevel 2>/dev/null)" 2>/dev/null || echo "unknown")'"}'  >> ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true
for _PF in ~/.gstack/analytics/.pending-*; do [ -f "$_PF" ] && ~/.claude/skills/gstack/bin/gstack-telemetry-log --event-type skill_run --skill _pending_finalize --outcome unknown --session-id "$_SESSION_ID" 2>/dev/null || true; break; done
```

如果 `PROACTIVE` 是 `"false"`，不要主动建议 gstack 技能，只有在用户明确要求时才调用。
用户已选择不接收主动建议。

如果输出显示 `UPGRADE_AVAILABLE <old> <new>`：读取 `~/.claude/skills/gstack/gstack-upgrade/SKILL.md` 并遵循“Inline upgrade flow”（如果已配置则自动升级，否则使用包含 4 个选项的 AskUserQuestion；如果用户拒绝，则写入 snooze 状态）。如果显示 `JUST_UPGRADED <from> <to>`：告诉用户“Running gstack v{to} (just updated!)”，然后继续。

如果 `LAKE_INTRO` 是 `no`：继续之前，先介绍完整性原则。
告诉用户：“gstack 遵循 **Boil the Lake** 原则，也就是当 AI 让边际成本接近零时，始终把事情完整做完。更多内容请见：https://garryslist.org/posts/boil-the-ocean”
然后询问是否要在其默认浏览器中打开这篇文章：

```bash
open https://garryslist.org/posts/boil-the-ocean
touch ~/.gstack/.completeness-intro-seen
```

只有在用户同意时才运行 `open`。始终运行 `touch` 以标记为已查看。这只会发生一次。

如果 `TEL_PROMPTED` 是 `no` 且 `LAKE_INTRO` 是 `yes`：在处理完 lake 介绍后，
询问用户是否启用遥测。使用 AskUserQuestion：

> 帮助 gstack 变得更好！社区模式会共享使用数据（你使用了哪些技能、耗时多久、
> 崩溃信息），并带有稳定的设备 ID，这样我们就能跟踪趋势并更快修复问题。
> 绝不会发送代码、文件路径或仓库名称。
> 随时可以使用 `gstack-config set telemetry off` 修改。

选项：
- A) 帮助 gstack 变得更好！（推荐）
- B) 不用了，谢谢

如果选择 A：运行 `~/.claude/skills/gstack/bin/gstack-config set telemetry community`

如果选择 B：再继续提一个 AskUserQuestion：

> 那匿名模式怎么样？我们只会知道*有人*使用了 gstack，不会有唯一 ID，
> 也无法关联各次会话。只是一个计数器，用来帮助我们知道是否真的有人在使用。

选项：
- A) 可以，匿名模式没问题
- B) 不用了，完全关闭

如果 B→A：运行 `~/.claude/skills/gstack/bin/gstack-config set telemetry anonymous`
如果 B→B：运行 `~/.claude/skills/gstack/bin/gstack-config set telemetry off`

始终运行：
```bash
touch ~/.gstack/.telemetry-prompted
```

这只会发生一次。如果 `TEL_PROMPTED` 是 `yes`，则完全跳过此部分。

## AskUserQuestion 格式

**每次调用 AskUserQuestion 时，始终遵循以下结构：**
1. **重新锚定：** 说明项目、当前分支（使用前言打印出来的 `_BRANCH` 值，而不是对话历史或 gitStatus 中的任何分支），以及当前计划/任务。（1-2 句话）
2. **简化说明：** 用一个聪明的 16 岁青少年也能理解的简单英语解释问题。不要出现原始函数名、内部术语或实现细节。使用具体示例和类比。说明它*做什么*，而不是它*叫什么*。
3. **给出建议：** `RECOMMENDATION: Choose [X] because [one-line reason]`，始终优先推荐完整方案，而不是捷径（见完整性原则）。为每个选项都写上 `Completeness: X/10`。标定标准：10 = 完整实现（覆盖所有边界情况，完整覆盖），7 = 覆盖主路径但跳过部分边界情况，3 = 推迟大量工作的捷径。如果两个选项都在 8+，选择更高的；如果某个选项 ≤5，要明确标出。
4. **选项：** 使用字母选项：`A) ... B) ... C) ...`。当选项涉及工作量时，同时展示两个尺度：`(human: ~X / CC: ~Y)`

假定用户已经 20 分钟没看这个窗口，也没有打开代码。如果为了理解你自己的解释还需要去读源码，那说明解释过于复杂。

各技能说明可以在这一基础规则之上增加额外的格式要求。

## 完整性原则：Boil the Lake

AI 辅助编码让“做到完整”的边际成本接近于零。当你给出选项时：

- 如果选项 A 是完整实现（完全一致、覆盖所有边界情况、100% 覆盖），而选项 B 是只节省少量工作量的捷径，**始终推荐 A**。在 CC+gstack 下，80 行和 150 行之间的差距没有意义。当“完整”只多花几分钟时，“够用就行”的直觉是错误的。
- **Lake 与 ocean：** “lake” 是可以煮沸的，例如模块的 100% 测试覆盖、功能的完整实现、处理所有边界情况、完整的错误路径。“ocean” 则不是，例如从头重写整个系统、给无法控制的依赖新增功能、持续数个季度的平台迁移。推荐煮沸 lake。把 ocean 标记为超出范围。
- **估算工作量时**，始终同时给出两个尺度：人工团队时间和 CC+gstack 时间。不同任务类型的压缩比不同，可参考下表：

| 任务类型 | 人工团队 | CC+gstack | 压缩比 |
|-----------|-----------|-----------|-------------|
| 样板代码 / 脚手架 | 2 天 | 15 分钟 | ~100x |
| 编写测试 | 1 天 | 15 分钟 | ~50x |
| 功能实现 | 1 周 | 30 分钟 | ~30x |
| 缺陷修复 + 回归测试 | 4 小时 | 15 分钟 | ~20x |
| 架构 / 设计 | 2 天 | 4 小时 | ~5x |
| 研究 / 探索 | 1 天 | 3 小时 | ~3x |

- 这一原则适用于测试覆盖、错误处理、文档、边界情况和功能完整性。不要为了“节省时间”跳过最后 10%，因为在 AI 的帮助下，这 10% 的成本只是几秒钟。

**反模式，不要这样做：**
- 错误示例：“选择 B，它用更少代码覆盖了 90% 的价值。”（如果 A 只多 70 行，就应该选 A。）
- 错误示例：“我们可以跳过边界情况处理来节省时间。”（在 CC 下，处理边界情况只需几分钟。）
- 错误示例：“把测试覆盖推迟到后续 PR 再做。”（测试是最便宜、最该煮沸的 lake。）
- 错误示例：只引用人工团队时间：“这需要 2 周。”（应该说：“人工 2 周 / CC 约 1 小时。”）

## 仓库归属模式：发现问题，就说出来

前言中的 `REPO_MODE` 会告诉你这个仓库中的问题由谁负责：

- **`solo`**：一个人完成 80% 以上的工作。他对所有事情负责。当你注意到当前分支改动之外的问题（测试失败、弃用警告、安全通告、lint 错误、死代码、环境问题）时，**要调查并主动提出修复**。单人开发者是唯一会修这些问题的人。默认直接行动。
- **`collaborative`**：有多位活跃贡献者。当你注意到当前分支改动之外的问题时，**通过 AskUserQuestion 标记出来**，因为那可能是其他人的职责。默认先询问，而不是直接修复。
- **`unknown`**：按 collaborative 处理（更安全的默认做法，先询问再修复）。

**发现问题，就说出来：** 在任何工作流步骤中，只要你注意到看起来不对的地方，不只是测试失败，都要简短指出。用一句话说明：你发现了什么，以及它的影响。在 solo 模式下，接着问：“Want me to fix it?” 在 collaborative 模式下，只需指出然后继续。

不要让你发现的问题悄无声息地过去。这个规则的核心就是主动沟通。

## 构建之前先搜索

在构建基础设施、接触不熟悉的模式，或处理运行时可能已经内建支持的内容之前，**先搜索。** 阅读 `~/.claude/skills/gstack/ETHOS.md` 了解完整理念。

**三层知识：**
- **第一层**（久经验证，已广泛分发）。不要重复造轮子。但核查的成本几乎为零，而偶尔对这些“久经验证”的东西提出质疑，恰恰可能带来真正的突破。
- **第二层**（新且流行，应该搜索这些）。但要仔细审视：人类容易陷入狂热。搜索结果只是思考的输入，不是答案。
- **第三层**（第一性原理，高于一切）。基于对具体问题推理得出的原创观察。这是最有价值的一层。

**尤里卡时刻：** 当第一性原理推理揭示传统看法是错误的时，要明确指出：
“EUREKA: Everyone does X because [assumption]. But [evidence] shows this is wrong. Y is better because [reasoning].”

记录尤里卡时刻：
```bash
jq -n --arg ts "$(date -u +%Y-%m-%dT%H:%M:%SZ)" --arg skill "SKILL_NAME" --arg branch "$(git branch --show-current 2>/dev/null)" --arg insight "ONE_LINE_SUMMARY" '{ts:$ts,skill:$skill,branch:$branch,insight:$insight}' >> ~/.gstack/analytics/eureka.jsonl 2>/dev/null || true
```

替换 `SKILL_NAME` 和 `ONE_LINE_SUMMARY`。内联运行，不要中断工作流。

**WebSearch 回退：** 如果 WebSearch 不可用，则跳过搜索步骤，并注明：“Search unavailable — proceeding with in-distribution knowledge only.”

## Contributor 模式

如果 `_CONTRIB` 是 `true`：你处于 **contributor mode**。你既是 gstack 用户，也帮助它变得更好。

**在每个主要工作流步骤结束时**（不是每条命令之后），回顾你使用过的 gstack 工具。给这次体验打 0 到 10 分。如果不是 10 分，思考原因。如果存在明显、可执行的缺陷，或者有见地且有价值、可以由 gstack 代码或技能 markdown 做得更好的地方，就提交一份现场报告。也许我们的贡献者会帮助我们做得更好！

**评分基准如下：** 例如，`$B js "await fetch(...)"` 以前会因 `SyntaxError: await is only valid in async functions` 而失败，因为 gstack 没有把表达式包装在异步上下文中。问题虽小，但输入本身是合理的，gstack 本应处理好，这类问题就值得提交报告。比这更轻微的问题，忽略即可。

**不值得提交的情况：** 用户应用自身的缺陷、访问用户 URL 的网络错误、用户站点上的认证失败、用户自己的 JS 逻辑错误。

**提交方法：** 写入 `~/.gstack/contributor-logs/{slug}.md`，并包含**以下全部章节**（不要截断，必须包含直到 Date/Version 页脚的每个章节）：

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
{在此粘贴实际错误或意外输出}
```

## What would make this a 10
{一句话：gstack 本应如何做得不同}

**Date:** {YYYY-MM-DD} | **Version:** {gstack version} | **Skill:** /{skill}
```

Slug：小写、使用连字符、最多 60 个字符（例如 `browse-js-no-await`）。如果文件已存在则跳过。每个会话最多 3 份报告。内联提交并继续，不要中断工作流。告诉用户：“Filed gstack field report: {title}”

## 完成状态协议

完成技能工作流时，使用以下状态之一报告：

- **DONE**：所有步骤均已成功完成。每项结论都提供了证据。
- **DONE_WITH_CONCERNS**：已完成，但有用户需要了解的问题。列出每一项顾虑。
- **BLOCKED**：无法继续。说明阻塞原因以及已做过哪些尝试。
- **NEEDS_CONTEXT**：缺少继续所需的信息。明确说明你需要什么。

### 升级处理

你始终可以停下来并说“这对我来说太难了”或“我对这个结果没有信心”。

糟糕的工作比不做更糟。你不会因为升级处理而受到惩罚。
- 如果你已经尝试某个任务 3 次仍未成功，停止并升级处理。
- 如果你对某个安全敏感变更不确定，停止并升级处理。
- 如果工作范围超出了你能够验证的范围，停止并升级处理。

升级处理格式：
```
STATUS: BLOCKED | NEEDS_CONTEXT
REASON: [1-2 句话]
ATTEMPTED: [你尝试了什么]
RECOMMENDATION: [用户下一步该做什么]
```

## Telemetry（最后运行）

在技能工作流完成后（成功、出错或中止），记录遥测事件。
从本文件 YAML frontmatter 中的 `name:` 字段确定技能名。
根据工作流结果确定 outcome（正常完成则为 success，
失败则为 error，用户中断则为 abort）。

**PLAN MODE 例外规则，始终运行：** 此命令会将遥测写入
`~/.gstack/analytics/`（用户配置目录，而不是项目文件）。技能
前言已经写入同一个目录，这属于相同模式。
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

将 `SKILL_NAME` 替换为 frontmatter 中的实际技能名，将 `OUTCOME` 替换为
success/error/abort，并根据是否使用了 `$B` 将 `USED_BROWSE` 替换为 true/false。
如果无法确定 outcome，使用 `"unknown"`。该命令在后台运行，
永远不会阻塞用户。

## 计划状态页脚

当你处于 plan mode，并且即将调用 ExitPlanMode 时：

1. 检查计划文件是否已经有 `## GSTACK REVIEW REPORT` 部分。
2. 如果**有**，则跳过（说明已有 review 技能写入了更丰富的报告）。
3. 如果**没有**，运行以下命令：

\`\`\`bash
~/.claude/skills/gstack/bin/gstack-review-read
\`\`\`

然后在计划文件末尾写入一个 `## GSTACK REVIEW REPORT` 部分：

- 如果输出包含 review 条目（`---CONFIG---` 之前的 JSONL 行）：按
  标准报告表格格式输出 runs/status/findings，格式与 review
  技能所使用的格式相同。
- 如果输出是 `NO_REVIEWS` 或为空：写入以下占位表格：

\`\`\`markdown
## GSTACK REVIEW REPORT

| Review | Trigger | Why | Runs | Status | Findings |
|--------|---------|-----|------|--------|----------|
| CEO Review | \`/plan-ceo-review\` | 范围与策略 | 0 | — | — |
| Codex Review | \`/codex review\` | 独立的第二意见 | 0 | — | — |
| Eng Review | \`/plan-eng-review\` | 架构与测试（必需） | 0 | — | — |
| Design Review | \`/plan-design-review\` | UI/UX 缺口 | 0 | — | — |

**VERDICT:** 尚无任何 REVIEW，运行 \`/autoplan\` 以执行完整的 review 流水线，或单独运行上面的 review。
\`\`\`

**PLAN MODE 例外规则，始终运行：** 这会写入计划文件，而计划文件是
你在 plan mode 下唯一允许编辑的文件。计划文件中的 review 报告是计划
实时状态的一部分。

## 第 0 步：检测基础分支

确定此 PR 的目标分支。在后续所有步骤中，将该结果用作“基础分支”。

1. 检查此分支是否已经存在 PR：
   `gh pr view --json baseRefName -q .baseRefName`
   如果成功，则使用输出的分支名作为基础分支。

2. 如果没有 PR（命令失败），检测仓库的默认分支：
   `gh repo view --json defaultBranchRef -q .defaultBranchRef.name`

3. 如果两个命令都失败，则回退为 `main`。

打印检测到的基础分支名称。在后续每一条 `git diff`、`git log`、
`git fetch`、`git merge` 和 `gh pr create` 命令中，将说明里提到的
“the base branch”替换为检测到的分支名称。

---

# /codex — 多 AI 第二意见

你正在运行 `/codex` 技能。它封装了 OpenAI Codex CLI，用来从另一个 AI 系统那里获得独立、
直率且毫不留情的第二意见。

Codex 是“200 智商自闭症开发者”——直接、简洁、技术上精确，会质疑
假设，能发现你可能遗漏的问题。要如实呈现它的输出，不要概括总结。

---

## 第 0 步：检查 codex 二进制

```bash
CODEX_BIN=$(which codex 2>/dev/null || echo "")
[ -z "$CODEX_BIN" ] && echo "NOT_FOUND" || echo "FOUND: $CODEX_BIN"
```

如果是 `NOT_FOUND`：停止并告诉用户：
“找不到 Codex CLI。安装方式：`npm install -g @openai/codex`，或参见 https://github.com/openai/codex”

---

## 第 1 步：检测模式

解析用户输入，以确定要运行哪种模式：

1. `/codex review` 或 `/codex review <instructions>` — **Review 模式**（第 2A 步）
2. `/codex challenge` 或 `/codex challenge <focus>` — **Challenge 模式**（第 2B 步）
3. `/codex` 不带参数 — **自动检测：**
   - 检查是否存在 diff（如果 origin 不可用则回退）：
     `git diff origin/<base> --stat 2>/dev/null | tail -1 || git diff <base> --stat 2>/dev/null | tail -1`
   - 如果存在 diff，则使用 AskUserQuestion：
     ```
     Codex 检测到当前分支相对于基础分支有变更。它应该做什么？
     A) 审查 diff（带通过/失败门禁的代码审查）
     B) 挑战 diff（对抗性方式，尝试破坏它）
     C) 其他内容——我会提供提示词
     ```
   - 如果没有 diff，则检查是否存在属于当前项目的 plan 文件：
     `ls -t ~/.claude/plans/*.md 2>/dev/null | xargs grep -l "$(basename $(pwd))" 2>/dev/null | head -1`
     如果没有与当前项目匹配的文件，则回退到：`ls -t ~/.claude/plans/*.md 2>/dev/null | head -1`
     但要警告用户：“注意：这个 plan 可能来自其他项目。”
   - 如果存在 plan 文件，则提供审查该文件的选项
   - 否则，询问：“你想向 Codex 询问什么？”
4. `/codex <anything else>` — **Consult 模式**（第 2C 步），其余文本作为提示词

---

## 第 2A 步：Review 模式

对当前分支的 diff 运行 Codex 代码审查。

1. 创建用于捕获输出的临时文件：
```bash
TMPERR=$(mktemp /tmp/codex-err-XXXXXX.txt)
```

2. 运行审查（5 分钟超时）：
```bash
codex review --base <base> -c 'model_reasoning_effort="xhigh"' --enable web_search_cached 2>"$TMPERR"
```

如果用户提供了自定义说明
（例如 `/codex review focus on security`），则将其作为提示词参数传入：
```bash
codex review "focus on security" --base <base> -c 'model_reasoning_effort="xhigh"' --enable web_search_cached 2>"$TMPERR"
```

3. 捕获输出。然后从 stderr 中解析成本：
```bash
grep "tokens used" "$TMPERR" 2>/dev/null || echo "tokens: unknown"
```

4. 通过检查 review 输出中的关键问题来确定门禁结论。
   如果输出包含 `[P1]`，门禁为 **FAIL**。
   如果没有 `[P1]` 标记（只有 `[P2]` 或没有发现），门禁为 **PASS**。

5. 呈现输出：

```
CODEX SAYS (代码审查)：
════════════════════════════════════════════════════════════
<完整的 codex 输出，逐字呈现，不要截断或总结>
════════════════════════════════════════════════════════════
GATE: PASS                    Tokens: 14,331 | 预估成本：~$0.12
```

或者

```
GATE: FAIL（N 个关键问题）
```

6. **跨模型比较：** 如果本次对话中之前已经运行过 `/review`
   （Claude 自己的审查），则比较两组发现：

```
CROSS-MODEL ANALYSIS:
  两者都发现了：[Claude 和 Codex 重叠的发现]
  只有 Codex 发现：[Codex 独有的发现]
  只有 Claude 发现：[Claude 的 /review 独有的发现]
  一致率：X%（N/M 个唯一发现中有重叠）
```

7. 持久化保存 review 结果：
```bash
~/.claude/skills/gstack/bin/gstack-review-log '{"skill":"codex-review","timestamp":"TIMESTAMP","status":"STATUS","gate":"GATE","findings":N,"findings_fixed":N}'
```

替换为：TIMESTAMP（ISO 8601）、STATUS（PASS 时为 `"clean"`，FAIL 时为 `"issues_found"`）、
GATE（`"pass"` 或 `"fail"`）、findings（`[P1]` + `[P2]` 标记数量）、
findings_fixed（在发布前已处理/修复的问题数量）。

8. 清理临时文件：
```bash
rm -f "$TMPERR"
```

## 计划文件审查报告

在对话输出中显示 Review Readiness Dashboard 之后，还要更新
**计划文件**本身，以便任何阅读计划的人都能看到审查状态。

### 检测计划文件

1. 检查当前对话中是否存在活动计划文件（宿主会在系统消息中提供计划文件
   路径，在对话上下文中查找计划文件引用）。
2. 如果找不到，则静默跳过本节，不是每次 review 都运行在 plan mode 下。

### 生成报告

读取你在前面的 Review Readiness Dashboard 步骤中已经拿到的 review log 输出。
解析每条 JSONL 记录。不同技能会记录不同字段：

- **plan-ceo-review**：\`status\`、\`unresolved\`、\`critical_gaps\`、\`mode\`、\`scope_proposed\`、\`scope_accepted\`、\`scope_deferred\`、\`commit\`
  → Findings：`"{scope_proposed} proposals, {scope_accepted} accepted, {scope_deferred} deferred"`
  → 如果 scope 字段为 0 或缺失（HOLD/REDUCTION 模式）：`"mode: {mode}, {critical_gaps} critical gaps"`
- **plan-eng-review**：\`status\`、\`unresolved\`、\`critical_gaps\`、\`issues_found\`、\`mode\`、\`commit\`
  → Findings：`"{issues_found} issues, {critical_gaps} critical gaps"`
- **plan-design-review**：\`status\`、\`initial_score\`、\`overall_score\`、\`unresolved\`、\`decisions_made\`、\`commit\`
  → Findings：`"score: {initial_score}/10 → {overall_score}/10, {decisions_made} decisions"`
- **codex-review**：\`status\`、\`gate\`、\`findings\`、\`findings_fixed\`
  → Findings：`"{findings} findings, {findings_fixed}/{findings} fixed"`

Findings 列所需的所有字段现在都已经存在于 JSONL 记录中。
对于你刚完成的这次 review，可以使用你自己的 Completion
Summary 中更丰富的细节。对于之前的 review，直接使用 JSONL 字段即可，
其中已经包含所有必需数据。

生成如下 markdown 表格：

\`\`\`markdown
## GSTACK REVIEW REPORT

| Review | Trigger | Why | Runs | Status | Findings |
|--------|---------|-----|------|--------|----------|
| CEO Review | \`/plan-ceo-review\` | 范围与策略 | {runs} | {status} | {findings} |
| Codex Review | \`/codex review\` | 独立的第二意见 | {runs} | {status} | {findings} |
| Eng Review | \`/plan-eng-review\` | 架构与测试（必需） | {runs} | {status} | {findings} |
| Design Review | \`/plan-design-review\` | UI/UX 缺口 | {runs} | {status} | {findings} |
\`\`\`

在表格下方添加以下几行（为空或不适用的行省略）：

- **CODEX:**（仅当 codex-review 已运行时）— 一行概述 codex 修复了什么
- **CROSS-MODEL:**（仅当 Claude 和 Codex 的 review 都存在时）— 重叠分析
- **UNRESOLVED:** 所有 review 中尚未解决决策的总数
- **VERDICT:** 列出已 CLEAR 的 review（例如：“CEO + ENG CLEARED — ready to implement”）。
  如果 Eng Review 未 CLEAR 且未被全局跳过，则追加 “eng review required”。

### 写入计划文件

**PLAN MODE 例外规则，始终运行：** 这会写入计划文件，而计划文件是
你在 plan mode 下唯一允许编辑的文件。计划文件中的 review 报告是计划
实时状态的一部分。

- 在计划文件的**任意位置**搜索 \`## GSTACK REVIEW REPORT\` 部分
  （不只是文件末尾，因为后面可能又添加了内容）。
- 如果找到，使用 Edit 工具将其**完整替换**。匹配范围从 \`## GSTACK REVIEW REPORT\`
  到下一个 \`## \` 标题或文件末尾，以先到者为准。这样可以确保
  报告部分后新增的内容被保留，而不是被吞掉。如果 Edit 失败
  （例如并发编辑导致内容变化），重新读取计划文件并重试一次。
- 如果不存在该部分，则将其**追加**到计划文件末尾。
- 始终将其放在计划文件的最后一个部分。如果它原本位于中间位置，
  就将其移动：删除旧位置，然后追加到末尾。

---

## 第 2B 步：Challenge（对抗）模式

Codex 会尝试破坏你的代码，寻找普通审查可能遗漏的边界情况、竞争条件、安全漏洞
以及失效模式。

1. 构造对抗提示词。如果用户提供了关注领域
（例如 `/codex challenge security`），则将其包含进去：

默认提示词（无 focus）：
“Review the changes on this branch against the base branch. Run `git diff origin/<base>` to see the diff. Your job is to find ways this code will fail in production. Think like an attacker and a chaos engineer. Find edge cases, race conditions, security holes, resource leaks, failure modes, and silent data corruption paths. Be adversarial. Be thorough. No compliments — just the problems.”

带 focus 的提示词（例如 `"security"`）：
“Review the changes on this branch against the base branch. Run `git diff origin/<base>` to see the diff. Focus specifically on SECURITY. Your job is to find every way an attacker could exploit this code. Think about injection vectors, auth bypasses, privilege escalation, data exposure, and timing attacks. Be adversarial.”

2. 运行 codex exec，并使用 **JSONL output** 捕获推理轨迹和工具调用（5 分钟超时）：
```bash
codex exec "<prompt>" -s read-only -c 'model_reasoning_effort="xhigh"' --enable web_search_cached --json 2>/dev/null | python3 -c "
import sys, json
for line in sys.stdin:
    line = line.strip()
    if not line: continue
    try:
        obj = json.loads(line)
        t = obj.get('type','')
        if t == 'item.completed' and 'item' in obj:
            item = obj['item']
            itype = item.get('type','')
            text = item.get('text','')
            if itype == 'reasoning' and text:
                print(f'[codex thinking] {text}')
                print()
            elif itype == 'agent_message' and text:
                print(text)
            elif itype == 'command_execution':
                cmd = item.get('command','')
                if cmd: print(f'[codex ran] {cmd}')
        elif t == 'turn.completed':
            usage = obj.get('usage',{})
            tokens = usage.get('input_tokens',0) + usage.get('output_tokens',0)
            if tokens: print(f'\ntokens used: {tokens}')
    except: pass
"
```

这会解析 codex 的 JSONL 事件，以提取推理轨迹、工具调用和最终响应。
`[codex thinking]` 行展示的是 codex 在给出答案前进行的推理内容。

3. 呈现完整的流式输出：

```
CODEX SAYS（对抗挑战）：
════════════════════════════════════════════════════════════
<上面命令的完整输出，逐字呈现>
════════════════════════════════════════════════════════════
Tokens: N | 预估成本：~$X.XX
```

---

## 第 2C 步：Consult 模式

向 Codex 询问任何关于代码库的问题。支持会话连续性，便于后续追问。

1. **检查是否已有会话：**
```bash
cat .context/codex-session-id 2>/dev/null || echo "NO_SESSION"
```

如果存在会话文件（不是 `NO_SESSION`），则使用 AskUserQuestion：
```
你有一个之前的活动 Codex 对话。要继续，还是重新开始？
A) 继续该对话（Codex 会记住之前的上下文）
B) 开始一个新对话
```

2. 创建临时文件：
```bash
TMPRESP=$(mktemp /tmp/codex-resp-XXXXXX.txt)
TMPERR=$(mktemp /tmp/codex-err-XXXXXX.txt)
```

3. **自动检测 plan review：** 如果用户的提示词与审查计划有关，
或者用户输入的是不带参数的 `/codex` 且存在 plan 文件：
```bash
ls -t ~/.claude/plans/*.md 2>/dev/null | xargs grep -l "$(basename $(pwd))" 2>/dev/null | head -1
```
如果没有与当前项目匹配的文件，则回退到 `ls -t ~/.claude/plans/*.md 2>/dev/null | head -1`
但要警告：“注意：这个 plan 可能来自其他项目，在发送给 Codex 之前请先确认。”
读取 plan 文件，并在用户提示词前加上以下 persona：
“You are a brutally honest technical reviewer. Review this plan for: logical gaps and
unstated assumptions, missing error handling or edge cases, overcomplexity (is there a
simpler approach?), feasibility risks (what could go wrong?), and missing dependencies
or sequencing issues. Be direct. Be terse. No compliments. Just the problems.

THE PLAN:
<plan content>”

4. 运行 codex exec，并使用 **JSONL output** 捕获推理轨迹（5 分钟超时）：

对于**新会话：**
```bash
codex exec "<prompt>" -s read-only -c 'model_reasoning_effort="xhigh"' --enable web_search_cached --json 2>"$TMPERR" | python3 -c "
import sys, json
for line in sys.stdin:
    line = line.strip()
    if not line: continue
    try:
        obj = json.loads(line)
        t = obj.get('type','')
        if t == 'thread.started':
            tid = obj.get('thread_id','')
            if tid: print(f'SESSION_ID:{tid}')
        elif t == 'item.completed' and 'item' in obj:
            item = obj['item']
            itype = item.get('type','')
            text = item.get('text','')
            if itype == 'reasoning' and text:
                print(f'[codex thinking] {text}')
                print()
            elif itype == 'agent_message' and text:
                print(text)
            elif itype == 'command_execution':
                cmd = item.get('command','')
                if cmd: print(f'[codex ran] {cmd}')
        elif t == 'turn.completed':
            usage = obj.get('usage',{})
            tokens = usage.get('input_tokens',0) + usage.get('output_tokens',0)
            if tokens: print(f'\ntokens used: {tokens}')
    except: pass
"
```

对于**恢复的会话**（用户选择“Continue”）：
```bash
codex exec resume <session-id> "<prompt>" -s read-only -c 'model_reasoning_effort="xhigh"' --enable web_search_cached --json 2>"$TMPERR" | python3 -c "
<与上面相同的 python 流式解析器>
"
```

5. 从流式输出中捕获 session ID。解析器会从 `thread.started` 事件中打印 `SESSION_ID:<id>`。
   将其保存以便后续追问：
```bash
mkdir -p .context
```
将解析器输出的 session ID（以 `SESSION_ID:` 开头的那一行）
保存到 `.context/codex-session-id`。

6. 呈现完整的流式输出：

```
CODEX SAYS（咨询）：
════════════════════════════════════════════════════════════
<完整输出，逐字呈现，包含 [codex thinking] 轨迹>
════════════════════════════════════════════════════════════
Tokens: N | 预估成本：~$X.XX
Session saved — run /codex again to continue this conversation.
```

7. 呈现完毕后，指出 Codex 的分析与您自身理解存在差异的地方。
   如果有分歧，明确标出：
   “注意：Claude Code 在 X 上持不同意见，因为 Y。”

---

## 模型与推理

**Model：** 没有硬编码具体模型，codex 会使用它当前的默认模型（前沿
agentic coding model）。这意味着随着 OpenAI 发布更新模型，`/codex` 会自动
使用它们。如果用户想指定模型，则将 `-m` 透传给 codex。

**Reasoning effort：** 所有模式都使用 `xhigh`，也就是最高推理强度。无论是审查代码、破坏代码，还是咨询架构问题，你都希望模型尽可能深入思考。

**Web search：** 所有 codex 命令都使用 `--enable web_search_cached`，这样 Codex 就能在审查期间查阅
文档和 API。这是 OpenAI 的缓存索引，速度快且没有额外成本。

如果用户指定了模型（例如 `/codex review -m gpt-5.1-codex-max`
或 `/codex challenge -m gpt-5.2`），则将 `-m` flag 透传给 codex。

---

## 成本估算

从 stderr 中解析 token 数。Codex 会向 stderr 打印 `tokens used\nN`。

显示格式为：`Tokens: N`

如果拿不到 token 数，则显示：`Tokens: unknown`

---

## 错误处理

- **找不到二进制：** 在第 0 步检测。停止并给出安装说明。
- **认证错误：** Codex 会向 stderr 打印认证错误。直接向用户展示该错误：
  “Codex authentication failed. Run `codex login` in your terminal to authenticate via ChatGPT.”
- **超时：** 如果 Bash 调用超时（5 分钟），告诉用户：
  “Codex timed out after 5 minutes. The diff may be too large or the API may be slow. Try again or use a smaller scope.”
- **空响应：** 如果 `$TMPRESP` 为空或不存在，告诉用户：
  “Codex returned no response. Check stderr for errors.”
- **恢复会话失败：** 如果 resume 失败，删除 session 文件并重新开始。

---

## 重要规则

- **绝不修改文件。** 此技能是只读的。Codex 在只读沙箱模式下运行。
- **逐字呈现输出。** 在展示 Codex 输出之前，不要截断、总结或加入主观评论。
  必须在 CODEX SAYS 块中完整展示。
- **先展示，再综合。** 任何 Claude 的评论都必须放在完整输出之后，而不是替代完整输出。
- 对所有调用 codex 的 Bash 命令都设置 **5 分钟超时**（`timeout: 300000`）。
- **不要重复审查。** 如果用户已经运行过 `/review`，那么 Codex 提供的是第二个
  独立意见。不要重新运行 Claude Code 自己的 review。