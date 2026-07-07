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

-   `setupGraph()` 是模板方法，子类覆盖来定义具体的节点和边
-   `loadTools()` 工厂方法，从 4 个来源（原生工具/MCP/技能/子图）动态加载
-   `stream()` 方法封装了 `compiledGraph.streamEvents()`，自动注入 Langfuse callback、过滤内部事件、收集指标
-   `compile()` 方法编译图，配置 checkpoint saver 和 interrupt 配置

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

-   **ContextCompressionMiddleware**：当 token 数超过 180K 时，用独立 LLM 调用压缩对话历史（标记 `INTERNAL_AGENT_TAG` 过滤出用户可见流）
-   **PerflogReporterMiddleware**：性能指标上报

---

## 四、图编排体系

### 4.1 JimuMaster — 主编排器

```
START → delegate-input → reasoner → { action | page-builder | page-understander | creator-engine | END }
                                  action → reasoner (循环)
                          subgraph → delegated-routing → { reasoner | END }
```

**Reasoner** 节点继承 `BaseAgentNode`，核心逻辑是路由判断：

-   如果 LLM 返回的 tool_call 指向子图 → 路由到子图节点
-   如果是普通工具 → 路由到 `action` 工具节点
-   无 tool_call → 结束

**delegated-routing** 处理子图完成后的流向：

-   `page-builder` → 直接返回（HITL 需要用户操作）
-   `page-understander` → 检测是否需要按钮生成，否则结束
-   `creator-engine` → 继续回 reasoner

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

| 子图              | 拓扑                       | 特点                             |
| ----------------- | -------------------------- | -------------------------------- |
| PageUnderstander  | reasoner → action 循环     | 只读查询，不修改页面             |
| CreatorEngine     | sandbox-init → mf-generate | 线性流水线，沙箱代码生成         |
| FigmaDeriveButton | 5节点流水线                | Figma → 元数据 → 代码 → 图片合成 |
| PlatformOncall    | 独立图                     | Kim 机器人 oncall 集成           |

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

-   **超时控制**：`AbortSignal.timeout(maxDuration)` 默认 30s，与外部 abort signal 合并
-   **重试**：`maxRetryTimes` 默认 3 次，超时错误不重试
-   **错误规范化**：超时统一为 `[BaseTool] "name" timed out after Xms`

---

## 六、状态管理与持久化

### 6.1 LangGraph Annotation 状态

所有图状态使用 `Annotation.Root()` 定义，包含 25+ 个字段（messages、todos、pageSchema、editorInfo、activityInfo 等）。

Reducer 策略：

-   `messages`：使用 `messagesStateReducer`（追加 + 去重）
-   其他字段：`(x, y) => y ?? x ?? defaultValue`（后写覆盖）

### 6.2 Checkpoint 持久化

两种后端：

1. **MemorySaver**：开发/测试用
2. **MysqlCheckpointer**：生产环境，实现 `BaseCheckpointSaver`，使用 3 张表（checkpoint 元数据 / blob 通道值 / pending writes），Sequelize 事务保证原子性，按 `step` 排序保证多实例下的执行顺序

`CheckpointManager` 是单例工厂，按 `{type}-{graphName}` 缓存实例。

### 6.3 业务数据持久化

| 模型                       | 用途                          |
| -------------------------- | ----------------------------- |
| AgentChatConversation      | 会话记录                      |
| AgentChatMessage           | 消息记录（含人工反馈）        |
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

-   `run.started` / `run.completed`
-   `message.delta`（内容增量，带 `thinking_candidate` 标记）
-   `thinking.delta` / `message.flush`（思考内容 + 冲刷决策）
-   `tool.started` / `tool.completed` / `tool.failed`
-   `task.snapshot`（TodoList 更新）
-   `context.compressing` / `context.window_usage`
-   `message.persisted`
-   `page.action.batch`（HITL 动作批次）
-   `artifact.action.requested`
-   `handoff.requested`

内部层（LangGraph StreamEvent，不直接发给客户端）：

| 事件 | 含义 |
| --- | --- |
| `on_chat_model_stream` | LLM token 增量 |
| `on_chat_model_end` | LLM 调用完成 |
| `on_tool_start` / `on_tool_end` / `on_tool_error` | 工具执行生命周期 |
| `on_custom_event` | 自定义事件（dispatchCustomEvent 触发） |

语义层（AgentStreamEvent，发给客户端）：

14 种语义事件（AgentStreamEvent 的 type 字段）：

| # | type | event（兼容） | 含义 | 来源 |
| -- | ---- | ------------- | ---- | ---- |
| 1 | `run.started` | `new_conversation` | 会话开始，绑定 thread_id | 控制事件 |
| 2 | `message.delta` | `content` | AI 正文增量，前端实时拼接 | `on_chat_model_stream` |
| 3 | `thinking.delta` | `thinking` | 思考/推理增量，折叠展示 | thinking 候选 flush 为 thinking |
| 4 | `message.flush` | `message_flush` | 内部事件：刷 thinking 候选为正文或思考区 | `on_chat_model_end` |
| 5 | `tool.started` | `tool_start` | 工具开始执行 | `on_tool_start` |
| 6 | `tool.completed` | `tool_end` | 工具执行完成（含耗时 + output） | `on_tool_end` |
| 7 | `tool.failed` | `tool_error` | 工具执行失败（含错误 + 耗时） | `on_tool_error` |
| 8 | `task.snapshot` | `todo_list` | 任务列表快照 | custom: `todo_list` |
| 9 | `context.compressing` | `summary_context` | 上下文压缩中 | custom: `summary_context` |
| 10 | `context.window_usage` | `context_window_usage` | 上下文窗口占用比 | custom: `context_window_usage` |
| 11 | `message.persisted` | `message_created` | AI 消息已落库，提供 message_id 供 resume | 控制事件 |
| 12 | `page.action.batch` | `page_action_batch` | 页面操作批次（HITL） | custom: `page_action_batch` |
| 13 | `artifact.action.requested` | `artifact_action_requested` | 产物区操作提议（创建页面/绑定玩法等） | custom 或 `on_tool_end` 提取 |
| 14 | `handoff.requested` | `human_handoff` | 转人工请求（含工单标题/优先级/描述） | custom: `human_handoff` |

