# gstack 开发说明

## Commands

```bash
bun install          # 安装依赖
bun test             # 运行免费测试（browse + snapshot + skill validation）
bun run test:evals   # 运行付费 evals：LLM judge + E2E（基于 diff，单次最多约 $4）
bun run test:evals:all  # 无论 diff 如何都运行所有付费 evals
bun run test:e2e     # 仅运行 E2E 测试（基于 diff，单次最多约 $3.85）
bun run test:e2e:all # 无论 diff 如何都运行所有 E2E 测试
bun run eval:select  # 显示基于当前 diff 会运行哪些测试
bun run dev <cmd>    # 以开发模式运行 CLI，例如 bun run dev goto https://example.com
bun run build        # 生成文档并编译二进制
bun run gen:skill-docs  # 从模板重新生成 SKILL.md 文件
bun run skill:check  # 所有 skills 的健康检查面板
bun run dev:skill    # watch 模式：变更时自动重新生成并校验
bun run eval:list    # 列出 ~/.gstack-dev/evals/ 下的所有 eval 运行记录
bun run eval:compare # 比较两次 eval 运行（默认自动选择最近两次）
bun run eval:summary # 汇总所有 eval 运行的统计信息
```

`test:evals` 需要 `ANTHROPIC_API_KEY`。Codex E2E 测试（`test/codex-e2e.test.ts`）
使用 Codex 自己在 `~/.codex/` 下的认证配置，不需要 `OPENAI_API_KEY` 环境变量。
E2E 测试会实时流式输出进度（通过 `--output-format stream-json --verbose`，逐工具输出）。
结果会持久化到 `~/.gstack-dev/evals/`，并自动与上一次运行结果进行比较。

**基于 diff 的测试选择：** `test:evals` 和 `test:e2e` 会根据相对于 base branch 的 `git diff`
自动选择要运行的测试。每个测试在 `test/helpers/touchfiles.ts` 中声明它依赖的文件。
如果修改了全局 touchfiles（session-runner、eval-store、llm-judge、gen-skill-docs），
则会触发所有测试。使用 `EVALS_ALL=1` 或 `:all` 脚本变体可强制运行全部测试。
运行 `eval:select` 可预览将要执行哪些测试。

## Testing

```bash
bun test             # 每次提交前运行——免费，<2s
bun run test:evals   # 发版前运行——付费，基于 diff（单次最多约 $4）
```

`bun test` 会运行 skill validation、gen-skill-docs 质量检查和 browse 集成测试。
`bun run test:evals` 会运行 LLM-judge 质量评估以及通过 `claude -p` 驱动的 E2E 测试。
创建 PR 前，两者都必须通过。

## 项目结构

