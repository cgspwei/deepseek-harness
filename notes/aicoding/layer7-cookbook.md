# 第 7 层详解：cookbook —— 「这类代码具体怎么写」

> 总览见 [how-deepseek-does-ai-coding.md](how-deepseek-does-ai-coding.md)。本文是第 7 层的完整拆解。
>
> **一句话**：cookbook 是把「仓库里加一类东西的标准路径」从口耳相传变成**带编号验证步骤的可执行文档**，让 AI 和新人都不必靠读旧代码猜惯例。

---

## 0. 为什么这一层单独存在

前六层各自回答了一个问题，但**都没有回答最日常的那个**：

| 层 | 回答的问题 | 没回答 |
|---|---|---|
| ① AGENTS.md | 这个仓库的规矩是什么 | 具体怎么落地 |
| ② skills | review/验证/翻译/归档**怎么做** | 业务代码怎么写 |
| ③ notes | 为什么这么定 | 定完之后怎么执行 |
| ④ scripts | 机器会查什么 | 怎么才能查得过 |
| ⑤ hooks/CI | 什么时候查 | 同上 |
| ⑥ review | 人/AI 最后看什么 | 同上 |
| **⑦ cookbook** | **「我现在要加一个工具，第一步干什么？」** | — |

这类知识的特征是：**量大、易变、且最容易只活在老员工脑子里**。在 agent-first 的仓库里，「只活在脑子里」等于「AI 拿不到」，等于**每个 agent 都要重新从旧代码里逆向推导惯例，且每次推导结果都可能不同**。

cookbook 是这个问题的答案，而且它的解法和第 2 层是同一个思路：**常驻的极简（AGENTS.md 一行）+ 按需的极详（cookbook 一千七百词）**。

---

## 1. 全景：9 份手册，约 9800 词

`docs/cookbook/` 下的英文正文（每份都有 `.zh.md` 对照，受 i18n 配对门禁约束）：

| cookbook | 词数 | 代码块 | 链接的 Agent Note | 教什么 |
|---|---|---|---|---|
| `adding-a-tool.md` | 1710 | 2 | **3** | 加一个模型可调用的工具 |
| `adding-a-package.md` | 1674 | 8 | 1 | 加一个 workspace 包 |
| `adding-a-conversation-node.md` | 1642 | 2 | 1 | 加一个 Web 会话节点 |
| `extension-cookbook.md` | 1399 | 6 | 2 | 各扩展点的插件形态 |
| `maintaining-dsh-code-review.md` | 1051 | 0 | 1 | **维护 review skill 本身** |
| `adding-a-settings-card.md` | 756 | 8 | 0 | 加一张设置卡片 |
| `responding-to-pr-review-on-a-stack.md` | 582 | 0 | 1 | 在 PR 栈上响应 review |
| `adding-an-llm-adapter.md` | 512 | 2 | 0 | 加一个 LLM 适配器 |
| `adding-a-vendored-package.md` | 512 | 6 | 1 | vendor 一个上游包 |
| **合计** | **9838** | **34** | **11** | |

三个立刻能读出来的事实：

1. **是 AGENTS.md 的 5 倍**（9838 vs 1929），且**没有词数预算**（`scripts/doc-budgets.manifest.json` 里搜不到 cookbook）。
2. **34 个代码块**——不是描述性文档，是可抄的模板。
3. **每份都链接 Agent Note**（9 份里 7 份有，共 11 处）——**理由不写在这里，写在第 3 层**。

---

## 2. 职责边界：被 `docs/AGENTS.md` 钉死

cookbook 不是「什么都能放的杂物间」。`docs/AGENTS.md` 的 tier 表里有一行专门管它：

| Tier | Job | **Does NOT belong there** |
|---|---|---|
| `cookbook/` | Step-by-step how-tos with numbered verify steps | **Design rationale**（→ the Agent Note each guide links） |

**两个关键词**：

- **`numbered verify steps`（带编号的验证步骤）**——这是 cookbook 的形式要求，不是「大概讲讲」。
- **`Design rationale` 被明确排除**——理由归 Agent Note。**同一个知识只有一个家**。

放在全局分工里看（`docs/AGENTS.md` 原文）：