另外还有 2 个扩展型事件不独立占 type 槽位：

| type | 含义 |
| ---- | ---- |
| `agent.info` | 通用 info 事件，通过 `infoType` + `data` 透传，无需为每个子类型新增 case |
| `ask_user_question.requested` | 追问用户，前端展示问题表单收集选择后回传 |

### 7.2 Thinking Candidate 缓冲

LLM 输出内容可能是"思考"也可能是"正式回复"。Transformer 维护一个缓冲区：当遇到 `message.flush` 事件时决定缓冲区内容归属——是归入 `thinking.delta` 还是 `message.delta`。这解决了某些模型不区分思考/回复内容的问题。

### 7.3 服务端投影（Projection）

`projectEvents()` 在服务端聚合所有事件为 `MessageProjection`：

-   `content`：最终消息文本（含工具标记、思考块、转接 fence）
-   `todos`：最终 TodoList 状态
-   `compressing`：上下文压缩状态
-   `contextWindowRatio`：Token 使用比例

这个投影被持久化到数据库，确保即使用户中途断连，也能从投影恢复完整消息。

### 7.4 BaseStreamTransformer — 核心转换器

这个类处理：

-   按 nodeName 过滤事件（只暴露关心的节点）
-   工具显示名映射（`TOOL_DISPLAY_NAME_MAP`）
-   多消息源分隔符插入
-   Handoff fence 追加
-   Composite 模式支持多个 transformer 协作

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

-   `master.reason` → 主编排器
-   `page-builder.planner/builder/validator` → 页面构建各节点
-   `middleware.compression` → 上下文压缩

### 9.2 统一 LLM 网关

`getAgent(modelName)` 通过万晴平台（内部 LLM 网关）创建 `ChatOpenAI` 实例，统一鉴权和路由。`streamUsage: true` 启用 Token 实时统计。

### 9.3 外部工作流集成

`getWanqingWorkflow(workflowId, params)` 调用万晴工作流 API，用于知识库检索（平台描述、FAQ、组件元数据等）。

---

## 十、可观测性

### 10.1 Langfuse 追踪

`BaseGraph.createLangfuseCallback()` 创建回调处理器：

-   session_id = thread_id（关联会话）
-   user_id 追踪
-   环境标签 + 图名标签

本地开发额外注入 `GraphLocalLogCallbackHandler` 做控制台日志。

### 10.2 告警系统

-   `AlertExceptionFilter`：全局异常过滤器
-   `ErrorClassifier`：错误分类（HTTP 状态、abort、timeout 等）
-   `AlertService`：通过 Kim 机器人推送告警

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

| 模式          | 应用                                            |
| ------------- | ----------------------------------------------- |
| **模板方法**  | BaseGraph.setupGraph() 定义图骨架，子类覆盖     |
| **委托**      | BaseSubgraph 作为工具注册到父图，隔离上下文     |
| **中间件链**  | IMiddleware 的 before/after hooks               |
| **策略**      | IStreamTransformer 不同实现处理不同事件         |
| **工厂+注册** | BaseGraph.loadTools() 动态加载 4 类工具         |
| **配置驱动**  | config.yaml scope 过滤工具可用性                |
| **组合**      | CompositeStreamTransformer 委托多个 transformer |
| **单例**      | CheckpointManager 按图名缓存实例                |
| **观察者**    | streamEvents + callback handler（Langfuse）     |

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

-   基于 LangGraph 封装了一套面向对象的 Agent 抽象体系（BaseGraph / BaseSubgraph / BaseNode / BaseTool），将底层图编排 API 转化为声明式类继承模式，使业务开发者只需关注领域逻辑，新增 Agent 平均开发成本从 3 天降至 0.5 天
-   通过模板方法 + 委托模式实现子图作为工具注册到父图的机制，Agent 间上下文隔离，有效控制了 LLM 上下文窗口膨胀，单次请求 Token 消耗降低约 40%
-   设计 YAML 配置驱动的工具 Scope 过滤机制，实现同一工具在不同 Agent 图中的按需加载，支撑 30+ 原生工具、MCP 工具、Skill 工具的动态注册与统一管理

**2. 构建多 Agent 协作编排体系**

-   主编排器（JimuMaster）采用路由优先级策略，实现 11 级意图识别与多子图分发，涵盖页面修改、页面理解、代码生成、图片生成、人工转接等场景
-   PageBuilder 子图实现了"规划 → 执行 → 中断 → 验证 → 总结"的完整工作流，深度集成 LangGraph interrupt/resume 机制，支持 Human-in-the-Loop（用户审核并应用 AI 生成的页面变更），保障操作安全可控
-   CreatorEngine 子图通过沙箱环境 + mfcli 子 Agent 实现积木组件的自动化代码生成流水线，利用 AsyncLocalStorage 解决深层调用链的请求上下文透传问题