```
gstack/
├── browse/          # 无头浏览器 CLI（Playwright）
│   ├── src/         # CLI + server + commands
│   │   ├── commands.ts  # 命令注册表（single source of truth）
│   │   └── snapshot.ts  # SNAPSHOT_FLAGS 元数据数组
│   ├── test/        # 集成测试 + fixtures
│   └── dist/        # 编译后的二进制
├── scripts/         # 构建与开发体验工具
│   ├── gen-skill-docs.ts  # 模板 → SKILL.md 生成器
│   ├── skill-check.ts     # 健康检查面板
│   └── dev-skill.ts       # Watch 模式
├── test/            # Skill validation + eval tests
│   ├── helpers/     # skill-parser.ts、session-runner.ts、llm-judge.ts、eval-store.ts
│   ├── fixtures/    # Ground truth JSON、planted-bug fixtures、eval baselines
│   ├── skill-validation.test.ts  # Tier 1：静态校验（免费，<1s）
│   ├── gen-skill-docs.test.ts    # Tier 1：生成器质量（免费，<1s）
│   ├── skill-llm-eval.test.ts    # Tier 3：LLM-as-judge（约 $0.15/次）
│   └── skill-e2e-*.test.ts       # Tier 2：通过 claude -p 的 E2E（约 $3.85/次，按类别拆分）
├── qa-only/         # /qa-only skill（仅报告 QA，不修复）
├── plan-design-review/  # /plan-design-review skill（仅报告式设计审计）
├── design-review/    # /design-review skill（设计审计 + 修复闭环）
├── ship/            # 发布工作流 skill
├── review/          # PR 评审 skill
├── plan-ceo-review/ # /plan-ceo-review skill
├── plan-eng-review/ # /plan-eng-review skill
├── autoplan/        # /autoplan skill（自动评审流水线：CEO → design → eng）
├── benchmark/       # /benchmark skill（性能回归检测）
├── canary/          # /canary skill（发布后监控闭环）
├── codex/           # /codex skill（通过 OpenAI Codex CLI 获取多 AI 第二意见）
├── land-and-deploy/ # /land-and-deploy skill（合并 → 部署 → canary 验证）
├── office-hours/    # /office-hours skill（YC Office Hours——创业诊断 + 构建者脑暴）
├── investigate/     # /investigate skill（系统化根因调试）
├── retro/           # 复盘 skill（包含 /retro 全局跨项目模式）
├── bin/             # 独立脚本（gstack-global-discover 等跨工具会话发现工具）
├── document-release/ # /document-release skill（发布后文档更新）
├── cso/             # /cso skill（OWASP Top 10 + STRIDE 安全审计）
├── design-consultation/ # /design-consultation skill（从零构建设计系统）
├── setup-deploy/    # /setup-deploy skill（一次性部署配置）
├── bin/             # CLI 工具（gstack-repo-mode、gstack-slug、gstack-config 等）
├── setup            # 一次性安装：构建二进制 + 建立技能软链接
├── SKILL.md         # 由 SKILL.md.tmpl 生成（不要直接编辑）
├── SKILL.md.tmpl    # 模板：编辑它，然后运行 gen:skill-docs
├── ETHOS.md         # 构建者理念（Boil the Lake、Search Before Building）
└── package.json     # browse 的构建脚本
```

## SKILL.md 工作流

`SKILL.md` 文件是由 `.tmpl` 模板**生成**的。更新文档时：

1. 编辑 `.tmpl` 文件（例如 `SKILL.md.tmpl` 或 `browse/SKILL.md.tmpl`）
2. 运行 `bun run gen:skill-docs`（或者运行会自动执行它的 `bun run build`）
3. 提交 `.tmpl` 文件和生成出的 `.md` 文件

要添加一个新的 browse 命令：把它加入 `browse/src/commands.ts`，然后重新构建。
要添加一个 snapshot flag：把它加到 `browse/src/snapshot.ts` 中的 `SNAPSHOT_FLAGS`，然后重新构建。

**SKILL.md 文件上的合并冲突：** 绝不要通过“接受任意一边”的方式解决生成出的 `SKILL.md` 文件冲突。
正确做法是：
1. 解决 `.tmpl` 模板和 `scripts/gen-skill-docs.ts`（真实来源）的冲突
2. 运行 `bun run gen:skill-docs` 重新生成所有 `SKILL.md`
3. stage 重新生成的文件

直接接受某一边的生成结果，会悄悄丢掉另一边的模板改动。

## 平台无关设计

Skills 绝不能硬编码框架相关命令、文件模式或目录结构。
正确方式是：

1. **读取 `CLAUDE.md`** 获取项目专属配置（测试命令、eval 命令等）
2. **如果缺失，就用 AskUserQuestion**——让用户告诉你，或者让 gstack 去仓库里搜索
3. **把答案持久化写入 `CLAUDE.md`**，这样以后就不必再问

这适用于测试命令、eval 命令、部署命令以及任何其他项目特定行为。
项目自己拥有配置；gstack 只负责读取。

## 编写 SKILL 模板

`SKILL.md.tmpl` 文件是 **Claude 读取的 prompt 模板**，不是 bash 脚本。
每个 bash 代码块都在独立 shell 中执行，变量不会跨代码块持久化。

