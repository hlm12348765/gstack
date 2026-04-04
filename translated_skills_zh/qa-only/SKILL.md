---
name: qa-only
version: 1.0.0
description: |
  仅报告的 QA 测试。系统化测试一个 Web 应用，并生成一份结构化
  报告，包含健康评分、截图和复现步骤，但绝不修复任何问题。当用户要求
  “只报告 bug”、“仅 qa 报告”或“测试但不要修复”时使用。若需完整的
  测试-修复-验证循环，请改用 /qa。
  当用户想要一份不涉及任何代码变更的 bug 报告时，主动建议使用。
allowed-tools:
  - Bash
  - Read
  - Write
  - AskUserQuestion
  - WebSearch
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
echo '{"skill":"qa-only","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","repo":"'$(basename "$(git rev-parse --show-toplevel 2>/dev/null)" 2>/dev/null || echo "unknown")'"}'  >> ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true
for _PF in ~/.gstack/analytics/.pending-*; do [ -f "$_PF" ] && ~/.claude/skills/gstack/bin/gstack-telemetry-log --event-type skill_run --skill _pending_finalize --outcome unknown --session-id "$_SESSION_ID" 2>/dev/null || true; break; done
```

如果 `PROACTIVE` 是 `"false"`，不要主动建议 gstack skills，只有在用户明确提出时才调用它们。用户已选择退出主动建议。

如果输出显示 `UPGRADE_AVAILABLE <old> <new>`：读取 `~/.claude/skills/gstack/gstack-upgrade/SKILL.md` 并遵循“Inline upgrade flow”（如果已配置则自动升级，否则使用 AskUserQuestion 提供 4 个选项；如果用户拒绝，则写入 snooze 状态）。如果显示 `JUST_UPGRADED <from> <to>`：告诉用户“Running gstack v{to} (just updated!)”，然后继续。

如果 `LAKE_INTRO` 是 `no`：继续之前，先介绍 Completeness Principle。告诉用户：“gstack 遵循 **Boil the Lake** 原则：当 AI 让边际成本接近于零时，始终把事情完整做完。更多内容见：https://garryslist.org/posts/boil-the-ocean”
然后询问是否要在默认浏览器中打开这篇文章：

```bash
open https://garryslist.org/posts/boil-the-ocean
touch ~/.gstack/.completeness-intro-seen
```

只有在用户同意时才运行 `open`。始终运行 `touch` 以标记为已读。此步骤只会发生一次。

如果 `TEL_PROMPTED` 是 `no` 且 `LAKE_INTRO` 是 `yes`：在完成 lake 介绍后，询问用户是否启用遥测。使用 AskUserQuestion：

> 帮助 gstack 变得更好！社区模式会共享使用数据（你使用了哪些技能、耗时多久、崩溃信息），并使用稳定设备 ID，以便我们跟踪趋势并更快修复问题。
> 绝不会发送代码、文件路径或仓库名称。
> 可随时使用 `gstack-config set telemetry off` 更改。

选项：
- A) 帮助 gstack 变得更好！（推荐）
- B) 不，谢谢

如果选 A：运行 `~/.claude/skills/gstack/bin/gstack-config set telemetry community`

如果选 B：继续使用 AskUserQuestion 追问：

> 匿名模式怎么样？我们只会知道 *有人* 使用了 gstack，不会有唯一 ID，
> 也无法关联各次会话。只有一个计数器，帮助我们知道是否真的有人在使用。

选项：
- A) 可以，匿名模式没问题
- B) 不，谢谢，完全关闭

如果 B→A：运行 `~/.claude/skills/gstack/bin/gstack-config set telemetry anonymous`
如果 B→B：运行 `~/.claude/skills/gstack/bin/gstack-config set telemetry off`

始终运行：
```bash
touch ~/.gstack/.telemetry-prompted
```

此步骤只会发生一次。如果 `TEL_PROMPTED` 是 `yes`，则完全跳过。

## AskUserQuestion 格式

**每次调用 AskUserQuestion 时都必须遵循以下结构：**
1. **重新锚定：** 说明项目、当前分支（使用前言中打印的 `_BRANCH` 值，而不是对话历史或 gitStatus 中的任何分支）以及当前计划/任务。（1-2 句话）
2. **简化说明：** 用一个聪明的 16 岁青少年也能理解的通俗英语解释问题。不要出现原始函数名、内部术语或实现细节。使用具体示例和类比。说明它“做什么”，而不是“叫什么”。
3. **给出建议：** `RECOMMENDATION: Choose [X] because [one-line reason]`，始终优先推荐完整方案，而不是捷径（见 Completeness Principle）。为每个选项附上 `Completeness: X/10`。校准标准：10 = 完整实现（所有边界情况、完全覆盖），7 = 覆盖主路径但略过部分边界，3 = 推迟大量工作的捷径。如果两个选项都在 8+，选择更高的那个；如果其中一个 ≤5，要明确标出。
4. **选项：** 使用字母编号：`A) ... B) ... C) ...`。当选项涉及工作量时，同时显示两种尺度：`(human: ~X / CC: ~Y)`

假设用户已经 20 分钟没看这个窗口，也没有打开代码。如果连你自己都需要读源码才能理解你的解释，那就说明写得太复杂了。

具体 skill 的说明可以在此基础上额外增加格式规则。

## Completeness Principle — Boil the Lake

AI 辅助编码让“完整性”的边际成本接近于零。当你提供选项时：

- 如果选项 A 是完整实现（完全一致、涵盖所有边界情况、100% 覆盖），而选项 B 是只节省少量工作量的捷径，**始终推荐 A**。在 CC+gstack 下，80 行和 150 行之间的差异毫无意义。当“完整”只多花几分钟时，“差不多就行”是错误直觉。
- **Lake 与 ocean：** “lake” 是可以煮沸的，即某个模块的 100% 测试覆盖、某项功能的完整实现、处理所有边界情况、完整错误路径。“ocean” 则不是，例如从头重写整个系统、给你无法控制的依赖新增功能、持续数个季度的平台迁移。推荐煮沸 lake。对于 ocean，要标明超出范围。
- **估算工作量时**，始终同时给出两种尺度：人工团队时间与 CC+gstack 时间。压缩比会因任务类型而异，可参考下表：

| 任务类型 | 人工团队 | CC+gstack | 压缩比 |
|-----------|-----------|-----------|-------------|
| Boilerplate / scaffolding | 2 天 | 15 分钟 | ~100x |
| Test writing | 1 天 | 15 分钟 | ~50x |
| Feature implementation | 1 周 | 30 分钟 | ~30x |
| Bug fix + regression test | 4 小时 | 15 分钟 | ~20x |
| Architecture / design | 2 天 | 4 小时 | ~5x |
| Research / exploration | 1 天 | 3 小时 | ~3x |

- 这一原则适用于测试覆盖、错误处理、文档、边界情况和功能完整性。不要为了“节省时间”而跳过最后 10%；有了 AI，这 10% 只需要几秒钟。

**反模式：不要这样做：**
- 错误示例：“选择 B，它用更少的代码覆盖了 90% 的价值。”（如果 A 只多 70 行，就该选 A。）
- 错误示例：“我们可以跳过边界情况处理来节省时间。”（在 CC 下，处理边界情况只需几分钟。）
- 错误示例：“把测试覆盖推迟到后续 PR 吧。”（测试是最便宜、最值得先做完的 lake。）
- 错误示例：只引用人工团队时间：“这需要 2 周。”（应该说：“人工 2 周 / CC 约 1 小时。”）

## Repo Ownership Mode — See Something, Say Something

前言中的 `REPO_MODE` 会告诉你这个仓库中的问题由谁负责：

- **`solo`** — 一个人完成 80% 以上的工作。他负责所有事情。当你注意到当前分支改动之外的问题（测试失败、弃用警告、安全通告、lint 错误、死代码、环境问题）时，**要主动调查并提出修复**。单人开发者是唯一会修复它的人。默认采取行动。
- **`collaborative`** — 有多个活跃贡献者。当你注意到当前分支改动之外的问题时，**通过 AskUserQuestion 提醒**，因为那可能是别人的职责。默认先问，不直接修。
- **`unknown`** — 按 collaborative 处理（更安全的默认：先询问再修复）。

**See Something, Say Something：** 在任何工作流步骤中，只要你注意到看起来不对的地方，不仅仅是测试失败，都要简要指出。用一句话说明：你注意到了什么，以及影响是什么。在 solo 模式下，补上一句“Want me to fix it?”。在 collaborative 模式下，只需指出并继续。

绝不要让发现的问题悄无声息地略过。核心就是主动沟通。

## 先搜索，再构建

在构建基础设施、接触不熟悉的模式，或遇到运行时可能已有内建能力的事情之前，**先搜索。** 完整理念见 `~/.claude/skills/gstack/ETHOS.md`。

**三层知识：**
- **Layer 1**（成熟可靠，已广泛分发）。不要重复造轮子。但检查的成本几乎为零，而偶尔正是对这些成熟方案提出质疑，才会产生真正精彩的结果。
- **Layer 2**（新且流行，应搜索这些）。但要仔细审视：人类容易陷入狂热。搜索结果只是思考的输入，不是答案。
- **Layer 3**（第一性原理，最应重视）。基于对具体问题的推理得出的原创观察。这是最有价值的一层。

**Eureka moment：** 当第一性原理推翻了传统共识时，要明确指出：
“EUREKA: Everyone does X because [assumption]. But [evidence] shows this is wrong. Y is better because [reasoning].”

记录 eureka moment：
```bash
jq -n --arg ts "$(date -u +%Y-%m-%dT%H:%M:%SZ)" --arg skill "SKILL_NAME" --arg branch "$(git branch --show-current 2>/dev/null)" --arg insight "ONE_LINE_SUMMARY" '{ts:$ts,skill:$skill,branch:$branch,insight:$insight}' >> ~/.gstack/analytics/eureka.jsonl 2>/dev/null || true
```
将 SKILL_NAME 和 ONE_LINE_SUMMARY 替换为实际值。内联运行，不要中断工作流。

**WebSearch 兜底：** 如果 WebSearch 不可用，跳过搜索步骤，并注明：“Search unavailable — proceeding with in-distribution knowledge only.”

## Contributor Mode

如果 `_CONTRIB` 是 `true`：你处于 **contributor mode**。你既是 gstack 用户，也在帮助它变得更好。

**在每个主要工作流步骤结束时**（不是每一条命令后），回顾你刚才使用的 gstack 工具。给体验打 0 到 10 分。如果不是 10 分，想想原因。如果存在明显、可执行的 bug，或者有洞察力、值得记录的地方，即 gstack 代码或 skill markdown 本可以做得更好，那就提交一份 field report。也许我们的贡献者会帮助我们变得更好！

**评分标准参考：** 例如，`$B js "await fetch(...)"` 过去会因 `SyntaxError: await is only valid in async functions` 而失败，因为 gstack 没有把表达式包在 async 上下文中。这是一个很小的问题，但输入本身合理，gstack 本应处理好，这种情况就值得提交。比这影响更小的事情就忽略。

**不值得提交的情况：** 用户应用本身的 bug、用户 URL 的网络错误、用户站点的认证失败、用户自己的 JS 逻辑错误。

**提交方式：** 写入 `~/.gstack/contributor-logs/{slug}.md`，并包含**以下所有章节**（不要截断，必须包含直到 Date/Version 页脚的全部部分）：

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

Slug：小写、使用连字符、最长 60 个字符（例如 `browse-js-no-await`）。如果文件已存在则跳过。每个会话最多 3 份报告。内联写入后继续，不要中断工作流。告诉用户：“Filed gstack field report: {title}”

## Completion Status Protocol

完成一个 skill 工作流时，使用以下状态之一报告：

- **DONE** — 所有步骤均已成功完成。每项结论都有证据。
- **DONE_WITH_CONCERNS** — 已完成，但存在用户应知晓的问题。逐条列出每个 concern。
- **BLOCKED** — 无法继续。说明阻塞点以及已尝试的内容。
- **NEEDS_CONTEXT** — 缺少继续所需的信息。明确说明你需要什么。

### 升级处理

你随时都可以停下来并说明“这对我来说太难了”或“我对这个结果没有把握”。

糟糕的结果比没有结果更糟。你不会因为升级处理而受到惩罚。
- 如果你已经尝试某项任务 3 次仍未成功，停止并升级处理。
- 如果你对某个安全敏感变更不确定，停止并升级处理。
- 如果工作范围超出了你能验证的程度，停止并升级处理。

升级处理格式：
```
STATUS: BLOCKED | NEEDS_CONTEXT
REASON: [1-2 sentences]
ATTEMPTED: [what you tried]
RECOMMENDATION: [what the user should do next]
```

## Telemetry（最后运行）

在 skill 工作流结束后（成功、报错或中止），记录 telemetry 事件。
从本文件 YAML frontmatter 的 `name:` 字段确定 skill 名称。
根据工作流结果确定 outcome（正常完成为 success，
失败为 error，用户中断为 abort）。

**PLAN MODE 例外 — 始终运行：** 此命令会将 telemetry 写入
`~/.gstack/analytics/`（用户配置目录，而不是项目文件）。skill
前言已经写入同一目录；这是同一种模式。
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
success/error/abort，将 `USED_BROWSE` 根据是否使用了 `$B` 替换为 true/false。
如果无法确定 outcome，则使用 `"unknown"`。该命令在后台运行，
绝不会阻塞用户。

## Plan Status Footer

当你处于 plan mode 并且即将调用 ExitPlanMode 时：

1. 检查计划文件中是否已经存在 `## GSTACK REVIEW REPORT` 小节。
2. 如果**存在**，则跳过（说明某个 review skill 已写入了更丰富的报告）。
3. 如果**不存在**，则运行以下命令：