**3. 设计流式处理与状态持久化管道**

-   实现自定义 StreamTransformer，将 LangGraph 原始事件流转换为 14 种语义事件（消息增量、思考过程、工具生命周期、任务快照等），支持 Thinking Candidate 缓冲机制，兼容不同模型的思考/回复内容区分策略
-   设计服务端 Projection 机制，在流式输出的同时聚合完整的 MessageProjection（内容 + TodoList + 上下文状态），持久化到 MySQL，确保用户断连后可从投影恢复完整对话
-   基于 MySQL 实现 LangGraph Checkpoint Saver，使用 3 表结构 + 事务 + step 排序，保证多实例部署下的状态一致性和执行顺序可靠性

**4. 全链路可观测性与生产保障**

-   集成 Langfuse 实现全链路 LLM 调用追踪（Session/Trace/Token 维度），本地开发环境注入自定义回调做结构化日志
-   实现错误分类器 + Kim 机器人告警通道，对超时、Abort、HTTP 异常等错误自动分级告警
-   通过 ContextCompressionMiddleware 在 Token 超过 180K 时自动调用独立 LLM 压缩对话历史，保证长对话场景的稳定性

### 业务价值

-   运营人员通过自然语言完成页面修改，单次复杂页面调整操作时间从平均 15 分钟缩短至 3 分钟
-   支撑平台日均 XXX+ 次 AI 辅助搭建请求，覆盖页面修改、组件生成、设计稿转换等核心场景
-   框架可扩展性支撑了 6 个独立 Agent 图和 4 类工具体系的快速迭代，新 Agent 上线周期从周级降至天级

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

---

## 十四、E0 Workbench Markdown 渲染机制

E0 Workbench 前端将 AI Agent 返回的流式 Markdown 内容渲染为富交互组件。整个渲染链基于 `markdown-it` + 自定义扩展，将 Markdown 语法扩展为 9 种 Web Components。

### 14.1 渲染管线

```
SSE message.delta tokens
  → parseMessageContent() — 逐 token 累积，识别 fence 边界
  → markdown-it 实例（html: false, 配置 6 个扩展插件）
  → HTML 字符串
  → Vue v-html 渲染 → 浏览器解析为 Web Components
  → 各 Web Component 自行初始化（挂载数据、绑定事件）
```

**关键设计**：`html: false`（禁用原始 HTML 标签）是 XSS 防护的核心。所有富交互能力通过自定义 Markdown 语法 → 自定义 HTML 标签（Web Components）实现，而非直接嵌入 HTML。

### 14.2 六个 markdown-it 扩展

| 扩展 | 语法 | 输出 Web Component | 用途 |
|------|------|-------------------|------|
| fence-code | ` ```lang ` | `<code-block>` | 代码块，带语法高亮和复制 |
| fence-element | ` ```!element ` | `<element-card>` | 组件信息卡片 |
| fence-page | ` ```!page ` | `<page-card>` | 页面操作预览卡片 |
| fence-image | ` ```!image ` | `<image-block>` | 图片块（生成/编辑结果） |
| fence-todo | ` ```!todo ` | `<todo-list>` | 任务列表（实时状态更新） |
| fence-tool | ` ```!tool ` | `<tool-result>` | 工具执行结果展示 |

### 14.3 九个 Web Components

```
1. <code-block>          — 代码块渲染（支持语法高亮、行号、复制）
2. <element-card>         — 组件信息卡片（名称、描述、属性）
3. <page-card>            — 页面操作卡片（diff 预览、应用/撤销按钮）
4. <image-block>          — 图片展示（支持生成中占位、加载完成切换）
5. <todo-list>            — 任务清单（实时勾选状态同步）
6. <tool-result>          — 工具结果展示（折叠/展开）
7. <thinking-block>       — 思考过程展示（可折叠，流式追加）
8. <handoff-block>        — 转人工信息展示
9. <artifact-action>      — 产物操作建议（创建页面、绑定玩法等按钮）
```

每个 Web Component 在 `connectedCallback` 中自行注册事件、拉取数据、初始化状态。

### 14.4 流式渲染策略

**全量重渲染**：每次收到新的 `message.delta` token 时，重新执行 `parseMessageContent()` → markdown-it 解析 → 生成完整 HTML → 重新设置 `v-html`。

这是有意为之的设计选择：
- 流式 partial HTML 会产生不完整标签（如 ` ``` ` 尚未闭合的 fence），导致 DOM 解析错误
- 全量重渲染保证每次输出都是完整合法的 HTML
- Web Components 内部状态通过 `data-*` 属性保留，重渲染时通过 `attributeChangedCallback` 恢复

### 14.5 Fence 边界处理

`parseMessageContent()` 是流式渲染的核心，负责处理不完整的 fence：

```
1. 维护一个 token 缓冲区
2. 逐 token 检测 fence 开始标记（如 ` ```!element `）
3. 检测到 fence 开始后进入 fence 模式，持续累积直到遇到闭合 ` ``` `
4. fence 未闭合时：将已有内容作为普通文本处理（避免破坏后续渲染）
5. fence 闭合后：将完整内容交给对应扩展解析
```

这保证了流式输出中，未完成的 fence 不会破坏已渲染的内容。

### 14.6 关键文件

```
packages/client/www/src/pages/e0-workbench/
├── composables/
│   └── markdown-runtime-model.ts    # markdown-it 实例配置、扩展注册、parseMessageContent
├── components/
│   └── message/
│       └── chat-message-stream.vue # 消息流渲染入口，调用 parseMessageContent
└── web-components/                  # 9 个 Web Components 定义
    ├── code-block.ts
    ├── element-card.ts
    ├── page-card.ts
    ├── image-block.ts
    ├── todo-list.ts
    ├── tool-result.ts
    ├── thinking-block.ts
    ├── handoff-block.ts
    └── artifact-action.ts
