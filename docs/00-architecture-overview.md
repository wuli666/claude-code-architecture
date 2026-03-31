# Claude Code 源码架构总览

> 基于 `@anthropic-ai/claude-code@2.1.88` 源码分析
> 源码路径: `restored-src/src/`

---

## 一、项目定位

Claude Code 是 Anthropic 官方的 **AI 编程助手 CLI**。它不是一个简单的命令行工具，而是一个完整的 **终端原生 AI IDE**——集成了对话管理、工具调用、权限控制、插件系统、终端 UI 渲染等能力，支持在终端中完成代码阅读、编写、调试、Git 操作等全流程开发任务。

---

## 二、技术栈概览

| 层面 | 技术选型 | 说明 |
|------|---------|------|
| 运行时 | Node.js 18+ / Bun | 同时支持两种运行时 |
| 语言 | TypeScript | 编译产物为单文件 bundle（含 sourcemap） |
| UI 框架 | **Ink**（深度 fork） | 基于 React 的终端 UI 框架，大量定制 |
| 布局引擎 | Yoga (WASM) | Facebook 的 Flexbox 终端布局 |
| CLI 框架 | Commander.js | 命令行参数解析 |
| API 通信 | Anthropic SDK | 与 Claude API 交互 |
| Schema | Zod | 输入输出验证 |
| 包管理 | pnpm（推测） | 构建后单文件分发 |

---

## 三、源码目录结构全景

