# gstack — AI 工程工作流

gstack 是一组 `SKILL.md` 文件，为 AI 智能体在软件开发中提供结构化角色。每个技能都是一种专项角色：CEO 审查者、工程经理、设计师、QA 负责人、发布工程师、调试专家等等。

## 可用技能

技能位于 `.agents/skills/` 中。通过名称调用它们（例如 `/office-hours`）。

| 技能 | 作用 |
|-------|-------------|
| `/office-hours` | 从这里开始。在你编写代码之前，先重新梳理你的产品想法。 |
| `/plan-ceo-review` | CEO 级评审：从需求中找出那个 10 星级产品。 |
| `/plan-eng-review` | 敲定架构、数据流、边界情况和测试。 |
| `/plan-design-review` | 将每个设计维度按 0-10 评分，并说明 10 分是什么样子。 |
| `/design-consultation` | 从零开始构建设计系统。 |
| `/review` | 合入前的 PR 评审。找出那些能通过 CI 但会在生产环境出问题的缺陷。 |
| `/debug` | 系统化的根因调试。不先调查，就不修复。 |
| `/design-review` | 设计审查 + 修复循环，并使用原子提交。 |
| `/qa` | 打开真实浏览器，查找问题，修复问题，再次验证。 |
| `/qa-only` | 与 /qa 相同，但仅报告，不改代码。 |
| `/ship` | 运行测试、审查、推送、创建 PR。一个命令完成。 |
| `/document-release` | 更新所有文档，使其与你刚发布的内容保持一致。 |
| `/retro` | 每周复盘，包含按人员拆分的分析和持续发布记录。 |
| `/browse` | 无头浏览器，真实 Chromium，真实点击，约 ~100ms/命令。 |
| `/setup-browser-cookies` | 从你的真实浏览器导入 cookies，以进行需要认证的测试。 |
| `/careful` | 在执行破坏性命令前发出警告（`rm -rf`、`DROP TABLE`、force-push）。 |
| `/freeze` | 将编辑限制在单个目录。是硬性阻止，不只是警告。 |
| `/guard` | 同时启用 careful + freeze。 |
| `/unfreeze` | 移除目录编辑限制。 |
| `/gstack-upgrade` | 将 gstack 更新到最新版本。 |

## 构建命令

```bash
bun install              # 安装依赖
bun test                 # 运行测试（免费，<5 秒）
bun run build            # 生成文档 + 编译二进制文件
bun run gen:skill-docs   # 从模板重新生成 SKILL.md 文件
bun run skill:check      # 所有技能的健康状态面板
```

## 关键约定

- `SKILL.md` 文件由 `.tmpl` 模板**生成**。请编辑模板，不要编辑输出文件。
- 运行 `bun run gen:skill-docs --host codex` 以重新生成面向 Codex 的输出。
- browse 二进制文件提供无头浏览器访问能力。在技能中使用 `$B <command>`。
- 安全技能（careful、freeze、guard）使用内联提示性说明文字，在执行破坏性操作前务必始终确认。