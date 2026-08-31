# 第 3 层详解：Agent Notes —— 「为什么」和「放弃了什么」

> 总览见 [how-deepseek-does-ai-coding.md](how-deepseek-does-ai-coding.md)。本文是第 3 层的完整拆解。
>
> **一句话**：Agent Note 是**由 AI 写、被机器校验格式、生命周期编码在路径里**的决策记录。它存在的唯一目的是**防止同一个决策被反复重新争论**。

---

## 0. 它解决什么问题

代码和文档能承载「是什么」，承载不了两件事：

| 信息 | 住在哪 |
|---|---|
| 这段代码**是什么** | 代码本身 + JSDoc |
| 这个包的**契约** | package README |
| 怎么**加一个** | cookbook（第 7 层） |
| **为什么**这么定 | **Agent Note** |
| **放弃了什么** | **Agent Note** ← 别处都没有 |

第二行「放弃了什么」是关键。`.agents/notes/README.md` 的原话：

> An **Agent Note** records a decision or proposal that affects this codebase — the *why* and **what we gave up**, the parts code and docs can't carry.

**在 agent-first 的仓库里这条格外重要**：人类团队里「我们试过 X，不行」可以靠口头传承；AI 每次会话都是新的，它**没有历史记忆**。如果不写下来，agent 会一次又一次提出同一个被否决过的方案，而 reviewer 要一次又一次地驳回它。

---

## 1. 规模：551 份活跃 + 142 份归档

实测（英文正文，不含 `.zh.md` 与 `AGENTS.md`）：

| 生命周期 | 数量 | 相当于 |
|---|---|---|
| `proposed/` | 25 | **spec / RFC（未落地）** |
| `implemented/` | 515 | ADR（已落地） |
| `rejected/` | 11 | 被否提案（冻结） |
| **活跃合计** | **551** | |
| `archived/` | 142 | 历史快照（永久冻结） |
| **总计** | **693** | |

按 class 分布（`implemented/`）：

| class | 数量 | 占比 |
|---|---|---|
| `feature` | 172 | 33% |
| `architecture` | 130 | 25% |
| `bug-fix` | 82 | 16% |
| `process` | 69 | 13% |
| `simplification` | 48 | 9% |
| `testing` | 13 | 3% |

> **注意 `simplification` 有 48 份**——**专门记录「删掉了什么」的决策**。绝大多数团队的决策记录里只有「加了什么」，删除是悄悄发生的、不留痕的。这里删除和新增享有同等的记录待遇，因为「为什么删」在未来同样会被质疑（「这个功能当年为什么没了？」）。

---

## 2. 两个轴，都编码在路径里

```
.agents/notes/{lifecycle}/{class}/yyyy-mm-dd-topic-title.md
                  ↑          ↑         ↑
                 状态      决策种类   首次提出的日期
```

### 轴一：lifecycle（状态）

**状态不是文件里的一个字段，而是它所在的文件夹。** 状态变化 = **移动文件**。

| 文件夹 | 含义 |
|---|---|
| `proposed/` | 实现前评审的提案；尚未建成（或只建了一部分） |
| `implemented/` | 决策已落地。**且必须与实际落地的代码保持同步** |
| `rejected/` | 提案被考虑后否决。**只在它的理由能阻止一个诱人且有意义的错误时才保留** |
| `archived/` | 已落地但未来价值低的记录，**永久冻结** |

`implemented/` 那条「保持同步」的要求最容易被忽略，但它是这层不腐烂的关键（`implemented/AGENTS.md`）：

> Keep paths, symbols, defaults, and mechanisms current **in the same change that alters them. Rewrite stale facts in place; do not append change history.**

**「原地重写，不许追加变更历史」**——所以 `implemented/` note 读起来永远是「当前事实」，不是考古现场。

而紧接着有一条极其重要的限制：

> ### This is not a license to rewrite the *decision*
> Update factual realization in place. **A reversal of the decision or its rationale requires a new Agent Note and cross-link.**

**事实可以原地改，决策不可以。** 改路径、改默认值、改符号名 → 原地更新；推翻决策 → 写新 note 并互链。这条区分让「保持同步」不会退化成「历史被静默改写」。

### 轴二：class（决策种类）—— 封闭集合

`scripts/agent-note-tree.ts` 里写死了六类：

```ts
export const AGENT_NOTE_CLASSES = ['feature', 'bug-fix', 'simplification', 'architecture', 'process', 'testing'] as const
```

