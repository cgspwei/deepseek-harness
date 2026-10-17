# 第 4 层详解：scripts —— 把「应该怎样」变成「机器会怎样检查」

> 总览见 [how-deepseek-does-ai-coding.md](how-deepseek-does-ai-coding.md)。本文是第 4 层的完整拆解。
>
> **一句话**：`scripts/` 是这套体系的执行机构。它的设计法则只有一条——**AGENTS.md 里每一条机器可查的承诺，都必须有一条会返回非 0 的命令。**

---

## 0. 立法原则：一句话决定了这一层的全部

这一层的宪法写在 `.agents/notes/implemented/process/2026-06-11-quality-gates.md` 的 `## Decision` 第一句：

> **Every mechanically checkable AGENTS.md promise gets a command that exits non-zero.**

而它的 `## Problem` 解释了为什么必须这样（这段是整个仓库最重要的自述之一）：

> This codebase is developed **primarily by coding agents**. **Agents follow enforced gates far more reliably than prose conventions** and "a lot of work" is not a cost argument when agents do the labor.

**注意它的措辞不是「AI 不可靠」，而是「强制门禁比散文约定可靠得多」。** 这个区分很重要——对人也一样成立，只是 AI 把这个差距放大了：

| | 散文约定（AGENTS.md 里的一句话） | 门禁（exit code 非 0） |
|---|---|---|
| 人 | 大概率遵守，偶尔忘 | 必须遵守 |
| AI | **每次会话都要重新读懂并自愿遵守** | 必须遵守 |

AI 的问题不是「不听话」，而是**它没有「上次被骂过」的记忆**。所以对 AI 而言，一条没有门禁的规则，等于一条**每次都要重新赌一次**的规则。

---

## 1. 规模：128 个脚本，其中 27 个 `verify-*`

实测 `scripts/*.ts`（不含 `.spec.ts`）：

| 类别 | 数量 | 说明 |
|---|---|---|
| `verify-*.ts` | **27** | 显式命名的校验门禁 |
| `gen-*.ts` | **13** | 生成器（catalog、graph、notices…） |
| 其余（共享库、工具、构建） | 约 88 | `agent-note-tree.ts`、`clean.ts`、`change-scope.ts`、`ts-project.ts` 等 |
| **合计** | **128** | |

但**「27 个 verify」低估了真实门禁数量**。因为很多门禁不叫 `verify-*`（`knip`、`publint`、`constraints`、`duplication`、`doc-typecheck`……）。按调度器里实际登记的 gate id 去数：

```
run-gates.ts 里的 pnpmScript/pnpmExec 调用：96 处
去重后的唯一 gate id：约 60 个
其中 doc-sync 这一个聚合就有 28 个叶子门禁
```

**所以真实规模是「约 60 个可独立执行的门禁，组织成 13 个聚合模式」。**

---

## 2. 四类脚本

### ① 门禁调度器：`run-gates.ts`（唯一一个，但最关键）

它的 JSDoc 一句话划清了和 `package.json` 的分工：

> **Package scripts own public aggregate names; this runner owns their validated dependency graphs, scheduler environment, and process diagnostics.**

**`package.json` 拥有「对外的名字」，调度器拥有「依赖图、调度环境、进程诊断」。** 这是一个很讲究的切分——外部（人、CI、文档）只需要知道 `pnpm run doc-sync`，不需要知道它背后是 28 个进程和一张依赖图。

### ② 文档一致性门禁（数量最多）

`doc-sync` 这一个聚合的 **28 个叶子门禁**（从代码提取的完整列表）：

