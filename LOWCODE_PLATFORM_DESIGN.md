# Jimu V5 低代码可视化搭建平台 — 设计与实现思路

> 本文聚焦 AI Agent 框架之外的低代码平台本身，涵盖编辑器、渲染引擎、组件生态、后端服务、发布部署等完整链路。

---

## 一、平台定位

快手商业化活动搭建平台（积木 V5），服务于公司内部运营人员的可视化页面搭建。运营人员通过拖拽组件、配置属性、设置事件交互，无需写代码即可生成可在快手 App 内运行的 H5 活动页面。平台支撑日均千级活动页面的创建与发布。

---

## 二、Monorepo 架构

采用 pnpm workspace + catalog 统一管理，包结构按职责分层：

```
packages/
  client/                          # 前端应用与共享库
    editor/                        # 可视化编辑器（B 端）— Vue 3 + Pinia + Element Plus
    engine/                        # C 端渲染引擎 — UMD 库
    www/                           # 管理后台 — 活动列表、数据看板、权限管理
    context/                       # 平台桥接层 — 快手/快手Pro/快影 API 适配
    console/                       # 运行时调试面板
    shared/
      definitions/                 # 纯类型定义（无运行时代码）
      editor/                      # B 端共享（defineJimuProps、控件、依赖管理）
      engine/                      # C 端共享（useJimuProps、事件、i18n、样式）
      platform/                    # 跨端共享（权限、API 客户端）
    system-elements/               # 内置系统组件（application/block/group）
    page-editing-core/             # Headless 页面编辑会话（E0 新架构，为 AI 集成设计）
    page-scripts/                  # 运行时脚本（polyfill、context-preloader、radar-seed）
    preview-extra/                 # 预览模式增强脚本（热更新、审查模式）
    service-worker/                # 编辑器/引擎资源缓存

  server/
    jimu-server/                   # 主后端 — Egg.js（活动 CRUD、组件市场、权限、渲染）
    jimu-server-page-service/      # 页面服务 — NestJS + TypeORM（CDN/BlobStore 高性能出页）
    jimu-agents-hub/               # AI Agent 服务（本文不展开）

  tools/
    cli/                           # @jimu/cli-next — 脚手架、构建、发布、本地开发
    local-deploy/                  # 本地部署脚本

  chore/
    permission/                    # 权限 DSL 解析器（ANTLR 文法 — ReBac）
    shared-utils/                  # 通用工具库
```

**设计思路**：B 端（编辑器）和 C 端（引擎）共享类型定义但各自独立实现，通过 `@jimu/shared-definitions` 做类型契约，`@jimu/shared-editor` 和 `@jimu/shared-engine` 分别提供运行时工具，避免 C 端打包引入 B 端冗余代码。

---

## 三、可视化编辑器架构

### 3.1 整体布局

经典 IDE 式布局（`editor-framework.vue`）：

| 区域 | 组件 | 职责 |
|------|------|------|
| 顶栏 | TopBar | 活动名、保存/发布、二维码预览、缩放、设置 |
| 左侧面板 | ElementSidePanel | 组件市场（搜索、分类、模板） |
| 中心画布 | MainStage | 引擎预览画布、浮动菜单、屏幕线、楼层指示器 |
| 中心底部 | LayerManager | 组件树（拖拽排序、搜索、编辑） |
| 右侧面板 | ControlPanel | 属性配置面板（单选/多选/事件/样式/代码） |
| 浮层 | CopilotChat | AI 对话面板、自定义代码编辑器、Figma D2C |

### 3.2 启动流程

```
main.ts
  → 创建 Pinia store
  → 初始化 i18n / DevTools / Logger
  → 注册自定义指令（focus/resize/log）
  → setupSharedEditor() — 注入 Vue + shared-editor + shared-engine 到模块系统
  → setupEditorContext() — 初始化编辑器上下文（app model、passport）
  → 注册控件组件
  → 登录校验 → 加载活动数据 → 挂载 #app
```

**关键设计**：`setupSharedEditor` 通过 `initExports` 将 Vue 实例和共享包注入模块系统，使共享包代码可以引用 Vue 而不需要直接 import，解耦了包间依赖。

### 3.3 Pinia 状态管理

