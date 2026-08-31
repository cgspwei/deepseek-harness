# 第 5 层详解：执行时机 —— hooks 与 CI 的边界划分

> 总览见 [how-deepseek-does-ai-coding.md](how-deepseek-does-ai-coding.md)。本文是第 5 层的完整拆解。
>
> **一句话**：门禁写得再好，挂错时机就等于没写。这一层的全部内容是一条边界——**本地只拦「便宜且高置信」的缺陷，CI 拥有全量矩阵**——以及为守住这条边界所做的取舍。

---

## 0. 这一层的核心命题

第 4 层造出了约 60 个门禁。摆在面前的问题是：**哪些在本地跑、哪些在 CI 跑？**

直觉答案是「本地尽量多跑，早发现早修」。这个直觉是**错的**，而且错得很典型。`lefthook.yml` 开头第一行就把正确答案写死了：

```yaml
# Git hooks (lefthook). Keep these local checkpoints fast; CI owns the full
# repository-wide gate matrix.
```

**为什么「本地多跑」是错的**——`2026-07-22-fast-local-git-hooks.md` 的 Problem 段给了三个理由：

> A full pre-push suite therefore **delays every publication**, **amplifies unrelated local flakes**, and **gives no new signal when CI immediately runs the exhaustive matrix again**.

| 代价 | 说明 |
|---|---|
| **延迟每一次发布** | 每次 push 都要等 |
| **放大无关的本地不稳定** | 本地环境的偶发问题会挡住和你改动无关的工作 |
| **不产生新信号** | CI 马上要把同样的东西再跑一遍——**这次运行的信息量是零** |

第三条是关键论证：**如果 CI 一定会跑，那本地跑同一件事就是纯粹的浪费。** 这不是「快 vs 严」的取舍，而是**重复劳动**。

而这条命题在这个仓库有一个前提，Problem 段的第一句就说了：

> **An agent already runs the tests and checks that exercise its change** while commit, push, and CI can each repeat increasingly broad subsets of the same work.

**agent 在提交之前已经跑过针对自己改动的检查了**（第 2 层 `dsh-pre-push-checks` 的证据匹配）。所以 hook 不需要替 agent 做这件事，只需要拦住 agent 可能漏掉的那类**廉价缺陷**。

---

## 1. 三个执行者，各管一段

| 执行者 | 时机 | 原则 | 实际内容 |
|---|---|---|---|
| **Git hook**（`lefthook.yml`） | `pre-commit` / `pre-merge-commit` / `pre-push` | **窄而快** | 6 + 2 + 1 个 job |
| **CI**（`.github/workflows/`） | push / PR | **全而慢** | 15 个 workflow，`ci.yml` 里 7 个阻塞 job |
| **人/agent 手动** | 按需 | 证据匹配 | `pnpm run check:all`（`check-all` 聚合） |

第三项有个精确的定位声明（fast-local-git-hooks note 原文）：

> The opt-in `check:all` package script selects the `check-all` scheduler inventory in `scripts/run-gates.ts` independently of the hooks; **it is a contributor command, not an agent instruction.**

> **「这是给贡献者的命令，不是给 agent 的指令」**——这个区分很讲究。人可能想「跑一遍全部，图个安心」；agent **不该**这么做，因为它有更精确的证据匹配规则可用。**同一个命令对人和对 AI 有不同的适用性，并且被明确写下来。**

---

## 2. `lefthook.yml` 逐 job 拆解（全文只有 50 行）

### `pre-commit`：6 个 job

```yaml
pre-commit:
  jobs:
    - name: translation pairing (staged records)
      glob: '*.i18n.yaml'
      exclude: ['.agents/notes/archived/**']
      run: tsx scripts/verify-translation-pairing.ts --cached {staged_files}

    - name: archived agent notes
      glob: '.agents/notes/archived/**'
      run: tsx scripts/verify-archived-agent-notes.ts

    - name: lint (staged)
      glob: '*.{ts,tsx,mts,cts,mjs}'
      exclude: ['vendor/*/src/**']
      run: tsx scripts/run-oxlint.ts --config .oxlintrc.staged.json --fix --no-error-on-unmatched-pattern {staged_files}
      stage_fixed: true

    - name: third-party notices (staged)
      glob: '{package.json,*/package.json,…,pnpm-lock.yaml,vendor/README.md,…}'
      run: tsx scripts/gen-third-party-notices.ts && git add THIRD_PARTY_NOTICES.md

    - name: whitespace (staged)
      run: git diff --cached --check

    - name: vendor manifest guard
      run: scripts/check-vendor-manifest.sh
```

