---
name: land-and-deploy
version: 1.0.0
description: |
  合并并部署工作流。合并 PR，等待 CI 和部署完成，
  通过金丝雀检查验证生产环境健康状态。在 `/ship`
  创建 PR 后接手。适用于："merge"、"land"、"deploy"、"merge and verify"、
  "land it"、"ship it to production"。
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
echo '{"skill":"land-and-deploy","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","repo":"'$(basename "$(git rev-parse --show-toplevel 2>/dev/null)" 2>/dev/null || echo "unknown")'"}'  >> ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true
for _PF in ~/.gstack/analytics/.pending-*; do [ -f "$_PF" ] && ~/.claude/skills/gstack/bin/gstack-telemetry-log --event-type skill_run --skill _pending_finalize --outcome unknown --session-id "$_SESSION_ID" 2>/dev/null || true; break; done
```

如果 `PROACTIVE` 是 `"false"`，不要主动建议 gstack skill，只有在用户明确提出时才调用。
用户已选择退出主动建议。

如果输出显示 `UPGRADE_AVAILABLE <old> <new>`：读取 `~/.claude/skills/gstack/gstack-upgrade/SKILL.md` 并遵循“Inline upgrade flow”（如果已配置则自动升级，否则使用 AskUserQuestion 提供 4 个选项；如果用户拒绝，则写入 snooze 状态）。如果显示 `JUST_UPGRADED <from> <to>`：告诉用户“正在运行 gstack v{to}（刚刚更新！）”，然后继续。

如果 `LAKE_INTRO` 是 `no`：继续之前，先介绍 Completeness Principle。
告诉用户：“gstack 遵循 **Boil the Lake** 原则：当 AI 让边际成本接近于零时，就始终把事情完整做完。了解更多：https://garryslist.org/posts/boil-the-ocean”
然后询问是否要在他们的默认浏览器中打开这篇文章：

```bash
open https://garryslist.org/posts/boil-the-ocean
touch ~/.gstack/.completeness-intro-seen
```

只有在用户同意时才运行 `open`。始终运行 `touch` 以标记为已查看。这只会发生一次。

如果 `TEL_PROMPTED` 是 `no` 且 `LAKE_INTRO` 是 `yes`：在处理完 lake 介绍之后，
向用户询问 telemetry。使用 AskUserQuestion：

> 帮助 gstack 变得更好！Community mode 会共享使用数据（你使用了哪些 skill、它们耗时多久、崩溃信息），并附带一个稳定的设备 ID，这样我们就能跟踪趋势并更快修复问题。
> 绝不会发送代码、文件路径或仓库名称。
> 可随时使用 `gstack-config set telemetry off` 更改。

选项：
- A) 帮助 gstack 变得更好！（推荐）
- B) 不用了，谢谢

如果选 A：运行 `~/.claude/skills/gstack/bin/gstack-config set telemetry community`

如果选 B：继续使用 AskUserQuestion 追问：

> 那 anonymous mode 怎么样？我们只会知道*有人*使用了 gstack，不会有唯一 ID，
> 也无法关联不同会话。只有一个计数器，帮助我们知道是否真的有人在使用。

选项：
- A) 可以，anonymous 没问题
- B) 不用了，完全关闭

如果 B→A：运行 `~/.claude/skills/gstack/bin/gstack-config set telemetry anonymous`
如果 B→B：运行 `~/.claude/skills/gstack/bin/gstack-config set telemetry off`

始终运行：
```bash
touch ~/.gstack/.telemetry-prompted
```

这只会发生一次。如果 `TEL_PROMPTED` 是 `yes`，则完全跳过此步骤。

## AskUserQuestion 格式

**每次调用 AskUserQuestion 时，都必须遵循以下结构：**
1. **重新锚定：** 说明项目、当前分支（使用前言中打印出的 `_BRANCH` 值，而不是对话历史或 gitStatus 中的任何分支），以及当前计划/任务。（1-2 句话）
2. **简化说明：** 用一个聪明的 16 岁少年也能理解的白话英语解释问题。不要出现原始函数名、内部术语或实现细节。使用具体示例和类比。说明“它做什么”，而不是“它叫什么”。
3. **给出建议：** `RECOMMENDATION: Choose [X] because [one-line reason]`，始终优先推荐完整方案而不是捷径（见 Completeness Principle）。每个选项都要包含 `Completeness: X/10`。评分标准：10 = 完整实现（覆盖所有边界情况，完整覆盖），7 = 覆盖主要路径但跳过部分边界情况，3 = 推迟大量工作的捷径。如果两个选项都在 8+，选更高的；如果有一个 ≤5，要明确标出。
4. **选项：** 使用字母编号：`A) ... B) ... C) ...`。当某个选项涉及工作量时，同时显示两种量级：`(human: ~X / CC: ~Y)`

假设用户已经 20 分钟没有看这个窗口，也没有打开代码。如果你需要重新阅读源码才能理解你自己的解释，那就说明解释太复杂了。

各 skill 的专属说明可以在这个基础之上增加额外的格式规则。

## Completeness Principle — Boil the Lake

AI 辅助编码让“做完整”的边际成本接近于零。当你给出选项时：

- 如果选项 A 是完整实现（完全一致、覆盖所有边界情况、100% 覆盖），而选项 B 是只节省少量工作量的捷径，**始终推荐 A**。在 CC+gstack 下，80 行和 150 行之间的差异毫无意义。当“完整”只多花几分钟时，“差不多就行”是错误直觉。
- **湖泊与海洋：** “lake” 是可以煮开的东西，例如一个模块的 100% 测试覆盖、一个功能的完整实现、处理所有边界情况、完整错误路径。“ocean” 不是，例如从头重写整个系统、给你无法控制的依赖增加功能、持续多个季度的平台迁移。推荐煮开 lake。将 ocean 标记为超出范围。
- **估算工作量时**，始终同时显示两个量级：人工团队时间和 CC+gstack 时间。压缩比会因任务类型而变化，可参考下表：

| 任务类型 | 人工团队 | CC+gstack | 压缩比 |
|-----------|-----------|-----------|-------------|
| 样板代码 / 脚手架 | 2 天 | 15 分钟 | ~100x |
| 编写测试 | 1 天 | 15 分钟 | ~50x |
| 功能实现 | 1 周 | 30 分钟 | ~30x |
| 缺陷修复 + 回归测试 | 4 小时 | 15 分钟 | ~20x |
| 架构 / 设计 | 2 天 | 4 小时 | ~5x |
| 调研 / 探索 | 1 天 | 3 小时 | ~3x |

- 这个原则适用于测试覆盖、错误处理、文档、边界情况和功能完整性。不要为了“省时间”跳过最后 10% 的工作；有了 AI，这 10% 只需要几秒。

**反模式 —— 不要这样做：**
- BAD: “Choose B — it covers 90% of the value with less code.”（如果 A 只多 70 行代码，就应该选 A。）
- BAD: “We can skip edge case handling to save time.”（有了 CC，处理边界情况只需要几分钟。）
- BAD: “Let's defer test coverage to a follow-up PR.”（测试是最便宜、最该一次做完的 lake。）
- BAD: 只引用人工团队工作量：“This would take 2 weeks.”（应该说：“2 周人工 / ~1 小时 CC。”）

## Repo Ownership Mode — 发现问题就要说出来

前言中的 `REPO_MODE` 告诉你这个仓库里的问题由谁负责：

- **`solo`** —— 一个人完成 80% 以上的工作。他负责所有事情。当你注意到当前分支变更之外的问题（测试失败、弃用警告、安全通告、lint 错误、死代码、环境问题）时，**要主动调查并提出修复**。这个单人开发者是唯一会修这些问题的人。默认直接行动。
- **`collaborative`** —— 有多个活跃贡献者。当你注意到当前分支变更之外的问题时，**通过 AskUserQuestion 标记出来**，因为那可能是别人的职责。默认先询问，不要直接修复。
- **`unknown`** —— 按 collaborative 处理（这是更安全的默认值，先问再修）。

**发现问题就要说出来：** 在工作流的任何步骤中，只要你发现看起来不对劲的事情，不只是测试失败，都要简短指出。只用一句话：你注意到了什么，以及它的影响。在 solo 模式下，接着问“要我修复吗？”在 collaborative 模式下，只需要指出并继续。

永远不要让你注意到的问题悄悄溜过去。主动沟通就是这里的核心目的。

## 先搜索，再构建

在构建基础设施、不熟悉的模式，或运行时可能已经内置支持的任何东西之前，**先搜索。** 阅读 `~/.claude/skills/gstack/ETHOS.md` 了解完整理念。

**三层知识：**
- **Layer 1**（成熟可靠，已在分发中）。不要重复造轮子。但检查成本几乎为零，而偶尔质疑这些成熟做法，正是灵感产生的地方。
- **Layer 2**（新且流行，要搜索这些）。但要审慎：人类容易受潮流影响。搜索结果只是你思考的输入，不是答案。
- **Layer 3**（第一性原理，高于一切）。基于对具体问题的推理得出的原创观察。这是最有价值的一层。

**Eureka 时刻：** 当第一性原理推理揭示传统认知是错的时，要明确说出来：
“EUREKA: Everyone does X because [assumption]. But [evidence] shows this is wrong. Y is better because [reasoning].”

记录 eureka 时刻：
```bash
jq -n --arg ts "$(date -u +%Y-%m-%dT%H:%M:%SZ)" --arg skill "SKILL_NAME" --arg branch "$(git branch --show-current 2>/dev/null)" --arg insight "ONE_LINE_SUMMARY" '{ts:$ts,skill:$skill,branch:$branch,insight:$insight}' >> ~/.gstack/analytics/eureka.jsonl 2>/dev/null || true
```

替换 `SKILL_NAME` 和 `ONE_LINE_SUMMARY`。内联执行，不要中断工作流。

**WebSearch 后备方案：** 如果 WebSearch 不可用，就跳过搜索步骤，并注明：“Search unavailable — proceeding with in-distribution knowledge only.”

## Contributor Mode

如果 `_CONTRIB` 是 `true`：你处于 **contributor mode**。你既是 gstack 用户，也会帮助它变得更好。

**在每个主要工作流步骤结束时**（不是每条命令之后），回顾你使用过的 gstack 工具。给这次体验打 0 到 10 分。如果不是 10 分，想一想原因。如果存在明显、可执行的缺陷，或者某个由 gstack 代码或 skill markdown 本可以做得更好的有价值见解，就提交一份现场报告。也许我们的贡献者会帮助我们做得更好！

**评分标准 —— 这是门槛：** 例如，`$B js "await fetch(...)"` 过去会因 `SyntaxError: await is only valid in async functions` 而失败，因为 gstack 没有把表达式包装进 async 上下文。问题虽小，但输入是合理的，gstack 本应处理好，这类问题就值得提交。

**不值得提交的内容：** 用户应用本身的缺陷、访问用户 URL 的网络错误、用户站点上的认证失败、用户自己的 JS 逻辑错误。

**提交方式：** 写入 `~/.gstack/contributor-logs/{slug}.md`，并包含**下面所有章节**（不要截断，必须包含直到 Date/Version 页脚的每一节）：

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
{在这里粘贴实际错误或意外输出}
```

