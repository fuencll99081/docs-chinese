# 面试准备 — 简历项目描述逐条拆解

> 面向 AI 全栈工程师岗位，针对简历 6 条项目描述，逐条整理面试话术、高频追问及回答要点。

---

## 第 1 条：可视化编辑器与渲染引擎

**简历原文**：负责低代码平台核心能力建设，主导可视化编辑器与渲染引擎开发。设计响应式 Map + Pinia 混合状态管理与 Diff-based 撤销重做系统（内存降低 80%+），实现双阶段加载引擎（FCP 降低 50%）。

### 描述话术

我负责编辑器核心和渲染引擎两块。编辑器侧，元素数据量很大（一个页面可能上百个组件），如果全放 Pinia 会导致每次更新都触发大范围响应式追踪，性能很差。所以我设计了响应式 Map + Pinia 的混合方案：元素数据用 `reactive(new Map())` 做细粒度追踪，宏观状态（选中、缩放、锁状态等）放 Pinia。

撤销重做方面，传统 snapshot 方案每次存全量 Schema，内存占用大。我改成了 Diff-based 方案——每次操作只记录元素级 diff（add/update/remove），undo 时反向应用 diff，内存降低 80%+。同时引入了事务机制，多步操作可以打包为原子单元，要么全成功要么全回滚。

引擎侧做了双阶段加载：首屏组件（视口内）优先激活，懒加载组件延迟按批次加载。组件 JS 初始以 `type="text/plain"` 放在 HTML 里不执行，引擎按视口检测逐个激活，首屏 FCP 降低约 50%。

### 面试官可能追问

**Q1：响应式 Map 相比直接用 Pinia 存储元素，具体优势是什么？为什么不用数组？**

用数组的话，增删改需要用索引或 filter 操作，Vue 3 对数组的响应式追踪是基于索引的，插入/删除中间元素会导致后续所有元素的 ref 失效。用 Map 的话，key 是 eid（唯一标识），增删改不影响其他 key 的响应式关系。而且 Map 查找是 O(1)，元素遍历用 `Map.values()` 即可。Pinia 适合存"全局只有一个"的状态（如当前选中、缩放比例），元素是"同类型多个实例"，天然适合 Map。

**Q2：Diff-based 撤销重做，如果连续修改同一个元素的同一个属性多次，diff 会很大吗？**

不会。每次 commit 只记录从上一个 baseline 到当前状态的 diff。比如一个元素的 width 改了 3 次：100→200→300→200，最终 diff 只记录 width: 200（相对于 baseline 100）。undo 时反向应用：width 恢复为 100。中间的 200→300 这些中间态不会被记录。另外我们有 history 上限控制（默认 50 步），超出时丢弃最旧的。

**Q3：双阶段加载怎么判断哪些组件是"首屏"？视口检测的 threshold 是多少？**

不是运行时判断的。发布时由服务端 render 模块根据组件的 layout 配置（FLOOR_AUTO / FLOOR_BLOCK）和顺序决定。简单来说：按文档流顺序，前 N 个（通常第一个楼层区域）放入 `initialSchema`（首屏），其余放入 `nextSchema`（懒加载）。首屏 HTML 里组件 script 标签是 `type="text/plain"`（不执行），引擎启动后按视口检测（IntersectionObserver，rootMargin 向下扩展一屏）激活组件脚本。

**Q4：事务机制如何处理嵌套场景？比如一个事务中包含子事务？**

事务支持嵌套，子事务的 commit 不会立即生成 HistoryEntry，而是把 diff 冒泡到父事务。只有最外层事务 commit 时才生成一条 HistoryEntry。但如果子事务 rollback，只回滚子事务的操作，父事务不受影响。这通过维护一个事务栈实现，每个事务持有自己的 diff 收集器。

---

## 第 2 条：多 Agent 编排框架

**简历原文**：设计基于 LangGraph + NestJS 的多 Agent 编排框架，实现 11 级路由编排与子图委托模式，Token 消耗降低约 40%。

### 描述话术

我基于 LangGraph + NestJS 封装了一套面向对象的 Agent 框架。核心是 BaseGraph / BaseSubgraph / BaseNode / BaseTool 四个抽象基类，把 LangGraph 的底层函数式 API 转成继承模式，业务开发者只需关注领域逻辑。

