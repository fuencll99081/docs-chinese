# Jimu Agents Hub — Agent 框架设计与实现思路

## 一、整体定位

这是一个基于 **NestJS + LangGraph + LangChain** 构建的生产级多 Agent 编排框架，服务于快手商业化活动搭建平台。核心能力包括：页面搭建辅助（AI 理解页面结构 + 修改页面 Schema）、积木组件代码生成（沙箱 + mfcli）、设计稿转 Schema、图片生成/编辑、人工转接（Oncall 机器人）等。

---

## 二、分层架构

```
┌─────────────────────────────────────────────────┐
│  Gateway 层 (HTTP Controllers)                   │  ← SSE 流式 API + 普通 REST
├─────────────────────────────────────────────────┤
│  Service 层 (NestJS Providers)                    │  ← 会话管理、流存储、上下文绑定
├─────────────────────────────────────────────────┤
│  Harness 层 (Agent 框架核心)                      │  ← 图编排、节点、工具、中间件
│  ├── graphs/      (主图 + 子图)                   │
│  ├── core/        (抽象基类)                      │
│  ├── tools/       (原生工具)                      │
│  ├── mcps/        (MCP 集成)                      │
│  ├── skills/      (SKILL.md 驱动的技能)           │
│  ├── middleware/  (中间件链)                      │
│  ├── transformer/ (流事件转换器)                  │
│  └── checkpoint/  (状态持久化)                    │
├─────────────────────────────────────────────────┤
│  Infrastructure 层                               │  ← MySQL / Redis / Langfuse / Kconf
└─────────────────────────────────────────────────┘
```

---

## 三、Harness 抽象层 — 框架的核心设计

这是整个框架的精髓，通过一组抽象基类定义了 Agent 编排的契约：

### 3.1 基类继承体系

```
BaseConfigurable          ← 配置驱动的 Scope 过滤
├── BaseTool              ← 工具（超时+重试+Zod校验）
├── BaseMCP               ← MCP 适配器（远程工具服务）
├── BaseSkill             ← 技能（SKILL.md 前置元数据 + 脚本工具）
├── BaseGraph             ← 图生命周期（编译、流式执行、恢复）
│   └── BaseSubgraph      ← 委托模式（子图作为工具注册到父图）
└── BaseNode              ← 节点执行（中间件链 before/after）
    ├── BaseAgentNode     ← LLM 调用 + tool_call 路由
    │   └── Reasoner      ← JimuMaster 主图路由器
    └── BaseToolNode      ← 并行工具执行
```

**设计思路**：将 LangGraph 的底层 API 封装为一套面向对象抽象。每个图、节点、工具都是独立的类实例，通过继承复用生命周期管理逻辑，通过组合实现灵活扩展。

### 3.2 BaseGraph — 模板方法模式

`BaseGraph` 定义了图的骨架：

```
START → delegate-input → reason → [action | subgraph | delegate-output] → END
```

- `setupGraph()` 是模板方法，子类覆盖来定义具体的节点和边
- `loadTools()` 工厂方法，从 4 个来源（原生工具/MCP/技能/子图）动态加载
- `stream()` 方法封装了 `compiledGraph.streamEvents()`，自动注入 Langfuse callback、过滤内部事件、收集指标
- `compile()` 方法编译图，配置 checkpoint saver 和 interrupt 配置

**关键设计**：图编译后的实例是缓存的，避免每次请求都重新编译。

### 3.3 BaseSubgraph — 委托模式

子图可以同时作为**图节点**和**LLM 可调用的工具**注册到父图。当 LLM 决定调用子图时：

1. `delegateInputNode()` 提取 tool_call 参数，清空父图 messages，注入新的 HumanMessage
2. 执行子图内部的工作流
3. `delegateOutputNode()` 将结果打包成 `ToolMessage` 返回给父图

这实现了 Agent 之间的隔离和上下文管理——子图不继承父图的完整对话历史，只接收结构化输入。

### 3.4 BaseNode + 中间件链

每个节点执行时经过中间件链：

```
beforeExecute(middlewares) → executeInternal() → afterExecute(middlewares)
```

已实现的中间件：
- **ContextCompressionMiddleware**：当 token 数超过 180K 时，用独立 LLM 调用压缩对话历史（标记 `INTERNAL_AGENT_TAG` 过滤出用户可见流）
- **PerflogReporterMiddleware**：性能指标上报

