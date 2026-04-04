---
name: careful
version: 0.1.0
description: |
  针对破坏性命令的安全护栏。在执行 `rm -rf`、`DROP TABLE`、
  force-push、`git reset --hard`、`kubectl delete` 以及类似的破坏性操作前发出警告。
  用户可以覆盖每一条警告。当操作 prod、调试在线系统、
  或在共享环境中工作时使用。当被要求“be careful”、“safety mode”、
  “prod mode”或“careful mode”时使用。
allowed-tools:
  - Bash
  - Read
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "bash ${CLAUDE_SKILL_DIR}/bin/check-careful.sh"
          statusMessage: "正在检查破坏性命令..."
---
<!-- AUTO-GENERATED from SKILL.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->

# /careful — 破坏性命令护栏

安全模式现已**启用**。每一条 bash 命令在运行前都会检查是否匹配破坏性模式。
如果检测到破坏性命令，你将收到警告，并且可以选择继续或取消。

```bash
mkdir -p ~/.gstack/analytics
echo '{"skill":"careful","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","repo":"'$(basename "$(git rev-parse --show-toplevel 2>/dev/null)" 2>/dev/null || echo "unknown")'"}'  >> ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true
```

## 受保护的内容

| 模式 | 示例 | 风险 |
|---------|---------|------|
| `rm -rf` / `rm -r` / `rm --recursive` | `rm -rf /var/data` | 递归删除 |
| `DROP TABLE` / `DROP DATABASE` | `DROP TABLE users;` | 数据丢失 |
| `TRUNCATE` | `TRUNCATE orders;` | 数据丢失 |
| `git push --force` / `-f` | `git push -f origin main` | 历史被重写 |
| `git reset --hard` | `git reset --hard HEAD~3` | 未提交工作丢失 |
| `git checkout .` / `git restore .` | `git checkout .` | 未提交工作丢失 |
| `kubectl delete` | `kubectl delete pod` | 对生产环境造成影响 |
| `docker rm -f` / `docker system prune` | `docker system prune -a` | 容器/镜像丢失 |

## 安全例外

以下模式允许直接执行而不发出警告：
- `rm -rf node_modules` / `.next` / `dist` / `__pycache__` / `.cache` / `build` / `.turbo` / `coverage`

## 工作原理

该 hook 会从工具输入 JSON 中读取命令，根据
上述模式进行检查；如果发现匹配项，则返回 `permissionDecision: "ask"` 并附带警告消息。
你始终可以覆盖警告并继续执行。

如需停用，请结束当前对话或重新开始一个新对话。Hooks 的作用范围仅限当前会话。