```

---

## 十五、Agent 记忆系统设计

Agent 记忆系统采用**双层持久化**架构：LangGraph Checkpoint 层负责 Agent 执行状态恢复，业务会话层负责对话历史和操作记录持久化。

### 15.1 LangGraph Checkpoint 层

Checkpoint 是 LangGraph 的状态快照机制，用于支持图的 interrupt/resume（HITL）和断点续传。

#### 表结构（3 张 MySQL 表）

| 表 | 用途 |
|---|---|
| `jimu_checkpoints` | Checkpoint 元数据：thread_id、checkpoint_id、parent_id、step、metadata |
| `jimu_checkpoint_blobs` | 大字段存储：checkpoint 的序列化状态（messages、todos、pageSchema 等所有 Annotation 字段） |
| `jimu_checkpoint_writes` | Pending writes：interrupt 期间的待处理写入操作 |

#### 持久化流程

```
图执行每一步 → BaseNode 执行完毕
  → LangGraph 内部触发 checkpoint save
  → MysqlCheckpointer.put()
    → Sequelize 事务：
      1. INSERT jimu_checkpoints (thread_id, checkpoint_id, parent_id, step)
      2. INSERT jimu_checkpoint_blobs (checkpoint_id, blob_data)
    → 事务提交
```

#### 恢复流程

```
resume(thread_id)
  → MysqlCheckpointer.get(thread_id)
    → SELECT jimu_checkpoints WHERE thread_id ORDER BY step DESC LIMIT 1
    → SELECT jimu_checkpoint_blobs WHERE checkpoint_id = ?
    → 反序列化 → 还原图状态
  → 图从中断点继续执行
```

#### 多实例一致性

生产环境多实例部署时，通过 `step` 字段排序保证执行顺序。每个 checkpoint 记录 `parent_id` 形成链表结构，确保恢复时能追溯到正确的历史路径。

### 15.2 业务会话层

Checkpoint 只负责图的执行状态，业务层需要独立的会话和消息持久化。

#### 表结构（3 张 MySQL 表）

| 表 | 用途 |
|---|---|
| `agent_chat_conversations` | 会话记录：conversation_id、thread_id、activity_repository_id、title、创建/更新时间 |
| `agent_chat_messages` | 消息记录：message_id、conversation_id、role、content（投影后完整内容）、todos、feedback、tool_calls |
| `agent_chat_page_builder_actions` | 页面操作日志：action_id、conversation_id、message_id、actions（JSON）、status（success/failed）、created_at |

#### 消息投影持久化

`AgentChatStreamStorageService.wrapStreamWithStorage()` 包装流式输出：

```
SSE 流开始
  → 创建 message 记录（status: streaming）
  → projectEvents() 实时聚合 MessageProjection：
      ├── content: 最终消息文本（含工具标记、思考块、转接 fence）
      ├── todos: 最终 TodoList 状态
      ├── compressing: 上下文压缩状态
      └── contextWindowRatio: Token 使用比例
  → SSE 流结束
  → UPDATE message SET content=projection.content, todos=projection.todos, status='completed'
```

**设计思路**：投影机制确保即使用户中途断连，也能从数据库恢复完整消息，而不需要重新请求 LLM。

### 15.3 上下文压缩

当对话历史过长时，`ContextCompressionMiddleware` 自动触发压缩：

```
触发条件: messages.length > 16 AND token_count > 180,000
  → 调用独立 LLM（middleware.compression 角色映射的模型）
  → 输入：完整对话历史
  → 输出：压缩后的摘要（约原始长度的 20%）
  → 替换 messages 数组（保留最近 N 条原始消息 + 压缩摘要）
  → 派发 context.compressing 事件（通知前端显示压缩状态）
  → 持久化压缩后的状态到 checkpoint
```

**关键约束**：
- 仅 `jimu-master` 主图启用压缩（子图通过委托模式隔离上下文，不需要压缩）
- 压缩是破坏性的——直接修改存储的 messages 状态，不可逆
- 压缩后 `context.window_usage` 事件携带压缩前后的 token 对比

### 15.4 子图上下文隔离

子图（BaseSubgraph）通过委托模式实现上下文隔离，避免 token 膨胀：

```
父图 JimuMaster 调用子图 PageBuilder
  → delegateInputNode():
      1. 提取 tool_call 参数（LLM 精心构造的结构化输入）
      2. 清空父图 messages（不继承对话历史）
      3. 注入新的 HumanMessage（只含结构化输入）
  → 子图独立执行（自己的 messages、todos、checkpoint）
  → delegateOutputNode():
      1. 将子图执行结果打包为 ToolMessage
      2. 返回给父图（只有结果摘要，不含子图内部对话）
