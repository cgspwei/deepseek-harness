# 第 6 层详解：code review —— 只查机器查不了的语义

> 总览见 [how-deepseek-does-ai-coding.md](how-deepseek-does-ai-coding.md)。本文是第 6 层的完整拆解。
>
> **一句话**：前面各层的全部意义，是把 review 的输入面积压缩到少量 reviewer（含 AI）扛得住的程度。而 review 本身被写成了一份 skill——**因为执行它的也是 AI**。

---

## 0. 这一层是整套体系最容易被挑战的一环

分享时几乎必然被问：

> **这么大代码量、这么快的产出，传统人工 review 绝对扛不住。那质量是不是全靠测试和 lint？**

这个质疑合理，而且答案不是「review 照旧」。真实答案分三层：

1. **review 没有缺失，反而被规范得最细**——它是 11 个 skill 里唯一一份「审查标准」。
2. **它是 skill，写给 AI 执行**——所以 review 侧和产出侧**用同一种方式扩容**。
3. **门禁不是替代 review，而是为 review 腾出空间**——有一条明文的分工声明。

---

## 1. 先纠正一个事实：它是 11 个 skill 里第 7 长的，但最"密"

实测词数排名：

| skill | 词数 |
|---|---|
| `dsh-find-simplifications` | 2038 |
| `record-browser-gif` | 1828 |
| `dsh-translate-docs` | 1399 |
| `dsh-prose-standard` | 1240 |
| `dsh-merging-stacked-prs` | 1161 |
| `dsh-pre-push-checks` | 1156 |
| **`dsh-code-review`** | **1073**（8278 字节） |
| `dsh-trim-cot-leakage` | 1005 |
| 其余三个 | 865–910 |

**它只有 1073 词，却装了 6 条硬阻断 + 14 项人工检查 + 7 个权威来源 + 报告规范。** 平均每条规则不到 45 词——**这份文档的信息密度是所有 skill 里最高的**，因为它把所有细节都外链给了别的文档（`AGENTS.md`、`defensive-patterns.md`、`packages/AGENTS.md`、`dsh-prose-standard`……）。

> **这本身就是一个设计选择**：review skill 不复述任何规则，它只负责**「指出该去查哪条规则」以及「哪些东西只能靠人/AI 判断」**。规则的家在别处（第 1、3、7 层），这里只做调度。

### 开篇第一句就是一个反预期声明

```
**This skill is guidance, not a complete checklist.**
```

**「这是指引，不是完整清单」**——写在最前面。

> 为什么重要：如果它自称「完整清单」，AI 就会**逐条打勾然后宣布通过**，清单外的问题一律不看。声明它不完整，是在要求 reviewer 保留判断力。第 2 层的 `dsh-find-simplifications` 里有一句同构的话（「召回策略只是探针，不是定义」）——**这个仓库反复防同一件事：把启发式清单误当成充分条件。**

---

## 2. 优先级：一句话定调

```
Prioritize correctness, lifecycle, security, and broken required behavior over style;
a short review with one substantiated blocker is better than a list of nits.
```

两条独立的规则被压在一句话里：

| 规则 | 治什么病 |
|---|---|
| **正确性 / 生命周期 / 安全 / 必需行为损坏 优先于风格** | review 变成风格辩论 |
| **一个有据可依的 blocker 好过一堆 nit** | **review 用数量伪装质量** |

> 第二条专治 AI reviewer 的典型病：它极容易产出 20 条「建议把这个变量名改得更清楚」。**这种 review 的伤害不是浪费时间，而是让真正的 blocker 淹没在噪音里**，同时给作者一种「这次 review 很认真」的错觉。
>
> 用「一个 substantiated blocker > 一堆 nit」当作明文标准，等于告诉 AI：**你的输出不按条数计分。**

---

## 3. 前置动作：`change-scope` 与「验证 base」

skill 要求在读 diff 之前先做两件事：

```
Verify and fetch the PR's live base and exact head, then run
pnpm --silent run change-scope --base <verified-base-ref> --head <verified-head-ref>
before reading the diff and enough surrounding code to understand the design.
```

三个动词值得注意：**Verify（验证）、fetch（拉取）、rerun after retarget（改基后重跑）**：

```
Re-establish the base and rerun it after a retarget or merge.
```

