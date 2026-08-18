# DeepSeek Harness 学习路线图

> 目标：掌握 deepseek-harness 的架构、用法与各个组件。
> 路线遵循仓库自身的分层：先看懂 **Cordis 地基**，再吃透 **核心 spine**，然后遍历 **能力 seam**，最后落到**扩展实战**。

## 贯穿全程的"三张地图"

这三样是任何时候迷路时的锚点：

1. `AGENTS.md` 的 Repository layout —— 包组速查。
2. `docs/architecture.md` —— 事件域 / plugin tree / turn flow。
3. `docs/subsystems/` —— 生成的服务 + 事件参考（想查某个服务/事件签名先来这里）。

---

## 阶段 0：跑起来，建立手感（0.5 天）

**目标**：把 Web UI 和 CLI 跑通，知道"它到底长什么样"。

1. `pnpm install`，配 `DEEPSEEK_API_KEY`。
2. `pnpm dsh web` 启动 Web UI，按 `docs/user/guide/index.zh.md` 走一遍（配模型 → 选工作区 → 跑一个任务）。
3. 试 CLI：`pnpm dsh --profile headless "Summarize this repository"`。
4. 熟悉 `AGENTS.md` 的 Commands 表——这是后面天天要用的。

**验收**：能跑一个真实任务，且知道 `pnpm vitest run <path>` 怎么跑单测。

---

## 阶段 1：概念地基（1–2 天）

**目标**：建立心智模型，后面所有组件都是这三块东西的实例。

| 顺序 | 读什么 | 抓什么 |
|---|---|---|
| 1 | `docs/cordis-primer.md` | Cordis 五要点 + 4 种事件派发模式（尤其 waterfall 必须 `next()`） |
| 2 | `docs/glossary.md` | 术语：Service Definition / Provider / Consumer、capability seam、profile/bundle/preset、session log |
| 3 | `docs/architecture.md` | 全局地图：event domains、plugin tree、turn flow |

**验收**：能口述"一次 turn 从 model request 到 tool 执行走过了哪些事件"，能说清"capability seam 为什么是三角色而非一个"。

---

## 阶段 2：Cordis 动手（2–3 天）

**目标**：把 Cordis 从"看懂了"变成"会写"。

- 完整过 `docs/cordis-tutorial/` 的 7 章，边读边写。
- 重点章节：生命周期与 effect、Typed Events、服务/依赖注入。

**验收**：能写一个带 `inject` 依赖、注册 effect、监听事件的插件，并说清 PENDING/LOADING/ACTIVE 状态。

---

## 阶段 3：核心 spine（3–5 天，最重的一段）

**目标**：吃透产品 API 主干——这是理解"各组件如何咬合"的关键。

按依赖顺序读 `packages/core/`：

1. **session**：session 生命周期 + **session log**（`deriveMessages()` 如何从 log 投影出 model context）。记住铁律 **model-visible ⟺ logged**。
2. **system-prompt / tools**：prompt 段和 tool schema 如何通过 `ctx.effect()` 注册。
3. **agent / agent-loop**：turn flow 的驱动者——一次 step = model request + 其 tool calls，穿过的 waterfall 链。
4. 事件系统本身：`docs/subsystems/` 里生成的 session / agent / capability 事件参考，理解 declaration merging。

**验收**：能画出一张"一次 tool call 从 user message → session log → deriveMessages → request → tool execute → log"的完整时序，并指出每个环节由哪个包负责。

---

## 阶段 4：能力 seam 全景（按需遍历，1–2 周）

**目标**：掌握每个能力组件，方法是**统一的三角色模板**（Definition / Provider / Consumer）。

每组都问同一组问题：*Service Definition 定义了哪些服务方法？Provider 有哪些实现？Consumer（通常是 tool）怎么把它暴露给模型？*

建议顺序（由简到繁）：

1. `packages/llm` — 服务定义 + DeepSeek provider，理解 model request 边界。
2. `packages/shell` / `packages/subprocess` — 命令执行 + 进程树。
3. `packages/fs` — 文件系统 + **策略（policy）**。
4. `packages/web` — search/fetch provider + tool consumer。
5. `packages/skill` / `packages/subagent` — 委派模式。
6. `packages/lsp` / `packages/compaction` / `packages/workflow` — 进阶能力。

**验收**：任选一个能力，能说出它的三角色分别落在哪个文件、服务方法签名是什么、tool 的 UI render intent 是什么。

---

## 阶段 5：扩展实战（持续，边做边学）

**目标**：从"读代码"到"改代码"，这是真正掌握的分水岭。

1. 读 `docs/cookbook/extension-cookbook.md` 和 `docs/cookbook/adding-a-tool.md`，跟着**加一个新 tool**（含 UI render intent）。
2. 读 `docs/defensive-patterns.md`（涉及生命周期/并发/子进程/teardown 前必读）。
3. 读 `docs/testing.md`——注意 `test:coverage` 才是 CI 门禁，且**非平凡的 model/产品可见行为改动要加 keyless snapshot**。
4. 尝试加一个 **capability seam**（三角色齐全），这是最高阶练习。

**验收**：能独立加一个带测试 + 快照的新工具，PR 符合 `AGENTS.md` 的约定（含 Agent Note）。

---

## 阶段 6：进阶主题（按兴趣选）

- **装配机制**：`packages/bundle` / `packages/preset` / profile 如何把插件 tree 组合起来（结合 `docs/cordis-primer.md` 的 loader 章节）。
- **类型图**：`packages/typert` 的类型图生成、loader、runtime registry。
- **外部接口**：`packages/sdk`（JSON-RPC 客户端）、`packages/acp`（ACP server）、`packages/hooks`（Claude Code/Codex 钩子桥）。
- **自修改**：`pnpm run demo:cordis`——agent 修改自己的 runtime，理解"可逆副作用 + 热重载"为什么是硬需求。

---

## 一个建议的里程碑节奏

- **第 1 周**：阶段 0–2，跑起来 + 懂 Cordis。
- **第 2–3 周**：阶段 3（核心 spine），这是最难也最值的一段。
- **第 4–6 周**：阶段 4–5，遍历能力 + 动手扩展。

---

## 下一步

- 先从**阶段 3（核心 spine）**做一份更细的、带具体文件和函数名的阅读清单，那是整条路线里信息密度最高、最容易迷路的部分。
