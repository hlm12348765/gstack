# 设计审查清单（精简版）

> **DESIGN_METHODOLOGY 的子集** — 当在此处添加条目时，也要更新 `scripts/gen-skill-docs.ts` 中的 `generateDesignMethodology()`，反之亦然。

## 说明

此清单适用于 **diff 中的源代码**，而不是渲染后的输出。阅读每个变更的前端文件（完整文件，而不只是 diff 片段），并标记反模式。

**触发条件：** 仅当 diff 触及前端文件时才运行此清单。使用 `gstack-diff-scope` 进行检测：

```bash
source <(~/.claude/skills/gstack/bin/gstack-diff-scope <base> 2>/dev/null)
```

如果 `SCOPE_FRONTEND=false`，则静默跳过整个设计审查。

**DESIGN.md 校准：** 如果仓库根目录中存在 `DESIGN.md` 或 `design-system.md`，先读取它。所有发现都要根据项目声明的设计系统进行校准。凡是在 DESIGN.md 中被明确认可的模式，都**不得**标记。如果不存在 DESIGN.md，则使用通用设计原则。

---

## 置信度分层

每个条目都带有一个检测置信度级别标签：

- **[HIGH]** — 可通过 grep/模式匹配可靠检测。属于明确的问题。
- **[MEDIUM]** — 可通过模式聚合或启发式方法检测。可作为问题标记，但要预期存在一些噪声。
- **[LOW]** — 需要理解视觉意图。表述为：“可能存在问题 — 请进行视觉验证或运行 /design-review。”

---

## 分类

**AUTO-FIX**（仅限机械性的 CSS 修复，HIGH 置信度，不需要设计判断）：
- `outline: none` 且没有替代方案 → 添加 `outline: revert` 或 `&:focus-visible { outline: 2px solid currentColor; }`
- 新增 CSS 中出现 `!important` → 删除并修正优先级
- 正文文本的 `font-size` < 16px → 提升到 16px

**ASK**（其余所有情况 — 需要设计判断）：
- 所有 AI slop 问题、排版结构、间距选择、交互状态缺失、违反 DESIGN.md 的情况

**LOW 置信度条目** → 表述为“Possible: [description]. Verify visually or run /design-review.”。绝不要 AUTO-FIX。

---

## 输出格式

```
Design Review: N issues (X auto-fixable, Y need input, Z possible)

**AUTO-FIXED:**
- [file:line] Problem → fix applied

**NEEDS INPUT:**
- [file:line] Problem description
  Recommended fix: suggested fix

**POSSIBLE (verify visually):**
- [file:line] Possible issue — verify with /design-review
```

如果未发现问题：`Design Review: No issues found.`

如果没有变更前端文件：静默跳过，不输出任何内容。

---

## 类别

### 1. AI Slop 检测（6 项）— 最高优先级

这些是 AI 生成式界面的典型迹象，任何受人尊敬的工作室里的设计师都不会发布这种设计。

- **[MEDIUM]** 紫色/紫罗兰/靛蓝渐变背景，或蓝到紫的配色方案。查找取值位于 `#6366f1`–`#8b5cf6` 范围内的 `linear-gradient`，或解析后为紫色/紫罗兰色的 CSS 自定义属性。

- **[LOW]** 三列功能网格：彩色圆形中的图标 + 粗体标题 + 两行描述，且对称重复 3 次。查找恰好包含 3 个子元素的 grid/flex 容器，并且每个子元素都包含一个圆形元素 + 标题 + 段落。

- **[LOW]** 将彩色圆形图标作为区块装饰。查找具有 `border-radius: 50%` 且使用背景色作为图标装饰容器的元素。

- **[HIGH]** 所有内容都居中：所有标题、描述和卡片都使用 `text-align: center`。统计 `text-align: center` 的密度；如果超过 60% 的文本容器使用居中对齐，则标记。

- **[MEDIUM]** 所有元素都使用统一且圆润的 `border-radius`：卡片、按钮、输入框、容器统一使用相同的大圆角（16px+）。聚合 `border-radius` 的值；如果超过 80% 使用同一个且 ≥16px 的值，则标记。

- **[MEDIUM]** 通用的 hero 文案："Welcome to [X]"、"Unlock the power of..."、"Your all-in-one solution for..."、"Revolutionize your..."、"Streamline your workflow"。对 HTML/JSX 内容执行 grep，查找这些模式。

### 2. 排版（4 项）

- **[HIGH]** 正文文本 `font-size` < 16px。对 `body`、`p`、`.text` 或基础样式中的 `font-size` 声明执行 grep。低于 16px 的值（或在基础字号为 16px 时低于 1rem）应标记。

- **[HIGH]** diff 中引入了超过 3 种字体族。统计不同的 `font-family` 声明数量。如果变更文件中出现超过 3 个唯一字体族，则标记。

- **[HIGH]** 标题层级跳级：同一文件/组件中，`h1` 后面直接跟 `h3`，中间没有 `h2`。检查 HTML/JSX 中的标题标签。

- **[HIGH]** 黑名单字体：Papyrus、Comic Sans、Lobster、Impact、Jokerman。对这些名称执行 `font-family` grep。

### 3. 间距与布局（4 项）

- **[MEDIUM]** 当 DESIGN.md 规定了间距尺度时，使用了不在 4px 或 8px 尺度上的任意间距值。根据声明的尺度检查 `margin`、`padding`、`gap` 的值。仅当 DESIGN.md 定义了尺度时才标记。

- **[MEDIUM]** 固定宽度但没有响应式处理：容器上使用 `width: NNNpx`，却没有 `max-width` 或 `@media` 断点。在移动端存在水平滚动风险。

- **[MEDIUM]** 文本容器缺少 `max-width`：正文文本或段落容器未设置 `max-width`，导致行宽超过 75 个字符。检查文本包装容器是否设置了 `max-width`。

- **[HIGH]** 新增 CSS 规则中出现 `!important`。对新增行中的 `!important` 执行 grep。它几乎总是用于逃避优先级问题，应当正确修复。

### 4. 交互状态（3 项）

- **[MEDIUM]** 交互元素（按钮、链接、输入框）缺少 hover/focus 状态。检查新的交互元素样式是否存在 `:hover` 和 `:focus` 或 `:focus-visible` 伪类。

- **[HIGH]** 使用了 `outline: none` 或 `outline: 0`，却没有替代性的焦点指示器。对 `outline:\s*none` 或 `outline:\s*0` 执行 grep。这会移除键盘可访问性。

- **[LOW]** 交互元素的触控目标小于 44px。检查按钮和链接上的 `min-height`/`min-width`/`padding`。需要结合多个属性计算实际尺寸，仅从代码判断时置信度较低。

### 5. 违反 DESIGN.md（3 项，条件启用）

仅在存在 `DESIGN.md` 或 `design-system.md` 时适用：

- **[MEDIUM]** 使用了不在声明调色板中的颜色。将变更 CSS 中的颜色值与 DESIGN.md 中定义的调色板进行比较。

- **[MEDIUM]** 使用了不在声明排版部分中的字体。将 `font-family` 的值与 DESIGN.md 的字体列表进行比较。

- **[MEDIUM]** 使用了不在声明尺度中的间距值。将 `margin`/`padding`/`gap` 的值与 DESIGN.md 的间距尺度进行比较。

---

## 抑制规则

不要标记以下情况：
- 在 DESIGN.md 中明确记录为有意选择的模式
- 第三方/供应商 CSS 文件（`node_modules`、`vendor` 目录）
- CSS reset 或 normalize 样式表
- 测试夹具文件
- 生成的/压缩后的 CSS