主图 JimuMaster 有 11 级路由优先级，根据用户意图分发到不同子图（页面修改、页面理解、代码生成等）。子图通过委托模式注册为工具——LLM 调用时，父图清空 messages、注入结构化输入，子图独立执行，结果打包成 ToolMessage 返回。这样子图不继承父图的完整对话历史，Token 消耗降低约 40%。

工具系统是配置驱动的，config.yaml 定义每个工具的 scope，运行时按 scope 过滤，同一个工具类只出现在合适的图里。支持 4 类工具来源：原生工具、MCP 工具、Skill 工具、子图工具。

### 面试官可能追问

**Q1：委托模式具体怎么实现上下文隔离的？为什么能降 40% Token？**

父图调用子图时走 `delegateInputNode`：提取 LLM 的 tool_call 参数，清空 messages，注入一条新的 HumanMessage（只包含结构化输入）。子图拿到的是干净的上下文，执行完毕后 `delegateOutputNode` 把结果打包成 ToolMessage 返回。父图收到的只有结果摘要，不含子图内部的完整对话。

40% 的测算：以 PageBuilder 为例，一次典型的页面修改任务子图内部可能有 5-8 轮 LLM 调用（planner + builder 循环 + validator），每轮包含完整的系统 prompt + 工具定义 + 历史消息。如果这些都在父图上下文里，父图后续每次调用都要携带这些消息。委托模式下父图只持有一条 ToolMessage（约 200-500 token），而子图内部完整对话可能有 5000-8000 token。

**Q2：11 级路由优先级是怎么设计的？有没有误判的情况？**

路由优先级在 prompt 的 `10-routing-priority.md` 中定义，从高到低：紧急转接 > 页面操作延续 > 明确的页面修改意图 > 页面理解 > 代码生成 > 设计稿转换 > 图片生成/编辑 > 平台信息查询 > 闲聊 > 其他。Reasoner 节点通过 LLM 的 tool_call 来路由——LLM 决定调用哪个子图工具，就路由到哪个子图。

误判确实存在。我们的应对策略：一是 prompt 中的路由优先级描述非常明确，给了大量正例/反例；二是子图可以拒绝执行（比如 PageBuilder 检测到实际不是页面修改任务时返回提示）；三是子图执行完后有 `delegated-routing` 节点决定是否回 reasoner 重新路由。

**Q3：BaseGraph 为什么要缓存编译后的实例？**

LangGraph 的 `compile()` 是有开销的——需要构建图结构、注册节点、绑定边、配置 checkpoint saver。如果每次请求都重新编译，浪费 CPU。缓存 key 是图类名，编译后的实例是无状态的（状态通过 checkpoint 按 thread_id 隔离），所以多请求复用安全。

**Q4：config.yaml 的 scope 机制如果配错了会怎样？**

不会导致运行时崩溃。scope 过滤发生在图初始化阶段（`loadTools()`），如果某工具的 scope 不包含当前图名，该工具不会被加载。LLM 在该图中看不到这个工具，自然不会调用。配错的后果是"工具缺失"而非"工具错误"。开发时通过单元测试验证 scope 配置的正确性。

---

## 第 3 条：HITL + 流式管道 + 可观测性

**简历原文**：集成 LangGraph interrupt/resume 实现 Human-in-the-Loop，保障 AI 修改安全可控。实现 14 种语义事件流式管道，集成 Langfuse 全链路追踪，支撑 AI 辅助页面修改效率提升 5 倍。

### 描述话术

页面修改是高危操作，AI 改错了影响线上活动。所以我用 LangGraph 的 interrupt/resume 机制实现了 Human-in-the-Loop：AI 生成变更后调用 `interrupt()` 暂停图执行，前端展示变更预览（diff 卡片），用户确认后调用 `resume()` 恢复，恢复时从父图同步最新页面 Schema 到子图 checkpoint，确保基于最新数据验证。

流式管道方面，LangGraph 原始事件是底层 API 级别的（`on_chat_model_stream`、`on_tool_start` 等），我封装了一层 StreamTransformer 转换为 14 种语义事件（消息增量、思考过程、工具生命周期、任务快照等）。还有一个 Thinking Candidate 缓冲机制——有些模型不区分思考和回复，我用缓冲区在 LLM 调用结束时决定内容归属。

服务端做了 Projection——流式输出的同时聚合完整消息内容，持久化到 MySQL，用户断连后能从数据库恢复，不需要重新请求 LLM。可观测性方面集成了 Langfuse 全链路追踪，每个节点/工具的耗时、Token 消耗都有记录。