```
restored-src/src/
├── main.tsx                    # 🚀 主入口 — Commander.js 注册 + REPL 启动
├── setup.ts                    # ⚙️ 初始化 — CWD/Hooks/Worktree/Analytics
├── context.ts                  # 📋 上下文构建 — Git status / CLAUDE.md / 日期
├── query.ts                    # 🔄 核心对话循环 — compaction → API → tools → continue
├── QueryEngine.ts              # 🎯 SDK/无头模式查询引擎
├── Task.ts                     # 📦 Task 类型定义与 ID 生成
├── tasks.ts                    # 📦 Task 注册表
├── Tool.ts                     # 🔧 Tool 基类/接口定义
├── tools.ts                    # 🔧 Tool 注册表与组装
├── commands.ts                 # ⌨️ Slash 命令注册表
├── cost-tracker.ts             # 💰 成本追踪
├── history.ts                  # 📖 会话历史持久化
├── ink.ts                      # 🎨 Ink 渲染入口（带主题）
│
├── entrypoints/                # 入口点
│   ├── cli.tsx                 #   CLI Bootstrap（快速路径分发）
│   ├── init.ts                 #   全局初始化（memoized，只运行一次）
│   └── sdk/                    #   SDK 入口点
│
├── bootstrap/                  # 启动态
│   └── state.ts                #   全局状态（SessionID、Model、Client Type）
│
├── cli/                        # CLI 通信层
│   ├── structuredIO.ts         #   结构化 stdio 读写
│   ├── remoteIO.ts             #   远程会话 IO
│   └── transports/             #   传输层（WebSocket / SSE / Hybrid）
│
├── tools/                      # 🔧 所有工具实现
│   ├── BashTool/               #   Shell 命令执行
│   ├── FileEditTool/           #   文件编辑（字符串替换）
│   ├── FileReadTool/           #   文件读取
│   ├── FileWriteTool/          #   文件写入
│   ├── GrepTool/               #   内容搜索（ripgrep）
│   ├── GlobTool/               #   文件名搜索
│   ├── AgentTool/              #   子 Agent 派生
│   ├── WebFetchTool/           #   网页抓取
│   ├── WebSearchTool/          #   网络搜索
│   ├── ToolSearchTool/         #   延迟工具发现
│   ├── SkillTool/              #   技能执行
│   ├── MCPTool/                #   MCP 工具代理
│   └── ...                     #   更多（30+ 工具）
│
├── commands/                   # ⌨️ Slash 命令实现
│   ├── help/                   #   /help
│   ├── model/                  #   /model
│   ├── compact/                #   /compact
│   ├── diff/                   #   /diff
│   ├── permissions/            #   /permissions
│   └── ...                     #   更多（80+ 命令）
│
├── services/                   # 🔌 核心服务
│   ├── api/                    #   Claude API 封装 + 重试逻辑
│   ├── compact/                #   对话压缩（Auto/Micro/Reactive）
│   ├── mcp/                    #   MCP 服务器管理
│   ├── plugins/                #   插件安装与管理
│   ├── oauth/                  #   OAuth 认证
│   ├── lsp/                    #   LSP 语言服务器
│   ├── analytics/              #   分析与事件上报
│   ├── SessionMemory/          #   会话记忆提取
│   ├── extractMemories/        #   自动记忆提取
│   └── ...                     #   更多服务
│
├── components/                 # 🎨 React/Ink UI 组件
│   ├── PromptInput/            #   输入框
│   ├── messages/               #   消息渲染（用户/助手/系统）
│   ├── permissions/            #   权限确认对话框
│   ├── diff/                   #   Diff 查看器
│   ├── skills/                 #   技能 UI
│   ├── mcp/                    #   MCP 管理 UI
│   └── ...                     #   更多组件
│
├── screens/                    # 📺 屏幕/页面
│   └── REPL.tsx                #   主 REPL 屏幕（~875KB 超大文件）
│
├── state/                      # 📊 状态管理
│   ├── store.ts                #   Pub/Sub Store
│   ├── AppStateStore.ts        #   AppState 类型定义（450+ 行）
│   └── selectors.ts            #   状态选择器
│
├── hooks/                      # 🪝 Hooks 执行引擎
│   ├── toolPermission/         #   工具权限处理
│   │   ├── PermissionContext.ts   权限上下文
│   │   └── handlers/           #   Interactive/Coordinator/Swarm 处理器
│   └── ...
│
├── ink/                        # 🎨 定制 Ink 框架
│   ├── ink.tsx                 #   主事件循环 & 终端管理
│   ├── reconciler.ts           #   React 协调器
│   ├── renderer.ts             #   Yoga → 屏幕缓冲区
│   ├── dom.ts                  #   DOM 节点 & 滚动
│   ├── focus.ts                #   焦点管理
│   ├── components/             #   Box, Text, Button, Link...
│   ├── events/                 #   键盘/鼠标事件解析
│   └── termio/                 #   CSI/DEC/OSC 终端控制码
│
├── keybindings/                # ⌨️ 快捷键系统
│   ├── resolver.ts             #   按键解析 & 和弦匹配
│   ├── defaultBindings.ts      #   默认绑定
│   └── ...
│
├── utils/                      # 🛠 工具函数
│   ├── permissions/            #   权限引擎（规则匹配、模式切换）
│   ├── settings/               #   设置加载（User/Project/MDM）
│   ├── git/                    #   Git 集成（无子进程读取）
│   ├── model/                  #   模型选择 & 配置
│   ├── sandbox/                #   沙箱适配器
│   ├── memory/                 #   记忆类型定义
│   ├── messages/               #   消息格式转换
│   ├── telemetry/              #   OpenTelemetry 遥测
│   ├── mcp/                    #   MCP 工具函数
│   ├── skills/                 #   技能变更检测
│   ├── bash/                   #   Bash 命令分析
│   ├── hooks/                  #   Hook 工具函数
│   └── ...
│
├── skills/                     # 🎯 技能定义
│   ├── bundledSkills.ts        #   内置技能注册
│   ├── loadSkillsDir.ts        #   文件系统技能加载
│   └── mcpSkillBuilders.ts     #   MCP 技能构建器
│
├── plugins/                    # 🧩 插件定义
│   └── bundled/                #   内置插件
│
├── coordinator/                # 🎪 多 Agent 协调器
│   └── coordinatorMode.ts      #   Coordinator 模式
│
├── tasks/                      # 📋 后台任务类型
│   ├── DreamTask/              #   记忆整合任务
│   ├── LocalAgentTask/         #   本地 Agent 任务
│   ├── LocalShellTask/         #   本地 Shell 任务
│   └── RemoteAgentTask/        #   远程 Agent 任务
│
├── types/                      # 📝 核心类型定义
│   ├── command.ts              #   Command 类型
│   ├── hooks.ts                #   Hook 类型
│   ├── plugin.ts               #   Plugin 类型
│   └── generated/              #   Protobuf 生成类型
│
├── remote/                     # 🌐 远程会话
├── server/                     # 🖥 MCP Server 模式
├── voice/                      # 🎤 语音模式
├── vim/                        # ⌨️ Vim 模式
└── outputStyles/               # 🎨 输出风格
```

---

## 四、核心数据流

