# DeepSeek Harness 如何做 AI Coding：七层全景图

> **备讲底稿**：结论 + 证据 + 弹药库，用来准备和答疑。**直接开口讲用 [speech-script.md](speech-script.md)**。
>
> 所有结论均有仓库内真实文件佐证，非归纳臆测。七层详解见 layer1–layer7。

**核心判断**：这套体系「每一层都可验证」，不存在靠自觉或信念的环节——这是它区别于「玄学 AGENTS.md」的本质。

**贯穿主线（如果只记一句话）**：

> **其余各层的全部意义，是把 code review 的负担压到少量 reviewer（含 AI）扛得住的程度。**

**第二条主线（回答「质量从哪来」）**：

> **他们不靠提示词让 AI 写好代码，而是让写不好的代码过不去。**

---

# 开场：一个「高产且高质量」的谜题

先抛一个现象作为引子：

**DeepSeek Harness 在极短时间里，产出了大量高质量、文档齐全的代码。** 靠什么做到的？

先放几个背景，帮大家校准预期——它**不是**「纯 AI 一键生成」的神话：

- **确实是重度 agent-first 开发**。开源当天，Prime Intellect 的 @eliebakouch 点出：这个 harness 是「heavily agent-first coded」，有一整个 `.agents/notes/` 文件夹记录 agents 留下的每个决策和提案，甚至包括「他们也用它做 post-training」。这个项目**自己 dogfooding 自己**。
- **团队有强系统工程背景**。负责人崔添翼（Jane Street 背景，量化交易系统经验）2026 年 3 月加入，团队约 5 月成型，8 月 13 日开源 v0.1（MIT）。量化系统那种「高可靠、可追踪、可回放」的思维被直接带进了代码库。
- **社区真正注意到的是「工程门禁」**。pre-commit / pre-merge / pre-push 链条做 lint、文档同步、类型检查——**把机器能查的尽量左移**。在 AI 产出极快时，人的 review 是瓶颈，这一点特别值钱。
- **官方没发过完整 retrospective**。没有「我们如何用 AI 写了这个项目」的复盘，但仓库本身就是证据：`AGENTS.md`、`.agents/notes/`、`scripts/` 已经把「agent-first + 严格决策/文档纪律」说清楚了。

## 关于「数字」：能说明什么，不能说明什么

讲高产就难免有人去数 commit。先把口径和边界说死，避免被数字带偏。

**公开侧可核对的量级**（数量级成立即可，不追求精确到天）：

| 维度 | 量级 | 口径 |
|---|---|---|
| 密集开发窗口 | 约 64 天 | 约 2026-06-10 → 2026-08-13（开源日） |
| commits | 约 1.2 万+ | 开源后仍在涨，现约 1.3 万级 |
| 日均 commits | 约百级 | 精确值以 `git shortlog` 为准 |
| 核心贡献者 | 约 8 人 | 小核心团队，非人海 |
| 头部贡献者 | 负责人一人 5000+ | 其后千级、再往后百级 |
| 代码量（TS/TSX） | 约 56.8 万行 | 其中测试约 30.7 万行，源码约 26 万行 |
| 文档（Markdown） | 约 17 万行 | 与代码量同量级，印证「文档齐全」 |
| Python | 约 4400 行 | 训练/评测侧辅助 |

**这些数字说明三件事**：

1. **极高频率，且带着文档/门禁/Note 的完整配套**——传统手写很难长期稳定在日均百级还保持配套完整，更符合「Agent 大量产小步提交 + 人做方向与合入」。
2. **小核心，不是人海**——极少贡献者 + 极高单人/agent 吞吐。
3. **和 Agent Notes / 门禁是配套的**——提交越碎越密，越需要决策记录和门禁兜底。**高频 commit 倒逼工程约束，而不是先有约束才偶尔用 AI。**

**对外话术边界（务必守住）**：

- **Commit 数 ≠ 有效产出的尺子**。万级提交里包含格式化、Note 同步、门禁修复、agent 自动提交、细粒度 WIP。稳妥表述：*「在约两个月的高强度窗口里，小团队留下万级提交量级，头部贡献者占比很高，与 agent-first 方式相符。」*
- **适合说**：标杆项目用小团队、短周期，达到了传统模式靠人力难以堆出的迭代密度；前提是「规则在仓内、决策可追溯、检查可自动」，否则高速只等于高速返工。
- **不适合说**：*「我们也要日均 189 commits」*（扭曲成无意义碎提交）、*「一个人 5000 commits = 写了 5000 份功能」*（忽略 agent 与提交粒度）。

**过渡句**：所以答案不在「AI 有多强」，而在**研发流程的纪律**。这套纪律拆成七层，每层都可验证。

---

# 总纲：全景图

```
AGENTS.md  →  skills  →  notes  →  scripts  →  执行时机  →  code review
(规则是什么)  (怎么干)   (为什么)   (机器查什么)   (何时强制跑)   (只查语义)
                            ▲                                      │
                            └────────── 漏网之鱼补成新门禁 ──────────┘

                        ＋ cookbook（这类代码怎么写，第 7 层）
```

**闭环逻辑**：规则 → 固化成手册 → 记录决策依据 → 变成机器可检查的约束 → 在正确时机强制执行 → review 只查机器查不了的语义 → review 发现的可机检问题**回流成新门禁**。

## 各层详解文档

本文是**总览**，每层只讲骨架和最能打的点。要讲透某一层，看对应详解：

| 层 | 详解文档 | 状态 |
|---|---|---|
| ① AGENTS.md | [layer1-agents-md.md](layer1-agents-md.md) | ✅ 已完成 |
| ② skills | [layer2-skills.md](layer2-skills.md) | ✅ 已完成 |
| ③ notes | [layer3-agent-notes.md](layer3-agent-notes.md) | ✅ 已完成 |
| ④ scripts | [layer4-executable-gates.md](layer4-executable-gates.md) | ✅ 已完成 |
| ⑤ 执行时机 | [layer5-hooks-and-ci.md](layer5-hooks-and-ci.md) | ✅ 已完成 |
| ⑥ code review | [layer6-code-review.md](layer6-code-review.md) | ✅ 已完成 |
| ⑦ cookbook | [layer7-cookbook.md](layer7-cookbook.md) | ✅ 已完成 |

## 一个必须先说清的事：这条链是按「约束的载体」切的

上面这条链是按**约束住在哪个文件里**排的（规则文件 / 手册 / 决策记录 / 脚本 / 时机 / review），它不是**开发生命周期**。生命周期是另一条正交的轴：

```
Issue(需求规格)  →  proposed note(设计)  →  cookbook(照着写)  →  门禁  →  PR(引用Issue+验证)  →  review
      ↑                                                                                         │
      └───────────────────── review 发现的需求歧义回流成新 Issue ──────────────────────────────┘
```