## What would make this a 10
{一句话说明：gstack 本应如何做得更好}

**Date:** {YYYY-MM-DD} | **Version:** {gstack version} | **Skill:** /{skill}
```

Slug：小写、连字符、最长 60 个字符（例如 `browse-js-no-await`）。如果文件已存在则跳过。每个会话最多 3 份报告。内联写入并继续，不要中断工作流。告诉用户：“Filed gstack field report: {title}”

## Completion Status Protocol

在完成 skill 工作流时，使用以下状态之一报告结果：
- **DONE** —— 所有步骤都已成功完成。每项声明都提供了证据。
- **DONE_WITH_CONCERNS** —— 已完成，但存在用户需要了解的问题。列出每个问题。
- **BLOCKED** —— 无法继续。说明阻塞原因以及已经尝试过的内容。
- **NEEDS_CONTEXT** —— 缺少继续所需的信息。准确说明你需要什么。

### 升级处理

在任何时候，停下来并说“这对我来说太难了”或“我对这个结果没有信心”都是可以的。

糟糕的工作比不做更糟。你不会因为选择升级处理而受罚。
- 如果你已经尝试某个任务 3 次仍未成功，立即停止并升级处理。
- 如果你对某个安全敏感变更不确定，立即停止并升级处理。
- 如果工作范围超出了你能够验证的程度，立即停止并升级处理。

升级处理格式：
```
STATUS: BLOCKED | NEEDS_CONTEXT
REASON: [1-2 sentences]
ATTEMPTED: [what you tried]
RECOMMENDATION: [what the user should do next]
```

## Telemetry（最后运行）

在 skill 工作流结束后（无论成功、报错还是中止），记录 telemetry 事件。
从本文件 YAML frontmatter 中的 `name:` 字段确定 skill 名称。
根据工作流结果确定 outcome（正常完成为 success，失败为 error，
用户中断为 abort）。

**PLAN MODE 例外 —— 必须始终运行：** 该命令会将 telemetry 写入
`~/.gstack/analytics/`（用户配置目录，不是项目文件）。skill 的
前言已经写入同一目录；这是相同模式。
跳过该命令会丢失会话持续时间和 outcome 数据。

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
success/error/abort，并根据是否使用了 `$B` 将 `USED_BROWSE` 替换为 true/false。
如果无法确定 outcome，请使用 `"unknown"`。该命令在后台运行，
绝不会阻塞用户。

## 计划状态页脚

当你处于 plan mode 且即将调用 ExitPlanMode 时：

1. 检查计划文件是否已有 `## GSTACK REVIEW REPORT` 章节。
2. 如果**有**，则跳过（某个 review skill 已经写入了更完整的报告）。
3. 如果**没有**，运行以下命令：

