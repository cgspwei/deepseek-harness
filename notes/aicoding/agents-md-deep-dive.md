# AGENTS.md 深度解读：Repository 级 System Prompt 的实现

> 本文是对 DeepSeek Harness 根目录 `AGENTS.md` 的逐节解读，重点在**设计意图**而非内容复述。
> 附带两个延伸专题：Pre-release stance 的生产系统适配、shim 概念澄清。

---

## 一、文件定位：它是「Repository 级 System Prompt」

这个文件是整个 DeepSeek Harness 方法论的入口文档，用 AI-Native 软件工程的语言说：

> **它是「不是人每次告诉 AI 怎么干，而是 Repo 自己告诉 AI 怎么干」这句话的实体。**

开头三句话完成三件事：

1. **一句话定义系统本质**（"everything is a plugin"）——AI 不需要读完全部代码就知道核心心智模型
2. **风险前置门**：改 `packages/` 前必须先读架构文档——把「最危险的区域」和「必读的上下文」绑定
3. **指路**：文档规范在哪

注意：没有欢迎语、没有项目背景故事、没有贡献指南。**每一个字节都在为 AI 会话服务**。

---

## 二、结构总览：12 个区段各司其职

| 区段 | 作用 | 类型 |
|---|---|---|
| 开篇定位 | 系统本质 + 必读顺序 | 导航 |
| Pre-release stance | 临时策略（带过期条件） | 时效性规则 |
| Repository layout | 代码库地图 | 世界模型 |
| Commands | 所有可执行命令 | 工具箱 |
| Host sandbox failures | 环境故障排查规则 | 故障预案 |
| Run relevant checks | 什么时候跑什么检查 | 工作流 |
| Secrets / .env | 密钥边界 | 安全 |
| Conventions | 编码规则（核心，33 条） | 行为约束 |
| Defensive patterns | 指针：做危险工作前先读 | 条件性导航 |
| Type safety & docs | 类型+文档标准 | 质量标准 |
| Editing these instructions | 如何编辑本文件 | 元规则 |
| Vendoring policy | 依赖管理 | 边界政策 |

---

## 三、关键区段的设计意图

### 1. Repository layout：给 Agent 的「世界模型」

47 行目录树，每个目录一行职责描述。关键设计：**这是地图，不是百科**。AI 拿着这张图准确定位「我要改的东西在哪」，然后去对应子目录读局部规则——这是「分层规则」的第一层实体。

### 2. Commands：命令即工具，注释即契约

每条命令的注释不是废话，而是**行为预期**：
- `test:coverage` 标注「CI coverage gate」——这是门禁级命令
- `test:e2e` 标注「self-skip without key」——没密钥不会挂，别误判
- `demo:cordis` 标注「the agent modifies its own runtime」——这个 demo 是干嘛的

「知道有哪些命令」和「知道命令的副作用/前提」是两个层次，这里两个都给了。

### 3. Host sandbox failures：预防 AI 的经典误诊

Agent 的经典失败模式：命令失败 → 开始「诊断」→ 怀疑代码 bug → 花半小时改代码 → 其实是沙箱挡了网络。

规则：**先排除环境因素（最小提权重试），再诊断代码**。后半句更重要——「never bypass genuine test failures」不许拿沙箱当借口跳过真正的测试失败。**这是一个双向防作弊条款**。

### 4. Run relevant checks：反「全量跑」的工作流规则

- **证据匹配**：什么改动配什么验证（行为 → 聚焦测试；模型/用户输出 → 快照；文档 → doc-sync）
- **禁止默认全量**：AI 有个坏习惯——不确定跑什么就全跑。CI 负责穷尽覆盖，本地负责相关验证

### 5. Conventions：33 条压缩的资深工程师判断（全文核心）

每条都是「用 1-3 行压缩的架构决策」，模式统一：**加粗关键短语 + 链接到 rationale 的家**。代表性条款：

**架构纪律类**：
- **Registrations are effects**：每个贡献走 `ctx.effect()`，`register()` 返回 disposer
- **Plugins, not loop changes**：新行为上扩展点；改 agent-loop 必须更新架构文档

**类型安全类**：
- **Switch on discriminant tags**：封闭联合以 `assertNever` 结尾
- **Opaque cross-boundary ids are branded**：跨边界 id 用 `Branded<B>`，不用裸 `string`