> **Placement: bugs → postmortems; rationale → Agent Notes; procedures → cookbooks; type definitions → subsystems; package contracts → READMEs; standing orders → root `AGENTS.md` with a rationale link.**

一句话映射：

```
出了事故     → postmortem
为什么这么定 → Agent Note
怎么做       → cookbook        ← 第 7 层
类型定义     → subsystems
包的契约     → package README
常驻铁律     → root AGENTS.md
```

> **这个「one home per fact」纪律是 cookbook 能长期不腐烂的原因。** 反面案例大家都见过：一份 wiki 里既有操作步骤、又有历史沿革、又有争论记录，三个月后没人敢删任何一段，因为不知道哪段还有人依赖。这里靠 tier 表把每类内容强制送进唯一的家。

---

## 3. 最能打的一个点：cookbook 会被提升成 AGENTS.md 里的硬规则

这是第 7 层与第 1 层之间的接缝，也是整层最值得讲的设计。

AGENTS.md 的 31 条 Conventions 里有这么一条：

> **A tool's UI render intent is part of its design, decided up front** (`generic`/`terminal`/`diff`, `locations`); presentation methods are pure functions of `args`（[cookbook](../../docs/cookbook/adding-a-tool.md)）

拆开看这条做了三件事：

| 部分 | 内容 | 住在哪 | 占预算吗 |
|---|---|---|---|
| **铁律** | UI render intent 必须**事先决定** | AGENTS.md（1 行） | 占（1950 词额度内） |
| **选项** | `generic` / `terminal` / `diff` / `search` / `web` 各自什么时候用 | cookbook（约 400 词） | **不占** |
| **理由** | 为什么设计成一个 render intent union | Agent Note `2026-07-02-tool-render-intent-union.md` | 不占 |

**「decided up front」（事先决定）就是 spec-first 落到代码层的具体形态**：加工具时，UI 渲染意图必须在写实现之前定下来，不能写完再补。

> **可直接迁移的模式**：
>
> **在 AGENTS.md 里放一句「加 X 必须事先决定 Y」（1 行，占预算）；在 cookbook 里放「Y 怎么决定、有哪几种选项、怎么验证」（一千词，不占预算）；在 Agent Note 里放「为什么是这几种选项」（不占预算）。**
>
> 这是第 2 层「两级 context 预算」的同一个套路在第 7 层的复用：**常驻的必须极简，按需的可以极详**。

同类接缝还有一处，在 `packages/AGENTS.md`：

```
[Naming rules](../docs/cookbook/adding-a-package.md#name-the-role-that-exists)
- Package READMEs document model token and KV-cache effects using the
  [canonical Model Experience format](../docs/cookbook/adding-a-package.md#4-write-the-package-readme)
```

**注意它链接的是 `#锚点`，不是整份文档**——规则指向 cookbook 里的**确切小节**，而 `verify-md-links` 会校验死锚点（第 4 层）。所以这个接缝本身是机器维护的：cookbook 里那个小节被改名，CI 立刻红。

---

## 4. 逐份拆解：四份最有信息量的

### A. `adding-a-tool.md`（1710 词）—— 契约清单，不是教程

它开头就把自己和教程分开：

> Reference for the **contracts** a model-facing tool must satisfy. For an ordered first tool, follow [Build a tool](../user/develop/basic/tool.md).

**两份文档，两个读者**：`docs/user/develop/basic/tool.md` 是「第一次写工具的人」的有序教程，cookbook 是「知道怎么写、需要核对契约」的参考清单。这是 `docs/AGENTS.md` 里「tutorial vs reference」二分法的实际落地：

> Classify every in-scope document as a **tutorial** or **reference**. Tutorials follow an ordered path to an outcome... References define a lookup scope and current behavior **without a teaching sequence**.

它的 `execute()` 契约有 7 条，每条都是**「不这么做会怎样」**而不是「建议这么做」。摘几条最有代表性的：

**① 参数已经帮你校验过了，但没校验完**

> Args are validated for you... **You still hand-check constraints the DSL does not express**, such as non-empty strings, positive numbers, or cross-field rules.

这条精确划出了「框架管到哪、你从哪接手」。**AI 最容易在这种边界上出错**——要么重复校验（第 1 层有条规则专门禁止：`Trust TypeScript at typed same-process boundaries`），要么漏校验。