### 三条设计原则，逐条从配置里读出来

#### ① glob 按改动文件精确触发

```
改了 *.i18n.yaml        → 才查翻译配对
碰了 archived/**        → 才查归档冻结
改了 *.ts/*.tsx/…       → 才 lint（且排除 vendor/*/src）
改了 package.json/lock  → 才重新生成 notices
```

**没碰归档就不跑归档检查。** 这是 hook 能保持快的第一个原因——大多数 commit 只会触发其中一两个 job。

而且 `{staged_files}` 只把**暂存的文件**传给检查器，不是整个仓库。翻译配对甚至用了 `--cached` 参数专门读暂存内容。

#### ② 能自动修的就修掉，不要拒绝

两个 job 体现了这一点，而且方式不同：

**lint 用 `--fix` + `stage_fixed: true`**：修完自动重新暂存，commit 继续。

**notices 直接重新生成**，而且这个决策有一段很长的注释解释为什么：

```yaml
# Regenerate rather than reject: a dependency edit that forgot the notices
# would otherwise fail the test lane long after the commit. The glob matches
# every input the generator reads, including the generator itself and the
# build-time pin source. Deleting a manifest cannot trigger this job —
# lefthook only inspects files present on disk — so that one case still
# falls through to the freshness assertion in the test lane.
```

三层信息：

1. **为什么生成而不是拒绝**：忘了更新 notices 的依赖改动，否则会在很久之后的 test lane 才失败——**失败离原因太远**。
2. **glob 覆盖生成器读的每一个输入**，**包括生成器自己**。（改了生成器逻辑也要重新生成，容易漏。）
3. **主动承认一个覆盖不到的情况**：`删除 manifest 无法触发这个 job——lefthook 只检查磁盘上存在的文件`，所以那种情况仍然会落到 test lane 的新鲜度断言上。

> **第 3 点是这份配置里最值得学的一句。** 它没有假装这个 hook 是完备的，而是**明确写出漏洞在哪、以及漏掉之后由谁兜底**。
>
> 这就是分层防御的正确记录方式：**每一层都声明自己不管什么，以及那部分归谁。** 对比常见做法——写个 hook 就以为万无一失，直到某天有人删了个 manifest，然后在 CI 里看到一个莫名其妙的失败。

#### ③ pre-commit 只做快事，重活推到 pre-push

```yaml
pre-push:
  jobs:
    - name: typecheck
      run: pnpm run typecheck
```

**pre-push 只有一个 job。** 而且 note 里精确说明了它为什么是这一个：

> Pre-push adds **only** the Host contract build required by repository typecheck.

（`pnpm run typecheck` = `build:lib:host && typecheck:contracts-ready`，即先生成 Host Typert 契约，再跑 Client 增量 typecheck。）

### `pre-commit` 明确不做什么——这是一份「否定清单」

note 原文：

> **Pre-commit does not run type analysis, tests, snapshots, documentation checks, builds, hygiene, or the gate scheduler.**

七项被明确排除。**把「不做什么」写下来，价值等同于写「做什么」**——因为下一个人（或 agent）看到 pre-commit 里没有测试时，会本能地想「是不是漏了？我加上吧」。有这句话在，他就知道那是刻意的。

### `pre-merge-commit`：为什么只有 2 个

```yaml
pre-merge-commit:
  jobs:
    - name: translation pairing (staged records)   # 同 pre-commit
    - name: archived agent notes                   # 同 pre-commit
```

只保留了**两个「合并本身可能破坏的东西」**：翻译配对哈希和归档冻结内容。

> 为什么是这两个：merge 会自动合并两侧的改动，而**这两项的正确性依赖于「内容与记录的哈希严格一致」**。一次自动合并完全可能产出一个哈希不匹配的中间状态，且没有任何人主动编辑过那个文件。lint、typecheck 这些在 merge commit 上跑意义不大（内容来自已经检查过的两侧），但哈希一致性是 merge **特有的**风险。
>
> **这体现了这一层的思考粒度：不是「把 pre-commit 的东西复制到 pre-merge-commit」，而是问「merge 这个动作特有的风险是什么」。**