**读法建议**：第 1–6 层是一条主干，按顺序读；「前置」回答「spec 和质量从哪来」；第 7 层是主干之外的一块，**哪里需要哪里查**（讲的时候放在第 2 层后面讲效果最好）。

两条轴的对应关系：

| 生命周期阶段 | 载体 | 在本文哪一层 |
|---|---|---|
| 提需求（**spec**） | `.github/ISSUE_TEMPLATE/` 五类模板 | **前置**（见下节） |
| 重大设计（**spec**） | `.agents/notes/proposed/` | 第 3 层 |
| 具体怎么写代码 | `docs/cookbook/` 九份手册 | **第 7 层** |
| 元操作（review/验证/翻译） | `.agents/skills/` | 第 2 层 |
| 全程约束 | `AGENTS.md` + `scripts/` + hooks/CI | 第 1、4、5 层 |
| 合入 | PR 模板 + review | 第 6 层 |

---

# 前置：这套体系里的「spec」在哪

在进第 1 层之前，先回答一个几乎必然被问到的问题：**「你讲的这几层都是辅助约束，那『写代码之前先写 spec』这一步呢？质量难道全靠 AGENTS.md 求出来的？」**

答案分四小节：spec 在 Issue 模板里（①②）、重量级 spec 在 `proposed/` note 里（③）、质量靠门禁不靠提示词（④）。**没有一层靠自觉。**

## ① 提需求就是写 spec，且不许开空白 Issue

`.github/ISSUE_TEMPLATE/config.yml` 第一行：

```yaml
blank_issues_enabled: false
```

**空白 Issue 被物理禁止**，必须从五类里选一类，每类的必填字段就是那类工作的 spec 骨架：

| Type | 一句话开场 | 必填字段 |
|---|---|---|
| `Feature` | 一句话说明**预期结果** | 验收条件 / 用户或模型可见变化 / **测试证据** |
| `Bug` | 记录现有预期行为的失效 | 复现 / 预期 |
| `Task` | 一句话说明要完成的工作 | 验收条件 / 交付物 / **测试证据** |
| `Research` | 一句话说明**待回答的问题** | 核心问题 / **证据标准** / 交付结论 |
| `Idea` | 一句话说明**价值假设** | 价值假设 / 需要验证 |

**注意三处**：

- **三个模板都要求「测试证据」**——从提需求那一刻就得说清「怎么证明做对了」，而不是写完代码再想怎么测。
- **`Research` 类要求「证据标准」**——连「调研」这种最容易糊的工作，都必须预先声明「什么样的证据才算得出结论」。
- **`Idea` 类的存在本身有信息量**——它把「还没承诺实施的想法」和「已承诺的 Feature」在类型层面分开，避免 backlog 里混着一堆薛定谔需求。

## ② 这些不是「模板建议」，是 CI 强制

关键证据：`.github/issue-management/policy.mjs`（约 700 行）由两个 workflow 驱动（`issue-policy.yml` / `issue-lifecycle.yml`），**校验不过直接 fail**：

```js
throw new Error(`Issue policy 未通过，共 ${errors.length} 项`)
```

它 push 的错误项包括（原文摘录）：

```
'PR 正文必须引用至少一个同仓库 Issue'
'Issue 必须在 Project 中且具有合法 Status'
'Type 必须是五种原生英文 Type 之一'
'Issue 标题必须包含中文'
'Priority 必须为空或为 P0–P3'
'正文必须包含默认收起的 <details> 区域'
'PR 必须恰好有一个允许的 kind/*'
'PR 必须至少有一个 area/*'
'Done 必须对应 Completed 关闭原因'
'No action 必须对应 Not planned 关闭原因'
```

**所以「无 Issue 不 PR」是机器拦的，不是口头约定。** Issue 还必须挂在 Project 上并处于合法状态，状态机是封闭集合：

```
Inbox → Backlog → Ready → In progress → In review → Done / No action
```

> 这一条值得单独强调：**需求侧和代码侧用的是同一套手法**——封闭集合 + 机器校验 + 状态编码在数据里。第 3 层的 Agent Note 用路径编码四态，Issue 用 Project 字段编码七态，**同构**。

## ③ 重量级设计 spec = `proposed/` Agent Note

实测数量：

| 状态 | 数量 | 相当于 |
|---|---|---|
| `proposed/` | 25 | **spec / RFC（未落地）** |
| `implemented/` | 515 | ADR（已落地） |
| `rejected/` | 11 | 被否提案（冻结） |
| `archived/` | 142 | 历史快照（冻结） |

`proposed/` 的骨架就是标准 spec：

```
## Problem → ## Proposal（将来时）→ ## Acceptance criteria → ## Risks
```

AGENTS.md 与 `.agents/notes/README.md` 明确了什么时候必须先写 spec：

> Every non-trivial change MUST add or update at least one Agent Note in the same PR. A change is non-trivial when it alters behavior, architecture, a contract shared across files or packages, process or tooling, testing strategy, an on-disk wire or configuration format, **or another decision a maintainer may reasonably revisit**.
> **A proposal for substantial future work starts in `proposed/`; a decision already made starts in `implemented/`.**

## ④ 最关键的一点：代码质量不是「求」出来的

这是回答「如何告诉 AI 产出更好的代码」时**唯一需要讲的观点**：

> **他们不靠提示词让 AI 写好代码，而是让写不好的代码过不去。**

AGENTS.md / `package.json` 里几条最硬的：

| 机制 | 原文/命令 | 治的什么病 |
|---|---|---|
| **逐文件 100% 覆盖率** | `test:coverage # CI coverage gate: per-file 100% on packages/*/*/src` | 「测试写个样子」 |
| **克隆检测** | `pnpm run duplication # cross-file TypeScript clone detection` | **AI 最爱复制粘贴** |
| **禁止 mock 顶替** | 「package tests、e2e-only assertions、mock-only fixtures **do not substitute for** the assembled application transcript」 | 「测过了」但只测了 mock |
| **真实可运行示例** | 「adds or updates a keyless snapshot **through a real runnable example** in the same PR」 | 行为变了没人看见 |
| **每个导出必须有 JSDoc** | `verify-export-jsdoc` 强制 `@param`/`@returns` | AI 不写文档 |
| **每个 `any` 必须解释** | 「every remaining `any` explains why narrowing is infeasible」 | 用 `any` 糊过类型 |

**「逐文件 100% 覆盖」这一条，比一百句「请写高质量代码」都管用**——它把「质量」从形容词变成了 CI 的一个布尔值。

> **顺带回答「是不是用了 superpowers / SDD 之类的方法论」：没有。** 仓库里搜不到任何 `spec-driven`、`superpowers` 的痕迹（`plan mode` 是产品自己的一个功能包 `packages/plan/`，不是团队的开发方法）。它的做法是**把 spec-first 的实质拆进 Issue 模板、`proposed/` note 和 cookbook 里，然后全部用机器强制**——不给方法论起名字，但每一条都可执行。这比买一套方法论更可迁移。

