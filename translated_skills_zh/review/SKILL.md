---
name: review
version: 1.0.0
description: |
  预落地 PR 审查。分析相对于基础分支的 diff，检查 SQL 安全性、LLM 信任
  边界违规、条件副作用以及其他结构性问题。适用于用户
  让你“review this PR”、“code review”、“pre-landing review”或“check my diff”时。
  当用户即将合并或落地代码变更时，主动建议使用。
allowed-tools:
  - Bash
  - Read
  - Edit
  - Write
  - Grep
  - Glob
  - Agent
  - AskUserQuestion
  - WebSearch
---
<!-- 从 SKILL.md.tmpl 自动生成 — 不要直接编辑 -->
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
echo '{"skill":"review","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","repo":"'$(basename "$(git rev-parse --show-toplevel 2>/dev/null)" 2>/dev/null || echo "unknown")'"}'  >> ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true
for _PF in ~/.gstack/analytics/.pending-*; do [ -f "$_PF" ] && ~/.claude/skills/gstack/bin/gstack-telemetry-log --event-type skill_run --skill _pending_finalize --outcome unknown --session-id "$_SESSION_ID" 2>/dev/null || true; break; done
```

如果 `PROACTIVE` 是 `"false"`，不要主动建议 gstack skills，只有在用户明确要求时才调用。
用户已选择退出主动建议。

如果输出显示 `UPGRADE_AVAILABLE <old> <new>`：读取 `~/.claude/skills/gstack/gstack-upgrade/SKILL.md` 并遵循 “Inline upgrade flow”（如果已配置则自动升级，否则使用 AskUserQuestion 提供 4 个选项；如果用户拒绝，则写入 snooze 状态）。如果是 `JUST_UPGRADED <from> <to>`：告诉用户 “Running gstack v{to} (just updated!)”，然后继续。

如果 `LAKE_INTRO` 是 `no`：继续前，先介绍 Completeness Principle。
告诉用户：“gstack 遵循 **Boil the Lake** 原则——当 AI 让边际成本接近零时，始终把事情完整做完。了解更多：https://garryslist.org/posts/boil-the-ocean”
然后询问是否要在默认浏览器中打开这篇文章：

```bash
open https://garryslist.org/posts/boil-the-ocean
touch ~/.gstack/.completeness-intro-seen
```

只有在用户同意时才运行 `open`。始终运行 `touch` 以标记为已看见。这只会发生一次。

如果 `TEL_PROMPTED` 是 `no` 且 `LAKE_INTRO` 是 `yes`：在处理完 lake 介绍后，
使用 AskUserQuestion 询问用户是否启用 telemetry：

> 帮助 gstack 变得更好！Community mode 会共享使用数据（你使用了哪些技能、耗时多久、
> 崩溃信息），并使用稳定的设备 ID，这样我们可以追踪趋势并更快修复 bug。
> 不会发送任何代码、文件路径或仓库名称。
> 可随时用 `gstack-config set telemetry off` 更改。

选项：
- A) 帮助 gstack 变得更好！（推荐）
- B) 不用了，谢谢

如果选 A：运行 `~/.claude/skills/gstack/bin/gstack-config set telemetry community`

如果选 B：继续使用 AskUserQuestion 追问：

> 那 anonymous mode 呢？我们只会知道*有人*使用了 gstack——没有唯一 ID，
> 也无法关联不同会话。只是一个计数器，用来帮助我们知道是否真的有人在使用。

选项：
- A) 可以，anonymous 就行
- B) 不用了，完全关闭

如果 B→A：运行 `~/.claude/skills/gstack/bin/gstack-config set telemetry anonymous`
如果 B→B：运行 `~/.claude/skills/gstack/bin/gstack-config set telemetry off`

始终运行：
```bash
touch ~/.gstack/.telemetry-prompted
```

这只会发生一次。如果 `TEL_PROMPTED` 是 `yes`，则完全跳过。

## AskUserQuestion 格式

**每次调用 AskUserQuestion 都必须遵循以下结构：**
1. **重新锚定：** 说明项目、当前分支（使用前言中打印的 `_BRANCH` 值——不要使用对话历史或 gitStatus 中的任何分支），以及当前计划/任务。（1-2 句）
2. **简化说明：** 用一个聪明的 16 岁青少年也能理解的通俗英语解释问题。不要出现原始函数名、内部术语或实现细节。使用具体示例和类比。说明它“做什么”，而不是“叫什么”。
3. **给出建议：** `RECOMMENDATION: Choose [X] because [one-line reason]` ——始终优先推荐完整选项而不是捷径（见 Completeness Principle）。为每个选项标注 `Completeness: X/10`。标定标准：10 = 完整实现（所有边界情况、完整覆盖），7 = 覆盖主路径但跳过部分边缘情况，3 = 推迟大量工作的捷径。如果两个选项都在 8+，选更高的；如果有一个 ≤5，要明确指出。
4. **选项：** 使用字母选项：`A) ... B) ... C) ...` ——当选项涉及工作量时，同时显示两种刻度：`(human: ~X / CC: ~Y)`

默认假设用户已经 20 分钟没看这个窗口，也没有打开代码。如果你的解释需要你自己再去读源码才能理解，那就太复杂了。

针对具体技能的说明可以在这个基础上额外增加格式规则。

## Completeness Principle — Boil the Lake

AI 辅助编码让“完整性”的边际成本接近于零。当你给出选项时：

- 如果选项 A 是完整实现（完全等价、覆盖所有边界情况、100% 覆盖），而选项 B 是省一点力气的捷径——**始终推荐 A**。在 CC+gstack 下，80 行和 150 行的差异毫无意义。当“完整”只多花几分钟时，“差不多就行”是错误直觉。
- **Lake 与 ocean：** “lake” 是能煮开的——模块 100% 测试覆盖、完整功能实现、处理所有边界情况、完整错误路径。“ocean” 不是——从头重写整个系统、给你无法控制的依赖加功能、跨多个季度的平台迁移。推荐煮开 lakes。对于 oceans，要指出超出范围。
- **估算工作量时，** 始终同时显示两种刻度：人工团队时间和 CC+gstack 时间。压缩比随任务类型变化——参考如下：

| 任务类型 | 人工团队 | CC+gstack | 压缩比 |
|-----------|-----------|-----------|-------------|
| 模板代码 / 脚手架 | 2 天 | 15 分钟 | ~100x |
| 编写测试 | 1 天 | 15 分钟 | ~50x |
| 功能实现 | 1 周 | 30 分钟 | ~30x |
| Bug 修复 + 回归测试 | 4 小时 | 15 分钟 | ~20x |
| 架构 / 设计 | 2 天 | 4 小时 | ~5x |
| 调研 / 探索 | 1 天 | 3 小时 | ~3x |

- 这个原则适用于测试覆盖、错误处理、文档、边界情况和功能完整性。不要为了“节省时间”而跳过最后 10%——有了 AI，这 10% 只需要几秒。

**反模式 —— 不要这样做：**
- 错误示例：“选 B——它用更少代码覆盖了 90% 的价值。”（如果 A 只多 70 行，就选 A。）
- 错误示例：“我们可以跳过边界情况处理来节省时间。”（有 CC 时，边界情况处理只要几分钟。）
- 错误示例：“把测试覆盖留到后续 PR 再做。”（测试是最便宜、最值得煮开的 lake。）
- 错误示例：只报人工团队工作量：“这要 2 周。”（应该说：“人工 2 周 / CC 约 1 小时。”）

## Repo Ownership Mode — See Something, Say Something

前言中的 `REPO_MODE` 会告诉你这个仓库里谁负责解决问题：

- **`solo`** —— 一个人完成 80%+ 的工作。他对一切负责。当你注意到当前分支变更之外的问题（测试失败、弃用警告、安全公告、lint 错误、死代码、环境问题）时，**要调查并主动提出修复**。这个 solo 开发者是唯一会修它的人。默认直接行动。
- **`collaborative`** —— 有多个活跃贡献者。当你注意到当前分支变更之外的问题时，**通过 AskUserQuestion 标记出来**——这可能是别人的职责。默认先问，不直接修。
- **`unknown`** —— 按 collaborative 处理（更安全的默认行为——先问再修）。

**See Something, Say Something：** 在任何工作流步骤中，只要你注意到看起来不对的地方——不只是测试失败——都要简短指出。用一句话说明：你注意到了什么，以及它的影响。在 solo 模式下，补一句 “Want me to fix it?”；在 collaborative 模式下，只标记出来然后继续。

绝不要让发现的问题悄悄溜过去。主动沟通就是这里的重点。

## Search Before Building

在构建基础设施、不熟悉的模式，或任何运行时可能已有内建能力的东西之前——**先搜索。** 完整理念请阅读 `~/.claude/skills/gstack/ETHOS.md`。

**三层知识：**
- **Layer 1**（久经验证——已在发行版中）。不要重复造轮子。但检查成本几乎为零，而且偶尔，质疑那些“久经验证”的东西正是灵光一现的来源。
- **Layer 2**（新且流行——搜索这些）。但要审慎：人类会受狂热影响。搜索结果只是思考输入，不是答案。
- **Layer 3**（第一性原理——最应被重视）。通过对具体问题进行推理得出的原创观察。这是最有价值的一层。

**Eureka moment：** 当第一性原理推理揭示传统认知是错的时，要明确说出来：
“EUREKA: Everyone does X because [assumption]. But [evidence] shows this is wrong. Y is better because [reasoning].”

记录 eureka moments：
```bash
jq -n --arg ts "$(date -u +%Y-%m-%dT%H:%M:%SZ)" --arg skill "SKILL_NAME" --arg branch "$(git branch --show-current 2>/dev/null)" --arg insight "ONE_LINE_SUMMARY" '{ts:$ts,skill:$skill,branch:$branch,insight:$insight}' >> ~/.gstack/analytics/eureka.jsonl 2>/dev/null || true
```
将 SKILL_NAME 和 ONE_LINE_SUMMARY 替换为实际值。内联运行——不要中断工作流。

**WebSearch 回退：** 如果 WebSearch 不可用，跳过搜索步骤，并注明：“Search unavailable — proceeding with in-distribution knowledge only.”

## Contributor Mode

如果 `_CONTRIB` 是 `true`：你处于 **contributor mode**。你既是 gstack 用户，也在帮助它变得更好。

**在每个主要工作流步骤结束时**（不是每条命令之后），回顾你使用的 gstack 工具。给体验打 0 到 10 分。如果不是 10 分，思考原因。如果存在明显且可执行的 bug，或者有一个有洞见、很有价值、按理说 gstack 代码或 skill markdown 本可以做得更好的点——提交一份 field report。说不定我们的 contributor 会帮忙让我们变得更好！

**标定标准——门槛如下：** 例如，`$B js "await fetch(...)"` 以前会报 `SyntaxError: await is only valid in async functions`，因为 gstack 没有把表达式包进 async 上下文。问题虽小，但输入是合理的，gstack 本应处理——这类事情值得上报。比这更轻微的，就忽略。

**不值得上报：** 用户应用自身的 bug、访问用户 URL 的网络错误、用户站点上的鉴权失败、用户自己的 JS 逻辑 bug。

**提交方式：** 写入 `~/.gstack/contributor-logs/{slug}.md`，并包含**以下所有章节**（不要截断——必须包含直到 Date/Version 页脚的所有部分）：

```
# {Title}