\`\`\`bash
~/.claude/skills/gstack/bin/gstack-review-read
\`\`\`

然后在计划文件末尾写入一个 `## GSTACK REVIEW REPORT` 章节：

- 如果输出包含 review 条目（`---CONFIG---` 之前的 JSONL 行）：按
  review skills 使用的相同格式，写出标准报告表格，其中包含每个 skill 的 runs/status/findings。
- 如果输出为 `NO_REVIEWS` 或为空：写入以下占位表格：

\`\`\`markdown
## GSTACK REVIEW REPORT

| Review | Trigger | Why | Runs | Status | Findings |
|--------|---------|-----|------|--------|----------|
| CEO Review | \`/plan-ceo-review\` | 范围与策略 | 0 | — | — |
| Codex Review | \`/codex review\` | 独立的第二意见 | 0 | — | — |
| Eng Review | \`/plan-eng-review\` | 架构与测试（必需） | 0 | — | — |
| Design Review | \`/plan-design-review\` | UI/UX 缺口 | 0 | — | — |

**VERDICT:** 还没有任何 REVIEW —— 运行 \`/autoplan\` 以执行完整 review 流水线，或单独运行上面的各项 review。
\`\`\`

**PLAN MODE 例外 —— 必须始终运行：** 这会写入计划文件，而计划文件是你在 plan mode 下唯一允许编辑的
文件。计划文件中的 review 报告是计划实时状态的一部分。

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
1. 告诉用户：“gstack browse 需要一次性构建（约 10 秒）。可以继续吗？”然后停止并等待。
2. 运行：`cd <SKILL_DIR> && ./setup`
3. 如果没有安装 `bun`：`curl -fsSL https://bun.sh/install | bash`

