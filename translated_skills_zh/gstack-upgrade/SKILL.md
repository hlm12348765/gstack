---
name: gstack-upgrade
version: 1.1.0
description: |
  将 gstack 升级到最新版本。检测是全局安装还是 vendored 安装，
  运行升级，并显示新内容。当被要求“升级 gstack”、
  “更新 gstack”或“获取最新版本”时使用。
allowed-tools:
  - Bash
  - Read
  - Write
  - AskUserQuestion
---
<!-- AUTO-GENERATED from SKILL.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->

# /gstack-upgrade

将 gstack 升级到最新版本并显示新内容。

## 内联升级流程

当检测到 `UPGRADE_AVAILABLE` 时，所有技能前导部分都会引用本节。

### 第 1 步：询问用户（或自动升级）

首先，检查是否启用了自动升级：
```bash
_AUTO=""
[ "${GSTACK_AUTO_UPGRADE:-}" = "1" ] && _AUTO="true"
[ -z "$_AUTO" ] && _AUTO=$(~/.claude/skills/gstack/bin/gstack-config get auto_upgrade 2>/dev/null || true)
echo "AUTO_UPGRADE=$_AUTO"
```

**如果 `AUTO_UPGRADE=true` 或 `AUTO_UPGRADE=1`：** 跳过 AskUserQuestion。记录“Auto-upgrading gstack v{old} → v{new}...”，然后直接进入第 2 步。如果自动升级期间 `./setup` 失败，则从备份（`.bak` 目录）恢复，并警告用户：“Auto-upgrade failed — restored previous version. Run `/gstack-upgrade` manually to retry.”

**否则**，使用 AskUserQuestion：
- 问题："gstack **v{new}** 已可用（你当前使用的是 v{old}）。现在升级吗？"
- 选项：["是，现在升级", "始终保持最新", "暂时不升级", "不再询问"]

**如果选择“是，现在升级”：** 进入第 2 步。

**如果选择“始终保持最新”：**
```bash
~/.claude/skills/gstack/bin/gstack-config set auto_upgrade true
```
告诉用户：“已启用自动升级。后续更新将自动安装。” 然后进入第 2 步。

**如果选择“暂时不升级”：** 以递增退避方式写入 snooze 状态（第一次 snooze = 24h，第二次 = 48h，第三次及之后 = 1 week），然后继续当前技能。不再提及此次升级。
```bash
_SNOOZE_FILE=~/.gstack/update-snoozed
_REMOTE_VER="{new}"
_CUR_LEVEL=0
if [ -f "$_SNOOZE_FILE" ]; then
  _SNOOZED_VER=$(awk '{print $1}' "$_SNOOZE_FILE")
  if [ "$_SNOOZED_VER" = "$_REMOTE_VER" ]; then
    _CUR_LEVEL=$(awk '{print $2}' "$_SNOOZE_FILE")
    case "$_CUR_LEVEL" in *[!0-9]*) _CUR_LEVEL=0 ;; esac
  fi
fi
_NEW_LEVEL=$((_CUR_LEVEL + 1))
[ "$_NEW_LEVEL" -gt 3 ] && _NEW_LEVEL=3
echo "$_REMOTE_VER $_NEW_LEVEL $(date +%s)" > "$_SNOOZE_FILE"
```
注意：`{new}` 是 `UPGRADE_AVAILABLE` 输出中的远程版本，请用更新检查结果中的版本替换它。

告诉用户 snooze 时长：“下次提醒将在 24h 后”（或 48h 或 1 week，取决于级别）。提示：“在 `~/.gstack/config.yaml` 中设置 `auto_upgrade: true` 可启用自动升级。”

**如果选择“不再询问”：**
```bash
~/.claude/skills/gstack/bin/gstack-config set update_check false
```
告诉用户：“已禁用更新检查。运行 `~/.claude/skills/gstack/bin/gstack-config set update_check true` 可重新启用。”
继续当前技能。

### 第 2 步：检测安装类型

```bash
if [ -d "$HOME/.claude/skills/gstack/.git" ]; then
  INSTALL_TYPE="global-git"
  INSTALL_DIR="$HOME/.claude/skills/gstack"
elif [ -d "$HOME/.gstack/repos/gstack/.git" ]; then
  INSTALL_TYPE="global-git"
  INSTALL_DIR="$HOME/.gstack/repos/gstack"
elif [ -d ".claude/skills/gstack/.git" ]; then
  INSTALL_TYPE="local-git"
  INSTALL_DIR=".claude/skills/gstack"
elif [ -d ".agents/skills/gstack/.git" ]; then
  INSTALL_TYPE="local-git"
  INSTALL_DIR=".agents/skills/gstack"
elif [ -d ".claude/skills/gstack" ]; then
  INSTALL_TYPE="vendored"
  INSTALL_DIR=".claude/skills/gstack"
elif [ -d "$HOME/.claude/skills/gstack" ]; then
  INSTALL_TYPE="vendored-global"
  INSTALL_DIR="$HOME/.claude/skills/gstack"
else
  echo "ERROR: gstack not found"
  exit 1
fi
echo "Install type: $INSTALL_TYPE at $INSTALL_DIR"
```

上面输出的安装类型和目录路径将在后续所有步骤中使用。

### 第 3 步：保存旧版本

使用下面第 2 步输出的安装目录：

```bash
OLD_VERSION=$(cat "$INSTALL_DIR/VERSION" 2>/dev/null || echo "unknown")
```

### 第 4 步：升级

使用第 2 步中检测到的安装类型和目录：

