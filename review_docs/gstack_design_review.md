# gstack 设计理念深度分析

## 结论先行

gstack 不是一个“收集了很多 prompt 的仓库”，而是一套把创始人/产品/工程/设计/QA/发布/复盘工作流编码成可执行技能的“AI 软件工厂操作系统”。

它真正厉害的地方不在于“多生成一点代码”，而在于把过去依赖资深团队默契才能稳定完成的事情，拆成了可以被 agent 重复执行的角色、阶段、产物、检查表和验证回路。于是，AI 的作用不再只是补代码，而是补组织能力、补方法论、补流程一致性、补反馈速度。

从 YC 视角看，gstack 传达的核心不是“AI 会替代团队”，而是：

1. 小团队必须先拥有成熟团队的决策结构，再去追求成熟团队的产出。
2. AI 时代最稀缺的不是写码速度，而是问题定义、优先级判断、完整性、品味与闭环。
3. 如果流程可以被写成技能，组织规模就不必线性增长。

---

## 1. 它本质上是什么

我认为 gstack 有三层结构：

### 1.1 哲学层：把工作信条写死

`ETHOS.md` 不是装饰文档，而是整个系统的价值观内核。它明确提出四个判断：

- AI 让单人产能出现数量级压缩，“10,000+ usable LOC/day” 不再是神话，而是新的边界条件。见 `ETHOS.md:9-30`。
- “Boil the Lake” 不是鼓励无节制加 scope，而是主张在 AI 把边际实现成本压低后，不要再用旧时代的人力稀缺逻辑接受 80%-90% 完成度。见 `ETHOS.md:34-55`。
- “Search Before Building” 强迫系统区分 tried-and-true、new-and-popular、first-principles 三层知识来源，防止 agent 同时犯“盲从惯例”和“自造轮子”两种错。见 `ETHOS.md:59-119`。
- “Build for Yourself” 则把创始人直觉视为产品发现的重要来源。见 `ETHOS.md:123-129`。

这层设计很关键。因为没有哲学层，技能只会变成“很多模板化对话”；有了哲学层，技能才会在不同阶段保持一致的判断方向。

### 1.2 工作流层：把组织角色编码成技能

README 对 gstack 的自我定义非常直白：它要把 Claude Code 变成“virtual engineering team”，其中有 CEO、eng manager、designer、reviewer、QA lead、security officer、release engineer 等角色。见 `README.md:21-23`。

这里最重要的设计不是“角色名字很酷”，而是它把复杂任务拆成了认知边界清晰的子问题：

- `/office-hours` 先判断“你到底在解决什么问题”，而且硬性禁止直接进入实现。见 `office-hours/SKILL.md.tmpl:29-33`。
- `/plan-ceo-review` 负责挑战 framing、scope、理想终局和扩张机会。见 `plan-ceo-review/SKILL.md.tmpl:27-48`。
- `/plan-eng-review` 锁技术路径。
- `/plan-design-review` 和 `/design-review` 强制把用户可见体验、状态覆盖和 AI slop 风险提前显式化。
- `/review`、`/qa`、`/ship`、`/retro` 让产出不是“写完了”，而是“被检查、被验证、被发布、被回顾了”。

这实际上是在把“组织分工”做成“上下文管理”工具。单个 agent 最容易失败的地方，往往不是代码能力，而是任务太宽、目标太混、评价标准不稳定。角色化技能降低了这种混乱。

### 1.3 运行时层：浏览器、生成器、验证器、跨宿主适配

Architecture 文档讲得很清楚：gstack 的“硬部分”是持久化浏览器，其余很多东西其实是 Markdown。见 `ARCHITECTURE.md:5-10`。

这句话非常准确。运行时层至少包括：

- 持久化浏览器守护进程，解决 agent 与真实页面交互时的延迟和状态丢失问题。见 `ARCHITECTURE.md:7-36`。
- 命令注册表 `browse/src/commands.ts`，把运行时 dispatch、文档生成、技能校验、健康检查统一到一个单一真相源。见 `browse/src/commands.ts:1-10`, `browse/src/commands.ts:13-36`, `browse/src/commands.ts:103-110`。
- 文档生成器 `scripts/gen-skill-docs.ts`，把技能模板和源代码元数据连起来，避免文档漂移。见 `scripts/gen-skill-docs.ts:1-10`, `ARCHITECTURE.md:179-220`。
- 静态校验、E2E、LLM-as-judge 三层测试体系。见 `ARCHITECTURE.md:230-257`, `package.json:15-33`。
- Claude/Codex/Kiro 等宿主差异的生成适配。见 `README.md:57-89`, `scripts/gen-skill-docs.ts:22-53`。