```
 1. doc-typecheck                      文档里的 ts 代码块必须编译
 2. cordis-catalog                     生成的 Cordis API 目录是否新鲜
 3. client-catalog                     客户端目录
 4. export-jsdoc                       每个导出必须有 JSDoc（含 @param/@returns）
 5. tool-catalog                       工具目录
 6. config-catalog                     配置目录
 7. persistence-catalog                持久化目录
 8. doc-graphs                         文档里的图
 9. scoped-events                      作用域事件
10. markdown-wrap                      一段一行（软换行）
11. markdown-links                     相对链接与 #锚点 不能死
12. public-repository-links            公开仓库链接
13. doc-refs                           文档引用
14. package-paths                      包路径
15. config-source-ownership            配置源归属
16. package-readme-model-experience    README 的 Model Experience 必需结构
17. mermaid                            mermaid 图能渲染
18. agent-note-classification          Agent Note 的 class 合法
19. agent-note-format                  Agent Note 的格式（11 类检查）
20. archived-agent-notes               归档冻结哈希
21. type-equivalence                   文档里粘贴的类型没漂移
22. skill-invocation-metadata          skill 调用元数据双客户端语义一致
23. translation-prompt                 翻译提示词
24. translation-pairing                中英文配对哈希
25. doc-budgets                        词数预算
26. docs-site-projection               文档站点投影
27. docs-site-build                    VitePress 构建（兼死链检查）
28. package-readme-limitations         README 的限制小节
```

> **28 个门禁里有 25 个在管文档。** 这和第 2 层「11 个 skill 里 6 个管文字」、第 7 层「9 份 cookbook 里 3 份管流程」是同一个现象的第三次出现：
>
> **在 agent-first 的仓库里，文档不是代码的附属品，而是和代码同等的、需要同等强度机器约束的生产物。**
>
> 原因很直接：AI 读文档来理解仓库。文档错了，AI 就系统性地错。而人读错文档还有「感觉不对，去看看代码」的本能，AI 更倾向于信任文档。**所以文档的正确性对 AI 比对人更关键。**

### ③ 代码/结构不变量

分布在 `hygiene` 和 `ci-static` 等聚合里：

```
runtime-closure                可执行部署清单必须提供依赖图里的每个 workspace peer
constraints                    workspace 规则（private / cordis peer+dev / 统一版本 / ESM）
package-invariants             包清单不变量
built-package-invariants       构建产物的不变量
dsh-package-licenses           许可证
cordis-config                  cordis.yml 里的 bare plugin 必须在 resolver manifest 的 dependencies 里
optional-dependency-imports    可选依赖的导入方式
node-next-types                NodeNext 消费者能否解析构建出的声明
knip                           死代码 / 死依赖
publint                        包发布正确性
module-graph                   模块图
client-domain-graph            客户端领域图
rescope-vendor                 vendored 包改名一致性
duplication                    跨文件 TypeScript 克隆检测
```

### ④ 纯工具与生成器

- **13 个 `gen-*.ts`**：生成 catalog（tool / config / persistence / cordis / client）、module-graph、doc-graphs、third-party-notices、translation-brief。**生成 + 校验成对出现**：`gen-tool-catalog.ts` 生成，`verify-tool-catalog` 检查是否新鲜。
- **`clean.ts`**：带安全校验的清理（见 §6）。
- **`change-scope.ts`**：给第 6 层 review 用的改动范围报告。
- **共享库**：`agent-note-tree.ts`（第 3 层的封闭集合）、`ts-project.ts`、`coverage-exempt.ts`。

---

## 3. `run-gates.ts` 的调度设计（这一层最有工程含量的部分）

### 13 个聚合模式

```ts
export type Mode =
  | 'ci-primary'      | 'ci-linux-primary'  | 'ci-static'
  | 'ci-lint-contracts-ready'  | 'ci-coverage'  | 'ci-snapshot'
  | 'ci-artifacts'    | 'ci-consumers'
  | 'ci-windows-blocking' | 'ci-windows-complete' | 'ci-windows-observational'
  | 'node-compat'     | 'check-all'  | 'doc-sync'
```

**模式是一个封闭联合类型**，`parseMode` 用穷举 switch 解析，非法值报错时**把全部合法值列出来**：

```ts
throw new Error(`run-gates: expected mode ci-primary | ci-linux-primary | … | doc-sync, got ${JSON.stringify(raw)}.`)
```

> **报错信息里列出全部合法值**——这是个小细节但反复出现在这个仓库里。对 AI 尤其有用：它不需要再去读源码找合法值，报错本身就是文档。

### 依赖图：`needs` + 三重校验

每个 gate 可以声明 `needs`，调度器在**执行任何子进程之前**做三件事（`validateGateGraph`）：