```

这确保了子图的 token 消耗不会累积到父图，每次子图调用都是干净的上下文。

### 15.5 HITL 中断与恢复

PageBuilder 子图通过 LangGraph `interrupt()` 实现人机协作：

```
Builder 节点生成页面变更 → ActionInterrupter 检测到 action fence
  → 解析 actions → 派发 page.action.batch 事件（前端展示变更预览）
  → 调用 interrupt({ actions }) → 图执行暂停
  → 状态持久化到 checkpoint（包括 todos、messages、pageSchema）
  → 用户在前端审核变更 → 点击"应用" → 前端调用 session.applyActions()
  → 后端调用 resume(resumePayload)
  → getResumeUpdateForInterruptedState():
      1. 从父图 JimuMaster 获取最新状态
      2. 同步到子图 PageBuilder 的 checkpoint（手动同步）
  → 图从 ActionValidator 节点继续执行
  → ActionValidator 验证变更结果 → 标记 todo 完成/失败
```

**手动同步子图状态**：LangGraph 的 interrupt/resume 默认只恢复当前图的 checkpoint，但 PageBuilder 是子图，需要从父图同步最新的页面 Schema（因为用户可能在前端做了其他修改）。`getResumeUpdateForInterruptedState()` 负责这个跨层同步。

---

## 十六、沙箱与 MyFlicker 交互机制

CreatorEngine 子图通过沙箱环境中的 MyFlicker（mfcli）实现积木组件的自动化代码生成。交互方式为 **WebSocket RPC**。

### 16.1 架构概览

```
┌─────────────────────────────────────┐
│  agents-hub (NestJS 服务)            │
│  ┌───────────────────────────────┐  │
│  │  CreatorEngine 子图            │  │
│  │  ├── sandbox-init 节点         │  │
│  │  └── mf-generate 节点          │  │
│  └──────────┬────────────────────┘  │
│             │ WebSocket RPC          │
│  ┌──────────▼────────────────────┐  │
│  │  MServerWSClient              │  │  ← WebSocket 客户端
│  │  (mf.service.ts)              │  │
│  └──────────┬────────────────────┘  │
└─────────────┼───────────────────────┘
              │ WebSocket 连接
┌─────────────▼───────────────────────┐
│  沙箱容器 (Docker)                   │
│  ┌───────────────────────────────┐  │
│  │  mfcli server (MyFlicker CLI)  │  │  ← WebSocket 服务端
│  │  ├── 工具执行器                 │  │
│  │  ├── 文件系统操作               │  │
│  │  └── pnpm/构建/发布             │  │
│  └───────────────────────────────┘  │
│  ┌───────────────────────────────┐  │
│  │  组件代码工作区                 │  │
│  │  └── packages/elements/xxx/    │  │
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘
```

### 16.2 MServerWSClient — RPC 客户端

`MServerWSClient`（`src/modules/sandbox/mf.service.ts`）是 agents-hub 侧的 WebSocket RPC 客户端：

```typescript
// 核心能力
class MServerWSClient {
    // 建立 WebSocket 连接到沙箱中的 mfcli server
    async connect(): Promise<void>

    // RPC 调用：发送方法名 + 参数，等待 JSON 响应
    async call<T>(method: string, params: unknown): Promise<T>

    // 流式调用：发送请求，持续接收 SSE 式事件流
    async callStream(method: string, params: unknown): AsyncGenerator<StreamEvent>

    // 工具审批回调：当 mfcli 需要用户确认时触发
    onToolApproval(callback: (tool: string, args: unknown) => Promise<boolean>): void
}
```

**RPC 协议**：每条消息是 JSON-RPC 2.0 格式：
```json
// 请求
{ "jsonrpc": "2.0", "id": "xxx", "method": "generate_code", "params": { ... } }
// 响应
{ "jsonrpc": "2.0", "id": "xxx", "result": { ... } }
// 流式事件
{ "jsonrpc": "2.0", "id": "xxx", "event": "token", "data": { "content": "..." } }
```

### 16.3 沙箱初始化（6 步）

`sandbox-init` 节点执行沙箱环境初始化：

```
1. registry.acquire(threadId)
   → 从沙箱池获取一个可用容器（或创建新的）
   → 返回容器 ID + WebSocket 地址

2. SSO 登录
   → 通过 AsyncLocalStorage 传递 HTTP 请求的 Cookie
   → 在沙箱内执行 SSO 认证，获取用户身份

3. mfcli 资源准备
   → 拉取 mfcli 运行时资源（npm 包、配置文件）

4. mf-server 启动
   → 在沙箱容器内启动 mfcli server
   → 监听 WebSocket 端口

5. MServerWSClient 连接
   → agents-hub 通过 WebSocket 连接到沙箱中的 mfcli server
   → 建立双向通信通道

6. 工作区准备
   → 创建组件代码目录（packages/elements/{name}/）
   → 初始化 pnpm workspace 配置