注释明确了「加一类」的代价：

> Adding a class is a deliberate act: **extend this list AND the README's Classification section.** The gate rejects any folder not listed here.

各类的定义与最容易搞混的那条界线：

| class | 覆盖什么 |
|---|---|
| `feature` | 新的用户或模型可见能力 |
| `bug-fix` | 修缺陷，或补上 postmortem 暴露的缺口 |
| `simplification` | **删代码/行为/表面积**，不新增能力 |
| `architecture` | 关于**已发布源码**的结构决策——包如何关联、运行时词汇是什么 |
| `process` | 代码**周边**的工具、策略、工作流——门禁、包管理器、vendoring；**不是运行时行为** |
| `testing` | 测试基础设施与策略 |

> **`architecture` / `process` 的界线**：architecture 管**我们发布的源码**，process 管**周边的工具与工作流**。

还有一条「刻意缺席」值得单独讲：

> (**`refactor` is deliberately absent** — it overlaps `simplification`, whose discriminator, "does observable behavior change?", already covers it.)

**明确记录「为什么没有 refactor 这一类」。** 这是元层面的 alternatives considered——连分类法本身的取舍都留了痕，否则下一个 agent 一定会问「为什么不加个 refactor？」。

---

## 3. 最反直觉、也最有价值的一条：`## Alternatives considered` 强制

**每一份 Agent Note 都必须有这一节。** 原文：

> Every Agent Note carries an `## Alternatives considered` section: each genuine alternative and why it lost, one bold-led paragraph per alternative or a `### Why not <X>?` subsection per contested one. **A decision recorded without what it beat invites re-litigation — the failure Agent Notes exist to prevent.**

翻译成一句话：**记录了决策却不记录它击败了什么，等于邀请反复争论。**

### 但它有一条更了不起的配套规则

> **Alternatives are recorded, never invented.**

**备选方案是「被记录」的，不是「被编造」的。** 那些格式规范生效（2026-07-05）之前、备选方案已经无法从记录里重建的老 note，必须携带一个**逐字精确**的注释：

```markdown
<!-- agent-note-format: alternatives-not-recorded (pre-format Agent Note) -->
```

而且门禁只对该日期之前的文件接受这个豁免（`verify-agent-note-format.ts`）：

```ts
if (hasGrandfather && note.date >= FORMAT_ADOPTED) fail(`the grandfather comment is only valid for Agent Notes dated before ${FORMAT_ADOPTED}`)
```

实测：**693 份里有 19 份携带这个豁免注释**。

> **这 19 份是整套体系里我最欣赏的一个设计**。换个团队的做法一定是：让 AI 把这 19 份的 Alternatives 补上——AI 完全能编出一段看起来很像回事的「我们考虑过 X，但因为 Y 放弃了」。而这里的选择是**在记录上留一个诚实的缺口**。
>
> 那份格式 note 自己的 Consequences 段写得很清楚：
>
> > Pre-format Agent Notes whose alternatives were not reconstructible carry the grandfather comment permanently — **an honest gap on the record rather than fabricated rationale**.
>
> **「宁可有个诚实的洞，也不要编造的理由。」** 这条原则的价值远超它管的那 19 个文件：一旦允许补编理由，整个决策记录体系的可信度就归零了——因为你再也无法区分哪条理由是真的。

---

## 4. 三种生命周期，三套骨架（时态不同）

同一个决策在不同状态下有**不同的必需小节**，因为它们的时态和用途不同。

### `proposed/`（= spec）

```markdown
## Problem
## Proposal              ← 可以用将来时
…bespoke sections…
## Alternatives considered
## Acceptance criteria   ← 什么可观测状态算做完
## Risks                 ← 可能出什么错 + 明知放弃了什么
```

`## Proposal` 里**允许**出现计划、迁移步骤、开放问题——因为工作还没建。

### `implemented/`（= ADR）

```markdown
## Problem
## Decision              ← 必须现在时，描述已落地的现实
…bespoke sections…
## Alternatives considered
## Consequences          ← 这个取舍付出了什么 AND 换来了什么
```

`## Consequences` 的定义值得注意：**「付出了什么 **和** 换来了什么」**——不是只记收益，也不是只记代价。

### `rejected/`（= 冻结的提案）

保留提案期的所有小节（包括 `## Acceptance criteria`、`## Plan`），**判决写在 `Status:` 行上**：