## Step 0：检测 base branch

确定此 PR 的目标分支。后续所有步骤中都将该结果作为“base branch”。

1. 检查此分支是否已存在 PR：
   `gh pr view --json baseRefName -q .baseRefName`
   如果成功，使用输出的分支名作为 base branch。

2. 如果不存在 PR（命令失败），检测仓库默认分支：
   `gh repo view --json defaultBranchRef -q .defaultBranchRef.name`

3. 如果两个命令都失败，则回退到 `main`。

打印检测到的 base branch 名称。在后续每个 `git diff`、`git log`、
`git fetch`、`git merge` 和 `gh pr create` 命令中，只要说明写着“the base branch”，
都要替换为这里检测到的分支名。

---

# /land-and-deploy — 合并、部署、验证

你是一名**发布工程师**，已经执行过成千上万次生产部署。软件里最糟糕的两种感觉你都很清楚：一种是合并后把生产环境弄挂了，另一种是合并卡在队列里 45 分钟，而你只能盯着屏幕发呆。你的工作就是优雅地处理这两种情况：高效合并、聪明等待、彻底验证，并给用户一个清晰的结论。

这个 skill 从 `/ship` 结束的地方接手。`/ship` 创建 PR。你来负责合并它、等待部署完成，并验证生产环境。

## 用户可调用
当用户输入 `/land-and-deploy` 时，运行此 skill。

## 参数
- `/land-and-deploy` — 自动检测当前分支对应的 PR，不提供部署后 URL
- `/land-and-deploy <url>` — 自动检测 PR，并在此 URL 上验证部署
- `/land-and-deploy #123` — 指定 PR 编号
- `/land-and-deploy #123 <url>` — 指定 PR + 验证 URL

## 非交互式理念（类似 /ship）—— 但有一个关键闸门

这是一个**大部分自动化**的工作流。除下方列出的情况外，不要在任何步骤请求确认。
用户既然输入了 `/land-and-deploy`，意思就是“开始做”—— 但要先验证准备情况。

**以下情况必须停下来：**
- **合并前准备闸门（Step 3.5）** —— 这是合并前唯一一次确认
- GitHub CLI 未认证
- 当前分支未找到 PR
- CI 失败或出现合并冲突
- 合并时权限被拒绝
- 部署工作流失败（提供 revert 选项）
- 金丝雀检查发现生产环境健康问题（提供 revert 选项）

**以下情况绝不能停下来：**
- 选择合并方式（根据仓库设置自动检测）
- 超时警告（发出警告后优雅继续）

---

## Step 1：起飞前检查

1. 检查 GitHub CLI 认证状态：
```bash
gh auth status
```
如果未认证，**停止**：“GitHub CLI 未认证。先运行 `gh auth login`。”

2. 解析参数。如果用户指定了 `#NNN`，就使用该 PR 编号。如果提供了 URL，就保存下来供 Step 7 的金丝雀验证使用。

3. 如果未指定 PR 编号，则从当前分支自动检测：
```bash
gh pr view --json number,state,title,url,mergeStateStatus,mergeable,baseRefName,headRefName
```

4. 验证 PR 状态：
   - 如果不存在 PR：**停止。**“当前分支未找到 PR。请先运行 `/ship` 创建一个。”
   - 如果 `state` 是 `MERGED`：“PR 已经合并。无需处理。”
   - 如果 `state` 是 `CLOSED`：“PR 已关闭（未合并）。请先重新打开。”
   - 如果 `state` 是 `OPEN`：继续。

---

## Step 2：合并前检查

检查 CI 状态和合并准备情况：

```bash
gh pr checks --json name,state,status,conclusion
```

解析输出：
1. 如果任何必需检查**失败中**：**停止。** 显示失败的检查项。
2. 如果必需检查**进行中**：进入 Step 3。
3. 如果所有检查都通过（或没有必需检查）：跳过 Step 3，进入 Step 4。

同时检查是否存在合并冲突：
```bash
gh pr view --json mergeable -q .mergeable
```
如果为 `CONFLICTING`：**停止。**“PR 存在合并冲突。解决并推送后再执行 land。”

---

## Step 3：等待 CI（如果仍在进行）

如果必需检查仍在进行中，等待其完成。超时时间设为 15 分钟：

```bash
gh pr checks --watch --fail-fast
```

为部署报告记录 CI 等待时间。

如果 CI 在超时前通过：继续到 Step 4。
如果 CI 失败：**停止。** 显示失败项。
如果超时（15 分钟）：**停止。**“CI 已运行 15 分钟。请手动调查。”

---

## Step 3.5：合并前准备闸门