```

### 16.4 代码生成流水线

`mf-generate` 节点委托给 mfcli 子 Agent 执行代码生成：

```
mf-generate 节点
  → MServerWSClient.callStream('run_agent', { prompt, ... })
  → mfcli server 接收请求 → 内部启动 MyFlicker Agent
  → 流式事件通过 WebSocket 传回：

    event: "thinking"     → Agent 思考过程（透传到前端 SSE）
    event: "tool_start"   → 工具开始执行（透传）
    event: "tool_call"   → 工具调用参数（透传）
    event: "tool_approval" → 需要用户审批（HITL）
        → agents-hub 暂停流 → 派发事件到前端
        → 用户在前端审批
        → agents-hub 通过 WebSocket 发送审批结果回 mfcli
    event: "token"        → 代码生成增量（透传）
    event: "file_write"  → 文件写入事件（透传）
    event: "done"        → 生成完成

  → agents-hub 接收所有事件 → 转换为语义 SSE 事件 → 前端实时展示
```

### 16.5 流式事件透传

mfcli 产生的事件通过 agents-hub 透传到前端 SSE：

```
mfcli (沙箱)                 agents-hub                前端 (E0 Workbench)
    │                            │                          │
    │── event: thinking ────────→│── thinking.delta SSE ──→│
    │                            │                          │
    │── event: tool_start ──────→│── tool.started SSE ───→│
    │                            │                          │
    │── event: tool_approval ──→│── (暂停流) ─────────────→│
    │                            │     ← 用户审批 ──────────│
    │←── approval result ───────│                          │
    │                            │                          │
    │── event: token ───────────→│── message.delta SSE ───→│
    │                            │                          │
    │── event: done ────────────→│── run.completed SSE ──→│
```

### 16.6 工具审批（HITL）

mfcli 中的 MyFlicker Agent 执行工具时可能需要用户审批：

```
1. mfcli 发送 tool_approval 事件 → WebSocket → agents-hub
2. agents-hub 暂停当前流式输出
3. agents-hub 转换为 artifact.action.requested 或自定义审批事件 → SSE → 前端
4. 前端展示审批 UI（工具名、参数、确认/拒绝按钮）
5. 用户点击确认 → 前端 POST 审批结果 → agents-hub
6. agents-hub 通过 WebSocket 发送审批结果 → mfcli
7. mfcli 继续执行工具
8. 流式输出恢复
```

### 16.7 AsyncLocalStorage 上下文传递

沙箱初始化需要 HTTP 请求的 Cookie（SSO 认证），但调用链很深（Controller → Service → Harness → Graph → Node → SandboxService）。使用 `AsyncLocalStorage` 解决：

```typescript
// request-context.ts
const asyncLocalStorage = new AsyncLocalStorage<RequestContext>();

// 请求入口（Middleware）
asyncLocalStorage.run({ cookie: req.headers.cookie, user: req.user }, () => {
    next();
});

// 沙箱初始化时（深层调用链中）
const ctx = asyncLocalStorage.getStore();
// 使用 ctx.cookie 进行 SSO 认证
```

这避免了在每层函数签名中传递上下文参数，保持了代码的整洁性。

### 16.8 关键文件

```
packages/server/jimu-agents-hub/src/
├── modules/sandbox/
│   ├── mf.service.ts              # MServerWSClient — WebSocket RPC 客户端
│   ├── sandbox.registry.ts        # 沙箱池管理（获取/释放容器）
│   └── sandbox-init.ts            # 沙箱初始化逻辑（6 步）
├── harness/graphs/creator-engine/
│   ├── pipeline/
│   │   ├── steps/
│   │   │   ├── sandbox-init.ts    # sandbox-init 节点
│   │   │   └── mf-generate.ts     # mf-generate 节点
│   │   └── pipeline.ts            # 流水线编排
│   └── graph.ts                   # CreatorEngine 图定义
├── common/
│   └── request-context.ts         # AsyncLocalStorage 上下文传递
└── checkpoint/
    └── mysql-checkpointer.ts      # MySQL Checkpoint Saver
```

---

## 十七、知识检索架构（Agentic RAG）

agents-hub 的知识检索不是传统的向量检索 RAG，而是 **Agentic Tool-Calling RAG**——LLM Agent 在运行时自主决定是否检索、检索什么，检索结果作为 `ToolMessage` 进入对话上下文。

### 17.1 与传统 RAG 的区别

| 传统 RAG | agents-hub 的 Agentic RAG |
|---|---|
| Embedding + 向量库 + 余弦相似度 | 无本地向量库、无 embedding 生成 |
| 检索是生成前的预处理步骤 | 检索是 LLM 运行时**自主决定**调用工具 |
| 单轮检索 | **多轮检索**——Agent 可迭代调用工具 |
| 结果注入固定 prompt 模板 | 结果作为 `ToolMessage` 动态进入对话上下文 |

本仓库没有任何 embedding 或向量检索代码，实际的语义检索逻辑全部封装在外部万擎平台内部。

### 17.2 四类知识来源

#### （1）万擎工作流（主要 RAG 后端，12+ 知识库）

所有语义检索委托给快手内部的**万擎平台**（LLM 工作流平台），通过 `getWanqingWorkflow()` 函数发起 HTTP 请求：

```
POST {wanqing.host}/api/llm/workflow/v1/chat/completions
  model: workflowId  (工作流 ID 充当 "model")
  inputs: [{ variable: 'user_query', data: '关键词' }]
  → 返回检索到的知识文本