| Store | 职责 |
|-------|------|
| `editor` | 选中状态、用户信息、锁状态、暗黑模式、搜索历史、网络状态 |
| `element` | **响应式 Map** 存储画布所有元素（by eid），增删改查、开发组件检测 |
| `activity` | 活动元数据、保存/发布流程、Schema 构建、版本管理 |
| `stage` | 画布状态：缩放、位置、吸附、屏幕线 |
| `history` | 撤销/重做（基于元素 diff），防抖自动收集 |
| `source` | 源组件注册表（组件定义），版本管理，状态：basic → full / error |
| `multiplayer` | 实时协作编辑（基于 CRDT — Koko） |
| `version-manager` | 版本历史查看与 diff |

**关键设计**：元素存储使用 `reactive(new Map<Eid, IElementModel>())` 而非 Pinia，因为元素操作极其频繁且需要细粒度响应式追踪。Pinia Store 负责宏观状态，响应式 Map 负责元素数据。

### 3.4 事务与历史系统

```
Transaction（事务）
  ├── 开始事务 → SchemaDiffer 开始监听元素变化
  ├── 执行多个操作（增/删/改/移动）
  ├── commit → 收集 IElementDiff → 生成 HistoryEntry → 推入 HistoryManager
  └── rollback → 回滚所有 diff
```

- **Transaction**：将多个操作组合为原子单元，支持嵌套（父子事务）
- **SchemaDiffer**：监听元素变化生成 diff
- **HistoryManager**：存储 diff 历史，undo/redo 通过应用/反转 diff 实现

**设计思路**：diff-based 而非 snapshot-based，大幅减少内存占用，支持细粒度撤销。

### 3.5 锁与权限

- 活动锁：`tryLock` / `forceLock` / `releaseLock`，防止多人同时编辑冲突
- 协作模式：基于 CRDT（Koko）实现实时协作，绕过活动锁
- 权限层级：`can_manage` / `can_edit` / `can_read`

---

## 四、渲染引擎设计

### 4.1 引擎职责

引擎是一个 UMD 库，运行在 C 端（用户浏览器），负责将 JSON Schema 渲染为可交互的 Vue 页面。

### 4.2 双阶段加载（首屏优化）

```
Phase 1: 首屏组件（视口内）
  → HTML 中组件 script 标签为 type="text/plain"（不执行）
  → 引擎激活首屏组件 script → 执行 → 注册组件 → 渲染

Phase 2: 懒加载组件（视口外）
  → 首屏渲染完成后，批量激活（每批 10 个）
```

**设计思路**：Schema 拆分为 `initialSchema`（首屏）和 `nextSchema`（懒加载），分别放在两个 `<script type="text/template">` 标签中。组件 JS 初始为 `type="text/plain"` 不执行，引擎按需激活，实现首屏极速加载。

### 4.3 Wrapper 组件 — 渲染核心

`Wrapper` 是所有组件的渲染容器：

```
<Wrapper eid="app">
  → 按 eid 查找元素模型
  → 创建 StyleHelper（生成布局/定位/动画 CSS）
  → 渲染 element.source.component（组件的 view.vue）
  → 设置 InViewportChecker（曝光埋点 + 懒加载触发）
  → 注入 ClickCollector（点击热力图）
  → 应用自定义 CSS/JS（SystemProp.COMPILED_CODE）
  → 递归渲染子组件 <Wrapper eid="child.eid" />
```

### 4.4 平台上下文桥接

根据 URL 参数 `jimu_target` 动态加载平台上下文：

```
jimu_target=kwai     → KwaiContext（快手主 App）
jimu_target=kwaipro  → KwaiProContext（快手 Pro）
jimu_target=kwaiying → KwaiYingContext（快影）
```

每个 Context 继承 `BaseContext`，提供统一接口：passport、storage、logger、radar、klink、share、media、liveRoom、bridge。

**设计思路**：一套页面 Schema 可以在不同平台运行，平台差异通过 Context 桥接抹平。

---

## 五、积木组件体系

### 5.1 标准组件结构

```
element-name/
  config.ts              # B 端定义：defineJimuProps / defineJimuStyles / defineJimuBehavior
  view.vue               # C 端渲染：useJimuProps / useJimuContext / useJimuEvent
  control.ts             # B 端控件导出
  control-components/    # B 端属性面板 Vue 组件
  jimu.d.ts              # 类型声明
  manifest.json          # 元数据：name/cname/version/group/icon/apps/tags
  meta.md                # 组件文档
```

### 5.2 组件定义 API（B 端）

`defineJimuProps` / `defineJimuStyles` / `defineJimuBehavior` 等函数在 `config.ts` 中定义组件的属性、样式、行为约束：