```ts
1. 空图 → 报错（'gate graph has no gates'）
2. 重复 id → 报错
3. 未知依赖 → 报错（'depends on unknown gate'）
4. 循环依赖 → 报错，且把完整环路径打出来（'a -> b -> c -> a'）
```

`findDependencyCycle` 是标准的三色 DFS，但**返回的是环的完整路径而不是布尔值**：

```ts
const cycleStart = active.get(id)
if (cycleStart !== undefined) return [...path.slice(cycleStart), id]
```

> **为什么值得讲**：一个只说「有循环依赖」的错误，人要自己去图里找；一个直接打印 `a -> b -> c -> a` 的错误，AI 能一次改对。**门禁的报错质量决定了它是「帮手」还是「障碍」。**

### 一个真实的依赖声明，附带解释性注释

```ts
// The prepared typecheck and build both drive Client tsc, while build also
// repeats the Host contract pass. Wait for all three consumers so build
// neither races tsbuildinfo nor replaces declarations while they are read.
pnpmScript('build', 'build', { needs: ['typecheck', 'lint', 'doc-typecheck'] }),
```

**`build` 依赖 `typecheck`、`lint`、`doc-typecheck`，不是因为逻辑上需要，而是为了避免竞态**——三者都会驱动 tsc，`build` 如果并行跑会和它们抢 `tsbuildinfo`，或者在它们读声明文件时把声明替换掉。

> 这条注释是「解释非显然约束」的范例。没有它，下一个 agent 看到「build 依赖 lint」会觉得莫名其妙，然后「优化」掉它——然后 CI 开始随机失败，而且难以复现。

### 有界并发：为什么本地封顶 4

```ts
export function defaultConcurrency(selectedMode: Mode, total: number, available = availableParallelism()): ConcurrencyDefault {
  if (selectedMode === 'ci-consumers') return { workers: total, source: 'ci-consumers gate count' }
  // Local modes cap workers: several doc gates each build a full ts.Program,
  // so an uncapped default on a large host trades wall clock for memory blowups.
  const localCap = selectedMode === 'check-all' || selectedMode === 'doc-sync'
  const modeLimit = localCap ? Math.min(4, available) : available
  return { workers: Math.min(total, modeLimit), source: localCap ? `${available} available CPU(s), ${selectedMode} cap 4` : `${available} available CPU(s)` }
}
```

三层策略：

| 模式 | 并发 | 理由 |
|---|---|---|
| `ci-consumers` | **= 门禁总数**（全并行） | CI 机器专用，就是要最快 |
| `check-all` / `doc-sync`（本地） | **封顶 4** | **好几个文档门禁各自要建一整个 `ts.Program`**，无上限会在大机器上把内存打爆 |
| 其他 | = CPU 数 | |

而且**并发来源会被打印出来**：

```
run-gates: doc-sync running 28 gate(s) with 4 worker(s) from 8 available CPU(s), doc-sync cap 4.
```

> **「从哪来的 4」被写进日志**——所以有人抱怨慢的时候，第一眼就能看出是被 cap 住了还是 CPU 少。可诊断性是这一层被反复照顾的属性。

覆盖率那两个 gate 的并发切分更精细，注释解释了为什么不是各拿全额：

```ts
// DSH_COVERAGE_MAX_WORKERS is the lane's worker budget, so the two parallel
// gates split it instead of each claiming it whole … The exempt
// gate's wall clock is dominated by its longest single file, so it takes the small share.
const exempt = Math.max(1, Math.floor(total / 3))
const instrumented = Math.max(1, total - exempt)
```

**豁免那条 lane 的墙钟时间由「最长的那个单文件」决定，所以它只拿 1/3**——这是真实测量出来的分配比，不是猜的。

### 依赖失败的传播：`skipped` 也算失败

调度器把「依赖失败导致没跑」标记为 `skipped`，并且**明确记录是哪个依赖挂了**：

```ts
error: `dependency failed or skipped: ${failedDeps.join(', ')}`
```

而最终退出码的判定是：

```ts
return results.some(result => result.gate.allowFailure !== true && (result.status === 'failed' || result.status === 'skipped')) ? 1 : 0
```