```

| 知识库 | 检索内容 | 检索模式 |
|---|---|---|
| `search-jimu-platform-desc` | 平台功能描述 | 关键词搜索 |
| `search-jimu-platform-faq-info` | 平台专属 FAQ | 关键词搜索 |
| `search-jimu-element-faq-info` | 组件专属 FAQ | 关键词搜索 |
| `search-element-meta-introduction` | 组件简介 | 关键词搜索 |
| `get-element-meta-by-names` | 组件元数据 | 精确名称查找 |
| `get-apis-usage` | `@jimu/shared-engine` API 用法文档 | 精确名称查找 |
| `get-element-intents` | 组件意图知识（坑点、依赖、模式） | 精确名称查找 |
| `get-coding-paradigms` | 编码范式模板（TS-x / LESS-x） | 精确 ID 查找 |
| `search-jimu-db-info` | C 端数据采集 SQL 信息 | 关键词搜索 |
| `search-kwai-link` | 快手短链信息 | 关键词搜索 |
| `get-image-understand-info` | 图片理解 | 图片输入 |

检索机制（向量搜索/关键词搜索等）封装在万擎工作流内部，对 agents-hub 是黑盒。

#### （2）MCP 网关服务器

| MCP Server | 能力 | 传输方式 |
|---|---|---|
| `kwaishou-search` | 快手内容/作品搜索 | MCP 协议（SSE） |
| `chinese-holiday` | 中国节假日查询 | MCP 协议（SSE） |

通过 `@langchain/mcp-adapters` 的 `MultiServerMCPClient` 连接远程 MCP 服务器，动态加载工具。

#### （3）本地静态知识文件

直接注入 system prompt，无需检索：

| 文件 | 用途 |
|---|---|
| `system-apis-overview.md` | 全量 API 目录（481 行） |
| `global-cognition.md` | 平台背景知识 |
| `schema-parse-method.md` | 页面 Schema 解析方法 |
| `meta-v2-parse-method.md` | 组件元数据 V2 解析方法 |
| `_shared-rules.yaml` | 通用编码纪律 |
| `ts-coding-standard.md` / `less-coding-standard.md` | 编码规范 |
| `schema-specification.md` / `props-logic.md` / `page-fundamentals.md` | 页面构建领域知识 |

#### （4）jimu-server HTTP API（V2 组件元数据）

直接调用主后端 API，支持按 fragment 检索：

```
POST {jimuServer.host}/api/element-store/element-meta-v2/batch-get
  body: { names: ['button', 'image'], fragments: ['index', 'props', 'events'] }
  → 返回组件元数据（V2 格式）
  → 失败时降级到万擎 V1 工作流
```

### 17.3 两种检索模式

**关键词搜索**（模糊，返回排序结果）：

```typescript
// FAQ、平台描述等
getWanqingWorkflow(workflowId, {
    inputs: [{
        variable: 'user_query',
        type: 'text-input',
        data: keyword  // LLM 从用户问题中提取的关键词
    }]
})
```

**精确名称查找**（返回特定文档）：

```typescript
// API 用法、编码范式等
getWanqingWorkflow(workflowId, {
    inputs: [{
        variable: 'api_name',
        type: 'text-input',
        data: 'useJimuElement'  // 精确标识符
    }]
})
```

### 17.4 检索上下文注入方式

#### 方式一：Agent 工具调用（主要机制）

```
Reasoner (LLM) → 决定调用 search_jimu_faq_info 工具
  → Action 节点执行工具 → getWanqingWorkflow()
  → 结果作为 ToolMessage 进入对话 messages 数组
  → Reasoner 下一轮读取 ToolMessage → 生成回答
```

#### 方式二：预注入静态知识（code-gen 子图）

code-gen 的 `Generator` 节点在构建 system prompt 时，将本地知识文件 + 从万擎预取的组件意图知识直接拼入 prompt：

```
buildSystemPrompt(state):
  1. generator-prompt.md（主 prompt）
  2. global-cognition.md（平台背景）
  3. schema-parse-method.md（Schema 解析方法）
  4. meta-v2-parse-method.md（元数据解析方法）
  5. paradigm-directory/_index.md（范式目录索引）
  6. _shared-rules.yaml（通用编码纪律）
  7. ts-coding-standard.md + less-coding-standard.md（编码规范）
  8. [动态] buildPageInventory(state.pageSchema) — 当前页面组件清单
  9. [动态] fetchComponentIntents(selectedEids) — 选中组件的意图知识
  → 全部拼接为 system prompt