---

## 3. 那句戳中所有人的话：太慢的 hook 会被绕过

fast-local-git-hooks 的 Consequences 段有一句关于「怎么保证 hook 快」的选择，很反常识：

> **Hook latency is observed in development and PR evidence rather than enforced by a timing test** whose result would depend on host load and cache state.

**没有「hook 必须在 N 秒内完成」的自动化测试**——因为那种测试的结果取决于机器负载和缓存状态，会变成一个不稳定的门禁。**改用在开发和 PR 证据里观察。**

> 这是个诚实的取舍：这一层的核心指标（延迟）**恰好是不适合做成门禁的那类东西**。承认「这条要靠人盯」比造一个会随机失败的计时测试要好。**不是所有约束都该自动化——这个判断本身也是工程能力。**

而 hook 快不快的后果，note 也写清了：

> Cold commits take the project-free staged Oxlint fix-and-validate critical path and warm pushes take the prepared incremental typecheck critical path.

**冷 commit 的关键路径是「无项目的 staged Oxlint」**（`--config .oxlintrc.staged.json`，注意是专门的 staged 配置，不加载完整项目），**热 push 的关键路径是增量 typecheck**。两条路径都被刻意压到最短。

---

## 4. CI 的职责：全量矩阵

`.github/workflows/` 下 **15 个 workflow**：

| workflow | 干什么 |
|---|---|
| `ci.yml` | **主门禁矩阵**（下面详讲） |
| `e2e.yml` | 真实 DeepSeek API 的 e2e |
| `e2b-e2e.yml` | E2B 沙箱 e2e |
| `pi-ai-provider-e2e.yml` | Azure OpenAI / Anthropic e2e |
| `issue-policy.yml` / `issue-lifecycle.yml` | **Issue/PR 规范强制**（见「前置」章节） |
| `landlock-run.yml` / `-release.yml` | 原生 landlock 沙箱 |
| `build-exe-for-python-sdk.yml` | Python SDK 单文件可执行 |
| `python-release.yml` / `release.yml` / `release-vendor.yml` | 发布 |
| `docs-pages.yml` | 文档站点部署 |
| `sandbox.yml` | 沙箱 |
| `expected-filenames.yml` | 预期文件名 |

### `ci.yml` 的 job 拓扑

**7 个阻塞 job**（`all-checks-passed.needs` 里列的）：

```
node-24                  静态门禁          → check:ci:static
node-24-coverage         全量覆盖率        → check:ci:coverage
node-24-consumers        快照 + 构建产物   → check:ci:consumers
node-compat              node 22.19 / 26 兼容矩阵
python-sdk               python 3.10 keyless 全套
python-runtime           发布形态的 Linux x64 Python 运行时
windows                  Windows Node 在 Wine 下跑两个阻塞面
```

**注意这 7 个 job 调用的正是第 4 层的聚合模式**——`check:ci:static` = `run-gates.ts ci-static`。**第 4 层定义「什么和什么一起跑」，第 5 层定义「在哪台机器上跑」。分层干净。**

### 三个「企业级 job」为什么要拆开

```yaml
# Three enterprise jobs isolate coverage, static analysis, and the
# build-backed consumer tail. The consumer job owns the only Linux build so
# all three jobs enter runner allocation independently.
```

**拆成三个的目的是「独立进入 runner 分配」**——三个 job 可以同时排队、同时开始，而不是串成一条。而且明确了「consumer job 拥有唯一的 Linux build」，避免三个 job 各建一遍。

每个 job 的并发参数是**按机器规格调过的**：

```yaml
node-24:            DSH_GATE_CONCURRENCY: '8'
node-24-coverage:   DSH_COVERAGE_MAX_WORKERS: 6（或 failover 时 8）, DSH_GATE_CONCURRENCY: '3'
node-24-consumers:  DSH_GATE_CONCURRENCY: '8', DSH_SNAPSHOT_MAX_CONCURRENCY: 32（failover 12）
windows-native:     DSH_COVERAGE_MAX_WORKERS: '2', DSH_GATE_CONCURRENCY: '2'
node-compat:        gate_concurrency: '1'
```