换句话说，gstack 把“prompt 工程”推进到了“prompt 供应链工程”。

---

## 2. 它的几个关键设计理念

## 2.1 不是给 agent 建一个大脑，而是给它一套工位

很多 AI coding 项目默认假设：只要模型够强、上下文够大、工具够多，就能把所有事情一起做好。

gstack 的假设相反：

- 问题定义、战略判断、工程实现、设计审美、QA、发布，这些不是同一种认知任务。
- 同一个 agent 可以扮演多个角色，但必须一次只扮演一个。
- 角色切换要靠明确 workflow，而不是靠一句“现在你是 CTO/PM/设计师了”。

这就是为什么 `/office-hours` 会在一开始硬性声明“只产出 design doc，不写代码”，而 `/plan-ceo-review` 会强调“不要 rubber-stamp，要把计划做得 extraordinary”。见 `office-hours/SKILL.md.tmpl:31-33`, `plan-ceo-review/SKILL.md.tmpl:29-38`。

本质上，这是在用角色隔离来替代传统团队的会议与分工成本。

## 2.2 把“隐性资深经验”变成“显性前导约束”

gstack 最聪明的结构之一是统一 preamble。

Architecture 文档写明：每个技能都先跑同一个 `{{PREAMBLE}}`，里面处理升级检测、session tracking、contributor mode、统一的 AskUserQuestion 格式，以及 Search Before Building。见 `ARCHITECTURE.md:211-220`。

生成器里也能直接看到这些逻辑被写入所有技能开头。见 `scripts/gen-skill-docs.ts:178-216`。

这意味着：

- agent 不是每次重新发明“该怎么问用户问题”；
- agent 不是每次重新决定“要不要先查一下”；
- agent 不是每次重新判断“是否需要给用户补上下文”；
- agent 的工作方式被仓库层统一，而不是由每个技能作者各自发挥。

我在当前仓库里统计到：

- 28 个 `SKILL.md`
- 28 个 `SKILL.md.tmpl`
- 23 个模板显式引用 `{{PREAMBLE}}`
- 146 处 `AskUserQuestion`
- 52 处 `STOP`

这说明 gstack 的重点不是“尽量少约束 agent”，而是“把关键交互和停顿点制度化”。

## 2.3 把 prompt 当源代码维护，而不是当聊天记录维护

`scripts/gen-skill-docs.ts` 的设计理念非常值得注意。

它不是手写一堆 SKILL.md，而是：

1. 维护 `.tmpl` 模板。
2. 从 `commands.ts`、`snapshot.ts` 等源代码里拉结构化元数据。
3. 生成最终 `SKILL.md`。
4. 用 `--dry-run`、`skill:check`、测试来校验文档是否和代码一致。

Architecture 文档把这个问题说得很准确：如果技能文档写了不存在的 flag，或者漏了新命令，agent 在运行时就会直接撞墙，所以手写文档一定会 drift。见 `ARCHITECTURE.md:181-209`。

更进一步，`browse/src/commands.ts` 自己还做了 load-time validation，强制 `COMMAND_DESCRIPTIONS` 与命令集合完全对应。见 `browse/src/commands.ts:103-110`。

这背后的理念是：

- 提示词不是一次性写作，而是需要像 API 文档一样可测试、可生成、可约束。
- “Markdown 工作流”仍然应该有编译链、静态分析和 freshness check。

这比大多数所谓“AI workflow repo”高了一个工程层级。

## 2.4 给 agent 一双眼睛，而且是有记忆的眼睛

gstack 最硬核的运行时设计是 browse daemon。

Architecture 文档的核心论点是：agent 操作浏览器不能每次冷启动，否则每个操作 3-5 秒，状态还会丢。于是它采用持久化 Chromium + localhost HTTP daemon 的模式，首个调用约 3 秒，之后每次调用约 100-200ms。见 `ARCHITECTURE.md:7-36`, `ARCHITECTURE.md:52-62`。