**对于 git 安装**（global-git、local-git）：
```bash
cd "$INSTALL_DIR"
STASH_OUTPUT=$(git stash 2>&1)
git fetch origin
git reset --hard origin/main
./setup
```
如果 `$STASH_OUTPUT` 包含 “Saved working directory”，则警告用户：“注意：本地更改已被 stash。运行技能目录中的 `git stash pop` 可恢复这些更改。”

**对于 vendored 安装**（vendored、vendored-global）：
```bash
PARENT=$(dirname "$INSTALL_DIR")
TMP_DIR=$(mktemp -d)
git clone --depth 1 https://github.com/garrytan/gstack.git "$TMP_DIR/gstack"
mv "$INSTALL_DIR" "$INSTALL_DIR.bak"
mv "$TMP_DIR/gstack" "$INSTALL_DIR"
cd "$INSTALL_DIR" && ./setup
rm -rf "$INSTALL_DIR.bak" "$TMP_DIR"
```

### 第 4.5 步：同步本地 vendored 副本

使用第 2 步中的安装目录。检查是否还存在需要更新的本地 vendored 副本：

```bash
_ROOT=$(git rev-parse --show-toplevel 2>/dev/null)
LOCAL_GSTACK=""
if [ -n "$_ROOT" ] && [ -d "$_ROOT/.claude/skills/gstack" ]; then
  _RESOLVED_LOCAL=$(cd "$_ROOT/.claude/skills/gstack" && pwd -P)
  _RESOLVED_PRIMARY=$(cd "$INSTALL_DIR" && pwd -P)
  if [ "$_RESOLVED_LOCAL" != "$_RESOLVED_PRIMARY" ]; then
    LOCAL_GSTACK="$_ROOT/.claude/skills/gstack"
  fi
fi
echo "LOCAL_GSTACK=$LOCAL_GSTACK"
```

如果 `LOCAL_GSTACK` 非空，则通过从刚升级完成的主安装中复制来更新它（与 README 中 vendored 安装的方法相同）：
```bash
mv "$LOCAL_GSTACK" "$LOCAL_GSTACK.bak"
cp -Rf "$INSTALL_DIR" "$LOCAL_GSTACK"
rm -rf "$LOCAL_GSTACK/.git"
cd "$LOCAL_GSTACK" && ./setup
rm -rf "$LOCAL_GSTACK.bak"
```
告诉用户：“另外还更新了位于 `$LOCAL_GSTACK` 的 vendored 副本，准备好后请提交 `.claude/skills/gstack/`。”

如果 `./setup` 失败，则从备份恢复，并警告用户：
```bash
rm -rf "$LOCAL_GSTACK"
mv "$LOCAL_GSTACK.bak" "$LOCAL_GSTACK"
```
告诉用户：“同步失败，已恢复 `$LOCAL_GSTACK` 中的先前版本。请手动运行 `/gstack-upgrade` 重试。”

### 第 5 步：写入标记并清除缓存

```bash
mkdir -p ~/.gstack
echo "$OLD_VERSION" > ~/.gstack/just-upgraded-from
rm -f ~/.gstack/last-update-check
rm -f ~/.gstack/update-snoozed
```

### 第 6 步：显示新内容

读取 `$INSTALL_DIR/CHANGELOG.md`。找出从旧版本到新版本之间的所有版本条目。按主题汇总为 5-7 个要点。不要信息过载，重点放在面向用户的变更上。跳过内部重构，除非它们影响较大。

格式：
```
gstack v{new} — 已从 v{old} 升级！

新内容：
- [要点 1]
- [要点 2]
- ...

祝发布顺利！
```

### 第 7 步：继续

显示完新内容后，继续执行用户最初调用的技能。升级已完成，无需进一步操作。

---

## 独立使用

当直接以 `/gstack-upgrade` 调用时（不是从前导部分调用）：

1. 强制执行一次新的更新检查（绕过缓存）：
```bash
~/.claude/skills/gstack/bin/gstack-update-check --force 2>/dev/null || \
.claude/skills/gstack/bin/gstack-update-check --force 2>/dev/null || true
```
使用输出结果判断是否有可用升级。

2. 如果 `UPGRADE_AVAILABLE <old> <new>`：按照上面的第 2-6 步执行。

3. 如果没有输出（主安装已是最新）：检查是否存在过期的本地 vendored 副本。

运行上面的第 2 步 bash 代码块，以检测主安装类型和目录（`INSTALL_TYPE` 和 `INSTALL_DIR`）。然后运行上面的第 4.5 步检测 bash 代码块，以检查本地 vendored 副本（`LOCAL_GSTACK`）。

**如果 `LOCAL_GSTACK` 为空**（没有本地 vendored 副本）：告诉用户“你已经在使用最新版本（v{version}）。”

**如果 `LOCAL_GSTACK` 非空**，比较版本：
```bash
PRIMARY_VER=$(cat "$INSTALL_DIR/VERSION" 2>/dev/null || echo "unknown")
LOCAL_VER=$(cat "$LOCAL_GSTACK/VERSION" 2>/dev/null || echo "unknown")
echo "PRIMARY=$PRIMARY_VER LOCAL=$LOCAL_VER"
```

**如果版本不同：** 按照上面的第 4.5 步同步 bash 代码块，从主安装更新本地副本。告诉用户：“全局版本 v{PRIMARY_VER} 已是最新。本地 vendored 副本已从 v{LOCAL_VER} 更新到 v{PRIMARY_VER}。准备好后请提交 `.claude/skills/gstack/`。”

**如果版本相同：** 告诉用户“你当前使用的是最新版本（v{PRIMARY_VER}）。全局安装和本地 vendored 副本都已是最新。”