> **为什么强调「验证 base」**：PR 的 base 是会变的（retarget、merge-forward、stack 上的父分支被合并）。**如果 reviewer 用了过期的 base，diff 里会混进不属于这个 PR 的改动，或者漏掉真正的改动。**这在第 7 层 `responding-to-pr-review-on-a-stack.md` 描述的 PR 栈场景里尤其常见。

### `change-scope.ts` 做什么，以及它刻意不做什么

它输出一份版本化的 JSON 报告：

```json
{
  "formatVersion": 1,
  "repositoryRoot": "...",
  "input":    { "base": "...", "head": "..." },
  "resolved": { "baseSha": "...", "headSha": "...", "mergeBaseSha": "..." },
  "paths": {
    "committed": [...],   // merge-base..head 的已提交改动
    "staged":    [...],   // 暂存
    "unstaged":  [...],   // 未暂存
    "untracked": [...]    // 未跟踪
  }
}
```

**四类路径分开报**——这就是总览里说的「dirty layers」。它让 reviewer 能区分「这个文件的改动是已提交的，还是本地没提交的脏状态」。

实现上有几处很讲究的严格性：

| 设计 | 目的 |
|---|---|
| `core.warnAmbiguousRefs=true` + 检测 `\bambiguous\b` | **歧义 ref 直接报错**，要求用完全限定 ref 或 commit ID |
| ref 必须解析到**恰好一个** commit | 否则报错 |
| merge-base 必须**唯一**（`--all` 后检查数量） | 多个 merge-base（criss-cross merge）会让 diff 含义不明，直接拒绝 |
| `TextDecoder('utf-8', { fatal: true })` | **非 UTF-8 的路径名直接报错**，并指出是第几条记录 |
| `-z` 分隔 + 手写 `parsePathSet` | 正确处理含换行/特殊字符的文件名 |
| `GIT_OPTIONAL_LOCKS=0`、`core.fsmonitor=false`、`LANG=C` | 不干扰用户的 git 状态，输出不受 locale 影响 |
| `--no-ext-diff --no-textconv --no-renames` | 不受用户 git 配置影响，rename 显示为「删+增」 |

> **`--no-renames` 是个有意思的选择**：它让重命名显示成一删一增。对 review 来说这更安全——**「重命名」的判定是启发式的，一个被 git 判定为 rename 的文件可能内容已经大改**。分开显示强迫 reviewer 看到两侧。

### 但工具的定位边界被明确写出来

```
The report identifies paths and dirty layers but **does not replace semantic review**.
```

> **「工具定位边界，AI/人做判断」**——这和第 4 层门禁的分工模式完全一致：**机器负责把事实摆清楚，判断留给下游。** `change-scope` 不告诉你「这个改动有问题」，只告诉你「改动碰了这些路径，其中这些是脏的」。

---

## 4. 六条硬阻断（blocking requirements）

逐条看，每条都对应一类**机器物理上查不了**的东西。

### ① 新增文字必须经过语义审查

```
New prose receives semantic review. Use dsh-prose-standard to critically review every
added or changed Markdown passage, JSDoc, comment, prompt, description, diagnostic,
and visible string. Verify required coverage, accuracy, placement, and editorial quality
against the owning code or behavior; **automated checks do not establish those properties.**
```

七类文字全覆盖：Markdown、JSDoc、注释、**提示词**、描述、**诊断信息**、可见字符串。

最后半句是这一层的核心论证之一：**「自动检查建立不了这些性质」**。`verify-export-jsdoc` 能查「有没有 JSDoc」，查不了「这段 JSDoc 说的是不是真的」。

### ② 文档与代码匹配

```
Config, defaults, errors, wire fields, events, and public behavior update the package
README and JSDoc in the same diff. Comments state non-obvious contracts; flag
implementation narration, test walkthroughs, review history, and duplicated rationale
for deletion or a link to their one home.
```

**注意它同时查两个方向**：

- **缺失**：配置/默认值/错误/线上字段/事件/公开行为改了，README 和 JSDoc 必须同 diff 更新。
- **多余**：实现叙述、测试走查、**review 历史**、重复的理由 → 标记为删除或链到唯一的家。

> 「多余」这个方向经常被忽略。而它对 AI 特别重要——**AI 写注释的倾向是「多写总比少写好」**，结果是注释里塞满了「这里我们先检查 X，然后如果 Y 就……」的控制流叙述。第 1 层 AGENTS.md 有明文禁止（`Do not narrate control flow`），这里由 review 兜底。