在代码里，`BrowserManager` 明确把“不要试图自愈半死浏览器”写成原则，Chromium 崩了就退出，让 CLI 在下一次命令时重启。见 `browse/src/browser-manager.ts:4-16`, `browse/src/browser-manager.ts:67-72`。这是一种很 YC 的选择：简单、可预测、易恢复，优先于“看起来更高级”的复杂容错。

这对生产力的提升非常直接：

- QA、调试、截图、表单测试不再是“描述页面应该怎样”，而是“真实点击和验证”。
- 登录态、cookie、tab、localStorage 能跨命令保留。
- 代理终于不只是写代码，而是能自己看到是否真的工作。

如果没有这一层，`/qa`、`/design-review`、`/canary`、`/benchmark` 这些技能都很难成立。

## 2.5 让验证成为主流程，而不是收尾动作

`/qa` 的模板非常能体现这点。

它不是“测一下页面”，而是完整定义了：

- 先确保工作树干净，否则停下来处理。见 `qa/SKILL.md.tmpl:52-68`。
- 基于计划产物、git diff 或 URL 进入上下文。见 `qa/SKILL.md.tmpl:86-96`。
- 测完先分级，再决定修哪些。见 `qa/SKILL.md.tmpl:127-135`。
- 每个问题独立修复、独立提交、独立回归验证。见 `qa/SKILL.md.tmpl:139-188`。
- 能验证的 bug 要求生成 regression test，而且 test 风格要贴近项目既有模式。见 `qa/SKILL.md.tmpl:189-220`。

这意味着 gstack 把 QA 从“最后看一眼”升级为“生成结构化证据并反哺代码库”。

同样地，`/review` 强调 scope drift、Greptile 评论、文档陈旧性、枚举完整性、自动修复与证据引用；`/investigate` 则把“no fixes without root cause”写成 Iron Law。见 `review/SKILL.md.tmpl`, `investigate/SKILL.md.tmpl:28-44`, `investigate/SKILL.md.tmpl:117-156`。

这套系统想解决的不是“写得更快”，而是“快的时候别变瞎”。

## 2.6 把第二意见制度化，而不是临时起意

从 changelog 可以看出，gstack 持续把“outside voices”做成系统特性，而不是偶尔手动切模型：

- `/office-hours` 可以引入 Codex 的冷启动 second opinion。见 `CHANGELOG.md:3-9`。
- 设计评审能并行调 Codex 与 Claude subagent，比较一致与分歧。见 `CHANGELOG.md:11-19`。
- `/autoplan` 直接规定 taste decisions、Codex disagreements 和 final approval gate。见 `CHANGELOG.md:109-113`, `autoplan/SKILL.md.tmpl:31-99`。
- Codex 本身还有独立 E2E 测试。见 `test/codex-e2e.test.ts:1-14`, `test/codex-e2e.test.ts:120-196`。

这点非常像高质量创业团队的工作方式：不是假设一个声音永远对，而是设计机制让不同视角的 disagreement 产生价值。

---

## 3. 为什么它会极大提升生产力

## 3.1 它消灭的是“组织摩擦”，不只是“编码摩擦”

传统软件团队慢，不只是因为写代码慢。还因为：

- 需求不清；
- 范围漂移；
- 设计晚介入；
- 测试与发布是补丁流程；
- 不同人对“完成”的定义不同；
- 很多关键判断只存在于资深同事脑子里。

gstack 通过技能把这些摩擦前置并模块化。于是一个人面对 AI 时，不用反复在同一个会话里扮演 PM、CEO、架构师、designer、QA、release manager，而是按阶段调用角色。

这会带来三个直接收益：

1. 减少空白提示词带来的思考损耗。
2. 减少在错误阶段做错误决策的概率。
3. 减少“做完了但没真完成”的返工。

## 3.2 它把 senior judgment 变成了可复用基础设施

`/office-hours` 里“Specificity is the only currency”“Interest is not demand”“Watch, don't demo”这些话，本质上是 YC partner 在高压场景里会反复逼问创始人的问题。见 `office-hours/SKILL.md.tmpl:84-99`, `office-hours/SKILL.md.tmpl:170-219`。