> **`skipped` 与 `failed` 同等对待**（除非显式 `allowFailure`）。这堵住了一个经典漏洞：**「因为上游挂了所以我没跑」被当成「我没问题」**。第 5 层 CI 里的 `all-checks-passed` 防的是同一类病，只是在更高一层。

还有一条防御性断言：

```ts
if (gate === undefined) throw new Error('run-gates: validated graph stalled without a failed dependency.')
```

**图已经被校验过了，所以「卡住但没有失败的依赖」在逻辑上不可能发生**——但还是加了断言。这符合 AGENTS.md 里「misconfiguration fails loud」的要求。

### 失败原因：三个事实并列，不许互相掩盖

```ts
/**
 * Format every independently observed failure fact for the aggregate summary.
 * @returns error, exit, and signal facts without allowing one to hide another.
 */
export function formatGateResultReason(result: GateResult): string {
  const facts: string[] = []
  if (result.error !== undefined) facts.push(result.error)
  if (result.exitCode !== null) facts.push(`exit ${result.exitCode}`)
  if (result.signalCode !== null) facts.push(`signal ${result.signalCode}`)
  return facts.length === 0 ? 'no exit code or signal' : facts.join(', ')
}
```

> **「不允许一个事实掩盖另一个」**（`without allowing one to hide another`）——被 SIGKILL 杀掉的进程 exitCode 是 null，如果只报 exit code 就会显示成「no exit code」，真正的原因（signal）就丢了。**三个都报，让诊断信息完整。**
>
> 这条设计和第 6 层 review 清单里「Enforcement：沿每条拒绝路径找能绕过它的备用调用方」是同一种思维方式：**假设信息会在某条路径上丢失，然后把那条路径堵上。**

### 跨平台：不生成 shell

```ts
function pnpmInvocation(args: string[]): Pick<Gate, 'command' | 'args'> {
  const entrypoint = process.env.npm_execpath
  if (entrypoint === undefined || entrypoint === '') {
    throw new Error('run-gates: npm_execpath is unavailable; invoke the runner through a pnpm package script.')
  }
  // Windows cannot spawn the pnpm.cmd shim directly; the JavaScript entrypoint keeps every host shell-free.
  return { command: process.execPath, args: [entrypoint, ...args] }
}
```

**不调 `pnpm` 命令，而是用当前 Node 去执行 pnpm 的 JS 入口**——因为 Windows 不能直接 spawn `pnpm.cmd`。**「keeps every host shell-free」**：全平台都不经过 shell，也就没有引号转义、路径空格、`PATH` 差异这些问题。

---

## 4. 关键设计：结构性唯一事实来源

这一层反复出现同一个模式：**把「合法值的集合」写成代码里的一个 `as const` 数组，并在注释里写明「扩展它是刻意行为」。**

### 例一：Agent Note 的 class（第 3 层的门禁基础）

```ts
/**
 * The closed set of Agent Note classes (nested folder under each lifecycle). Adding a
 * class is a deliberate act: extend this list AND the README's Classification
 * section. The gate rejects any folder not listed here.
 */
export const AGENT_NOTE_CLASSES = ['feature', 'bug-fix', 'simplification', 'architecture', 'process', 'testing'] as const
```

三层含义：

1. **封闭集合**：不在列表里的文件夹 → 门禁红。
2. **扩展是刻意行为**：加一类必须**同时**改这个列表**和** README 的 Classification 节。
3. **注释解释了为什么连 lifecycle 集合也要封闭**：

```ts
// The lifecycle set is closed too: any directory under .agents/notes/ that is not
// a known lifecycle would otherwise hold Agent Notes invisible to the walk below.
```

**如果 lifecycle 不封闭，有人建个 `draft/` 文件夹，里面的 note 就对所有门禁隐形了**——不会报错，只是不被检查。**这正是最危险的失效模式：静默地不生效。**

### 例二：聚合模式

`Mode` 联合类型 + 穷举 switch。**加一个模式必须改类型、改 `parseMode`、改 `gatesForMode`**，TypeScript 会强制你改全（穷举 switch 少一个 case 就编译不过）。

### 例三：词数预算清单

`scripts/doc-budgets.manifest.json` —— 只有 9 个条目：

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