### ③ 核心类型文档匹配

```
Changes to spine or seam vocabulary update the appropriate subsystems page and any
type-equiv entry. **Internal types need no catalog entry.**
```

最后一句是**范围限定**：内部类型不需要目录条目。**没有这句，AI 会给每个新增的内部 interface 都要求一个 subsystems 条目**，把文档撑爆。

### ④ 注册必须清理

```
Verify each new registry contribution passes the disposal tests required by packages/AGENTS.md.
```

对应 AGENTS.md 的核心 Convention：**`Registrations are effects`**——每个贡献都走 `ctx.effect()`/`ctx.on()`，注册返回 disposer。**注册了但没清理，是插件化架构里最典型的泄漏**，而且它只在 HMR 或反复挂载时才暴露。

### ⑤ 不变量伴随物必须有语义 —— 这条最精细

```
For every touched ./invariant, require an owner event-stream or mutable-data relationship
at the point where that package can observe it; service or method presence, plugin metadata
or effects, and fixed pure examples belong in type, load, or unit tests.
**Accept an empty installer when its package-specific reason establishes that no plausible
runtime relationship exists; do not demand an invented check merely to eliminate emptiness.**
```

背景：`packages/AGENTS.md` 规定**每个包都拥有一个 `./invariant`**，要么检查一个事件/数据关系，要么给空安装器一个包特定的 `No runtime invariant:` 理由。

这条 review 规则做了两件事：

| 方向 | 内容 |
|---|---|
| **反对空洞的检查** | 「服务或方法是否存在」「插件元数据」「固定的纯例子」**不算**运行时不变量，那些属于类型/加载/单元测试 |
| **反对为了非空而编造** | **「不要仅仅为了消除空白而要求一个编造出来的检查」** |

> **第二条是这份 skill 里我最欣赏的一条。** 「每个包都要有 invariant」这个规则有一个显然的退化方式：**为了让门禁变绿，写一个毫无意义的检查**（比如断言某个 service 存在）。这条 review 规则明确把「带理由的空」置于「编造的非空」之上。
>
> 这和第 3 层「**Alternatives are recorded, never invented**」（备选方案是被记录的，不是被编造的）是**同一条原则的第二次出现**：
>
> **宁可有一个有理由的空缺，也不要一个编造出来的填充。**
>
> 这条原则在 AI 协作里价值极高，因为**AI 填空的能力远强于它承认「这里不需要填」的意愿**。

### ⑥ 必需证据存在

```
Verify the author ran the relevant local checks for the diff and that CI covers the
exhaustive matrix; review the semantic gaps neither can detect.
```

**这一条正是第 5 层交下来的职责。** fast-local-git-hooks note 的 Consequences 说得很明白：

> Agents must select relevant behavioral evidence, **reviewers must evaluate whether that selection matches the diff**, and CI supplies the comprehensive signal.

**hook 变快的代价，是 review 多了「评估证据选择是否匹配改动」这一项。** 两层的记录互相咬合。

---

## 5. 十四项人工检查：为什么机器做不到

这是回答「为什么还需要 review」最有力的材料——**逐项都是语义推理，没有一项能被门禁覆盖**。

### 挑五项最能说明问题的

#### 「Capability and consumer fit」——查 API 不必要的膨胀

```
trace every current consumer, then flag consumer-specific behavior leaking into the
interface … Flag the inverse too: **a new public method on a generic service (registry,
session, agent) whose only caller is one internal consumer is an unnecessary API
expansion** — require a private capability closure handed to that consumer at construction instead.
```

**「一个通用服务上的新公有方法，如果只有一个内部调用者，就是不必要的 API 膨胀」**——而且给了替代方案：**在构造时把一个私有能力闭包交给那个消费者**。

> 这是纯语义判断：「只有一个调用者」这个事实机器能查（甚至 knip 能查未使用），但**「因此它不该是公有 API」这个结论需要理解设计意图**。而且它双向查——既查「消费者特定行为泄漏进接口」，也查反向的「为一个消费者开了个公共口子」。
>
> **这条对 AI 写的代码尤其对症**：AI 倾向于「加个 public 方法最省事」，而不是设计一个窄的能力闭包。

#### 「Scope, ownership, and necessity」——查投机性泛化

```
map each abstraction, state machine, option, defensive copy, and compatibility path to
its current contract, production consumer, and owning plugin or service. **Challenge
unrelated features and speculative generality.**
```

