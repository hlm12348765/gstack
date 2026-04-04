---
name: setup-deploy
version: 1.0.0
description: |
  为 `/land-and-deploy` 配置部署设置。检测你的部署平台
  （Fly.io、Render、Vercel、Netlify、Heroku、GitHub Actions、自定义），
  生产 URL、健康检查端点以及部署状态命令。将
  配置写入 `CLAUDE.md`，这样以后所有部署都会自动进行。
  使用场景："setup deploy"、"configure deployment"、"set up land-and-deploy"、
  "how do I deploy with gstack"、"add deploy config"。
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
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
echo '{"skill":"setup-deploy","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","repo":"'$(basename "$(git rev-parse --show-toplevel 2>/dev/null)" 2>/dev/null || echo "unknown")'"}'  >> ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true
for _PF in ~/.gstack/analytics/.pending-*; do [ -f "$_PF" ] && ~/.claude/skills/gstack/bin/gstack-telemetry-log --event-type skill_run --skill _pending_finalize --outcome unknown --session-id "$_SESSION_ID" 2>/dev/null || true; break; done
```

如果 `PROACTIVE` 是 `"false"`，不要主动推荐 gstack 技能，只在用户明确要求时调用。
用户已选择退出主动建议。

如果输出显示 `UPGRADE_AVAILABLE <old> <new>`：读取 `~/.claude/skills/gstack/gstack-upgrade/SKILL.md`，并遵循“Inline upgrade flow”（如果已配置则自动升级，否则使用 AskUserQuestion 提供 4 个选项；如果用户拒绝，则写入 snooze 状态）。如果显示 `JUST_UPGRADED <from> <to>`：告诉用户“正在运行 gstack v{to}（刚刚已更新！）”，然后继续。

如果 `LAKE_INTRO` 是 `no`：继续之前，先介绍 Completeness Principle。
告诉用户：“gstack 遵循 **Boil the Lake** 原则：当 AI 使边际成本接近零时，始终把事情完整做完。阅读更多：https://garryslist.org/posts/boil-the-ocean”
然后询问是否要在默认浏览器中打开这篇文章：

```bash
open https://garryslist.org/posts/boil-the-ocean
touch ~/.gstack/.completeness-intro-seen
```

只有在用户同意时才运行 `open`。始终运行 `touch` 以标记为已查看。这只会发生一次。

如果 `TEL_PROMPTED` 是 `no` 且 `LAKE_INTRO` 是 `yes`：在完成 lake 介绍后，
询问用户是否愿意启用 telemetry。使用 AskUserQuestion：

> 帮助 gstack 变得更好！社区模式会通过稳定的设备 ID 共享使用数据（你使用了哪些技能、耗时多久、崩溃信息），这样我们就能跟踪趋势并更快修复问题。
> 绝不会发送代码、文件路径或仓库名称。
> 可随时通过 `gstack-config set telemetry off` 修改。

选项：
- A) 帮助 gstack 变得更好！（推荐）
- B) 不用了，谢谢

如果选 A：运行 `~/.claude/skills/gstack/bin/gstack-config set telemetry community`

如果选 B：继续使用 AskUserQuestion 追问：

> 那匿名模式怎么样？我们只会知道 *有人* 使用了 gstack，不会有唯一 ID，
> 也无法关联不同会话。只是一个计数器，帮助我们知道是否真的有人在使用。

选项：
- A) 可以，匿名模式没问题
- B) 不用了，完全关闭

如果 B→A：运行 `~/.claude/skills/gstack/bin/gstack-config set telemetry anonymous`
如果 B→B：运行 `~/.claude/skills/gstack/bin/gstack-config set telemetry off`

始终运行：
```bash
touch ~/.gstack/.telemetry-prompted
```

这只会发生一次。如果 `TEL_PROMPTED` 是 `yes`，则完全跳过这一部分。

## AskUserQuestion 格式

**每次调用 AskUserQuestion 都必须遵循以下结构：**
1. **重新锚定：** 说明项目、当前分支（使用前言中打印的 `_BRANCH` 值，而不是对话历史或 gitStatus 中的任何分支），以及当前计划/任务。（1-2 句）
2. **简化说明：** 用一个聪明的 16 岁青少年也能理解的直白英语解释问题。不要出现原始函数名、内部术语或实现细节。使用具体示例和类比。说明它“做什么”，而不是“叫什么”。
3. **给出建议：** `RECOMMENDATION: Choose [X] because [one-line reason]`，始终优先推荐完整方案而不是捷径（参见 Completeness Principle）。为每个选项标注 `Completeness: X/10`。标定标准：10 = 完整实现（所有边界情况、完整覆盖），7 = 覆盖主路径但跳过部分边界情况，3 = 推迟大量工作的捷径。如果两个选项都在 8 以上，选择更高的；如果其中一个 ≤5，要明确提示。
4. **选项：** 使用字母编号：`A) ... B) ... C) ...`。当选项涉及工作量时，同时显示两个尺度：`(human: ~X / CC: ~Y)`

假设用户已经 20 分钟没看这个窗口，而且没有打开代码。如果你需要重新阅读源码才能理解你自己的解释，那就说明它太复杂了。

各技能的专属说明可以在此基础上增加额外的格式规则。

## Completeness Principle — Boil the Lake

AI 辅助编码使完整性的边际成本接近于零。当你给出选项时：

- 如果选项 A 是完整实现（完全对等、处理所有边界情况、100% 覆盖），而选项 B 是只节省少量工作量的捷径，**始终推荐 A**。在 CC+gstack 下，80 行和 150 行之间的差异毫无意义。当“完整”只多花几分钟时，“差不多就行”就是错误直觉。
- **湖泊与海洋：** “lake” 是可以煮沸的，例如模块的 100% 测试覆盖、完整功能实现、处理所有边界情况、完整错误路径。“ocean” 则不是，例如从零重写整个系统、给你无法控制的依赖增加功能、持续数个季度的平台迁移。建议煮沸 lake。明确指出 ocean 超出范围。
- **估算工作量时**，始终同时展示两个尺度：人工团队时间和 CC+gstack 时间。压缩比会因任务类型而不同，可参考下表：

| 任务类型 | 人工团队 | CC+gstack | 压缩比 |
|-----------|-----------|-----------|-------------|
| Boilerplate / scaffolding | 2 天 | 15 分钟 | ~100x |
| Test writing | 1 天 | 15 分钟 | ~50x |
| Feature implementation | 1 周 | 30 分钟 | ~30x |
| Bug fix + regression test | 4 小时 | 15 分钟 | ~20x |
| Architecture / design | 2 天 | 4 小时 | ~5x |
| Research / exploration | 1 天 | 3 小时 | ~3x |

- 这个原则适用于测试覆盖、错误处理、文档、边界情况和功能完整性。不要为了“省时间”而跳过最后 10%：在 AI 的帮助下，这 10% 只需要几秒。

**反模式：不要这样做：**
- 错误示例：“选 B，它用更少的代码覆盖了 90% 的价值。”（如果 A 只多 70 行，就该选 A。）
- 错误示例：“我们可以跳过边界情况处理来节省时间。”（有了 CC，处理边界情况只需几分钟。）
- 错误示例：“把测试覆盖推迟到后续 PR 再做吧。”（测试是最便宜、最该煮沸的 lake。）
- 错误示例：只给出人工团队工作量：“这大概要 2 周。”（应该说：“人工 2 周 / CC 约 1 小时。”）

## Repo Ownership Mode — 发现问题就说出来

前言中的 `REPO_MODE` 表示这个仓库里的问题归谁负责：

- **`solo`** — 一个人完成 80% 以上的工作。他们对所有内容负责。当你发现当前分支改动之外的问题（测试失败、弃用警告、安全告警、lint 错误、死代码、环境问题）时，**要主动调查并提出修复**。单人开发者是唯一会修复这些问题的人。默认直接行动。
- **`collaborative`** — 有多个活跃贡献者。当你发现当前分支改动之外的问题时，**通过 AskUserQuestion 提示**，因为那可能是其他人的责任。默认先询问，不直接修复。
- **`unknown`** — 按 collaborative 处理（这是更安全的默认做法，先问再修）。

**发现问题就说出来：** 在任何工作流程步骤中，只要你注意到看起来不对的地方，而不仅仅是测试失败，都要简短指出。一句话即可：你注意到了什么，以及它的影响。在 solo 模式下，接着问“要我修复吗？”在 collaborative 模式下，只提示然后继续。

绝不要让已经发现的问题悄悄溜过去。整个原则的重点就是主动沟通。

## 先搜索，再构建

在构建基础设施、不熟悉的模式，或任何运行时可能已有内建能力的东西之前，**先搜索。**
完整理念请阅读 `~/.claude/skills/gstack/ETHOS.md`。

**三层知识：**
- **第 1 层**（久经验证，已广泛分发）。不要重复造轮子。但检查的成本接近于零，而偶尔，质疑这些“久经验证”的东西，正是灵感产生之处。
- **第 2 层**（新且流行，应搜索这些）。但要审慎：人类容易陷入狂热。搜索结果只是你思考的输入，不是答案。
- **第 3 层**（第一性原理，高于一切）。通过对具体问题的推理得出的原创观察。这是最有价值的一层。

**Eureka 时刻：** 当第一性原理推理揭示传统看法是错的时，要明确指出：
“EUREKA：大家都做 X，是因为 [assumption]。但 [evidence] 表明这不对。Y 更好，因为 [reasoning]。”

记录 eureka 时刻：
```bash
jq -n --arg ts "$(date -u +%Y-%m-%dT%H:%M:%SZ)" --arg skill "SKILL_NAME" --arg branch "$(git branch --show-current 2>/dev/null)" --arg insight "ONE_LINE_SUMMARY" '{ts:$ts,skill:$skill,branch:$branch,insight:$insight}' >> ~/.gstack/analytics/eureka.jsonl 2>/dev/null || true
```

将 `SKILL_NAME` 和 `ONE_LINE_SUMMARY` 替换为实际值。内联运行，不要中断工作流程。

**WebSearch 回退：** 如果 WebSearch 不可用，跳过搜索步骤，并注明：“Search unavailable — proceeding with in-distribution knowledge only.”

## Contributor Mode

如果 `_CONTRIB` 是 `true`：你处于 **contributor mode**。你既是 gstack 用户，也会帮助它变得更好。

**在每个主要工作流步骤结束时**（不是每一条命令之后），回顾你使用过的 gstack 工具。给体验打 0 到 10 分。如果不是 10 分，思考原因。如果存在明显、可执行的缺陷，或者对 gstack 代码或技能 markdown 可以改进之处有深刻而有价值的观察，就提交一份 field report。也许我们的贡献者会帮助我们变得更好！

**标定标准：** 例如，`$B js "await fetch(...)"` 过去会因为 `SyntaxError: await is only valid in async functions` 而失败，因为 gstack 没有在异步上下文中包装表达式。这种问题虽小，但输入是合理的，gstack 本应处理好，这就是值得提交的问题。比这影响更小的，就忽略。

**不值得提交：** 用户应用自身的缺陷、访问用户 URL 时的网络错误、用户站点上的认证失败、用户自己的 JS 逻辑错误。

**提交方式：** 写入 `~/.gstack/contributor-logs/{slug}.md`，并包含**以下所有部分**（不要截断，必须包含直到 Date/Version 页脚为止的每一节）：

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
{一句话：gstack 本应如何做得更好}

**Date:** {YYYY-MM-DD} | **Version:** {gstack version} | **Skill:** /{skill}
```

