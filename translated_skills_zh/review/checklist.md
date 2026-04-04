# 预落地审查清单

## 说明

针对下面列出的问题审查 `git diff origin/main` 的输出。要具体，引用 `file:line` 并建议修复方案。跳过没问题的内容。只标记真实问题。

**两轮审查：**
- **第 1 轮（CRITICAL）：** 先执行 SQL 与数据安全，以及 LLM 输出信任边界。严重级别最高。
- **第 2 轮（INFORMATIONAL）：** 执行其余所有类别。严重级别较低，但仍需处理。

所有发现的问题都通过 Fix-First Review 处理：明显的机械性修复会自动应用，
真正存在歧义的问题会合并成一个向用户提出的问题。

**输出格式：**

```
Pre-Landing Review: N issues (X critical, Y informational)

**AUTO-FIXED:**
- [file:line] 问题 → 已应用修复

**NEEDS INPUT:**
- [file:line] 问题描述
  Recommended fix: 建议的修复方案
```

如果未发现问题：`Pre-Landing Review: No issues found.`

保持简洁。每个问题：一行描述问题，一行给出修复方案。不要前言，不要总结，不要写“整体看起来不错”。

---

## 审查类别

### 第 1 轮 — CRITICAL

#### SQL 与数据安全
- SQL 中的字符串插值（即使值经过 `.to_i`/`.to_f` 处理也不行，必须使用参数化查询（Rails: sanitize_sql_array/Arel；Node: prepared statements；Python: parameterized queries））
- TOCTOU 竞态：先检查再设置的模式，应改为原子性的 `WHERE` + `update_all`
- 通过直接写数据库绕过模型校验（Rails: update_column；Django: QuerySet.update()；Prisma: raw queries）
- N+1 查询：对循环/视图中使用的关联缺少预加载（Rails: .includes()；SQLAlchemy: joinedload()；Prisma: include）

#### 竞态条件与并发
- 读取-检查-写入流程没有唯一性约束，或没有捕获重复键错误并重试（例如 `where(hash:).first` 然后 `save!`，却没有处理并发插入）
- find-or-create 没有唯一数据库索引，并发调用可能创建重复记录
- 状态转换没有使用原子性的 `WHERE old_status = ? UPDATE SET new_status`，并发更新可能跳过转换或重复应用转换
- 对用户可控数据进行不安全的 HTML 渲染（Rails: .html_safe/raw()；React: dangerouslySetInnerHTML；Vue: v-html；Django: |safe/mark_safe）（XSS）

#### LLM 输出信任边界
- 将 LLM 生成的值（邮箱、URL、姓名）写入数据库或传给 mailer 之前未做格式校验。持久化前要添加轻量校验（`EMAIL_REGEXP`、`URI.parse`、`.strip`）。
- 接受结构化工具输出（数组、哈希）后，在写入数据库前没有做类型/结构校验。

#### 枚举与取值完整性
当 diff 引入新的枚举值、状态字符串、层级名称或类型常量时：
- **追踪到每一个使用方。** 读取（不只是 grep，是真的要 READ）每个对该值做分支、过滤或展示的文件。如果任何使用方没有处理新值，就要标记。常见遗漏：前端下拉框新增了一个值，但后端模型/计算方法没有持久化它。
- **检查 allowlist/过滤数组。** 搜索包含同级值的数组或 `%w[]` 列表（例如，如果给 tiers 增加 `"revise"`，就要找到每个 `%w[quick lfg mega]` 并确认在需要的地方包含了 `"revise"`）。
- **检查 `case`/`if-elsif` 链。** 如果现有代码对该枚举有分支，新值是否会落入错误的默认分支？
要做到这一点：使用 Grep 查找所有对同级值的引用（例如，grep `"lfg"` 或 `"mega"`，以找到所有 tier 使用方）。读取每一处匹配。此步骤要求读取 diff **之外**的代码。

### 第 2 轮 — INFORMATIONAL

