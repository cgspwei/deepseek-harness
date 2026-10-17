# 第 1 层详解：AGENTS.md —— 仓库级 System Prompt

> [← 返回六层全景图](how-deepseek-does-ai-coding.md)
>
> 本文把根 `AGENTS.md` 讲透：它是什么、每一节为什么存在、靠什么机制不腐化、你怎么抄。
> 所有数字和引文均对照仓库真实文件核实（核实日期见 git 记录）。

**一句话定位**：

> AGENTS.md 不是「给新人看的 README」，而是**每次 AI 会话都会被自动加载的常驻 system prompt**——它是「不是人每次告诉 AI 怎么干，而是仓库自己告诉 AI 怎么干」这句话的实体。

---

## 目录

1. [先看数字：这个文件到底多大](#1-先看数字这个文件到底多大)
2. [九个区段各自解决什么问题](#2-九个区段各自解决什么问题)
3. [开篇三句话：信息密度的极致](#3-开篇三句话信息密度的极致)
4. [Pre-release stance：给 AI 做「胆量校准」](#4-pre-release-stance给-ai-做胆量校准)
5. [Conventions：31 条压缩的资深工程师判断](#5-conventions31-条压缩的资深工程师判断)
6. [词数预算：靠机器防腐化的核心机制](#6-词数预算靠机器防腐化的核心机制)
7. [分层 AGENTS.md：就近原则](#7-分层-agentsmd就近原则)
8. [CLAUDE.md 软链：一份规则，多个入口](#8-claudemd-软链一份规则多个入口)
9. [四个可迁移的设计模式](#9-四个可迁移的设计模式)
10. [反 AI 弱点的针对性条款](#10-反-ai-弱点的针对性条款)
11. [怎么抄：给自己项目的落地步骤](#11-怎么抄给自己项目的落地步骤)

---

## 1. 先看数字：这个文件到底多大

讲设计之前先建立体感——**「小」是这个文件最重要的特征**。

| 指标 | 实测值 |
|---|---|
| 文件大小 | 15.62 KB |
| 词数（`wc -w`） | **1929 词** |
| 门禁上限（ceiling） | **1950 词** |
| 长期目标（target） | **≤ 1600 词** |
| 顶级区段数 | 9 个 |
| Conventions 条数 | **31 条** |

**注意 1929 / 1950 这个数字**：只剩 **21 词余量**（约 1%）。这不是巧合，是「棘轮机制」的必然结果（见[第 6 节](#6-词数预算靠机器防腐化的核心机制)）——**这个文件永远处于「快撑爆」的状态，任何新增都必须先删掉点什么。**

> **纠正一个常见误解**：很多介绍说「AGENTS.md 限制 1600 词」。准确说法是：**当前门禁上限 1950 词，1600 是 `docs/AGENTS.md` 里写明的长期目标值**，文件目前超出目标、正在往下棘轮。

---

## 2. 九个区段各自解决什么问题

```
1. 开篇定位（无标题）        系统本质 + 必读顺序        → 导航
2. Pre-release stance        临时策略（带过期条件）      → 时效性规则
3. Repository layout         代码库地图                 → 世界模型
4. Commands                  所有可执行命令 + 语义       → 工具箱
   ├─ Host sandbox failures  环境故障排查规则            → 故障预案
   └─ Run relevant checks    什么改动跑什么检查          → 工作流
5. Secrets / .env            密钥边界                   → 安全
6. Conventions（31 条）      编码规则                   → 行为约束（核心）
7. Defensive patterns        指针：危险工作前先读        → 条件性导航
8. Type safety and docs      类型 + 文档 + 文风标准      → 质量标准
9. Editing these instructions 如何编辑本文件            → 元规则
10. Vendoring policy         依赖管理边界               → 边界政策
```

**关键观察**：**没有欢迎语、没有项目背景故事、没有贡献指南、没有徽章。** 每个字节都在为 AI 会话服务。

这九节可以归成四类，对应 AI 干活时的四个需求：

| 类别 | 区段 | AI 的疑问 |
|---|---|---|
| **世界模型** | 开篇、Repository layout | 「这是什么系统？东西都在哪？」 |
| **工具箱** | Commands、Secrets | 「我能跑什么？前提是什么？」 |
| **行为约束** | Pre-release stance、Conventions、Type safety | 「什么该做，什么不该做？」 |
| **元规则** | Defensive patterns、Editing、Vendoring | 「特殊情况怎么办？」 |

---

## 3. 开篇三句话：信息密度的极致

```markdown
DeepSeek Harness is a plugin-based agent harness on vendored Cordis:
**everything is a plugin**. Read [docs/architecture.md](docs/architecture.md)
before changing `packages/`; follow [docs/AGENTS.md](docs/AGENTS.md) for
documentation.
```

**三句话干了三件事**：

1. **一句话定义系统本质**（`everything is a plugin`）——AI 不必读完代码就拿到核心心智模型。后续所有「为什么这么设计」都能从这句推导。
2. **风险前置门**——`Read architecture.md before changing packages/`：把**最危险的区域**和**必读的上下文**绑定。这是一条「条件触发的加载指令」。
3. **指路**——文档规范在 `docs/AGENTS.md`。

**可迁移点**：你的 AGENTS.md 第一句应该回答「**这个系统的一句话心智模型是什么**」，而不是「这是一个用 XX 技术栈做的 YY 系统」。

---

## 4. Pre-release stance：给 AI 做「胆量校准」

这一节是整个文件里**最容易被忽略、但设计最精妙**的部分。

```markdown
## Pre-release stance: foundation over blast radius

**Remove this section at the first tagged release.** With no external
consumers, prefer the correct foundation over compatibility shims: rename or
repackage freely and update every reference together. Backends reject old
on-disk formats. SQLite uses monotonic `SCHEMA_VERSION`; `dsh-session` keeps
`SESSION_FORMAT_VERSION` at `0` with no compatibility promise.
```

### 4.1 它在回答「AI 该多大胆？」

改任何系统都面临一个选择：

- **激进**：重命名、重打包、改结构——爆炸半径大，但地基正确
- **保守**：留 shim、加废弃层、写迁移代码——爆炸半径小，但积累垃圾

**AI 的默认倾向是保守的**（训练数据里全是「不要破坏兼容性」）。不明确告诉它，它本能地写兼容层——因为保守永远不会被骂。

### 4.2 给出的答案与理由

理由藏在 `With no external consumers` 里：**爆炸半径理论上是零**。此时写 shim 是在制造未来要删的垃圾。

三个例子全在说「不要写迁移代码」：

| 例子 | 含义 |
|---|---|
| `Backends reject old on-disk formats` | 直接拒绝旧格式，不是「兼容读取」 |
| `SQLite uses monotonic SCHEMA_VERSION` | 版本号只增不减 |
| `SESSION_FORMAT_VERSION at 0 with no compatibility promise` | 明确不承诺兼容 |

### 4.3 最精妙之处：规则自带销毁条件

`**Remove this section at the first tagged release.**` 这句话意味着：

1. 这是**限时授权**——一旦打 tag，外部消费者可能出现，「大胆重构」的许可必须立刻失效
2. 过期条件**机械可判定**——「第一个 tag 是否存在」一个脚本就能查
3. **不会漂移**——大多数团队的临时策略死于「没人记得该撤」

### 4.4 可迁移的通用模式

给 AI 的胆量校准必须显式写出来，且绑定生效条件：

```
适用窗口 + 当下偏好（激进/保守） + 现在为什么可以 + 何时失效
```

**生产系统不能照搬内容，但要照搬模式**——而且更需要。因为：

> Pre-release 项目的病是「AI 太保守，写一堆没必要的 shim」。
> **成熟生产系统的病是反过来的：永远保守，shim 永远不删，「临时兼容」变成永久 legacy。**

生产系统没有「打 tag」这种全局开关，于是兼容层只进不出——这就是祖传代码的成因。

**生产版模板**（可直接抄改）：

```markdown
## Risk stance

按边界分级，不搞一刀切：

- **对外 API / RPC 契约**：版本化演进。新字段必须可选；删字段/改语义走
  deprecation 流程（公告 → 双写观察期 → 下线）。禁止直接改。
- **存储 schema**：单调 version + 迁移脚本，禁止原地 ALTER 有数据的表。
- **模块间接口**：expand-contract（加新路径 → 迁移调用方 → 删旧路径）。
- **模块内部 / 单一调用方的代码**：鼓励激进重构，不留 shim。
- **配置**：未知/缺失配置 fail loud，禁止静默回退默认值。
- **所有兼容层必须带销毁条件**：合入时写明「在 X 日期 / 迁移完 Y 流量后删除」，
  到期未删视为 bug。
```

最后一条就是 `Remove this section at the first tagged release` 的生产镜像——**给每个兼容层一个自己的「删除本节」条款**。

---

## 5. Conventions：31 条压缩的资深工程师判断

这是全文核心，占了文件的大半。**每条都是「用 1-3 行压缩的架构决策」**，格式统一：

```
**加粗的关键短语**：具体规则（[链接到 rationale 的家](路径)）
```

### 5.1 六大类别与代表条款

**① 架构纪律**

```markdown
- **Registrations are effects**: every contribution goes through
  `ctx.effect()` / `ctx.on()`; a registry's `register()` returns the disposer.

- **Plugins, not loop changes**: new behavior goes on documented extension
  points; changing `agent-loop` requires updating docs/architecture.md.
```

第二条尤其精彩：它不禁止改核心循环，而是**给改动挂上一个「同步成本」**——你可以改，但必须改架构文档。

**② 类型安全**

```markdown
- **Switch on discriminant tags.** Closed unions end in `assertNever`;
  merge-extensible unions fall through a documented default.

- **Opaque cross-boundary ids are branded** (`Branded<B>` from `dsh-brand`),
  never bare `string`.
```

**③ 失败语义**

```markdown
- **Misconfiguration fails loud** at load when self-contained, otherwise at
  the earliest resolvable point; never silently skip a missing referent.

- **An empty `catch` names what it swallows** and why nothing else can reach
  it; keep the `try` to one statement.
```

**④ 验证边界**（最精彩的一条）

```markdown
- **Trust TypeScript at typed same-process boundaries.** Do not add runtime
  validation, fallback behavior, or hostile-input tests solely for values the
  static interface requires; validate at parser/config, queued, model/tool
  JSON, durable/file, worker, process, and wire boundaries.
```

这一条**同时防两个方向的错误**：

```
防过度：AI 爱到处加防御式校验 → 同进程边界信任类型系统
防不足：该校验的边界裸奔      → 明确列举 7 类必须校验的边界
```

**校验强度与爆炸半径成正比**：同进程边界编译期就炸（半径小），wire/持久化边界运行时才炸（半径大）。

**⑤ 可观测性**

```markdown
- **Model-visible ⟺ logged**: anything that reaches a model request must be
  reconstructable from the session log; a new model-visible input requires a
  session event.
```

这条是整个项目「可回放、可 fork」能力的地基——**双向蕴含**（⟺）意味着两个方向都是规则：模型能看到的必须被记录，被记录的定义了模型能看到什么。

**⑥ 流程纪律**

```markdown
- **Non-trivial changes MUST include an Agent Note in the same PR;** only
  mechanical/local edits are exempt. Archived notes are frozen: never edit or
  treat them as current authority.
```

**这条是第 1 层与第 3 层（notes）的接缝**——AGENTS.md 在这里把控制权交给 Agent Note 体系。

### 5.2 关键观察：每条都「可裁定」

**这 31 条里几乎每条都能被机器检查，或被 review 一眼裁定对错。**

没有一条是「代码要优雅」「注意性能」这种空话。

> **判定标准**：如果一条规则被违反了，**你无法一眼指出违反在哪**，那它就不配写进 AGENTS.md。

这就是「规则必须能约束 Agent」的实践检验——也是你写自己 AGENTS.md 时最该守住的门槛。

### 5.3 「链接到 rationale 的家」模式

注意几乎每条规则末尾的括号链接：

```markdown
([policy](.agents/notes/implemented/process/2026-07-26-dependencies-over-hand-rolling.md))
([mechanism](.agents/notes/implemented/architecture/2026-08-10-session-log-version-mechanism.md))
([glossary](docs/glossary.md#capability-seam))
```

**规则在这里，理由在别处。** 这是保住词数预算的关键手法：

```
AGENTS.md 只放「什么」（1-3 行）
     ↓ 链接
Agent Note 放「为什么 + 放弃了什么」（不限长度）
```

AI 读规则时不需要理由；**只有当它想违反或修改规则时，才需要点进去看理由**。这是一种「按需加载」。

---

## 6. 词数预算：靠机器防腐化的核心机制

**这是 AGENTS.md 能长期保持有效的唯一原因，也是最值得抄的机制。**

### 6.1 为什么必须有预算

AGENTS.md 的天敌是**熵增**：

```
出了问题 → 加一条规则 → 文件变长 → 重点被稀释 → AI 记不住 → 出更多问题 → 加更多规则
```

大多数团队的「编码规范」就是这么死的：长到没人读，包括 AI。

### 6.2 机制实现

**声明**（`scripts/doc-budgets.manifest.json`）：

```json
{
  "AGENTS.md": 1950,
  "docs/AGENTS.md": 1320,
  "docs/architecture.md": 2400,
  "docs/cordis-primer.md": 600,
  "docs/defensive-patterns.md": 550,
  "docs/testing.md": 1150,
  "examples/AGENTS.md": 310,
  "packages/AGENTS.md": 675,
  "packages/README.md": 994
}
```

**执行**（`scripts/verify-doc-budgets.ts`）：`wc -w` 等价计数，超出即 `exit 1`。

### 6.3 红灯时的三步处置（有优先级）

`docs/AGENTS.md` 规定得极清楚：

```
1. Relocate（迁移）— 内容属于别的层级，搬走，留一行链接
2. Condense（精简）— 内容该在这，但可以更短
3. Raise（提高上限）— 只在「文字确实需要空间」时，且必须在 PR 里说明理由
```

**顺序不能颠倒**。默认动作是「搬走」，不是「放宽」。

### 6.4 棘轮机制（ratchet）—— 最精妙的部分

> 「棘轮」是只能单向转动的机械装置（扎带是日常例子：勒紧容易，松不回来）。工程里借指**只允许单向变化**——覆盖率棘轮是「只许升」，这里是「上限只许降」。两者的共同目的：**防止指标在无人察觉时悄悄滑坡。**

```
Ceilings ratchet down with at least 5% headroom.
Ceilings are guardrails, not reduction targets.
At or below target, retain at least 5% headroom;
above target, freeze the ceiling until relocation or condensation
brings the document under target.
Lower a ceiling only when the document still has room,
and raise it when content would otherwise be deleted.
```

翻译成机制：

| 状态 | 规则 |
|---|---|
| 文档在目标值以下 | 保留至少 5% 余量，**上限可以往下棘轮** |
| 文档超出目标值 | **上限冻结**，只能靠迁移/精简降下来 |
| 什么时候能降上限 | 只在文档还有余量时 |
| 什么时候能升上限 | 只在「否则内容就得被删」时 |

**当前状态**：根 AGENTS.md 是 1929 / 1950 词，目标 1600——**处于「超出目标、上限冻结」状态**。也就是说，现在**任何新增内容都必须先删掉等量的东西**。

**还有一条护栏**：`Ceilings are guardrails, not reduction targets`（上限是护栏，不是缩减目标）——防止有人为了「达标」把有用信息删掉。**一个过低的上限是 budget bug**。

### 6.5 为什么这是最该抄的机制

因为它把「保持精简」从**一句劝告**变成了**一个会让 CI 变红的事实**。

> 没有预算的 AGENTS.md，三个月后一定会变成没人读的长文档。

---

## 7. 分层 AGENTS.md：就近原则

根 AGENTS.md 不是唯一的——仓库里有 **13 个真实的 AGENTS.md**（另有 4 个是测试 fixture）：

| 路径 | 词数 | 管什么 |
|---|---|---|
| `AGENTS.md`（根） | 1929 | 全仓库规则 |
| `packages/client/AGENTS.md` | 2236 | 客户端包（**比根还大**） |
| `docs/AGENTS.md` | 1320 | 文档标准 |
| `packages/AGENTS.md` | 672 | 包编写规则 |
| `examples/AGENTS.md` | 237 | 示例规则 |
| `.github/AGENTS.md` | 69 | CI / issue 管理 |
| `scripts/AGENTS.md` | 34 | 门禁脚本规则 |
| `.agents/notes/AGENTS.md` | — | Agent Note 规则 |
| `.agents/notes/implemented/AGENTS.md` | — | implemented 态规则 |
| `.agents/notes/archived/AGENTS.md` | — | archived 态规则 |
| `packages/schedule/AGENTS.md` | — | 调度包 |
| `packages/web/AGENTS.md` | — | web 包 |
| `native/landlock-run/AGENTS.md` | — | native 模块 |

### 7.1 三个值得注意的设计

**① 规则放在「离代码最近的地方」**

AI 改 `scripts/` 下的文件时，会加载 `scripts/AGENTS.md`（仅 34 词）；改 `packages/client/` 时加载那份 2236 词的。**上下文按需加载，不是一次性灌满。**

**② 子树预算更严**

`docs/AGENTS.md` 明文：`subtree AGENTS.md ≤ 600 words`（除了 `packages/AGENTS.md ≤ 650` 和它自己 `≤ 1250`）。

**③ `scripts/AGENTS.md` 只有 34 词**

全文就一句话：

> Gate scripts invoke pnpm shell-free, normalize repository-relative glob
> paths to `/`, and keep platform adaptation in the gate that needs it
> instead of a shared platform layer.

**34 个词，三条约束，全部可裁定。** 这是「规则精简」的极致示范——不是所有目录都需要长文档。

### 7.2 `packages/client/AGENTS.md` 为什么能超过根文件

2236 词 > 1929 词，而且**它不在预算清单里**（`doc-budgets.manifest.json` 只列了 9 个文件）。

`docs/AGENTS.md` 解释了：`Review governs unbudgeted tiers`——**未列入预算的层级由 review 管**。

这体现了一个判断：**根文件必须极简（因为每次会话都加载），叶子文件可以厚（因为只在碰那块代码时才加载）。**

---

## 8. CLAUDE.md 软链：一份规则，多个入口

```
CLAUDE.md          -> AGENTS.md
packages/CLAUDE.md -> AGENTS.md
examples/CLAUDE.md -> AGENTS.md
```

三个位置都是**符号链接**（`ls -l` 实测确认）。

`## Editing these instructions` 一节写明：

> `CLAUDE.md` symlinks `AGENTS.md` at root, `packages/`, and `examples/`;
> **edit the real file.**

**设计意图**：不同 AI 客户端约定的文件名不同（Claude Code 读 `CLAUDE.md`，`.agents/` 规范读 `AGENTS.md`）。用软链而不是复制，保证：

- **单一事实来源**——不可能出现两份不同步
- **工具无关**——换 agent 客户端不需要迁移规则

> 这和第 2 层 `.agents/skills/` 与 `.claude/skills/` 的镜像是同一个思路：**兼容多客户端，但不制造第二份真相**。（skills 那边是内容镜像，这边是软链，因为单文件可以软链。）

---

## 9. 四个可迁移的设计模式

### 模式 1：入口极简，深处外链

```
AGENTS.md（1929 词，每次加载）
    ├─→ Agent Note（为什么，按需点开）
    ├─→ docs/architecture.md（架构细节）
    └─→ 子目录 AGENTS.md（局部规则，按需加载）
```

**规则在这，理由在家。**

### 模式 2：命令即工具，注释即契约

```sh
pnpm run test:coverage  # CI coverage gate: per-file 100% on packages/*/*/src
pnpm run test:e2e       # real-API tests; self-skip without DEEPSEEK_API_KEY
pnpm run demo:cordis    # the agent modifies its own runtime (needs key)
pnpm run check:windows-wine  # ONLY when diagnosing a known Windows failure
```

每条注释都在传递**副作用和前提**，不只是「这命令叫什么」：

- `test:coverage` 标注「CI gate」→ 这是门禁级命令
- `test:e2e` 标注「self-skip without key」→ **没密钥不会挂，别误判为失败**
- `check:windows-wine` 标注「ONLY when...」→ 别随便跑

**「知道有哪些命令」和「知道命令的语义与前提」是两个层次，这里两个都给了。**

### 模式 3：规则必须可裁定

见 [5.2](#52-关键观察每条都可裁定)。**违反后无法一眼指出，就不配写进去。**

### 模式 4：预算 + 棘轮防腐化

见[第 6 节](#6-词数预算靠机器防腐化的核心机制)。**把「保持精简」变成会让 CI 变红的事实。**

---

## 10. 反 AI 弱点的针对性条款

**这个文件本身就是一份「AI 弱点清单」的对策集**——这是它区别于普通编码规范的地方。

| AI 的弱点 | 对策条款 | 原文 |
|---|---|---|
| **误诊环境问题为代码 bug** | Host sandbox failures | 先最小提权重试，再诊断代码 |
| **拿环境当借口跳过测试** | 同上（反向条款） | `never bypass genuine test failures` |
| **不确定就全量跑** | Run relevant checks | `Never default to the full suite` |
| **重复跑已通过的检查** | 同上 | `never ... repeat a passing check for commit or push` |
| **过度防御式编程** | Trust TypeScript | 同进程边界不加运行时校验 |
| **默认写兼容 shim** | Pre-release stance | `prefer the correct foundation over compatibility shims` |
| **文档写空话/比喻** | Type safety and docs | `Do not use metaphors`；禁 `shape`/`boundary` 等模糊词 |
| **注释复述代码** | Conventions | `Do not comment on facts obvious from code` |
| **注释写成推理过程** | Type safety and docs | `state complete contracts, not reasoning transcripts` |

### 特别值得学的两条

**① 双向防作弊**（Host sandbox failures）：

```
正向：命令失败先怀疑沙箱，用最小提权重试
反向：Require sandbox evidence; never bypass genuine test failures
```

给了台阶，同时堵死了滥用台阶的路。**这是写 AI 规则的高级技巧**——任何「例外许可」都要配一个「不许滥用」的反向条款。

**② 禁用模糊词**（Type safety and documentation）：

> Before writing `contract`, `boundary`, or `shape`, ask whether a more exact
> term names the subject: write `response fields`, `JSON validation`, or
> `ESM exports` instead of `response shape`, `validation boundary`, or
> `module shape`.

AI 写文档特别爱用这类**「听起来专业但指代不清」**的词。这条不是禁止用词，而是**要求先自问有没有更精确的词**——并给了三组正反例。

---

## 11. 怎么抄：给自己项目的落地步骤

### 第一步：写骨架，但**故意不写 Conventions**

```markdown
# AGENTS.md

<一句话说清系统本质> + <改 X 之前必须先读 Y>

## Repository layout
<目录树，每行一句职责>

## Commands
<所有命令 + 每条的语义和前提>

## Conventions
（先留空——等踩坑）

## Editing these instructions
<谁能改、怎么改、预算多少>
```

**为什么故意留空**：一上来凭想象写的规则，AI 大概率不会照做，因为它们通常不可裁定（「代码要清晰」）。**规则应该是踩坑的产物。**

### 第二步：建立「踩坑 → 补规则」的回路

每次 AI 犯错，先问一个问题：

> **这个错误，能不能写成一条「违反后能一眼看出」的规则？**

- **能** → 写进 Conventions（1-3 行 + 链接到详细理由）
- **不能** → 说明你还没想清楚，先别写；或者它其实该是一条门禁脚本

### 第三步：一开始就上词数预算

**不要等文件变长了再治理**，那时已经没人读了。

最小实现（不依赖 TypeScript）：

```bash
#!/usr/bin/env bash
# scripts/check-doc-budget.sh
set -e
LIMIT=1500
WORDS=$(wc -w < AGENTS.md)
if [ "$WORDS" -gt "$LIMIT" ]; then
  echo "AGENTS.md: $WORDS words exceeds $LIMIT — relocate or condense first."
  exit 1
fi
echo "AGENTS.md: $WORDS / $LIMIT words."
```

挂到 CI，然后把「红灯三步处置」（迁移 → 精简 → 才是放宽）写进文档。

### 第四步：分层，但从根开始

先只写根 AGENTS.md。**当某个子目录反复出现「只有这里才适用」的规则时**，再建子目录 AGENTS.md 并把那些规则搬下去。

参照 DeepSeek 的预算比例：

```
根        ≤ 1600 词
子树      ≤ 600 词
特殊叶子   不设预算，由 review 管
```

### 第五步：给每条规则找个「家」

规则写在 AGENTS.md（1-3 行），理由写在别处（ADR / Agent Note / 设计文档），用链接连起来。

**没有「家」的规则会不断膨胀**——因为写规则的人总想解释清楚，一解释就长。

---

## 附：本文与其他笔记的关系

| 笔记 | 关系 |
|---|---|
| [六层全景图](how-deepseek-does-ai-coding.md) | 本文是其第 1 层的展开 |
| [agents-md-deep-dive.md](agents-md-deep-dive.md) | 更早的逐节解读，含 shim 概念澄清、生产系统适配专题 |
| [agents-md-skeleton-template.md](agents-md-skeleton-template.md) | 可直接套用的骨架模板 |
| [how-to-write-agents-md.md](how-to-write-agents-md.md) | 写作方法论 |

## 附：本文核实过的事实清单

| 结论 | 核实方式 |
|---|---|
| 1929 词 / 1950 上限 | `wc -w AGENTS.md` + `doc-budgets.manifest.json` |
| 1600 是目标非上限 | `docs/AGENTS.md` Wordcount Budgets 节原文 |
| Conventions 31 条 | `awk` 截取区段后 `grep -c '^- '` |
| 9 个顶级区段 | `grep -c '^## ' AGENTS.md` |
| 13 个真实 AGENTS.md | 文件搜索（排除 4 个测试 fixture） |
| CLAUDE.md 是软链 | `ls -l` 显示 `-> AGENTS.md` |
| 棘轮 / 5% 余量机制 | `docs/AGENTS.md` + `verify-doc-budgets.ts` 头部注释 |