- **Props**：带类型默认值、校验规则、依赖关系，支持点号扁平化嵌套
- **Styles**：通过 `StyleKey` 枚举声明可配置样式（宽高/背景/字体/边距等）
- **Layout**：布局类型（FLOOR_AUTO / FLOOR_BLOCK 等）
- **Behavior**：行为约束（canAddChild / canDelete / canCopy / canDragInTree）
- **Emitters**：声明组件可发射的事件

**执行机制**：`runWithDefineContext` 执行 config.ts 函数，将所有定义捕获到 `IDefineContext` 中，编辑器据此生成属性面板和约束规则。

### 5.3 组件运行时 API（C 端）

`useJimuProps` / `useJimuContext` / `useJimuEvent` 等 Composition API 在 `view.vue` 中使用：

- `useJimuElement()` — 获取当前元素模型
- `useJimuProps()` — 获取所有属性（响应式对象）
- `useJimuPropRef(path)` — 获取单个属性的响应式 ref
- `useJimuContext()` — 获取平台上下文（passport/storage/logger）
- `useJimuEvent()` — 事件系统（emit/handle/subscribe/broadcast）
- `useJimuLogger()` — 埋点日志（sendClick/sendShow/sendTask）
- `defineJimuOutput(exposed)` — 定义可被其他组件引用的输出值

### 5.4 组件注册流程

```
编辑器：
  组件 JS 加载为 <script> → window.JIMU.registerElement(elementExport)
  → elementExport 包含 { manifest, config, component, controlComponent, i18n }
  → loadSourceElementFromExport 存入 source store（status: basic）
  → execElement 执行 config 函数 → 升级为 full

引擎：
  系统组件先加载 → 组件脚本从 HTML 激活 → registerElement
  → importElements(schema) 创建元素模型
  → registerElementSchema 填充 props/events/父子关系
```

### 5.5 依赖系统

三类依赖关系：

| 类型 | 说明 | 场景 |
|------|------|------|
| 强依赖 | 组件 A 不能脱离 B 存在 | 删除时级联删除 |
| 引用依赖 | 属性引用另一个元素的值 | 组件间数据绑定 |
| 事件依赖 | 组件间事件订阅 | 删除时清理事件链 |

### 5.6 属性值类型系统

Schema 中属性支持 4 种值类型，实现灵活的数据绑定：

```typescript
type IPropSchema =
  | { type: 's', value: unknown }                    // 静态值
  | { type: 'q', dst: string, path: string }         // 引用（指向另一个元素的属性）
  | { type: 'c', code: string, dependencies: [] }   // 计算属性（函数体）
  | { type: 'i' }                                     // 继承父级
```

**设计思路**：引用类型实现组件间响应式数据绑定，计算类型支持自定义逻辑，继承类型支持主题/布局参数透传。

---

## 六、页面 Schema 与数据模型

### 6.1 Schema 结构

```typescript
type ISchema = IElementSchema[]

interface IElementSchema {
  eid: string              // 元素 ID（十六进制）
  source: string            // "componentName@version"
  props: Record<string, IPropSchema>
  parent: string            // 父元素 eid
  children?: string[]       // 子元素 eid 列表
  master?: string           // 主元素 eid（主从模式）
  slaves?: string[]         // 从元素 eid 列表
  events?: IEventSchema[]   // 事件配置
}
```

### 6.2 双 Schema 架构

- **Editor Schema**：包含 `editorOnly` 属性（活动名、项目 ID）、校验结果、额外参数
- **Engine Schema**：剥离编辑器专属数据，事件 ID 压缩，用于 C 端渲染

### 6.3 活动数据模型

```typescript
interface IPage {
  id: number;
  activityId: number;
  version: number;
  dependencies: ActivityDependencies;  // 组件 JS/CSS/引擎/上下文 URL
  engineSchema: ISchema;                 // C 端 Schema
  editorSchema: ISchema;                 // B 端 Schema
  pageContent: PageContent;              // HTML/CSS/脚本 外壳
  urlParams: { app, params }[];
}
```

`dependencies` 对象记录页面运行所需的所有资源 URL，是引擎加载组件的清单。

---

## 七、后端服务架构

### 7.1 jimu-server（Egg.js — 主后端）

**路由分层**：

| 路由 | 职责 |
|------|------|
| pageRouter | HTML 页面路由（编辑器、预览、管理后台） |
| apiRouter | JSON API（20+ 子路由：活动/组件/模板/权限等） |
| bizRouter | 业务路由（抽奖/排行榜/钱包等业务能力） |
| proxyRouter | API 代理 |