```
Status: rejected — <一行原因>
```

> `Status:` 是三种状态里**唯一带内容的**，理由是：**「a rejected Agent Note's verdict is the fact readers come for.」**（读者来看被否 note，要的就是那个判决。）

### `rejected/` 的实际样子——这些一行判决就是全部价值

实测的几条（截断）：

| note | `Status:` 行 |
|---|---|
| `evaluate-landstrip-for-windows-sandbox-rung` | rejected — landstrip **不够经受考验**（否决时是个几天前建立、单维护者、约 48 star 的项目）；一个承载安全不变量的依赖必须…… |
| `drop-durable-step-boundaries` | rejected — `step/end` 是「模型步骤已完成」的**持久化标志**，保留对称的 `step/start`/`step/end` 让崩溃修复不变量与转录检查比从相邻事件推断更清晰 |
| `truncate-interrupted-turns` | rejected — 单个 turn 可能包含大量真实工作，包括多个 step 和大量工具输出。**保留被中断的 turn 优于在加载时静默丢弃那段尾巴** |
| `prune-unused-skill-registry-api` | rejected — 运行时直接注册 skill 是**面向第三方插件的刻意扩展路径** |
| `fold-compaction-package-split` | rejected — 更多 compaction 后端已在计划中，所以 Service Definition 与 basic provider 包保持分离 |
| `builtin-timer-promises-for-hand-rolled-sleeps` | rejected — **实现（PR #679）证伪了等价前提**：vitest 的 fake clock 不拦截 `node:timers/promises`，所以这次替换的代价是…… |
| `dependency-swaps-rejected-by-nih-audit` | rejected — 下面每一项替换在证据面前都过不了「净简化」的门槛；**记录下来，这样这份调研不必从零重跑** |

> **把这七行连起来读，就明白这一层的真实价值了**。这些全都是「看起来显然应该做」的简化和替换——删掉没用的 API、用内置模块替换手写的 sleep、合并包、砍掉冗余事件。**每一个都会被下一个 agent 再次提出来**，因为从局部看它们确实都像是改进。
>
> 特别看最后两条：
> - `builtin-timer-promises`——**它是被实现证伪的**（「implementation falsified the parity premise」）。有人真去做了，做完发现前提不成立。**这个实验的结论被保存下来了**，而不是随着那个被关掉的 PR 一起消失。
> - `dependency-swaps-rejected-by-nih-audit`——**一次完整调研的结论被冻结，明确写着「so the survey is not re-run from scratch」**（这样这份调研不必从零重跑）。
>
> **这才是 `rejected/` 的正确用法：它不是垃圾桶，是「已经付过的学费」的存档。**

而且保留 `rejected/` note 有明确的**删除条件**，不是永久堆积：

> Keep it **only while its rationale prevents a tempting, meaningful mistake**; otherwise delete the complete triplet.

「诱人（tempting）且有意义（meaningful）的错误」——这个双重限定挡住了「万一以后有用」的囤积倾向。

---

## 5. 状态迁移是机械化改写，且门禁会拦

`proposed/` → `implemented/` 不是改个文件夹名就完事：

> Moving a file between lifecycle folders means updating the `Status:` line **and re-satisfying that folder's skeleton in the same change** — **the gate fails the move otherwise.**

具体的改写规则（README 原文）：

```
## Proposal            → 重写成现在时的 ## Decision
## Acceptance criteria ┐
## Risks               ┘→ 折叠进 ## Consequences
                         （或折成现在时的 ## Testing / ## Verification，
                           记录现在是什么在钉住这个行为）
计划                    → 丢掉，换成实际发布了什么
```

而且门禁**主动禁止**已落地 note 里出现提案期用语（`verify-agent-note-format.ts`）：

```ts
const BANNED_IMPLEMENTED = /^## (?:Proposal\b|Plan\b|Migration plan\b|Acceptance criteria\b)/i
```

报错信息本身就是指导：

> `` `## Proposal` is a proposal-era heading; an implemented Agent Note states what is (fold it into Decision/Consequences/Testing)

> **为什么这条很重要**：这防的是决策记录最常见的腐烂方式——**一份文档里前半段是「我们打算……」，后半段是「我们已经……」，读者无法判断哪部分是现实**。而 AI 写文档时最容易犯的正是这个错（它会保留提案原文再往后追加实现情况）。这里用一条正则彻底封死。
>
> 这也回应了 `docs/AGENTS.md` slop checklist 里那条：
> > **Spec-speak in `implemented/` Agent Notes**: "should", migration plans, acceptance checklists. An implemented Agent Note **describes what is**.

