# DeepSeek Harness 开发方式剖析

> 核心结论：这是一个极度 **spec 驱动、文档优先、规范被 CI 门禁强制执行** 的工程。

---

## 一、文档分层体系（one home per fact）

项目把知识严格分层，**每个事实只有一个家**，其余地方只放链接。

| 层级 | 职责 | 关键约束 |
|---|---|---|
| 根 `AGENTS.md` | 常驻指令（每个 agent 会话都加载），1-3 行一条 | ≤1600 词 |
| `docs/architecture.md` | 有序地图：组合、核心包、循环、seam、扩展点 | ≤1800 词 |
| `docs/subsystems/` | 每子系统一页：类型定义 + 语义 + 生成的 Cordis API | — |
| **Agent Notes**（`.agents/notes/`） | **「为什么」+「放弃了什么」** 的决策记录 | 非平凡变更必须附带 |
| `docs/postmortem/` | 事故复盘（唯一允许讲故事的地方） | — |
| `docs/cookbook/` | 分步 how-to，带编号验证步骤 | — |
| 包 README | 每包契约 | 不含 JSDoc 复述 |
| `docs/development.md` | 贡献者搭建 + 工作流 | — |

**核心原则**：文档不是写完代码补的，而是**和代码在同一个 PR 里、被 CI 门禁强制校验**。

---

## 二、Agent Notes 机制（最核心的设计）

每个**非平凡变更**都**必须在同一个 PR 里附带一份 Agent Note**，记录：

```
## Problem                动机（不依赖解决方案也能成立）
## Decision               用现在时描述已交付的现实
## Alternatives considered  每个被否决方案 + 原因（必需，防反复争论）
## Consequences           权衡的代价和收益
```

**生命周期状态机**：

```
proposed/ → implemented/ → archived/（永久冻结）
          ↘ rejected/
```

**文件名编码**：`{生命周期}/{类别}/{日期-标题}.md`

**类别（封闭集合，门禁校验）**：
- `feature` — 面向用户/模型的新能力
- `bug-fix` — 修缺陷或补 postmortem 缺口
- `simplification` — 不增加能力的前提下删代码
- `architecture` — 交付源码的结构性决策
- `process` — 代码周边的工具/策略/工作流
- `testing` — 测试基础设施与策略

**关键规则**：
- `Alternatives considered` 是**必需的**——记录决策但不记录被否决的方案 = 邀请反复争论
- `implemented/` 记录用现在时，与实际交付保持同步
- `archived/` 永久冻结，绝不编辑
- 交叉引用用相对 Markdown 链接，不用纯文字/编号

---

## 三、Spec 驱动的证据：机械化门禁（gates）

规范**不是靠人自觉，而是被编成可执行脚本**：

| 门禁 | 作用 |
|---|---|
| `verify-agent-note-format` | 校验 Agent Note 格式 |
| `verify-type-equiv` | 文档里的类型声明与源码逐字一致（`ts type-equiv` 围栏） |
| `verify-md-links` | 交叉引用必须用相对链接，拒绝死链 |
| `verify-doc-budgets` | 文档字数预算上限 |
| `verify-cordis-config` | 配置里的插件必须在依赖清单中 |
| `verify-export-jsdoc` | 每个导出都要有 JSDoc |
| `verify-archived-agent-notes` | 归档记录冻结完整性 |

**形成闭环**：文档即契约，且契约被测试。

---

## 四、分层测试策略

```
单元测试(test)
  → 覆盖率门禁(100% 按文件，packages/*/*/src)
  → 真实 API e2e（带密钥，缺密钥自动跳过）
  → 快照测试（无密钥，ACP/headless 回放）
  → Web 浏览器快照
```

**核心哲学**：
- **优先真实实现而非 mock**：只 mock 昂贵边界（LLM、网络、时钟），下游一切真实
- **验证外部世界，而非自我报告**：e2e 断言重新运行命令或读文件，不做关键词探测
- **行覆盖率是必要条件，不是充分条件**：证明行被执行过，不证明功能正确
- **测试真实入口路径**：测构建后的 `lib/bin.js`，不测 tsx 源码（暴露 tsx 掩盖的失败）

---

## 五、双语文档约定

每个 `.md` 都有 `.zh.md` 对侧文件，通过「配对」机制保证同步：
- `dsh-translate-docs` 工作流（仅用户显式调用）
- 配对合并驱动自动处理 `.i18n.yaml` 冲突
- 术语表统一词汇翻译

---

## 六、给开发者的启示

1. **文档和代码是同一个交付物**：改代码不改文档，CI 会红
2. **决策必须留痕**：`Alternatives considered` 是强制项，防止团队反复争论同一个问题
3. **规范要机械化**：能写进门禁的规则，就不要靠 code review 口头提醒
4. **测试要分层且各有边界**：单元测行为、快照测输出、e2e 测真实世界、覆盖率当必要条件
5. **文档有预算**：字数上限逼你把细节放到正确的位置（one home per fact）

---

## 相关文档索引

- 根指令：`AGENTS.md` / `CLAUDE.md`
- 文档标准：`docs/AGENTS.md`
- Agent Notes 规范：`.agents/notes/README.zh.md`
- 测试策略：`docs/testing.zh.md`
- 开发指南：`docs/development.zh.md`
- 技能集：`.agents/skills/`（每个技能一个 SKILL.md）