**每个抽象、状态机、选项、防御性拷贝、兼容路径，都要能映射到「当前契约 + 生产消费者 + 拥有它的插件」。**

> 「投机性泛化」（speculative generality）是 AI 最典型的产出模式之一：它会为了「以后可能需要」而加一个策略参数、一个 abstract 基类、一个配置项。**每一个单独看都合理，累积起来就是一个没人理解的代码库。**
>
> 这条要求的证明责任是「**当前**的生产消费者」——没有当前消费者的抽象，就是投机。

#### 「Configuration and public choices」——每个默认值都要有证据

```
ask what current-consumer evidence or prior art supports each default, public operation
set, format, or imported external concept. **Require an explicit choice or deferral when
that evidence is absent.**
```

**证据缺失时，要求「一个明确的选择或一次明确的推迟」**——不允许「就这么定了」的默认值悄悄进入代码。

这和 AGENTS.md 的 Convention 呼应：**`No hardcoded tunables in plugins`** + **`a DEFAULT_* constant or test hook is not configurability`**。

#### 「Model perspective」——从模型的视角审查

```
inspect the exact prompts, tool schemas, results, and diagnostics the model receives
across affected modes. **Flag concepts outside the model's task**, then verify stable
text verbatim and dynamic behavior through snapshots or end-to-end coverage.
```

**「标记出不属于模型任务的概念」**——这是一个只有 agent harness 才会有的 review 项，但思路可以泛化：**审查「被审查对象实际收到的输入」，而不是「我们以为它收到的」**。

> 具体场景：一个改动往 system prompt 里加了一段关于内部实现的说明。它不会让任何测试失败，但**它污染了模型的 context**——模型开始考虑不属于它任务的概念。这是 prompt 工程里的真实 bug 类型，而**只有人（或另一个 AI）读一遍完整 prompt 才能发现**。

#### 「Enforcement」——沿拒绝路径找绕过者

```
follow every denial path to the operation that executes it; exercise direct and alternate
callers that **can bypass schemas, prompts, facades, wrappers, or listener ordering**.
```

**这是十四项里最难自动化的一项。**

它要求：找到每一条「拒绝」逻辑，追到真正执行操作的地方，然后**主动去找能绕过它的备用调用方**——绕过 schema 的、绕过提示词的、绕过门面的、绕过包装器的、**利用监听器顺序的**。

> 这是安全审查的标准思路（找绕过路径），但被写进了日常 review 清单。**任何静态分析都做不到这一点**，因为「能绕过」是关于**调用图 + 语义等价性**的联合判断。
>
> 举个具体例子：`tools/pre-execute` 上挂了一个权限门。绕过方式可能是——直接调用底层执行器、注册一个更靠后的监听器覆盖决定、走一个内部门面绕开工具层。**要发现这些，必须理解整个扩展点体系。**

### 另外几项的一句话价值

| 检查项 | 为什么机器做不到 |
|---|---|
| **Intent and interface contracts** | 「实现是否匹配 PR 描述和 Agent Note」——需要读三份东西并比对意图 |
| **Lifecycle and concurrency** | 竞态、await 期间的取消、独立错误上报、回调隔离、**重入前的所有权**、完整 detach、静默销毁 |
| **Borrowed and derived state** | 「这个保留的值是借来的还是拥有的」——所有权语义 |
| **Bounds cover the final operation** | 边界检查要覆盖**最终**产出（含包装器和元数据）；探测极小/精确边界、**超大单块**、**多字节文本对字节上限的影响** |
| **Real entry path** | 「测试是否走了真实的 Loader/bin/worker/子进程」——手工挂载的插件抓不到非法 Loader 导出 |
| **Test strength** | 断言必须在目标回归上失败，且验证外部状态而非复述实现 |
| **Invariant lifecycle and negative controls** | **刻意的非法用例必须通过真实 runner 因目标规则而失败** |
| **Implemented Agent Notes match shipped reality** | 落地时必须把 proposed note 改写成现在时（第 3 层的机械化改写） |
| **Transcript changes** | 「把期望输出的 diff 当作行为变更审查，不是格式噪音」 |
| **Bilingual changes** | **「绿的配对哈希不证明翻译质量」** |

其中「Lifecycle and concurrency」直接外链到 `docs/defensive-patterns.md`——**一份只有 518 词、7 条规则的文档，每条都是真实翻过或差点翻过的车**：