`proposed/` → `rejected/` 反而最简单：**只在 `Status:` 行加原因，然后冻结文件。**

---

## 6. 门禁在这一层做了什么（第 4 层的落点）

三个脚本 + 一个共享结构模块：

| 脚本 | 查什么 |
|---|---|
| `agent-note-tree.ts` | **共享的结构真相源**：封闭 lifecycle 集合、封闭 class 集合、路径深度、文件名格式 |
| `verify-agent-note-format.ts` | 头部三行、状态语法、必需小节、禁用小节、Alternatives 强制、豁免注释有效期 |
| `verify-agent-note-classification.ts` | class 文件夹合法性 |
| `verify-archived-agent-notes.ts` | 封闭 class 树、完整三元组、归档元数据、sidecar 哈希、**append-only 冻结内容清单** |

### `verify-agent-note-format.ts` 逐条查什么

它一共校验 **10 类**违规，值得逐条看，因为每一条都对应一种真实的腐烂方式：

```ts
1. 第 1 行必须是 `# Agent Note: <title>`
2. 第 2 行必须空
3. 第 3 行必须匹配所在文件夹的状态语法
4. 第 4 行必须空
5. 全文件只能有一个 `Status:` 行          ← 防「又加了一行新状态」
6. 第一个小节必须是 `## Problem`
7. 该 lifecycle 的必需小节不能缺
8. implemented 里不许出现提案期小节
9. Alternatives 小节与豁免注释不能同时存在，也不能都不存在
10. 豁免注释只对 2026-07-05 之前的文件有效
11. 不许出现已退役的技术债标记（`XXX: legacy ADR/RFC body format`）
```

有两个实现细节很讲究：

**① 代码围栏里的格式标记不算文档结构**

```ts
// Format tokens inside fenced examples are not document structure.
let inFence = false
const prose = lines.filter((l) => { if (l.startsWith('```')) { inFence = !inFence; return false } return !inFence })
```

**否则 README 里那些展示骨架的示例代码块自己就会把门禁弄红**。这类细节决定门禁是「能用」还是「大家都在加豁免」。

**② 退役的技术债标记被明确禁止「爬回来」**

```ts
/** The retired debt marker that flagged pre-format bodies; banned so it cannot creep back. */
const LEGACY_MARKERS = ['XXX: legacy ADR/RFC body format', 'XXX: legacy ADR/Agent Note body format']
```

历史上有 39 个这样的债务标记，格式统一后全部清除。**清完之后把这个标记本身设为禁词**——防的是有人（或 AI）从老 commit 里抄回一个带标记的旧格式。

> **这就是第 1 层「棘轮」思想在第 3 层的复现：清理完成后，把「回退」这条路本身封掉。**

### 归档的冻结机制

`archived/` 的规则极其严格：

> Once sealed, every archived triplet is **permanently frozen**. Do not edit, translate, reformat, update, move, or delete it, and **do not treat it as authority for current behavior**.

归档时**唯一允许的内容改动**是：

```
移动完整三元组（英文 / 中文 / sidecar）
保留 Status: implemented
在状态行下方插入相同的 Archived: YYYY-MM-DD（两个语言文件都要）
重新记录 sidecar
修复或删除入链
```

由 `verify-archived-agent-notes.ts` 用 **append-only 的冻结内容清单 + sidecar 哈希**强制。**改一个字，哈希就不对，CI 红。**

> 「**不许把归档 note 当作当前行为的权威**」这条同样重要——它防的是 AI 检索到一份三个月前归档的 note，然后按里面早已过时的描述去改代码。第 2 层 `dsh-code-review` 里也有对应条款（archived notes 不是当前权威）。

### 还有一个「反索引」门禁

```ts
if (entry.name === 'INDEX.md') {
  errors.push('structure: INDEX.md — centralized Agent Note indexes are forbidden; browse the lifecycle/class tree or search the repository')
}
```

**集中式索引被明令禁止**，且有专门的 note 记录理由（`2026-07-19-remove-generated-agent-note-index.md`）。

> **为什么禁索引**：551 份 note 的索引文件必然是生成的，而生成物会和真相漂移，且 review 时没人真读得动那个 diff。**目录树本身就是索引**——`{lifecycle}/{class}/日期-标题.md` 的路径已经编码了全部检索维度。**能用目录结构表达的东西，就不要再生成一份清单去维护。**
>
> 这和第 2 层「`.claude/skills` 用软链而不是副本」是同一个判断：**能靠结构消灭的一致性问题，不要用工具去检查。**

---

## 7. 什么时候必须写（这是硬规则）

AGENTS.md 与 `.agents/notes/README.md` 双处规定：

> **Every non-trivial change MUST add or update at least one Agent Note in the same PR.**

「非平凡」有明确定义（不是靠感觉）：

| 改动了…… | 就是非平凡 |
|---|---|
| 行为 | ✅ |
| 架构 | ✅ |
| 跨文件或跨包共享的契约 | ✅ |
| 流程或工具 | ✅ |
| 测试策略 | ✅ |
| 磁盘上/线上/配置格式 | ✅ |
| **另一个维护者可能合理地重新审视的决策** | ✅ |

豁免只有一种：

> Only a **purely mechanical or local** edit with no change to behavior, contracts, structure, process, or rationale is exempt.

三条配套规则防止这条被滥用：

**① 更新已有的那份就够了，不要造重复**

> Updating the Agent Note that already owns the decision satisfies the rule; **do not create a duplicate.**

**② 一份 note 永远不能被改成「另一个决策」**

> An Agent Note is **never** edited into a *different decision*: supersede it with a new one, and keep both notes cross-linked.

**③ 合并删除有极严格的前置条件**

被完全取代的 implemented note **可以**被合并进当前那份然后删除，但删之前必须：

```
保留每一条独有的：rationale / alternative / consequence /
                  required verification / 已命名的覆盖缺口