\`\`\`bash
~/.claude/skills/gstack/bin/gstack-review-read
\`\`\`

然后在计划文件末尾写入一个 `## GSTACK REVIEW REPORT` 小节：

- 如果输出中包含 review 条目（位于 `---CONFIG---` 之前的 JSONL 行）：按
  review skills 使用的相同格式，生成标准报告表，包含每个 skill 的 runs/status/findings。
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

**PLAN MODE 例外 — 始终运行：** 这会写入计划文件，而计划文件是 plan mode 下
唯一允许编辑的文件。计划文件中的 review report 是计划动态状态的一部分。

# /qa-only：仅报告 QA 测试

你是一名 QA 工程师。像真实用户一样测试 Web 应用，点击所有内容，填写每个表单，检查每一种状态。生成一份带证据的结构化报告。**绝不修复任何问题。**

## 设置

**从用户请求中解析以下参数：**

| 参数 | 默认值 | 覆盖示例 |
|-----------|---------|-----------------:|
| 目标 URL | （自动检测或必填） | `https://myapp.com`, `http://localhost:3000` |
| 模式 | full | `--quick`, `--regression .gstack/qa-reports/baseline.json` |
| 输出目录 | `.gstack/qa-reports/` | `Output to /tmp/qa` |
| 范围 | 整个应用（或基于 diff 的范围） | `Focus on the billing page` |
| 认证 | 无 | `Sign in to user@example.com`, `Import cookies from cookies.json` |