### 面试官可能追问

**Q1：HITL 的 interrupt/resume 在分布式环境下如何保证状态一致？**

状态存在 MySQL 的 checkpoint 表里，不是内存。interrupt 时 LangGraph 自动持久化 checkpoint（thread_id + checkpoint_id + step），resume 时从数据库加载。多实例部署下，resume 请求可能打到不同实例，但因为状态在共享存储中，任何实例都能恢复。

一个需要注意的点是子图的状态同步。LangGraph 的 resume 默认只恢复当前图的 checkpoint，但 PageBuilder 是子图，用户在 HITL 期间可能在前端做了其他修改导致父图 Schema 变了。所以我实现了 `getResumeUpdateForInterruptedState()`，在 resume 时从父图 JimuMaster 拉取最新状态手动同步到子图 checkpoint。

**Q2：Thinking Candidate 缓冲具体怎么工作的？**

某些模型（如通义千问某些版本）的 streaming 输出不区分"思考内容"和"正式回复"——所有 token 都走同一个 stream channel。Transformer 维护一个缓冲区，收到 `on_chat_model_stream` 事件时把 token 存入缓冲区而不立即发出。当收到 `on_chat_model_end` 事件时，调用 flush 逻辑判断：如果这轮 LLM 调用属于"思考"阶段（通过节点配置标记），缓冲内容作为 `thinking.delta` 发出；否则作为 `message.delta` 发出。这样前端可以分别展示思考区和正文区。

**Q3：14 种语义事件为什么要这么多？能不能简化？**

每种事件对应前端不同的 UI 渲染逻辑。比如 `tool.started` 和 `tool.completed` 是分开的，因为前端要展示工具执行中的 loading 状态和完成后的结果；`task.snapshot` 单独一个事件是因为 TodoList 是状态快照而非增量，前端整量替换；`page.action.batch` 是 HITL 专属事件，触发前端 diff 卡片展示和"应用/撤销"按钮。如果合并成 3-4 种事件，前端就需要自己解析原始事件做状态机推断，复杂度反而更高。

**Q4：Projection 机制如果流式中途服务挂了怎么办？**

Projection 是实时累积的——每次收到事件就更新内存中的 projection 对象。流正常结束时写库。如果中途挂了，已发出的 SSE 事件客户端已经收到了，客户端本地有内容。用户下次进入对话时，从数据库加载消息——如果是 streaming 状态的消息（status 字段标记），前端会显示"生成中断"并保留已收到的部分内容。不会出现"白屏"的情况。

---

## 第 4 条：Headless 编辑核心 + 双侧 API

**简历原文**：设计 Headless 页面编辑核心与程序化操作 API，打通 AI Agent 与低代码平台。设计 define/use 双侧 API 体系，支撑 100+ 组件灵活数据绑定与动态渲染。

### 描述话术

传统编辑器把 UI 和逻辑耦合在一起，AI Agent 想操作页面得模拟用户交互（模拟点击、模拟拖拽），既慢又不可靠。我把编辑器核心能力抽出来做成 Headless 编辑会话（`page-editing-core`），提供程序化 API：增删改元素、选中、校验、构建保存载荷，完全不需要 Vue 组件参与。

AI Agent 通过 page-builder 子图调用这些 API，操作完直接构建 Schema 提交给后端。这是低代码平台和 AI Agent 之间的桥梁——AI 操作的是 Schema 数据结构，不是 UI 交互。

组件生态方面，我设计了 define/use 双侧 API：B 端用 `defineJimuProps`/`defineJimuStyles` 声明组件的属性/样式/行为约束，C 端用 `useJimuProps`/`useJimuEvent` 获取响应式数据和事件能力。中间通过 `@jimu/shared-definitions` 的类型契约协作。另外设计了 4 种属性值类型（静态/引用/计算/继承），实现组件间响应式数据绑定。

### 面试官可能追问

**Q1：Headless 编辑核心和带 UI 的编辑器是什么关系？能完全替代吗？**

不是替代关系，是复用关系。Headless 编辑会话提供了元素管理、选中状态、操作执行、校验、保存等核心能力。带 UI 的编辑器在 Headless 之上加了一层 Vue 组件——画布、属性面板、组件树等 UI 组件消费 Headless 的数据。AI Agent 直接用 Headless API，不经过 UI 层。

不能完全替代——画布拖拽、实时预览、属性面板交互这些是 UI 专属的，Headless 不提供。Headless 提供的是"数据层"能力，UI 提供的是"交互层"能力。

