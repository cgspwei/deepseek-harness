# 第 2 层详解：skills —— 按需加载的操作手册

> [← 返回六层全景图](how-deepseek-does-ai-coding.md)
>
> 本文把 `.agents/skills/` 讲透：为什么需要它、它靠什么机制不挤爆 context、11 个 skill 各自管什么、你怎么抄。
> 所有数字和引文均对照仓库真实文件核实。

**一句话定位**：

> 如果 AGENTS.md 是「宪法」，skills 就是「部门操作手册」——**宪法必须短到人人随身携带，手册可以厚到只在需要时才翻开**。

---

## 目录

1. [先看数字：和第 1 层的强烈对比](#1-先看数字和第-1-层的强烈对比)
2. [为什么必须有第 2 层：AGENTS.md 装不下什么](#2-为什么必须有第-2-层agentsmd-装不下什么)
3. [核心机制：两级 context 预算](#3-核心机制两级-context-预算)
4. [description 是这一层的命门](#4-description-是这一层的命门)
5. [SKILL.md 的解剖](#5-skillmd-的解剖)
6. [11 个 skill 全景](#6-11-个-skill-全景)
7. [重点拆解四个 skill](#7-重点拆解四个-skill)
8. [skill 之间的引用图：一条规则一个家](#8-skill-之间的引用图一条规则一个家)
9. [「guidance, not a script」——反清单化的自我约束](#9-guidance-not-a-script反清单化的自我约束)
10. [三种附件：references / scripts / agents](#10-三种附件references--scripts--agents)
11. [连元数据都有门禁](#11-连元数据都有门禁)
12. [`.claude/skills` 是软链，不是副本](#12-claudeskills-是软链不是副本)
13. [Dogfooding：harness 自己读自己的 skills](#13-dogfoodingharness-自己读自己的-skills)
14. [怎么抄：给自己项目的落地步骤](#14-怎么抄给自己项目的落地步骤)

---

## 1. 先看数字：和第 1 层的强烈对比

先建立体感。把第 1 层和第 2 层的体量放在一起看，才能看懂这个设计：

| 维度 | 第 1 层 `AGENTS.md` | 第 2 层 `.agents/skills/` |
|---|---|---|
| 文件数 | 1（+ 12 个分层） | **11 个 `SKILL.md`** |
| 总词数 | **1929 词** | **13578 词**（正文） |
| 含附件总词数 | — | **16957 词** |
| 与第 1 层的倍数 | 1× | **约 7×**（正文）/ **约 8.8×**（含附件） |
| 词数预算门禁 | **有**，上限 1950 | **无** |
| 何时进入 context | **每次会话，全文** | **仅被触发时，单份** |

**这张表就是第 2 层的全部设计意图**：

> 体量差 7 倍，却没有任何词数预算——因为它**不常驻**。

单个 SKILL.md 的实测词数：

| skill | 词数 | 行数 |
|---|---|---|
| `dsh-find-simplifications` | **2038** | 146 |
| `record-browser-gif` | 1828 | 110 |
| `dsh-translate-docs` | 1399 | 74 |
| `dsh-prose-standard` | 1240 | 81 |
| `dsh-merging-stacked-prs` | 1161 | 127 |
| `dsh-pre-push-checks` | 1156 | 115 |
| `dsh-code-review` | 1073 | 49 |
| `dsh-trim-cot-leakage` | 1005 | 45 |
| `dsh-doc-site-sync` | 910 | 86 |
| `dsh-doc-standards` | 903 | 56 |
| `dsh-archive-agent-notes` | 865 | 68 |

**注意最长的那个**：`dsh-find-simplifications`（找简化机会）2038 词——**单份就超过了整个 AGENTS.md**。一份「教你怎么删代码」的手册比宪法还长，这不荒谬，恰好说明分层的必要：这 2038 词一年可能只触发几十次，让它每次会话都占 context 才是荒谬的。

---

## 2. 为什么必须有第 2 层：AGENTS.md 装不下什么

第 1 层有硬上限 1950 词，那么**哪些内容注定进不去**？看两个真实对比。

### 对比一：同一件事，两种粒度

AGENTS.md 里关于「push 前跑什么检查」，**只有 3 行**：

```
Run checks before pushes via [dsh-pre-push-checks](.agents/skills/dsh-pre-push-checks/SKILL.md); report only commands run. After `gh stack sync` validate immediately; do not merge before checks pass.

- Match evidence to the surface: focused tests for behavior, snapshots for model or user output, `doc-sync` for docs, build/hygiene and built smokes for published paths, and real-API e2e for provider behavior.
- Never default to the full suite or repeat a passing check for commit or push. CI owns exhaustive coverage and the platform matrix...
```

而 `dsh-pre-push-checks/SKILL.md` 里同一件事有 **1156 词**，包含：

- 怎么确认当前分支和 PR base（含具体命令）
- 五类改动面各自对应什么证据（每类一段）
- 覆盖率如何按源文件收窄（含 `--coverage.include` 完整用法）
- `vitest related` 的能力边界（哪些依赖它发现不了）
- 改写历史的 push 如何用 `--force-with-lease=<branch>:<oid>`
- `gh stack sync` 之后的四步补救验证
- 检查失败时如何证明「这是环境问题」

**这就是分工**：AGENTS.md 说「**必须做**」并给出指针；SKILL.md 说「**具体怎么做**」。

### 对比二：AGENTS.md 明确把某类内容推出去

`docs/AGENTS.md` 的层级表里有一行专门定义 skills 的职责边界：

| Tier | Job | Does NOT belong there |
|---|---|---|
| Skills (`.agents/skills/`) | Reusable workflows and specialized decision standards | Product and runtime contracts (→ docs or source) |

同一张表里对根 `AGENTS.md` 的规定是：

| Tier | Job | Does NOT belong there |
|---|---|---|
| Root `AGENTS.md` | Standing orders: rules an agent needs in context in **every session**, **one to three lines each**, linking its home | **Stories, worked examples, situational procedures**, anything restated from a linked home |

**关键词是 `situational procedures`（情境化流程）**：只有在特定情境才需要的流程，被明确禁止写进第 1 层。这不是风格偏好，是被门禁的词数上限逼出来的必然分工。

---

## 3. 核心机制：两级 context 预算

这是第 2 层**最能打的一个点**，值得单独讲清楚。

### 问题

AI 的 context 是稀缺资源。而一个成熟仓库的流程知识远超 context 能容纳的量（这里是 16957 词，且会持续增长）。两个朴素解法都不行：

- **全塞进去**：挤爆 context，且大部分内容与当前任务无关，纯属噪音。
- **不写**：AI 每次凭通用经验瞎猜，回到第 0 层。

### 解法：把知识拆成「目录」和「正文」两级

harness 自己的 skill 子系统实现了这个机制。看 `docs/subsystems/skills.md` 里对模型可见内容的规定：

> The catalog contains sorted skill `name` and normalized, XML-escaped `description` **only**; it **omits bodies, paths, sources, providers, and routing hints**.

翻译成大白话：

```
常驻进 context 的  ：11 × (name + description)   ← 目录，几百词
按需加载的        ：某一份 SKILL.md 的完整正文    ← 正文，千词级
永远不进 context 的：其余 10 份的正文
```

模型在会话开始时看到的是一份**目录**（通过 `<system-reminder>` 注入，含 `<available_skills>` 标签），它只知道「有 11 项能力，各自什么时候该用」。真正需要时才调用 `skill({ name })` 工具把某一份正文读进来。

### 代价（也是设计的精髓）

这个机制把全部压力转移到了一个地方：**目录必须能让模型准确判断「该不该展开」**。

如果 description 写得含糊，模型该用时不用、不该用时乱用，整个第 2 层就失效了。所以——

---

## 4. description 是这一层的命门

看实测：11 份 SKILL.md 的 `description` 字段词数在 **21 到 80 词**之间。

| skill | description 词数 |
|---|---|
| `record-browser-gif` | 80 |
| `dsh-merging-stacked-prs` | 76 |
| `dsh-trim-cot-leakage` | 68 |
| `dsh-doc-standards` | 53 |
| `dsh-archive-agent-notes` | 48 |
| `dsh-pre-push-checks` | 47 |
| `dsh-find-simplifications` | 45 |
| `dsh-doc-site-sync` | 41 |
| `dsh-prose-standard` | 37 |
| `dsh-code-review` | 35 |
| `dsh-translate-docs` | 21 |

**几十词，用来写什么？** 关键发现：**它们几乎全部以 `Use when` 开头，写的是「触发条件」而不是「功能介绍」。**

对比两种写法：

```
❌ 功能介绍式（无用）
description: A skill for running pre-push checks.

✅ 触发条件式（实际写法）
description: Use before pushing, force-pushing, marking ready for review,
or claiming checks pass on a deepseek-harness branch, and immediately after
gh stack sync publishes rewritten branches, to select the smallest tests and
checks that cover the outgoing or just-published diff without reflexively
running the full repository suite.
```

第二种写法把**触发时机穷举了**：push / force-push / 标记 ready for review / 声称检查通过 / `gh stack sync` 之后。模型不需要理解这个 skill 是什么，只需要模式匹配「我现在是不是在做这几件事之一」。

再看 `dsh-trim-cot-leakage` 的 description——它甚至把**要抓的特征字符串**都列进了触发条件：

> ...dead design-session citations such as `(decision N)`, audit item codes, or `§N` of uncommitted drafts; change narration such as **"used to"**, **"no longer"**, **"this cut"**; stack or review vantage (**"a later PR in this stack"**, **"rejected in review"**)...

这样模型在读到代码里出现 `used to` 时就能联想到这个 skill，不必先加载正文才知道它管这个。

### 配套的工程约束

`catalogDescriptionMaxLength` 是消费端配置，**默认 500、整数最小值 3**：超长的 description 会被规范化截断。也就是说「用 description 偷偷塞正文」这条路被堵死了。

> **落地要点**：写 skill 时，`description` 花的心思应该**不低于**正文。正文写错了模型加载后还有机会自我纠正；description 写错了模型根本不会加载它。

---

## 5. SKILL.md 的解剖

### frontmatter：四个字段

```yaml
---
name: dsh-translate-docs
description: Manually run the extended DeepSeek Harness bilingual-document workflow...
disable-model-invocation: true
user-invocable: true
---
```

| 字段 | 作用 | 缺省 |
|---|---|---|
| `name` | kebab-case 标识符，须匹配 `^[a-z0-9]+(?:-[a-z0-9]+)*$` | 必填 |
| `description` | 路由用的触发条件（见上节） | 必填 |
| `disable-model-invocation` | `true` = 禁止模型自动调用 | `false` |
| `user-invocable` | `false` = 人也不能手动调 | `true` |

**四种组合都有意义**（`docs/subsystems/skills.md`）：

```
{ model: true,  user: true  }  默认：模型自动 + 人手动都行
{ model: true,  user: false }  只给模型
{ model: false, user: true  }  只给人手动触发    ← dsh-translate-docs
{ model: false, user: false }  只能被受信任的 ctx.skills.get() 调用
```

**11 份里只有 1 份是 manual-only**：`dsh-translate-docs`。它的正文第一节直接叫「Invocation boundary」，写明：

> Run this extended workflow **only when the user explicitly invokes `dsh-translate-docs` by name**. Never select or load it for ordinary documentation work from another skill or from an inferred translation need; routine translation follows the one-shot one-pass rule in `docs/AGENTS.md`.

**为什么单独把它锁住**：翻译是个消耗巨大、容易被误触发的重流程（1399 词的完整双语工作流）。日常改一句中文文档不需要它，AGENTS.md 里的轻量路径就够了。这条边界不靠「希望模型识趣」，靠元数据硬关。

### 正文：没有统一模板，但有稳定的骨架惯例

11 份 skill 结构不完全一致（这本身是刻意的——见第 9 节），但反复出现这些节：

```
# <动作导向的标题>          「Reviewing a DSH PR」「Landing an official GitHub PR stack」
                            —— 不叫「PR Review Skill」，用动词
开篇一段                    定位 + 边界声明 + 指向 sources of truth
## Sources of truth         列出权威文档，明确「读它，不要在这儿重复总结」
## <核心判断标准>            这个 skill 独有的决策规则
## Workflow                 编号步骤
## Validation / Reporting    跑什么检查、怎么汇报
```

注意 `dsh-doc-standards` 里那句自我约束：

> ## Sources of truth (**read, don't re-summarize**)

**skill 不复制权威内容，只做路由和判断**。这是下一节要讲的引用图的基础。

---

## 6. 11 个 skill 全景

按管的事情分五类看，比按字母序清楚：

### A. 证据纪律（怎么证明代码是对的）

| skill | 管什么 |
|---|---|
| `dsh-pre-push-checks` | push 前按改动面选最小验证集，禁止反射性跑全量 |
| `dsh-code-review` | review 只查机器查不了的语义（第 6 层详解） |

### B. 文字纪律（怎么写字）

| skill | 管什么 |
|---|---|
| `dsh-prose-standard` | 必需的文档覆盖面 + 编辑判断（完整命题规则） |
| `dsh-doc-standards` | 文档放哪儿、层级、词数预算、验证 |
| `dsh-trim-cot-leakage` | 清理「思维链泄漏」——读起来像会话记录的文字 |
| `dsh-translate-docs` | 完整双语工作流（manual-only） |

### C. 决策记录纪律（第 3 层的操作面）

| skill | 管什么 |
|---|---|
| `dsh-archive-agent-notes` | Agent Note 的审计、归档、删除判据 |
| `dsh-find-simplifications` | 找可删可折叠的东西，写成 proposed Note |

### D. Git 与发布流程

| skill | 管什么 |
|---|---|
| `dsh-merging-stacked-prs` | 用 GitHub 原生 stack 落地依赖 PR 链 |
| `dsh-doc-site-sync` | 文档站点 manifest 与页面发布 |

### E. 证据产出工具

| skill | 管什么 |
|---|---|
| `record-browser-gif` | 录制 UI 演示 GIF（改 GUI 的 PR 必须附） |

**这个分类本身有信息量**：11 个里有 **6 个**（B 类 4 个 + C 类 2 个）在管「文字」。一个代码仓库把超过半数的操作手册用于约束文字生产——这直接呼应第 3 层（notes）和第 1 层的文档纪律。**在 agent-first 的仓库里，文字就是生产资料。**

---

## 7. 重点拆解四个 skill

### 7.1 `dsh-pre-push-checks`：证据匹配

核心一句：**按改动面选最小验证集，禁止默认全量跑。**

```
包/脚本行为            → 跑 owning vitest 文件（聚焦测试名）
文档/Agent Notes/目录   → pnpm run doc-sync
模型/编辑器/CLI 可见输出 → 跑聚焦的 keyless snapshot
包清单/导出/构建配置    → pnpm run build + hygiene + built smoke
真实 provider/agent 行为 → pnpm run test:e2e（需 key，绝不打印 secret）
```

三句关键约束：

> **There is no universal local baseline beyond the hooks.**
> （除 hook 之外没有「本地必跑基线」——每次都要按 diff 重新选。）

> **Do not manually repeat a passing check merely because commit or push follows.** In particular, do not run typecheck immediately before pushing solely to duplicate the pre-push hook.
> （不要因为要 commit/push 就重复跑一个已通过的检查。）

> **CI owns exhaustive coverage and the platform matrix.**
> （全量覆盖和平台矩阵是 CI 的事，不是本地的事。）

**一个容易忽略但很精彩的细节**：它区分了「选哪些测试跑」和「测哪些源文件的覆盖率」是两件事——

```sh
pnpm exec vitest run packages/<group>/<package>/tests/<behavior>.spec.ts \
  --coverage \
  --coverage.include='packages/<group>/<package>/src/**/*.ts'
```

因为仓库默认对所有 `packages/*/*/src/**/*.ts` 测覆盖率，只用文件过滤器跑聚焦测试会导致覆盖率大面积不达标。所以要**同时收窄两边**。

并且立刻堵住了滥用口子：

> Do **not** use `--passWithNoTests`, lower coverage thresholds, or narrow `--coverage.include` merely to hide an uncovered affected file.

**为什么这条最能戳中听众**：所有人都受过「改一行跑半小时」的苦，也都干过「跑不过就把阈值调低」的事。这份 skill 两头都堵了。

### 7.2 `dsh-prose-standard`：完整命题规则

这份 skill 的核心是一条反直觉的规则——**它不是「精简文档」的 skill**。

> Write enough to preserve the contract, then remove reasoning transcripts, repetition, and decoration.
> ...
> **A smaller word count alone is not an improvement.**

它要求编辑前先枚举段落里的每一个命题，并逐项保留：

```
- actor and action                      谁做什么
- condition, timing, and ordering       条件、时机、顺序
- modality such as must, may, or never  情态（必须/可以/绝不）
- negative guarantee and exception      否定性保证和例外
- ownership, side effect, failure mode  归属、副作用、失败模式
```

**这条规则的价值在于它防的是 AI 的一个典型故障**：让 AI「精简文档」，它很容易把 `must not` 删成 `should`、把「除 X 外都支持」删成「都支持」——字数确实少了，但契约被悄悄改写了。「完整命题」把这件事变成了可检查的：改完之后逐条对，每个命题还在不在。

它还给出了 **12 种文字场景各自的必需覆盖面**（public JSDoc / 内部注释 / 模块注释 / 测试 / cookbook / README / Agent Notes / postmortem / skills / 示例配置 / prompt 与可见字符串 / 诊断信息），每类写明「必须包含什么」和「必须删掉什么」。例如：

> **Prompts and visible strings:** treat wording as behavior. Inspect generated output and run behavior validation or state why no snapshot applies.
> （提示词和可见字符串：**措辞就是行为**。改了要么跑快照验证，要么说明为什么不适用。）

### 7.3 `dsh-trim-cot-leakage`：一句话判据

这份 skill 抓的是一类非常具体的 AI 产出污染：**「思维链泄漏」——文字的视角是「写作时那次会话」而不是「仓库现在的状态」**。

它的判据只有一句，叫 **The one test**：

> **Could a reader at HEAD, with no access to any session transcript, PR thread, or uncommitted draft, resolve every reference and verify every claim?**
> （一个只能看到当前 HEAD、拿不到任何会话记录/PR 讨论/未提交草稿的读者，能否解析每一处引用、验证每一个断言？）

不能，就是泄漏。它列了 8 类：

```
1. 死掉的设计会话引用    (decision 7)、(audit C2)、design §4.7、phase T4
2. Stack / PR 视角       "a later PR in this stack"、"this PR adds"
3. 变更叙述与版本戳      "used to"、"no longer"、"the old X"、"v1"、"today"
4. Review 编排           "Rejected in review:"、"the reviewer confirmed"
5. 对 reviewer 的辩解     "the cast is safe — it simply…"、"this is correct because…"
6. 复述与推导记录        控制流叙述、测试走读、显然分支的证明
7. 对冲与计划残留        "probably fine for now"、"should be enough"
8. 写作语言残留          英文文档里漏的中文片段（端、设计稿、---- 私有 ----）
```

**第 5 类值得单独念出来**——它的判据极精辟：

> A comment arguing its own correctness addresses **a reviewer, not a maintainer**.
> （一段为自己正确性辩护的注释，说话对象是 reviewer，不是维护者。）

而修复方式**不是删掉了事**：

> The fix is **never deletion alone** when a passage carries factual clauses — restate each so it stands at HEAD, then delete the transcript around it.

它还专门列了「**什么不是泄漏**」（issue 引用 `#1470`、suppression 的理由、`without X, Y happens` 这类反事实回归标注、`(measured: 512 nests ≈ 0.15s)` 这类实测数据、运行时的 old/new 状态、外部标准的 §引用），并给出理由：

> Unaided citation passes fail **in both directions** by deleting durable references and keeping dead ones.
> （不带判据的引用清理会两头出错：把该留的删了，把该删的留了。）

**这是「反向规则」的典范**：一条规则如果只说「删什么」，执行者必然过度删除。必须同时写明「留什么」。

### 7.4 `dsh-find-simplifications`：给「删代码」立证据标准

最长的 skill（2038 词），管一件反直觉的事：**怎么严谨地提议删东西**。

它先定义什么算**强候选**（有真实证据表明现设计的成本超过收益）：

```
- 某个公开方法/事件/配置项/包/持久化事件没有生产消费者
- 只有测试和文档在用，且它们钉住的行为不承重
- 两种表示镜像同一个事实（尤其跨持久事件和瞬时 agent/* 事件）
- seam 上有所有实现都必须支持、但没人调用的方法
- 手搓代码重实现了成熟外部包或 Node 内建已提供的能力
```

然后定义什么算**弱候选**（不值得写 Note）：

> deleting one typo, running `knip` once, removing an intentionally documented backend/adapter, or flagging "this looks complex" **without call-site proof**.

**最关键的一条自我限制**——它明确划出了「不许碰」的保护区：

> Treat dual LLM adapters and dual persistence backends as **intentional by default**. Do not propose deleting either twin/backend as "low effort" **unless the user explicitly overrides that constraint**.

**为什么这条重要**：AI 做简化时最危险的行为是「看起来重复，删一个」——而双适配器、双持久化后端恰恰是刻意保留的冗余设计，有 Agent Note 记录其理由。这条规则直接把「已记录的设计决策」变成了 AI 提议删除时的默认否决项。

它还规定了「拒绝或降级候选」的四个条件，其中一条把第 3 层拉了进来：

> The API is explicitly justified by **an implemented Agent Note** or a hard-won defensive pattern, and the new evidence **does not beat that reason**.

——**不是不能推翻已记录的决策，是推翻它的证据必须比原理由更强**。这就是第 3 层 notes 的实际用法：不是档案馆，是当前决策的举证责任基线。

---

## 8. skill 之间的引用图：一条规则一个家

11 份 skill 不是 11 个孤岛。实测互相引用关系：

```
dsh-code-review ──────────► dsh-prose-standard
dsh-doc-standards ────────► dsh-prose-standard, dsh-trim-cot-leakage,
                            dsh-find-simplifications
dsh-prose-standard ───────► dsh-doc-standards, dsh-trim-cot-leakage
dsh-trim-cot-leakage ─────► dsh-prose-standard, dsh-translate-docs
dsh-translate-docs ───────► dsh-prose-standard, dsh-code-review,
                            dsh-pre-push-checks
dsh-find-simplifications ─► dsh-archive-agent-notes
dsh-archive-agent-notes ──► dsh-pre-push-checks
dsh-doc-site-sync ────────► dsh-doc-standards, dsh-pre-push-checks
dsh-pre-push-checks ──────► （无出边，是叶子）
dsh-merging-stacked-prs ──► （无出边）
record-browser-gif ───────► （无出边）
```

**引用不是「相关阅读」，是明确的职责移交**。看 `dsh-prose-standard` 开头怎么写的：

> **This skill owns editorial judgment and required prose coverage**; use [dsh-doc-standards] for **placement, budgets, bilingual pairs, and documentation gates**, and [dsh-trim-cot-leakage] for **hunting and fixing reasoning-transcript leakage**.

三份文字类 skill 明确切分：

| skill | 拥有什么 |
|---|---|
| `dsh-prose-standard` | 编辑判断 + 必需覆盖面（写多少算够） |
| `dsh-doc-standards` | 位置 + 预算 + 双语配对 + 门禁（放哪儿、多长） |
| `dsh-trim-cot-leakage` | CoT 泄漏的分类学与召回策略（删什么） |

`dsh-trim-cot-leakage` 甚至标了强制前置依赖：

> **REQUIRED BACKGROUND:** [dsh-prose-standard] owns **the complete-proposition rule this skill applies**.

**这解决了一个真实的腐化问题**：如果「完整命题规则」在 11 份 skill 里各写一遍，三个月后就会有 11 个互相矛盾的版本。写一遍 + 10 个指针，是唯一能长期维持一致的结构。

同样的原则也体现在 skill 引用外部文档的方式上：

> `.agents/notes/README.md` — ...the header block, per-lifecycle skeleton, and Alternatives-considered mandate, gated by `verify-agent-note-format`
>
> Follow the deletion rule in the [Agent Note rules]; **do not duplicate or weaken it here.**

**「不要在这里复制或弱化它」**——这句话直接写在 skill 正文里，是对未来编辑者（人或 AI）的防御。

---

## 9. 「guidance, not a script」——反清单化的自我约束

11 份 skill 里有 6 份显式写了这句免责声明。位置都很扎眼：

`dsh-code-review` 的正文**第一句**：

> **This skill is guidance, not a complete checklist.**

`dsh-find-simplifications` 开篇：

> It is **guidance, not a checklist**: follow the code, keep judgment active, and prefer **a few well-proven candidates over a pile of thin guesses**.

`dsh-doc-standards`：

> It is guidance, not a script; ...and **never treat length alone as a defect**.

### 为什么要专门声明这个

因为 AI 面对清单有一个确定的失败模式：**把清单当成完成条件**。给它 20 项检查，它会逐项打勾然后宣布通过，而不去想清单没覆盖的那类问题。

`dsh-code-review` 的处理方式很值得学——它直接给出了优先级和「好 review 的样子」：

> Prioritize correctness, lifecycle, security, and broken required behavior over style; **a short review with one substantiated blocker is better than a list of nits.**
> （一条有实据的阻断性问题，胜过一堆吹毛求疵。）

`dsh-trim-cot-leakage` 更进一步，明确说自己的搜索模式（recall batteries）**不是定义**：

> The batteries are **probes, not the definition** — each review round of the original purge found cases the batteries missed, so **also read the densest prose in scope without a pattern in hand**.
> （召回策略只是探针，不是定义——原始清理的每一轮 review 都发现了探针漏掉的案例，所以还要在不带任何模式的情况下通读范围内最密的文字。）

它甚至给出了「**过度纠正陷阱**」清单，要求删除前必查：

> trims that **flip an obligation into an endorsement**, **promote a hypothetical to a shipped feature**, **delete a true fact**, or **drop provenance**.

**这一条是整个第 2 层的元设计**：skill 的作用是**校准判断**，不是**替代判断**。承认这一点，才能防止「跑完流程 = 质量达标」的假象。

---

## 10. 三种附件：references / scripts / agents

skill 不只是一份 Markdown。目录结构支持三类附件，实测分布：

```
.agents/skills/
├── dsh-archive-agent-notes/
│   ├── SKILL.md
│   └── agents/openai.yaml              ← 跨产品调用元数据
├── dsh-prose-standard/
│   ├── SKILL.md
│   ├── agents/openai.yaml
│   └── references/examples.md          ← 1220 词的校准样例
├── dsh-trim-cot-leakage/
│   ├── SKILL.md
│   └── references/
│       ├── examples.md                 ← 1649 词
│       └── recall-batteries.md         ← 510 词的搜索命令库
└── record-browser-gif/
    ├── SKILL.md
    ├── agents/openai.yaml
    └── scripts/encode_gif.py           ← 可执行工具
```

### `references/`：第三级按需加载

这是**同一个「按需加载」思想又用了一层**。`dsh-trim-cot-leakage` 正文 1005 词，但它的 references 加起来 2159 词——比正文还多一倍。

正文里这样引用它们：

> For calibration or unfamiliar cases, read [the distilled examples](references/examples.md).
>
> Audit read-only first: run the [recall batteries](references/recall-batteries.md) (with `--hidden` so `.agents/` is searched), then judge every hit semantically.

**三级加载**：

```
目录（description）  →  正文（SKILL.md）  →  样例库（references/）
  几十词，常驻          千词，触发时加载      千词，判断不准时才加载
```

`dsh-prose-standard` 更进一步，把 references 变成了**学习闭环的写入端**：

> After the user decides, **distill the principle and versions into [the examples](references/examples.md)**, without PR history or reviewer narration, and **apply the learned rule to every analogous passage in scope**.

——人做完一次仲裁后，结论要沉淀回样例库。**这是第 2 层自己的自我更新机制**：模糊判断被裁决一次，就变成下次的校准锚点。

### `agents/openai.yaml`：跨客户端的调用元数据

6 份 skill 有这个文件。内容极简：

```yaml
interface:
  display_name: "DSH Pre-Push Checks"
  short_description: "Run the relevant DeepSeek Harness checks before push"
  default_prompt: "Use $dsh-pre-push-checks before pushing this DeepSeek Harness branch."
```

`dsh-translate-docs` 多一段 policy：

```yaml
policy:
  allow_implicit_invocation: false
```

**这就是 manual-only 在另一个产品侧的表达**。同一个语义，Claude Code 侧写 `disable-model-invocation: true`，Codex 侧写 `allow_implicit_invocation: false`。两个字段的真值恰好相反——正是这种「同义反向」最容易改一边忘一边。所以有了下一节。

### `scripts/`：skill 可以携带可执行工具

`record-browser-gif/scripts/encode_gif.py`。skill 不止是指令，可以带确定性的实现。它的定位是「deterministic encoding」——把「录出来的 GIF 时长/尺寸/体积可复现」这件事交给脚本，而不是让模型每次自己拼 ffmpeg 参数。

---

## 11. 连元数据都有门禁

第 4 层（scripts）在这一层的落点：`scripts/verify-skill-invocation-metadata.ts`。

它做的事：**扫描每个带 `agents/openai.yaml` 的 skill 目录，确保两个产品的调用策略语义一致。**

核心逻辑（原文）：

```ts
const claudeManualOnly = disableModelInvocation === true
const codexManualOnly  = allowImplicitInvocation === false

if (claudeManualOnly !== codexManualOnly) {
  violations.push(
    `${relativeRoot}: Claude Code manual-only=${String(claudeManualOnly)}`
    + ` but Codex manual-only=${String(codexManualOnly)}`,
  )
}
if (claudeManualOnly && userInvocable === false) {
  violations.push(`${relativeRoot}/SKILL.md: a manual-only skill must remain user-invocable`)
}
```

三层检查：

1. **格式**：`disable-model-invocation` / `user-invocable` / `allow_implicit_invocation` 必须是 boolean（写成 `"true"` 字符串会被拒）；frontmatter 必须是合法 YAML 对象。
2. **一致性**：Claude 侧和 Codex 侧算出的 manual-only 必须相等。
3. **逻辑自洽**：一个 manual-only 的 skill 必须保持 user-invocable——否则它就谁都调不到了，是死代码。

通过时输出：

```
verify-skill-invocation-metadata: 6 cross-product skill policy pair(s) aligned.
```

**这个门禁为什么值得讲**：它检查的东西极其细微（一个 boolean 的跨文件语义一致性），根本不影响代码能不能跑。但如果错了，故障表现是「某个 skill 在某个客户端悄悄失效」——**没有报错，只是不再被调用**。这类静默失效是人和 review 最容易漏的，正是自动门禁的最佳适用场景。

> 顺带说明第 3 节那个「11 个 skill 只有 1 个 manual-only」的判断是怎么核实的：直接跑这个门禁看它报告了几对，再逐个读 frontmatter。**这就是「每一层都可验证」的实际体感**。

---

## 12. `.claude/skills` 是软链，不是副本

一个常见误解值得纠正。很多介绍说 harness「把 skill 镜像到 `.claude/skills/`」，听起来像有两份内容需要同步。实测：

```
$ ls -la .claude/
skills -> ../.agents/skills

$ diff -rq .agents/skills .claude/skills
（无输出，完全相同）
```

**`.claude/skills` 是指向 `.agents/skills` 的符号链接**。

**为什么这个细节重要**：如果真是两份副本，就多出一个「同步」问题——而这个仓库的一贯做法是**从结构上消灭同步需求**，而不是加一个检查同步的门禁。软链让「两份不一致」在物理上不可能发生。

同样的手法在第 1 层也出现过：`CLAUDE.md -> AGENTS.md` 也是软链。

> **一条可迁移的原则**：**能用软链消灭的一致性问题，不要用门禁去检查。** 门禁是给「无法从结构上消灭」的一致性问题准备的（比如上一节那个跨产品 boolean 语义）。

---

## 13. Dogfooding：harness 自己读自己的 skills

有一个容易被忽略的闭环：**DeepSeek Harness 本身就是一个 agent harness，它自己实现了 skill 加载机制**。看 `docs/subsystems/skills.md` 里本地 provider 的发现优先级表：

| Rank | Source | Root |
|---|---|---|
| 100 | `project-dsh` | `<projectRoot>/.dsh/skills` |
| **200** | **`project-agents`** | **`<projectRoot>/.agents/skills`** |
| 300 | `custom` | `Config.customSkillDirs` |
| 400 | `user-dsh` | `<dshHome>/skills` |
| 500 | `user-agents` | `<agentsHome>/skills` |
| 600 | `bundled` | `Config.bundledSkillDir` |

**Rank 200 就是本文讲的那个目录。** 也就是说：用 harness 在 harness 仓库里干活时，它会按 rank 200 加载这 11 个 skill——**开发这个工具所用的流程知识，就是这个工具的一等公民数据**。

这个 dogfooding 有实际后果，不只是好听：

- **skill 加载的边缘情况会被自己踩到**。比如「监视器失效时怎么办」这类问题，团队自己每天在受影响。
- 所以子系统里有这些设计：Chokidar 监视 root 的增删；缺失的 root 从最近的已存在祖先开始逐段跟进；模型侧的 `write`/`edit` 命中目录时同步失效缓存；监视器失败只把当次观察标为 incomplete，**不隐藏仍可直接加载的候选**。
- 目录变更通过 `agent.inject()` 追加**完整替换**的持久消息；删空则显式发一条空替换；incomplete 的快照保留上一次 good 视图。

**分享时可以用这个点回应「他们是不是过度设计」的质疑**：这些机制不是想象出来的，是「一边用一边改自己的 skill 文件」这种工作方式必然会撞上的问题。

---

## 14. 怎么抄：给自己项目的落地步骤

### 第 1 步：别一次写 11 个，从 1 个开始

选**最痛的那个重复劳动**。判断标准：你是不是已经在 AGENTS.md 或 README 里写了一段流程，但它一直在变长、且只在特定情境下有用？那段就该拆成 skill。

首选候选（按性价比排序）：

1. **「push / 提 PR 前跑什么」**——几乎所有团队都有这个问题，且收益立刻可见。
2. **「review 时看什么」**——把资深同事的隐性判断显式化。
3. **「加一个新模块要改哪些地方」**——新人和 AI 犯同类错误的高发区。

### 第 2 步：先写 description，写 3 个版本挑一个

这是最反直觉的建议，但是这一层的命门（第 4 节）。检验标准：

> 把 description 单独拿给一个不了解这个 skill 的人看，问他：**「你现在做的事该不该用它？」** 如果他要反问「这个 skill 具体干什么」，你的 description 就写失败了。

写法要求：
- 以 `Use when` 开头，**穷举触发场景**，不写功能介绍。
- 把**特征关键词**写进去（像 `dsh-trim-cot-leakage` 把 `"used to"`、`"no longer"` 直接列出来）。
- 控制在 500 字符以内。

### 第 3 步：正文只写「这个 skill 独有的判断」

三条硬规矩：

1. **不复制权威文档**。列 sources of truth 然后写「read, don't re-summarize」。
2. **写清边界**。加一句「guidance, not a checklist」，并给出优先级（什么算严重问题、什么算 nit）。
3. **正反都写**。只写「要删什么」必然导致过度删除，必须配一节「什么不算问题」。

### 第 4 步：新增第 2 个 skill 时，立刻处理重叠

两个 skill 一旦有重叠内容，**当场决定谁拥有它**，另一边改成链接。不要留到「以后整理」——那个「以后」不会来，而两份副本会各自演化。

判断句式照抄这个：

> This skill owns **X**; use `<other-skill>` for **Y**.

### 第 5 步：需要跨客户端时才加 `agents/`，能软链就软链

- 只用一个 agent 客户端：不需要 `agents/openai.yaml`。
- 多客户端读同一份内容：**用软链**（`.claude/skills -> ../.agents/skills`），不要复制。
- 只有当两个客户端**语义相同但字段不同**（无法软链）时，才加元数据文件，并**同时加一个校验一致性的门禁**——参考第 11 节，二十行脚本的量级。

### 第 6 步：给 skill 加一个「学习写入口」

这是最容易被跳过、但长期价值最高的一步。参照 `dsh-prose-standard`：

> 每次人对一个模糊判断做了仲裁 → 把「原则 + 两三个版本 + 选了哪个及为什么」写进 `references/examples.md` → 下次 AI 遇到同类情况先读样例。

没有这一步，skill 就是一份静态文档，会随着现实漂移而失效；有了这一步，它是一个**会随争议积累而变准的校准器**。

---

## 附：一页速查

| 问题 | 答案 |
|---|---|
| 在哪 | `.agents/skills/<name>/SKILL.md`（`.claude/skills` 是软链） |
| 多少个 | 11 个 |
| 多大 | 正文 13578 词，含附件 16957 词（AGENTS.md 的 7–8.8 倍） |
| 有词数预算吗 | **没有**——因为不常驻 context |
| 怎么被找到 | 会话目录只含 `name` + `description`，正文按需加载 |
| description 怎么写 | `Use when ...` 穷举触发条件，≤ 500 字符 |
| 能关掉自动调用吗 | 能，`disable-model-invocation: true`（11 个里 1 个） |
| 谁保证跨客户端一致 | `scripts/verify-skill-invocation-metadata.ts` |
| 最能打的一个点 | 两级 context 预算：常驻的必须极简，按需的可以极详 |
| 最容易抄错的地方 | 把 description 当功能介绍写；用复制代替软链；只写「删什么」不写「留什么」 |

---

> [← 返回六层全景图](how-deepseek-does-ai-coding.md)　|　[第 1 层：AGENTS.md →](layer1-agents-md.md)
