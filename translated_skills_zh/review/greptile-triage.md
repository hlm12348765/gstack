# Greptile 评论分诊

用于获取、筛选和分类 GitHub PR 上 Greptile 审查评论的共享参考。`/review`（步骤 2.5）和 `/ship`（步骤 3.75）都会引用本文档。

---

## 获取

运行以下命令来检测 PR 并获取评论。两个 API 调用会并行运行。

```bash
REPO=$(gh repo view --json nameWithOwner --jq '.nameWithOwner' 2>/dev/null)
PR_NUMBER=$(gh pr view --json number --jq '.number' 2>/dev/null)
```

**如果任一命令失败或结果为空：** 静默跳过 Greptile 分诊。这个集成是附加能力，缺少它工作流仍然可以运行。

```bash
# 并行获取行级审查评论和 PR 顶层评论
gh api repos/$REPO/pulls/$PR_NUMBER/comments \
  --jq '.[] | select(.user.login == "greptile-apps[bot]") | select(.position != null) | {id: .id, path: .path, line: .line, body: .body, html_url: .html_url, source: "line-level"}' > /tmp/greptile_line.json &
gh api repos/$REPO/issues/$PR_NUMBER/comments \
  --jq '.[] | select(.user.login == "greptile-apps[bot]") | {id: .id, body: .body, html_url: .html_url, source: "top-level"}' > /tmp/greptile_top.json &
wait
```

**如果 API 报错，或两个端点合计为零条 Greptile 评论：** 静默跳过。

行级评论上的 `position != null` 过滤会自动跳过因强制推送代码而过期的评论。

---

## 抑制项检查

推导项目专属的历史记录路径：
```bash
REMOTE_SLUG=$(browse/bin/remote-slug 2>/dev/null || ~/.claude/skills/gstack/browse/bin/remote-slug 2>/dev/null || basename "$(git rev-parse --show-toplevel 2>/dev/null || pwd)")
PROJECT_HISTORY="$HOME/.gstack/projects/$REMOTE_SLUG/greptile-history.md"
```

如果 `$PROJECT_HISTORY` 存在，则读取它（按项目的抑制记录）。每一行记录一次之前的分诊结果：

```
<date> | <repo> | <type:fp|fix|already-fixed> | <file-pattern> | <category>
```

**Categories**（固定集合）：`race-condition`、`null-check`、`error-handling`、`style`、`type-safety`、`security`、`performance`、`correctness`、`other`

将每条已获取评论与满足以下条件的记录进行匹配：
- `type == fp`（只抑制已知误报，不抑制之前已修复的真实问题）
- `repo` 与当前仓库匹配
- `file-pattern` 与评论的文件路径匹配
- `category` 与评论中的问题类型匹配

将匹配到的评论跳过，并标记为 **SUPPRESSED**。

如果历史文件不存在，或存在无法解析的行，则跳过这些行并继续，绝不能因为格式错误的历史文件而失败。

---

## 分类

对于每条未被抑制的评论：

1. **行级评论：** 读取 `path:line` 指定位置的文件内容及周边上下文（±10 行）
2. **顶层评论：** 读取完整评论正文
3. 将评论与完整 diff（`git diff origin/main`）以及审查检查清单交叉比对
4. 分类：
   - **VALID & ACTIONABLE**：当前代码中真实存在的 bug、竞争条件、安全问题或正确性问题
   - **VALID BUT ALREADY FIXED**：真实问题，但已在该分支后续提交中修复。识别出修复该问题的提交 SHA。
   - **FALSE POSITIVE**：评论误解了代码、标记了在别处已处理的情况，或只是风格噪音
   - **SUPPRESSED**：已在上面的抑制项检查中被过滤

---

## 回复 API

回复 Greptile 评论时，请根据评论来源使用正确的端点：

**行级评论**（来自 `pulls/$PR/comments`）：
```bash
gh api repos/$REPO/pulls/$PR_NUMBER/comments/$COMMENT_ID/replies \
  -f body="<reply text>"
```

**顶层评论**（来自 `issues/$PR/comments`）：
```bash
gh api repos/$REPO/issues/$PR_NUMBER/comments \
  -f body="<reply text>"
```

**如果回复 POST 失败**（例如 PR 已关闭、没有写权限）：发出警告并继续。不要因为回复失败而停止工作流。

---

## 回复模板

对每一条 Greptile 回复都使用这些模板。始终包含具体证据，绝不要发布含糊的回复。

### Tier 1（首次回复）- 友好，包含证据

**用于 FIXES（用户选择修复该问题）：**