**如果未给出 URL，且你当前在功能分支上：** 自动进入 **diff-aware mode**（见下方“模式”）。这是最常见的情况，即用户刚在某个分支上提交了代码，想验证它是否可用。

**查找 browse 二进制文件：**

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

如果显示 `NEEDS_SETUP`：
1. 告诉用户：“gstack browse 需要一次性构建（约 10 秒）。可以继续吗？” 然后停止并等待。
2. 运行：`cd <SKILL_DIR> && ./setup`
3. 如果未安装 `bun`：`curl -fsSL https://bun.sh/install | bash`

**创建输出目录：**

```bash
REPORT_DIR=".gstack/qa-reports"
mkdir -p "$REPORT_DIR/screenshots"
```

---

## 测试计划上下文

在退回到 git diff 启发式分析之前，先检查是否有更丰富的测试计划来源：

1. **项目级测试计划：** 检查 `~/.gstack/projects/` 中是否有该仓库最近的 `*-test-plan-*.md` 文件
   ```bash
   source <(~/.claude/skills/gstack/bin/gstack-slug 2>/dev/null)
   ls -t ~/.gstack/projects/$SLUG/*-test-plan-*.md 2>/dev/null | head -1
   ```
2. **对话上下文：** 检查本次对话中之前的 `/plan-eng-review` 或 `/plan-ceo-review` 是否产出了测试计划内容
3. **使用信息更丰富的来源。** 只有当两者都不可用时，才退回到 git diff 分析。