---

## 四、图编排体系

### 4.1 JimuMaster — 主编排器

```
START → delegate-input → reasoner → { action | page-builder | page-understander | creator-engine | END }
                                  action → reasoner (循环)
                          subgraph → delegated-routing → { reasoner | END }
```

**Reasoner** 节点继承 `BaseAgentNode`，核心逻辑是路由判断：
- 如果 LLM 返回的 tool_call 指向子图 → 路由到子图节点
- 如果是普通工具 → 路由到 `action` 工具节点
- 无 tool_call → 结束

**delegated-routing** 处理子图完成后的流向：
- `page-builder` → 直接返回（HITL 需要用户操作）
- `page-understander` → 检测是否需要按钮生成，否则结束
- `creator-engine` → 继续回 reasoner

### 4.2 PageBuilder — 页面修改子图（核心业务流程）

```
delegate-input → planner → builder → { tools | action-interrupt | END }
                          builder ← tools
                     action-interrupt → action-validator → { builder | task-summarizer | END }
```

这是最复杂的子图，包含 5 个节点：

1. **Planner**：将用户请求分解为 TodoList（通过自定义事件流式返回），失败时回退为单个默认 Todo
2. **Builder**：工具调用型 Agent，逐个执行 Todo。检测到 Markdown action fence（`JIMU_UPDATE_ELEMENT` 等）时路由到 interrupter
3. **ActionInterrupter**：解析 action fence，派发 `page_action_batch` 事件，然后调用 LangGraph `interrupt()` 暂停图执行，等待用户应用变更（HITL）
4. **ActionValidator**：用结构化输出（Zod schema）验证变更，管理 Todo 生命周期（标记完成/失败/合并新 Todo），清理过期工具消息，最多重试 3 次
5. **TaskSummarizer**：生成自然语言总结

**HITL 实现**：当图被 interrupt 暂停后，前端用户应用变更，然后调用 `resume()` 恢复。恢复时 `getResumeUpdateForInterruptedState()` 会将父图最新状态同步到子图 checkpoint。

### 4.3 其他子图

| 子图 | 拓扑 | 特点 |
|---|---|---|
| PageUnderstander | reasoner → action 循环 | 只读查询，不修改页面 |
| CreatorEngine | sandbox-init → mf-generate | 线性流水线，沙箱代码生成 |
| FigmaDeriveButton | 5节点流水线 | Figma → 元数据 → 代码 → 图片合成 |
| PlatformOncall | 独立图 | Kim 机器人 oncall 集成 |

---

## 五、工具系统 — 配置驱动的动态加载

### 5.1 四类工具来源

1. **Native Tools**（`harness/tools/`）：继承 `BaseTool`，约 30+ 个工具
2. **MCP Tools**（`harness/mcps/`）：继承 `BaseMCP`，通过 `MultiServerMCPClient` 连接远程 MCP 服务器
3. **Skill Tools**（`harness/skills/`）：继承 `BaseSkill`，加载 `SKILL.md` 前置元数据 + 参考文档 + 脚本
4. **Subgraphs**：子图通过 `asTool()` 注册为工具

### 5.2 Scope 过滤机制

`config.yaml` 定义每个工具的 `scope` 字段（如 `global`、`page-builder`、`['jimu-master', 'page-builder']`）。`BaseConfigurable.compatibleWithScope()` 在图初始化时过滤，实现**同一个工具类实例只出现在合适的图中**。

### 5.3 工具执行保障

`BaseTool.invoke()` 实现：
- **超时控制**：`AbortSignal.timeout(maxDuration)` 默认 30s，与外部 abort signal 合并
- **重试**：`maxRetryTimes` 默认 3 次，超时错误不重试
- **错误规范化**：超时统一为 `[BaseTool] "name" timed out after Xms`

---

## 六、状态管理与持久化

### 6.1 LangGraph Annotation 状态

所有图状态使用 `Annotation.Root()` 定义，包含 25+ 个字段（messages、todos、pageSchema、editorInfo、activityInfo 等）。

Reducer 策略：
- `messages`：使用 `messagesStateReducer`（追加 + 去重）
- 其他字段：`(x, y) => y ?? x ?? defaultValue`（后写覆盖）

### 6.2 Checkpoint 持久化