---

# 第 1 层：AGENTS.md（规则是什么）

> 📖 **详解**：[layer1-agents-md.md](layer1-agents-md.md) —— 九个区段拆解、31 条 Conventions、词数预算与棘轮机制、分层 AGENTS.md、落地五步

现在从主干的第一环开始：规则放在哪、怎么防它腐烂。

**解决什么问题**：AI 每次会话都需要知道「这个仓库的规矩」，否则凭通用经验乱猜。

**是什么**：根目录 `AGENTS.md`，每次会话自动加载的常驻 system prompt。1929 词、31 条规则、门禁上限 1950——**只剩 21 词余量**。

**最能打的一个点**：**词数预算 + 棘轮**——保证规则不随时间腐烂。规则文档的天敌是只增不减：每次踩坑都想加一条，三个月后就长到 AI 吸收不了，规则沦为摆设。预算门禁让「加一条新规则」必须先删一条旧的——这个删除的代价逼你每次重新排序「什么最重要」。

> 「棘轮」= 只能单向转动的机械装置（如扎带：只能收紧，松不回来）。这里指**词数上限只许降、不许悄悄升**——想升必须在 PR 里写理由。上限因此永远不会漂移失效，文件只会越勒越紧。

**效果**：因为「不知道仓库规矩」而犯的错，在写代码那一刻就消失。对比下游：scripts 拦「已写出的错」，review 打捞「机器拦不住的错」——这一层让一部分错根本不发生，所以叫「源头」。

---

# 第 2 层：skills（怎么干）

> 📖 **详解**：[layer2-skills.md](layer2-skills.md) —— 两级 context 预算、description 的写法、11 个 skill 全景、四个重点拆解、元数据门禁、落地六步

第 1 层立了规矩，但规矩里装不下「复杂任务具体怎么一步步做」——装不下是因为有词数预算。

**解决什么问题**：AGENTS.md 有 1950 词的硬上限，装不下「复杂任务具体怎么一步步做」。

**是什么**：`.agents/skills/` 下 **11 个 `SKILL.md`**，正文合计 **13578 词**——是 AGENTS.md 的 **7 倍**，却**没有任何词数预算**。

**最能打的一个点**：**两级 context 预算**——第 1 层受严格限额，第 2 层放开管，差别只在于**常驻还是按需**。会话开始时进入 context 的只有 11 组 `name + description`（几百词的「目录」），正文一份都不进；模型判断需要时才用 `skill` 工具把某一份读进来。所以「规则必须极简」和「流程必须极详」这对矛盾被拆到了两层，各自都不用妥协。

> 代价全部压在 description 上：它写不准，模型该用时不用、不该用时乱用，整层失效。所以 11 份的 description 全部以 `Use when` 开头、**写触发条件而不是功能介绍**，甚至把要抓的特征字符串（如 `"used to"`、`"no longer"`）直接列进去让模型模式匹配。

| skill | 干什么 |
|---|---|
| `dsh-pre-push-checks` | push 前选最小验证集（核心，见下） |
| `dsh-code-review` | review 只查语义（第 6 层详解） |
| `dsh-prose-standard` | 必需的文档覆盖面 + 编辑判断 |
| `dsh-doc-standards` | 文档放哪儿、多长、怎么验证 |
| `dsh-trim-cot-leakage` | 清理思维链泄漏 |
| `dsh-translate-docs` | 完整双语工作流（**唯一 manual-only**） |
| `dsh-archive-agent-notes` | Agent Note 的审计与归档判据 |
| `dsh-find-simplifications` | 给「提议删代码」立证据标准 |
| `dsh-merging-stacked-prs` | 用 GitHub 原生 stack 落地依赖 PR 链 |
| `dsh-doc-site-sync` | 文档站点同步 |
| `record-browser-gif` | 录制浏览器 GIF（改 GUI 的 PR 必须附） |

> 注意：11 个里有 **6 个在管「文字」**。一个代码仓库过半的操作手册用于约束文字生产——**在 agent-first 的仓库里，文字就是生产资料**。

## 重点讲：`dsh-pre-push-checks` 的「证据匹配」

核心一句：**按改动面选最小验证集，禁止默认全量跑。**

```
包/脚本行为        → 跑 owning vitest 文件（聚焦测试名）
文档/Agent Notes   → pnpm run doc-sync
模型/编辑器可见输出 → 跑聚焦的 keyless snapshot
包清单/导出/构建   → pnpm run build + hygiene + built smoke
真实 provider 行为 → pnpm run test:e2e（需 key）
```

明确禁止：「**不要因为要 commit/push 就重复跑一个已通过的检查**」；「**CI 拥有全量覆盖和平台矩阵**」。两头都堵：既禁止「一改就跑全套」，也禁止「跑不过就调低阈值」（`--passWithNoTests`、降覆盖率门槛、缩 `--coverage.include` 都被点名禁止）。

**效果**：AI（和人）不再「一改就跑全套」，反馈快且不浪费。

> **这是最能戳中听众的点之一**——所有人都受过「改一行跑半小时」的苦。

## 顺带讲：这一层也有门禁

`scripts/verify-skill-invocation-metadata.ts` 校验同一个 skill 在 Claude Code 侧（`disable-model-invocation: true`）和 Codex 侧（`allow_implicit_invocation: false`）的调用策略语义一致——**两个字段真值相反，最容易改一边忘一边**。故障表现是「某个 skill 在某客户端悄悄失效，不报错、只是不再被调用」，正是人和 review 最容易漏的那类静默失效。

而 `.claude/skills` **是指向 `.agents/skills` 的软链，不是副本**——能用软链消灭的一致性问题，就不要用门禁去检查。

---

# 第 3 层：notes（为什么）

> 📖 **详解**：[layer3-agent-notes.md](layer3-agent-notes.md) —— 四态生命周期完整规则、三套骨架、`rejected/` 的 11 条一行判决、门禁 11 类检查、`quality-gates` 剖析、落地五步

第 2 层解决了「怎么做」的传递，但 agent 干活时还会**做决策**——选 A 不选 B、否决某个方案。第 3 层解决这些决策怎么留痕。

**解决什么问题**：代码和文档承载「是什么」，承载不了「为什么」和「放弃了什么」。

**是什么**：`.agents/notes/` 下的 Agent Note，本质是**由 AI 写的 RFC/ADR**。实测 **551 份活跃 + 142 份归档**。四态生命周期编码在路径里：

```
proposed/      25 份   提案（= spec / RFC，还没做或做了一半）
implemented/  515 份   已落地（记录决定 + 被否决的方案）
rejected/      11 份   被否（只留「避免再犯错」的理由）
archived/     142 份   冻结的历史快照（永不再改，哈希校验）
```