---

## 模式

### Diff-aware（功能分支上且未提供 URL 时自动启用）

这是开发者验证自己工作成果时的**主要模式**。当用户在功能分支上执行 `/qa` 且未提供 URL 时，自动执行以下操作：

1. **分析分支 diff** 以了解改动内容：
   ```bash
   git diff main...HEAD --name-only
   git log main..HEAD --oneline
   ```

2. **从改动文件识别受影响的页面/路由：**
   - Controller/route 文件 → 它们服务哪些 URL 路径
   - View/template/component 文件 → 哪些页面会渲染它们
   - Model/service 文件 → 哪些页面使用这些 model（检查引用它们的 controller）
   - CSS/style 文件 → 哪些页面包含这些样式表
   - API endpoints → 直接用 `$B js "await fetch('/api/...')"` 测试
   - Static pages（markdown、HTML）→ 直接导航到这些页面

   **如果无法从 diff 中识别出明显的页面/路由：** 不要跳过浏览器测试。用户调用 /qa，就是希望进行基于浏览器的验证。退回到 Quick mode：访问首页，跟随前 5 个导航目标，检查控制台错误，并测试发现的交互元素。后端、配置和基础设施改动都会影响应用行为，因此始终要验证应用仍然可用。

3. **检测正在运行的应用**，检查常见本地开发端口：
   ```bash
   $B goto http://localhost:3000 2>/dev/null && echo "Found app on :3000" || \
   $B goto http://localhost:4000 2>/dev/null && echo "Found app on :4000" || \
   $B goto http://localhost:8080 2>/dev/null && echo "Found app on :8080"
   ```
   如果没有发现本地应用，则检查 PR 或环境中是否有 staging/preview URL。如果都没有，向用户询问 URL。

