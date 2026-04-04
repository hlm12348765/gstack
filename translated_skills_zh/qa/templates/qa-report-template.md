# QA 报告：{APP_NAME}

| 字段 | 值 |
|-------|-------|
| **日期** | {DATE} |
| **URL** | {URL} |
| **分支** | {BRANCH} |
| **提交** | {COMMIT_SHA} ({COMMIT_DATE}) |
| **PR** | {PR_NUMBER} ({PR_URL}) 或 "—" |
| **级别** | 快速 / 标准 / 彻底 |
| **范围** | {SCOPE or "Full app"} |
| **耗时** | {DURATION} |
| **访问页面数** | {COUNT} |
| **截图数** | {COUNT} |
| **框架** | {DETECTED or "Unknown"} |
| **索引** | [所有 QA 运行](./index.md) |

## 健康评分：{SCORE}/100

| 类别 | 分数 |
|----------|-------|
| 控制台 | {0-100} |
| 链接 | {0-100} |
| 视觉 | {0-100} |
| 功能 | {0-100} |
| 用户体验 | {0-100} |
| 性能 | {0-100} |
| 可访问性 | {0-100} |

## 最需要修复的 3 个问题

1. **{ISSUE-NNN}: {title}** — {one-line description}
2. **{ISSUE-NNN}: {title}** — {one-line description}
3. **{ISSUE-NNN}: {title}** — {one-line description}

## 控制台健康状况

| 错误 | 数量 | 首次出现 |
|-------|-------|------------|
| {error message} | {N} | {URL} |

## 概要

| 严重级别 | 数量 |
|----------|-------|
| 严重 | 0 |
| 高 | 0 |
| 中 | 0 |
| 低 | 0 |
| **总计** | **0** |

## 问题

### ISSUE-001: {Short title}

| 字段 | 值 |
|-------|-------|
| **严重级别** | critical / high / medium / low |
| **类别** | visual / functional / ux / content / performance / console / accessibility |
| **URL** | {page URL} |

**描述：** {What is wrong, expected vs actual.}

**复现步骤：**

1. 访问 {URL}
   ![步骤 1](screenshots/issue-001-step-1.png)
2. {Action}
   ![步骤 2](screenshots/issue-001-step-2.png)
3. **观察：** {what goes wrong}
   ![结果](screenshots/issue-001-result.png)

---

## 已应用的修复（如适用）

| 问题 | 修复状态 | 提交 | 变更文件 |
|-------|-----------|--------|---------------|
| ISSUE-NNN | verified / best-effort / reverted / deferred | {SHA} | {files} |

### 修复前/后证据

#### ISSUE-NNN: {title}
**修复前：** ![修复前](screenshots/issue-NNN-before.png)
**修复后：** ![修复后](screenshots/issue-NNN-after.png)

---

## 回归测试

| 问题 | 测试文件 | 状态 | 描述 |
|-------|-----------|--------|-------------|
| ISSUE-NNN | path/to/test | committed / deferred / skipped | description |

### 延后的测试

#### ISSUE-NNN: {title}
**前置条件：** {setup state that triggers the bug}
**操作：** {what the user does}
**预期：** {correct behavior}
**延后原因：** {reason}

---

## 发布就绪情况

| 指标 | 值 |
|--------|-------|
| 健康评分 | {before} → {after} ({delta}) |
| 发现的问题 | N |
| 已应用的修复 | N（verified: X, best-effort: Y, reverted: Z） |
| 延后处理 | N |

**PR 摘要：** "QA found N issues, fixed M, health score X → Y."

---

## 回归情况（如适用）

| 指标 | 基线 | 当前 | 变化 |
|--------|----------|---------|-------|
| 健康评分 | {N} | {N} | {+/-N} |
| 问题数 | {N} | {N} | {+/-N} |

**相较基线已修复：** {list}
**相较基线新增：** {list}