```
**Fixed** in `<commit-sha>`.

\`\`\`diff
- <old problematic line(s)>
+ <new fixed line(s)>
\`\`\`

**Why:** <1-sentence explanation of what was wrong and how the fix addresses it>
```

**用于 ALREADY FIXED（该问题已在分支上的先前提交中处理）：**

```
**Already fixed** in `<commit-sha>`.

**What was done:** <1-2 sentences describing how the existing commit addresses this issue>
```

**用于 FALSE POSITIVES（该评论不正确）：**

```
**Not a bug.** <1 sentence directly stating why this is incorrect>

**Evidence:**
- <specific code reference showing the pattern is safe/correct>
- <e.g., "The nil check is handled by `ActiveRecord::FinderMethods#find` which raises RecordNotFound, not nil">

**Suggested re-rank:** This appears to be a `<style|noise|misread>` issue, not a `<what Greptile called it>`. Consider lowering severity.
```

### Tier 2（Greptile 在先前回复后再次标记）- 坚定，提供压倒性证据

当下面的升级检测识别出同一线程上已有先前 GStack 回复时，使用 Tier 2。应包含尽可能充分的证据，以结束讨论。

```
**This has been reviewed and confirmed as [intentional/already-fixed/not-a-bug].**

\`\`\`diff
<full relevant diff showing the change or safe pattern>
\`\`\`

**Evidence chain:**
1. <file:line permalink showing the safe pattern or fix>
2. <commit SHA where it was addressed, if applicable>
3. <architecture rationale or design decision, if applicable>

**Suggested re-rank:** Please recalibrate — this is a `<actual category>` issue, not `<claimed category>`. [Link to specific file change permalink if helpful]
```

---

## 升级检测

在编写回复之前，检查此评论线程中是否已经存在先前的 GStack 回复：

1. **对于行级评论：** 通过 `gh api repos/$REPO/pulls/$PR_NUMBER/comments/$COMMENT_ID/replies` 获取回复。检查是否有任一回复正文包含 GStack 标记：`**Fixed**`、`**Not a bug.**`、`**Already fixed**`。

2. **对于顶层评论：** 扫描已获取的 issue 评论，查找在 Greptile 评论之后发布且包含 GStack 标记的回复。

3. **如果已存在先前的 GStack 回复，并且 Greptile 在相同 file+category 上再次发帖：** 使用 Tier 2（坚定）模板。

4. **如果不存在先前的 GStack 回复：** 使用 Tier 1（友好）模板。

如果升级检测失败（API 错误、线程不明确）：默认使用 Tier 1。存在歧义时绝不升级。

---

## 严重性评估与重新分级

在分类评论时，也要评估 Greptile 隐含的严重性是否符合实际：

- 如果 Greptile 将某项标记为 **security/correctness/race-condition** 问题，但它实际上只是 **style/performance** 细节问题：在回复中加入 `**Suggested re-rank:**`，请求更正分类。
- 如果 Greptile 将低严重性的风格问题标记得像关键问题一样严重：在回复中明确提出异议。
- 始终具体说明为什么需要重新分级，引用代码和行号，而不是表达意见。

---

## 历史文件写入

写入之前，确保两个目录都存在：
```bash
REMOTE_SLUG=$(browse/bin/remote-slug 2>/dev/null || ~/.claude/skills/gstack/browse/bin/remote-slug 2>/dev/null || basename "$(git rev-parse --show-toplevel 2>/dev/null || pwd)")
mkdir -p "$HOME/.gstack/projects/$REMOTE_SLUG"
mkdir -p ~/.gstack
```

将每个分诊结果各追加一行到 **两个** 文件中（项目级用于抑制，全局用于回溯）：
- `~/.gstack/projects/$REMOTE_SLUG/greptile-history.md`（按项目）
- `~/.gstack/greptile-history.md`（全局汇总）

格式：
```
<YYYY-MM-DD> | <owner/repo> | <type> | <file-pattern> | <category>
```

示例条目：
```
2026-03-13 | garrytan/myapp | fp | app/services/auth_service.rb | race-condition
2026-03-13 | garrytan/myapp | fix | app/models/user.rb | null-check
2026-03-13 | garrytan/myapp | already-fixed | lib/payments.rb | error-handling
```

---

## 输出格式

在输出头部中包含 Greptile 摘要：
```
+ N Greptile comments (X valid, Y fixed, Z FP)
```

对于每条已分类评论，显示：
- 分类标签：`[VALID]`、`[FIXED]`、`[FALSE POSITIVE]`、`[SUPPRESSED]`
- 文件:行引用（对于行级评论）或 `[top-level]`（对于顶层评论）
- 一行正文摘要
- 永久链接 URL（`html_url` 字段）