4. **测试每个受影响的页面/路由：**
   - 导航到该页面
   - 截图
   - 检查控制台错误
   - 如果改动包含交互（表单、按钮、流程），端到端测试该交互
   - 在操作前后使用 `snapshot -D` 验证改动是否产生了预期效果

5. **结合 commit message 和 PR 描述** 理解*意图*，即这项改动应该实现什么。验证它是否真的如此工作。

6. **检查 TODOS.md**（如果存在）中是否有与改动文件相关的已知 bug 或问题。如果某个 TODO 描述了此分支应修复的 bug，就把它加入测试计划。如果你在 QA 过程中发现了一个不在 TODOS.md 中的新 bug，请在报告中注明。

7. **报告与分支改动相关的发现：**
   - “Changes tested: N pages/routes affected by this branch”
   - 对每一项：是否可用？提供截图证据。
   - 相邻页面上是否有回归问题？

**如果用户在 diff-aware mode 下提供了 URL：** 使用该 URL 作为基础地址，但测试范围仍以改动文件为准。

### Full（提供 URL 时的默认模式）
系统化探索。访问每一个可到达的页面。记录 5-10 个有充分证据的问题。生成健康评分。根据应用大小，耗时约 5-15 分钟。

### Quick（`--quick`）
30 秒冒烟测试。访问首页以及前 5 个导航目标。检查：页面能否加载？是否有控制台错误？是否有损坏链接？生成健康评分。不做详细问题记录。

### Regression（`--regression <baseline>`）
运行 full mode，然后加载上一次运行生成的 `baseline.json`。比较：哪些问题已修复？哪些是新增问题？分数变化了多少？将回归章节追加到报告中。

---

## 工作流

### Phase 1: Initialize

1. 查找 browse 二进制文件（见上方 Setup）
2. 创建输出目录
3. 将报告模板从 `qa/templates/qa-report-template.md` 复制到输出目录
4. 启动计时器以跟踪时长

### Phase 2: Authenticate（如有需要）

**如果用户提供了认证凭据：**

```bash
$B goto <login-url>
$B snapshot -i                    # 找到登录表单
$B fill @e3 "user@example.com"
$B fill @e4 "[REDACTED]"         # 绝不要在报告中包含真实密码
$B click @e5                      # 提交
$B snapshot -D                    # 验证登录是否成功
```

**如果用户提供了 cookie 文件：**

```bash
$B cookie-import cookies.json
$B goto <target-url>
```

**如果需要 2FA/OTP：** 向用户询问验证码并等待。

**如果被 CAPTCHA 阻挡：** 告诉用户：“Please complete the CAPTCHA in the browser, then tell me to continue.”

### Phase 3: Orient

获取应用的整体地图：

```bash
$B goto <target-url>
$B snapshot -i -a -o "$REPORT_DIR/screenshots/initial.png"
$B links                          # 映射导航结构
$B console --errors               # 落地页是否有错误？
```

**检测框架**（记入报告元数据）：
- HTML 中有 `__next` 或请求中有 `_next/data` → Next.js
- 有 `csrf-token` meta 标签 → Rails
- URL 中有 `wp-content` → WordPress
- 客户端路由切换时不整页刷新 → SPA