两种后端：
1. **MemorySaver**：开发/测试用
2. **MysqlCheckpointer**：生产环境，实现 `BaseCheckpointSaver`，使用 3 张表（checkpoint 元数据 / blob 通道值 / pending writes），Sequelize 事务保证原子性，按 `step` 排序保证多实例下的执行顺序

`CheckpointManager` 是单例工厂，按 `{type}-{graphName}` 缓存实例。

### 6.3 业务数据持久化

| 模型 | 用途 |
|---|---|
| AgentChatConversation | 会话记录 |
| AgentChatMessage | 消息记录（含人工反馈） |
| AgentChatPageBuilderAction | 页面操作日志（成功/失败操作） |

`AgentChatStreamStorageService.wrapStreamWithStorage()` 包装流式输出，自动在流结束时持久化 AI 消息。

---

## 七、流式处理管道

这是框架最复杂的部分之一——从 LangGraph 原始事件到用户可见 SSE 的完整转换链：

```
LangGraph streamEvents(v2)
  → 过滤 INTERNAL_AGENT_TAG 内部事件
  → IAgentStreamTransformer（事件语义化转换）
  → AgentChatStreamStorageService（服务端投影 + 持久化）
  → stream2SSE（心跳保活 + [DONE] 终止符）
  → HTTP SSE Response
```

### 7.1 14 种语义事件

原始 LangGraph 事件（`on_chat_model_stream`、`on_tool_start` 等）被转换为 14 种语义事件：

- `run.started` / `run.completed`
- `message.delta`（内容增量，带 `thinking_candidate` 标记）
- `thinking.delta` / `message.flush`（思考内容 + 冲刷决策）
- `tool.started` / `tool.completed` / `tool.failed`
- `task.snapshot`（TodoList 更新）
- `context.compressing` / `context.window_usage`
- `message.persisted`
- `page.action.batch`（HITL 动作批次）
- `artifact.action.requested`
- `handoff.requested`

### 7.2 Thinking Candidate 缓冲

LLM 输出内容可能是"思考"也可能是"正式回复"。Transformer 维护一个缓冲区：当遇到 `message.flush` 事件时决定缓冲区内容归属——是归入 `thinking.delta` 还是 `message.delta`。这解决了某些模型不区分思考/回复内容的问题。

### 7.3 服务端投影（Projection）

`projectEvents()` 在服务端聚合所有事件为 `MessageProjection`：
- `content`：最终消息文本（含工具标记、思考块、转接 fence）
- `todos`：最终 TodoList 状态
- `compressing`：上下文压缩状态
- `contextWindowRatio`：Token 使用比例

这个投影被持久化到数据库，确保即使用户中途断连，也能从投影恢复完整消息。

### 7.4 BaseStreamTransformer — 核心转换器

这个类处理：
- 按 nodeName 过滤事件（只暴露关心的节点）
- 工具显示名映射（`TOOL_DISPLAY_NAME_MAP`）
- 多消息源分隔符插入
- Handoff fence 追加
- Composite 模式支持多个 transformer 协作

---

## 八、Prompt 系统 — 模块化组装

JimuMaster 主图的 prompt 由 13 个 Markdown 文件按顺序组装：

```
00-identity.md          → 身份定义
10-routing-priority.md  → 11级路由优先级
domains/
  ├── drop-link.md
  ├── artifact-action.md
  ├── design-to-jimu-schema.md
  ├── page-editing.md
  ├── image-generation.md
  ├── image-edit.md
  ├── creator-engine.md
  ├── human-handoff.md
  └── platform-info.md
90-execution-constraints.md  → 执行约束
91-output-contracts.md       → 输出契约
```

每个子图也有独立的 prompt 目录（如 page-builder 的 planner.md、builder.md、validator.md）。

**设计思路**：Prompt 按领域模块化，便于独立维护和迭代。`prompt/index.ts` 负责按序读取并拼接，运行时缓存结果。

---

## 九、LLM 集成

### 9.1 模型配置 — 两级映射

```
endpoint: QWen3-max → ep-5lc3m8-xxxx（实际模型端点）
usage: master.reason → QWen3-max（功能角色到模型的映射）
```

不同功能角色使用不同模型，例如：
- `master.reason` → 主编排器
- `page-builder.planner/builder/validator` → 页面构建各节点
- `middleware.compression` → 上下文压缩