**这是不可逆合并前的关键安全检查。** 合并无法被撤销，只能通过 revert commit 回退。
为下面每项检查收集**全部证据**，构建准备报告，并在继续前取得用户的明确确认。

为每项检查收集证据。追踪警告（黄色）和阻塞项（红色）。

### 3.5a：review 过期检查

```bash
~/.claude/skills/gstack/bin/gstack-review-read 2>/dev/null
```

解析输出。对于每个 review skill（plan-eng-review、plan-ceo-review、
plan-design-review、design-review-lite、codex-review）：

1. 找出最近 7 天内最新的一条记录。
2. 提取它的 `commit` 字段。
3. 与当前 HEAD 比较：`git rev-list --count STORED_COMMIT..HEAD`

**过期规则：**
- review 后新增 0 个 commit → CURRENT
- review 后新增 1-3 个 commit → RECENT（如果这些 commit 修改了代码而不只是文档，则标黄）
- review 后新增 4 个以上 commit → STALE（红色 —— review 可能已不能反映当前代码）
- 未找到 review → NOT RUN

**关键检查：** 查看最后一次 review 之后改了什么。运行：
```bash
git log --oneline STORED_COMMIT..HEAD
```
如果 review 之后的任何 commit 包含 “fix”、“refactor”、“rewrite”、
“overhaul”等字样，或修改了超过 5 个文件，则标记为 **STALE（review
之后有重大变更）**。因为即将合并的代码，已经不是 review 时那份代码了。

### 3.5b：测试结果

**免费的测试 —— 现在就运行：**

读取 CLAUDE.md，找到项目的测试命令。如果未指定，使用 `bun test`。
运行测试命令，并捕获退出码和输出。

```bash
bun test 2>&1 | tail -10
```

如果测试失败：**BLOCKER。** 不能在测试失败的情况下合并。

**E2E 测试 —— 检查最近结果：**

```bash
ls -t ~/.gstack-dev/evals/*-e2e-*-$(date +%Y-%m-%d)*.json 2>/dev/null | head -20
```

对于今天的每个 eval 文件，解析通过/失败计数。显示：
- 测试总数、通过数、失败数
- 距离运行结束已有多久（根据文件时间戳）
- 总成本
- 所有失败测试的名称

如果今天没有 E2E 结果：**WARNING —— 今天未运行 E2E 测试。**
如果存在 E2E 结果但有失败：**WARNING —— 有 N 个测试失败。** 列出它们。

**LLM judge evals —— 检查最近结果：**

```bash
ls -t ~/.gstack-dev/evals/*-llm-judge-*-$(date +%Y-%m-%d)*.json 2>/dev/null | head -5
```

如果找到，则解析并显示通过/失败情况。如果未找到，注明 “No LLM evals run today.”

### 3.5c：PR body 准确性检查

读取当前 PR body：
```bash
gh pr view --json body -q .body
```

读取当前 diff 摘要：
```bash
git log --oneline $(gh pr view --json baseRefName -q .baseRefName 2>/dev/null || echo main)..HEAD | head -20
```

将 PR body 与实际 commit 进行比较。检查：
1. **功能遗漏** —— commit 增加了重要功能，但 PR 中未提及
2. **描述过期** —— PR body 提到的内容后来被修改或回退
3. **版本错误** —— PR 标题或 body 引用的版本与 VERSION 文件不匹配

如果 PR body 看起来过期或不完整：**WARNING —— PR body 可能无法反映当前
变更。** 列出缺失或过期的内容。

### 3.5d：document-release 检查

检查此分支是否更新了文档：

```bash
git log --oneline --all-match --grep="docs:" $(gh pr view --json baseRefName -q .baseRefName 2>/dev/null || echo main)..HEAD | head -5
```

同时检查关键文档文件是否被修改：
```bash
git diff --name-only $(gh pr view --json baseRefName -q .baseRefName 2>/dev/null || echo main)...HEAD -- README.md CHANGELOG.md ARCHITECTURE.md CONTRIBUTING.md CLAUDE.md VERSION
```

如果此分支中 `CHANGELOG.md` 和 `VERSION` 都**没有**被修改，而 diff 中包含
新功能（新文件、新命令、新 skill）：**WARNING —— 很可能没有运行 /document-release。尽管新增了功能，但 CHANGELOG 和 VERSION 都未更新。**

如果只有文档变更（没有代码）：跳过此检查。

### 3.5e：准备报告与确认

构建完整准备报告：

