---
name: setup-browser-cookies
version: 1.0.0
description: |
  将真实浏览器（Comet、Chrome、Arc、Brave、Edge）中的 cookies 导入到
  无头浏览会话中。会打开一个交互式选择界面，由你选择要导入哪些
  cookie 域名。在对需要身份验证的页面进行 QA 测试前使用。当被要求
  “导入 cookies”、“登录网站”或“验证浏览器身份”时使用。
allowed-tools:
  - Bash
  - Read
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
echo '{"skill":"setup-browser-cookies","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","repo":"'$(basename "$(git rev-parse --show-toplevel 2>/dev/null)" 2>/dev/null || echo "unknown")'"}'  >> ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true
for _PF in ~/.gstack/analytics/.pending-*; do [ -f "$_PF" ] && ~/.claude/skills/gstack/bin/gstack-telemetry-log --event-type skill_run --skill _pending_finalize --outcome unknown --session-id "$_SESSION_ID" 2>/dev/null || true; break; done
```

如果 `PROACTIVE` 是 `"false"`，不要主动建议 gstack skills，只有在用户明确要求时才调用。
用户已选择不接收主动建议。

如果输出显示 `UPGRADE_AVAILABLE <old> <new>`：读取 `~/.claude/skills/gstack/gstack-upgrade/SKILL.md` 并遵循“Inline upgrade flow”（如果已配置则自动升级，否则使用 AskUserQuestion 提供 4 个选项；如果用户拒绝，则写入 snooze 状态）。如果显示 `JUST_UPGRADED <from> <to>`：告诉用户 “Running gstack v{to} (just updated!)”，然后继续。

如果 `LAKE_INTRO` 是 `no`：继续之前，先介绍完整性原则。
告诉用户：“gstack 遵循 **Boil the Lake** 原则：当 AI 使边际成本接近于零时，就始终把事情完整做完。详见：https://garryslist.org/posts/boil-the-ocean”
然后询问是否要在默认浏览器中打开这篇文章：

```bash
open https://garryslist.org/posts/boil-the-ocean
touch ~/.gstack/.completeness-intro-seen
```

只有在用户同意时才运行 `open`。始终运行 `touch` 以标记为已查看。这只会发生一次。

如果 `TEL_PROMPTED` 是 `no` 且 `LAKE_INTRO` 是 `yes`：在完成 lake intro 后，
询问用户是否开启 telemetry。使用 AskUserQuestion：

> 帮助 gstack 变得更好！Community mode 会通过稳定的设备 ID 共享使用数据（你使用了哪些 skills、耗时多久、崩溃信息），这样我们就能追踪趋势并更快修复问题。
> 绝不会发送代码、文件路径或仓库名称。
> 你可以随时用 `gstack-config set telemetry off` 更改。

选项：
- A) 帮助 gstack 变得更好！（推荐）
- B) 不用了，谢谢

如果选 A：运行 `~/.claude/skills/gstack/bin/gstack-config set telemetry community`

如果选 B：继续用 AskUserQuestion 追问：

> 那匿名模式怎么样？我们只会知道 *有人* 使用了 gstack，不会有唯一 ID，
> 也无法关联各次会话。只是一个计数器，帮助我们确认是否真的有人在使用。

选项：
- A) 可以，匿名模式没问题
- B) 不用了，完全关闭

如果 B→A：运行 `~/.claude/skills/gstack/bin/gstack-config set telemetry anonymous`
如果 B→B：运行 `~/.claude/skills/gstack/bin/gstack-config set telemetry off`

始终运行：
```bash
touch ~/.gstack/.telemetry-prompted
```

这只会发生一次。如果 `TEL_PROMPTED` 是 `yes`，则完全跳过。

## AskUserQuestion 格式

**每一次调用 AskUserQuestion 时都必须遵循以下结构：**
1. **重新锚定：** 说明项目、当前分支（使用前言中打印出的 `_BRANCH` 值，而不是对话历史或 gitStatus 中的任何分支），以及当前计划/任务。（1-2 句）
2. **简化说明：** 用一个聪明的 16 岁学生也能理解的朴素英文解释问题。不要出现原始函数名、内部术语或实现细节。使用具体例子和类比。说明它“做什么”，而不是“它叫什么”。
3. **给出建议：** `RECOMMENDATION: Choose [X] because [one-line reason]`，始终优先推荐完整方案，而不是捷径（参见 Completeness Principle）。为每个选项都标明 `Completeness: X/10`。校准标准：10 = 完整实现（覆盖所有边界情况，完整覆盖），7 = 覆盖主要路径但省略部分边界情况，3 = 推迟大量工作的捷径。如果两个选项都在 8 分以上，选更高的；如果其中一个 ≤5，要明确提示。
4. **选项：** 使用字母选项：`A) ... B) ... C) ...`。当某个选项涉及投入时，同时显示两个尺度：`(human: ~X / CC: ~Y)`

假设用户已经 20 分钟没有看这个窗口，而且也没有打开代码。如果你需要重新读源码才能理解自己的解释，那说明内容太复杂了。

各 skill 的专属说明可以在此基础上增加额外格式规则。

## Completeness Principle — Boil the Lake

AI 辅助编码让“完整做到位”的边际成本接近于零。当你给出选项时：

- 如果选项 A 是完整实现（完全对齐、覆盖所有边界情况、100% 覆盖），而选项 B 是只节省少量工作的捷径，**始终推荐 A**。在 CC+gstack 下，80 行和 150 行之间的差异微不足道。当“完整”只多花几分钟时，“差不多就行”是错误的直觉。
- **Lake vs. ocean：** “lake” 是可以煮沸的，例如一个模块的 100% 测试覆盖、一个功能的完整实现、处理所有边界情况、覆盖完整错误路径。“ocean” 则不是，例如从零重写整个系统、给你无法控制的依赖添加功能、跨多个季度的平台迁移。推荐煮沸 lakes。明确指出 oceans 超出范围。
- **估算工作量时**，始终同时给出两个尺度：人工团队时间和 CC+gstack 时间。压缩比会因任务类型而异，参考如下：

| 任务类型 | 人工团队 | CC+gstack | 压缩比 |
|-----------|-----------|-----------|-------------|
| 模板代码 / 脚手架 | 2 天 | 15 分钟 | ~100x |
| 编写测试 | 1 天 | 15 分钟 | ~50x |
| 功能实现 | 1 周 | 30 分钟 | ~30x |
| 修复 bug + 回归测试 | 4 小时 | 15 分钟 | ~20x |
| 架构 / 设计 | 2 天 | 4 小时 | ~5x |
| 调研 / 探索 | 1 天 | 3 小时 | ~3x |

- 这个原则适用于测试覆盖、错误处理、文档、边界情况和功能完整性。不要为了“节省时间”而跳过最后 10%，因为有了 AI，这 10% 的成本只要几秒钟。

**反面模式：不要这样做：**
- 错误示例：“选择 B，它用更少的代码覆盖了 90% 的价值。”（如果 A 只多 70 行，就该选 A。）
- 错误示例：“我们可以跳过边界情况处理来节省时间。”（在 CC 的帮助下，处理边界情况只要几分钟。）
- 错误示例：“我们把测试覆盖放到后续 PR 再做。”（测试是最便宜、最该煮沸的 lake。）
- 错误示例：只报人工团队工作量：“这要 2 周。”（应该说：“人工 2 周 / CC 约 1 小时。”）

## Repo Ownership Mode — See Something, Say Something

前言中的 `REPO_MODE` 表示这个仓库里的问题归谁负责：

- **`solo`** — 一个人完成 80% 以上的工作。他负责一切。当你注意到当前分支变更之外的问题时（测试失败、弃用警告、安全告警、lint 错误、死代码、环境问题），**要主动调查并提出修复**。这个独立开发者是唯一会修它的人。默认直接行动。
- **`collaborative`** — 有多个活跃贡献者。当你注意到当前分支变更之外的问题时，**通过 AskUserQuestion 提示出来**，因为那可能属于别人的职责。默认先询问，不直接修复。
- **`unknown`** — 按 collaborative 处理（更安全的默认做法：先问再修）。

**See Something, Say Something：** 无论在哪个工作流步骤中，只要你发现看起来不对劲的地方，不只是测试失败，都要简要指出。一句话说明：你发现了什么，以及它会造成什么影响。在 solo 模式下，接着问一句 “Want me to fix it?”。在 collaborative 模式下，只需指出然后继续。

不要让你注意到的问题悄悄略过。这个机制的核心就是主动沟通。

## Search Before Building

在构建基础设施、遇到陌生模式，或任何运行时可能已有内建能力的情况之前，**先搜索。** 完整理念请阅读 `~/.claude/skills/gstack/ETHOS.md`。

**三层知识：**
- **Layer 1**（久经验证，已广泛存在）。不要重复造轮子。但检查一下的成本接近于零，而偶尔，质疑这些“老办法”正是灵感出现的地方。
- **Layer 2**（新且流行，应该搜索这些）。但要仔细审视：人类会受到狂热情绪影响。搜索结果只是思考输入，不是答案。
- **Layer 3**（第一性原理，高于一切）。基于对具体问题的推理得出的原创观察。这是最有价值的。

**Eureka moment：** 当基于第一性原理的推理揭示出传统认知是错误的时，要明确指出：
“EUREKA: Everyone does X because [assumption]. But [evidence] shows this is wrong. Y is better because [reasoning].”

记录 eureka moments：
```bash
jq -n --arg ts "$(date -u +%Y-%m-%dT%H:%M:%SZ)" --arg skill "SKILL_NAME" --arg branch "$(git branch --show-current 2>/dev/null)" --arg insight "ONE_LINE_SUMMARY" '{ts:$ts,skill:$skill,branch:$branch,insight:$insight}' >> ~/.gstack/analytics/eureka.jsonl 2>/dev/null || true
```

将 SKILL_NAME 和 ONE_LINE_SUMMARY 替换为实际内容。内联运行，不要中断工作流。

**WebSearch 回退：** 如果 WebSearch 不可用，则跳过搜索步骤，并说明：“Search unavailable — proceeding with in-distribution knowledge only.”

## Contributor Mode

如果 `_CONTRIB` 是 `true`：你处于 **contributor mode**。你既是 gstack 用户，也会帮助它变得更好。

**在每个主要工作流步骤结束时**（不是每执行一条命令后），回顾你刚才使用的 gstack 工具。给这次体验打 0 到 10 分。如果不是 10 分，想想原因。如果存在明显且可执行的 bug，或者有值得记录的、有洞察力的改进点，说明 gstack 代码或 skill markdown 本可以做得更好，就提交一份 field report。也许我们的 contributor 会帮忙把它做得更好！

**评分校准标准如下：** 例如，`$B js "await fetch(...)"` 曾因 `SyntaxError: await is only valid in async functions` 失败，因为 gstack 没有把表达式包在 async 上下文里。问题虽小，但输入是合理的，gstack 本应正确处理，这类问题就值得提交。

**不值得提交的内容：** 用户应用本身的 bug、访问用户 URL 时的网络错误、用户站点的认证失败、用户自己的 JS 逻辑错误。

**提交方式：** 在 `~/.gstack/contributor-logs/{slug}.md` 中写入**以下所有章节**（不要截断，必须包含直到 Date/Version 页脚在内的全部部分）：

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

Slug 规则：全小写、使用连字符、最长 60 个字符（例如 `browse-js-no-await`）。如果文件已存在则跳过。每个会话最多 3 份报告。内联提交然后继续，不要中断工作流。告诉用户：“Filed gstack field report: {title}”

## Completion Status Protocol

完成 skill 工作流时，使用以下状态之一报告结果：
- **DONE** — 所有步骤都已成功完成。每项结论都提供了证据。
- **DONE_WITH_CONCERNS** — 已完成，但有用户需要知道的问题。逐条列出每个问题。
- **BLOCKED** — 无法继续。说明阻塞原因以及你尝试过什么。
- **NEEDS_CONTEXT** — 缺少继续所需的信息。明确说明你需要什么。

### Escalation

在任何时候停下来并说“这对我来说太难了”或“我对这个结果没有把握”都是可以的。

做得差比不做更糟。你不会因为升级问题而受到惩罚。
- 如果你已经尝试某项任务 3 次仍未成功，停止并升级处理。
- 如果你对某项安全敏感的变更没有把握，停止并升级处理。
- 如果工作范围超出你能验证的程度，停止并升级处理。

升级格式：
```
STATUS: BLOCKED | NEEDS_CONTEXT
REASON: [1-2 sentences]
ATTEMPTED: [what you tried]
RECOMMENDATION: [what the user should do next]
```

## Telemetry（最后运行）

在 skill 工作流结束后（成功、报错或中止），记录 telemetry 事件。
从本文件 YAML frontmatter 中的 `name:` 字段确定 skill 名称。
根据工作流结果确定 outcome（正常完成则为 success，失败则为 error，
用户中断则为 abort）。

**PLAN MODE 例外规则——始终运行：** 此命令会将 telemetry 写入
`~/.gstack/analytics/`（用户配置目录，而不是项目文件）。skill 的
前言已经会写入同一个目录，这是相同的模式。
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
如果你无法确定 outcome，请使用 `"unknown"`。该命令在后台运行，
绝不会阻塞用户。

## Plan Status Footer

当你处于 plan mode 并且即将调用 ExitPlanMode 时：

1. 检查计划文件中是否已经有 `## GSTACK REVIEW REPORT` 小节。
2. 如果有，跳过（说明某个 review skill 已经写入了更丰富的报告）。
3. 如果没有，运行以下命令：