**最反直觉、也最有价值的一条规则**：

> 每份 Note **必须**有 `## Alternatives considered`——记录每个被否决的方案 + 落选原因。
> **「记录决策却不记录它击败了什么，等于邀请反复争论。」**

**而它有一条更了不起的配套规则**：

> **`Alternatives are recorded, never invented.`**（备选方案是被记录的，不是被编造的。）

补不出来的老 note 必须携带一个逐字精确的豁免注释，**实测 19 份带着这个诚实的缺口**。那份格式 note 自己解释了为什么：**「an honest gap on the record rather than fabricated rationale」**——宁可有个诚实的洞，也不要编造的理由。

> **这条价值远超它管的 19 个文件**：一旦允许 AI 补编理由，整个决策记录体系的可信度就归零——因为你再也无法区分哪条理由是真的。

**生命周期用不同骨架**（时态不同）：

```
proposed     → ## Proposal（将来时）→ ## Acceptance criteria → ## Risks
implemented  → ## Decision（现在时）→ ## Consequences
rejected     → 冻结提案 + Status: rejected — 原因
```

proposed → implemented 是**机械化改写**（Proposal→Decision，Acceptance criteria 折进 Consequences），且门禁用一条正则拒绝 implemented note 里出现提案期用语：

```ts
const BANNED_IMPLEMENTED = /^## (?:Proposal\b|Plan\b|Migration plan\b|Acceptance criteria\b)/i
```

## 顺带讲：`rejected/` 的 11 条一行判决 = 已经付过的学费

`Status:` 行是三种状态里唯一带内容的，因为「读者来看被否 note，要的就是那个判决」。挑三条：

| note | 判决（截断） |
|---|---|
| `builtin-timer-promises-for-hand-rolled-sleeps` | rejected — **实现（PR #679）证伪了等价前提**：vitest 的 fake clock 不拦截 `node:timers/promises` |
| `dependency-swaps-rejected-by-nih-audit` | rejected — 每项替换在证据面前都过不了「净简化」门槛；**记录下来，这样这份调研不必从零重跑** |
| `evaluate-landstrip-for-windows-sandbox-rung` | rejected — landstrip **不够经受考验**（几天前建立、单维护者、约 48 star） |

> **第一条是被实现证伪的**（有人真去做了，发现前提不成立），**第二条冻结了一次完整调研的结论**。这些「看起来显然该做」的简化，每一个都会被下一个 agent 再次提出来。**`rejected/` 不是垃圾桶，是学费存档。**

**效果**：未来无论人或 AI，都不会重走已讨论过的坑，也不会重复争论。

> **这是第二个能戳中听众的点**——对应「经验随人走」的痛。

---

# 第 4 层：scripts（机器查什么）

> 📖 **详解**：[layer4-executable-gates.md](layer4-executable-gates.md) —— 四类脚本全景、run-gates 依赖图与并发调度、封闭集合三例、`verify-doc-budgets` 完整剖析、副作用封堵、落地六步

前三层产出了规则、手册、决策记录——**全是文字**。第 4 层把文字变成代码：**每条「应该怎样」都要有一条会返回非 0 的命令**。

**解决什么问题**：规则如果只能靠人读文档遵守，就等于没有规则。

**是什么**：`scripts/` 下 **128 个脚本**（27 个 `verify-*`、13 个 `gen-*`）+ `run-gates.ts` 调度器。真实规模是**约 60 个门禁组织成 13 个聚合模式**。

**立法原则只有一条**（`quality-gates` note 的 Decision 第一句）：

> **Every mechanically checkable AGENTS.md promise gets a command that exits non-zero.**
> （AGENTS.md 里每一条机器可查的承诺，都要有一条会返回非 0 的命令。）

而它的理由是这套体系最重要的一句自述：

> This codebase is developed **primarily by coding agents**. **Agents follow enforced gates far more reliably than prose conventions**…

> **注意措辞不是「AI 不可靠」，而是「强制门禁比散文约定可靠得多」。** 对人也成立，只是 AI 把差距放大了——**AI 没有「上次被骂过」的记忆**，所以一条没有门禁的规则，对它等于一条每次都要重新赌一次的规则。

**四类脚本**：

1. **门禁调度器**（`run-gates.ts`，唯一一个）：13 个聚合模式，依赖图三重校验，有界并发。
2. **文档一致性门禁**（数量最多）：`doc-sync` 这一个聚合就有 **28 个叶子门禁，其中 25 个管文档**。
3. **代码/结构不变量**：`verify-package-invariants`、`verify-runtime-closure`、`knip`、`publint`、`duplication` 等。
4. **纯工具与生成器**：13 个 `gen-*`（生成与校验成对出现）、`clean.ts`、`change-scope.ts`。

> **28 个门禁里 25 个管文档**——这是「文字就是生产资料」的第三次出现（第 2 层 11 个 skill 里 6 个管文字，第 7 层 9 份 cookbook 里 3 份管流程）。原因很直接：**AI 读文档来理解仓库，文档错了 AI 就系统性地错**。而人读错文档还有「感觉不对，去看看代码」的本能。

**关键设计：结构性唯一事实来源**

`agent-note-tree.ts` 写死了封闭集合：

```ts
export const AGENT_NOTE_CLASSES = ['feature', 'bug-fix', 'simplification', 'architecture', 'process', 'testing'] as const
```

注释明确：「新增类别是刻意的行为——同时更新这个列表 AND README 的 Classification 节」。而且连 lifecycle 集合也封闭，理由写在注释里：

```ts
// The lifecycle set is closed too: any directory under .agents/notes/ that is not
// a known lifecycle would otherwise hold Agent Notes invisible to the walk below.
```

**如果不封闭，有人建个 `draft/` 文件夹，里面的 note 就对所有门禁隐形了**——不报错，只是不被检查。**这正是最危险的失效模式：静默地不生效。**

## 顺带讲：一条极容易漏的门禁设计

`verify-doc-budgets` 查三类失败，其中第二类才是精髓：

```
1. ceiling 不是正整数
2. 被预算的文件不存在   ← 光查「超字数」的话，删掉文件就能让门禁变绿
3. 词数超过 ceiling
```

**「约束被删除本身消灭了」是所有清单式门禁的通病。** 而且它的报错信息复述了解决顺序（`relocate or condense`，提升上限是最后手段），**所以 AI 看到报错就知道该按什么顺序尝试**，而不是直接去改 manifest。

**效果**：规则从「人自觉」变成「机器强制」，AI 违反立刻报错，而不是等 review 才发现。

---

# 第 5 层：执行时机（何时强制跑）