而且每个数字都有注释解释来源，例如：

```yaml
# The hosted 16-core runner uses six coverage workers. The failover pool
# shares one 64-core VM across six always-on runner instances, so each
# instance may use eight while keeping the worst case at 8 × 6 = 48
# workers; process-bound suites remain isolated in forks.
```

> **`8 × 6 = 48` 这个算术被写进注释里**——因为这是个容易被后人「优化」掉的数字。没有这句话，有人看到 64 核只用 8 个 worker 会觉得浪费，然后改大，然后 6 个实例一起把 VM 打爆。

---

## 5. 这一层最精妙的设计：`all-checks-passed`

这个 job 只做一件事：**给分支保护提供一个稳定的必需检查名**。但它的实现有三处非显然的正确性设计。

```yaml
all-checks-passed:
  name: all checks passed
  needs: [node-24, node-24-coverage, node-24-consumers, node-compat, python-sdk, python-runtime, windows]
  if: always() && github.event_name == 'pull_request'
  steps:
    - name: Fail if any needed job did not succeed
      if: contains(needs.*.result, 'failure') || contains(needs.*.result, 'cancelled') || contains(needs.*.result, 'skipped')
      run: |
        echo "::error::Needed job results: ${{ join(needs.*.result, ', ') }}"
        exit 1
```

### 设计一：为什么需要这个 job

```yaml
# Single stable required check for branch protection: require "all checks
# passed" instead of enumerating matrix legs whose names change as lanes and
# node versions evolve.
```

**分支保护里列举矩阵腿的名字是个陷阱**：`node 22.19`、`node 26` 这些名字会随版本演进而变，而分支保护的配置在 GitHub 设置里（不在仓库里）。**改了 workflow 忘了改分支保护设置 → 那条检查不再被要求 → 静默失去保护。**

用一个永不改名的汇总 job，把「哪些是必需的」这个信息**移回仓库内的 `needs` 列表**——从而纳入 code review。

### 设计二：`if: always()` 是承重的

```yaml
# `if: always()` is load-bearing: without it a failed dependency
# would SKIP this job, and GitHub counts a skipped required check as passing
# — so this job always runs and fails on any non-success result, including
# 'cancelled' and 'skipped'.
```

**这是全文最关键的一句。** 拆开讲：

```
默认行为：依赖失败 → 这个 job 被 SKIP
GitHub 行为：被 skip 的必需检查算作「通过」
合起来 = 上游全挂 → 汇总 job 被跳过 → 分支保护认为一切正常 → 可以合并
```

**这是一个会让整个 CI 保护静默失效的漏洞**，而修复只是加 `if: always()`。

而且它的失败条件把三种非成功状态都列了：

```
'failure' || 'cancelled' || 'skipped'
```

> **`skipped` 必须算失败**——这和第 4 层 `run-gates.ts` 里「`skipped` 与 `failed` 同等对待」是**同一条原则在两个层次上的实现**。
>
> 「跳过被误判为通过」是 CI 系统里最经典的静默失效，因为它不产生任何红色信号。这套体系在**调度器层**和**CI 汇总层**各堵了一次。

### 设计三：明确声明哪个 job 故意不在 needs 里

```yaml
# The required Wine job is listed as `windows`; `windows-native` is
# deliberately absent so its independent result never delays or changes this
# verdict. (`needs` cannot reach across workflow files; e2e.yml stays its own check.)
```

`windows-native`（真实 Windows 内核，最长 120 分钟）**故意不在 needs 里**——它保留自己独立的结论，但不阻塞 PR。

对应地，`windows` job（Wine 下跑 Windows Node）承担阻塞责任，注释解释了这个看起来很怪的选择：

```yaml
# The required pull-request Windows signal: the two blocking win32 surfaces
# (workspace build, production site) execute with real, checksum-verified
# Windows Node under Wine on standard hosted Linux. The independent
# windows-native job below keeps the complete native-kernel inventory —
# including the observational portability gates this lane does not run —
# on real Windows.
```

> **「阻塞的用快的（Wine，15 分钟超时），观察的用真的（原生 Windows，120 分钟）」**——这是「阻塞 vs 观察」的分类被贯彻到平台级别。真实内核的信号有价值，但让它阻塞每一个 PR 的代价太大，于是**拆成两个 job，一个进 needs 一个不进，并把这个决定写在注释里**。

