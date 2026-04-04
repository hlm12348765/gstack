---
name: freeze
version: 0.1.0
description: |
  将本次会话中的文件编辑限制在特定目录内。阻止在允许路径之外进行 Edit 和
  Write。可在调试时使用，以防意外“修复”无关代码，或在你希望将更改范围限定在
  单个模块时使用。当被要求“freeze”、“restrict edits”、“only edit this folder”，
  或“lock down edits”时使用。
allowed-tools:
  - Bash
  - Read
  - AskUserQuestion
hooks:
  PreToolUse:
    - matcher: "Edit"
      hooks:
        - type: command
          command: "bash ${CLAUDE_SKILL_DIR}/bin/check-freeze.sh"
          statusMessage: "正在检查 freeze 边界..."
    - matcher: "Write"
      hooks:
        - type: command
          command: "bash ${CLAUDE_SKILL_DIR}/bin/check-freeze.sh"
          statusMessage: "正在检查 freeze 边界..."
---
<!-- 从 SKILL.md.tmpl 自动生成——请勿直接编辑 -->
<!-- 重新生成：bun run gen:skill-docs -->

# /freeze — 将编辑限制到某个目录

将文件编辑锁定到特定目录。任何以允许路径之外的文件为目标的 Edit 或 Write 操作都会被**阻止**（而不只是警告）。

```bash
mkdir -p ~/.gstack/analytics
echo '{"skill":"freeze","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","repo":"'$(basename "$(git rev-parse --show-toplevel 2>/dev/null)" 2>/dev/null || echo "unknown")'"}'  >> ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true
```

## 设置

询问用户要将编辑限制到哪个目录。使用 AskUserQuestion：

- 问题："Which directory should I restrict edits to? Files outside this path will be blocked from editing."
- 文本输入（不是多项选择）——由用户输入一个路径。

一旦用户提供目录路径：

1. 将其解析为绝对路径：
```bash
FREEZE_DIR=$(cd "<user-provided-path>" 2>/dev/null && pwd)
echo "$FREEZE_DIR"
```

2. 确保末尾带有斜杠，并保存到 freeze 状态文件：
```bash
FREEZE_DIR="${FREEZE_DIR%/}/"
STATE_DIR="${CLAUDE_PLUGIN_DATA:-$HOME/.gstack}"
mkdir -p "$STATE_DIR"
echo "$FREEZE_DIR" > "$STATE_DIR/freeze-dir.txt"
echo "Freeze boundary set: $FREEZE_DIR"
```

告诉用户："Edits are now restricted to `<path>/`. Any Edit or Write outside this directory will be blocked. To change the boundary, run `/freeze` again. To remove it, run `/unfreeze` or end the session."

## 工作原理

该 hook 会从 Edit/Write 工具输入的 JSON 中读取 `file_path`，然后检查该路径是否以 freeze 目录开头。如果不是，它会返回 `permissionDecision: "deny"` 以阻止该操作。

freeze 边界会通过状态文件在本次会话中持续生效。hook 脚本会在每次调用 Edit/Write 时读取它。

## 说明

- freeze 目录末尾的 `/` 可防止 `/src` 匹配到 `/src-old`
- freeze 仅适用于 Edit 和 Write 工具——Read、Bash、Glob、Grep 不受影响
- 这用于防止意外编辑，而不是安全边界——像 `sed` 这样的 Bash 命令仍然可以修改边界之外的文件
- 要停用它，请运行 `/unfreeze` 或结束对话