**失败语义类**：
- **Misconfiguration fails loud**：配置错在加载时炸，绝不静默跳过

**验证边界类**（特别精彩）：
- **Trust TypeScript at typed same-process boundaries**：精确定义哪里要运行时校验（wire/持久化边界）、哪里相信类型系统（同进程边界）——既防 AI 到处加防御代码（过度），又防该校验的边界裸奔（不足）

**关键观察**：这 33 条里几乎每条都**机器可检查**或**可被 review 快速裁定**。没有一条是「代码要优雅」这种空话。写不进这种形式的规则，就不配上这个文件——这就是「规范必须能约束 Agent」的实践。

### 6. Type safety and documentation：连文风都有规范

- 注释/文档陈述完整契约，不是推理过程
- 禁用比喻；禁用 `shape`/`boundary` 这类模糊词（AI 写文档爱用这些「听起来专业但指代不清」的词，这条直接堵住）
- 金句：**「把可机械检查的不变量接入会被执行的顶层门禁，并证明每个改动的验收路径会拒绝非法 case」**——光有门禁不够，门禁必须被证明真的会拦

### 7. Editing these instructions：文件自我管理的元规则

- **CLAUDE.md 是软链**——一份规则源，多个工具入口，工具无关
- **规则必须自包含**——被单独读到也要能理解
- **预算的逃生舱**——内容真的放不下可以调高字数上限，但要在 PR 里说明

---

## 四、作为 AI-facing 文档的四个设计模式（最有迁移价值）

**模式 1：入口极简，深处外链**
本文件 ≤1600 词预算，每条规则 1-3 行。「为什么」全部链接到 Agent Note / docs。规则在这，理由在家。

**模式 2：命令即工具，注释即契约**
Commands 区段每条注释都在告诉 Agent「这条命令的语义和前提」。

**模式 3：规则必须可裁定**
每条 Convention 都能被机器检查或被 review 一眼裁定对错。

**模式 4：反 AI 习惯的针对性条款**
- 防误诊：sandbox failure 先排查环境再查代码
- 防浪费：never default to full suite
- 防作弊：never bypass genuine test failures
- 防文档空话：禁比喻、禁模糊词

**这个文件本身就是一份「AI 弱点清单」的对策集。**

---

## 五、专题一：Pre-release stance 到底在干啥

### 5.1 它在回答一个问题：「AI 该多大胆？」

改任何系统都面临选择：
- **正确做法**：重命名、重打包、改结构——爆炸半径大
- **保守做法**：留 shim、加废弃层、写迁移代码——爆炸半径小

AI 的默认倾向是保守的（训练数据里全是「不要破坏兼容」），不明确告诉它，它本能地写兼容层。

### 5.2 给出的答案：现在选「正确做法」

理由藏在 `With no external consumers`（没有外部消费者）里——**爆炸半径理论上是零**。此时写 shim 是在制造未来要删的垃圾。

三个例子都在说「不要做迁移代码」：
- `Backends reject old on-disk formats`——直接拒绝旧格式，不是「兼容读取」
- `SQLite uses monotonic SCHEMA_VERSION`——版本号只增不减
- `SESSION_FORMAT_VERSION at 0 with no compatibility promise`——明确不承诺任何兼容

### 5.3 最精妙之处：规则自带销毁条件

`Remove this section at the first tagged release` 意味着：

1. 这是**限时规则**——一旦打了 tag，外部消费者可能出现，「大胆重构」的授权必须立刻失效
2. 过期条件**机械可判定**——「第一个 tag 是否存在」一个脚本就能查
3. 没有「删除义务」漂移的风险——大多数团队的临时策略死于没人记得该撤

### 5.4 通用模式

给 AI 的「胆量校准」必须显式写出来，且要绑定生效条件：

```
适用窗口 + 当下的偏好（激进/保守） + 现在为什么可以 + 何时失效
```

不写这个，AI 会永远按「最保守」的方式干活——因为保守永远不会被骂。

---

## 六、专题二：生产系统怎么适配这一节

**内容不能照搬**——那一节的前提是零外部消费者；线上系统的前提完全相反：真实流量、下游 RPC 依赖、历史数据兼容。把「拒绝旧格式」直接搬过去就是事故。

**但模式可以搬**。生产系统的病是反过来的：

> Pre-release 项目的风险是「AI 太保守，写一堆没必要的 shim」。
> 成熟生产系统的病是：**永远保守，shim 永远不删，「临时兼容」变成永久 legacy。**