---

## 6. 几个值得单独讲的 CI 设计

### ① `concurrency` 的取消策略：为什么 push 不取消

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name != 'push' }}
```

配套注释近 20 行，核心逻辑：

> Cancel a superseded run on every event **EXCEPT push**. A push run carries the two self-hosted standby drills, which take longer than the interval between master merges, so **cancelling supersedes a drill before it reaches a verdict and the lane yields no readiness evidence.**

**master 上的 push 跑的是「自托管备用池演练」，耗时比两次 merge 的间隔还长。** 如果按常规取消旧运行，演练永远得不出结论，那条 lane 就永远不产出可用性证据。

而且注释解释了为什么用「否定 push」而不是「等于 pull_request」：

> Negated rather than `== 'pull_request'` so **workflow_dispatch keeps cancelling**: a re-dispatched runner benchmark holds up to 12 larger runners for 15 minutes and shares this group with the drills on master.

还主动声明了这个保证的**边界**：

> The guarantee is narrow — evaluated against the newly triggered run, so **a dispatch on master still cancels a mid-flight drill**, and a newer pending push displaces an older one.

> **一个三行的 YAML 配置，配了 20 行注释，讲清了：为什么这样、为什么不用更直观的写法、以及这个保证在哪里不成立。** 这是这个仓库注释风格的极致体现——**注释不描述代码在做什么，只解释代码为什么不是别的样子**。

### ② Failover：切换开关是「仓库变量」，不需要合 PR

三个 Linux 阻塞 job 的 `runs-on` 都是一个表达式：

```yaml
runs-on: >-
  ${{ vars.DSH_CI_FAILOVER_LINUX == 'selfhosted'
      && github.event.pull_request.user.login != 'dependabot[bot]'
      && fromJSON('["self-hosted", "linux", "x64", "vm-backup"]')
      || 'dsh-ubuntu-24-04-16core' }}
```

设计要点：

```
不设变量        → 用托管的企业池（正常）
设成 selfhosted → 三个 job 全部改到自建 vm-backup 池
切换动作        → 改一个仓库变量 + 重跑失败的 job，就这样
                  （writer 可管理的仓库状态，不是 PR 可编辑的，不需要合并）
Dependabot PR   → 排除在自建池外
Windows 有独立开关 DSH_CI_FAILOVER_WINDOWS
```

> **「切换开关不在代码里，所以不需要走 PR 流程」**——托管 runner 挂了的时候，你最不需要的就是「先提个 PR 改 workflow，但 CI 挂了所以 PR 合不进去」这种死锁。**把开关放在仓库变量里，是为了让它在 CI 已经不可用时仍然可操作。**

### ③ 热备演练：可用性被持续证明，不是假设

```yaml
serial-linux-selfhosted:
  if: github.event_name == 'push' && github.ref == 'refs/heads/master'
  name: serial / linux (self-hosted standby)
  runs-on: [self-hosted, linux, x64, vm-backup]