**只有 9 份「常驻文档」有预算**，cookbook（9838 词）、skills（13578 词）、Agent Notes（551 份）全部无预算——这正是第 2、7 层讲的「常驻的极简 + 按需的极详」在门禁配置里的体现。**预算清单本身就是「哪些文档是常驻的」这个判断的唯一记录。**

---

## 5. 剖析一个完整门禁：`verify-doc-budgets.ts`

选它是因为它足够短（60 行）能完整读完，而且展示了这层门禁的所有典型特征。

### 头部注释就是完整契约

```ts
/**
 * Enforce `wc -w`-style ceilings from `scripts/doc-budgets.manifest.json`.
 * Missing files and invalid ceilings fail; `--list` reports current usage.
 * Only listed standing docs are budgeted. Ceilings ratchet down with at least
 * 5% headroom; raising one requires the justification defined in `docs/AGENTS.md`.
 */
```

四句话说清：查什么、什么算失败、范围是什么、**棘轮方向**（只降不升，升要理由）。

### 它检查三类失败，不只是「超了」

```ts
1. ceiling 不是正整数        → 'ceiling must be a positive integer'
2. 被预算的文件不存在        → 'budgeted file does not exist (renamed or deleted?
                                update scripts/doc-budgets.manifest.json in the same change)'
3. 词数超过 ceiling          → 'N words exceeds the M-word ceiling'
```

> **第 2 类才是精髓。** 光查「超了」的话，删掉那个文件就能让门禁变绿——**约束被删除本身消灭了**。所以「清单里的文件必须存在」和「文件必须不超字数」是同等重要的两条。**这是所有清单式门禁都必须配的一条，极容易漏。**

### 报错信息里带解决方案，且有优先级

```ts
`${path}: ${words} words exceeds the ${ceiling}-word ceiling — relocate or condense per docs/AGENTS.md (raising the ceiling requires justification in the PR)`
```

**「relocate or condense」（先搬走，再压缩）**——和 `docs/AGENTS.md` 里的三步一致：

```
1. Relocate  搬到该去的 tier
2. Condense  压缩
3. Raise     只有在内容确实需要空间时才提升上限，且要在 PR 里说明
```

**顺序本身是规则**：提升上限是最后手段。而门禁的报错信息复述了这个顺序，所以**AI 看到报错就知道该按什么顺序尝试**，而不是直接去改 manifest（最省事但最坏的做法）。

### 结尾还有一句指路

```ts
console.error('\nSee docs/AGENTS.md for the documentation standard and the relocation-first rule.')
```

### `--list` 模式：门禁兼报表

```ts
rows.push(`${words <= ceiling ? 'ok  ' : 'OVER'}  ${String(words).padStart(6)} / ${String(ceiling).padEnd(6)} ${path}`)
```

`pnpm run verify-doc-budgets --list` 输出当前全部用量。**同一个脚本既是门禁又是观测工具**——所以「还剩多少余量」这个问题永远有权威答案，不需要另写一个统计脚本（那样两者又会漂移）。

---

## 6. `clean.ts`：一个「危险操作」怎么被约束

删除文件是脚本里最危险的操作，尤其当写脚本的是 AI。这个脚本的防护值得单独看。

### 类名就声明了边界

```ts
/** Plans and removes repository-owned build output without crossing the repository boundary. */
export class RepositoryCleaner {
```

**「不跨越仓库边界」**写在类的 JSDoc 里，是这个类的核心契约。

### 先完整规划，再统一执行

```ts
async clean(): Promise<string[]> {
  const targets = await this.plan()
  // Planning validates every target first, so an unsafe orphan prevents all deletion.
  for (const target of targets) await rm(target, { recursive: true, force: true })
  return targets.map(target => repositoryPath(this.root, target))
}
```

> **「规划阶段会校验每一个目标，所以一个不安全的孤儿就能阻止全部删除。」**
>
> 这是 plan/apply 分离的经典价值：**要么全部安全，要么一个都不删**。如果边扫边删，扫到第 5 个发现不安全时，前 4 个已经没了。

### 用 `realpath` 防符号链接逃逸

```ts
const canonicalRoot = await realpath(this.root)
```

解析真实路径再比对，防的是「符号链接指向仓库外，删除时跟着链接跑出去」。

