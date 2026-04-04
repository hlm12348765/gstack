# TODOS.md 格式参考

规范 `TODOS.md` 格式的共享参考。被 `/ship`（步骤 5.5）和 `/plan-ceo-review`（`TODOS.md` 更新部分）引用，以确保 TODO 条目结构保持一致。

---

## 文件结构

```markdown
# TODOS

## <Skill/Component>     ← 例如，## Browse、## Ship、## Review、## Infrastructure
<items sorted P0 first, then P1, P2, P3, P4>

## Completed
<finished items with completion annotation>
```

**分区：** 按技能或组件组织（`## Browse`、`## Ship`、`## Review`、`## QA`、`## Retro`、`## Infrastructure`）。在每个分区内，按优先级排序条目（P0 置顶）。

---

## TODO 条目格式

每个条目都是其所属分区下的一个 H3 标题：

```markdown
### <Title>

**What:** 工作内容的一行描述。

**Why:** 它解决的具体问题，或它带来的价值。

**Context:** 提供足够的细节，使得三个月后接手这项工作的人能够理解其动机、当前状态以及从哪里开始。

**Effort:** S / M / L / XL
**Priority:** P0 / P1 / P2 / P3 / P4
**Depends on:** <prerequisites, or "None">
```

**必填字段：** What、Why、Context、Effort、Priority
**可选字段：** Depends on、Blocked by

---

## 优先级定义

- **P0** — 阻塞：必须在下一个版本发布前完成
- **P1** — 关键：应在本周期内完成
- **P2** — 重要：在 P0/P1 清空后处理
- **P3** — 锦上添花：在有采用/使用数据后再回顾
- **P4** — 未来某时：是个好想法，但不紧急

---

## 已完成条目格式

当某个条目完成时，将其移动到 `## Completed` 分区，保留其原始内容，并追加：

```markdown
**Completed:** vX.Y.Z (YYYY-MM-DD)
```