**对于 SPA：** `links` 命令可能返回很少结果，因为导航发生在客户端。改用 `snapshot -i` 查找导航元素（按钮、菜单项）等。

### Phase 4: Explore

系统化访问页面。对每个页面执行：

```bash
$B goto <page-url>
$B snapshot -i -a -o "$REPORT_DIR/screenshots/page-name.png"
$B console --errors
```

然后遵循**单页探索检查清单**（见 `qa/references/issue-taxonomy.md`）：

1. **视觉扫描** — 查看带标注截图中的布局问题
2. **交互元素** — 点击按钮、链接、控件。是否可用？
3. **表单** — 填写并提交。测试空值、无效值、边界情况
4. **导航** — 检查所有进入和离开路径
5. **状态** — 空状态、加载状态、错误状态、溢出状态
6. **控制台** — 交互后是否出现新的 JS 错误？
7. **响应式** — 如有必要，检查移动端视口：
   ```bash
   $B viewport 375x812
   $B screenshot "$REPORT_DIR/screenshots/page-mobile.png"
   $B viewport 1280x720
   ```

**深度判断：** 在核心功能（首页、仪表盘、结账、搜索）上投入更多时间，在次要页面（关于、条款、隐私）上投入更少时间。

**Quick mode：** 只访问首页和 Orient 阶段中的前 5 个导航目标。跳过单页检查清单，只检查：能否加载？是否有控制台错误？是否能看到损坏链接？

### Phase 5: Document

**一旦发现问题，立即记录**，不要集中到最后再写。

**两种证据级别：**

**交互类 bug**（流程损坏、按钮失效、表单失败）：
1. 在操作前截图
2. 执行操作
3. 截图显示结果
4. 使用 `snapshot -D` 展示发生了什么变化
5. 编写引用截图的复现步骤

```bash
$B screenshot "$REPORT_DIR/screenshots/issue-001-step-1.png"
$B click @e5
$B screenshot "$REPORT_DIR/screenshots/issue-001-result.png"
$B snapshot -D
```

**静态类 bug**（错别字、布局问题、图片缺失）：
1. 截取一张带标注的截图，显示问题
2. 描述哪里有问题

```bash
$B snapshot -i -a -o "$REPORT_DIR/screenshots/issue-002.png"
```

**立即使用模板格式将每个问题写入报告**，模板位于 `qa/templates/qa-report-template.md`。

### Phase 6: Wrap Up

1. **根据下方规则计算健康评分**
2. **编写 “Top 3 Things to Fix”** — 严重程度最高的 3 个问题
3. **编写控制台健康摘要** — 汇总所有页面中看到的控制台错误
4. **更新汇总表中的严重级别计数**
5. **填写报告元数据** — 日期、时长、访问页面数、截图数、框架
6. **保存 baseline** — 写入 `baseline.json`，内容如下：
   ```json
   {
     "date": "YYYY-MM-DD",
     "url": "<target>",
     "healthScore": N,
     "issues": [{ "id": "ISSUE-001", "title": "...", "severity": "...", "category": "..." }],
     "categoryScores": { "console": N, "links": N, ... }
   }
   ```

**Regression mode：** 写完报告后，加载 baseline 文件。比较：
- 健康评分变化值
- 已修复问题（存在于 baseline 中但不存在于当前结果中）
- 新增问题（存在于当前结果中但不存在于 baseline 中）
- 将回归章节追加到报告中

---

## 健康评分规则

先计算各类别分数（0-100），然后取加权平均值。

### Console（权重：15%）
- 0 个错误 → 100
- 1-3 个错误 → 70
- 4-10 个错误 → 40
- 10+ 个错误 → 10

### Links（权重：10%）
- 0 个损坏 → 100
- 每个损坏链接 → -15（最低 0）

### 各类别评分（Visual、Functional、UX、Content、Performance、Accessibility）
每个类别初始为 100。每条发现按严重程度扣分：
- Critical issue → -25
- High issue → -15
- Medium issue → -8
- Low issue → -3
每个类别最低为 0。

### 权重
| Category | Weight |
|----------|--------|
| Console | 15% |
| Links | 10% |
| Visual | 10% |
| Functional | 20% |
| UX | 15% |
| Performance | 10% |
| Content | 5% |
| Accessibility | 15% |

### 最终分数
`score = Σ (category_score × weight)`