规则：
- **用自然语言表达逻辑和状态。** 不要用 shell 变量在代码块之间传递状态。应告诉 Claude 需要记住什么，并在 prose 中引用它（例如“Step 0 中检测到的 base branch”）。
- **不要硬编码分支名。** 动态检测 `main` / `master` / 等，通过 `gh pr view` 或 `gh repo view`。对于面向 PR 的 skills 使用 `{{BASE_BRANCH_DETECT}}`。在 prose 中用 “the base branch”，在代码块占位中用 `<base>`。
- **让每个 bash 块自包含。** 每个代码块都应能独立工作。如果某个代码块需要前一步的上下文，就在上方 prose 中重新说明。
- **用英文说明条件分支，而不是把复杂条件塞进 bash。** 与其写层层嵌套的 `if/elif/else`，不如写成编号决策步骤：“1. If X, do Y. 2. Otherwise, do Z.”

## 浏览器交互

当你需要与浏览器交互时（QA、dogfooding、cookie setup），使用 `/browse` skill，
或者直接通过 `$B <command>` 调用 browse 二进制。绝不要使用
`mcp__claude-in-chrome__*` 工具——它们缓慢、不稳定，也不是这个项目所使用的方案。

## Vendored symlink awareness

在开发 gstack 时，`.claude/skills/gstack` 可能是一个指回当前工作目录的符号链接（已被 gitignore）。
这意味着 skill 改动会**立即生效**——非常适合快速迭代；但在大规模重构期间也很危险，
因为尚未写完的技能可能会影响其他同时在使用 gstack 的 Claude Code 会话。

**每个会话检查一次：** 运行 `ls -la .claude/skills/gstack`，确认它是 symlink 还是实际拷贝。
如果它是一个指向当前工作目录的 symlink，要意识到：
- 模板改动 + `bun run gen:skill-docs` 会立即影响所有 gstack 调用
- 对 `SKILL.md.tmpl` 的破坏性改动，会破坏其他并发 gstack 会话
- 在大型重构期间，应移除该 symlink（`rm .claude/skills/gstack`），让全局安装路径 `~/.claude/skills/gstack/` 生效

**对于 plan reviews：** 当评审会修改 skill templates 或 gen-skill-docs 流水线的计划时，
要考虑这些改动是否应先在隔离环境中测试，再对外生效（尤其当用户正在其他窗口中活跃使用 gstack 时）。

## Commit 风格

**永远做可 bisect 的 commits。** 每个 commit 都应只包含一个逻辑变更。
如果你做了多个改动（例如重命名 + 重写 + 新测试），在 push 之前必须拆成多个 commits。
每个 commit 都应能够被独立理解、独立回退。

良好的 bisection 示例：
- 重命名 / 移动，与行为变更分开
- 测试基础设施（touchfiles、helpers）与测试实现分开
- 模板变更与生成文件重新生成分开
- 机械性重构与新功能分开

当用户说 “bisect commit” 或 “bisect and push” 时，把 staged / unstaged 改动拆成逻辑清晰的多个 commits 再推送。

## CHANGELOG + VERSION 风格

**VERSION 和 CHANGELOG 是分支级别的。** 每个会发布的 feature branch，都应该有自己的版本号提升和 CHANGELOG 条目。这个条目描述的是“本分支新增了什么”，而不是 main 上原本已有的内容。

**何时写 CHANGELOG 条目：**
- 在 `/ship` 时（Step 5），而不是在开发过程中或分支中途。
- 该条目要覆盖“本分支相对于 base branch 的全部改动”。
- 不要把新工作折叠进 main 上已经发布的旧条目里。如果 main 已有 v0.10.0.0，而你的分支又新增功能，就应升到 v0.10.1.0 并写新条目，而不是编辑 v0.10.0.0 那条。

**写之前先问自己：**
1. 我当前在哪个分支？这个分支改了什么？
2. base branch 的版本是否已经发布？（如果是，则需要提升版本并创建新条目。）
3. 当前分支上是否已有条目覆盖了更早的工作？（如果有，就把它替换为一条统一的最终版本说明。）

`CHANGELOG.md` 是**写给用户看的**，不是写给贡献者看的。它必须像产品发布说明：