修复每一条入链
同一个改动里删掉中文对照与一致性记录
```

而且**部分取代不算**：

> Partial supersession does not qualify: keep both notes cross-linked and update every fact that remains current.
> Consolidation must not rewrite the old file into its opposite or **rely on git history as the only copy of rationale**.

> **最后半句是重点：不许把 git 历史当作理由的唯一副本。** 理由必须活在工作树里能被检索到的文件中——因为**agent 检索的是工作树，不是 git log**。这是一条纯粹为 AI 协作而设的规则。

### 一个特别精细的规则：功能新增 note 什么时候能合并进删除 note

只有当这个功能**彻底不存在**时才行——判据列了七项：

```
production 代码 / 配置 / schema / 持久化或线上格式 / 迁移 / 兼容行为   全都没有
且没有任何当前文档把它呈现为可用
且没有任何测试把它当作受支持行为来验证
```

而且明确了什么**不算**彻底删除：

> **Removing one transport, default, implementation, or presentation is partial supersession**, as is any surviving durable data or compatibility handling.

还有一条防伪证据的规则：

> **Obsolete implementation inventories and tests that only verified the deleted behavior are not current verification evidence.**

> 这一段的精细程度说明了一件事：**「删除」在这个体系里是被严肃对待的操作**。第 2 层的 `dsh-find-simplifications` 给「提议删代码」立证据标准，这里给「删决策记录」立证据标准——**两头都堵住了「悄悄让东西消失」这条路。**

---

## 8. 剖析一份关键 Note：`2026-06-11-quality-gates.md`

这份是整个仓库的**元 note**——它记录的是「为什么要有门禁」这个决策，也是总览里被引用最多的一份。

### 它的 `## Problem` 是整套体系最重要的一段自述

> **This codebase is developed primarily by coding agents.** Agents follow enforced gates far more reliably than prose conventions and **"a lot of work" is not a cost argument when agents do the labor.** Early evidence: tests that didn't typecheck shipped (vitest doesn't typecheck) and **were only caught by a review**.

三句话，三个重要结论：

1. **「主要由 coding agent 开发」**——官方从没发 retrospective，但这句话就在仓库里。
2. **「agent 遵守强制门禁远比遵守散文约定可靠」**——这是整个第 4 层存在的理论基础。**不是不信任 AI，而是认识到「写在文档里的规矩」和「exit code 非 0」对 AI 是两种完全不同的信号。**
3. **「当干活的是 agent 时，『工作量大』不构成成本理由」**——这句专门回击「我们没人力写门禁」。

第三句还有一个更深的含义：**门禁的经济学被 AI 改变了**。传统上写 36 个 verify 脚本是不划算的（人力太贵），但当写脚本的也是 agent 时，这个成本项接近消失，于是「应该有多少门禁」的最优解整个变了。