Slug：小写、连字符、最长 60 个字符（例如 `browse-js-no-await`）。如果文件已存在则跳过。每个会话最多 3 份报告。内联提交并继续，不要中断工作流程。告诉用户：“已提交 gstack field report：{title}”

## Completion Status Protocol

完成一个技能工作流时，使用以下状态之一进行报告：
- **DONE** — 所有步骤均已成功完成。为每项结论提供了证据。
- **DONE_WITH_CONCERNS** — 已完成，但存在用户应当知晓的问题。列出每个问题。
- **BLOCKED** — 无法继续。说明阻塞原因以及已经尝试过的内容。
- **NEEDS_CONTEXT** — 缺少继续所需的信息。明确说明你需要什么。

### 升级处理

在任何时候，停下来并说“这对我来说太难了”或“我对这个结果没有把握”都是可以的。

糟糕的工作比不做更糟。你不会因为升级处理而受到惩罚。
- 如果你已经尝试某项任务 3 次仍未成功，停止并升级处理。
- 如果你对安全敏感的修改没有把握，停止并升级处理。
- 如果工作范围超出你可验证的范围，停止并升级处理。

升级处理格式：
```
STATUS: BLOCKED | NEEDS_CONTEXT
REASON: [1-2 句话]
ATTEMPTED: [你尝试了什么]
RECOMMENDATION: [用户下一步应该做什么]
```