```
1. 独立上报正交结果         （进程可以同时「超时」且「exit 0」——它捕获了信号）
2. 双向遵守公开契约         （实现可能抛也可能发 finish error，公开 API 必须归一化）
3. 异步状态不是同步状态     （followup 没有单条完成信号；whenIdle 不是某条消息的结果）
4. dispose 必须到达静默     （kill → await done；**先关监听器再杀进程**）
5. 在分派器里隔离回调异常   （一个坏订阅者不能拖垮核心生命周期）
6. 永不给不可信输出环境变量和可预测路径（擦掉 *KEY*/*SECRET*/*TOKEN*；私有 0700 目录 + 随机名 + `wx`/0600）
7. 用 unlink 删链接形状的路径（Windows junction 上 rmSync 会递归进目标）
```

> **第 1 条和第 4 条最能说明「为什么需要人/AI review」**。
>
> 第 1 条（正交结果独立上报）和第 4 层 `run-gates.ts` 里那个 `formatGateResultReason`「三个事实并列，不许互相掩盖」**是同一条规则的两次应用**——一次写在防御模式文档里作为普遍规则，一次落实在具体代码里。**门禁能查前者吗？不能。但 reviewer 读到 `if (timedOut) return ... else if (exitCode)` 这种嵌套，立刻知道违规了。**
>
> 第 4 条那个「**先关监听器再杀进程**」的顺序要求，是纯粹的语义顺序约束，任何 lint 都无法表达。

---

## 6. 分工声明：门禁不是替代 review，而是为它腾出空间

这是这一层最关键的一句话，藏在 Reporting findings 里：

```
Separate blockers from suggestions and **omit issues already enforced by a green gate**.
```

**「门禁已经绿了的问题，review 不要再提」。**

这一句话把 review 的**输入面积**压缩了一个数量级：

```
门禁把「事实类问题」清零
  格式 / 类型 / 死链 / 中英配对 / 覆盖率 / 死代码 / 词数 / 包清单 / 归档哈希 …
                    ↓
review 只剩语义问题
  意图 / 生命周期 / 所有权 / 必要性 / 绕过路径 / 模型视角 / 测试强度 …
                    ↓
少量 reviewer（含 AI）扛得住高吞吐
```

> **这条规则是双向的**：它既让 review 更轻，也**制造了补门禁的动力**——如果某类问题反复在 review 里被提，正确反应是「给它写个门禁」，因为写了之后 review 就不用再提了。**这就是那条回边的激励机制。**

---

## 7. 他们自己明确否认「测试和覆盖率就够了」

回击「是不是全靠测试 lint」的最有力证据，全是原文：

| 原文 | 出处 | 意思 |
|---|---|---|
| `automated checks do not establish those properties` | 硬阻断 ① | 自动检查建立不了文字的准确性/质量 |
| `Coverage is necessary but not evidence that the scenario is correct` | Test strength | **覆盖率必要，但不证明场景正确** |
| `rather than restating the implementation or trusting an agent's report` | Test strength | **不要复述实现，也不要相信 agent 的自我报告** |
| `a green pairing hash does not prove translation quality` | Bilingual | 哈希绿了不代表翻译对 |

而第 4 层那份 quality-gates Agent Note 的 Problem 段是最直接的现场证据：

> This codebase is developed primarily by coding agents… Early evidence: tests that didn't typecheck shipped (vitest doesn't typecheck) and **were only caught by a review**.

> **一箭双雕**：既承认「主要由 coding agent 写」，又承认 **review 抓到了门禁漏掉的 bug**——然后他们才去补门禁（把 typecheck 加进 CI）。
>
> **这就是「踩坑 → 固化成门禁」闭环的第一现场，有文件、有编号。**

---

## 8. 四条 review 纪律，条条针对 AI 的通病

### ① 反谄媚 —— 这条是整份 skill 最锐利的一句

```
When receiving review, verify each claim and fix or rebut it on technical grounds
**without performative agreement**.
```

**「收到 review 时，逐条核实，然后要么修、要么用技术理由反驳——不许表演性同意。」**