`/plan-ceo-review` 里的“Zero silent failures”“Every error has a name”“Observability is scope”则像一个极强的 founder/CTO 在审设计文档。见 `plan-ceo-review/SKILL.md.tmpl:40-48`。

把这些判断写进技能后，单人开发者不再需要“在情绪稳定、睡眠充足、脑子很清楚的状态下”才做得出高质量判断。系统会替他重复这些问题。

这是 gstack 最大的杠杆之一：它把“资深度”部分外化了。

## 3.3 它默认闭环，而不是默认开放环

很多 agent 工具的默认结束条件是：“任务看起来做完了。”

gstack 的默认结束条件更像：

- 写了没？
- 测了没？
- 证据呢？
- 有回归测试吗？
- 文档该不该更新？
- 能发布吗？
- 发布后怎么监控？
- 这次学到了什么？

从 `/qa`、`/review`、`/ship`、`/land-and-deploy`、`/retro` 这一串可以看出，作者想要的不是“更会写”，而是“更像一个能上线并持续改进的系统”。README 的 quick start 也明确把 `/office-hours`、`/plan-ceo-review`、`/review`、`/qa` 当成核心入口，而不是只强调 build feature。见 `README.md:32-39`。

这对产能的提升非常关键，因为真实世界里的时间常常死在“最后 10%”：补测试、过 review、追 bug、手工验收、发版、回滚、对齐文档。gstack 试图把这部分一起吞掉。

## 3.4 它让“完整性”成为默认答案

ETHOS 中最有争议、也最有力量的一点，是把“完整性”从奢侈品变成默认策略。见 `ETHOS.md:34-55`。

这背后的 productivity 逻辑不是“多做一点”，而是：

- 在 AI 时代，留下 10% 未完成事项不再是节省时间，而常常是在制造未来债务。
- 当测试、错误路径、文档、状态覆盖都变便宜时，最贵的反而是没有做它们。

`/autoplan` 把这个理念进一步程序化成 6 条决策原则，其中第一条就是 completeness，第二条是 boil lakes。见 `autoplan/SKILL.md.tmpl:43-57`。

这会极大改变 agent 的决策分布。它不再倾向于“先做个 MVP 但其实埋了一堆隐患”，而是更倾向于“在 boilable scope 内做完整版本”。

## 3.5 它极大压缩了“验证延迟”

浏览器 daemon、diff-aware QA、计划产物回流、E2E runner、Codex/Gemini/Claude 多宿主测试，共同解决的是一个问题：反馈太慢。

一旦反馈足够快：

- 设计错误不会到上线后才被发现。
- QA 不是人工抽检，而是可重复执行。
- skill 修改之后，不需要靠直觉相信它没坏。

`test/helpers/session-runner.ts` 很能说明这一点。它不是 mock agent，而是真的起 `claude -p` 子进程、流式读 NDJSON、记录心跳、计算 inter-turn latency。见 `test/helpers/session-runner.ts:1-26`, `test/helpers/session-runner.ts:115-184`, `test/helpers/session-runner.ts:186-240`。

这意味着 gstack 连“技能本身”都被当成软件产品来测，而不是当成不可测的文案。

## 3.6 它让并行成为有序并行，而不是混乱并行

README 提到它和 Conductor 结合时，可以让多个 Claude Code session 同时跑不同阶段，但前提是 sprint 结构清晰，否则十个 agent 只是十个混乱源。见 README 中的 “Parallel sprints” 段落。

这正是 gstack 的价值：

- 有阶段。
- 有角色。
- 有产物。
- 有交接。
- 有 stop 条件。

没有这些，所谓“多 agent 并行”通常只是把噪音并行化。gstack 的设计是在给并行提供交通规则。

---

## 4. 它体现出的 YC 工作方式

如果只看表面，会觉得 gstack 是 YC CEO 写的 skill 仓库；但如果往深一点看，它其实是在把 YC 的工作方式产品化。

## 4.1 先重构问题，再重构方案

`/office-hours` 最像 YC 的地方，不是“问六个问题”，而是它拒绝把用户最初的 feature request 当真。

它会逼着人回答：

- 真实用户是谁？
- 现在怎么解决？
- 付费或痛苦证据是什么？
- 最窄可付费 wedge 是什么？
- 你观察到什么反直觉行为？