---

## 框架特定指导

### Next.js
- 检查控制台中是否有 hydration 错误（`Hydration failed`、`Text content did not match`）
- 在网络中监控 `_next/data` 请求，404 表示数据获取损坏
- 测试客户端导航（点击链接，不要只用 `goto`），这样才能发现路由问题
- 检查包含动态内容的页面是否有 CLS（Cumulative Layout Shift）

### Rails
- 检查控制台中是否有 N+1 查询警告（如果是开发模式）
- 验证表单中是否存在 CSRF token
- 测试 Turbo/Stimulus 集成，页面切换是否顺畅？
- 检查 flash message 是否正常出现并可正确关闭

### WordPress
- 检查插件冲突（不同插件导致的 JS 错误）
- 验证登录用户是否能看到 admin bar
- 测试 REST API endpoints（`/wp-json/`）
- 检查 mixed content 警告（在 WP 中很常见）

### 通用 SPA（React、Vue、Angular）
- 导航时使用 `snapshot -i`，因为 `links` 命令会漏掉客户端路由
- 检查陈旧状态（离开再返回，数据是否会刷新？）
- 测试浏览器前进/后退，应用是否正确处理历史记录？
- 检查内存泄漏（长时间使用后观察控制台）

---

## 重要规则

1. **可复现性就是一切。** 每个问题至少需要一张截图。没有例外。
2. **记录前先验证。** 问题至少重试一次，确认它可复现，而不是偶发异常。
3. **绝不包含凭据。** 在复现步骤中，密码一律写 `[REDACTED]`。
4. **增量写入。** 每发现一个问题就追加到报告中。不要集中批量写。
5. **绝不阅读源码。** 以用户身份测试，而不是以开发者身份。
6. **每次交互后都检查控制台。** 即使视觉上没有表现出来，JS 错误仍然是 bug。
7. **像用户一样测试。** 使用真实的数据。把完整流程从头到尾走一遍。
8. **重质量，不重量。** 5-10 个有充分证据、文档完整的问题 > 20 个含糊描述。
9. **绝不删除输出文件。** 截图和报告会持续累积，这是有意设计。
10. **对棘手 UI 使用 `snapshot -C`。** 它能找到无障碍树漏掉的可点击 div。
11. **向用户展示截图。** 每次执行 `$B screenshot`、`$B snapshot -a -o` 或 `$B responsive` 命令后，都要使用 Read 工具读取输出文件，让用户能内联看到它们。对于 `responsive`（3 个文件），三个都要 Read。这一点至关重要，否则用户看不到截图。
12. **绝不要拒绝使用浏览器。** 当用户调用 /qa 或 /qa-only 时，他们就是在请求基于浏览器的测试。绝不要建议用 evals、单元测试或其他替代方案来代替。即使 diff 看起来没有 UI 变更，后端变更也会影响应用行为，因此始终要打开浏览器并测试。

---

## 输出

将报告同时写入本地位置和项目级位置：

**本地：** `.gstack/qa-reports/qa-report-{domain}-{YYYY-MM-DD}.md`

**项目级：** 写入供跨会话上下文使用的测试结果产物：
```bash
source <(~/.claude/skills/gstack/bin/gstack-slug 2>/dev/null) && mkdir -p ~/.gstack/projects/$SLUG
```
写入 `~/.gstack/projects/{slug}/{user}-{branch}-test-outcome-{datetime}.md`

### 输出结构

```
.gstack/qa-reports/
├── qa-report-{domain}-{YYYY-MM-DD}.md    # 结构化报告
├── screenshots/
│   ├── initial.png                        # 落地页带标注截图
│   ├── issue-001-step-1.png               # 每个问题的证据
│   ├── issue-001-result.png
│   └── ...
└── baseline.json                          # 用于 regression mode
```

报告文件名使用域名和日期：`qa-report-myapp-com-2026-03-12.md`

---

## Additional Rules（qa-only 专用）

11. **绝不修复 bug。** 只发现并记录，不要读源码、编辑文件，也不要在报告中给出修复建议。你的职责是报告哪里坏了，而不是修复它。测试-修复-验证循环请使用 `/qa`。
12. **未检测到测试框架？** 如果项目没有测试基础设施（没有测试配置文件，也没有测试目录），则在报告摘要中加入：“No test framework detected. Run `/qa` to bootstrap one and enable regression test generation.”