> **`performative agreement`（表演性同意）** 精确命名了 AI 最让人头疼的行为：*「您说得对，我马上改」*，然后改出一个更差的版本，或者压根没理解意见。
>
> 更深的危害是：**表演性同意让 review 失去了纠错功能**。reviewer 提了一个错误的意见，如果 AI 直接照办，那个错误就进了代码库；而如果 AI 用技术理由反驳，reviewer 才有机会发现自己判断错了。**review 是双向的信息交换，谄媚把它降级成单向命令。**
>
> 这条和第 7 层 `responding-to-pr-review-on-a-stack.md` 里那条是配套的：
> > Triage every comment **on the merits** before acting: verify the claim against the code — **a reviewer flagging the right symptom can still misdiagnose the cause.**
>
> **「reviewer 指出了正确的症状，仍然可能误诊病因」**——所以照着 review 意见改，可能修错地方。

### ② Agent Note 不是免死金牌

```
Agent Notes: design rationale. **Treat disagreement with an Agent Note as a design
discussion, not an automatic veto.**
```

**与 Agent Note 分歧要当作设计讨论，不是自动否决。**

> 这条防的是第 3 层的过度僵化：551 份决策记录如果都是不可挑战的，代码库就会被历史决策锁死。而它和第 2 层 `dsh-find-simplifications` 里那条形成了一对：
> > 推翻已记录的决策，其证据必须比原理由更强。
>
> **合起来就是完整的立场：可以推翻，但要举证。**

### ③ 不许相信 agent 的自我报告

```
assertions fail on the intended regression and verify external state, logs, events, or
disposal rather than restating the implementation or **trusting an agent's report**.
```

配合第 7 层那条更狠的：

> a sub-agent's report describes intent, not necessarily what landed… **a guard that passes both ways guards nothing.**

**「两边都绿的守卫什么都没守」**——回归测试必须先证明它在未修复代码上会红。

### ④ 报告的形态也有规范

```
State the defect, location, impact, and evidence. Place a localized defect inline on the
tightest relevant diff range; use a PR-level comment for cross-cutting architecture,
scope, or review-wide synthesis. … Use the existing GitHub review thread for replies.
```

| 要素 | 要求 |
|---|---|
| 内容 | **缺陷 + 位置 + 影响 + 证据** 四件套 |
| 局部问题 | 行内评论，放在**最紧的相关 diff 范围**上 |
| 跨切面问题 | PR 级评论 |
| 回复 | **用已有的 review thread**，不要开新的顶层评论 |

> 「用已有 thread 回复」看似琐碎，实则重要：**顶层评论会让讨论脱离上下文，几轮之后没人知道在说哪一行。** 第 7 层那份 cookbook 甚至给了具体的 API 调用（`gh api repos/{owner}/{repo}/pulls/{pr}/comments/{id}/replies`）。

---

## 9. Review 标准自己怎么演进——回边的完整机制

总览说「review 的漏网之鱼回流成新门禁」，而 review 标准**自身**的演进也有机制。这就是第 7 层 `maintaining-dsh-code-review.md` + `proposed/process/2026-07-13-human-review-skill-maintenance.md`。

### 工作流

```
1. 选窗口内已 merge 且 merge commit 可从 origin/master 到达的 PR
     不可达的（父被 squash 的 stack 分支）/ 超 250 commit 的
     → 记入 skipped-pulls.json，跳过而不中止整轮
2. 收集 merge 前的人类 review 反馈（带 commit 锚点），
   对比「反馈时」与「最终落地」的 patch
     刻意不采集 PR 会话评论——因为 GitHub 现状给不出「抗强推的反馈时基线」
     且排除仅目标分支的改动作为采纳证据
3. 两个独立配置的 reviewer adapter 分类：
     谁写的 / 是否被采纳 / 相对当前 skill 是否已覆盖
4. 主 adapter 起草完整的新 SKILL.md；两个 adapter review 同一个 diff；
   有 blocking finding 就循环，直到两个都通过
5. 跑 doc-sync + lint，才宣告成功
6. 人做终审：读 diff、抽查证据链、三选一（丢弃 / 攒批 / 提升为 PR）
```

### 这个机制的六条设计，每条都在防一种失效

| 设计 | 防什么 |
|---|---|
| **两个 adapter 字节相同就拒绝运行** | 「互相 review」退化成自我确认剧场 |
| **全 provider 故障不许折叠成「无候选」** | **静默失效伪装成正常** |
| **不许原样提交 adapter 输出**（收紧措辞、删只在源 PR 语境下成立的例子、并入已有规则 都是「预期动作」） | 失去 workflow 依赖的「reviewer 判断」 |
| **不许因为「reviewers approved」就放行**，人必须自己读 diff | AI 批准被当作权威 |
| **必须抽查至少一条**：那条被引用的人类评论真的支持这条新规则吗？那个 PR 真的采纳了吗？ | 证据链断裂而无人察觉 |
| **「今天没候选」是正常，且不发通知**（避免告警疲劳） | 把「没变化」误当作「卡住了」 |