**Q2：define/use 双侧 API 如何保证类型安全？**

B 端 `defineJimuProps` 返回一个 `IJimuPropsDefine` 类型对象，包含每个 prop 的类型、默认值、校验规则。C 端 `useJimuProps()` 返回的对象类型由 `IJimuPropsDefine` 推导而来（通过 TypeScript 泛型）。两边通过 `@jimu/shared-definitions` 中的接口定义做契约——`defineJimuProps` 的类型参数约束了 prop 的 key 和 value 类型，`useJimuProps` 的返回类型由运行时注入的 define 对象决定。开发时 TypeScript 能做到编辑器配置面板的 prop key 补全和类型检查。

**Q3：4 种属性值类型（静态/引用/计算/继承）运行时怎么解析？**

Schema 中每个 prop 是一个 `IPropSchema` 对象，有 `type` 字段：
- `{ type: 's', value: 'hello' }` — 静态值，直接用
- `{ type: 'q', dst: 'elementB', path: 'props.title' }` — 引用，运行时通过 `resolveQuery()` 从目标元素读取值，Vue 的响应式会自动追踪依赖
- `{ type: 'c', code: 'return this.a + this.b', dependencies: ['a', 'b'] }` — 计算，运行时用 `new Function` 执行 code，dependencies 声明依赖的 prop 名
- `{ type: 'i' }` — 继承，运行时向上遍历父元素，找到第一个非 inherit 类型的同名 prop 作为值

**Q4：AI Agent 调用 Headless API 的典型流程是什么？**

以"把按钮的文案改成'立即参与'"为例：
1. PageBuilder 子图的 Builder 节点生成一个 `JIMU_UPDATE_ELEMENT` action fence（Markdown 格式的操作指令）
2. ActionInterrupter 解析 fence 提取 actions（eid + prop path + value）
3. 派发 `page_action_batch` 事件到前端
4. 前端调用 `session.applyActions(actions)` → Headless 编辑会话执行 `ComponentOperationExecutor.update()`
5. 用户确认后 resume，ActionValidator 通过 Headless API 读取修改后的 Schema 验证结果

---

## 第 5 条：Monorepo + SSE 流式 + 工程优化

**简历原文**：优化 Monorepo 工程体系、编辑器状态管理与 SSE 流式通信，提升研发效率与用户体验。

### 描述话术

工程层面我们用 pnpm workspace + catalog 统一管理 20+ 包的依赖版本。B 端和 C 端通过 `@jimu/shared-definitions` 做类型契约，`@jimu/shared-editor` 和 `@jimu/shared-engine` 分别提供运行时工具，确保 C 端打包不引入 B 端冗余代码。

SSE 流式通信方面，从 LLM 到用户端全程流式——LangGraph streamEvents → StreamTransformer 转换 → 服务端 Projection 聚合 → stream2SSE（心跳保活 + [DONE] 终止符）→ HTTP SSE Response。中间有心跳保活防止网关超时断连。

编辑器状态管理也在持续优化，比如元素存储从数组改成响应式 Map、事务嵌套支持、多标签页冲突检测（BroadcastChannel）等。

### 面试官可能追问

**Q1：SSE 心跳保活具体怎么做？为什么要做？**

SSE 是单向长连接，如果长时间没数据发送，中间的网关/负载均衡可能判定为空闲连接而断开。我们每 15 秒发一个 `:heartbeat\n\n`（SSE 注释行，客户端忽略），保持连接活跃。当流正常结束时发 `[DONE]` 标记，客户端收到后关闭 EventSource。

**Q2：pnpm catalog 是什么？和普通 pnpm workspace 有什么区别？**

pnpm catalog 是 pnpm 9.x 引入的版本集中管理机制。在 `pnpm-workspace.yaml` 中定义 catalog，所有包引用时写 `catalog:` 而非具体版本号。好处是升级 Vue 时只改 catalog 一处，所有包同步升级，不会出现 A 包用 Vue 3.3、B 包用 Vue 3.4 的版本碎片问题。

**Q3：多标签页冲突检测怎么做的？**

用 `BroadcastChannel('jimu_editor')` 实现。打开编辑器时广播一个 `JOIN` 消息（带 activityId），已有标签页收到后回复 `EXIST` 消息。新标签页检测到冲突后弹窗提示"该活动已在其他标签页打开"。同时限制最多 3 个并发编辑窗口（`MAX_COUNT`），超过时阻止打开。这是为了防止多标签页操作产生不可预知的冲突——虽然有协同编辑（OT），但多标签页是同一用户操作，容易误操作。