```

> Hot-standby drill for the in-house self-hosted pool: **every master move re-runs the complete unsharded aggregate** on the persistent 64-core VM, **continuously proving that environment can take over a required lane** if the hosted pools degrade.

**每次 master 移动，都在自建 VM 上跑一遍完整的不分片聚合。**

> **这是「灾备被持续验证」而不是「灾备被写在文档里」。** 绝大多数团队的 failover 方案是一份从未演练过的文档，真出事时才发现自建池的 Node 版本不对、Playwright 没装、store 是空的。这里每次 master 合并都演练一次，**所以切换开关的那一刻，你知道它能用**。
>
> 注意 `windows-native` 那个 job 的注释里还提到，Playwright 安装步骤本身也是演练的一部分：
> > this step also **proves that browser provisioning remains usable for failover**.

而且演练用 push 触发是刻意的：

```yaml
# Push-triggered, so this lane always executes the base branch's own workflow definition.
```

**push 触发保证跑的是基线分支自己的 workflow 定义**（而不是 PR 分支可能被改过的定义）——这是一条安全边界。同一段注释还说清了 failover 时的真正边界不是触发方式而是仓库成员制：

> (Under failover, pull_request jobs do reach these runners with the PR merge ref's workflow — the boundary there is **repository membership: private, forking disabled, Dependabot excluded**.)

### ④ 缓存的生产者/消费者分工

```yaml
# Pull requests consume the default-branch cache but do not put cache
# compression and upload on the paid latency-critical path.
- uses: actions/cache/restore@v4      # ← 注意是 restore，不是 cache
```

**PR 只 restore 不 save**，master 上的 job 负责 save。理由是「不把缓存压缩和上传放在付费的延迟关键路径上」。

而 `serial-linux` 里有一段注释解释了为什么不用 `setup-node` 内置的 `cache: pnpm`：

```yaml
# The store cache stays a hand-rolled actions/cache step rather than
# setup-node's `cache: pnpm`: the enterprise pull-request jobs above
# restore exactly this key and path, and setup-node's built-in cache
# uses its own key format — converting this producer would silently
# starve their documented restore-only optimization.
```

> **「转换这个生产者会静默饿死那些 job 的 restore-only 优化」**——又是一个「看起来该简化，实际不能动」的地方，被注释钉住了。**如果没有这句话，某个 agent 一定会把它「优化」成内置缓存，然后所有 PR 的缓存命中率悄悄归零，而没有任何测试会失败。**

### ⑤ 用注释挡住「这个缓存为什么跳过」

failover 时跳过缓存恢复，理由是实测数字：

```yaml
# Skipped under failover: the self-hosted VM's persistent pnpm store
# already serves warm installs, while restoring the hosted archive
# would spend ~52 s pulling ~180 MB into that populated store.
```

**「~52 秒拉 ~180 MB 到一个已经填满的 store 里」**——具体数字，不是「因为比较慢」。

### ⑥ 安全：为什么 e2e 不用 `pull_request_target`

`e2e.yml` 的注释：

> keeps an untrusted fork from exfiltrating the key. `pull_request_target` runs in the BASE repo's context **WITH secrets** while still able to check out untrusted fork code — **a textbook key-leak vector** especially once this repo is public.

**`pull_request_target` 是教科书级的密钥泄漏向量**——它带着 base 仓库的 secrets 运行，却能 checkout 不受信任的 fork 代码。被明确拒绝，理由记录在 Agent Note 里。

### ⑦ CI 不许污染生产遥测

```yaml
env:
  # CI runs must never report to the production telemetry endpoint baked
  # into apps/cli/cordis.yml (AppCLIEntry disables the row when set).
  DSH_TELEMETRY_DISABLED: '1'
```

---

## 7. 一个被明确排除的选项：hook 里跑全套

fast-local-git-hooks 的 Consequences 段坦白了这个边界的代价：

> **Local publication no longer proves the exhaustive repository matrix.** Agents must select relevant behavioral evidence, reviewers must evaluate whether that selection matches the diff, and CI supplies the comprehensive signal once per pushed revision.

**三方责任被重新分配**：

| 角色 | 新责任 |
|---|---|
| Agent | **选择相关的行为证据**（第 2 层的证据匹配） |
| Reviewer | **评估那个选择是否匹配 diff** ← 这是新增的 review 职责 |
| CI | 每个推送版本提供一次全面信号 |

> **注意中间那条**：「reviewer 必须评估 agent 选的证据是否匹配改动」——这成了第 6 层 review 清单里的一项。**hook 变快的代价，是 review 多了一项职责。**
>
> 这就是这套体系诚实的地方：它不宣称「快 hook 是纯收益」，而是明确指出**代价被转移到了哪一层，以及那一层要为此做什么**。

而这份 note 还有一个 `## Supersedes` 小节，示范了「部分取代」的标准写法：

> This decision supersedes **the local-hook portion** of [Parallel pre-push gates] and **the hook/CI symmetry** in [Mechanical quality gates over prose guidelines]. **Their CI scheduler, package-gate, and mechanical-enforcement decisions remain in force.**

**取代哪部分、哪部分仍然有效，都说清了。** 对应地，被取代的 quality-gates note 头部也有一行反向声明（见第 3 层详解）。**双向互链，两侧都改。**

---

## 8. 这一层如何和其他层咬合