```

Agent 在运行时还可通过工具（`get_coding_paradigms`、`get_apis_usage`、`get_elements_meta_v2`）做按需深度检索。

#### 方式三：Platform Oncall 的强制 RAG 策略

oncall prompt 中明确要求检索优先：

> "先检索后回答：禁止凭记忆直接答，必须有工具证据"
> "先执行 FAQ 递进（专题 → 通用），再视缺口搜索知识库，禁止跳过 FAQ 直接查询"
> "检索无结果时，禁止凭记忆编造答案"

### 17.5 知识检索工具一览

#### 全局工具（所有图可用）

| 工具 | 来源 | 检索类型 |
|---|---|---|
| `search_jimu_platform_desc` | 万擎工作流 | 关键词搜索 |
| `search_jimu_faq_info` | 万擎工作流 | 关键词搜索 |
| `search_jimu_platform_faq_info` | 万擎工作流 | 关键词搜索 |
| `search_jimu_element_faq_info` | 万擎工作流 | 关键词搜索 |
| `search_jimu_db_info` | 万擎工作流 | 关键词搜索 |
| `search_element_meta_introduction` | 万擎工作流 | 关键词搜索 |
| `search_kwai_link` | 万擎工作流 | 关键词搜索 |
| `get_current_time` / `get_timestamp` | 本地计算 | N/A |
| `get_image_understand_info` | 万擎工作流 | 图片输入 |
| `emit_human_handoff` | 内部事件 | N/A |

#### 页面修改/理解子图工具

| 工具 | 来源 | 检索类型 |
|---|---|---|
| `get_element_meta_by_names` | 万擎工作流 (V1) | 精确名称查找 |
| `search_element_meta_full_info` | 万擎工作流 | 关键词搜索 |
| `get_profession_agent_prompt` | 本地文件 | 全文加载 |

#### 代码生成子图工具

| 工具 | 来源 | 检索类型 |
|---|---|---|
| `get_apis_overview` | 本地文件 | 全文加载 |
| `get_apis_usage` | 万擎工作流 | 精确名称查找 |
| `get_elements_intro_v2` | jimu-server HTTP API + 万擎降级 | 精确名称 + fragment |
| `get_elements_meta_v2` | jimu-server HTTP API + 万擎降级 | 精确名称 + fragment |
| `get_coding_paradigms` | 万擎工作流 | 精确 ID 查找 |
| `get_element_intents` | 万擎工作流 | 精确组件名查找 |

### 17.6 核心函数：getWanqingWorkflow

```typescript
// src/utils/wanqing.ts
export async function getWanqingWorkflow(
    workflowId: string,
    params: { inputs: { variable: string; type: string; data: string }[] },
    options?: { timeoutMs?: number; signal?: AbortSignal }
) {
    const response = await fetch(
        `${wanqing.host}/api/llm/workflow/v1/chat/completions`,
        {
            method: 'POST',
            headers: {
                Authorization: `Bearer ${wanqing.apiKey}`,
                'X-WQ-End-User-ID': 'jimu-agents-hub',
            },
            body: JSON.stringify({
                model: workflowId,  // 工作流 ID 作为 "model"
                inputs: params.inputs,
                stream: false,
            }),
            signal: options?.signal,
        }
    );
    return data.choices[0].message.content;
}
```

常量配置：
- `WANQING_WORKFLOW_TIMEOUT_MS = 30_000`（单次调用超时 30 秒）
- `WANQING_WORKFLOW_CONCURRENCY = 5`（批量调用最大并发 5）

### 17.7 检索架构总览

```
用户问题
    │
    v
[jimu-master Graph (LangGraph)]
    │
    +-- Reasoner 节点 (LLM)
    │       │
    │       ├── [平台咨询] → search_jimu_platform_faq_info → getWanqingWorkflow() → 万擎知识库
    │       ├── [组件问题] → search_jimu_element_faq_info → getWanqingWorkflow() → 万擎知识库
    │       ├── [页面操作] → page-builder 子图
    │       │                  ├── get_element_meta_by_names (万擎 V1)
    │       │                  ├── search_element_meta_full_info (万擎)
    │       │                  └── get_profession_agent_prompt (本地文件)
    │       └── [代码生成] → code-gen 子图
    │                          ├── get_apis_overview (本地文件)
    │                          ├── get_apis_usage (万擎)
    │                          ├── get_elements_meta_v2 (jimu-server API + 万擎降级)
    │                          ├── get_coding_paradigms (万擎)
    │                          ├── get_element_intents (万擎)
    │                          └── [预注入: 范式目录、共享规则、编码规范]
    │
    +-- Action 节点 (工具执行)
    │
    +-- [MCP 工具: kwaishou-search, chinese-holiday]
            │
            v
        MCP Gateway (外部 SSE 服务器)
```

### 17.8 关键文件

```
packages/server/jimu-agents-hub/src/
├── utils/
│   └── wanqing.ts                          # getWanqingWorkflow — 万擎工作流调用核心
├── harness/
│   ├── tools/
│   │   ├── jimu-platform.ts                # 平台 FAQ/描述/DB 信息搜索工具
│   │   ├── element-meta.ts                 # 组件元数据搜索工具 (V1)
│   │   ├── element-meta-v2.ts             # 组件元数据 V2 (HTTP API + 万擎降级)
│   │   ├── element-intents.ts             # 组件意图知识工具
│   │   ├── api.ts                          # API 用法文档工具
│   │   ├── coding-paradigm.ts             # 编码范式模板工具
│   │   ├── kwai-link.ts                    # 快手短链工具
│   │   ├── time.ts                         # 时间工具
│   │   ├── image.ts                        # 图片理解工具
│   │   ├── handoff.ts                      # 转人工工具
│   │   └── profession.ts                  # 页面构建领域知识工具
│   ├── mcps/
│   │   ├── kwaishou-search.ts              # 快手搜索 MCP
│   │   └── chinese-holiday.ts             # 节假日 MCP
│   └── graphs/
│       ├── jimu-master/prompt/
│       │   ├── 10-routing-priority.md      # 路由优先级（决定何时检索）
│       │   └── domains/platform-info.md    # 平台咨询路由指令
│       ├── code-gen/
│       │   ├── cognition/                  # 本地静态知识文件
│       │   └── knowledge/                  # 范式目录、共享规则
│       └── platform-oncall/
│           └── prompt.md                   # 强制 RAG 策略 prompt
├── config/
│   └── config.default.ts                   # workflow-ids 配置（万擎知识库映射）
└── harness/config.yaml                     # 工具 scope 配置
```