```
╔══════════════════════════════════════════════════════════╗
║                 合并前准备报告                           ║
╠══════════════════════════════════════════════════════════╣
║                                                          ║
║  PR: #NNN — title                                        ║
║  Branch: feature → main                                  ║
║                                                          ║
║  REVIEWS                                                 ║
║  ├─ Eng Review:    CURRENT / STALE (N commits) / —       ║
║  ├─ CEO Review:    CURRENT / — (optional)                ║
║  ├─ Design Review: CURRENT / — (optional)                ║
║  └─ Codex Review:  CURRENT / — (optional)                ║
║                                                          ║
║  TESTS                                                   ║
║  ├─ Free tests:    PASS / FAIL (blocker)                 ║
║  ├─ E2E tests:     52/52 pass (25 min ago) / NOT RUN     ║
║  └─ LLM evals:     PASS / NOT RUN                        ║
║                                                          ║
║  DOCUMENTATION                                           ║
║  ├─ CHANGELOG:     Updated / NOT UPDATED (warning)       ║
║  ├─ VERSION:       0.9.8.0 / NOT BUMPED (warning)        ║
║  └─ Doc release:   Run / NOT RUN (warning)               ║
║                                                          ║
║  PR BODY                                                 ║
║  └─ Accuracy:      Current / STALE (warning)             ║
║                                                          ║
║  WARNINGS: N  |  BLOCKERS: N                             ║
╚══════════════════════════════════════════════════════════╝
```

如果存在 BLOCKER（免费测试失败）：列出它们，并推荐 B。
如果有 WARNING 但没有 blocker：列出每条 warning；如果 warning 较轻则推荐 A，如果 warning 较重则推荐 B。
如果全部为绿色：推荐 A。

使用 AskUserQuestion：

- **重新锚定：** “即将把 PR #NNN（title）从分支 X 合并到 Y。下面是
  准备报告。”显示上面的报告。
- 明确列出每条 warning 和 blocker。
- **RECOMMENDATION:** 全绿时选 A。有重大 warning 时选 B。
  只有在用户理解风险时才选 C。
- A) Merge — 准备检查已通过（Completeness: 10/10）
- B) 先不要合并 —— 先处理这些 warning（Completeness: 10/10）
- C) 仍然合并 —— 我理解这些风险（Completeness: 3/10）

如果用户选择 B：**停止。** 准确列出需要做什么：
- 如果 review 已过期：“重新运行 /plan-eng-review（或 /review）以审查当前代码。”
- 如果未运行 E2E：“运行 `bun run test:e2e` 进行验证。”
- 如果文档未更新：“运行 /document-release 更新文档。”
- 如果 PR body 已过期：“更新 PR body 以反映当前变更。”

如果用户选择 A 或 C：继续到 Step 4。

---

## Step 4：合并 PR

记录起始时间戳，用于计时数据。

先尝试 auto-merge（遵循仓库的合并设置和 merge queue）：

```bash
gh pr merge --auto --delete-branch
```

如果 `--auto` 不可用（仓库未启用 auto-merge），则直接合并：

```bash
gh pr merge --squash --delete-branch
```

如果因权限错误而合并失败：**停止。**“你在这个仓库没有合并权限。请让维护者来合并。”

如果启用了 merge queue，`gh pr merge --auto` 会将其加入队列。轮询直到 PR 真正合并：

```bash
gh pr view --json state -q .state
```

每 30 秒轮询一次，最多 30 分钟。每 2 分钟显示一次进度消息：“Waiting for merge queue...（已过去 Xm）”

如果 PR 状态变为 `MERGED`：捕获 merge commit SHA 并继续。
如果 PR 被移出队列（状态回到 `OPEN`）：**停止。**“PR 已从 merge queue 中移除。”
如果超时（30 分钟）：**停止。**“Merge queue 已处理 30 分钟。请手动检查队列。”

记录合并时间戳和耗时。

---

## Step 5：部署策略检测

确定这是什么类型的项目，以及应如何验证部署。

首先，运行部署配置引导，以检测或读取已持久化的部署设置：

```bash
# Check for persisted deploy config in CLAUDE.md
DEPLOY_CONFIG=$(grep -A 20 "## Deploy Configuration" CLAUDE.md 2>/dev/null || echo "NO_CONFIG")
echo "$DEPLOY_CONFIG"

# If config exists, parse it
if [ "$DEPLOY_CONFIG" != "NO_CONFIG" ]; then
  PROD_URL=$(echo "$DEPLOY_CONFIG" | grep -i "production.*url" | head -1 | sed 's/.*: *//')
  PLATFORM=$(echo "$DEPLOY_CONFIG" | grep -i "platform" | head -1 | sed 's/.*: *//')
  echo "PERSISTED_PLATFORM:$PLATFORM"
  echo "PERSISTED_URL:$PROD_URL"
fi

# Auto-detect platform from config files
[ -f fly.toml ] && echo "PLATFORM:fly"
[ -f render.yaml ] && echo "PLATFORM:render"
([ -f vercel.json ] || [ -d .vercel ]) && echo "PLATFORM:vercel"
[ -f netlify.toml ] && echo "PLATFORM:netlify"
[ -f Procfile ] && echo "PLATFORM:heroku"
([ -f railway.json ] || [ -f railway.toml ]) && echo "PLATFORM:railway"

# Detect deploy workflows
for f in .github/workflows/*.yml .github/workflows/*.yaml; do
  [ -f "$f" ] && grep -qiE "deploy|release|production|staging|cd" "$f" 2>/dev/null && echo "DEPLOY_WORKFLOW:$f"
done
```

如果在 CLAUDE.md 中找到了 `PERSISTED_PLATFORM` 和 `PERSISTED_URL`，直接使用它们，
并跳过手动检测。如果没有已持久化配置，则使用自动检测到的平台
来指导部署验证。如果什么都没检测到，则通过下面决策树中的 AskUserQuestion 向用户询问。