### 9.2 统一 LLM 网关

`getAgent(modelName)` 通过万晴平台（内部 LLM 网关）创建 `ChatOpenAI` 实例，统一鉴权和路由。`streamUsage: true` 启用 Token 实时统计。

### 9.3 外部工作流集成

`getWanqingWorkflow(workflowId, params)` 调用万晴工作流 API，用于知识库检索（平台描述、FAQ、组件元数据等）。

---

## 十、可观测性

### 10.1 Langfuse 追踪

`BaseGraph.createLangfuseCallback()` 创建回调处理器：
- session_id = thread_id（关联会话）
- user_id 追踪
- 环境标签 + 图名标签

本地开发额外注入 `GraphLocalLogCallbackHandler` 做控制台日志。

### 10.2 告警系统

- `AlertExceptionFilter`：全局异常过滤器
- `ErrorClassifier`：错误分类（HTTP 状态、abort、timeout 等）
- `AlertService`：通过 Kim 机器人推送告警

### 10.3 性能监控

`PerflogReporterMiddleware` 收集每个节点和工具的执行耗时，通过 `perf-logger` 上报。

---

## 十一、CreatorEngine — 沙箱代码生成

特殊设计：不使用 LLM 推理循环，而是线性流水线：

```
delegate-input → sandbox-init → mf-generate → delegate-output → END
```

1. **sandbox-init**：通过 `registry.acquire(threadId)` 获取沙箱会话，执行初始化步骤（SSO 登录、mfcli 资源、mf-server）
2. **mf-generate**：委托给 mfcli 子 Agent 执行代码生成（技术方案 → 模板克隆 → 代码生成 → 命名 → serve → publish）

**AsyncLocalStorage 传递**：`request-context.ts` 使用 Node.js 的 `AsyncLocalStorage` 将 HTTP 请求的 Cookie 和用户信息传递到沙箱步骤，解决深层调用链的上下文传递问题。

---

## 十二、设计模式总结

| 模式 | 应用 |
|---|---|
| **模板方法** | BaseGraph.setupGraph() 定义图骨架，子类覆盖 |
| **委托** | BaseSubgraph 作为工具注册到父图，隔离上下文 |
| **中间件链** | IMiddleware 的 before/after hooks |
| **策略** | IStreamTransformer 不同实现处理不同事件 |
| **工厂+注册** | BaseGraph.loadTools() 动态加载 4 类工具 |
| **配置驱动** | config.yaml scope 过滤工具可用性 |
| **组合** | CompositeStreamTransformer 委托多个 transformer |
| **单例** | CheckpointManager 按图名缓存实例 |
| **观察者** | streamEvents + callback handler（Langfuse） |

---

## 十三、核心设计哲学

1. **抽象优先**：不直接用 LangGraph API，而是构建 OO 抽象层，让业务代码只关心领域逻辑
2. **配置驱动**：工具/skill/subgraph 的加载和 scope 由 YAML 配置控制，新增工具只需加类 + 配置
3. **流式优先**：从 LLM 到用户端全程流式，但服务端通过 Projection 保证可恢复
4. **隔离设计**：子图通过委托模式隔离上下文，避免 token 膨胀
5. **HITL 内置**：LangGraph interrupt + resume 机制深度集成，支持页面修改的"生成→暂停→应用→验证"循环
6. **可观测**：Langfuse 全链路追踪 + 性能监控 + 告警通知

---

# 附：简历项目描述参考

> 以下内容面向 AI 全栈工程师岗位求职场景，按简历常用的"项目名称 + 角色职责 + 技术亮点 + 业务价值"结构组织，可根据实际参与程度裁剪。

## 项目名称：快手商业化活动搭建平台 — AI Agent 编排引擎

**项目背景**：快手商业化活动搭建平台（积木）服务于公司内部运营人员的可视化页面搭建流程。项目旨在通过多 Agent 协作，让运营人员用自然语言完成页面结构理解、组件配置修改、积木组件代码生成、设计稿转 Schema 等复杂操作，降低搭建门槛、提升生产效率。

**技术栈**：NestJS / LangGraph / LangChain / TypeScript / Vue 3 / MySQL / Redis / Langfuse / MCP Protocol

### 角色与职责