这和 YC 对早期创业公司的要求高度一致：不要沉迷于 solution narrative，要先穿透到 demand reality。见 `office-hours/SKILL.md.tmpl:80-99`, `office-hours/SKILL.md.tmpl:177-219`。

对创业团队的启示是：

- feature brainstorming 不应该从 backlog 开始，而应该从痛点证据开始；
- 最危险的不是 scope 太小，而是你在解决一个被讲述得很漂亮但实际不疼的问题。

## 4.2 大胆与克制必须被同时制度化

`/plan-ceo-review` 的四种模式很有意思：扩张、选择性扩张、保持范围、缩减范围。见 `plan-ceo-review/SKILL.md.tmpl:31-37`。

这说明作者并不迷信“永远想更大”或“永远先做最小”。真正的重点是：

- ambition 要被显式讨论；
- scope 变化必须由用户明确同意；
- 系统不能偷偷扩 scope，也不能偷偷缩 scope。

这对变革团队尤其重要。很多组织改革之所以失败，不是因为没有想象力，而是因为 ambition 与现实约束之间没有明确协商机制。gstack 用 AskUserQuestion 和 mode selection 把这件事程序化了。

## 4.3 可观察性、错误路径、交付物都是 scope

YC 式强运营团队的一个特点，是他们会把“能否知道发生了什么”看成产品本身的一部分，而不是附属物。

`/plan-ceo-review` 里直接写：

- Zero silent failures
- Every error has a name
- Observability is scope
- Everything deferred must be written down

见 `plan-ceo-review/SKILL.md.tmpl:40-48`。

这对创业/变革团队的启示很大：

- 不要把 runbook、告警、失败模式、TODO 记账当成“成熟后再做”的企业流程。
- 对小团队来说，它们恰恰是能让你继续快下去的前提。

## 4.4 用文档与产物替代会议记忆

gstack 到处在产出工件：

- design doc
- plan file
- review dashboard
- QA report
- screenshots
- ship summary
- retro
- TODOs
- restore point / handoff note

`/autoplan` 甚至要求先写 restore point，再加载技能文件，再写 audit trail。见 `autoplan/SKILL.md.tmpl:103-165`。

这非常符合高效创业团队的一个隐性原则：会议可以少，但上下文必须可继承。人少的时候，工件就是组织记忆。

## 4.5 第二意见不是“否定”，而是“去盲点”

YC partner 模式本来就很像高压 second opinion：你拿一个想法来，不会被礼貌附和，而会被逼着把最薄弱的地方讲清楚。

gstack 把这个做成了多模型机制。它不是为了“更智能”而多模型，而是为了：

- 降低单模型盲区；
- 显化分歧；
- 把 disagreement 变成决策点。

这对创业和变革团队的启示是：高速度团队并不应该压制 dissent，反而应该把 dissent 结构化。

## 4.6 速度提升后，质量门槛应该提高，不应该降低

这是 gstack 传达的最重要 YC 观点之一。

传统团队常说：

- 先别写测试；
- 先别补设计状态；
- 先别管文档；
- 先别做浏览器验证；
- 先别管部署监控；

因为这些都“太贵”。

gstack 的回答是：既然 AI 已经把实现成本压下来了，那就应该把质量标准抬上去，而不是维持旧时代的低标准。这就是 “Boil the Lake” 的组织含义。

对创业团队来说，这意味着：

- AI 不是让你更轻率；
- AI 应该让你更完整。

---

## 5. 它的局限与代价

这套体系非常强，但也不是没有代价。

## 5.1 它是强方法论系统，不是中性工具箱

gstack 的价值很大程度来自它的强观点，但强观点也意味着：

- 它更适合 founder-led、产品驱动、节奏极快的小团队；
- 对流程已经很重、分工已经很细的大组织，直接照搬可能会显得“个人英雄主义”；
- 对极小的 trivial task，它可能有明显的 ceremony overhead。

也就是说，gstack 不是无风格的基础设施，而是一种明确的工作观。

## 5.2 它依赖真实工程管线，不是纯文本魔法

我在当前 checkout 里执行了两轮自动验证：

- `bun run skill:check`
- `bun test`

结果说明这套系统对工程环境有真实依赖，而不是“只靠大模型就行”：

- `skill:check` 当前失败，因为工作区没有安装 `diff` 依赖。
- `bun test` 跑到了 420 个测试中的 307 通过、102 跳过、11 失败、11 error；主要错误也来自当前环境缺少 `diff` 与 `@anthropic-ai/sdk`。