嘿，gstack 团队——我在使用 /{skill-name} 时遇到了这个问题：

**我当时想做什么：** {what the user/agent was attempting}
**实际发生了什么：** {what actually happened}
**我的评分：** {0-10} — {one sentence on why it wasn't a 10}

## 复现步骤
1. {step}

## 原始输出
```
{paste the actual error or unexpected output here}
```

## 怎样才能打到 10 分
{one sentence: what gstack should have done differently}

**日期：** {YYYY-MM-DD} | **版本：** {gstack version} | **技能：** /{skill}
```

Slug：小写、用连字符、最长 60 个字符（例如 `browse-js-no-await`）。如果文件已存在则跳过。每个会话最多 3 份报告。内联提交并继续——不要中断工作流。告诉用户：“Filed gstack field report: {title}”

## Completion Status Protocol

完成技能工作流时，使用以下状态之一汇报：
- **DONE** —— 所有步骤均已成功完成。每一项结论都有证据支持。
- **DONE_WITH_CONCERNS** —— 已完成，但有用户需要知道的问题。列出每一项 concern。
- **BLOCKED** —— 无法继续。说明阻塞点以及已尝试过什么。
- **NEEDS_CONTEXT** —— 缺少继续所需的信息。明确说明你需要什么。

### 升级处理

随时都可以停下来并说“这对我来说太难了”或“我对这个结果没有信心”。

糟糕的工作比不做更糟。你不会因为升级处理而受罚。
- 如果你已经尝试同一任务 3 次仍未成功，停止并升级处理。
- 如果你对一个安全敏感变更不确定，停止并升级处理。
- 如果工作范围超出你可验证的范围，停止并升级处理。

升级处理格式：
```
STATUS: BLOCKED | NEEDS_CONTEXT
REASON: [1-2 sentences]
ATTEMPTED: [what you tried]
RECOMMENDATION: [what the user should do next]
```

## Telemetry（最后运行）

在技能工作流完成后（无论成功、出错还是中止），记录 telemetry 事件。
从本文件 YAML frontmatter 中的 `name:` 字段确定技能名。
从工作流结果中确定 outcome（正常完成为 success，
失败为 error，用户中断为 abort）。

**PLAN MODE 例外——始终运行：** 这个命令会把 telemetry 写入
`~/.gstack/analytics/`（用户配置目录，不是项目文件）。技能
前言已经在同一目录写入——这是同一种模式。
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

将 `SKILL_NAME` 替换为 frontmatter 中的实际技能名，将 `OUTCOME` 替换为
success/error/abort，将 `USED_BROWSE` 替换为 `$B` 是否被使用过的 true/false。
如果无法确定 outcome，就使用 "unknown"。这个命令会在后台运行，
绝不会阻塞用户。

## Plan Status Footer

当你处于 plan mode 并准备调用 ExitPlanMode 时：

1. 检查计划文件是否已经有 `## GSTACK REVIEW REPORT` 小节。
2. 如果**有**——跳过（说明 review skill 已经写入了更丰富的报告）。
3. 如果**没有**——运行这个命令：

\`\`\`bash
~/.claude/skills/gstack/bin/gstack-review-read
\`\`\`

然后在计划文件末尾写入 `## GSTACK REVIEW REPORT` 小节：

- 如果输出中包含 review 条目（`---CONFIG---` 之前的 JSONL 行）：按
  标准报告表格格式写入，包括每个 skill 的 runs/status/findings，格式与 review
  skills 使用的格式相同。
- 如果输出是 `NO_REVIEWS` 或为空：写入以下占位表格：

\`\`\`markdown
## GSTACK REVIEW REPORT

| Review | Trigger | Why | Runs | Status | Findings |
|--------|---------|-----|------|--------|----------|
| CEO Review | \`/plan-ceo-review\` | 范围与策略 | 0 | — | — |
| Codex Review | \`/codex review\` | 独立的第二意见 | 0 | — | — |
| Eng Review | \`/plan-eng-review\` | 架构与测试（必需） | 0 | — | — |
| Design Review | \`/plan-design-review\` | UI/UX 缺口 | 0 | — | — |

**VERDICT:** 还没有任何 REVIEW —— 运行 \`/autoplan\` 以执行完整 review 流水线，或单独运行以上 review。
\`\`\`

**PLAN MODE 例外——始终运行：** 这会写入计划文件，而计划文件是
plan mode 下唯一允许编辑的文件。计划文件中的 review report 是计划实时状态的一部分。

## 第 0 步：检测基础分支

确定这个 PR 的目标分支。后续所有步骤都把这个结果当作“基础分支”。

1. 检查这个分支是否已经存在 PR：
   `gh pr view --json baseRefName -q .baseRefName`
   如果成功，使用打印出的分支名作为基础分支。

2. 如果不存在 PR（命令失败），检测仓库默认分支：
   `gh repo view --json defaultBranchRef -q .defaultBranchRef.name`

3. 如果两个命令都失败，则回退到 `main`。

打印检测到的基础分支名。在后续所有 `git diff`、`git log`、
`git fetch`、`git merge` 和 `gh pr create` 命令中，只要说明里写着“the base branch”，
都要替换为检测到的分支名。

---

# 预落地 PR 审查

你正在执行 `/review` 工作流。分析当前分支相对于基础分支的 diff，找出测试无法捕捉到的结构性问题。

---

## 第 1 步：检查分支

1. 运行 `git branch --show-current` 获取当前分支。
2. 如果当前就在基础分支上，输出：**"Nothing to review — you're on the base branch or have no changes against it."** 然后停止。
3. 运行 `git fetch origin <base> --quiet && git diff origin/<base> --stat` 检查是否有 diff。如果没有 diff，输出相同信息并停止。

---

## 第 1.5 步：范围漂移检测

在审查代码质量之前，先检查：**他们做的是否正是被要求的内容——不少也不多？**

1. 读取 `TODOS.md`（如果存在）。读取 PR 描述（`gh pr view --json body --jq .body 2>/dev/null || true`）。
   读取提交信息（`git log origin/<base>..HEAD --oneline`）。
   **如果不存在 PR：** 依赖提交信息和 TODOS.md 来判断声明的意图——这是常见情况，因为 /review 通常发生在 /ship 创建 PR 之前。
2. 识别**声明的意图**——这个分支原本应该完成什么？
3. 运行 `git diff origin/<base> --stat`，将变更文件与声明的意图进行比较。
4. 带着怀疑态度评估：

   **SCOPE CREEP 检测：**
   - 改动了与声明意图无关的文件
   - 计划中没有提到的新功能或重构
   - “既然都改到这里了……”式的变更，扩大了影响范围

   **MISSING REQUIREMENTS 检测：**
   - TODOS.md/PR 描述中的要求在 diff 中没有体现
   - 针对声明要求的测试覆盖缺口
   - 部分实现（开了头但没做完）

5. 输出（在主审查开始前）：
   ```
   Scope Check: [CLEAN / DRIFT DETECTED / REQUIREMENTS MISSING]
   Intent: <1-line summary of what was requested>
   Delivered: <1-line summary of what the diff actually does>
   [If drift: list each out-of-scope change]
   [If missing: list each unaddressed requirement]
   ```

6. 这是**信息性**检查——不会阻塞 review。继续执行第 2 步。

---

## 第 2 步：读取清单

读取 `.claude/skills/review/checklist.md`。

**如果无法读取该文件，停止并报告错误。** 没有清单就不要继续。

---

## 第 2.5 步：检查 Greptile review comments

读取 `.claude/skills/review/greptile-triage.md`，并按照其中的 fetch、filter、classify 和 **escalation detection** 步骤执行。

**如果不存在 PR、`gh` 失败、API 返回错误，或 Greptile comments 数量为零：** 静默跳过这一步。Greptile 集成是附加增强项——没有它 review 也能进行。

**如果发现 Greptile comments：** 保存分类结果（VALID & ACTIONABLE、VALID BUT ALREADY FIXED、FALSE POSITIVE、SUPPRESSED）——你会在第 5 步中用到它们。

---

## 第 3 步：获取 diff

拉取最新基础分支，避免因本地状态过旧而产生误报：

```bash
git fetch origin <base> --quiet
```

运行 `git diff origin/<base>` 获取完整 diff。这包含相对于最新基础分支的已提交和未提交变更。

---

## 第 4 步：两轮审查

针对 diff 按清单执行两轮审查：

1. **第 1 轮（CRITICAL）：** SQL & Data Safety、Race Conditions & Concurrency、LLM Output Trust Boundary、Enum & Value Completeness
2. **第 2 轮（INFORMATIONAL）：** Conditional Side Effects、Magic Numbers & String Coupling、Dead Code & Consistency、LLM Prompt Issues、Test Gaps、View/Frontend、Performance & Bundle Impact

**Enum & Value Completeness 需要读取 diff 之外的代码。** 当 diff 引入新的 enum value、status、tier 或 type constant 时，使用 Grep 找出所有引用同级值的文件，再读取这些文件，检查新值是否被处理。这是唯一一个仅看 diff 不够的类别。

**推荐前先搜索：** 当你推荐某种修复模式时（尤其是并发、缓存、鉴权或特定框架行为）：
- 确认该模式对当前使用的框架版本仍然是最佳实践
- 在推荐 workaround 之前，先检查新版本里是否已有内建方案
- 对照当前文档核对 API 签名（不同版本之间 API 会变化）

这只需几秒，却能避免推荐过时模式。如果 WebSearch 不可用，说明这一点，然后基于已有分布内知识继续。

遵循清单中规定的输出格式。尊重 suppressions——**不要**标记 “DO NOT flag” 小节里列出的项目。

---

## 第 4.5 步：Design Review（条件执行）

## Design Review（条件执行，限定 diff 范围）

使用 `gstack-diff-scope` 检查 diff 是否涉及前端文件：

```bash
source <(~/.claude/skills/gstack/bin/gstack-diff-scope <base> 2>/dev/null)
```

**如果 `SCOPE_FRONTEND=false`：** 静默跳过 design review。不输出内容。

**如果 `SCOPE_FRONTEND=true`：**

1. **检查 DESIGN.md。** 如果仓库根目录存在 `DESIGN.md` 或 `design-system.md`，读取它。所有设计类 findings 都要以它为基准进行判断——在 DESIGN.md 中被明确认可的模式不要标记。如果找不到，则使用通用设计原则。

2. **读取 `.claude/skills/review/design-checklist.md`。** 如果无法读取该文件，则跳过 design review，并注明：“Design checklist not found — skipping design review.”

3. **读取每个改动过的前端文件**（完整文件，而不只是 diff hunk）。前端文件由清单中的模式定义。

4. **对改动文件应用 design checklist**。对每一项：
   - **[HIGH] 机械式 CSS 修复**（`outline: none`、`!important`、`font-size < 16px`）：分类为 AUTO-FIX
   - **[HIGH/MEDIUM] 需要设计判断**：分类为 ASK
   - **[LOW] 基于意图的检测**：表述为 “Possible — verify visually or run /design-review”

5. **把 findings 纳入** review 输出中的 “Design Review” 标题下，并遵循清单中的输出格式。Design findings 会与代码 review findings 合并到同一个 Fix-First 流程中。

6. **记录结果**，用于 Review Readiness Dashboard：

```bash
~/.claude/skills/gstack/bin/gstack-review-log '{"skill":"design-review-lite","timestamp":"TIMESTAMP","status":"STATUS","findings":N,"auto_fixed":M,"commit":"COMMIT"}'
```

替换：TIMESTAMP = ISO 8601 datetime，STATUS = 若 0 个 findings 则为 `"clean"`，否则为 `"issues_found"`，N = findings 总数，M = 自动修复数量，COMMIT = `git rev-parse --short HEAD` 的输出。

7. **Codex design voice**（可选；若可用则自动执行）：

```bash
which codex 2>/dev/null && echo "CODEX_AVAILABLE" || echo "CODEX_NOT_AVAILABLE"
```

如果 Codex 可用，对 diff 运行一个轻量级设计检查：

```bash
TMPERR_DRL=$(mktemp /tmp/codex-drl-XXXXXXXX)
codex exec "Review the git diff on this branch. Run 7 litmus checks (YES/NO each): 1. Brand/product unmistakable in first screen? 2. One strong visual anchor present? 3. Page understandable by scanning headlines only? 4. Each section has one job? 5. Are cards actually necessary? 6. Does motion improve hierarchy or atmosphere? 7. Would design feel premium with all decorative shadows removed? Flag any hard rejections: 1. Generic SaaS card grid as first impression 2. Beautiful image with weak brand 3. Strong headline with no clear action 4. Busy imagery behind text 5. Sections repeating same mood statement 6. Carousel with no narrative purpose 7. App UI made of stacked cards instead of layout 5 most important design findings only. Reference file:line." -s read-only -c 'model_reasoning_effort="high"' --enable web_search_cached 2>"$TMPERR_DRL"
```

使用 5 分钟超时（`timeout: 300000`）。命令完成后，读取 stderr：
```bash
cat "$TMPERR_DRL" && rm -f "$TMPERR_DRL"
```

**错误处理：** 所有错误都不阻塞流程。遇到鉴权失败、超时或空响应时——简短说明并继续。

将 Codex 输出放在 `CODEX (design):` 标题下，并与上面的 checklist findings 合并展示。

将所有 design findings 与第 4 步和第 4.5 步中的 findings 一并纳入处理。它们在第 5 步中遵循同样的 Fix-First 流程——机械式 CSS 修复归为 AUTO-FIX，其余全部归为 ASK。

---

## 第 4.75 步：测试覆盖图

目标是 100% 覆盖。评估 diff 中变更的每一条代码路径并找出测试缺口。缺口会成为遵循 Fix-First 流程的 INFORMATIONAL findings。

### 测试框架检测

在分析覆盖率之前，先检测项目使用的测试框架：

1. **读取 CLAUDE.md** ——查找 `## Testing` 小节中的测试命令和框架名称。如果找到，以它作为权威来源。
2. **如果 CLAUDE.md 没有测试小节，则自动检测：**

```bash
# 检测项目运行时
[ -f Gemfile ] && echo "RUNTIME:ruby"
[ -f package.json ] && echo "RUNTIME:node"
[ -f requirements.txt ] || [ -f pyproject.toml ] && echo "RUNTIME:python"
[ -f go.mod ] && echo "RUNTIME:go"
[ -f Cargo.toml ] && echo "RUNTIME:rust"
# 检查现有测试基础设施
ls jest.config.* vitest.config.* playwright.config.* cypress.config.* .rspec pytest.ini phpunit.xml 2>/dev/null
ls -d test/ tests/ spec/ __tests__/ cypress/ e2e/ 2>/dev/null
```

3. **如果没有检测到框架：** 仍然输出覆盖图，但跳过测试生成。

**第 1 步：追踪 diff 中变更的每一条代码路径**，使用 `git diff origin/<base>...HEAD`：

读取每个变更文件。对每一个文件，追踪数据如何在代码中流动——不要只是列函数名，而要真正沿着执行路径走一遍：

1. **读取 diff。** 对每个变更文件，读取完整文件（不只是 diff hunk）以理解上下文。
2. **追踪数据流。** 从每个入口点（route handler、导出函数、事件监听器、组件渲染）开始，沿着每个分支追踪数据：
   - 输入来自哪里？（request params、props、database、API call）
   - 什么对它进行了转换？（validation、mapping、computation）
   - 它流向哪里？（database write、API response、rendered output、side effect）
   - 每一步可能出什么错？（null/undefined、invalid input、network failure、empty collection）
3. **绘制执行图。** 对每个变更文件，画出 ASCII 图，展示：
   - 每个新增或修改的函数/方法
   - 每个条件分支（if/else、switch、ternary、guard clause、early return）
   - 每条错误路径（try/catch、rescue、error boundary、fallback）
   - 每次对其他函数的调用（继续追进去——它里面是否也有未测试的分支？）
   - 每条边界：如果输入是 null 会怎样？空数组？无效类型？

这是关键步骤——你是在构建一张图，标出每一行会因输入不同而产生不同执行结果的代码。图中的每个分支都需要一个测试。

**第 2 步：映射用户流程、交互和错误状态：**

代码覆盖还不够——你还需要覆盖真实用户与变更代码的交互方式。对每个变更功能，完整想一遍：

- **用户流程：** 用户会通过什么动作序列触发这段代码？画出完整旅程（例如，“用户点击 ‘Pay’ → 表单校验 → API 调用 → 成功/失败页面”）。旅程中的每一步都需要测试。
- **交互边界情况：** 当用户做出意外操作时会怎样？
  - 双击/快速重复提交
  - 操作中途离开页面（返回按钮、关闭标签页、点击其他链接）
  - 使用过期数据提交（页面开着放了 30 分钟、会话已过期）
  - 网络很慢（API 需要 10 秒——用户会看到什么？）
  - 并发操作（两个标签页、同一个表单）
- **用户可见的错误状态：** 对代码处理的每一种错误，用户实际会经历什么？
  - 是有清晰报错信息，还是静默失败？
  - 用户能恢复吗（重试、返回、修正输入），还是会被卡住？
  - 没有网络时会怎样？API 返回 500 时呢？服务器返回无效数据时呢？
- **空/零/边界状态：** 零结果时 UI 展示什么？10,000 条结果时呢？单字符输入？最大长度输入？

把这些内容也加到图中，与代码分支一起展示。没有测试的用户流程，和未测试的 if/else 一样，都是缺口。

**第 3 步：把每个分支与现有测试逐一比对：**

按你的图一条条检查——既包括代码路径，也包括用户流程。对每一条，搜索是否存在覆盖它的测试：
- 函数 `processPayment()` → 查找 `billing.test.ts`、`billing.spec.ts`、`test/billing_test.rb`
- 一个 if/else → 查找是否同时覆盖了 true 和 false 两条路径
- 一个错误处理器 → 查找是否有测试能触发那个特定错误条件
- 一个对 `helperFn()` 的调用，而它本身又有分支 → 这些分支也需要测试
- 一个用户流程 → 查找是否有 integration 或 E2E 测试走完整条旅程
- 一个交互边界情况 → 查找是否有测试模拟该意外行为

质量评分规则：
- ★★★  测试了行为、边界情况和错误路径
- ★★   测试了正确行为，但仅覆盖主路径
- ★    烟雾测试 / 存在性检查 / 无关痛痒的断言（例如 “it renders”、“it doesn't throw”）

### E2E 测试决策矩阵

在检查每个分支时，也要判断应该使用单元测试还是 E2E/integration 测试：

**推荐 E2E（在图中标记为 [→E2E]）：**
- 跨越 3+ 个组件/服务的常见用户流程（例如 signup → verify email → first login）
- mock 会掩盖真实失败的集成点（例如 API → queue → worker → DB）
- auth/payment/data-destruction 流程——过于关键，不能只信任单元测试

**推荐 EVAL（在图中标记为 [→EVAL]）：**
- 需要做质量评估的关键 LLM 调用（例如提示词变更 → 测试输出仍满足质量标准）
- prompt templates、system instructions 或 tool definitions 的变更

**继续使用 UNIT TESTS：**
- 输入输出清晰的纯函数
- 没有副作用的内部 helper
- 单个函数的边界情况（null input、empty array）
- 不面向客户的冷门/少见流程

### REGRESSION RULE（强制）

**铁律：** 当覆盖审计识别出 REGRESSION——也就是以前能正常工作、但这次 diff 把它弄坏了——必须立刻写回归测试。不要 AskUserQuestion。不要跳过。回归测试是最高优先级，因为它能证明某个东西确实坏了。

以下情况属于 regression：
- diff 修改了已有行为（而不是新增代码）
- 现有测试套件（如果有）没有覆盖被修改的路径
- 这个改动给现有调用方引入了新的失败模式

如果不确定某个改动是否属于 regression，宁可写测试。

格式：提交信息写成 `test: regression test for {what broke}`

**第 4 步：输出 ASCII 覆盖图：**

在同一张图中同时包含代码路径和用户流程。标记出适合 E2E 和适合 eval 的路径：

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

**快速路径：** 所有路径都已覆盖 → “Step 4.75: All new code paths have test coverage ✓” 然后继续。

**第 5 步：为缺口生成测试（Fix-First）：**

如果检测到了测试框架，且识别出了缺口：
- 按 Fix-First Heuristic 将每个缺口分类为 AUTO-FIX 或 ASK：
  - **AUTO-FIX：** 纯函数的简单单元测试、已测试函数的边界情况
  - **ASK：** E2E 测试、需要新增测试基础设施的测试、行为定义模糊的测试
- 对 AUTO-FIX 缺口：生成测试，运行它，并以 `test: coverage for {feature}` 提交
- 对 ASK 缺口：把它们和其他 review findings 一起放进 Fix-First 批量问题中
- 标记为 [→E2E] 的路径：始终 ASK（E2E 测试工作量更大，需要用户确认）
- 标记为 [→EVAL] 的路径：始终 ASK（eval 测试需要用户确认质量标准）

如果未检测到测试框架 → 仅将缺口作为 INFORMATIONAL findings，不生成测试。

**如果 diff 仅包含测试变更：** 完全跳过第 4.75 步：`"No new application code paths to audit."`

这一步包含了第 2 轮中的 “Test Gaps” 类别——不要在 checklist 的 Test Gaps 项和这个 coverage diagram 之间重复报告 findings。将所有 coverage gaps 与第 4 步和第 4.5 步中的 findings 一起纳入。它们遵循同样的 Fix-First 流程——缺口属于 INFORMATIONAL findings。

---

## 第 5 步：Fix-First Review

**每个 finding 都要有动作——不只是 critical 的。**

输出一个摘要头：`Pre-Landing Review: N issues (X critical, Y informational)`

### 第 5a 步：分类每个 finding

对每个 finding，按照
checklist.md 中的 Fix-First Heuristic 分类为 AUTO-FIX 或 ASK。Critical findings 倾向于 ASK；informational findings 倾向于 AUTO-FIX。

### 第 5b 步：自动修复所有 AUTO-FIX 项

直接应用每一个修复。对每一项，输出一行摘要：
`[AUTO-FIXED] [file:line] Problem → what you did`

### 第 5c 步：批量询问 ASK 项

如果仍有 ASK 项，用**一次** AskUserQuestion 展示它们：

- 列出每一项的编号、严重级别标签、问题，以及推荐修复方式
- 对每一项提供选项：A) 按建议修复，B) 跳过
- 包含一个整体 RECOMMENDATION