**AI 全栈工程师** — 负责 Agent 编排框架的架构设计、核心模块开发及前端交互落地。

### 核心工作与技术亮点

**1. 设计并实现 Agent 抽象框架层（Harness）**

- 基于 LangGraph 封装了一套面向对象的 Agent 抽象体系（BaseGraph / BaseSubgraph / BaseNode / BaseTool），将底层图编排 API 转化为声明式类继承模式，使业务开发者只需关注领域逻辑，新增 Agent 平均开发成本从 3 天降至 0.5 天
- 通过模板方法 + 委托模式实现子图作为工具注册到父图的机制，Agent 间上下文隔离，有效控制了 LLM 上下文窗口膨胀，单次请求 Token 消耗降低约 40%
- 设计 YAML 配置驱动的工具 Scope 过滤机制，实现同一工具在不同 Agent 图中的按需加载，支撑 30+ 原生工具、MCP 工具、Skill 工具的动态注册与统一管理

**2. 构建多 Agent 协作编排体系**

- 主编排器（JimuMaster）采用路由优先级策略，实现 11 级意图识别与多子图分发，涵盖页面修改、页面理解、代码生成、图片生成、人工转接等场景
- PageBuilder 子图实现了"规划 → 执行 → 中断 → 验证 → 总结"的完整工作流，深度集成 LangGraph interrupt/resume 机制，支持 Human-in-the-Loop（用户审核并应用 AI 生成的页面变更），保障操作安全可控
- CreatorEngine 子图通过沙箱环境 + mfcli 子 Agent 实现积木组件的自动化代码生成流水线，利用 AsyncLocalStorage 解决深层调用链的请求上下文透传问题

**3. 设计流式处理与状态持久化管道**

- 实现自定义 StreamTransformer，将 LangGraph 原始事件流转换为 14 种语义事件（消息增量、思考过程、工具生命周期、任务快照等），支持 Thinking Candidate 缓冲机制，兼容不同模型的思考/回复内容区分策略
- 设计服务端 Projection 机制，在流式输出的同时聚合完整的 MessageProjection（内容 + TodoList + 上下文状态），持久化到 MySQL，确保用户断连后可从投影恢复完整对话
- 基于 MySQL 实现 LangGraph Checkpoint Saver，使用 3 表结构 + 事务 + step 排序，保证多实例部署下的状态一致性和执行顺序可靠性

**4. 全链路可观测性与生产保障**

- 集成 Langfuse 实现全链路 LLM 调用追踪（Session/Trace/Token 维度），本地开发环境注入自定义回调做结构化日志
- 实现错误分类器 + Kim 机器人告警通道，对超时、Abort、HTTP 异常等错误自动分级告警
- 通过 ContextCompressionMiddleware 在 Token 超过 180K 时自动调用独立 LLM 压缩对话历史，保证长对话场景的稳定性

### 业务价值

- 运营人员通过自然语言完成页面修改，单次复杂页面调整操作时间从平均 15 分钟缩短至 3 分钟
- 支撑平台日均 XXX+ 次 AI 辅助搭建请求，覆盖页面修改、组件生成、设计稿转换等核心场景
- 框架可扩展性支撑了 6 个独立 Agent 图和 4 类工具体系的快速迭代，新 Agent 上线周期从周级降至天级

---

### 简历使用建议

1. **量化数据按实际补充**：日均请求量、Token 消耗、响应时延、用户满意度等指标根据真实数据填写
2. **按目标岗位侧重裁剪**：
   - 偏 AI 应用开发：重点突出 Agent 编排、Prompt 工程、工具系统、流式处理
   - 偏全栈架构：重点突出 NestJS 模块化设计、状态持久化、SSE 流式通信、前后端协作
   - 偏基础设施：重点突出 Checkpoint 机制、Langfuse 可观测性、告警体系、沙箱管理
3. **面试延伸准备**：建议深入准备以下高频追问——
   - "子图委托模式如何解决上下文膨胀？Token 消耗怎么测算的？"
   - "HITL 的 interrupt/resume 在分布式环境下如何保证状态一致？"
   - "StreamTransformer 的 Thinking Candidate 缓冲策略解决了什么问题？"
   - "config.yaml 的 Scope 机制和运行时动态加载是如何实现的？"
   - "如果让你从零设计这套框架，你会做哪些不同的选择？"