### 它的 `## Decision` 是第 4/5 层的总纲

> **Every mechanically checkable AGENTS.md promise gets a command that exits non-zero.** CI invokes the exhaustive set while Git hooks reserve their latency budget for cheap local defects.

**「AGENTS.md 里每一条机器可查的承诺，都要有一条会返回非 0 的命令」**——这是第 1 层和第 4 层之间的映射法则。

具体清单（原文摘录，也是第 4 层的内容预告）：

```
· 最严 TypeScript（noUncheckedIndexedAccess、exactOptionalPropertyTypes…）
· Oxlint + 类型感知规则 + @stylistic + SonarJS（文件内重复逻辑检查）；vendored 代码排除
· jscpd 检测跨文件克隆；窄范围例外必须写明「刻意并行实现」的理由
· packages/*/*/src 逐文件 100% 覆盖（v8）；
    不可达的防御分支用 /* v8 ignore */ 并写明理由，而不是删掉
· knip（死代码/死依赖）、publint（包正确性）、workspace constraints、
    以及针对已构建包声明的 NodeNext 消费者 typecheck
· lefthook pre-commit：无项目的 Oxlint 校验 + 有界重试的安全修复、
    拒绝暂存的空白字符、检查 vendor manifest；pre-push 跑增量 typecheck
· CI 跑 node 22.19/24/26 全矩阵 + Headless/TUI/ACP/JSON-RPC/code-runtime 的构建后冒烟
```

> **注意「不可达的防御分支用 `/* v8 ignore */` 并写明理由，而不是删掉」**——这条特别值得讲。逐文件 100% 覆盖率有个恶性副作用：**为了凑覆盖率，人们会删掉那些「不可能走到」的防御性代码**。这里明确禁止，改用带理由的忽略标记。**门禁的副作用被预判并封堵了。**

### 它的 `## Consequences` 承认了一个自己制造的问题

> **100%-coverage pressure can produce assertion-free tests** — mutation testing is the planned counterweight (see [the mutation-testing proposal](../../proposed/testing/2026-06-11-mutation-testing.md)).

**逐文件 100% 覆盖会诱发「无断言测试」**——即代码被执行了，但没有任何断言会因为它出错而失败。

这就是 `proposed/` 里那份 `2026-06-11-mutation-testing.md` 的由来，它的 Problem 段写得更狠：

> The per-file 100% coverage gate proves every line *executes* under test — **not that any assertion would notice if the line were wrong.** Under agent-written tests coverage pressure can produce execution-without-assertion. Mutation testing measures what coverage cannot: **whether the suite *kills* deliberately injected bugs.**

> **这两份 note 连起来读，展示了这套体系最健康的一面：它在自己的决策记录里承认自己的门禁有副作用，并且把对策作为一份未落地的提案挂在 `proposed/` 里。**
>
> 对比一下常见做法：门禁上线后就当问题解决了，副作用要么没人提，要么在私下吐槽里。**这里副作用是有文件、有编号、有提案的。**
>
> 而且它诚实地留在 `proposed/`（没落地）而不是假装已经解决——**「已知但未解决」本身就是一条有价值的记录。**

### 它头部那句「被取代」声明的写法

```
The hook/CI symmetry in this record is superseded by [Fast local Git hooks](2026-07-22-fast-local-git-hooks.md); CI remains the exhaustive enforcement path.
```

**部分取代的标准写法**：明确说清「哪一部分」被取代（hook/CI 对称性）、被谁取代（链接）、以及**哪一部分仍然有效**（CI 依然是全量执行路径）。这正是前面那条「partial supersession 必须双向互链且更新仍然有效的事实」的实例。

### 它自己带着豁免注释

这份 note 的最后一行是：

```markdown
<!-- agent-note-format: alternatives-not-recorded (pre-format Agent Note) -->
```

**日期 2026-06-11，在格式规范生效（07-05）之前，备选方案无法重建。** 于是这份最重要的元 note 自己就是那 19 个诚实缺口之一——**规则对自己也一样适用，没有例外。**

---

## 9. 双语与一致性记录（sidecar）

每份 Agent Note 是一个**三元组**：

```
2026-06-16-pnpm-over-yarn.md          英文
2026-06-16-pnpm-over-yarn.zh.md       中文
2026-06-16-pnpm-over-yarn.i18n.yaml   一致性记录（sidecar）
```