> 📖 **详解**：[layer5-hooks-and-ci.md](layer5-hooks-and-ci.md) —— lefthook 逐 job 拆解、ci.yml 的 7 个阻塞 job、`all-checks-passed` 的承重设计、failover 与热备演练、落地六步

第 4 层造出了约 60 个门禁，摆在面前的问题是：**哪些在本地跑、哪些在 CI 跑？**

**解决什么问题**：脚本写得再好，不挂到正确时机就等于没写。

| 执行者 | 时机 | 原则 | 实际内容 |
|---|---|---|---|
| **Git hook**（`lefthook.yml`） | `pre-commit` / `pre-merge-commit` / `pre-push` | 窄而快 | **6 + 2 + 1 个 job** |
| **CI**（`.github/workflows/`） | push / PR | 全而慢 | 15 个 workflow，`ci.yml` 里 **7 个阻塞 job** |
| **人手动**（`pnpm run check:all`） | 按需 | 证据匹配 | **「贡献者命令，不是 agent 指令」** |

**为什么本地不该多跑**——`fast-local-git-hooks` note 给了三个理由，第三条是关键：

```
① 延迟每一次发布
② 放大无关的本地不稳定
③ CI 马上要把同样的东西再跑一遍 → 这次运行的信息量是零
```

**这不是「快 vs 严」的取舍，而是重复劳动。** 前提是 agent 已按第 2 层的证据匹配跑过针对性检查。

**本地 hook 的三个原则**：

```
① glob 按改动文件精确触发：只改 ts 才跑 lint，没碰归档不跑 archived 检查
② 能自动修的修掉而不是拒绝：lint --fix + stage_fixed；notices 直接重新生成
③ pre-commit 只做快事：pre-push 只有一个 job（typecheck）
```

核心注释：**「Keep these local checkpoints fast; CI owns the full repository-wide gate matrix.」**

而 pre-commit 有一份**否定清单**（明确排除 7 项）：

> **Pre-commit does not run type analysis, tests, snapshots, documentation checks, builds, hygiene, or the gate scheduler.**

> **把「不做什么」写下来，价值等同于写「做什么」**——否则下一个人（或 agent）看到 pre-commit 里没有测试，会本能地想「是不是漏了，我加上吧」。

## 重点讲：`all-checks-passed` 里那个承重的 `if: always()`

CI 有一个只做汇总的 job，给分支保护提供稳定的必需检查名。它的注释点出了一个会让整个 CI 保护静默失效的漏洞：

```yaml
# `if: always()` is load-bearing: without it a failed dependency
# would SKIP this job, and GitHub counts a skipped required check as passing
```

拆开看：

```
默认行为：依赖失败 → 这个 job 被 SKIP
GitHub 行为：被 skip 的必需检查算作「通过」
合起来 = 上游全挂 → 汇总被跳过 → 分支保护认为一切正常 → 可以合并
```

所以它的失败条件把三种非成功状态都列了：`failure` || `cancelled` || **`skipped`**。

> **`skipped` 必须算失败**——这和第 4 层 `run-gates.ts` 里「`skipped` 与 `failed` 同等对待」是**同一条原则在两个层次上各堵了一次**。「跳过被误判为通过」是 CI 里最经典的静默失效，因为它不产生任何红色信号。

另外两个值得一提的设计：

- **阻塞用快的，观察用真的**：`windows`（Wine 跑 Windows Node，15 分钟）进 needs 阻塞 PR；`windows-native`（真实 Windows，120 分钟）**故意不在 needs 里**，保留独立结论但不拖慢 PR。
- **灾备被持续演练，不是写在文档里**：每次 master push 都在自建 VM 上跑一遍完整聚合，**持续证明它能接管**。切换开关是仓库变量不是代码——**CI 已经挂了的时候，你最不需要的就是「先提个 PR 改 workflow」这种死锁**。

**效果**：绕过本地 hook 的逃不过 CI；而本地 hook 足够快，所以没人用 `--no-verify` 绕过。

> **这是第三个能戳中听众的点**：**太慢的 hook 一定会被绕过，最后形同虚设。**
>
> 有趣的是，他们**刻意没有给「hook 必须多快」写计时测试**——因为那种测试依赖机器负载，会变成不稳定门禁。改用在开发和 PR 证据里观察。**不是所有约束都该自动化，这个判断本身也是工程能力。**

**代价要说清**：本地不再证明全量矩阵，于是「**评估 agent 选的证据是否匹配 diff**」成了第 6 层 review 的新职责。**这一层的收益和下一层的负担是绑在一起的。**

---

# 第 6 层：code review（只查语义）

> 📖 **详解**：[layer6-code-review.md](layer6-code-review.md) —— 6 条硬阻断逐条、14 项人工检查为何机器做不到、`change-scope` 的严格性、四条反 AI 通病的纪律、标准自身的演进机制

门禁再全，也有机器物理上查不了的东西（「这个抽象有没有存在的必要」「这个测试是不是在复述实现」）。这最后一道防线就是 review——**也是前面各层存在的目的**。

> **这一层是分享时最容易被挑战的一环。**

**必然会被问到**：这么大代码量、这么快产出，传统人工 review 绝对扛不住；那质量是不是就全靠测试和 lint 了？

## ① Review 没有缺失，反而被规范得最细

`dsh-code-review/SKILL.md` 只有 **1073 词**（8.08 KB），却装了 **6 条硬阻断 + 14 项人工检查 + 7 个权威来源 + 报告规范**——平均每条规则不到 45 词，**是所有 skill 里信息密度最高的一份**。

它之所以这么短，是因为**不复述任何规则**：细节全部外链给 `AGENTS.md`、`defensive-patterns.md`、`packages/AGENTS.md`、`dsh-prose-standard`。它只负责两件事——**指出该去查哪条规则，以及哪些东西只能靠判断**。

而开篇第一句是个反预期声明：

```
**This skill is guidance, not a complete checklist.**
```

> **「这是指引，不是完整清单」**——如果它自称完整，AI 就会逐条打勾然后宣布通过，清单外的问题一律不看。

**关键在于它是 skill——写给 AI 执行的**。所以真实模型是：

> **AI review AI，人只做仲裁。Review 侧和产出侧用同一种方式扩容，所以扛得住。**

这也解释了为什么 review 标准必须是 skill 而不是 wiki 页面：**wiki 给人读，人读完会内化；skill 给 AI 读，AI 每次 review 都要重新读一遍**——所以它必须自包含、密度极高、每条可判定。

## ② 门禁不是替代 review，而是「为 review 腾出空间」

skill 的 Reporting findings 有一句决定性的分工声明：

```
omit issues already enforced by a green gate
（门禁已经绿了的问题，review 不要再提）
```

这把 review 的**输入面积**压到最小：

```
门禁把「事实类问题」清零（格式/类型/链接/配对/覆盖率/死代码…）
              ↓
review 只剩语义问题 → 少量 reviewer（含 AI）扛得住高吞吐
```