#### 条件性副作用
- 代码路径基于某个条件分支，但某个分支忘记应用副作用。示例：条目被提升为 verified，但只有在次级条件为真时才附加 URL，另一条分支会在没有 URL 的情况下完成提升，导致记录不一致。
- 日志声称某个动作已发生，但该动作实际上因条件判断被跳过。日志应反映真实发生的情况。

#### 魔法数字与字符串耦合
- 多个文件中使用裸数字字面量，应提取为统一说明的命名常量
- 错误消息字符串在别处被用作查询过滤条件（grep 这个字符串，是否有地方在依赖它匹配？）

#### 死代码与一致性
- 变量被赋值但从未读取
- PR 标题与 VERSION/CHANGELOG 文件之间版本不一致
- CHANGELOG 条目对变更描述不准确（例如写着“从 X 改为 Y”，但 X 实际上从未存在）
- 代码变更后，注释/docstring 仍描述旧行为

#### LLM Prompt 问题
- prompt 中使用从 0 开始编号的列表（LLM 通常稳定返回从 1 开始编号）
- prompt 文本中列出的可用工具/能力，与 `tool_classes`/`tools` 数组里实际接入的不一致
- 在多个位置声明 word/token 限制，可能发生漂移

#### 测试缺口
- 负路径测试只断言类型/状态，却没有断言副作用（附加了 URL 吗？字段填充了吗？callback 触发了吗？）
- 对字符串内容做断言，却没有检查格式（例如断言 title 存在，但未检查 URL 格式）
- 当某个代码路径本应明确 **不** 调用外部服务时，缺少 `.expects(:something).never`
- 安全强制功能（拦截、限流、鉴权）缺少集成测试来验证强制路径端到端生效

#### 完整性缺口
- 使用了捷径式实现，而完整实现的 CC 时间成本其实 <30 分钟（例如，枚举处理不完整、错误路径不完整、缺少但很容易补上的边界情况）
- 给出的选项只有人工团队工作量估算，应同时展示人工和 CC+gstack 时间
- 测试覆盖缺口属于“lake”而不是“ocean”（例如缺少负路径测试、缺少可直接参照 happy-path 结构补上的边界测试）
- 功能只实现了 80-90%，而通过适度增加代码即可达到 100%

#### 加密与熵
- 截断数据而不是哈希（取最后 N 个字符而不是 SHA-256），熵更低，更容易碰撞
- 对安全敏感值使用 `rand()` / `Random.rand`，应改用 `SecureRandom`
- 对 secret 或 token 使用非常量时间比较（`==`），容易受到计时攻击

#### 时间窗口安全
- 使用日期键查询时，假设“今天”覆盖完整 24 小时；例如 PT 上午 8 点出报表时，`today` 键下实际上只能看到 0 点到 8 点的数据
- 相关功能之间时间窗口不一致；一个使用按小时分桶，另一个对同一份数据使用按天的 key

#### 边界处的类型强制转换
- 值跨越 Ruby→JSON→JS 边界时，类型可能变化（数字 vs 字符串）；哈希/摘要输入必须归一化类型
- 哈希/摘要输入在序列化前没有调用 `.to_s` 或等价方法；`{ cores: 8 }` 和 `{ cores: "8" }` 会产生不同哈希

#### 视图/前端
- partial 中内联 `<style>` 块（每次渲染都会重新解析）
- 视图中存在 O(n*m) 查找（循环中用 `Array#find`，而不是 `index_by` 哈希）
- 在数据库结果上使用 Ruby 侧 `.select{}` 过滤，而本可写成 `WHERE` 子句（除非是有意避免前导通配符 `LIKE`）