| 关系 | 内容 |
|---|---|
| **← 第 4 层** | 直接调用 13 个聚合模式：`check:ci:static`、`check:ci:coverage`、`check:ci:consumers`、`doc-sync`、`check-all`……**第 4 层定义「什么一起跑」，第 5 层定义「在哪跑」** |
| **← 第 2 层** | `dsh-pre-push-checks` 的证据匹配是 hook 能保持窄的**前提**——agent 已经跑过针对性检查 |
| **← 第 3 层** | pre-commit / pre-merge-commit 都有 `archived agent notes` job，守归档冻结 |
| **← 第 7 层** | `check-vendor-manifest.sh` 是 `adding-a-vendored-package.md` 第 3 步教的内容的机器强制 |
| **→ 第 6 层** | 把「评估 agent 选的证据是否匹配 diff」这项职责交给了 review |

---

## 9. 落到自己项目：六步

1. **先写下 hook 的「否定清单」**：明确 pre-commit **不**跑测试、不跑构建、不跑全量 lint。**先定边界，再填内容**，否则 hook 会单调增长直到被人用 `--no-verify` 绕过。
2. **glob 精确触发**：只改文档就不该跑代码检查。这是 hook 保持快的最大来源。
3. **能自动修的就修**（`--fix` + 重新暂存 / 直接重新生成），不要拒绝。**拒绝的成本是一次人工往返，修复的成本是零。**
4. **给分支保护造一个永不改名的汇总检查**，并且**一定要加 `if: always()`**，失败条件包含 `skipped` 和 `cancelled`。这一条能挡住 CI 保护静默失效这个最严重的漏洞。
5. **区分「阻塞」和「观察」**：慢而有价值的信号（真实平台、长尾兼容）单独成 job，**不进 needs**，并把这个决定写进注释。
6. **不要为「hook 必须多快」写计时测试**。它依赖机器负载，会变成不稳定门禁。**改用在 PR 证据里观察——不是所有约束都该自动化。**

> **最容易被忽略的一条**：hook 变快之后，「验证充分性」的判断责任转移到了 review。如果你只抄「快 hook」而不接手这项 review 职责，结果就是**又快又没保障**。**这一层的收益和第 6 层的负担是绑在一起的。**

---

## 10. 一页总结

| 问题 | 答案 |
|---|---|
| 核心边界 | **本地窄而快，CI 全而慢**——`Keep these local checkpoints fast; CI owns the full repository-wide gate matrix` |
| 为什么本地不多跑 | 三条：延迟每次发布、放大本地 flake、**CI 马上要重跑一遍所以信息量为零** |
| 前提条件 | agent 已按第 2 层的证据匹配跑过针对性检查 |
| pre-commit | **6 个 job**，glob 精确触发，能修的自动修 |
| pre-commit 不做什么 | **明确排除 7 项**：类型分析、测试、快照、文档检查、构建、hygiene、门禁调度器 |
| pre-push | **只有 1 个**：`pnpm run typecheck` |
| pre-merge-commit | 只有 2 个——**merge 特有的风险**（翻译配对哈希、归档冻结） |
| CI 规模 | 15 个 workflow；`ci.yml` 里 **7 个阻塞 job** |
| 三个企业级 job 为何拆分 | 为了**独立进入 runner 分配**；consumer job 拥有唯一的 Linux build |
| 最精妙的设计 | `all-checks-passed`：**`if: always()` 是承重的**——否则依赖失败会让它被 skip，而 GitHub 把 skip 的必需检查算作通过 |
| 阻塞 vs 观察 | 阻塞用 Wine Windows（15 min 超时），观察用原生 Windows（120 min，**故意不在 needs 里**） |
| Failover 设计 | 开关是**仓库变量**不是代码——CI 已挂时仍可操作，不需要合 PR |
| 灾备可信度 | **每次 master push 都演练**自建池的完整聚合——可用性被持续证明，不是文档假设 |
| 缓存分工 | PR 只 restore，master 负责 save；**不用 `setup-node` 内置缓存**（key 格式不同会静默饿死 PR 的优化） |
| 安全拒绝 | `pull_request_target` 被拒——**教科书级密钥泄漏向量** |
| 不自动化的那一条 | **hook 延迟不做计时测试**，因为它依赖机器负载；改用 PR 证据观察 |
| 这一层的代价 | 本地不再证明全量矩阵 → **「评估 agent 选的证据是否匹配 diff」成为 review 的新职责** |
