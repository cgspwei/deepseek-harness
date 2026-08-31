# DeepSeek Harness 学习路线图

这是一条从「理解框架」到「动手扩展」再到「深入源码」的递进路径。整个知识体系以 **Cordis 框架**为核心，往上层层叠加。

---

## 阶段一：理解 Cordis 框架（地基）

> 这是所有内容的前提。dsh 的每一部分都是插件，挂载到共享 `ctx` 上。

1. **读概念入门** → `docs/cordis-primer.zh.md`
   - 掌握 5 个核心概念：插件、上下文（`ctx`）、`inject` 依赖、类型化事件、可逆副作用
   - 重点理解 4 种事件分发模式（`emit` / `waterfall` / `parallel` / `serial`）

2. **动手做 7 章教程** → `docs/cordis-tutorial/index.zh.md`
   - 按顺序做：`01` → `07`，每一章都是可运行示例
   - 准备：`pnpm install`，然后 `mkdir -p tmp/cordis-tutorial`
   - 第 7 章「进入 harness」是关键，让你第一次把插件接入真实服务

3. **查 API 参考**（按需查阅）→ `docs/cordis-api/`
   - `context.zh.md`（上下文）、`service.zh.md`（服务）、`events.zh.md`（事件）、`registry.zh.md`、`fiber.zh.md`

---

## 阶段二：掌握总体架构（地图）

4. **读架构总览** → `docs/architecture.zh.md`
   - 重点吃透三块：
     - **Profile 与组合包**：插件树如何叠加（`dsh.profile` → bundle → patch）
     - **轮次流程**：`turn/start` → `agent/pre-step` → `step` → `turn/end` 的完整时序
     - **能力 seam**：Service Definition / Provider / Consumer 三角色
   - 用 `dsh --profile web --dump-config` 看真实配置树

5. **读术语表** → `docs/glossary.zh.md`
   - 校准领域词汇：seam、scope、turn、step、Round、goal、Ralph 等，避免后面读文档时概念混淆

6. **读事件生产/消费矩阵** → `docs/event-producer-consumer.zh.md`
   - 这是理解「谁派发、谁监听」的权威地图，选对事件域是做任何改动的第一步

---

## 阶段三：深入核心子系统（按依赖顺序）

> 先读 `docs/subsystems/README.zh.md` 获得全局索引，再按下面的顺序精读。

7. **核心循环（必读）**
   - `subsystems/core.zh.md` — agent loop、`AgentHandle`、投递/取消/拦截约定（最大篇幅，分次读）
   - `subsystems/session.zh.md` — `SessionEventMap`、`deriveMessages()`、轮次/步骤
   - `subsystems/llm-streaming.zh.md` — `Message`/`ContentBlock`、`StreamChunk`、适配器约定

8. **关键 seam（理解可替换能力）**
   - `subsystems/tools.zh.md` — 工具定义与执行流水线
   - `subsystems/system-prompt.zh.md` — 提示词组装
   - `subsystems/scope.zh.md` — 按 agent 划分的作用域注册

9. **按需扩展阅读**（根据你的方向选读）
   - 文件/进程：`filesystem`、`shell`、`subprocess`、`sandbox`、`terminal`
   - 会话持久化：`persistence`、`session-query`、`session-projection`
   - 高级能力：`subagent`、`compaction`、`workflow`、`skills`、`goal`、`schedule`

---

## 阶段四：动手扩展（从读到写）

10. **读扩展手册** → `docs/cookbook/extension-cookbook.zh.md`
    - 功能 → 机制映射表（钩子、goal、subagent、MCP、UI、遥测各自对应哪个扩展点）

11. **按指南写第一个东西**（选一个切入点）
    - 加一个工具：`cookbook/adding-a-tool.zh.md` + `user/develop/basic/tool.md`
    - 加一个包：`cookbook/adding-a-package.zh.md`
    - 加 LLM 适配器：`cookbook/adding-an-llm-adapter.zh.md`
    - 加 Chat 节点 / 设置卡片：对应 `adding-a-conversation-node` / `adding-a-settings-card`

12. **读能力图** → `docs/capability-seams.zh.md`
    - 完整理解每个 seam 的三个角色如何协同

---

## 阶段五：深入源码与工程实践

13. **读开发指南** → `docs/development.zh.md`
    - 前置条件、TS 项目布局（Host/Client 双 aggregate）、构建顺序、日常命令、CI 门禁
    - 了解 `pnpm run typecheck` / `build` / `check:all` 等命令

14. **探索源码包** → `packages/`
    - 从 `packages/core/agent-loop`、`packages/core/session`、`packages/core/tools` 入手
    - 参考 `examples/*/cordis.yml` 看真实组装

15. **进阶阅读**
    - `docs/agent-lifecycle.zh.md`（时序图）、`docs/tool-execution-pipeline.zh.md`（工具流水线）
    - `docs/postmortem/`（事故复盘，理解边界情况）

---

## 建议的学习顺序小结

```
Cordis 概念 → Cordis 7 章教程 → 架构总览 → 术语表 → 事件矩阵
        → 核心子系统(core/session/llm) → seam(tools/system-prompt/scope)
        → 扩展手册 → 动手写一个工具/包 → 开发指南 → 源码探索
```

**最关键的三个里程碑**：

1. 完成 Cordis 教程第 7 章（真正接入 harness）
2. 读懂 `architecture.zh.md` 的轮次流程（理解运行时）
3. 亲手注册一个工具（打通「注册 → 事件 → 执行」闭环）

---

## 学习进度

- [ ] 阶段一：理解 Cordis 框架
- [ ] 阶段二：掌握总体架构
- [ ] 阶段三：深入核心子系统
- [ ] 阶段四：动手扩展
- [ ] 阶段五：深入源码与工程实践
