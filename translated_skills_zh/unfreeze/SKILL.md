---
name: unfreeze
version: 0.1.0
description: |
  清除由 /freeze 设置的冻结边界，再次允许编辑所有目录。
  当你想在不结束会话的情况下扩大编辑范围时使用。
  当被要求“unfreeze”“unlock edits”“remove freeze”或
  “allow all edits”时使用。
allowed-tools:
  - Bash
  - Read
---
<!-- AUTO-GENERATED from SKILL.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->

# /unfreeze — 清除冻结边界

移除由 `/freeze` 设置的编辑限制，允许编辑所有目录。

```bash
mkdir -p ~/.gstack/analytics
echo '{"skill":"unfreeze","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","repo":"'$(basename "$(git rev-parse --show-toplevel 2>/dev/null)" 2>/dev/null || echo "unknown")'"}'  >> ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true
```

## 清除边界

```bash
STATE_DIR="${CLAUDE_PLUGIN_DATA:-$HOME/.gstack}"
if [ -f "$STATE_DIR/freeze-dir.txt" ]; then
  PREV=$(cat "$STATE_DIR/freeze-dir.txt")
  rm -f "$STATE_DIR/freeze-dir.txt"
  echo "冻结边界已清除（之前为：$PREV）。现在已允许在所有位置进行编辑。"
else
  echo "未设置冻结边界。"
fi
```

将结果告知用户。注意，`/freeze` 的 hooks 仍然为当前会话保持注册状态；只是由于不存在状态文件，它们将允许所有操作。若要重新冻结，请再次运行 `/freeze`。