生产系统没有「打 tag」这样的全局开关，于是兼容层只进不出——这就是祖传代码的成因。

### 6.1 项目自己保留的例外即是示范

即使在最激进的 pre-release 阶段，它也不是均匀激进的：
- 给 SQLite 留了单调版本号——因为**磁盘数据比代码活得久**
- `Trust TypeScript at typed same-process boundaries`：同进程边界信任类型（编译期炸，半径小）；wire/持久化边界加校验（运行时才炸，半径大）——**校验强度与爆炸半径成正比**
- `Misconfiguration fails loud`：这条在生产系统比 pre-release 更重要——防「静默降级无人发现」

### 6.2 生产版 Risk Stance 模板（可直接抄改）

```markdown
## Risk stance

按边界分级，不搞一刀切：

- **对外 API / RPC 契约**：版本化演进。新字段必须可选；删字段/改语义走
  deprecation 流程：公告 → 双写观察期 → 下线。禁止直接改。
- **存储 schema**：单调 version + 迁移脚本，禁止原地 ALTER 有数据的表结构。
- **模块间接口**：expand-contract（先加新路径 → 迁移调用方 → 删旧路径）。
- **模块内部实现 / 单一调用方的代码**：鼓励激进重构，不留 shim。
- **配置**：未知/缺失配置 fail loud，禁止静默回退默认值。
- **所有兼容层必须带销毁条件**：合入时写明「在 X 日期 / 迁移完 Y 流量后删除」，
  到期未删视为 bug。
```

最后一条是 `Remove this section at the first tagged release` 的生产镜像——**给每个兼容层一个自己的「删除本节」条款**。AI 和人一样，不会主动删兼容代码，除非删除条件是显式的、可被检查的。

### 6.3 一句话总结

那节教你的不是「大胆」，而是「**把风险姿态显式化、条件化、可撤销**」。pre-release 的姿态是「零半径所以全激进」，生产系统的姿态应该是「半径分级 + 兼容必须带过期日」——后者比前者更需要写下来，因为生产系统的保守倾向会无声地累积成 legacy。

---

## 七、概念澄清：什么是 Shim

**Shim（垫片/兼容层）**：一小段「薄薄的」兼容代码——不实现任何新功能，唯一作用是**让旧的调用方式继续能跑**，实际逻辑早已转交给新实现。

### 7.1 直观例子

函数改名后，有 50 处旧代码还在调 `getUser()`：

```typescript
// 新实现（真实逻辑都在这）
function fetchUser(id: string) { ... }

// shim：无逻辑，纯转发
function getUser(id: string) {
  return fetchUser(id);
}
```

激进做法：50 处调用点全改，删掉旧名字。

### 7.2 词源

Shim 原意是木工/机械里的**垫片**——一块薄铁片垫在缝隙里让尺寸不匹配的零件装得上。不承重、不参与结构，只负责填缝。软件里的 shim 一模一样：垫在新旧两套不匹配的接口之间，只负责「翻译转发」，不含业务逻辑。

### 7.3 生产系统里 shim 的真正问题

它填缝是填上了，但**没人回头拆它**。「临时兼容」活过三个月就成了永久 legacy——调用方永远迁移不完，因为旧接口一直活着，就没有迁移的紧迫性。

### 7.4 近亲术语辨析

| 术语 | 区别 |
|---|---|
| **shim** | 保持旧接口可用，纯转发到新实现 |
| **adapter** | 设计模式版本，两个既定接口之间的转换器（偏主动设计） |
| **facade** | 把复杂接口**简化**暴露，不是为兼容，是为易用 |
| **deprecation** | 不是代码，是「此接口即将删除」的公告流程 |

**生产系统的健康姿势不是「禁止 shim」，而是：shim 可以垫，但垫片必须是临时的（出生时就带死亡日期）。**

---

## 附：本文件与既有笔记的关系

| 笔记 | 关系 |
|---|---|
| `notes/development-methodology.md` | 讲整个文档体系「是什么」，本文是其中根规则文件的显微镜级解读 |
| `notes/ai-native-software-engineering.md` | 讲范式（公式与四层），本文是「第一层：让 AI 知道怎么做」的具体实现样本 |
| `notes/adoption-plan.md` | 落地方案；本文第五/六专题（stance + shim）可作为其「风险姿态」章节的补充素材 |