**② 只返回一个规范 JSON 值，不要返回内容块**

> **Declare and return one canonical JSON value.** ... `execute` returns only the inferred value... **Do not return content blocks from the body or make callers parse prose for ids and fields.**

「不要让调用方从人话里解析 id」——这条在后面 Code Mode 一节又强化了一次：

> A successful background branch returns a typed canonical handle such as `{ kind: 'background', jobId }`; its Native renderer may keep human prose such as `started background job bash-1`, but **Code Mode must never parse that prose to recover the id**.

**③ 抛异常 = `isError`，但「域内失败」不许抛**

> Throw for infrastructure failures. **Represent a successful domain outcome in the canonical value even when its Native renderer explains a non-ideal state, such as a non-zero process exit.**

进程退出码非 0 **不是**异常——它是一次成功的执行，只是结果不理想。这个区分极其容易搞错，而且搞错了不会立刻报错，只会让上层的错误处理逻辑逐渐腐烂。

#### 这份文档里最硬的一段：presenter 必须是纯函数

> **Purity.** These run on live streaming AND on session-log **REPLAY**, so they must be pure functions of `args` (+ the result) — NO I/O, NO reading session state, NO clock/random.

然后紧跟一句**直接对写代码的人（或 AI）说话**的话：

> **If you find yourself wanting the file's old content or the working directory inside `presentCall`, stop** — that belongs in durable result metadata or the adapter, not the presenter.

> **这句「If you find yourself wanting X, stop」是 cookbook 独有的语气**——它预判了实现者会在哪里走上歧路，并在那个岔路口立牌子。这比事后 review 抓住它便宜得多。**写 cookbook 的最高性价比就在这里：把 review 里反复提的同一个意见，前移成一句预警。**

同一节还有一条把「UI 方便」和「模型正确」彻底切开的规则：