**核心服务**：

- `activity.ts`（2900+ 行）：活动 CRUD、Schema 持久化、页面文件生成、发布流程、截图、Blob 存储
- `render.ts`（900+ 行）：HTML 页面渲染器，生成完整 HTML（预加载、内联样式、脚本排序、HTML 压缩、CDN 保护）
- `element-store.ts`（1800+ 行）：组件市场 — 组件 CRUD、版本管理、Blob 存储
- `version.ts`：版本管理（ONLINE/OFFLINE 标签）
- `permission.ts`：权限管理

**数据模型**：ActivityMeta、ActivityVersion、ActivitySchema、ActivitySnapshot、ElementInfo、ElementVersion、ElementGroup 等。

### 7.2 jimu-server-page-service（NestJS — 出页服务）

专门的高性能页面分发服务，独立于主后端：

```
GET /doodle/:token
  → PageCacheManager（Redis 缓存）
  → FetchingCache（请求去重 — 合并相同请求）
  → BlobStoreService（主数据源 — 发布的 HTML 文件）
  → ProxyService（降级 → 回源 jimu-server）
  → CDN 域名替换 + Brotli 压缩 + ETag/304
```

**设计思路**：出页服务与编辑后端分离，独立扩缩容。多级缓存（Redis → 请求去重 → BlobStore → 降级回源）保障高并发下的出页性能。

---

## 八、发布与部署流程

### 8.1 完整数据流

```
1. 编辑器（B 端）
   ├── 用户拖拽组件到画布 → 组件 JS 加载 → registerElement
   ├── 用户配置属性 → 存入 IElementModel.props（响应式）
   ├── 点击保存
   │   ├── buildEditorSchema() — 序列化所有元素（含 editorOnly）
   │   ├── buildEngineSchema() — 序列化 C 端 Schema（精简）
   │   ├── buildPageContent() — 生成 HTML 外壳、CSS、meta
   │   └── POST /api/activity/save { meta, schema, content }
   └── 点击发布 → POST /api/activity/publish

2. 主后端（jimu-server）
   ├── Save: 存储 Schema 到 ActivitySchema 表 → 创建版本记录
   └── Publish:
       ├── updateOnlineVersionTag() — 新版本标记 ONLINE，旧版本标记 OFFLINE
       ├── ActivityRenderer.renderFormal()
       │   ├── 构建 window.JIMU 全局对象（activityId/context/scripts/initialElements）
       │   ├── 从 activity.tpl 模板生成 HTML
       │   │   ├── <head>: preloads、meta、内联 CSS
       │   │   ├── <body>: loading 屏 + #app
       │   │   ├── <script id="jimu-schema">: 首屏 Schema JSON
       │   │   ├── <script id="jimu-next-schema">: 懒加载 Schema JSON
       │   │   ├── 组件脚本（type="text/plain" 延迟激活）
       │   │   └── 引擎脚本（defer）
       │   ├── HTML 压缩（html-minifier-terser）
       │   └── CDN 保护脚本注入
       └── 存储 HTML 到 Blob Storage（Brotli 压缩）

3. 出页服务（page-service）
   ├── GET /doodle/:token
   ├── Redis 缓存 → 请求去重 → BlobStore → 降级回源
   └── 返回 HTML（ETag/Last-Modified/Content-Encoding）

4. 渲染引擎（C 端）
   ├── 浏览器加载 HTML
   ├── Polyfill → 引擎脚本 → Context 脚本激活
   ├── 组件脚本双阶段加载（首屏 → 懒加载）
   ├── 引擎读取 #jimu-schema → importElements → registerElementSchema
   ├── createApp(Engine).mount('#app')
   ├── Wrapper 递归渲染组件树
   └── 组件 view.vue: useJimuProps/useJimuContext/useJimuEvent
```

### 8.2 版本管理

- 每次保存创建版本记录
- 版本标签：ONLINE / OFFLINE
- 支持 SSE 流式版本 diff
- 支持版本回退

### 8.3 定时发布

- `ScheduleTaskModel` 存储定时发布任务
- `node-schedule` 定时触发

---

## 九、组件开发工具链

`@jimu/cli-next` 提供完整的组件开发生命周期：

| 命令 | 用途 |
|------|------|
| `create <name>` | 脚手架创建标准组件结构 |
| `serve <name>` | 本地开发服务器（HMR + WebSocket 连接编辑器） |
| `build <name>` | 生产构建 |
| `publish <name>` | 发布到组件市场（上传 Blob + 注册版本） |
| `push/pull <name>` | 推送/拉取组件源码 |
| `clone <name>` | 克隆已有组件 |
| `gen-dts/gen-meta` | 生成类型声明和元数据 |