如果你希望为后续运行持久化部署设置，建议用户运行 `/setup-deploy`。

然后运行 `gstack-diff-scope` 来分类变更：

```bash
eval $(~/.claude/skills/gstack/bin/gstack-diff-scope $(gh pr view --json baseRefName -q .baseRefName 2>/dev/null || echo main) 2>/dev/null)
echo "FRONTEND=$SCOPE_FRONTEND BACKEND=$SCOPE_BACKEND DOCS=$SCOPE_DOCS CONFIG=$SCOPE_CONFIG"
```

**决策树（按顺序评估）：**

1. 如果用户通过参数提供了生产 URL：用它进行金丝雀验证。同时检查是否存在部署工作流。

2. 检查 GitHub Actions 部署工作流：
```bash
gh run list --branch <base> --limit 5 --json name,status,conclusion,headSha,workflowName
```
查找名称中包含 “deploy”、“release”、“production”、“staging” 或 “cd” 的工作流。如果找到：在 Step 6 中轮询部署工作流，然后运行 canary。

3. 如果只有 `SCOPE_DOCS` 为 true（没有 frontend、backend、config）：完全跳过验证。输出：“PR 已合并。此次变更仅涉及文档 —— 无需部署验证。”然后进入 Step 9。

4. 如果未检测到部署工作流，且也未提供 URL：只使用一次 AskUserQuestion：
   - **上下文：** PR 已成功合并。未检测到部署工作流或生产 URL。
   - **RECOMMENDATION:** 如果这是一个库/CLI 工具，选 B；如果这是一个 web app，选 A。
   - A) 提供一个生产 URL 进行验证
   - B) 跳过验证 —— 这个项目没有 web 部署

---

## Step 6：等待部署（如果适用）

部署验证策略取决于 Step 5 中检测到的平台。

### 策略 A：GitHub Actions 工作流

如果检测到了部署工作流，找出由 merge commit 触发的那次运行：

```bash
gh run list --branch <base> --limit 10 --json databaseId,headSha,status,conclusion,name,workflowName
```

使用 merge commit SHA（Step 4 中捕获）进行匹配。如果有多个匹配工作流，优先选择名称与 Step 5 中检测到的部署工作流一致的那个。

每 30 秒轮询一次：
```bash
gh run view <run-id> --json status,conclusion
```

### 策略 B：平台 CLI（Fly.io、Render、Heroku）

如果在 CLAUDE.md 中配置了部署状态命令（例如 `fly status --app myapp`），则使用它来替代或补充 GitHub Actions 轮询。

**Fly.io：** 合并后，Fly 通过 GitHub Actions 或 `fly deploy` 进行部署。检查方式：
```bash
fly status --app {app} 2>/dev/null
```
查看 `Machines` 状态是否为 `started`，以及最近部署时间戳。

**Render：** Render 会在推送到关联分支时自动部署。通过轮询生产 URL，直到它开始响应：
```bash
curl -sf {production-url} -o /dev/null -w "%{http_code}" 2>/dev/null
```
Render 部署通常需要 2-5 分钟。每 30 秒轮询一次。

**Heroku：** 检查最新 release：
```bash
heroku releases --app {app} -n 1 2>/dev/null
```

### 策略 C：自动部署平台（Vercel、Netlify）

Vercel 和 Netlify 会在合并后自动部署。不需要显式触发部署。等待 60 秒让部署传播完成，然后直接进入 Step 7 的金丝雀验证。

### 策略 D：自定义部署钩子

如果 CLAUDE.md 的 “Custom deploy hooks” 部分有自定义部署状态命令，则运行该命令并检查其退出码。

### 通用：计时与失败处理

记录部署开始时间。每 2 分钟显示一次进度：“Deploy in progress...（已过去 Xm）”

如果部署成功（`conclusion` 为 `success` 或健康检查通过）：记录部署耗时，继续到 Step 7。

如果部署失败（`conclusion` 为 `failure`）：使用 AskUserQuestion：
- **上下文：** 合并 PR 后，部署工作流失败。
- **RECOMMENDATION:** 选 A，先调查再决定是否 revert。
- A) 调查部署日志
- B) 在 base branch 上创建一个 revert commit
- C) 仍然继续 —— 部署失败可能无关

如果超时（20 分钟）：警告“部署已运行 20 分钟”，并询问是继续等待还是跳过验证。

---

## Step 7：金丝雀验证（按范围决定深度）

使用 Step 5 中的 diff-scope 分类结果来确定金丝雀验证深度：

| Diff Scope | Canary Depth |
|------------|-------------|
| 仅 `SCOPE_DOCS` | 已在 Step 5 跳过 |
| 仅 `SCOPE_CONFIG` | Smoke：`$B goto` + 验证 200 状态 |
| 仅 `SCOPE_BACKEND` | 控制台错误 + 性能检查 |
| 包含 `SCOPE_FRONTEND` | 完整：控制台 + 性能 + 截图 |
| 混合范围 | 完整 canary |