\`\`\`bash
~/.claude/skills/gstack/bin/gstack-review-read
\`\`\`

然后在计划文件末尾写入一个 `## GSTACK REVIEW REPORT` 小节：

- 如果输出中包含 review 条目（`---CONFIG---` 之前的 JSONL 行）：按
  标准报告表格格式写入 runs/status/findings，格式与 review
  skills 使用的格式相同。
- 如果输出是 `NO_REVIEWS` 或为空：写入下面这个占位表格：

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

**PLAN MODE 例外规则——始终运行：** 这会写入计划文件，而计划文件是
在 plan mode 下唯一允许编辑的文件。计划文件中的 review report 是该计划
实时状态的一部分。

# Setup Browser Cookies

将真实 Chromium 浏览器中已登录的会话导入到无头浏览会话中。

## 工作原理

1. 找到 browse 二进制文件
2. 运行 `cookie-import-browser` 来检测已安装的浏览器并打开选择界面
3. 用户在浏览器中选择要导入哪些 cookie 域名
4. cookies 会被解密并加载到 Playwright 会话中

## 步骤

### 1. 找到 browse 二进制文件

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
3. 如果没有安装 `bun`：`curl -fsSL https://bun.sh/install | bash`

### 2. 打开 cookie 选择器

```bash
$B cookie-import-browser
```

这会自动检测已安装的 Chromium 浏览器（Comet、Chrome、Arc、Brave、Edge），并在默认浏览器中打开
一个交互式选择界面，你可以在其中：
- 在已安装的浏览器之间切换
- 搜索域名
- 点击 “+” 导入某个域名的 cookies
- 点击垃圾桶图标移除已导入的 cookies

告诉用户：**“Cookie picker opened — select the domains you want to import in your browser, then tell me when you're done.”**

### 3. 直接导入（可选方式）

如果用户直接指定了域名（例如 `/setup-browser-cookies github.com`），跳过界面：

```bash
$B cookie-import-browser comet --domain github.com
```

如果指定了其他浏览器，请将 `comet` 替换为相应浏览器。

### 4. 验证

在用户确认完成后：

```bash
$B cookies
```

向用户展示已导入 cookies 的摘要（各域名数量）。

## 说明

- 每个浏览器的首次导入可能会触发 macOS Keychain 对话框，点击 “Allow” / “Always Allow”
- Cookie picker 与 browse server 使用同一个端口（无需额外进程）
- 界面中只显示域名和 cookie 数量，不会暴露 cookie 值
- browse 会话会在命令之间保留 cookies，因此导入后会立即生效