**Q4：B 端和 C 端的代码隔离具体怎么做？**

三个层次的隔离：
1. 包隔离：`@jimu/shared-editor`（B 端运行时）和 `@jimu/shared-engine`（C 端运行时）是独立 npm 包
2. 类型契约：`@jimu/shared-definitions` 只有类型定义无运行时代码，B/C 两端都依赖它但不互相依赖
3. 模块注入：`setupSharedEditor` 通过 `initExports` 把 Vue 实例注入共享包，共享包不直接 import Vue，避免 C 端打包时引入 Vue 的 B 端相关代码

---

## 第 6 条：大型活动保障 + 平台定制

**简历原文**：参与大型营销活动技术保障及平台定制能力建设，支撑日均千级活动页面创建与发布。

### 描述话术

平台支撑日均千级活动页面的创建与发布。出页服务做了多级缓存（Redis → 请求去重 → BlobStore → 降级回源），高并发场景下保障出页稳定性。发布时 HTML 经 Brotli 压缩存储到 Blob Storage，CDN 分发。

平台定制能力方面，通过 Context 桥接模式适配快手/快手Pro/快影三个平台——一套 Schema 多平台运行，平台差异通过统一 `IContext` 接口抹平。每个平台继承 BaseContext 提供 passport/storage/logger/radar/klink 等能力的不同实现。

营销活动保障方面，大促前会做容量评估、缓存预热、降级预案。出页服务独立于编辑后端部署，可独立扩缩容，大促时临时扩容出页实例。

### 面试官可能追问

**Q1：多级缓存出页的缓存失效策略是什么？发布后怎么保证用户立即看到新版本？**

发布时更新版本标签（新版本标记 ONLINE，旧版本标记 OFFLINE），同时生成新的 HTML 写入 Blob Storage（key 包含版本号）。出页服务的 Redis 缓存 key 是 `page:{token}`，token 中编码了版本号。发布后 token 变化（因为版本变了），所以新请求自然 miss 缓存、回源 BlobStore 拿到新版本。旧版本的 Redis 缓存靠 TTL 自然过期（通常 10 分钟）。

另外发布时会触发 CDN 缓存刷新（通过 CDN API 主动 purge），确保 CDN 层也不会返回旧版本。

**Q2：请求去重是怎么做的？多个用户同时请求同一个页面会怎样？**

用 `FetchingCache` 实现——对同一个 cache key 的并发请求只发一个实际回源请求，其他请求 await 同一个 Promise。第一个请求完成后所有等待者共享同一个结果。这是基于 `Map<string, Promise>` 实现的：请求时检查 Map 中是否已有 inflight Promise，有则直接 await，没有则创建并放入 Map，完成后删除。

**Q3：Context 桥接模式有没有遇到平台差异特别大、接口统一不了的情况？**

有的。比如快手的 passport 能力在 App 内通过 JSBridge 调用，快影的可能通过 Webview 的 `postMessage`。统一接口层面我们定义 `IPassport.getUserInfo(): Promise<UserInfo>`，各平台实现里处理差异。遇到真正无法统一的场景（比如某平台根本没有 liveRoom 能力），接口返回 `undefined` 或 `unsupported` 错误码，组件通过 `useJimuContext()` 调用时自行处理降级逻辑。不会因为平台能力缺失导致整个页面崩溃。

**Q4：日均千级页面，QPS 峰值大概多少？出页服务多少实例？**

（按实际数据回答。示例：大促期间出页 QPS 峰值约 X 万，常态约 X 千。出页服务常态 4-8 实例，大促临时扩到 16-32 实例。Redis 用集群模式，BlobStore 用快手内部的对象存储服务。）

---

## 通用建议

1. **先说"做了什么"，再说"为什么这么做"**：面试官更关心设计思考过程，而非堆砌技术名词
2. **准备好量化数据的来源**：80% 内存降低怎么测的、40% Token 降低怎么算的、50% FCP 提升的对比基准
3. **主动提及踩过的坑**：比如子图状态同步问题、流式渲染 fence 边界问题，展示解决问题的深度
4. **对技术选型有观点**：为什么选 LangGraph 而不是自己写状态机？为什么用 markdown-it 而不是 marked？这些能展示技术判断力