示例格式：
```
I auto-fixed 5 issues. 2 need your input:

1. [CRITICAL] app/models/post.rb:42 — Race condition in status transition
   Fix: Add `WHERE status = 'draft'` to the UPDATE
   → A) Fix  B) Skip

2. [INFORMATIONAL] app/services/generator.rb:88 — LLM output not type-checked before DB write
   Fix: Add JSON schema validation
   → A) Fix  B) Skip

RECOMMENDATION: Fix both — #1 is a real race condition, #2 prevents silent data corruption.
```

如果 ASK 项不超过 3 个，你也可以分别使用单独的 AskUserQuestion，而不是批量询问。

### 第 5d 步：应用用户批准的修复

对用户选择 “Fix” 的项应用修复。输出已修复的内容。

如果没有 ASK 项（全部都是 AUTO-FIX），则完全跳过提问。

### 结论验证

在输出最终 review 结果之前：
- 如果你声称 “this pattern is safe” → 引用能证明安全的具体代码行
- 如果你声称 “this is handled elsewhere” → 读取并引用处理该情况的代码
- 如果你声称 “tests cover this” → 指明测试文件和测试方法
- 不要说 “likely handled” 或 “probably tested” ——要么验证，要么标记为未知

**防止合理化：** “This looks fine” 不能算 finding。要么引用证据证明它确实没问题，要么标记为未验证。