这并不说明设计失败，反而说明它是一个真正依赖生成器、解析器、E2E 与 SDK 的系统，而不是幻觉式流程。

但代价也很清楚：

- 依赖没装好，很多高阶保障就发挥不出来；
- 要让它在不同宿主上稳定工作，维护成本并不低。

## 5.3 Markdown 技能越强，越接近“软件”，也越需要软件级治理

随着 gstack 把越来越多规则写进技能：

- 技能本身会变长；
- 模板会变复杂；
- 跨宿主适配会增加维护面；
- prompt 供应链也会有自己的技术债。

因此它必须继续保持生成、校验、E2E、freshness check 这套治理结构，否则自己也会变成难以维护的复杂系统。

---

## 6. 我认为最值得借鉴的五条原则

如果要把 gstack 的价值抽象成对创业/变革团队最有用的五条，我会选这五条：

1. 先设计工作流，再放大 agent 数量。
2. 先验证问题定义，再验证代码实现。
3. 把资深经验写成默认约束，而不是期待每次临场发挥。
4. 把 QA、review、deploy、retro 接成一个闭环，而不是离散动作。
5. AI 带来的不应只是速度，而是完整性标准的上升。

---

## 7. 证据摘录

### 7.1 仓库层面的直接证据

- “virtual engineering team” 的定位：`README.md:21-23`
- 推荐入口顺序是 `/office-hours` → `/plan-ceo-review` → `/review` → `/qa`，不是“直接写功能”：`README.md:32-39`
- 跨宿主支持与 28 skills：`README.md:57-89`
- Builder ethos 自动注入每个 workflow preamble：`ETHOS.md:3-5`
- 持久化浏览器与 100-200ms 命令延迟：`ARCHITECTURE.md:7-36`
- 模板系统与占位符生成：`ARCHITECTURE.md:179-220`
- 命令注册表是 single source of truth：`browse/src/commands.ts:1-10`
- 生成器支持 Claude/Codex host 适配：`scripts/gen-skill-docs.ts:22-53`
- E2E 通过真实 `claude -p` 子进程跑：`test/helpers/session-runner.ts:1-26`, `test/helpers/session-runner.ts:152-184`
- Codex 也有独立 E2E：`test/codex-e2e.test.ts:1-14`, `test/codex-e2e.test.ts:120-196`

### 7.2 本次自动扫描统计

- `SKILL.md` 数量：28
- `SKILL.md.tmpl` 数量：28
- `*.test.ts` 数量：38
- 模板中 `{{PREAMBLE}}` 引用：23
- 模板中 `AskUserQuestion` 出现：146
- 模板中 `STOP` 出现：52

这些数字共同说明：gstack 的核心不是“自由发挥”，而是“规范化停顿、规范化提问、规范化前导与规范化交付”。

---

## 8. 后续 Todo

- Todo 1：在安装依赖后重新执行 `bun run skill:check` 与 `bun test`，确认当前 checkout 的失败是否完全来自环境缺失，而不是代码回归。
- Todo 2：在一个最小示例仓库里跑完整链路：`/office-hours` → `/autoplan` → `/qa` → `/ship`，记录 turns、耗时、产物数量与人工介入点。
- Todo 3：量化“blank prompt vs gstack workflow”的差异，尤其是需求澄清轮次、scope drift、回归 bug 数、文档补写成本。
- Todo 4：进一步比较 gstack 与其他 agent workflow 项目，区分哪些是普适原则，哪些是 Garry Tan/YC 语境下的特定判断。
- Todo 5：专门研究 `plan-eng-review` 与 `review` 的完整 checklist，提炼出一套适用于中国创业团队的本地化版本。

---

## 最后的判断

如果只把 gstack 看成“Claude/Codex 技能包”，会低估它。

它真正表达的是一种组织论：

- 用技能代替部分会议，
- 用工件代替部分记忆，
- 用验证代替部分自信，
- 用第二意见代替单点盲区，
- 用 AI 把完整性重新变成默认选项。

这也是它为什么能极大提升生产力。因为它优化的不只是写代码的速度，而是从想清楚、做完整、验明白、发出去、复盘掉这整条价值链。