配套还有 `scripts/change-scope.ts`：skill **强制要求** review 前先跑 `pnpm run change-scope --base <ref> --head <ref>`，拿到结构化改动范围报告。但紧接着写明：

> The report identifies paths and dirty layers but **does not replace semantic review**.

**工具定位边界，AI/人做判断**——和门禁一样的分工模式。

## ③ 他们自己明确否认「测试和覆盖率就够了」

回击「是不是全靠测试 lint」最有力的证据，全是 skill 原文：

| 原文 | 意思 |
|---|---|
| `Coverage is necessary but not evidence that the scenario is correct` | **覆盖率必要，但不证明场景正确** |
| `rather than ... trusting an agent's report` | **不要相信 agent 的自我报告** |
| `automated checks do not establish those properties` | 自动检查建立不了这些性质 |
| `a green pairing hash does not prove translation quality` | 哈希绿了不代表翻译对 |

而 `quality-gates` 这份 Agent Note 的 Problem 段落是直接的自述现场：

> This codebase is developed **primarily by coding agents**. Agents follow enforced gates far more reliably than prose conventions... Early evidence: tests that didn't typecheck shipped (vitest doesn't typecheck) and were **only caught by a review**.

一箭双雕：既承认「主要由 coding agent 写」，又承认 **review 抓到了门禁漏掉的 bug**——然后他们才去补门禁。**这就是「踩坑 → 固化成门禁」闭环的第一现场。**

## Review 具体查什么：全是机器物理上查不了的

`Manual checks` 共 14 项，无一项能被门禁覆盖：

| 检查项 | 为什么机器查不了 |
|---|---|
| Lifecycle and concurrency | 竞态、await 期间取消、销毁顺序——需理解意图 |
| Capability and consumer fit | 「新公有方法只有一个内部调用者」= 不必要的 API 膨胀 |
| Scope, ownership, necessity | 投机性泛化、无关功能夹带 |
| Configuration and public choices | 每个默认值有没有「当前消费者证据」支撑 |
| Model perspective | 模型收到的 prompt 里有没有「不属于它任务」的概念 |
| Enforcement | 沿每条拒绝路径找**能绕过它的备用调用方** |
| Borrowed and derived state | 值是借用还是拥有 |
| Test strength | 断言是否真能在目标回归上失败 |
| Bilingual changes | 双语「意思」是否一致 |

`Enforcement` 最典型：「follow every denial path to the operation that executes it; exercise direct and alternate callers that can bypass schemas, prompts, facades, wrappers, or listener ordering」——**纯语义推理，任何 lint 都做不到**。

## 几条值得单独讲的 review 纪律

- **优先级明确**：`Prioritize correctness, lifecycle, security, and broken required behavior over style`——先查正确性/生命周期/安全，不是风格。
- **反对 nit 堆砌**：`a short review with one substantiated blocker is better than a list of nits`。
- **反谄媚**：`verify each claim and fix or rebut it on technical grounds without performative agreement`——收到 review 逐条核实，**该反驳就用技术理由反驳，不许表演性同意**。这条专治 AI 的「您说得对」通病。
- **Agent Note 不是免死金牌**：`Treat disagreement with an Agent Note as a design discussion, not an automatic veto`。

**效果**：review 从「查一切」收缩到「只查语义」，输入面积被压缩一个数量级；加上 reviewer 本身可以是 AI，review 才跟得上 agent 的产出速度。

---

# 第 7 层：cookbook（这类代码具体怎么写）

> 📖 **详解**：[layer7-cookbook.md](layer7-cookbook.md) —— 九份手册逐一拆解、四份重点、角色词表、「decided up front」接缝、与 skills 的分界、三类门禁、落地五步

主干讲完了，还剩一块没归位的东西：`docs/cookbook/`。它不属于那条主干链，但**它是 11 个 skill 之外、体量第二大的操作手册**，讲的时候建议放在第 2 层后面。

它回答的是前面各层都没回答的问题：**「我现在要加一个工具/一个包，第一步干什么？」**

**解决什么问题**：AGENTS.md 说「规则」，skills 说「元操作怎么做」（review、验证、翻译），但都没说**「这个仓库里加一类具体东西的标准路径」**。这类知识最容易「只在老员工脑子里」。

**是什么**：`docs/cookbook/` 下 **9 份分步手册，合计 9838 词、34 个代码块**——比 AGENTS.md 大 5 倍，且**零词数预算**。

| cookbook | 词数 | 教什么 |
|---|---|---|
| `adding-a-tool.md` | 1710 | 加一个模型可调用的工具 |
| `adding-a-package.md` | 1674 | 加一个 workspace 包（含 19 行**角色词表**） |
| `adding-a-conversation-node.md` | 1642 | 加一个会话节点 |
| `extension-cookbook.md` | 1399 | 各扩展点用法 + **「产品功能 → 插件机制」全映射表** |
| `maintaining-dsh-code-review.md` | 1051 | **维护 review skill 本身** |
| `adding-a-settings-card.md` | 756 | 加一张设置卡片 |
| `responding-to-pr-review-on-a-stack.md` | 582 | 在 PR 栈上响应 review |
| `adding-an-llm-adapter.md` | 512 | 加一个 LLM 适配器 |
| `adding-a-vendored-package.md` | 512 | 加一个 vendored 包 |

`docs/AGENTS.md` 用一张表钉死了它的职责边界：

| 层 | 该放什么 | **不该放什么** |
|---|---|---|
| `cookbook/` | **带编号验证步骤的分步 how-to** | 设计理由（→ 每份手册链接的 Agent Note，9 份里 7 份有，共 11 处） |

一句话概括全局分工：

> **Placement: bugs → postmortems; rationale → Agent Notes; procedures → cookbooks; type definitions → subsystems; package contracts → READMEs; standing orders → root `AGENTS.md`.**

## 最能打的一个点：cookbook 会被提升成 AGENTS.md 里的硬规则

AGENTS.md 的 31 条 Conventions 里有这么一条：

> **A tool's UI render intent is part of its design, decided up front** (`generic`/`terminal`/`diff`, `locations`); presentation methods are pure functions of `args`（[cookbook](docs/cookbook/adding-a-tool.md)）

**「decided up front」（事先决定）**——这就是 spec-first 落到代码层的具体形态：加工具时，UI 渲染意图必须**在写实现之前**定下来，不能写完再补。而**怎么定、有哪几种选项、怎么验证**，全在 cookbook 里。

一条规则被切成三段，各住各家：

| 部分 | 住在哪 | 占预算吗 |
|---|---|---|
| 铁律「必须事先决定」 | AGENTS.md（**1 行**） | 占 |
| 五种选项各自何时用 | cookbook（约 400 词） | **不占** |
| 为什么设计成这五种 | Agent Note `2026-07-02-tool-render-intent-union.md` | 不占 |