### Greptile comment resolution

在输出你自己的 findings 之后，如果第 2.5 步中对 Greptile comments 做了分类：

**在输出头中加入 Greptile 摘要：** `+ N Greptile comments (X valid, Y fixed, Z FP)`

在回复任何 comment 之前，先运行 greptile-triage.md 中的 **Escalation Detection** 算法，以确定应该使用 Tier 1（友好）还是 Tier 2（强硬）回复模板。

1. **VALID & ACTIONABLE comments：** 把它们纳入你的 findings——遵循 Fix-First 流程（机械类问题自动修；非机械类则纳入 ASK 批量提问）(A: 现在修，B: 确认，C: False positive)。如果用户选择 A（修复），使用 greptile-triage.md 中的 **Fix reply template** 回复（包含 inline diff + explanation）。如果用户选择 C（误报），使用 **False Positive reply template** 回复（包含证据 + 建议重新排序），并同时保存到 per-project 和全局 greptile-history。

2. **FALSE POSITIVE comments：** 通过 AskUserQuestion 单独展示每一项：
   - 展示 Greptile comment：file:line（或 [top-level]）+ body 摘要 + permalink URL
   - 简要说明为什么这是误报
   - 选项：
     - A) 回复 Greptile，解释为什么这是错误的（如果明显错误，推荐）
     - B) 仍然修复它（如果成本低且无害）
     - C) 忽略——不回复，也不修

   如果用户选择 A，使用 greptile-triage.md 中的 **False Positive reply template** 回复（包含证据 + 建议重新排序），并同时保存到 per-project 和全局 greptile-history。