### 它防的两个具体腐烂方式（note 的 Problem 段）

> **Treating every comment as a lesson produces checklist bloat**; treating merge thread resolution or an author's "fixed" reply as **proof of adoption** promotes feedback that the final code may not implement.

| 错误做法 | 后果 |
|---|---|
| 把每条评论都当成教训 | **清单膨胀**（然后没人读，然后 review 质量下降） |
| 把「thread 已解决」或作者回复「已修」当作采纳证据 | 提升了最终代码**其实没实现**的意见 |

**所以它比对的是「反馈时的 patch」和「最终落地的 patch」**——采纳与否由代码证明，不由对话证明。

> **这是整套体系里「怀疑一切自我报告」原则的最高级形式**：连「作者说改了」都不算证据，要看代码。

### 一个诚实的取舍：机制在仓库外

> The tool source, adapter binaries, provider credentials, and intended daily scheduler are kept **private to the maintainer's machine** rather than committed to this repository.

理由（Alternatives considered 里写了）：

> **Ship the tool inside this repository.** Rejected for a single-maintainer scope: repository maintenance overhead (typecheck, lint, coverage, cross-cutting refactors) would exceed the value of committed source and review history. **Retained option for a later handoff.**

而**风险也被记录了**：note 的 Risks 段有「single-maintainer bus factor」，且交接必须写新 Agent Note：

> **Do not silently transfer the tool** — the "single-maintainer bus factor" in the Agent Note's Risks section is the reason the handoff needs a documented decision.

> 注意这份 note **至今还在 `proposed/`**——它是 25 份未落地提案之一。所以这个机制在仓库的自我记录里是「已提出、部分运行、尚未定案」的状态，**而不是被包装成一个已完成的成就**。

---

## 10. 这一层如何和其他层咬合

### 上游交给它什么

| 层 | 交出什么 |
|---|---|
| **① AGENTS.md** | 31 条 Conventions 作为判据；review 明文要求「test the PR against the root rules」 |
| **② skills** | review 本身就是 skill；`dsh-prose-standard` 被硬阻断 ① 直接调用 |
| **③ notes** | 551 份决策记录作为设计理由；**但分歧算讨论不算否决** |
| **④ scripts** | `change-scope.ts` 提供范围报告；**门禁绿了的不再提** |
| **⑤ hooks/CI** | 把「评估 agent 选的证据是否匹配 diff」交给 review（硬阻断 ⑥） |
| **⑦ cookbook** | `defensive-patterns.md`（7 条 bug 类）、`packages/AGENTS.md`、`maintaining-dsh-code-review.md` |

### 它交出去什么——那条回边

```
review 发现「机器本可以查」的问题
        ↓
写成一条新的 verify-*.ts（第 4 层）
        ↓
下次这类问题在门禁就红了，review 不用再提（`omit issues already enforced by a green gate`）
        ↓
review 的输入面积继续缩小
```

**门禁集合是被 review 喂大的。** 所以：

> **项目越大，review 负担反而越低**——因为踩过的坑都变成了门禁。

而 postmortem 是另一条支流：`docs/postmortem/` 有 4 份事故报告，且 `packages/AGENTS.md` 的规则直接链接它们作为理由：

```
Plugin exports: … Mixing the forms makes the Loader discard the function plugin's
namespace ([postmortem](../docs/postmortem/0001-acp-default-export-drops-inject.md)).
```

**事故 → postmortem → 提升为 `packages/AGENTS.md` 里的规则 → 被 review 的「Real entry path」检查项覆盖。** 这是完整的一条从事故到常态化检查的路径。

---

## 11. 这一层的真实模型：为什么扛得住

把前面的事实串起来，答案有三个支柱：

```
支柱一：输入面积被压缩了一个数量级
        约 60 个门禁把事实类问题清零，且明文规定「绿了的不再提」

支柱二：reviewer 可以是 AI
        review 标准是一份 skill（1073 词，写给 AI 执行）
        → review 侧和产出侧用同一种方式扩容

支柱三：人只做仲裁
        人的职责是「语义审查 + 决策合理性 + 解决歧义」，不是逐行看代码
```