> **可直接迁移的模式**：AGENTS.md 里放一句「加 X 必须事先决定 Y」（1 行，占预算），cookbook 里放「Y 怎么决定」（1700 词，不占预算），ADR 里放「为什么」。**又是一次「常驻极简 + 按需极详」，和第 2 层同一个套路。**

而且 `packages/AGENTS.md` 链接 cookbook 时用的是 **`#锚点`**（如 `#name-the-role-that-exists`），`verify-md-links` 会校验死锚点——**这个接缝由机器维护，小节改名 CI 立刻红**。

## 顺带讲：示例代码被纳入 typecheck

34 个代码块里，除少数标 `ignore-check` 的，**其余 `ts` 块都必须真的编译通过**（`doc-typecheck`）。

> 这解决了技术文档最经典的腐烂方式：**示例代码在某次重构后不能跑了，但没人发现，直到有个新人（或 AI）照抄了它。** 把示例纳入 typecheck，等于让文档和代码共享同一个正确性门禁。

## 为什么它不该混进第 2 层（skills）

两者性质不同，混讲会把第 2 层「两级 context 预算」那个点讲糊：

| | skills（第 2 层） | cookbook（第 7 层） |
|---|---|---|
| 内容 | **元操作**：review、验证、归档、翻译、找简化 | **业务代码怎么写**：加工具、加包、加适配器 |
| 加载方式 | 模型按 `description` 触发，`skill` 工具按需读入 | **普通文档**，被 AGENTS.md / README 链接后主动去读 |
| 读者 | 主要是 AI | **人和 AI 都读** |
| 词数预算 | 无 | 无 |
| 变更频率 | 低（流程稳定） | 跟着架构走 |

**一句话区分**：**skills 管「怎么做事」，cookbook 管「怎么写这类代码」。**

**技术原因**：skill 的价值建立在「模型能通过 description 精准判断该不该读」之上。而「加一个工具」的触发信号是**用户直接说出来的意图**，不需要模型猜。把 cookbook 塞进 skills，等于给 11 个 description 再加 9 个，**稀释整个触发机制的准确率，收益为零**。

## 一个自指的细节值得单独讲

`maintaining-dsh-code-review.md`（1051 词）——**一份「怎么维护 review skill」的 cookbook**，是第 6 层那条回边的 SOP。

工作流：每天跑，选窗口内已 merge 的 PR，收集**人类 review 反馈**并对比「反馈时」与「最终落地」的 patch，**两个独立配置的 reviewer adapter** 分类哪些意见被采纳且当前 skill 未覆盖，起草新 `SKILL.md`，两个 adapter 互相 review 直到都通过，跑 `doc-sync` + `lint`，最后**人做终审**。

其中两条最锐利：

- **双 adapter 字节相同就拒绝运行**——「The tool refuses to run when the two reviewer commands resolve to byte-identical executables.」**把「双 reviewer 独立性」从「注意一下」变成启动时硬校验**；否则「互相 review」就是自我确认剧场。
- **全 provider 故障不许退化成「无候选」**——「it never collapses a total-provider outage into 'no candidate.'」**静默失效不许伪装成正常**。

还有一条明确禁止 AI 崇拜：**「Do not defer to 'the reviewers approved'」**——不许因为两个 AI 都批了就放行，人必须自己读 diff，并抽查「那条被引用的人类评论真的支持这条新规则吗」。

> **这份 cookbook 的存在本身就是论据**：总览说「review 漏网之鱼回流成新门禁」听起来像句漂亮话，而它证明**那条回边有日常机制、有工件目录、有交叉验证、有人类终审、有故障策略**。闭环不是口号，是有 SOP 的。

## 另一条值得单独念的：怎么和 AI 协作

`responding-to-pr-review-on-a-stack.md` 里这段是全仓库对 AI 协作最清醒的一段：

> Treat delegated fixes as trust-but-verify: **a sub-agent's report describes intent, not necessarily what landed.** ... for a regression guard, **prove it FAILS on the unfixed code** (introduce the regression, watch red, revert) — **a guard that passes both ways guards nothing**. **A sub-agent that reframes a problem as already handled is a signal to dig in personally.**

三层意思：① 子 agent 的报告是意图不是事实，自己重跑门禁；② **回归测试必须先证明它在未修复代码上会红**——这是对付「AI 写了个永远通过的测试」的唯一有效手段；③ **AI 把问题重述成「已经处理了」是危险信号**——精确命中 AI 最典型的失败模式：不是报错，而是**说服你问题不存在**。

## 一个反直觉的观察

9 份里有 **3 份不教「写代码」**：`maintaining-dsh-code-review`（维护规则本身）、`responding-to-pr-review-on-a-stack`（和 review 协作）、`adding-a-vendored-package`（管理依赖边界）。

**三分之一的「代码手册」在管流程和协作**——和第 2 层「11 个 skill 里 6 个在管文字」是同一现象：

> **AI 不会通过「跟着老员工干两周」学会团队默契。默契必须被显式化，否则它对 AI 不存在。**

**效果**：「这个仓库里加一类东西的标准路径」从口耳相传变成可执行文档，新人（和新 agent）不需要靠读旧代码猜惯例。

---

# 收尾：闭环与效果

## 七层如何跑顺

```
      ┌── Issue（五类模板，无 Issue 不 PR，CI 强制）
      ▼
┌──────────────────────────────────────────────────────┐
│  ① AGENTS.md      规则是什么（常驻，1950 词上限）       │
└──────────────┬───────────────────────────────────────┘
               │ 复杂任务触发                  ┌──────────────────────────┐
┌──────────────▼──────────────────────────┐   │ ⑦ cookbook               │
│  ② skills         怎么干（11 份手册）     │◄──┤ 这类代码怎么写（9 份）     │
└──────────────┬──────────────────────────┘   │ 被 ① 的规则条款直接引用    │
               │ 干活中做决策                  └──────────────────────────┘
┌──────────────▼───────────────────────────────────────┐
│  ③ notes          为什么（proposed=spec / implemented）│
└──────────────┬───────────────────────────────────────┘
               │ 规则固化成机器可查
┌──────────────▼───────────────────────────────────┐
│  ④ scripts        机器查什么（verify-*.ts 门禁） ◄──┐
└──────────────┬───────────────────────────────────┼─┘
               │ 挂到正确时机                       │
┌──────────────▼───────────────────────────────────┼──┐
│  ⑤ lefthook + CI  何时强制跑（本地窄快 + CI 全慢） │  │
└──────────────┬───────────────────────────────────┼──┘
               │ 事实类问题已清零                    │
┌──────────────▼───────────────────────────────────┼──┐
│  ⑥ code review    只查语义（AI review AI，人仲裁）│  │
└──────────────┬───────────────────────────────────┼──┘
               │ 发现「门禁本该拦住」的漏网之鱼        │
               └──────────────────────────────────┘
```