## Telemetry（最后运行）

在技能工作流完成后（成功、错误或中止），记录 telemetry 事件。
从本文件 YAML frontmatter 中的 `name:` 字段确定技能名。
根据工作流结果确定 outcome（如果正常完成则为 success，
如果失败则为 error，如果被用户中断则为 abort）。

**PLAN MODE 例外 —— 必须始终运行：** 此命令会将 telemetry 写入
`~/.gstack/analytics/`（用户配置目录，而不是项目文件）。技能
前言已经写入同一目录，这属于相同模式。
跳过此命令会丢失会话持续时间和结果数据。

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
success/error/abort，并根据是否使用过 `$B` 将 `USED_BROWSE` 替换为 true/false。
如果你无法确定 outcome，请使用 `"unknown"`。该命令会在后台运行，
绝不会阻塞用户。

## Plan Status Footer

当你处于 plan mode 并且即将调用 ExitPlanMode 时：

1. 检查计划文件中是否已经存在 `## GSTACK REVIEW REPORT` 部分。
2. 如果**存在**，跳过（说明某个 review 技能已经写入了更完整的报告）。
3. 如果**不存在**，运行以下命令：

\`\`\`bash
~/.claude/skills/gstack/bin/gstack-review-read
\`\`\`

然后在计划文件末尾写入一个 `## GSTACK REVIEW REPORT` 部分：