---

## 十、Page Editing Core — Headless 编辑会话（E0 新架构）

`packages/client/page-editing-core` 是为 AI 集成和程序化编辑设计的无 UI 编辑会话：

```
createPageEditingSession()
  ├── 元素 Map 管理
  ├── 选中状态管理
  ├── 画布挂载控制（PageCanvasController）
  ├── 原子操作执行器（ComponentOperationExecutor: add/move/delete）
  ├── 页面校验器（PageValidator）
  └── 保存载荷构建
```

**设计思路**：将编辑器核心能力从 Vue UI 中解耦，使得 AI Agent 可以通过程序化 API 操作页面，无需模拟用户交互。这是 AI Agent 框架与低代码平台之间的桥梁。

---

## 十一、关键设计模式总结

| 模式 | 应用 |
|------|------|
| **双 Schema 架构** | 编辑器和引擎共享类型定义但各自独立实现，B 端/C 端代码隔离 |
| **属性值多类型系统** | 静态/引用/计算/继承 4 种值类型，实现灵活的组件间数据绑定 |
| **双阶段渲染** | 首屏组件优先加载，懒加载组件延迟激活，优化 FCP |
| **Context 桥接** | 平台差异通过统一接口抹平，一套 Schema 多平台运行 |
| **事务 + Diff 历史** | 操作原子化 + diff-based 撤销重做，低内存高细粒度 |
| **依赖注入（initExports）** | 共享包通过模块系统注入 Vue 实例，解耦包间依赖 |
| **Headless 编辑会话** | 编辑核心能力与 UI 解耦，支持 AI 程序化操作 |
| **多级缓存出页** | Redis → 请求去重 → BlobStore → 降级回源，高并发保障 |
| **组件市场模式** | 组件版本化管理 + Blob 存储 + CDN 分发 + CLI 工具链 |

---

## 十二、核心设计哲学

1. **B/C 端隔离**：编辑器和引擎共享类型但独立实现，C 端不携带 B 端冗余代码
2. **Schema 驱动**：一切皆 Schema，页面 = 组件树 + 属性 + 事件 + 依赖清单，可序列化、可恢复
3. **组件即合约**：config.ts（B 端定义）和 view.vue（C 端渲染）通过共享类型契约协作，平台提供 define/use 双侧 API
4. **性能优先**：双阶段加载、HTML 压缩、Brotli、CDN、多级缓存、请求去重
5. **扩展性**：组件市场 + CLI 工具链 + Skill 系统，新组件可独立开发发布
6. **Headless 化**：编辑核心能力与 UI 解耦，为 AI 集成和程序化编辑铺路

---

# 附：简历项目描述参考（低代码平台部分）

> 面向 AI 全栈工程师岗位，建议将低代码平台与 AI Agent 框架作为同一项目的两个维度呈现，突出"平台 + AI"的复合能力。

## 项目名称：快手商业化活动搭建平台（积木 V5）— 低代码可视化搭建 + AI 辅助

**项目背景**：服务于快手商业化运营团队的 H5 活动页面搭建流程，支撑日均千级页面的创建与发布。平台提供可视化拖拽编辑器、组件市场生态、多平台渲染引擎、一键发布部署的完整链路，并集成 AI Agent 实现自然语言搭建。

**技术栈**：Vue 3 / Pinia / NestJS / Egg.js / TypeScript / pnpm Monorepo / MySQL / Redis / CDN / Blob Storage

### 核心工作与技术亮点

**1. 可视化编辑器架构设计与实现**

- 基于 Vue 3 + Pinia 构建 IDE 式可视化编辑器，包含组件市场、画布、属性面板、组件树、代码编辑器等模块，支撑 100+ 积木组件的拖拽编排与属性配置
- 设计基于响应式 Map + Pinia 的混合状态管理方案：元素数据使用 `reactive(Map)` 实现细粒度响应式追踪，宏观状态使用 Pinia Store 管理，兼顾性能与开发体验
- 实现事务 + Diff-based 撤销重做系统，将多步操作封装为原子事务，通过元素 diff 而非快照实现撤销，内存占用降低 80%+
- 基于 CRDT（Koko）实现多人实时协作编辑，支持多用户同时操作同一活动页面

**2. C 端渲染引擎设计与首屏性能优化**

