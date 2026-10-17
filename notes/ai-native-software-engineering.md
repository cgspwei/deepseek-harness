# AI-Native Software Engineering 方法论

> 视角：不把 DeepSeek Harness 当「Agent runtime 项目」看，而是当「一个团队深度使用 AI Coding 后，如何重新设计软件工程流程」的 case study 来拆。
>
> 核心问题：**当 Coding Agent 成为主要生产力、开发速度提高一个数量级时，工程质量如何不崩掉？**

---

## 一、核心公式

```
AI Native Development =
      Context Engineering
    + Executable Constraints
    + Automated Verification
    + Traceability
    + Human Architecture Control
```

**贯穿因子：`one-home-per-fact`（每个事实只有一个家）**

这个因子是乘法，不是加法。它决定前面五项是「相互咬合」还是「各自漂移」：

- 没有它，Context 会把同一约定在三处写、三处漂移
- 没有它，Constraints 校验的文档和代码会各说各话
- 没有它，Traceability 的记录会和新代码脱节

---

## 二、六层拆解（范式转变 + 迁移要点）

### 1. Context Engineering：从 prompt-level 到 repository-level

**范式转变**：不再优化「prompt 写得好不好」，而是优化「仓库能不能被 Agent 读懂」。

传统 AI Coding：`我 → 给 prompt → Agent 写代码 → 我看`。规模一大必然 architecture drift，因为 Agent 每次都是局部最优。

AI-Native 的做法：`Repository（AGENTS.md + 约定 + Notes + skills + tests + CI）→ 自己告诉 Agent 怎么干`。

> **不是人每次告诉 AI 怎么干，而是 Repo 自己告诉 AI 怎么干。**

**迁移要点**：投入从「写 prompt」转向「把约定、架构、决策固化进仓库文件」。

---

### 2. Executable Constraints：规范从约束人 → 约束 Agent

**范式转变**：工程规范以前主要约束人，以后必须能够约束 Agent。

核心思想：**不要相信 Agent 会遵守规范，让机器验证规范。**

```
代码修改 → CI(unit/integration/typecheck/lint/coverage/snapshot/invariant)
   ↓ 失败
Agent 自己修 → 再次运行 → 直到通过
```

**判断一条规范值不值得写的唯一标准**：它的 **machine-checkable 程度**。

| 规则 | 价值 |
|---|---|
| "代码要优雅" | ≈ 0（Agent 无法执行） |
| dependency boundary / API schema / type system / test / coverage / lint / cycle detection / architecture check | 高 |

**迁移要点**：团队规范优先挑「能机械校验」的来写；写不进门禁的软规范，价值会打对折。

---

### 3. Traceability：Agent Notes 是「AI Coding 的 semantic commit」

**范式转变**：Git 存 `what changed`，Agent Note 存 `why + how reasoned + what contract changed + how verified`。

Agent 写得太快（一天 50-200 commits），人类靠 Git diff 根本理解不了系统变化，必须产生更高层的 artifact：

```
代码 diff → Agent Note（Problem / Decision / Design / Invariant / Implementation / Verification）
```

这形成 **Software Engineering Memory**：

```
历史设计决策 → Coding Agent retrieval → 下一次修改
```

**迁移要点**：决策留痕不是为了「防争论」（那是副产品），而是为了「在高速变更下，人类仍能理解系统为什么变成现在这样，Agent 也能检索历史决策」。

---

### 4. Human Architecture Control：人的 leverage 从「写代码」→「定义系统」

**范式转变**：

```
传统：Human 设计 + 写代码 + debug + 写测试 + review
AI-Native：
  Human  define problem / architecture / invariant / acceptance criteria / review decisions / resolve ambiguity
  Agent  explore repo / implementation / refactor / unit tests / docs / lint+typecheck / iteration
```

人的 leverage 从「写代码」转向「**定义一个 Agent 不容易把项目搞坏的系统**」。

**迁移要点**：人的核心产出物变了——不再是代码行，而是「清晰的问题定义 + 架构边界 + 不变量 + 验收标准」。

---

### 5. Agent-Legible Repository：仓库对 Agent 可读

**范式转变**：以前强调 `code readable by humans`，现在要增加 `repository legible to agents`。

Agent 进入仓库后，应该能快速回答：

1. 系统是什么？
2. 模块在哪里？
3. dependency 是什么？
4. 哪些地方不能动？
5. 改这个模块要跑哪些 test？
6. API contract 是什么？
7. 类似功能以前怎么实现？
8. architecture decision 在哪里？
9. 怎么证明修改正确？

如果 Agent 需要「grep 30 分钟 + 猜架构 + 看 100 个文件」，这个 repo 就不是 Agent-friendly 的。

**迁移要点**：文档分层的价值标准，从「人读得懂」升级为「Agent 能快速定位到正确信息」。

---

### 6. Repo 本身就是一个 Harness

Agent Harness 和 AI Coding Repo 的一一对应：

| Agent Harness | AI Coding Repo |
|---|---|
| System Prompt | `AGENTS.md` |
| Skill | repo skills / workflows |
| Memory | Agent Notes / git history |
| Tools | shell / git / tests |
| Environment | repository |
| Guardrail | lint / types / CI |
| Evaluator | tests / benchmark |
| Trace | commits / Agent Notes |
| World Model | architecture docs + code |

> **DeepSeek Harness 不只是 Harness 的实现，它自己的开发过程也在实践 Harness 思想。**

---

## 三、完整链路

```
                 AI Coding System
                     Human
                       │ requirement / design
                       ▼
                   AGENTS.md
                       │
            ┌──────────┴──────────┐
            ▼                     ▼
        Coding Agent           Skills
            │
            ▼
        Repository → implementation
            │
            ▼
 ┌───────────────────────────────┐
 │ tests / lint / types / CI     │
 │ architecture constraints      │
 └───────────────────────────────┘
            │
            ▼
        Agent Note
            │
            ▼
        Git history
            │
            └──────────────┐
                           ▼
                 Future Agent Context
```

---

## 四、判断「深度 AI Coding」的四条判据

未来判断一个工程团队 AI Coding 能力，不看「有多少人装了 Claude Code」，而看四件事：

1. **代码库是不是 Agent-readable？**
2. **工程约束是不是 machine-verifiable？**
3. **Agent 能不能自己完成 implement → test → diagnose → fix 的闭环？**
4. **在一天几十上百次 AI 修改之后，人类是否仍能理解系统为什么变成现在这样？**

四条都满足，才算深度使用 AI Coding。否则，大多数所谓 AI Coding 只是「用 Claude Code 替代了键盘」。

---

## 五、一句话总结

> DeepSeek Harness 最值得借鉴的，不是「用了 Codex 哪个命令」，而是它在探索：**当 Coding Agent 成为主要生产力以后，Software Engineering 本身应该怎么重新设计。**

这个方向，应该被当作一个 **AI-Native Software Engineering case study** 来研究，而不只是学习它的 Agent 架构。