> **UI-only formatting stays out of the model result.** A fenced ` ```console ` block, a diff, a relativized path—none of these belongs in the canonical value or Native content **merely to serve a UI**.

以及一条防止 replay 崩溃的软失败设计：

> **`defineTool` soft-validates the display path.** Malformed or older logged arguments make the wrapper return `undefined` (a generic fallback) rather than throw — **display must never crash a replay**。

### B. `adding-a-package.md`（1674 词）—— 词汇表本身就是规范

这份是标准的 5 步 checklist（建包 → 注册根配置 → 决定拓扑 → 写 README → 验证），但真正有信息量的是**第 3 步里的角色词表**。

它先立规则：

> **Name the role that exists.** Name the stable current responsibility. **Do not name the first implementation, a possible future expansion, or the Cordis base class.**

然后给出一张 **19 行的词表**，每个词都有「什么时候用 / 什么时候不许用」：

| Word | Use it when | Do not use it when（摘要） |
|---|---|---|
| `Controller` | 接受命令/用户意图，改**一个**既有状态 | 执行任意工作、拥有 provider 群 |
| `Store` | 拥有一份数据，主要提供 CRUD/快照/订阅 | 校验状态机、仲裁权限、分派工作。**「有个 map 不代表它是 store」** |
| `Registry` | 拥有一组动态命名注册（含查找、重名、生命周期、释放） | 主要契约是分派/执行/取消/策略/编排 |
| `Runtime` | 跑活的工作，拥有分派/取消/provider 协调 | 只存记录、只返回目录、只解析一个值 |
| `Resolver` | 从输入算出/定位**一个**答案，不拥有其生命周期 | 拥有可变集合或长跑执行 |
| `Presenter` | 从域值到 render intent 的**纯转换** | 有 I/O、订阅、改状态、管生命周期 |
| `Policy` | 决定什么被允许/选中/限制/观察 | 执行该决定所许可的机制 |
| `Service` | 上面所有更锐利的角色词都不诚实时的兜底 | **「这个名字只是因为类继承了 Cordis `Service`」** |

还有单复数规则：

> Use a **singular** `ctx` key for one engine, runtime, policy, controller, resolver, store, or current configuration. Use a **plural** key for a registry or a service that owns multiple named members. **The class role and key number must agree.**

> **为什么这个词表值得单独讲**：命名是 AI 最容易「看起来对、实际上错」的地方——它会写出 `TaskManager`、`DataHelper`、`XxxService` 这类什么都没说的名字。这张表把命名从审美问题变成了**查表问题**：先说清这个类做什么，再从表里找唯一匹配的词。**而且它是 `packages/AGENTS.md` 直接链接的锚点**，属于被规则引用的强制内容，不是建议。
>
> 尤其那句 `The name exists only because the class extends Cordis Service` ——**这是把一个具体的偷懒模式点名禁止**，比「请起有意义的名字」有用一百倍。

第 4 步「写 README」则给出了一段**逐字模板**（`Model Experience` → `What the model sees` → `Token effect` → `KV Cache effect`），并且有配套门禁：

```
scripts/verify-package-readme-model-experience.ts   # 校验必需小节结构
scripts/verify-package-readme-limitations.ts        # 校验限制小节（含白名单）
```

**cookbook 给格式，scripts 强制格式，Agent Note 记理由**——三层各司其职，这是整套体系里最典型的一次三方配合。

### C. `adding-a-conversation-node.md`（1642 词）—— 用性能契约约束实现

这份是唯一明确自称 tutorial 的（`This tutorial adds one business-owned row...`），5 个编号步骤，含一个近百行的完整可编译示例。

它最特别的地方是**用一条性能不变量来约束实现自由度**：

> With `D` registered Definitions, one incoming event performs `D` current-event matches and **constant-time** Context-key lookup after a match. **Definition code must preserve that property: do not traverse the complete event window, every Context, `context.matches`, or the rendered Node collection on the normal append path.**

然后给出替代方案（**不是只说「不许」**）：

> Use **State** for accumulated facts, **Location data** for same-Turn/Step sharing, and **`reader.previous()`** for indexed predecessor dependencies.

> **「禁止 X，改用 Y/Z/W」是 cookbook 里反复出现的句式**。只写「禁止」的规则，AI 会绕出更糟的写法；给出替代路径，才真正关闭了歧路。这一点和第 2 层 `dsh-trim-cot-leakage` 明确列出「什么不算泄漏」是同一种设计意识：**边界要双向封闭**。

它的第 5 步「Verify」是 9 份里最典型的**编号验证步骤**范例——6 条,每条都是可写成测试的断言：

```
1. 完整窗口 replace → 期望的最终 State / Location data / Node payload / anchorSeq
2. 只有 update 的尾部保持 pending；前插唯一 start 后结果等同完整 replace
3. 初始历史 + 实时 append 结果等同重放合并窗口
4. 前插更旧的页只加行，不替换 data 未变的既有 keyed Node
5. 重复可见增量保持 context.key，每动画帧最多发布一次
6. keyed renderer 只消费 node.data 和受约束的 Location hooks
```

**注意第 2、3 条都是「两条路径结果必须相同」的等价性断言**——这是重放系统正确性的核心，也是最容易被漏测的地方。cookbook 直接把它列成必测项。

### D. `maintaining-dsh-code-review.md`（1051 词）—— 自指的那一份

**一份「怎么维护 review skill」的 cookbook。** 这是整层最值得单独讲的一份，因为它是第 6 层那条回边的**操作面**。

它先声明读者是谁——**包括「想理解为什么 skill 更新总是小步周期 PR」的普通贡献者**：

> This cookbook is the entry point for that operator — and for anyone taking over the role — and for **repo contributors who want to understand why skill updates arrive as small periodic PRs rather than one-off audits**.

工作流本身（每天跑，两个 UTC 日重叠窗口）：

```
1. 选窗口内 merge 且 merge commit 可从 origin/master 到达的 PR
   （不可达的 / 超 250 commit 的记入 skipped-pulls.json，跳过而不中止整轮）
2. 收集 merge 前的人类 review 反馈（带 commit 锚点），
   对比「反馈时」与「最终落地」的 patch
3. 两个独立配置的 reviewer adapter 分类：谁写的、是否被采纳、
   相对当前 skill 是否已覆盖
4. 主 adapter 起草完整的新 SKILL.md；两个 adapter review 同一个 diff；
   有 blocking finding 就循环，直到两个都通过
