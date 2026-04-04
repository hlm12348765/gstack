# 技能深度解析

每个 gstack 技能的详细指南：理念、工作流和示例。

| 技能 | 你的专家 | 他们做什么 |
|-------|----------------|--------------|
| [`/office-hours`](#office-hours) | **YC Office Hours** | 从这里开始。在你写代码之前，先用六个强制性问题重新定义你的产品。它会反驳你的表述方式、质疑前提，并生成实现方案备选。设计文档会流入后续每一个技能。 |
| [`/plan-ceo-review`](#plan-ceo-review) | **CEO / Founder** | 重新思考问题。在需求中找出那个隐藏着的 10 星产品。四种模式：Expansion、Selective Expansion、Hold Scope、Reduction。 |
| [`/plan-eng-review`](#plan-eng-review) | **Eng Manager** | 锁定架构、数据流、图表、边界情况和测试。强迫隐藏假设浮出水面。 |
| [`/plan-design-review`](#plan-design-review) | **Senior Designer** | 交互式计划模式设计评审。对每个维度按 0-10 打分，说明 10 分是什么样，并修正计划。在计划模式下工作。 |
| [`/design-consultation`](#design-consultation) | **Design Partner** | 从零开始构建设计系统。了解整体设计生态，提出有创意的风险点，生成逼真的产品模型图。设计是所有其他阶段的核心。 |
| [`/review`](#review) | **Staff Engineer** | 找出那些能通过 CI 却会在生产环境中爆炸的 bug。自动修复明显的问题。标记完整性缺口。 |
| [`/investigate`](#investigate) | **Debugger** | 系统化的根因调试。铁律：不先调查就不修复。追踪数据流，验证假设，连续 3 次修复失败后停止。 |
| [`/design-review`](#design-review) | **Designer Who Codes** | 线上站点视觉审计与修复循环。先做 80 项审计，再修复发现的问题。原子提交，前后截图对比。 |
| [`/qa`](#qa) | **QA Lead** | 测试你的应用，找出 bug，用原子提交修复，然后重新验证。为每次修复自动生成回归测试。 |
| [`/qa-only`](#qa) | **QA Reporter** | 与 /qa 使用相同方法，但只出报告。适用于你只想要纯 bug 报告、不做代码修改时。 |
| [`/ship`](#ship) | **Release Engineer** | 同步 main，运行测试，审查覆盖率，推送，创建 PR。如果你还没有测试框架，它会帮你搭好。一条命令完成。 |
| [`/cso`](#cso) | **Chief Security Officer** | 基于 OWASP Top 10 + STRIDE 威胁建模进行安全审计。扫描注入、认证、加密和访问控制问题。 |
| [`/document-release`](#document-release) | **Technical Writer** | 更新所有项目文档，使其与你刚发布的内容一致。自动捕捉过期的 README。 |
| [`/retro`](#retro) | **Eng Manager** | 具备团队感知的每周复盘。按人拆解、发版连胜、测试健康趋势、成长机会。 |
| [`/browse`](#browse) | **QA Engineer** | 让代理拥有“眼睛”。真实 Chromium 浏览器、真实点击、真实截图。每条命令约 ~100ms。 |
| [`/setup-browser-cookies`](#setup-browser-cookies) | **Session Manager** | 把你真实浏览器中的 cookies（Chrome、Arc、Brave、Edge）导入无头会话。测试需要登录的页面。 |
| | | |
| **Multi-AI** | | |
| [`/codex`](#codex) | **Second Opinion** | 来自 OpenAI Codex CLI 的独立评审。三种模式：代码评审（pass/fail 门禁）、对抗式挑战、带会话连续性的开放式咨询。当 `/review` 和 `/codex` 都运行过后，可进行跨模型分析。 |
| | | |
| **Safety & Utility** | | |
| [`/careful`](#safety--guardrails) | **Safety Guardrails** | 在危险命令前发出警告（rm -rf、DROP TABLE、force-push、git reset --hard）。任何警告都可覆盖。常见构建清理已加入白名单。 |
| [`/freeze`](#safety--guardrails) | **Edit Lock** | 将所有文件编辑限制在单个目录内。阻止边界外的 Edit 和 Write。用于调试时防止误操作。 |
| [`/guard`](#safety--guardrails) | **Full Safety** | 一条命令同时启用 /careful + /freeze。适合生产环境工作的最高安全级别。 |
| [`/unfreeze`](#safety--guardrails) | **Unlock** | 移除 /freeze 边界，再次允许在任意位置编辑。 |
| [`/gstack-upgrade`](#gstack-upgrade) | **Self-Updater** | 将 gstack 升级到最新版本。检测全局安装还是项目内置安装，同步二者，并展示变更内容。 |

---

## `/office-hours`

这是每个项目都应该开始的地方。

在你规划之前，在你评审之前，在你写代码之前，先和一个 YC 风格的合作者坐下来，思考你真正要构建的是什么。不是你以为自己在构建什么，而是你*实际上*在构建什么。

### 重新定义

下面是真实项目中发生过的事。用户说：“我想做一个面向日历的每日简报应用。”这是个合理的需求。接着它追问痛点，要具体例子，不要假设。用户描述了一位助手总会漏掉事情、多个 Google 账号下的日历条目里信息陈旧、准备文档充满 AI 味，还有活动地点写错却要花很久才能查清。

它随后回应：*"我要反驳一下这个表述方式，因为我觉得你已经超出这个框架了。你说的是‘面向多 Google Calendar 管理的每日简报应用’。但你真正描述的是一个个人幕僚型 AI。"*

然后它提取出了用户自己都没意识到正在描述的五项能力：

1. **监控你的日历**，覆盖所有账号，并检测陈旧信息、缺失地点、权限缺口
2. **生成真正有用的准备工作**，不是后勤摘要，而是为董事会会议、播客、募资活动做准备所需的*思考工作*
3. **管理你的 CRM**，包括你要见谁、关系如何、对方想要什么、过往历史是什么
4. **为你的时间排序优先级**，在需要提前开始准备时发出标记，主动预留时间，并按重要程度排列活动
5. **用金钱换杠杆**，主动寻找可委派或可自动化的事项

这次重新定义彻底改变了整个项目。他们本来正准备做一个日历应用。现在他们要做的是一个价值高出十倍的东西，因为这个技能听的是他们的痛点，而不是功能需求。

### 前提质疑

完成重新定义之后，它会提出一些需要你验证的前提。不是“这样听起来好吗？”这种问题，而是关于产品的、可被证伪的真实主张：

1. 日历是锚点数据源，但真正的价值在上层的智能层
2. 助手不会被替代，而是会被赋能
3. 最窄的切入口，是一个真正好用的每日简报
4. CRM 集成不是锦上添花，而是必须具备

你可以同意、不同意，或者调整。每一个你接受的前提，都会成为设计文档中的承重部分。

### 实现方案备选

接着它会生成 2-3 个具体实现方案，并附带诚实的工作量估算：

- **Approach A: Daily Briefing First** — 最窄切口，明天就能上线，M 工作量（人工：约 3 周 / CC：约 2 天）
- **Approach B: CRM-First** — 先构建关系图谱，L 工作量（人工：约 6 周 / CC：约 4 天）
- **Approach C: Full Vision** — 一次性全做，XL 工作量（人工：约 3 个月 / CC：约 1.5 周）

它会推荐 A，因为你可以从真实使用中学习。CRM 数据会在第二周自然到来。

### 两种模式

**Startup mode** — 面向正在创业或在企业内部创业的人。你会得到六个强制性问题，这些问题浓缩自 YC 合伙人评估产品的方式：真实需求、现状方案、迫切而具体的痛点、最窄切口、观察与意外、未来适配。这些问题故意让人不舒服。如果你连一个明确需要你产品的具体人都说不出来，那么在写任何代码之前，这就是你最重要该弄清楚的事。

**Builder mode** — 面向 hackathon、副项目、开源、学习和图个乐的人。你会得到一个热情的合作者，帮助你找出自己想法里最酷的版本。什么会让人说“哇”？什么是最快能做出可分享成果的路径？这些问题是生成式的，不是盘问式的。

### 设计文档

两种模式最后都会产出一份写入 `~/.gstack/projects/` 的设计文档，而这份文档会直接流入 `/plan-ceo-review` 和 `/plan-eng-review`。完整生命周期现在是：`office-hours → plan → implement → review → QA → ship → retro`。

设计文档获批之后，`/office-hours` 会回顾它观察到的你的思考方式，不是空泛的夸赞，而是对你在会话中说过的话做具体回扣。这些观察也会出现在设计文档里，因此你日后重读时会再次遇到它们。

---

## `/plan-ceo-review`

这是我的**创始人模式**。

在这里，我希望模型以品味、野心、用户同理心和长期视角来思考。我不希望它按字面照做。我希望它先问一个更重要的问题：

**这个产品真正是为了解决什么？**

我把它看作 **Brian Chesky mode**。

重点不是实现那个显而易见的工单。重点是从用户视角重新思考问题，找出那个让人觉得理所当然、愉悦，甚至带点魔力的版本。

### 示例

假设我在做一个类似 Craigslist 的发布应用，我说：

> “让卖家可以为商品上传一张照片。”

弱一点的助手会加一个文件选择器，然后保存图片。

那不是这个产品真正该做的事。

在 `/plan-ceo-review` 中，我希望模型去问，“上传照片”到底是不是那个真正的功能。也许真正的功能，是帮助用户创建一个真正能卖出去的发布内容。

如果那才是真正的任务，整个计划都会改变。

现在模型应该问：

* 我们能从照片中识别出商品吗？
* 我们能推断出 SKU 或型号吗？
* 我们能搜索网络并自动起草标题和描述吗？
* 我们能拉取规格、分类和价格对标吗？
* 我们能建议哪张照片最适合作为主图吗？
* 我们能检测上传的照片是否丑、过暗、杂乱或缺乏可信感吗？
* 我们能让这个体验感觉高级，而不是像一个 2007 年留下来的死板表单吗？

这就是 `/plan-ceo-review` 为我做的事。

它不只是问：“我该怎么加上这个功能？”
它会问，**“这个需求里隐藏着什么样的 10 星产品？”**

### 四种模式

- **SCOPE EXPANSION** — 放开去想。代理提出更有野心的版本。每个扩展都会作为一个独立决策呈现，由你选择是否纳入。会积极推荐。
- **SELECTIVE EXPANSION** — 以当前范围为基线，但看看还有什么可能。代理逐项提出机会，并给出中性建议，由你挑选值得做的部分。
- **HOLD SCOPE** — 对现有计划进行最严格的打磨。不提出任何扩展。
- **SCOPE REDUCTION** — 找出最小可行版本。其他内容全部砍掉。

愿景和决策会持久化到 `~/.gstack/projects/`，因此不会随着对话结束而丢失。特别出色的愿景还可以提升到仓库里的 `docs/designs/`，供团队使用。

---

## `/plan-eng-review`

这是我的**工程经理模式**。

一旦产品方向正确，我需要的就是完全不同的一种智能。我不想再要更多发散式创意。我不想再听“要是这样岂不是很酷”。我希望模型变成我最好的技术负责人。

这个模式应该牢牢拿下：

* 架构
* 系统边界
* 数据流
* 状态迁移
* 故障模式
* 边界情况
* 信任边界
* 测试覆盖

还有一个对我来说特别大的突破点：**图表**。

当你强迫 LLM 把系统画出来时，它们会完整得多。时序图、状态图、组件图、数据流图，甚至测试矩阵。图表会逼着隐藏假设浮出水面，让空泛的规划更难混过去。

所以 `/plan-eng-review` 是我希望模型构建技术脊梁的地方，让这根脊梁能承载产品愿景。

### 示例

还是那个发布应用的例子。

假设 `/plan-ceo-review` 已经完成了它的工作。我们决定，真正的功能不只是上传照片，而是一个智能发布流程，它能：

* 上传照片
* 识别商品
* 从网络补全发布信息
* 起草强有力的标题和描述
* 推荐最合适的主图

现在由 `/plan-eng-review` 接手。

这时我希望模型回答这样的问题：

* 上传、分类、补全和草稿生成的架构应该是什么？
* 哪些步骤同步执行，哪些步骤应该放进后台任务？
* 应用服务器、对象存储、视觉模型、搜索/补全 API 以及发布数据库之间的边界在哪里？
* 如果上传成功但补全失败，会发生什么？
* 如果商品识别置信度低，会发生什么？
* 重试机制如何工作？
* 我们如何防止重复任务？
* 哪些数据在什么时候持久化，哪些可以安全地重新计算？

而这正是我希望看到图表的地方：架构图、状态模型、数据流图、测试矩阵。图表会逼着隐藏假设浮出水面，让空泛规划更难过关。

这就是 `/plan-eng-review`。

不是“把想法变小一点。”
而是 **“把想法变得可构建。”**

### Review Readiness Dashboard

每次评审（CEO、Eng、Design）都会记录结果。每轮评审结束时，你都会看到一个仪表板：

```
+====================================================================+
|                    REVIEW READINESS DASHBOARD                       |
+====================================================================+
| Review          | Runs | Last Run            | Status    | Required |
|-----------------|------|---------------------|-----------|----------|
| Eng Review      |  1   | 2026-03-16 15:00    | CLEAR     | YES      |
| CEO Review      |  1   | 2026-03-16 14:30    | CLEAR     | no       |
| Design Review   |  0   | —                   | —         | no       |
+--------------------------------------------------------------------+
| VERDICT: CLEARED — Eng Review passed                                |
+====================================================================+
```

Eng Review 是唯一必需的门禁（可用 `gstack-config set skip_eng_review true` 关闭）。CEO 和 Design 只提供信息参考，分别推荐用于产品改动和 UI 改动。

### 从计划到 QA 的流程

当 `/plan-eng-review` 完成测试评审部分时，它会把一份测试计划产物写入 `~/.gstack/projects/`。之后你运行 `/qa` 时，它会自动读取这份测试计划，也就是说你的工程评审会直接流入 QA 测试，无需手动复制粘贴。

---

## `/plan-design-review`

这是我的**资深设计师在你写下一行代码之前对计划进行评审**的模式。

大多数计划只描述后端要做什么，却从不说明用户真正会看到什么。空状态？错误状态？加载状态？移动端布局？AI 味风险？这些决策都会被推迟到“实现时再看”——于是工程师最后就会交付一个空状态写着 “No items found.”，因为根本没有人指定更好的方案。

`/plan-design-review` 会在规划阶段就抓出这些问题，因为那时修起来最便宜。

它的工作方式和 `/plan-ceo-review`、`/plan-eng-review` 一样：交互式、一次处理一个问题，并使用 **STOP + AskUserQuestion** 模式。它会对每个设计维度按 0-10 打分，说明 10 分是什么样，然后修改计划以达到那个水平。评分会驱动工作：分数低 = 要修很多，分数高 = 快速通过。

它会对计划做七轮检查：信息架构、交互状态覆盖、用户旅程、AI 味风险、设计系统一致性、响应式/无障碍，以及未解决的设计决策。每一轮里，它会找出缺口，要么直接修掉明显问题，要么请你做设计选择，处理真实存在的权衡。

### 示例

```
You:   /plan-design-review

Claude: Initial Design Rating: 4/10

        "This plan describes a user dashboard but never specifies
         what the user sees first. It says 'cards with icons' —
         which looks like every SaaS template. It mentions zero
         loading states, zero empty states, and no mobile behavior."

        Pass 1 (Info Architecture): 3/10
        "A 10 would define primary/secondary/tertiary content
         hierarchy for every screen."
        → Added information hierarchy section to plan

        Pass 2 (Interaction States): 2/10
        "The plan has 4 UI features but specifies 0 out of 20
         interaction states (4 features × 5 states each)."
        → Added interaction state table to plan

        Pass 4 (AI Slop): 4/10
        "The plan says 'clean, modern UI with cards and icons'
         and 'hero section with gradient'. These are the top 2
         AI-generated-looking patterns."
        → Rewrote UI descriptions with specific, intentional alternatives

        Overall: 4/10 → 8/10 after fixes
        "Plan is design-complete. Run /design-review after
         implementation for visual QA."
```

当你再次运行它时，已经达到 8 分以上的部分只会快速过一遍。低于 8 分的部分则会被完整处理。若要在实现后对线上站点进行视觉审计，请使用 `/design-review`。

---

## `/design-consultation`

这是我的**设计合伙人模式**。

`/plan-design-review` 审核的是一个已经存在的站点。`/design-consultation` 则适用于你现在什么都还没有的时候：没有设计系统、没有字体选择、没有配色方案。你从零开始，希望有一位资深设计师坐下来，和你一起构建完整的视觉识别系统。

这是一场对话，不是一张表单。代理会询问你的产品、用户和受众。它会思考你的产品需要传达什么：信任、速度、匠心、温度，或者任何适合的气质，然后从这些目标反推到具体选择。接着它会提出一整套完整且一致的设计系统：审美方向、字体系统（3 种以上字体并指定具体职责）、带 hex 值的配色、间距尺度、布局方法和动效策略。每条建议都有理由。每个选择都会相互强化。

但一致性只是基本要求。每个开发工具仪表板看起来都一样：干净的无衬线字体、克制的灰色、一个蓝色强调色。它们都很一致，也都很容易被忘记。一个产品看起来“不错”和一个产品真正能让人认出来之间的差异，在于**有意识地承担创意风险**：标题使用意想不到的衬线字体、采用同类产品里没人用的大胆强调色、更紧凑的间距让数据看起来更有权威感，而不是松散飘忽。

这才是 `/design-consultation` 真正关心的事。它不只是提出一套安全的系统。它会同时提出安全选择和冒险选择，并告诉你哪些属于哪一类。“这些选择能让你保持在这个品类中的可识别性。而这些地方，我认为你应该打破惯例，以及原因是什么。”你来决定要承担哪些风险。无论你怎么选，代理都会检查整套系统是否依然保持一致。

如果你愿意，代理还会研究你所在领域里已经存在的产品，截取真实站点的截图，分析它们的字体、颜色和间距，这样你就可以在做选择之前先看清全貌。这不是为了抄袭，而是为了进入合理区间，先知道惯例是什么，再决定哪些值得打破。

在你同意这套系统之后，它会生成一个交互式 HTML 预览页，不只是色板和字体样张，而是真实感很强的产品页面。如果你在做仪表板，你会看到带侧边栏、数据表和统计卡片的仪表板。如果你在做营销站点，你会看到带真实文案和 CTA 的 hero 区。所有内容都用你的设计系统、你的产品名来渲染，并同时提供浅色和深色模式。在一行生产代码都没写之前，你就能先看到你的产品可能会是什么感觉。

然后它会把 `DESIGN.md` 写到仓库根目录，作为项目设计的权威来源，并更新 `CLAUDE.md`，让今后的每次 Claude Code 会话都遵循这套系统。从那时起，`/design-review` 可以基于它做审计，任何在你前端上工作的代理也都知道规则是什么。

### 示例

```
You:   /design-consultation

Claude: From the README, this looks like a developer analytics dashboard
        for engineering teams. Sound right? Tell me about the feeling
        you want — and would you like me to look at what's out there
        first so we can see the landscape?

You:    Yes, it's for eng managers. Show me what's out there.

Claude: [browses Linear, Vercel, Datadog, Posthog — takes screenshots]

        Here's the landscape. They all converge on clean sans-serifs
        and muted blue-gray palettes. Linear stands out with dark-first
        and tight spacing. The rest feel interchangeable.

        Here's my proposal — and where I'd play it safe vs. take risks:

        AESTHETIC: Industrial/Utilitarian — data-first, every pixel earns its place
        TYPOGRAPHY: Geist (body + UI), Instrument Serif (display), JetBrains Mono (data)
        COLOR: Restrained — slate neutrals, single teal accent (#0D9488)
        SPACING: 4px base, compact density
        LAYOUT: Grid-disciplined, 12-column
        MOTION: Minimal-functional — only transitions that aid comprehension

        SAFE CHOICES (match category expectations):
          Geist for body — your users already know this font from Vercel
          Slate neutrals — warm grays feel right for data-heavy UIs
          Grid-disciplined layout — dashboards need predictable alignment

        RISKS (where you'd stand out):
          Instrument Serif for display headings — nobody in dev tools
            uses a serif. It creates instant hierarchy contrast and
            signals "we care about craft" in a space full of sans-serifs.
          Single teal accent instead of blue — everyone else in this space uses blue.
            Teal is close enough to feel trustworthy, different enough
            to be recognizable.
          4px compact density — tighter than Linear, says "we respect
            your screen real estate."

        The safe choices keep you literate. The risks are where your
        product gets its own face. Which risks appeal to you?

You:    Love the serif risk. Let's see it.

Claude: Preview opened in your browser.
        [/tmp/design-consultation-preview-1710000000.html]

        The page shows a realistic dashboard: sidebar nav, header with
        avatar, stat cards, data table — all in your design system.
        Instrument Serif at hero scale, Geist in body text, JetBrains
        Mono in the data table. Teal accent on buttons and active states.
        Light and dark mode toggle in the corner.

        How does it feel? Want to adjust anything before I write DESIGN.md?

You:    Ship it.

Claude: Wrote DESIGN.md (typography, color, spacing, layout, motion).
        Updated CLAUDE.md with design system guidelines.
        All future sessions will respect these choices.
```

---

## `/design-review`

这是我的**会写代码的设计师模式**。

`/plan-design-review` 会在实现之前审查你的计划。`/design-review` 则会在实现之后审计并修复线上站点。

它会先对你的线上站点进行 80 项视觉审计，然后进入修复循环：针对每一项设计问题，定位源文件，做最小化的 CSS/样式修改，使用 `style(design): FINDING-NNN` 提交，重新访问页面验证，并拍摄修复前后截图。每个修复一条提交，完全可二分定位。

它的自我调节启发式专门针对设计工作进行调整：仅 CSS 的改动可以直接通过（它们天生更安全且可逆），但对组件 JSX/TSX 文件的修改会计入风险预算。最多修 30 项。若风险分数超过 20%，它就会停下并询问。

### 示例

```
You:   /design-review https://myapp.com

Claude: [Runs full 80-item visual audit on the live site]
        Design Score: C  |  AI Slop Score: D
        12 findings (4 high, 5 medium, 3 polish)

        Fixing 9 design issues...

        style(design): FINDING-001 — replace 3-column icon grid with asymmetric layout
        style(design): FINDING-002 — add heading scale 48/32/24/18/16
        style(design): FINDING-003 — remove gradient hero, use bold typography
        style(design): FINDING-004 — add second font for headings
        style(design): FINDING-005 — vary border-radius by element role
        style(design): FINDING-006 — left-align body text, reserve center for headings
        style(design): FINDING-007 — add hover/focus states to all interactive elements
        style(design): FINDING-008 — add prefers-reduced-motion media query
        style(design): FINDING-009 — set max content width to 680px for body text

        Final audit:
        Design Score: C → B+  |  AI Slop Score: D → A
        9 fixes applied (8 verified, 1 best-effort). 3 deferred.

        [Report with before/after screenshots saved to .gstack/design-reports/]
```

九条提交，每条只处理一个关注点。AI Slop 分数从 D 提升到 A，因为最容易被认出来的三种模式（渐变 hero、三栏网格、统一圆角）都被去掉了。

---

## `/review`

这是我的**偏执型资深工程师模式**。

测试通过，并不意味着这个分支就是安全的。

`/review` 存在的原因，是有一整类 bug 可以活过 CI，却依然会在生产环境里狠狠干你一拳。这个模式不是为了想得更大，也不是为了把计划修饰得更漂亮。它问的是：

**还有什么可能会坏？**

这是结构性审计，不是吹毛求疵的样式检查。我希望模型寻找的是这类问题：

* N+1 查询
* 脏旧读取
* 竞态条件
* 糟糕的信任边界
* 缺失索引
* 转义 bug
* 被破坏的不变量
* 错误的重试逻辑
* 虽然测试通过，但没有覆盖真实故障模式的测试
* 被遗忘的 enum 处理器——新增一个状态或类型常量时，`/review` 会沿着整个代码库追踪它经过的每个 switch 语句和 allowlist，而不仅仅是你修改过的文件

### Fix-First

发现问题之后要采取行动，而不只是列出来。明显的机械性修复（死代码、过期注释、N+1 查询）会被自动应用——你会看到每一项都显示为 `[AUTO-FIXED] file:line Problem → what was done`。真正有歧义的问题（安全、竞态条件、设计决策）则会交给你判断。

### 完整性缺口

`/review` 现在还会标记那些取巧实现的地方：如果完整版本花费不到 30 分钟 CC 时间，而你只做了一个 80% 的版本，那么评审就会指出这一点。前提是，100% 的方案是一片湖，而不是一片海。

### 示例

假设那个智能发布流程已经实现，而且测试全绿。

`/review` 仍然应该继续追问：

* 我在渲染发布照片或草稿建议时是否引入了 N+1 查询？
* 我是否相信了客户端提供的文件元数据，而没有去验证真实文件？
* 两个标签页会不会发生竞态，导致主图选择或商品详情被覆盖？
* 失败的上传会不会永远在存储里留下孤儿文件？
* “恰好只有一张主图” 这个规则在并发下会不会失效？
* 如果补全 API 部分失败，我是优雅降级，还是把垃圾数据保存了进去？
* 我是不是在把网络数据喂给草稿生成时，意外制造了 prompt injection 或信任边界问题？

这就是 `/review` 的意义。

我不想在这里听恭维。
我希望模型在事故真正发生之前，就先把生产事故想出来。

---

## `/investigate`

当某些东西坏了而你又不知道原因时，`/investigate` 就是你的系统化调试器。它遵循铁律：**不先调查根因，就不进行修复。**

它不会靠猜测和打补丁来碰运气，而是追踪数据流，对照已知 bug 模式，并一次只验证一个假设。如果连续三次修复尝试失败，它就会停下来，转而质疑架构，而不是继续瞎折腾。这能防止那种“让我再试最后一次”的恶性循环，白白浪费几个小时。

---

## `/qa`

这是我的**QA 负责人模式**。

`/browse` 给代理带来“眼睛”。`/qa` 则给它一套测试方法论。

最常见的使用场景是：你在一个功能分支上，刚写完代码，想验证一切都正常。只要输入 `/qa` —— 它会读取你的 git diff，识别你的改动影响了哪些页面和路由，启动浏览器，并逐个测试。无需提供 URL。无需手写测试计划。

四种模式：

- **Diff-aware**（功能分支上自动启用）— 读取 `git diff main`，识别受影响页面，并专门测试这些页面
- **Full** — 系统性探索整个应用。5-15 分钟。记录 5-10 个证据充分的问题。
- **Quick**（`--quick`）— 30 秒冒烟测试。首页 + 前 5 个导航目标。
- **Regression**（`--regression baseline.json`）— 运行完整模式，然后与之前的 baseline 做 diff。

### 自动回归测试

当 `/qa` 修复一个 bug 并验证通过后，它会自动生成一个回归测试，准确捕捉当初出问题的场景。测试会包含完整归因，可追溯到 QA 报告。

### 示例

```
You:   /qa https://staging.myapp.com

Claude: [Explores 12 pages, fills 3 forms, tests 2 flows]

        QA Report: staging.myapp.com — Health Score: 72/100

        Top 3 Issues:
        1. CRITICAL: Checkout form submits with empty required fields
        2. HIGH: Mobile nav menu doesn't close after selecting an item
        3. MEDIUM: Dashboard chart overlaps sidebar below 1024px

        [Full report with screenshots saved to .gstack/qa-reports/]
```

**测试需要登录的页面：** 先使用 `/setup-browser-cookies` 导入你真实浏览器中的会话，然后 `/qa` 就能测试登录后的页面。

---

## `/ship`

这是我的**发布机器模式**。

一旦我已经决定要构建什么，敲定了技术方案，也完成了严肃评审，我就不想再讨论了。我想要执行。

`/ship` 面向最后一公里。它适用于一个已经准备好的分支，而不是用来决定要构建什么。

在这里，模型应该停止扮演头脑风暴伙伴，开始像一个纪律严明的发布工程师那样行动：同步 main，运行合适的测试，确认分支状态正常，必要时更新 changelog 或版本号，推送，并创建或更新 PR。

### 测试引导

如果你的项目还没有测试框架，`/ship` 会帮你搭建一个——检测你的运行时，研究最合适的框架，安装它，为你的真实代码编写 3-5 个真正可用的测试，配置 CI/CD（GitHub Actions），并创建 `TESTING.md`。目标是 100% 测试覆盖——测试让 vibe coding 变得安全，而不是 yolo coding。

### 覆盖率审计

每次运行 `/ship`，都会根据你的 diff 构建一张代码路径图，搜索对应测试，并产出一个带质量星级的 ASCII 覆盖率图。缺口会自动生成测试。你的 PR 正文里会显示覆盖变化：`Tests: 42 → 47 (+5 new)`。

### 评审门禁

`/ship` 在创建 PR 之前会检查 [Review Readiness Dashboard](#review-readiness-dashboard)。如果缺少 Eng Review，它会询问，但不会阻止你。决定会按分支保存，所以不会重复问你。

很多分支会死在最有趣的工作已经完成、只剩无聊发布工作的那一步。人会拖延这部分。AI 不应该。

---

## `/cso`

这是我的**Chief Security Officer**。

在任意代码库上运行 `/cso`，它就会执行一次基于 OWASP Top 10 + STRIDE 威胁模型的审计。它会扫描注入漏洞、认证失效、敏感数据泄露、XML 外部实体、访问控制失效、安全配置错误、XSS、不安全反序列化、已知漏洞组件以及日志不足。每项发现都会包含严重级别、证据和推荐修复方案。

```
You:   /cso

Claude: Running OWASP Top 10 + STRIDE security audit...

        CRITICAL: SQL injection in user search (app/models/user.rb:47)
        HIGH: Session tokens stored in localStorage (app/frontend/auth.ts:12)
        MEDIUM: Missing rate limiting on /api/login endpoint
        LOW: X-Frame-Options header not set

        4 findings across 12 files scanned. 1 critical, 1 high.
```

---

## `/document-release`

这是我的**技术写作模式**。

在 `/ship` 创建 PR 之后、合并之前，`/document-release` 会读取项目中的每一份文档文件，并与 diff 交叉比对。它会更新文件路径、命令列表、项目结构树，以及其他任何已经漂移的内容。高风险或带主观性的修改会作为问题抛出来，其他部分则会自动处理。

```
You:   /document-release

Claude: Analyzing 21 files changed across 3 commits. Found 8 documentation files.

        README.md: updated skill count from 9 to 10, added new skill to table
        CLAUDE.md: added new directory to project structure
        CONTRIBUTING.md: current — no changes needed
        TODOS.md: marked 2 items complete, added 1 new item

        All docs updated and committed. PR body updated with doc diff.
```

它还会润色 CHANGELOG 的语气风格（但绝不会覆盖现有条目）、清理已完成的 TODOS、检查跨文档一致性，并且只在合适的时候才询问 VERSION 版本升级。

---

## `/retro`

这是我的**工程经理模式**。

到了周末，我想知道实际发生了什么。不是感觉，而是数据。`/retro` 会分析提交历史、工作模式和发版速度，并写出一份坦率的复盘。

它具备团队感知能力。它会识别是谁在运行这个命令，对你自己的工作给出最深入的分析，然后再对每位贡献者分别给出具体表扬和成长机会。它会计算提交数、LOC、测试比例、PR 大小、修复比例等指标。它会从提交时间戳中识别编码时段，找出热点文件，跟踪发版连胜，并识别本周最大的那次交付。

它还会跟踪测试健康度：测试文件总数、本周期新增测试数、回归测试提交数，以及趋势变化。如果测试比例低于 20%，它会将其标记为一个成长方向。

### 示例

```
You:   /retro

Claude: Week of Mar 1: 47 commits (3 contributors), 3.2k LOC, 38% tests, 12 PRs, peak: 10pm | Streak: 47d

        ## Your Week
        32 commits, +2.4k LOC, 41% tests. Peak hours: 9-11pm.
        Biggest ship: cookie import system (browser decryption + picker UI).
        What you did well: shipped a complete feature with encryption, UI, and
        18 unit tests in one focused push...

        ## Team Breakdown

        ### Alice
        12 commits focused on app/services/. Every PR under 200 LOC — disciplined.
        Opportunity: test ratio at 12% — worth investing before payment gets more complex.

        ### Bob
        3 commits — fixed the N+1 query on dashboard. Small but high-impact.
        Opportunity: only 1 active day this week — check if blocked on anything.

        [Top 3 team wins, 3 things to improve, 3 habits for next week]
```

它会把一个 JSON 快照保存到 `.context/retros/`，这样下一次运行时就可以展示趋势。

---

## `/browse`

这是我的**QA 工程师模式**。

`/browse` 是补上闭环的那个技能。在它出现之前，代理虽然能思考、能写代码，但依旧几乎是半盲状态。它不得不去猜 UI 状态、认证流程、跳转、控制台错误、空状态和破损布局。现在它可以直接去看。

它是一个编译好的二进制程序，通过 Playwright 与一个持久运行的 Chromium 守护进程通信，这套东西由 Microsoft 构建。第一次调用会启动浏览器（约 ~3 秒）。之后每次调用大约只需 ~100-200ms。浏览器会在命令之间持续运行，因此 cookies、标签页和 localStorage 都会保留下来。

### 示例

```
You:   /browse staging.myapp.com — log in, test the signup flow, and check
       every page I changed in this branch

Claude: [18 tool calls, ~60 seconds]

        > browse goto https://staging.myapp.com/signup
        > browse snapshot -i
        > browse fill @e2 "test@example.com"
        > browse fill @e3 "password123"
        > browse click @e5                    (Submit)
        > browse screenshot /tmp/signup.png
        > Read /tmp/signup.png

        Signup works. Redirected to onboarding. Now checking changed pages.

        > browse goto https://staging.myapp.com/dashboard
        > browse screenshot /tmp/dashboard.png
        > Read /tmp/dashboard.png
        > browse console

        Dashboard loads. No console errors. Charts render with sample data.

        All 4 pages load correctly. No console errors. No broken layouts.
        Signup → onboarding → dashboard flow works end to end.
```

18 次工具调用，大约一分钟。一次完整 QA 检查。没有打开浏览器窗口。

### 浏览器交接

当无头浏览器卡住时——例如 CAPTCHA、MFA、复杂认证——就交给用户：

```
Claude: I'm stuck on a CAPTCHA at the login page. Opening a visible
        Chrome so you can solve it.

        > browse handoff "Stuck on CAPTCHA at login page"

        Chrome opened at https://app.example.com/login with all your
        cookies and tabs intact. Solve the CAPTCHA and tell me when
        you're done.

You:    done

Claude: > browse resume

        Got a fresh snapshot. Logged in successfully. Continuing QA.
```

浏览器会在交接过程中保留所有状态（cookies、localStorage、标签页）。执行 `resume` 之后，代理会获得你停下位置的最新快照。如果 browse 工具连续 3 次失败，它会自动建议使用 `handoff`。

**安全说明：** `/browse` 运行的是一个持久化的 Chromium 会话。cookies、localStorage 和会话状态会在命令之间持续保留。除非你确实有意这样做，否则不要把它用于敏感的生产环境——这是一个带有真实状态的真实浏览器。会话在空闲 30 分钟后会自动关闭。

完整命令参考见 [BROWSER.md](../BROWSER.md)。

---

## `/setup-browser-cookies`

这是我的**会话管理模式**。

在 `/qa` 或 `/browse` 能测试需要认证的页面之前，它们需要 cookies。与其每次都通过无头浏览器手动登录，不如直接用 `/setup-browser-cookies` 从你日常使用的浏览器中导入真实会话。

它会自动检测已安装的 Chromium 浏览器（Comet、Chrome、Arc、Brave、Edge），通过 macOS Keychain 解密 cookies，并把它们加载进 Playwright 会话。交互式选择器界面允许你精确选择要导入哪些域名，且绝不会显示任何 cookie 值。

```
You:   /setup-browser-cookies

Claude: Cookie picker opened — select the domains you want to import
        in your browser, then tell me when you're done.

        [You pick github.com, myapp.com in the browser UI]

You:    done

Claude: Imported 2 domains (47 cookies). Session is ready.
```

或者完全跳过界面：

```
You:   /setup-browser-cookies github.com

Claude: Imported 12 cookies for github.com from Comet.
```

---

## `/codex`

这是我的**第二意见模式**。

当 `/review` 以 Claude 的视角发现 bug 时，`/codex` 会带来一个完全不同的 AI —— OpenAI 的 Codex CLI —— 来审查同一份 diff。不同的训练、不同的盲点、不同的强项。两者的重叠部分说明那些问题大概率是真的。两边各自独有的发现，则是你能找到那些单靠任何一方都抓不到的 bug 的地方。

### 三种模式

**Review** — 对当前 diff 运行 `codex review`。Codex 会读取每一个改动文件，按严重级别对发现分类（P1 critical、P2 high、P3 medium），并返回 PASS/FAIL 结论。只要有任何 P1 发现，就是 FAIL。整个评审完全独立——Codex 看不到 Claude 的评审。

**Challenge** — 对抗模式。Codex 会主动尝试破坏你的代码。它会寻找边界情况、竞态条件、安全漏洞，以及在高负载下会失败的假设。使用最高推理强度（`xhigh`）。你可以把它理解为对逻辑进行一次渗透测试。

**Consult** — 带会话连续性的开放式对话。你可以向 Codex 询问任何关于代码库的问题。后续追问会复用同一个会话，因此上下文会保留下来。非常适合那种“我这样想对吗？”的时刻。

### 跨模型分析

当 `/review`（Claude）和 `/codex`（OpenAI）都审查过同一个分支时，你会得到一份跨模型对比：哪些发现是重叠的（高置信度），哪些是 Codex 独有的（不同视角），哪些是 Claude 独有的。这就是代码评审里的“两个医生，同一个病人”。

```
You:   /codex review

Claude: Running independent Codex review...

        CODEX REVIEW: PASS (3 findings)
        [P2] Race condition in payment handler — concurrent charges
             can double-debit without advisory lock
        [P3] Missing null check on user.email before downcase
        [P3] Token comparison not using constant-time compare

        Cross-model analysis (vs /review):
        OVERLAP: Race condition in payment handler (both caught it)
        UNIQUE TO CODEX: Token comparison timing attack
        UNIQUE TO CLAUDE: N+1 query in listing photos
```

---

## Safety & Guardrails

四个技能，为任意 Claude Code 会话增加安全护栏。它们通过 Claude Code 的 PreToolUse hooks 工作，透明、仅限当前会话、无需配置文件。

### `/careful`

当你接近生产环境、要运行危险命令，或者只是想要一个安全网时，可以说 “be careful” 或运行 `/careful`。每条 Bash 命令都会与已知危险模式进行匹配检查：

- `rm -rf` / `rm -r` — 递归删除
- `DROP TABLE` / `DROP DATABASE` / `TRUNCATE` — 数据丢失
- `git push --force` / `git push -f` — 重写历史
- `git reset --hard` — 丢弃提交
- `git checkout .` / `git restore .` — 丢弃未提交工作
- `kubectl delete` — 删除生产资源
- `docker rm -f` / `docker system prune` — 容器/镜像丢失

常见的构建产物清理（`rm -rf node_modules`、`dist`、`.next`、`__pycache__`、`build`、`coverage`）已加入白名单——日常操作不会误报。

你可以覆盖任何警告。这些护栏是为了防止误操作，不是访问控制。

### `/freeze`

把所有文件编辑限制在单个目录中。当你在调试一个计费 bug 时，你不会希望 Claude 顺手去“修复” `src/auth/` 里的无关代码。`/freeze src/billing` 会阻止该路径之外的所有 Edit 和 Write 操作。

`/investigate` 会自动启用这一点——它会检测正在调试的模块，并将编辑冻结在那个目录内。

```
You:   /freeze src/billing

Claude: Edits restricted to src/billing/. Run /unfreeze to remove.

        [Later, Claude tries to edit src/auth/middleware.ts]

Claude: BLOCKED — Edit outside freeze boundary (src/billing/).
        Skipping this change.
```

注意：这只会阻止 Edit 和 Write 工具。像 `sed` 这样的 Bash 命令仍然可以修改边界之外的文件——这是防误操作机制，不是安全沙箱。

### `/guard`

完整安全模式——一条命令同时启用 `/careful` + `/freeze`。既有危险命令警告，也有目录级编辑限制。适用于修改生产环境或调试线上系统时。

### `/unfreeze`

移除 `/freeze` 边界，再次允许在所有地方编辑。hooks 仍然会保留在当前会话中——只是此时允许一切。再次运行 `/freeze` 可设置新的边界。

---

## `/gstack-upgrade`

用一条命令让 gstack 保持最新。它会检测你的安装类型（全局安装在 `~/.claude/skills/gstack`，还是项目内置在 `.claude/skills/gstack`），执行升级，如果你同时有两份安装就同步二者，并展示发生了什么变化。

```
You:   /gstack-upgrade

Claude: Current version: 0.7.4
        Latest version: 0.8.2

        What's new:
        - Browse handoff for CAPTCHAs and auth walls
        - /codex multi-AI second opinion
        - /qa always uses browser now
        - Safety skills: /careful, /freeze, /guard
        - Proactive skill suggestions

        Upgraded to 0.8.2. Both global and project installs synced.
```

在 `~/.gstack/config.yaml` 中设置 `auto_upgrade: true`，即可完全跳过提示——当检测到新版本可用时，gstack 会在每次会话开始时静默升级。

---

## Greptile 集成

[Greptile](https://greptile.com) 是一家 YC 公司，可以自动评审你的 PR。它能抓到真正的 bug——竞态条件、安全问题，以及那些能通过 CI 却会在生产环境爆炸的东西。它确实不止一次救过我。我真的很喜欢这帮人。

### 设置

在 [greptile.com](https://greptile.com) 上把 Greptile 安装到你的 GitHub 仓库里——大概只要 30 秒。一旦它开始评审你的 PR，gstack 就会自动接收它的评论。无需额外配置。

### 工作方式

任何自动评审工具的难点都在于分诊。Greptile 很不错，但不是每条评论都是真问题。有些是假阳性。有些指出的是你三次提交之前就已经修掉的问题。如果没有分诊层，这些评论会不断堆积，你最后就会开始忽略它们——而这正违背了初衷。

gstack 解决了这个问题。`/review` 和 `/ship` 现在都具备 Greptile 感知能力。它们会读取 Greptile 的评论，对每一条进行分类，并采取动作：

- **Valid issues** 会被加入关键发现，并在发版前修复
- **Already-fixed issues** 会自动回复，确认它抓得对
- **False positives** 会被反驳——由你确认，然后发出回复解释为什么它是错的

结果就是形成了一个双层评审体系：Greptile 在 PR 上异步抓问题，然后 `/review` 和 `/ship` 再把这些发现纳入正常工作流中完成分诊。没有任何问题会从缝隙里漏掉。

### 从历史中学习

每一个你确认的 false positive，都会被保存到 `~/.gstack/greptile-history.md`。后续运行时，会自动跳过你的代码库中已知的 FP 模式。而且 `/retro` 还会持续跟踪 Greptile 的命中率，这样你就能看到它的信噪比是否在改善。

### 示例

```
You:   /ship

Claude: [syncs main, runs tests, pre-landing review...]

        Greptile found 3 comments on this PR:

        [VALID] app/services/payment_service.rb:47 — Race condition:
        concurrent charges can double-debit. Recommend DB-level advisory
        lock around the charge block.

        [ALREADY FIXED] app/models/user.rb:12 — Missing null check
        on email before downcase.
        → Already fixed in commit abc1234. Auto-replying.

        [FALSE POSITIVE] lib/auth.rb:88 — "Token comparison should
        use constant-time comparison." We already use
        ActiveSupport::SecurityUtils.secure_compare here.

        Fixing the race condition in payment_service.rb...

        The auth token comment is a false positive.
        Options: A) Reply to Greptile explaining  B) Fix anyway  C) Ignore

You:    A

Claude: Replied to Greptile. All tests pass.
        PR: github.com/you/app/pull/42
```

三条 Greptile 评论。一个真实修复。一个自动确认。一个假阳性被回退并附带解释回复。总共额外花费时间：大约 30 秒。