**最后那条回边是整套体系的灵魂**：review 抓到的每个「机器本可以查」的问题，都会变成一条新的 `verify-*.ts`。所以**门禁集合是被 review 喂大的**，review 负担因此持续下降——项目越大反而越稳。

## 最终效果

1. **每一层都可验证**：没有靠自觉的环节，规则 → 机器检查 → 强制执行，闭环完整。**连「提需求」和「开 PR」都是机器校验的**（`policy.mjs`）。
2. **反馈速度分层**：hook 秒级、CI 分钟级、手动按需。开发者体验好，不会绕过。
3. **文档漂移物理上不可能**：文档与代码同一个 diff 诞生，Agent Note 强制随 PR 提交，机器校验格式和配对。
4. **Review 只查语义**：事实类问题已被门禁清零；**且 reviewer 本身可以是 AI**——review 与产出用同一种方式扩容，这才是高吞吐下质量不塌的真正原因。
5. **质量不靠提示词，靠「过不去」**：逐文件 100% 覆盖、克隆检测、禁止 mock 顶替真实运行示例——把「质量」从形容词变成 CI 的布尔值。
6. **自我校正**：踩坑 → 写成规则 → 规则被执行 → 防止再踩；review 的漏网之鱼回流成新门禁。且「与 Agent Note 分歧当作设计讨论」保证不僵化。

---

# 答疑弹药库

## 挑战一：「这个流程里没有 code review 吧？」

**最可能被问到的问题，准备好这五张牌**（详见第 6 层）：

| # | 牌 | 一句话 |
|---|---|---|
| 1 | **事实纠正** | `dsh-code-review/SKILL.md` 1073 词装 6 条硬阻断 + 14 项人工检查 + 7 个权威来源，**信息密度最高的一份 skill** |
| 2 | **人怎么扛得住** | 扛不住，所以**不靠人扛**。它是 skill，写给 AI 执行，人只做仲裁 |
| 3 | **是不是全靠测试 lint** | 他们自己否认：「覆盖率不证明场景正确」「不要信 agent 的自我报告」 |
| 4 | **门禁 vs review** | 是减负关系不是替代关系：`omit issues already enforced by a green gate` |
| 5 | **review 真的在跑吗** | `quality-gates` Note 承认有 bug「**only caught by a review**」，然后才去补门禁 |

**追问「那 PR 流程呢？」**：`.github/pull_request_template.md` 强制「进入评审的非 Draft 人类 PR 至少引用一个同仓库 Issue」并填「变更 + 验证」；`AGENTS.md` 规定「Choose PR history deliberately」（拆分独立变更、先修引入 PR 再传播）。**而且这不是模板提示，是 `policy.mjs` 在 CI 里 `throw`**（「PR 正文必须引用至少一个同仓库 Issue」）。**PR 流程是完整的，只是 reviewer 的构成变了。**

## 挑战二：「这里面没有 spec 环节吧？光靠 AGENTS.md 求 AI 写好代码？」

**第二可能被问到的问题**（详见「前置：这套体系里的 spec 在哪」），四张牌：

| # | 牌 | 一句话 |
|---|---|---|
| 1 | **提需求就是写 spec** | `blank_issues_enabled: false` 禁止空白 Issue；五类模板，三类必填「测试证据」，`Research` 类必填「证据标准」 |
| 2 | **重大设计有 spec** | `.agents/notes/proposed/` 25 份，骨架 = `Problem → Proposal → Acceptance criteria → Risks`，就是 RFC |
| 3 | **怎么写代码有手册** | `docs/cookbook/` 9 份约 9800 词；AGENTS.md 甚至把它提升成硬规则「UI render intent **decided up front**」 |
| 4 | **质量不靠求，靠过不去** | **逐文件 100% 覆盖率**（`test:coverage`）、克隆检测（`duplication`）、禁止 mock 顶替真实运行示例、每个 `any` 必须解释 |

**追问「用了 superpowers / SDD 之类方法论吗？」**：**没有**。仓库里搜不到 `spec-driven`、`superpowers`（`plan mode` 是产品功能包 `packages/plan/`，不是团队方法论）。它把 spec-first 的**实质**拆进 Issue 模板、`proposed/` note、cookbook，然后全部机器强制——**不起名字，但每条都可执行**。这反而更可迁移：你不需要引进一套方法论，只需要挑几条能被 CI 校验的。

## 挑战三：「这是不是只适合他们那种项目？」

回到那条**回边**：门禁集合是被 review 逐步喂大的，不是一开始就有 36 个。**任何项目都可以从 3 个门禁起步**，让 review 告诉你下一个该补什么。

## 挑战四：「我们没有那么多人力写门禁」

`quality-gates` Note 原文回应过这个质疑：

> **「a lot of work」is not a cost argument when agents do the labor.**
> （当干活的是 agent 时，「工作量大」不构成成本理由。）

---

# 落到自己项目的迁移要点

1. **先关掉空白 Issue，立 2–3 类模板，每类必填「验收条件 + 测试证据」**——这是投入产出比最高的一步，改一个 YAML 文件即事。
2. **先写 AGENTS.md，但第一步故意不写 Conventions**——等踩坑再补，且每条都要满足「可裁定性」。
3. **skills 只写「复杂且重复」的元操作**（review、验证、归档），不是所有任务都要写。
4. **cookbook 从「团队里问得最多的那个『怎么加 X』」写起**，一份就够；配套在 AGENTS.md 里加一行「加 X 必须事先决定 Y」。
5. **notes 先跑通 proposed → implemented 两态**，rejected / archived 后期再加。
6. **scripts 先做能自动检查的少数几个**（lint、文档格式），不要一上来堆 36 个。
7. **本地 hook 只放快的**，全量留给 CI——太慢的 hook 会被绕过。
8. **review 清单先立「机器查不了什么」**，并约定「门禁绿了的不再提」，这样 review 才有减负效果。

## 五个最能打动听众的「钩子」

分享时建议**各配一个团队真实案例**，比讲 DeepSeek 本身更有说服力：

| 钩子 | 对应痛点 |
|---|---|
| **质量不靠提示词，靠过不去**（前置 ④） | 「反复叮嘱 AI 写好点，没用」 |
| **证据匹配**（第 2 层） | 「改一行跑半小时全套测试」 |
| **Alternatives considered 强制**（第 3 层） | 「经验随人走，换个人又讨论一遍」 |
| **太慢的 hook 会被绕过**（第 5 层） | 「CI/规范形同虚设」 |
| **两边都绿的守卫什么都没守**（第 7 层） | 「AI 写的测试永远通过」 |