- 如果输出中包含 review 条目（`---CONFIG---` 之前的 JSONL 行）：按
  review 技能所用的相同格式，写出标准报告表，包含每个技能的 runs/status/findings。
- 如果输出为 `NO_REVIEWS` 或为空：写入以下占位表格：

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

**PLAN MODE 例外 —— 必须始终运行：** 这会写入计划文件，而那是 plan mode 中唯一允许编辑的文件。计划文件中的 review 报告是计划持续状态的一部分。

# /setup-deploy — 为 gstack 配置部署

你正在帮助用户配置他们的部署，以便 `/land-and-deploy` 能够
自动工作。你的任务是检测部署平台、生产 URL、健康检查以及部署状态命令，
然后把这些内容持久化到 `CLAUDE.md`。

此流程运行一次后，`/land-and-deploy` 会读取 `CLAUDE.md` 并完全跳过检测。

## 用户可调用
当用户输入 `/setup-deploy` 时，运行此技能。

## 说明

### 第 1 步：检查现有配置

```bash
grep -A 20 "## Deploy Configuration" CLAUDE.md 2>/dev/null || echo "NO_CONFIG"
```

如果配置已存在，显示它并询问：

- **上下文：** `CLAUDE.md` 中已存在部署配置。
- **RECOMMENDATION:** 如果你的设置发生了变化，选择 A 进行更新。
- A) 从头重新配置（覆盖现有配置）
- B) 编辑特定字段（显示当前配置，让我只改一项）
- C) 完成 — 配置看起来正确

如果用户选择 C，则停止。

### 第 2 步：检测平台

运行部署 bootstrap 中的平台检测：

```bash
# 平台配置文件
[ -f fly.toml ] && echo "PLATFORM:fly" && cat fly.toml
[ -f render.yaml ] && echo "PLATFORM:render" && cat render.yaml
[ -f vercel.json ] || [ -d .vercel ] && echo "PLATFORM:vercel"
[ -f netlify.toml ] && echo "PLATFORM:netlify" && cat netlify.toml
[ -f Procfile ] && echo "PLATFORM:heroku"
[ -f railway.json ] || [ -f railway.toml ] && echo "PLATFORM:railway"

# GitHub Actions 部署工作流
for f in .github/workflows/*.yml .github/workflows/*.yaml; do
  [ -f "$f" ] && grep -qiE "deploy|release|production|staging|cd" "$f" 2>/dev/null && echo "DEPLOY_WORKFLOW:$f"
done

# 项目类型
[ -f package.json ] && grep -q '"bin"' package.json 2>/dev/null && echo "PROJECT_TYPE:cli"
ls *.gemspec 2>/dev/null && echo "PROJECT_TYPE:library"
```

### 第 3 步：按平台执行配置

根据检测结果，引导用户完成特定平台的配置。

#### Fly.io

如果检测到 `fly.toml`：

1. 提取应用名：`grep -m1 "^app" fly.toml | sed 's/app = "\(.*\)"/\1/'`
2. 检查是否安装了 `fly` CLI：`which fly 2>/dev/null`
3. 如果已安装，进行验证：`fly status --app {app} 2>/dev/null`
4. 推断 URL：`https://{app}.fly.dev`
5. 设置部署状态命令：`fly status --app {app}`
6. 设置健康检查：`https://{app}.fly.dev`（如果应用有 `/health`，则使用它）

询问用户确认生产 URL。有些 Fly 应用会使用自定义域名。

#### Render

如果检测到 `render.yaml`：

1. 从 render.yaml 提取服务名称和类型
2. 检查是否有 Render API key：`echo $RENDER_API_KEY | head -c 4`（不要暴露完整 key）
3. 推断 URL：`https://{service-name}.onrender.com`
4. Render 会在推送到已连接分支时自动部署，不需要部署工作流
5. 设置健康检查：使用推断出的 URL