5. 跑 doc-sync + lint，才宣告成功
```

**这是「AI review AI」在流程层面的完整实现**：不是一个模型自己改自己的规则，而是**两个独立配置的适配器互相把关，最后由人做终审**。

而且有几条设计极其克制，值得逐条讲：

**① 明确禁止「reviewer 批过了所以我信」**

> **Read the diff on its own merits. Do not defer to "the reviewers approved"**; the maintainer contract is that the operator makes the final decision.

要找什么也说清了：**checklist 膨胀、历史叙述、从单一事件过度外推、与现有内容重复覆盖**。这四项正好是 skill 文档腐烂的四种典型方式。

**② 抽查证据链是必需动作**

> Spot-check at least one candidate: **does the linked human comment actually support the added rule? Does the linked PR actually adopt it?**

manifest 里记录了「每条提议规则 → 源反馈 → 落地证据」的映射，所以这个抽查是可执行的，不是「凭感觉看看」。

**③ 不许原样提交 AI 输出**

> **Do not commit adapter output verbatim.** Small edits during promotion — tightening wording, removing an example that only makes sense with the source PR's context, folding a rule into an existing one — are **expected** and preserve the "reviewer judgment" the workflow depends on.

**④ 「今天没产出候选」是正确行为，不是卡住**

> That is the common case... The tool records "no candidate" in its daily log, **sends no notification (to avoid alert fatigue)**, and moves on. **Days without a skill update are the workflow behaving correctly, not a stall.**

**⑤ 两个 adapter 不许是同一个可执行文件**

> The tool **refuses to run when the two reviewer commands resolve to byte-identical executables**.

**这条是整份文档里最锐利的一句**——它把「双 reviewer 独立性」从「我们注意一下」变成了**启动时的硬校验**。如果两个 adapter 是同一个模型，「互相 review」就是自我确认剧场。

**⑥ 全 provider 故障不许退化成「无候选」**

> If either adapter produces no valid result for any nonempty batch in an operation, the run **fails**, writes a failure record, and notifies the operator; **it never collapses a total-provider outage into "no candidate."**

**「静默失效不许伪装成正常」**——和第 2 层 `verify-skill-invocation-metadata.ts` 防的是同一类病（某个东西悄悄不工作了，但没人收到信号）。

**⑦ 单点维护者的 bus factor 被写进 Risks，且交接必须留决策记录**

> **Do not silently transfer the tool** — the "single-maintainer bus factor" in the Agent Note's Risks section is the reason the handoff needs a documented decision.

> **这份 cookbook 的存在本身就是论据**：总览里说「review 的漏网之鱼回流成新门禁」——听起来像一句漂亮的话。而这份文档证明**那条回边有配套的日常机制、有工件目录、有双 adapter 交叉验证、有人类终审、有故障处理策略**。**闭环不是口号，是有 SOP 的。**

---

## 5. 剩下五份的一句话价值

| cookbook | 最值得抄的一点 |
|---|---|
| `extension-cookbook.md` | 结尾一张 **「产品功能 → 插件机制」全映射表**（约 30 行），把「一切皆插件」这个架构主张变成**可逐行核对的清单**，并附一句 `No row modifies the loop`。这是架构声明的可验证形式。 |
| `adding-an-llm-adapter.md` | 「**两个实现共同验证过的协议义务**」——`usage` 必须在 `finish` 之前、`finish` 之后不许再发、tool-call `arguments` 全程是原始 JSON 字符串、不支持的字段必须抛 `UNSUPPORTED` **而不是静默丢弃**。**「静默丢弃」是 AI 写适配器时最常见的错。** |
| `adding-a-settings-card.md` | 开头一句 `Nothing in this path needs a change inside this repository`——**插件加一张设置卡片不需要改仓库任何地方**。这是插件化架构是否真成立的试金石，而 cookbook 把它当作事实陈述写出来。 |
| `adding-a-vendored-package.md` | 有个 pre-commit 钩子 `check-vendor-manifest.sh`：**staged 了 `vendor/*/src` 却没 staged `vendor/README.md` 就直接失败**。第 5 层在第 7 层的落点，防的是「改了 vendor 源码忘记记 manifest」。 |
| `responding-to-pr-review-on-a-stack.md` | 见下节，它有一条独立成段的价值。 |

### `responding-to-pr-review-on-a-stack.md` 里的两条硬规则

**① 修在「引入问题的那个 PR」上，然后往上传播**

> A fix lands on the PR that **INTRODUCED** the issue, then flows up-stack. When a comment on PR `B` points at code `B` introduced, fix it on `B` and propagate `B` into `C` — even if `C` also carries the file. **Originating the fix downstream leaves `B` shipping the unfixed code and hides the fix from `B`'s reviewer.**

**② 委派出去的修复要「信任但验证」，且回归守卫必须证明它会红**

> Treat delegated fixes as trust-but-verify: **a sub-agent's report describes intent, not necessarily what landed.** Re-run the gates yourself on the actual tree, and for a regression guard, **prove it FAILS on the unfixed code** (introduce the regression, watch red, revert) — **a guard that passes both ways guards nothing**. **A sub-agent that reframes a problem as already handled is a signal to dig in personally.**

> **这段是整个仓库里对「AI 协作」最清醒的一段话**，三层意思：
> 1. **子 agent 的报告描述意图，不必然是落地事实**——所以自己重跑门禁。
> 2. **回归测试必须先证明它在未修复代码上会失败**——「两边都绿的守卫什么都没守」。这条是对付「AI 写了个永远通过的测试」的唯一有效手段。
> 3. **子 agent 把问题重新表述成「已经处理了」是一个危险信号**，应当亲自去查。这精确命中了 AI 最典型的失败模式：不是报错，而是**说服你问题不存在**。

---

## 6. cookbook 与 skills 的分界（为什么不合并）

两者都是「一千词量级的操作手册」，很容易被当成一回事。但它们的性质不同，混在一起讲会把第 2 层「两级 context 预算」那个最有力的点讲糊。

| 维度 | skills（第 2 层） | cookbook（第 7 层） |
|---|---|---|
| **内容** | **元操作**：review、选验证集、归档 note、翻译、清理 CoT、找简化 | **业务代码怎么写**：加工具、加包、加适配器、加会话节点 |
| **加载方式** | 模型读到 `description` 后判断触发，用 `skill` 工具按需读入正文 | **普通 Markdown**，被 AGENTS.md / `packages/AGENTS.md` / README 链接后主动去读 |
| **常驻 context** | 只有 `name + description`（几百词目录） | 完全不常驻，靠链接被找到 |
| **主要读者** | 主要是 AI | **人和 AI 都读**（新人上手 + agent 执行） |
| **词数预算** | 无 | 无 |
| **数量** | 11 | 9 |
| **正文规模** | 13578 词 | 9838 词 |
| **变更频率** | 低（流程稳定） | **跟着架构走**（架构变则手册变） |
| **门禁** | `verify-skill-invocation-metadata.ts` 校验调用元数据 | `verify-md-links` 校验锚点、i18n 配对门禁校验中英一致 |

**一句话区分**：

> **skills 管「怎么做事」，cookbook 管「怎么写这类代码」。**

**为什么不能合并的技术原因**：skill 的价值建立在「模型能通过 description 精准判断该不该读」之上。而「加一个工具」这种任务的触发信号是**用户直接说出来的意图**，不需要模型猜——它需要的是「在写代码时手边有一份可查的契约清单」，这正是普通文档 + 明确链接的用法。**把 cookbook 塞进 skills，等于给 11 个 description 再加 9 个，稀释了整个触发机制的准确率**，而收益是零。

---

## 7. 这一层的门禁（第 4、5 层在这里的落点）

cookbook 没有专属的 `verify-cookbook-*.ts`，但它被三类通用门禁覆盖：

| 门禁 | 查什么 | 为什么对 cookbook 关键 |
|---|---|---|
| `verify-md-links` | 相对路径目标存在、`#fragment` 锚点不死 | **`packages/AGENTS.md` 链接的是 `#name-the-role-that-exists` 这类锚点**——小节改名立刻红，接缝由机器维护 |
| i18n 配对门禁（`verify-translation-pairing`） | 9 份 `.md` 与 9 份 `.zh.md` 结构逐节对应 | 中文读者拿到的手册不会落后于英文 |
| `doc-typecheck` | ` ```ts ` 代码块必须能编译 | **34 个代码块里的模板不会腐烂**——API 改了，cookbook 里的示例编译不过 |

第三条最值得强调。cookbook 里标了 `ignore-check` 的代码块是少数（`adding-a-conversation-node.md` 的完整示例、`adding-an-llm-adapter.md` 的骨架等），**其余 `ts` 块都必须真的编译通过**。

> **这解决了技术文档最经典的腐烂方式：文档里的示例代码在某次重构后就不能跑了，但没人发现，直到有个新人（或 AI）照抄了它。** 把示例纳入 typecheck，等于让文档和代码共享同一个正确性门禁。

再加一条 pre-commit 钩子（第 5 层）：`check-vendor-manifest.sh` —— 这是 `adding-a-vendored-package.md` 第 3 步教的内容被机器强制的例子。**cookbook 教流程，hook 保证流程不被跳过。**

---

## 8. 一个反直觉的观察：9 份里有 3 份不教「写代码」

| cookbook | 教的其实是 |
|---|---|
| `maintaining-dsh-code-review.md` | **怎么维护规则本身** |
| `responding-to-pr-review-on-a-stack.md` | **怎么和 review 协作** |
| `adding-a-vendored-package.md` | 怎么管理外部依赖边界 |

**三分之一的「代码手册」在管流程和协作**。这和第 2 层「11 个 skill 里 6 个在管文字」是同一个现象的延续：

> **在 agent-first 的仓库里，「怎么协作」和「怎么写代码」一样需要被写成可执行文档。** 因为协作方里有一半不是人——AI 不会通过「跟着老员工干两周」学会你们团队的默契。**默契必须被显式化，否则它对 AI 不存在。**

---

## 9. 落到自己项目：五步

1. **先只写一份**，选团队里被问得最多的那个「怎么加 X」（大概率是「怎么加一个接口/一个模块/一个页面」）。
2. **形式上强制两点**：编号步骤 + 结尾一节可执行的「怎么验证」。没有验证步骤的手册会退化成散文。
3. **在 AGENTS.md 里加一行接缝**：「加 X 必须事先决定 Y（见 cookbook）」。**这一步是 cookbook 从「参考资料」变成「被引用的规范」的关键**——否则它就是一份没人读的 wiki。
4. **理由不要写进 cookbook**，写进 ADR/决策记录并从 cookbook 链过去。混进来之后这份文档就再也不敢删任何一段了。
5. **把代码示例纳入编译/lint**（哪怕只是一个能跑的示例文件被 include 进 tsconfig）。**示例不编译的文档，寿命只有一次重构。**

> **收益预期要说实话**：cookbook 不会立刻让 AI 写得更好，它的收益体现在**同一个 review 意见不再被反复提第三次**。所以最好的 cookbook 素材来源，就是你 review 时发现自己在重复说的话——**把它前移成一句「If you find yourself wanting X, stop」，比在第 100 个 PR 里再说一遍便宜得多**。

---

## 10. 一页总结

| 问题 | 答案 |
|---|---|
| 是什么 | `docs/cookbook/` 9 份分步手册，9838 词，34 个代码块 |
| 相对 AGENTS.md | **5 倍体量，零词数预算** |
| 职责 | 带编号验证步骤的 how-to；**设计理由被明确排除**（→ Agent Note） |
| 与第 1 层的接缝 | AGENTS.md 一行铁律「必须事先决定 X」+ cookbook 一千词「怎么决定」 |
| 与第 2 层的区别 | skills 管元操作（按需注入），cookbook 管业务代码（普通文档+链接） |
| 与第 4/5 层的接缝 | `verify-md-links` 保锚点、`doc-typecheck` 保示例、i18n 门禁保中英、`check-vendor-manifest.sh` 保流程 |
| 与第 6 层的接缝 | `maintaining-dsh-code-review.md` 是「review 漏网之鱼回流成规则」这条回边的 SOP |
| 最能打的一点 | **AGENTS.md 里 1 行占预算的铁律 + cookbook 里 1700 词不占预算的操作细节**——「常驻极简 + 按需极详」的第二次复用 |
| 最锐利的一句 | 双 reviewer adapter **字节相同就拒绝运行**——独立性从「注意一下」变成启动时硬校验 |
| 最有迁移价值的一句 | `a guard that passes both ways guards nothing`——回归测试必须先证明它在未修复代码上会红 |