3. **VALID BUT ALREADY FIXED comments：** 使用 greptile-triage.md 中的 **Already Fixed reply template** 回复——不需要 AskUserQuestion：
   - 包含具体做了什么以及修复该问题的 commit SHA
   - 同时保存到 per-project 和全局 greptile-history

4. **SUPPRESSED comments：** 静默跳过——这些是之前 triage 中确认过的已知误报。

---

## 第 5.5 步：TODOS 交叉引用

读取仓库根目录中的 `TODOS.md`（如果存在）。将 PR 与未完成 TODO 交叉比对：

- **这个 PR 是否关闭了任何未完成 TODO？** 如果是，在输出中注明对应条目：“This PR addresses TODO: <title>”
- **这个 PR 是否产生了应该写成 TODO 的后续工作？** 如果是，将其标记为 informational finding。
- **是否存在能为本次 review 提供上下文的相关 TODO？** 如果有，在讨论相关 findings 时引用它们。

如果不存在 TODOS.md，则静默跳过这一步。

---

## 第 5.6 步：文档过时检查

将 diff 与文档文件交叉比对。对仓库根目录中的每个 `.md` 文件（README.md、ARCHITECTURE.md、CONTRIBUTING.md、CLAUDE.md 等）：

1. 检查 diff 中的代码变更是否影响了该文档里描述的功能、组件或工作流。
2. 如果该文档文件在当前分支中**没有更新**，但它所描述的代码**发生了变更**，则将其标记为 INFORMATIONAL finding：
   `"Documentation may be stale: [file] describes [feature/component] but code changed in this branch. Consider running `/document-release`."`