询问用户确认。Render 会从已连接的 git 分支自动部署，合并到 main 后，
Render 会自动接管部署。`/land-and-deploy` 中的 “deploy wait”
应轮询 Render URL，直到它响应新版本。

#### Vercel

如果检测到 vercel.json 或 .vercel：

1. 检查是否安装了 `vercel` CLI：`which vercel 2>/dev/null`
2. 如果已安装：`vercel ls --prod 2>/dev/null | head -3`
3. Vercel 会在推送时自动部署：PR 生成 preview，合并到 main 后部署 production
4. 将健康检查设置为 vercel 项目设置中的生产 URL

#### Netlify

如果检测到 `netlify.toml`：

1. 从 netlify.toml 提取站点信息
2. Netlify 会在推送时自动部署
3. 将健康检查设置为生产 URL

#### 仅 GitHub Actions

如果检测到部署工作流但没有平台配置：

1. 读取工作流文件，理解其作用
2. 提取部署目标（如果有提及）
3. 向用户询问生产 URL

#### 自定义 / 手动

如果什么都没有检测到：

使用 AskUserQuestion 收集信息：

1. **部署是如何触发的？**
   - A) 推送到 main 时自动触发（Fly、Render、Vercel、Netlify 等）
   - B) 通过 GitHub Actions 工作流触发
   - C) 通过部署脚本或 CLI 命令触发（请描述）
   - D) 手动触发（SSH、控制台等）
   - E) 这个项目不部署（library、CLI、tool）

2. **生产 URL 是什么？**（自由文本 —— 应用运行的 URL）

3. **gstack 应如何检查部署是否成功？**
   - A) 通过特定 URL 的 HTTP 健康检查（例如 `/health`、`/api/status`）
   - B) CLI 命令（例如 `fly status`、`kubectl rollout status`）
   - C) 检查 GitHub Actions 工作流状态
   - D) 没有自动方式，只要检查 URL 能否打开即可

4. **是否有任何 pre-merge 或 post-merge hooks？**
   - 合并前要运行的命令（例如 `bun run build`）
   - 合并后、部署验证前要运行的命令

### 第 4 步：写入配置

读取 `CLAUDE.md`（如果不存在则创建）。查找并替换 `## Deploy Configuration` 部分；
如果不存在，则追加到文件末尾。

```markdown
## Deploy Configuration (configured by /setup-deploy)
- Platform: {platform}
- Production URL: {url}
- Deploy workflow: {workflow file or "auto-deploy on push"}
- Deploy status command: {command or "HTTP health check"}
- Merge method: {squash/merge/rebase}
- Project type: {web app / API / CLI / library}
- Post-deploy health check: {health check URL or command}

### Custom deploy hooks
- Pre-merge: {command or "none"}
- Deploy trigger: {command or "automatic on push to main"}
- Deploy status: {command or "poll production URL"}
- Health check: {URL or command}
```

### 第 5 步：验证

写入后，验证配置是否有效：

1. 如果配置了健康检查 URL，尝试访问它：
```bash
curl -sf "{health-check-url}" -o /dev/null -w "%{http_code}" 2>/dev/null || echo "UNREACHABLE"
```

2. 如果配置了部署状态命令，尝试运行它：
```bash
{deploy-status-command} 2>/dev/null | head -5 || echo "COMMAND_FAILED"
```

报告结果。如果有任何失败，记录下来但不要阻塞流程。
即使健康检查暂时不可达，这份配置依然有用。

### 第 6 步：总结

```
DEPLOY CONFIGURATION — COMPLETE
════════════════════════════════
Platform:      {platform}
URL:           {url}
Health check:  {health check}
Status cmd:    {status command}
Merge method:  {merge method}

已保存到 CLAUDE.md。/land-and-deploy 会自动使用这些设置。

后续步骤：
- 运行 /land-and-deploy，合并并部署你当前的 PR
- 编辑 CLAUDE.md 中的 "## Deploy Configuration" 部分以修改设置
- 再次运行 /setup-deploy 以重新配置
```

## 重要规则

- **绝不要暴露秘密信息。** 不要打印完整的 API key、token 或密码。
- **与用户确认。** 在写入之前，始终展示检测到的配置并让用户确认。
- **CLAUDE.md 是唯一真实来源。** 所有配置都存放在那里，而不是单独的配置文件。
- **幂等。** 多次运行 `/setup-deploy` 会干净地覆盖之前的配置。
- **平台 CLI 是可选的。** 如果没有安装 `fly` 或 `vercel` CLI，则回退为基于 URL 的健康检查。