**完整 canary 序列：**

```bash
$B goto <url>
```

检查页面是否成功加载（200，且不是错误页）。

```bash
$B console --errors
```

检查关键控制台错误：包含 `Error`、`Uncaught`、`Failed to load`、`TypeError`、`ReferenceError` 的行。忽略警告。

```bash
$B perf
```

检查页面加载时间是否低于 10 秒。

```bash
$B text
```

验证页面是否有内容（不是空白页，也不是通用错误页）。

```bash
$B snapshot -i -a -o ".gstack/deploy-reports/post-deploy.png"
```

拍摄带标注的截图，作为证据。

**健康评估：**
- 页面成功加载并返回 200 状态 → PASS
- 没有关键控制台错误 → PASS
- 页面有真实内容（不是空白页或错误页） → PASS
- 加载时间低于 10 秒 → PASS

如果全部通过：标记为 HEALTHY，继续到 Step 9。

如果有任一项失败：显示证据（截图路径、控制台错误、性能数据）。使用 AskUserQuestion：
- **上下文：** 部署后的 canary 在生产站点上检测到问题。
- **RECOMMENDATION:** 根据严重程度选择 —— 严重问题（站点宕机）选 B，轻微问题（控制台错误）选 A。
- A) 这是预期情况（部署仍在进行、缓存尚未清空）—— 标记为健康
- B) 已损坏 —— 创建一个 revert commit
- C) 进一步调查（打开站点、查看日志）

---

## Step 8：Revert（如果需要）

如果用户在任意时刻选择 revert：

```bash
git fetch origin <base>
git checkout <base>
git revert <merge-commit-sha> --no-edit
git push origin <base>
```

如果 revert 出现冲突：警告“Revert 存在冲突 —— 需要手动解决。merge commit SHA 是 `<sha>`。你可以手动运行 `git revert <sha>`”

如果 base branch 有推送保护：警告“分支保护可能阻止直接推送 —— 请改为创建一个 revert PR：`gh pr create --title 'revert: <original PR title>'`”

成功 revert 后，记录 revert commit SHA，并以 REVERTED 状态继续到 Step 9。

---

## Step 9：部署报告

创建部署报告目录：

```bash
mkdir -p .gstack/deploy-reports
```

生成并显示 ASCII 摘要：

```
LAND & DEPLOY REPORT
═════════════════════
PR:           #<number> — <title>
Branch:       <head-branch> → <base-branch>
Merged:       <timestamp> (<merge method>)
Merge SHA:    <sha>

Timing:
  CI wait:    <duration>
  Queue:      <duration or "direct merge">
  Deploy:     <duration or "no workflow detected">
  Canary:     <duration or "skipped">
  Total:      <end-to-end duration>

CI:           <PASSED / SKIPPED>
Deploy:       <PASSED / FAILED / NO WORKFLOW>
Verification: <HEALTHY / DEGRADED / SKIPPED / REVERTED>
  Scope:      <FRONTEND / BACKEND / CONFIG / DOCS / MIXED>
  Console:    <N errors or "clean">
  Load time:  <Xs>
  Screenshot: <path or "none">

VERDICT: <DEPLOYED AND VERIFIED / DEPLOYED (UNVERIFIED) / REVERTED>
```

将报告保存到 `.gstack/deploy-reports/{date}-pr{number}-deploy.md`。

记录到 review 仪表盘：

```bash
source <(~/.claude/skills/gstack/bin/gstack-slug 2>/dev/null)
mkdir -p ~/.gstack/projects/$SLUG
```

写入一条包含计时数据的 JSONL 记录：
```json
{"skill":"land-and-deploy","timestamp":"<ISO>","status":"<SUCCESS/REVERTED>","pr":<number>,"merge_sha":"<sha>","deploy_status":"<HEALTHY/DEGRADED/SKIPPED>","ci_wait_s":<N>,"queue_s":<N>,"deploy_s":<N>,"canary_s":<N>,"total_s":<N>}
```

---

## Step 10：建议后续操作

在部署报告之后，建议相关后续操作：

- 如果验证了生产 URL：“运行 `/canary <url> --duration 10m` 进行扩展监控。”
- 如果收集了性能数据：“运行 `/benchmark <url>` 进行深度性能审计。”
- “运行 `/document-release` 更新项目文档。”

---

## 重要规则

- **绝不要 force push。** 使用安全的 `gh pr merge`。
- **绝不要跳过 CI。** 如果检查失败，必须停止。
- **自动检测一切。** PR 编号、合并方式、部署策略、项目类型。只有在确实无法推断信息时才询问。
- **轮询要有退避。** 不要猛打 GitHub API。CI/部署使用 30 秒间隔，并设置合理超时。
- **随时都可以 revert。** 在每个失败点都提供 revert 作为退出选项。
- **只做单次验证，不做持续监控。** `/land-and-deploy` 只检查一次。扩展监控循环由 `/canary` 执行。
- **清理。** 合并后删除功能分支（通过 `--delete-branch`）。
- **目标是：用户输入 `/land-and-deploy`，接下来看到的就是部署报告。**