这只是信息性提示——绝不属于 critical。对应的修复动作是 `/document-release`。

如果不存在任何文档文件，则静默跳过这一步。

---

## 第 5.7 步：对抗性审查（自动伸缩）

对抗性审查的深入程度会根据 diff 大小自动伸缩。无需任何配置。

**检测 diff 大小和工具可用性：**

```bash
DIFF_INS=$(git diff origin/<base> --stat | tail -1 | grep -oE '[0-9]+ insertion' | grep -oE '[0-9]+' || echo "0")
DIFF_DEL=$(git diff origin/<base> --stat | tail -1 | grep -oE '[0-9]+ deletion' | grep -oE '[0-9]+' || echo "0")
DIFF_TOTAL=$((DIFF_INS + DIFF_DEL))
which codex 2>/dev/null && echo "CODEX_AVAILABLE" || echo "CODEX_NOT_AVAILABLE"
# 尊重旧的退出配置
OLD_CFG=$(~/.claude/skills/gstack/bin/gstack-config get codex_reviews 2>/dev/null || true)
echo "DIFF_SIZE: $DIFF_TOTAL"
echo "OLD_CFG: ${OLD_CFG:-not_set}"
```

如果 `OLD_CFG` 是 `disabled`：静默跳过这一步。继续下一步。