#### 性能与打包体积影响
- package.json 中新增了已知较重的 `dependencies`：moment.js（→ date-fns，330KB→22KB）、完整 lodash（→ lodash-es 或按函数导入）、jquery、完整 core-js polyfill
- lockfile 显著膨胀（一次新增引入了很多新的传递依赖）
- 新增图片没有 `loading="lazy"` 或明确的 width/height 属性（会造成布局偏移 / CLS）
- 向仓库提交了大型静态资源（单文件 >500KB）
- 同步 `<script>` 标签缺少 async/defer
- 样式表中使用 CSS `@import`（会阻塞并行加载，应改用 bundler imports）
- `useEffect` 中的 fetch 依赖另一个 fetch 的结果（请求瀑布，应合并或并行化）
- 在可 tree-shake 的库上把 named import 改成 default import（会破坏 tree-shaking）
- 在 ESM 代码库中新增 `require()` 调用

**不要标记：**
- 新增 `devDependencies`（不会影响生产包体积）
- 动态 `import()` 调用（代码拆分，这是好事）
- 小型工具依赖新增（gzip 后 <5KB）
- 仅服务端使用的依赖

---

## 严重级别分类

```
CRITICAL（最高严重级别）：        INFORMATIONAL（较低严重级别）：
├─ SQL & Data Safety              ├─ Conditional Side Effects
├─ Race Conditions & Concurrency  ├─ Magic Numbers & String Coupling
├─ LLM Output Trust Boundary      ├─ Dead Code & Consistency
└─ Enum & Value Completeness      ├─ LLM Prompt Issues
                                   ├─ Test Gaps
                                   ├─ Completeness Gaps
                                   ├─ Crypto & Entropy
                                   ├─ Time Window Safety
                                   ├─ Type Coercion at Boundaries
                                   ├─ View/Frontend
                                   └─ Performance & Bundle Impact

所有发现的问题都通过 Fix-First Review 处理。严重级别决定
展示顺序，以及归类为 AUTO-FIX 还是 ASK；critical
发现更倾向于 ASK（风险更高），informational 发现
更倾向于 AUTO-FIX（更偏机械性）。
```

---

## Fix-First Heuristic

这个启发式规则同时被 `/review` 和 `/ship` 引用。它用于判断
agent 是自动修复某个发现的问题，还是询问用户。

```
AUTO-FIX（agent 无需询问直接修复）：   ASK（需要人工判断）：
├─ Dead code / unused variables       ├─ Security（auth、XSS、injection）
├─ N+1 queries（缺少预加载）           ├─ Race conditions
├─ 与代码矛盾的过时注释                ├─ Design decisions
├─ Magic numbers → named constants    ├─ Large fixes（>20 行）
├─ 缺少 LLM 输出校验                  ├─ Enum completeness
├─ 版本/路径不匹配                     ├─ 移除功能
├─ 变量被赋值但从未读取                └─ 任何会改变用户可见
└─ Inline styles、O(n*m) 视图查找         行为的修改
```

**经验法则：** 如果修复是机械性的，而且高级工程师会直接修改
而无需讨论，那就是 AUTO-FIX。如果合理的工程师之间可能会对
修复方案有分歧，那就是 ASK。

**Critical 发现默认更倾向于 ASK**（它们天然风险更高）。
**Informational 发现默认更倾向于 AUTO-FIX**（它们更偏机械性）。

---

## Suppressions — 不要标记这些

- “X 与 Y 重复”这一类问题，如果这种重复无害且有助于可读性（例如，`present?` 与 `length > 20` 重复）
- “添加注释解释为什么选择这个阈值/常量”这一类建议；阈值会在调优中变化，注释容易过时
- “这个断言本可以更严格”，如果该断言已经覆盖了行为
- 仅为了风格一致性而建议修改（例如，把某个值包进条件判断里，只是为了和另一个常量的保护方式一致）
- “Regex 没有处理边界情况 X”，如果输入本身受限，而 X 在实际中不会出现
- “测试同时覆盖了多个 guard”——这没有问题，测试不需要把每个 guard 都隔离开
- Eval 阈值变更（max_actionable、min scores）——这些是基于经验调优的，而且经常变化
- 无害的空操作（例如，对一个永远不在数组中的元素调用 `.reject`）
- 你正在审查的 diff 中**已经处理过的任何内容**——评论前先完整阅读 FULL diff