- 设计并实现 UMD 渲染引擎，将 JSON Schema 递归渲染为可交互 Vue 页面，支持 100+ 组件的动态注册与渲染
- 实现双阶段加载策略：首屏组件优先激活（视口检测），懒加载组件按批次（每批 10 个）延迟加载，首屏 FCP 降低约 50%
- 设计平台上下文桥接模式，通过统一 `IContext` 接口抹平快手/快手Pro/快影三个平台差异，一套 Schema 多平台运行
- 实现 Wrapper 容器组件统一处理样式生成、曝光埋点、热力图采集、自定义代码注入、动画控制等横切关注点

**3. 积木组件生态设计与工具链**

- 设计 `defineJimuProps` / `useJimuProps` 双侧 API 体系：B 端通过 `define*` 声明组件属性/样式/行为/事件，C 端通过 `use*` 组合式 API 获取响应式数据，通过类型契约协作
- 设计 4 种属性值类型（静态/引用/计算/继承），实现组件间响应式数据绑定、自定义逻辑、主题参数透传
- 实现组件依赖系统（强依赖/引用依赖/事件依赖），支撑级联删除、事件链清理等复杂场景
- 开发 `@jimu/cli-next` CLI 工具链，提供创建/开发/构建/发布/克隆的组件全生命周期管理

**4. 后端服务架构与发布部署**

- 设计编辑后端（Egg.js）+ 出页服务（NestJS）分离架构，出页服务独立扩缩容
- 实现多级缓存出页策略：Redis → 请求去重 → BlobStore → 降级回源，支撑高并发页面分发
- 实现 HTML 页面生成器：组件脚本排序、双 Schema 嵌入、延迟激活、HTML 压缩、CDN 保护，生成产物 Brotli 压缩后存储
- 设计版本管理系统：ONLINE/OFFLINE 标签、版本 diff、版本回退、定时发布

**5. Headless 编辑核心（为 AI 集成铺路）**

- 将编辑器核心能力从 Vue UI 中解耦，设计 Headless 页面编辑会话（`page-editing-core`），提供程序化 API（元素管理、选中、操作执行、校验、保存）
- 通过 `ComponentOperationExecutor` 实现原子操作（add/move/delete），为 AI Agent 的页面操作提供底层接口
- 该架构使 AI Agent 无需模拟用户交互即可直接操作页面 Schema，是 AI Agent 框架与低代码平台之间的桥梁

### 业务价值

- 运营人员无需写代码即可搭建 H5 活动页面，单页面创建时间从小时级降至分钟级
- 支撑日均千级活动页面的创建与发布，100+ 积木组件覆盖抽奖/排名/任务等核心业务场景
- 组件市场 + CLI 工具链支持组件独立开发发布，新组件上线周期从周级降至天级
- 首屏双阶段加载优化使页面 FCP 降低约 50%，多级缓存出页保障高并发稳定性

---

### 简历使用建议

1. **与 AI Agent 项目的关系**：建议将低代码平台和 AI Agent 框架作为同一个大项目的两个维度。低代码平台是"基建"，AI Agent 是"上层应用"。面试时强调你同时理解基建和应用层，这是 AI 全栈工程师的核心竞争力。

2. **按目标岗位侧重裁剪**：
   - **偏前端架构**：重点突出编辑器状态管理、渲染引擎、双阶段加载、组件 API 设计、Monorepo 架构
   - **偏全栈**：重点突出前后端分离、多级缓存、发布部署流程、版本管理、Headless 编辑核心
   - **偏 AI 应用**：重点突出 Headless 编辑核心如何为 AI 铺路、组件 Schema 如何成为 AI 操作对象、低代码 + AI 的结合点

3. **量化数据按实际补充**：日均页面数、组件数量、FCP 优化百分比、内存降低比例、协作人数等

4. **面试延伸准备**：
   - "双 Schema 架构（Editor/Engine）如何保证数据一致性？"
   - "diff-based 撤销重做与 snapshot-based 相比的优劣？"
   - "双阶段加载如何决定哪些组件属于首屏？threshold 如何计算？"
   - "属性值类型系统（静态/引用/计算/继承）的运行时解析流程是什么？"
   - "Headless 编辑核心如何解耦 UI 与逻辑？AI Agent 是如何调用这些 API 的？"
   - "多级缓存出页的缓存失效策略是什么？发布后如何保证用户立即看到新版本？"
   - "组件依赖系统在删除/复制/粘贴时如何保证一致性？"