**用户覆盖：** 如果用户明确要求某个特定级别（例如 “run all passes”、“paranoid review”、“full adversarial”、“do all 4 passes”、“thorough review”），无论 diff 大小如何，都要遵从。直接跳到对应级别部分。

**根据 diff 大小自动选择级别：**
- **小型（改动 < 50 行）：** 完全跳过对抗性审查。输出：“Small diff ($DIFF_TOTAL lines) — adversarial review skipped.” 然后继续下一步。
- **中型（50–199 行）：** 运行 Codex adversarial challenge（如果 Codex 不可用，则使用 Claude adversarial subagent）。跳到“Medium tier”部分。
- **大型（200+ 行）：** 运行所有剩余轮次——Codex structured review + Claude adversarial subagent + Codex adversarial。跳到“Large tier”部分。

---

### Medium tier（50–199 行）

Claude 的 structured review 已经运行过。现在增加一个**跨模型对抗性挑战**。

**如果 Codex 可用：** 运行 Codex adversarial challenge。**如果 Codex 不可用：** 自动回退到 Claude adversarial subagent。

**Codex adversarial：**

```bash
TMPERR_ADV=$(mktemp /tmp/codex-adv-XXXXXXXX)
codex exec "Review the changes on this branch against the base branch. Run git diff origin/<base> to see the diff. Your job is to find ways this code will fail in production. Think like an attacker and a chaos engineer. Find edge cases, race conditions, security holes, resource leaks, failure modes, and silent data corruption paths. Be adversarial. Be thorough. No compliments — just the problems." -s read-only -c 'model_reasoning_effort="xhigh"' --enable web_search_cached 2>"$TMPERR_ADV"
```