sidecar 的内容就是两侧的 git blob 哈希：

```yaml
# Bilingual-pair consistency record (docs/i18n/README.md): the git blob hash of each
# side as of the last confirmed-consistent state. Both languages carry equal authority;
# after editing either side bring the other along and re-record with:
#   pnpm run verify-translation-pairing --write .agents/notes/…/2026-06-16-pnpm-over-yarn.md
2026-06-16-pnpm-over-yarn.md: 30b34c65fdea94b20dec4d627a0fca40de760fd1
2026-06-16-pnpm-over-yarn.zh.md: 7560b748654cf79409e16b6c2b1c701b367f01d4
```

几个设计点：

- **「两种语言享有同等权威」**（`Both languages carry equal authority`）——中文不是英文的附属翻译，改任一侧都必须带上另一侧。
- **改了一侧不改另一侧 → 哈希不匹配 → 门禁红。** 这让「中文文档悄悄落后半年」在物理上不可能。
- **格式门禁跳过 `.zh.md`**（结构由英文侧定义），**配对门禁负责一致性**——两个门禁职责不重叠。
- 机器校验的头部标记（`# Agent Note: ` 和 `Status:` 行）**在中文文件里也保持英文原文**，这样一套正则同时管两种语言。

---

## 10. 落到自己项目：五步

1. **先只跑两态**：`proposed/` 和 `implemented/`。`rejected/` 和 `archived/` 等真的有需要再加——过早引入四态只会让大家不知道该放哪。
2. **`## Alternatives considered` 从第一天就强制**，这是全部价值所在。**其他小节都可以商量，这一节不行。**
3. **状态编码在路径里，不要在文件里加 `status:` 字段**。移动文件是个明确的动作，改字段太容易被忘。
4. **写一个最小门禁**：只查三件事——头部格式、必需小节存在、implemented 里没有 `## Proposal`。**二十行代码，挡住 80% 的腐烂。**
5. **明确「非平凡」的定义**，把它写进 AGENTS.md。定义模糊的话，「什么时候要写 note」会变成每个 PR 都要争论一次的事。

> **两条最容易被忽略但价值极高的规则，建议直接抄**：
>
> - **「Alternatives are recorded, never invented.」**——补不出来就留一个带标记的诚实缺口，绝不让 AI 编。一旦允许编造，整个记录体系的可信度归零。
> - **「不许把 git 历史当作理由的唯一副本。」**——AI 检索工作树，不检索 git log。理由必须活在文件里。

---

## 11. 一页总结

| 问题 | 答案 |
|---|---|
| 是什么 | 由 AI 写的 RFC/ADR；551 份活跃 + 142 份归档 |
| 两个轴 | lifecycle（4 态）× class（6 类封闭集合），**都编码在路径里** |
| 唯一强制小节 | `## Alternatives considered`——**记录决策却不记录它击败了什么，等于邀请反复争论** |
| 最了不起的配套规则 | **「备选方案是被记录的，不是被编造的」**——19 份老 note 带着诚实的缺口标记 |
| 三套骨架 | proposed 用将来时（Proposal/Acceptance criteria/Risks）；implemented 用现在时（Decision/Consequences）；rejected 冻结提案 + 判决写在 Status 行 |
| 状态迁移 | **机械化改写**，门禁拒绝「implemented 里出现 `## Proposal`」 |
| `implemented/` 的维护 | 事实原地重写、不追加历史；**但推翻决策必须写新 note** |
| `rejected/` 的价值 | 11 份一行判决 = **已经付过的学费**（含一次「被实现证伪」的实验和一次完整调研的冻结结论） |
| 门禁 | `verify-agent-note-format`（11 类检查）、`-classification`、`-archived`（哈希冻结）、`agent-note-tree.ts`（封闭集合） |
| 反索引 | **`INDEX.md` 被明令禁止**——目录树本身就是索引 |
| 双语 | 三元组（en/zh/sidecar），**两种语言同等权威**，哈希不匹配则 CI 红 |
| 最重要的一段自述 | quality-gates note：「本代码库主要由 coding agent 开发」+「当干活的是 agent 时，『工作量大』不构成成本理由」 |
| 最健康的一面 | 在决策记录里**承认自己的门禁有副作用**（100% 覆盖诱发无断言测试），并把对策挂在 `proposed/` 里没落地 |