### 已知残留是白名单

```ts
const knownOrphanEntries = new Set(['node_modules', 'lib', '.typecheck'])
```

**只有这三种目录内容才算「可安全清理的残留」**。一个被删掉的包目录里如果还有别的东西，它就是「不安全的孤儿」，会阻止整轮清理——**而不是被顺手删掉**。

---

## 7. 一个反直觉的事实：门禁自己也有测试

```
scripts/*.spec.ts        （run-gates、archived-agent-notes、gen-* 等都有）
scripts/test-invariants.ts
```

`run-gates.ts` 为了可测试，把调度核心做成了**依赖注入**的纯函数：

```ts
export async function runGates(
  gates: Gate[],
  maxActive: number,
  execute: GateExecutor,      // ← 注入执行器
  observe: ResultObserver = () => {},
): Promise<GateResult[]>
```

测试传入一个假的 `execute`，就能验证「依赖失败时下游被正确标记为 skipped」这类调度逻辑，**不用真的跑 28 个子进程**。

而真实执行器 `runGate` 是单独导出的一个函数，注释写明它是「真实的 shell-free 子进程边界」。

> **这条呼应了 AGENTS.md 里的一条 Convention**：
> > **Explicit > implicit at package boundaries**
>
> 以及 quality-gates note 里那句自我认知：
> > **The gates themselves are code to maintain; config changes are reviewed like any change.**
>
> **门禁是代码，所以门禁也要被 review、也要有测试。** 这挡住了一个很常见的堕落路径：门禁脚本被当作「运维脚本」，写得很脏、没测试、没人 review，最后成为整个仓库最不可靠的部分。

---

## 8. 门禁的副作用被预判并封堵

这是第 3 层已经提过、但值得在这层再讲一次的点，因为它是「门禁设计成熟度」的最好指标。

**逐文件 100% 覆盖率有两个恶性副作用，都被显式处理了：**

### 副作用一：为凑覆盖率删掉防御性代码

quality-gates note 原文：

> Per-file 100% coverage on `packages/*/*/src` (v8); **unreachable defensive guards carry `/* v8 ignore */` with stated reasons instead of deletion.**

**不可达的防御分支用带理由的忽略标记，而不是删掉。** 如果没有这条，100% 覆盖率会系统性地让代码库变得更脆弱——这是个真实存在的、广为人知的反模式。

### 副作用二：无断言测试

> **100%-coverage pressure can produce assertion-free tests** — mutation testing is the planned counterweight.

对策（`proposed/testing/2026-06-11-mutation-testing.md`）诚实地留在未落地状态，且它的 Problem 段把问题说得比谁都狠：

> The per-file 100% coverage gate proves every line *executes* under test — **not that any assertion would notice if the line were wrong.**

> **一个体系能不能长期健康，看它是否在自己的记录里承认自己的机制有副作用。** 这里两个副作用都有文件、有编号，一个有对策（`v8 ignore` + 理由），一个有未落地的提案。

### 还有一条「豁免不能变成后门」的设计

覆盖率有个豁免名单 `scripts/coverage-exempt.ts`，但注释说清了豁免的**理由和边界**：

```ts
// The heavy suites run uninstrumented beside the thresholded gate: their
// compiler- and subprocess-bound fixtures pay a multiple of their runtime
// under v8 instrumentation while contributing nothing the thresholds need
```

**豁免的是「插桩后慢几倍但对阈值毫无贡献」的重型套件，而且它们仍然会跑**（只是不插桩），不是不测了。**豁免的是测量，不是测试。**

---

## 9. 这一层如何和其他层咬合

| 上游 | 这一层做什么 |
|---|---|
| **第 1 层 AGENTS.md** | 每条机器可查的承诺 → 一条 exit 非 0 的命令；`verify-doc-budgets` 直接守住第 1 层的词数上限 |
| **第 2 层 skills** | `verify-skill-invocation-metadata` 守双客户端调用元数据语义一致 |
| **第 3 层 notes** | `agent-note-tree.ts`（封闭集合）+ `verify-agent-note-format`（11 类检查）+ `-classification` + `-archived`（哈希冻结） |
| **第 7 层 cookbook** | `verify-md-links` 守 `#锚点`、`doc-typecheck` 守 34 个代码块、i18n 配对守中英一致 |