**而支柱二是真正的关键**：如果 review 只能由人做，那么产出速度提升 10 倍必然导致 review 成为瓶颈——**无论门禁多完善**。只有当 review 也能被 AI 执行，两侧才能同步扩容。

> **这也解释了为什么 review 标准必须写成 skill 而不是 wiki 页面**：
>
> - wiki 页面是给人读的，人读完会内化。
> - skill 是给 AI 读的，**AI 每次 review 都要重新读一遍**——所以它必须自包含、必须密度极高、必须每条都可判定。
>
> 1073 词装 20 条规则、7 个外链，就是这个约束的产物。

---

## 12. 落到自己项目：六步

1. **先立「机器查不了什么」清单**，而不是「要查什么」清单。前者才是 review 的真正职责。
2. **明文约定「门禁绿了的不再提」**。这一条同时降低 review 负担并制造补门禁的动力——**没有它，review 会永远在提格式问题。**
3. **写下优先级**：正确性/生命周期/安全 优先于风格；**一个有据可依的 blocker 好过一堆 nit**。否则 AI reviewer 会用数量伪装质量。
4. **加反谄媚条款**：收到 review 逐条核实，该反驳就用技术理由反驳。这是 AI 参与 review 时最必需的一条。
5. **把 review 标准写成给 AI 执行的文档**（自包含、每条可判定、细节全外链），而不是给人看的规范页面。
6. **让 review 发现的可机检问题回流成门禁**。定期问一句：「这个月 review 里重复提过的意见，哪些该变成脚本？」

> **两条最有迁移价值、且和「AI」无关的原则**：
>
> - **「宁可有一个有理由的空缺，也不要一个编造出来的填充。」**（不变量可以为空但要写理由；备选方案可以缺失但要标记）
> - **「两边都绿的守卫什么都没守。」**（回归测试必须先证明它在未修复代码上会失败）

---

## 13. 一页总结

| 问题 | 答案 |
|---|---|
| 规模 | `dsh-code-review/SKILL.md` **1073 词**（8278 字节），6 条硬阻断 + 14 项人工检查 + 7 个权威来源 |
| 为什么这么短 | **不复述任何规则**，只负责「该去查哪条」和「什么只能靠判断」；细节全外链 |
| 开篇第一句 | **「这是指引，不是完整清单」**——防止 AI 打勾即通过 |
| 优先级 | 正确性/生命周期/安全/必需行为 **优先于风格**；**一个有据可依的 blocker 好过一堆 nit** |
| 前置动作 | 验证 live base + 精确 head，跑 `change-scope`，**改基或 merge 后重跑** |
| `change-scope` 的严格性 | 歧义 ref 拒绝、merge-base 必须唯一、非 UTF-8 路径名报错、`--no-renames`、不干扰用户 git 状态 |
| 工具的边界 | **`does not replace semantic review`**——机器摆事实，判断留给下游 |
| 最精细的一条硬阻断 | 不变量伴随物：**「不要仅仅为了消除空白而要求一个编造出来的检查」** |
| 最难自动化的一项 | **Enforcement**：沿每条拒绝路径找能绕过它的备用调用方（schema/提示词/门面/包装器/监听器顺序） |
| 分工声明 | **`omit issues already enforced by a green gate`**——压缩输入面积 + 制造补门禁的动力 |
| 他们自己的否认 | 「覆盖率必要但不证明场景正确」「不要相信 agent 的自我报告」「哈希绿了不代表翻译对」 |
| 最锐利的一条纪律 | **反谄媚**：`without performative agreement`——不许表演性同意，该反驳用技术理由 |
| 不僵化的保证 | 与 Agent Note 分歧算**设计讨论**不算自动否决（配套：推翻它的证据必须更强） |
| 标准怎么演进 | 每天跑，对比「反馈时 patch」与「落地 patch」，**双独立 adapter 交叉审，人做终审**；**字节相同拒绝运行** |
| 为什么扛得住 | ① 输入面积压缩一个数量级 ② **reviewer 可以是 AI，两侧同步扩容** ③ 人只做仲裁 |
| 那条回边 | review 抓到的可机检问题 → 新 `verify-*.ts` → 门禁集合被 review 喂大 → **项目越大 review 负担越低** |
