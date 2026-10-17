# DeepSeek Harness 研发流程总结

> 把之前拆解过的各个组件（AGENTS.md、Agent Notes、门禁、测试策略）串成一条**从「接需求」到「合并代码」的完整流水线**。
>
> 贯穿全流程的核心原则：**每一步都在回答「你怎么证明这步做对了」，而不是「这步做了什么」。**

---

## 一、整体流程（五阶段）

```
① 接需求/定决策
      ↓
② 实现（Agent 写代码 + 同步写文档）
      ↓
③ 本地验证（选最小证据集，不跑全量）
      ↓
④ 推送 + CI 全量门禁
      ↓
⑤ 人类 review（只查机器查不了的）
```

### 各阶段关键动作与产物

| 阶段 | 做什么 | 谁做 | 核心产物 |
|---|---|---|---|
| ① 决策 | 复杂改动先写 proposed Agent Note，接受评审 | 人/Agent | `proposed/*.md` |
| ② 实现 | 写代码 + 同 PR 更新文档 + 落地 implemented Note | Agent | 代码 + 文档 + `implemented/*.md` |
| ③ 本地验证 | 按改动面选「最小证据集」 | Agent | 聚焦测试/快照/doc-sync |
| ④ CI 门禁 | 全量覆盖 + 平台矩阵 + 真实 API | 机器 | 绿灯/红灯 |
| ⑤ Review | 语义审查 + 决策合理性 | 人 | review 意见 |

---

## 二、四个区别于传统流程的设计

### 1. 「证据匹配」原则贯穿全程

核心技能 `dsh-pre-push-checks` 教 Agent **按改动面选最小验证集，而不是默认全量跑**：

| 改动类型 | 本地验证 |
|---|---|
| 包/脚本行为 | 所属 vitest 文件 |
| 文档/Agent Note | `doc-sync` |
| 模型/CLI/终端可见输出 | 快照 |
| 包 manifest/导出/构建 | build + hygiene |
| 真实 provider | e2e |

**经济学逻辑**：AI 写代码快、机器验证也快，都便宜；本地只跑「能抓住这个改动回归」的最小集，全量留给 CI。

`never default to full suite` 是明确写进规则的反 AI 坏习惯条款——AI 不确定跑什么就全跑，会浪费大量时间。

### 2. 文档和代码在「同一个 diff」里诞生

硬约束：**非平凡改动必须在同一 PR 里更新文档 + Agent Note**。

结果：「写代码」和「写文档」不是两个阶段，而是**同一个动作**。文档漂移在物理上不可能——改了代码不改文档，CI 直接红（`verify-type-equiv`、`verify-doc-budgets`、`doc-sync`）。

### 3. 三层验证的职责边界极其清晰

| 层 | 查什么 | 谁跑 |
|---|---|---|
| Git 钩子（pre-commit/pre-push） | 极窄：lint 修复、空白、vendor 元数据、typecheck | 本地自动 |
| 本地证据集 | 本次 diff 的相关行为 | Agent 手动选 |
| CI 门禁 | 全量覆盖 + 平台矩阵 + 真实 API | 机器 |

Git 钩子**故意设计得很窄**（不跑测试、快照、文档检查），因为「钩子管窄、CI 管全、本地管相关」——对验证责任的精确切分，避免重复劳动。

### 4. Review 只查「语义」，机器已经查完「事实」

`dsh-code-review` 技能开头点破：

> "a short review with one substantiated blocker is better than a list of nits."

（一个言之有据的 blocker，好过一堆 nit。）

Review 的 blocking requirements 全是**机器查不了的东西**：
- 新 prose 的语义质量
- 文档是否匹配代码意图
- 核心类型文档是否一致
- 注册是否正确清理
- 不变量是否有语义
- 证据是否存在

而机器能查的（格式、死链、覆盖率、类型）一条都不在 review 清单里——因为它们已经在 CI 被拦了。

---

## 三、流程的灵魂：一个「自我校正的闭环」

```
踩坑 → 固化成规则（AGENTS.md/Agent Note）→ 规则被执行（门禁/技能）→ 防止再踩
```

`dsh-code-review` 里的关键句：

> "Treat disagreement with an Agent Note as a design discussion, not an automatic veto."

（把与 Agent Note 的分歧当作设计讨论，而不是自动否决。）

**含义**：Agent Note 是「当前权威」，但不是「不可推翻」。如果 review 发现某个历史决策过时了，就发起新的设计讨论、写新的 Agent Note。这保证流程不会僵化成「死守旧决策」，而是能自我演化。

---

## 四、人类在流程中的角色（对应 AI-Native 范式）

| 环节 | 人类 | Agent |
|---|---|---|
| 决策 | 定义问题、评审 proposed | 起草 proposed |
| 实现 | — | 写代码 + 文档 + Note |
| 本地验证 | — | 选证据集、跑、自修 |
| CI | 维护门禁脚本 | 被门禁检查 |
| Review | 语义审查、决策合理性、解决歧义 | 修 review 意见 |

人类的核心产出从「代码行」变成了「清晰的问题定义 + 架构边界 + 验收标准 + 决策评审」。

---

## 五、与其他笔记的关系

- `development-methodology.md`：拆解各组件（是什么）
- `ai-native-software-engineering.md`：范式层（公式与四层）
- 本文：把组件串成**流程**（怎么流转）

三者分别回答「有什么 → 为什么 → 怎么连起来」。