| 下游 | 这一层交出什么 |
|---|---|
| **第 5 层 hooks/CI** | 13 个聚合模式，正是 CI job 和 git hook 直接调用的单位 |
| **第 6 层 review** | `change-scope.ts` 提供改动范围报告；**门禁绿了的问题 review 不再提**（`omit issues already enforced by a green gate`） |

**而那条回边（review 漏网之鱼 → 新门禁）落地的地方就是这一层**：quality-gates note 里那句「tests that didn't typecheck shipped and were **only caught by a review**」，就是第一个被 review 喂出来的门禁的由来。

---

## 10. 落到自己项目：六步

1. **从「AGENTS.md 里哪条规则最常被违反」开始**，只给那一条写门禁。不要一上来铺 60 个。
2. **每个清单式门禁都配一条「清单里的文件必须存在」**。否则删文件就能让门禁变绿，约束会被静默消灭。
3. **报错信息里写解决方案，并给出优先级**（先搬、再压、最后才提上限）。AI 会照着报错做，报错写得好等于规则被执行得好。
4. **封闭集合写成 `as const` + 穷举 switch**，并在注释里写「扩展它是刻意行为，要同时改 X 和 Y」。让 TypeScript 帮你强制。
5. **门禁自己要有测试**：把调度/校验逻辑做成可注入的纯函数。门禁是代码，不是运维脚本。
6. **预判门禁的副作用并显式处理**。上覆盖率门禁前，先想清楚「为了凑覆盖率，人们会做什么坏事」，然后把那条路堵上。

> **成本问题的答案**：60 个门禁听起来是巨大的工程量。quality-gates note 的回应是——
>
> > **「a lot of work」is not a cost argument when agents do the labor.**
>
> **写门禁这件事本身就是 agent 干的活。** 门禁的经济学被 AI 改变了：传统上写 60 个 verify 脚本不划算（人力太贵），但当写脚本的也是 agent 时，「应该有多少门禁」的最优解整个变了。

---

## 11. 一页总结

| 问题 | 答案 |
|---|---|
| 立法原则 | **AGENTS.md 里每条机器可查的承诺，都要有一条 exit 非 0 的命令** |
| 为什么必须这样 | 「agent 遵守强制门禁远比遵守散文约定可靠」——AI 没有「上次被骂过」的记忆 |
| 规模 | 128 个脚本；27 个 `verify-*`、13 个 `gen-*`；**约 60 个门禁组织成 13 个聚合** |
| 最大的聚合 | `doc-sync` = **28 个叶子门禁，其中 25 个管文档** |
| 调度器分工 | `package.json` 拥有公开名字，`run-gates.ts` 拥有依赖图/调度/诊断 |
| 图校验 | 空图、重复 id、未知依赖、循环依赖（**打印完整环路径**），全在跑子进程之前 |
| 并发策略 | CI 全并行；**本地 `doc-sync`/`check-all` 封顶 4**（多个文档门禁各建一个 `ts.Program`，会爆内存）；并发来源写进日志 |
| 最重要的失败语义 | **`skipped` 与 `failed` 同等对待**——「上游挂了所以我没跑」不算通过 |
| 诊断设计 | error/exit/signal **三个事实并列，不许互相掩盖** |
| 跨平台 | 不 spawn `pnpm.cmd`，用 Node 执行 pnpm 的 JS 入口，**全平台 shell-free** |
| 封闭集合三例 | Agent Note class（`as const`）、Mode（穷举 switch）、词数预算清单（9 份常驻文档） |
| 最容易漏的一条 | 清单式门禁必须查「**被列出的文件是否存在**」，否则删文件即变绿 |
| 危险操作的约束 | `clean.ts`：plan/apply 分离（**一个不安全孤儿阻止全部删除**）+ `realpath` 防链接逃逸 + 白名单残留 |
| 成熟度指标 | **门禁自己有测试**（调度核心是可注入的纯函数）+ **副作用被预判封堵**（`v8 ignore` 带理由、mutation testing 提案） |
| 成本回应 | 「当干活的是 agent 时，『工作量大』不构成成本理由」 |