```
用户输入
  │
  ▼
┌─────────────────────────────────────────────────────┐
│  PromptInput（终端输入组件）                          │
│  ├─ 历史记录补全 / Vim 模式 / 图片粘贴               │
│  └─ Slash 命令识别 → processUserInput()              │
└────────────────────┬────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────┐
│  query() — 核心对话循环 (AsyncGenerator)             │
│                                                      │
│  ┌────────────────────────────────────────────┐      │
│  │ 1. Compaction 检查                          │      │
│  │    ├─ Microcompact（缓存编辑/时间触发）     │      │
│  │    ├─ Context Collapse（上下文折叠）        │      │
│  │    └─ Auto-compact（完整摘要）              │      │
│  └─────────────────┬──────────────────────────┘      │
│                    ▼                                  │
│  ┌────────────────────────────────────────────┐      │
│  │ 2. API 调用                                 │      │
│  │    ├─ System Prompt 构建                    │      │
│  │    ├─ withRetry() 重试包装                  │      │
│  │    ├─ 流式响应处理                          │      │
│  │    └─ Tool Use 块提取                       │      │
│  └─────────────────┬──────────────────────────┘      │
│                    ▼                                  │
│  ┌────────────────────────────────────────────┐      │
│  │ 3. Tool 执行                                │      │
│  │    ├─ validateInput() → 输入校验             │      │
│  │    ├─ checkPermissions() → 权限检查          │      │
│  │    │   ├─ 规则匹配（allow/deny/ask）        │      │
│  │    │   ├─ Hooks 执行                        │      │
│  │    │   ├─ 分类器（auto 模式）               │      │
│  │    │   └─ 用户交互确认                      │      │
│  │    └─ call() → 实际执行                     │      │
│  └─────────────────┬──────────────────────────┘      │
│                    ▼                                  │
│  ┌────────────────────────────────────────────┐      │
│  │ 4. 继续/终止判定                             │      │
│  │    ├─ 有 tool_use → 继续循环                │      │
│  │    ├─ stop_reason=stop → 终止               │      │
│  │    └─ 异常恢复 → compact/truncate/retry     │      │
│  └────────────────────────────────────────────┘      │
│                                                      │
└────────────────────┬────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────┐
│  REPL 渲染                                           │
│  ├─ 消息列表渲染（Markdown / Diff / Code）           │
│  ├─ 工具执行进度                                     │
│  ├─ 成本统计 & Token 使用                            │
│  └─ 状态栏 & 通知                                    │
└─────────────────────────────────────────────────────┘
```

---

## 五、七大核心子系统

### 1. 启动系统（Entrypoints + Bootstrap）

**职责**: 从 `node cli.js` 到 REPL 就绪的全部初始化工作

```
cli.tsx（快速路径分发）
  → main.tsx（Commander.js + 参数解析）
    → init.ts（全局初始化：TLS / OAuth / 代理 / 遥测）
      → setup.ts（CWD / Hooks / Worktree / 终端恢复）
        → REPL 启动
```

关键设计：**分层渐进初始化**——早期只做必要工作（版本号输出零依赖），后期才加载重型模块。

📖 详见 [01-startup-and-bootstrap.md](./01-startup-and-bootstrap.md)

---

### 2. Tool 工具系统

**职责**: 定义、注册、发现、权限校验、执行所有 AI 可调用的工具

核心三层架构：
```
Tool 定义（Tool.ts）       → 接口契约（输入 Schema / 权限 / 执行 / 结果映射）
Tool 注册表（tools.ts）    → 组装池（内置 + MCP + 延迟加载）
Tool 实现（tools/*/）      → 30+ 具体实现（Bash / Edit / Read / Agent...）
```

关键设计：**三层校验**（validateInput → checkPermissions → call），**延迟工具池**（ToolSearch 按需加载大体积工具）

📖 详见 [02-tool-system.md](./02-tool-system.md)

---

### 3. 对话循环与 Task 系统

**职责**: 管理与 Claude API 的交互循环、消息流、上下文压缩

```
QueryEngine（SDK 编排器）
  → query()（AsyncGenerator 对话循环）
    → compaction（Auto / Micro / Reactive 三级压缩）
    → withRetry()（智能重试 + 降级）
    → Coordinator（多 Worker Agent 编排）
```

关键设计：**生成器模式**驱动流式处理，**三级 Compaction** 防止上下文溢出

📖 详见 [03-conversation-and-task.md](./03-conversation-and-task.md)

---

### 4. 权限与安全系统

**职责**: 控制每个工具调用的审批流程，确保安全

```
权限模式（default / acceptEdits / plan / bypass / auto）
  → 规则匹配（allow / deny / ask 列表）
    → Hooks（用户自定义 Shell 拦截）
      → 分类器（AI 自动审批 — 仅内部使用）
        → 用户交互确认
```

关键设计：**多源规则合并**（MDM > User > Project > Local > CLI > Session），**沙箱隔离**

📖 详见 [04-permissions-and-security.md](./04-permissions-and-security.md)

---

### 5. 扩展系统（MCP / Plugins / Skills）

**职责**: 提供三种扩展机制，允许用户和企业扩展 Claude Code 能力

| 扩展类型 | 粒度 | 分发方式 | 能力 |
|---------|------|---------|------|
| MCP Server | 服务级 | 配置文件 | 工具 + 资源 + Prompt |
| Plugin | 包级 | Marketplace / Git | 命令 + Agent + 技能 + Hook + MCP |
| Skill | 文件级 | `.claude/skills/` | 可复用 Prompt 片段 |