- 开头先讲用户现在**能做什么以前做不到的事**。把功能卖出去。
- 用自然语言，不要堆实现细节。写“你现在可以……”，不要写“重构了……”。
- **永远不要提到 `TODOS.md`、内部追踪、eval 基础设施，或面向贡献者的细节。** 这些对用户不可见，也没有意义。
- 贡献者 / 内部变更放到单独的 “For contributors” 小节。
- 每个条目都应让读者产生“哦，这不错，我想试试”的感觉。
- 不要讲术语：写“现在每个问题都会告诉你你在哪个项目、哪个分支”，而不是“AskUserQuestion format standardized across skill templates via preamble resolver”。

## AI effort compression

在估算或讨论工作量时，始终同时给出“人类团队时间”和“CC+gstack 时间”：

| 任务类型 | 人类团队 | CC+gstack | 压缩比 |
|-----------|-----------|-----------|--------|
| Boilerplate / scaffolding | 2 天 | 15 分钟 | ~100x |
| Test writing | 1 天 | 15 分钟 | ~50x |
| Feature implementation | 1 周 | 30 分钟 | ~30x |
| Bug fix + regression test | 4 小时 | 15 分钟 | ~20x |
| Architecture / design | 2 天 | 4 小时 | ~5x |
| Research / exploration | 1 天 | 3 小时 | ~3x |

完整性很便宜。对于“lake”（可实现）而不是 “ocean”（跨多个季度的大迁移）的任务，
不要推荐走捷径。完整哲学见 skill preamble 中的 Completeness Principle。

## Search before building

在设计任何涉及并发、陌生模式、基础设施，或者运行时 / 框架可能已有内建能力的方案之前：

1. 搜索 `"{runtime} {thing} built-in"`
2. 搜索 `"{thing} best practice {current year}"`
3. 查官方运行时 / 框架文档

三层知识：tried-and-true（Layer 1）、new-and-popular（Layer 2）、
first-principles（Layer 3）。最高优先级永远是 Layer 3。
完整的构建者哲学请见 `ETHOS.md`。

## Local plans

贡献者可以把长期愿景文档和设计文档存放在 `~/.gstack-dev/plans/`。
这些是本地文件（不纳入版本控制）。在评审 `TODOS.md` 时，也要检查 `plans/`，
看看是否有已经成熟、可以提升为 TODO 或直接实施的候选方案。

## E2E eval failure blame protocol

当 `/ship` 或其他工作流中的 E2E eval 失败时，**绝不能在没有证明的情况下声称
“这和我们的改动无关”。** 这些系统存在不可见耦合：一段 preamble 文案变化会影响代理行为，
一个新的 helper 会改变时序，重新生成的 `SKILL.md` 会改变 prompt 上下文。

**在把失败归因为“原本就存在”之前，必须做到：**
1. 在 main（或 base branch）上运行同一 eval，并展示它在那里也失败
2. 如果它在 main 上通过、在当前分支失败——那就是你的改动造成的。继续追查归因。
3. 如果你无法在 main 上运行，就写“未验证——可能相关，也可能无关”，并在 PR 描述中把它标记为风险

没有证据就说“本来就这样”，是一种懒惰的说法。要么证明，要么别说。

## 长任务：不要放弃

在运行 evals、E2E tests 或任何长时间后台任务时，**要持续轮询直到完成。**
使用 `sleep 180 && echo "ready"` + `TaskOutput`，每 3 分钟循环一次。
不要切成阻塞模式后因为一次轮询超时就放弃。不要说“完成后我会收到通知”，
然后就不再检查——必须一直轮询，直到任务结束，或用户明确让你停下。

完整的 E2E 套件可能要 30-45 分钟，也就是 10-15 次轮询。全部都要做。
每次检查都要汇报进度（哪些测试已通过、哪些正在运行、目前有哪些失败）。
用户想看到的是“这次运行被跟到了结束”，而不是“我之后会再检查”的承诺。

## 部署到当前正在使用的 skill

当前启用的 skill 位于 `~/.claude/skills/gstack/`。完成修改后：

1. 推送你的分支
2. 在 skill 目录中获取并重置：`cd ~/.claude/skills/gstack && git fetch origin && git reset --hard origin/main`
3. 重新构建：`cd ~/.claude/skills/gstack && bun run build`

或者直接复制二进制：`cp browse/dist/browse ~/.claude/skills/gstack/browse/dist/browse`