将 Bash 工具的 `timeout` 参数设为 `300000`（5 分钟）。**不要**使用 `timeout` shell 命令——macOS 上没有它。命令完成后，读取 stderr：
```bash
cat "$TMPERR_ADV"
```

将完整输出原样展示。这是信息性内容——不会阻止 ship。

**错误处理：** 所有错误都不阻塞——对抗性审查是质量增强项，不是前置条件。
- **鉴权失败：** 如果 stderr 包含 "auth"、"login"、"unauthorized" 或 "API key"：输出 “Codex authentication failed. Run \`codex login\` to authenticate.”
- **超时：** 输出 “Codex timed out after 5 minutes.”
- **空响应：** 输出 “Codex returned no response. Stderr: <paste relevant error>.”

遇到任何 Codex 错误时，自动回退到 Claude adversarial subagent。

**Claude adversarial subagent**（Codex 不可用或报错时的回退）：

通过 Agent 工具派发。该 subagent 拥有全新上下文——不带 structured review 的 checklist 偏见。这种真正的独立性有助于发现主审查者看不到的问题。

Subagent prompt：
"Read the diff for this branch with `git diff origin/<base>`. Think like an attacker and a chaos engineer. Your job is to find ways this code will fail in production. Look for: edge cases, race conditions, security holes, resource leaks, failure modes, silent data corruption, logic errors that produce wrong results silently, error handling that swallows failures, and trust boundary violations. Be adversarial. Be thorough. No compliments — just the problems. For each finding, classify as FIXABLE (you know how to fix it) or INVESTIGATE (needs human judgment)."

将 findings 放在 `ADVERSARIAL REVIEW (Claude subagent):` 标题下展示。**FIXABLE findings** 会进入与 structured review 相同的 Fix-First 流水线。**INVESTIGATE findings** 作为信息性内容展示。

如果 subagent 失败或超时：输出 “Claude adversarial subagent unavailable. Continuing without adversarial review.”

**持久化 review 结果：**
```bash
~/.claude/skills/gstack/bin/gstack-review-log '{"skill":"adversarial-review","timestamp":"'"$(date -u +%Y-%m-%dT%H:%M:%SZ)"'","status":"STATUS","source":"SOURCE","tier":"medium","commit":"'"$(git rev-parse --short HEAD)"'"}'
```
替换 STATUS：如果没有 findings 则为 `"clean"`，否则为 `"issues_found"`。SOURCE：如果运行了 Codex，则为 `"codex"`；如果运行了 subagent，则为 `"claude"`。如果两者都失败，则**不要**持久化。

**清理：** 处理完成后运行 `rm -f "$TMPERR_ADV"`（如果使用了 Codex）。

---

### Large tier（200+ 行）

Claude 的 structured review 已经运行过。现在运行**剩余三个轮次全部执行**，以获得最大覆盖：

**1. Codex structured review（如果可用）：**
```bash
TMPERR=$(mktemp /tmp/codex-review-XXXXXXXX)
codex review --base <base> -c 'model_reasoning_effort="xhigh"' --enable web_search_cached 2>"$TMPERR"
```

将 Bash 工具的 `timeout` 参数设为 `300000`（5 分钟）。**不要**使用 `timeout` shell 命令——macOS 上没有它。将输出放在 `CODEX SAYS (code review):` 标题下展示。
检查是否存在 `[P1]` 标记：存在 → `GATE: FAIL`，不存在 → `GATE: PASS`。

如果 GATE 是 FAIL，使用 AskUserQuestion：
```
Codex found N critical issues in the diff.

A) Investigate and fix now (recommended)
B) Continue — review will still complete
```

如果选 A：处理这些 findings。然后重新运行 `codex review` 进行验证。

读取 stderr 中的错误（错误处理与 medium tier 相同）。

处理 stderr 后：`rm -f "$TMPERR"`

**2. Claude adversarial subagent：** 派发一个 adversarial prompt 的 subagent（与 medium tier 相同的 prompt）。无论 Codex 是否可用，这一步都要运行。

**3. Codex adversarial challenge（如果可用）：** 用 adversarial prompt 运行 `codex exec`（与 medium tier 相同）。

如果第 1 步和第 3 步无法使用 Codex，要向用户说明：“Codex CLI not found — large-diff review ran Claude structured + Claude adversarial (2 of 4 passes). Install Codex for full 4-pass coverage: `npm install -g @openai/codex`”

**在所有轮次完成后再持久化 review 结果**（而不是每个子步骤之后）：
```bash
~/.claude/skills/gstack/bin/gstack-review-log '{"skill":"adversarial-review","timestamp":"'"$(date -u +%Y-%m-%dT%H:%M:%SZ)"'","status":"STATUS","source":"SOURCE","tier":"large","gate":"GATE","commit":"'"$(git rev-parse --short HEAD)"'"}'
```
替换：STATUS = 如果**所有轮次**都没有 findings，则为 `"clean"`；只要任意一轮发现问题，则为 `"issues_found"`。SOURCE = 如果运行了 Codex，则为 `"both"`；如果只有 Claude subagent 运行，则为 `"claude"`。GATE = Codex structured review 的 gate 结果（`"pass"`/`"fail"`），如果 Codex 不可用，则为 `"informational"`。如果所有轮次都失败，则**不要**持久化。

---

### 跨模型综合（medium 和 large tiers）

所有轮次完成后，对所有来源的 findings 进行综合：

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

高置信度 findings（被多个来源同时发现）应优先修复。

---

## 重要规则

- **评论之前先读完整 diff。** 不要标记那些已经在 diff 中解决的问题。
- **先修复，不做只读审查。** AUTO-FIX 项直接应用。ASK 项只有在用户批准后才应用。绝不要 commit、push 或创建 PR——那是 /ship 的职责。
- **保持简洁。** 一行写问题，一行写修复。不要写前言。
- **只标记真实问题。** 没问题的内容跳过。
- **使用 greptile-triage.md 中的 Greptile reply templates。** 每条回复都必须包含证据。绝不要发布模糊回复。