📖 详见 [05-extension-system.md](./05-extension-system.md)

---

### 6. 终端 UI 与状态管理

**职责**: 在终端中渲染复杂的交互式 UI

```
React（声明式组件）
  → Ink Reconciler（自定义协调器）
    → Yoga Layout（Flexbox 计算）
      → Screen Buffer（字符网格）
        → ANSI Diff（增量输出）
          → Terminal（stdout 写入）
```

关键设计：**深度 fork 的 Ink**（支持鼠标、Alt-Screen、Kitty 键盘协议），**Pub/Sub Store** 状态管理

📖 详见 [06-terminal-ui-and-state.md](./06-terminal-ui-and-state.md)

---

### 7. 基础设施与服务

**职责**: 提供全局共享的基础能力

- **Git 集成**: 无子进程读取 `.git/` 目录，文件监视缓存
- **OAuth 认证**: PKCE 流程，Console + Claude.ai 双通道
- **LSP 集成**: 语言服务器管理，诊断与代码智能
- **Analytics**: 队列排空模式，Datadog + 1P 双通道
- **Memory**: 自动记忆提取，分叉 Agent 后台执行
- **Telemetry**: OpenTelemetry 全链路追踪

📖 详见 [07-infrastructure-and-services.md](./07-infrastructure-and-services.md)

---

## 六、关键架构模式

### 1. 渐进初始化（Staged Initialization）
启动分为 4 个阶段，每个阶段只做必要工作：
- **Phase 0**: 零导入快速路径（`--version` 直接输出）
- **Phase 1**: `init()` — 配置、TLS、代理、OAuth
- **Phase 2**: `setup()` — CWD、Hooks、Worktree、Analytics
- **Phase 3**: REPL 启动 — 工具加载、MCP 连接

### 2. 失败关闭默认（Fail-Closed Defaults）
```typescript
// Tool.ts — buildTool() 的默认值
isConcurrencySafe: () => false,  // 默认不安全 → 串行执行
isReadOnly: () => false,          // 默认非只读 → 需要权限
isDestructive: () => false,       // 默认非破坏性
```

### 3. 分叉 Agent 模式（Forked Agent Pattern）
Session Memory / Auto-Memory / Dream 等后台任务通过 **完美分叉** 主对话实现：
- 共享父级 Prompt Cache → 零额外缓存成本
- AsyncLocalStorage 隔离 → 不污染主线程状态
- 后台运行 → 不中断用户交互

### 4. 队列排空模式（Queue-and-Drain Pattern）
Analytics 系统避免导入循环：
- 事件产生时：如果 Sink 未附加 → 入队
- Sink 附加时：`queueMicrotask` 排空所有积压事件
- 保证零事件丢失

### 5. 文件监视缓存（File Watcher Cache）
Git 集成通过 `fs.watchFile` 监视 `.git/HEAD` 等文件：
- Dirty Flag 模式 → 变更时标脏
- 清除脏标记 → 异步重新计算 → 设置期间如果再次变更 → 重新标脏
- 避免竞态条件

### 6. 延迟加载（Deferred Loading）
大体积工具（WebFetch、WebSearch）通过 `shouldDefer: true` 标记：
- 不出现在初始 Prompt 中 → 节省 Token
- 模型需要时调用 `ToolSearch` → 按需加载
- 关键词/精确匹配 双路发现

---

## 七、代码规模统计

| 指标 | 数值 |
|------|------|
| 总文件数 | ~4,500 |
| 源码文件数（.ts/.tsx） | ~1,900 |
| 工具实现 | 30+ |
| Slash 命令 | 80+ |
| 目录深度 | 最多 7 层 |
| REPL.tsx 大小 | ~875 KB |
| AppState 类型定义 | 450+ 行 |

---

## 八、文档导航

| 文档 | 内容 |
|------|------|
| [01 - 启动流程](./01-startup-and-bootstrap.md) | 入口点 → Bootstrap → 初始化 → REPL 就绪 |
| [02 - Tool 系统](./02-tool-system.md) | 工具定义 → 注册 → 权限 → 执行 → 结果 |
| [03 - 对话循环](./03-conversation-and-task.md) | QueryEngine → 消息流 → Compaction → Coordinator |
| [04 - 权限安全](./04-permissions-and-security.md) | 权限模型 → Hooks → Sandbox → Settings |
| [05 - 扩展系统](./05-extension-system.md) | MCP → Plugins → Skills → DXT |
| [06 - 终端 UI](./06-terminal-ui-and-state.md) | Ink → 状态 → 输入 → 渲染 → 快捷键 |
| [07 - 基础设施](./07-infrastructure-and-services.md) | Git → OAuth → LSP → Analytics → Memory |
