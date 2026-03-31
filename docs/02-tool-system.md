# 02 - Tool 工具系统

> Claude Code 最核心的能力载体——30+ 工具的定义、注册、权限、执行全链路

---

## 一、设计哲学

Tool 系统是 Claude Code 的 **能力骨架**。模型（Claude）本身只能生成文本，而通过 Tool 系统，它可以：
- 读写文件、执行命令、搜索代码
- 派生子 Agent、调用 MCP 服务器
- 与用户交互、管理任务

核心设计原则：
1. **失败关闭** — 默认不安全、默认非只读、默认不可并发
2. **三层校验** — 输入验证 → 权限检查 → 执行
3. **延迟加载** — 大体积工具按需发现，节省 Token

---

## 二、Tool 接口定义

**文件**: `src/Tool.ts`

每个 Tool 是一个实现了以下接口的对象：

```typescript
interface Tool<Input, Output> {
  // ── 身份 ──
  name: string                    // 唯一标识符（如 "Bash"）
  aliases?: string[]              // 别名（向后兼容）
  searchHint?: string             // ToolSearch 关键词提示（3-10 词）

  // ── Schema ──
  inputSchema: ZodSchema<Input>   // Zod 输入验证
  outputSchema?: ZodSchema<Output>// Zod 输出验证（可选）
  maxResultSizeChars: number      // 超过此大小 → 持久化到磁盘

  // ── 行为标记 ──
  isConcurrencySafe(): boolean    // 是否可并发执行（默认 false）
  isReadOnly(): boolean           // 是否只读（默认 false）
  isDestructive(): boolean        // 是否破坏性（默认 false）
  shouldDefer?: boolean           // 是否延迟加载（ToolSearch 按需）

  // ── 三层生命周期 ──
  validateInput?(input, context)   // 第一层：输入校验
  checkPermissions(input, context) // 第二层：权限检查
  call(input, context, ...)        // 第三层：实际执行

  // ── 结果映射 ──
  mapToolResultToToolResultBlockParam(output, toolUseID)
    // 将工具输出转为 API 兼容格式

  // ── UI 渲染 ──
  renderToolUseMessage?(input)              // 工具调用中显示
  renderToolUseProgressMessage?(progress)   // 执行进度显示
  renderToolResultMessage?(output)          // 结果显示
  renderToolUseRejectedMessage?(input)      // 被拒绝显示

  // ── 高级功能 ──
  toAutoClassifierInput?(input): string     // 安全分类器输入
  backfillObservableInput?(input): void     // 观察者可见输入修正
  inputsEquivalent?(a, b): boolean          // 语义等价判断
  interruptBehavior?(): 'cancel' | 'block'  // 中断行为
}
```

### buildTool() 工厂函数

为避免每个工具都要实现所有方法，`buildTool()` 提供安全默认值：

```typescript
const TOOL_DEFAULTS = {
  isEnabled:          () => true,
  isConcurrencySafe:  () => false,  // 🔒 默认不安全
  isReadOnly:         () => false,  // 🔒 默认非只读
  isDestructive:      () => false,
  checkPermissions:   () => ({ behavior: 'allow' }),  // 委托给通用系统
  toAutoClassifierInput: () => '',  // 跳过分类器
}
```

> **设计洞察**: 默认 `isConcurrencySafe: false` 意味着新增工具如果忘记设置此标记，会被串行执行而非并发执行——这是一个安全的默认行为。

---

## 三、Tool 注册与发现

**文件**: `src/tools.ts`

### 工具池组装流程

```
getAllBaseTools()
  │
  ├─ 条件导入 50+ 工具（基于 feature flag、环境变量）
  │
  ▼
getTools(permissionContext)
  │
  ├─ filterToolsByDenyRules() — 移除被 blanket deny 的工具
  │
  ▼
assembleToolPool(permissionContext, mcpTools)
  │
  ├─ 合并内置工具 + MCP 工具
  ├─ 按名称去重（内置工具优先）
  └─ 按名称排序（Prompt Cache 稳定性）
```

### 简单模式

当设置 `CLAUDE_CODE_SIMPLE=1` 时，只加载 3 个核心工具：

```typescript
// 简单模式工具集
[BashTool, FileReadTool, FileEditTool]
```

### 延迟工具（Deferred Tools）

大体积工具通过 `shouldDefer: true` 标记延迟加载：

```
初始 Prompt（~30 个工具定义）
  │
  │  模型发现需要 WebFetch...
  │
  ▼
ToolSearch("web fetch")
  │
  ├─ 关键词匹配/精确匹配
  ├─ 返回工具引用: { type: 'tool_reference', tool_name: 'WebFetch' }
  └─ 下一次 API 调用中包含完整 WebFetch 定义
```

> **为什么延迟?** WebFetch 和 WebSearch 的 Schema 描述很大（几千 Token），全部放在初始 Prompt 中会浪费缓存空间。延迟加载只在需要时付出成本。

---

## 四、Tool 执行生命周期

```
模型返回 tool_use 块
  │
  ▼
┌──────────────────────────────────────────────┐
│ 1. validateInput(input, context)              │
│    ├─ 输入格式校验                            │
│    ├─ 文件存在性检查                          │
│    ├─ 大小限制检查                            │
│    ├─ 过期检测（文件自上次读取后是否被修改）    │
│    └─ 返回: { result: true } 或              │
│            { result: false, message, errorCode }
└────────────────┬─────────────────────────────┘
                 │ 通过
                 ▼
┌──────────────────────────────────────────────┐
│ 2. checkPermissions(input, context)           │
│    ├─ 规则匹配（allow/deny/ask 列表）         │
│    ├─ 工具特定逻辑（如域名白名单）            │
│    └─ 返回: PermissionResult                  │
│         { behavior: 'allow'|'deny'|'ask' }   │
└────────────────┬─────────────────────────────┘
                 │ allow
                 ▼
┌──────────────────────────────────────────────┐
│ 3. call(input, context, canUseTool, msg, onProgress)
│    ├─ 实际执行工具逻辑                        │
│    ├─ 可通过 onProgress 报告进度              │
│    └─ 返回: ToolResult<Output>               │
│         { data, newMessages?, contextModifier? }
└────────────────┬─────────────────────────────┘
                 │
                 ▼
┌──────────────────────────────────────────────┐
│ 4. mapToolResultToToolResultBlockParam()      │
│    ├─ 将 Output 转为 API ToolResultBlockParam │
│    ├─ 如果结果 > maxResultSizeChars → 持久化  │
│    └─ 返回文件路径而非完整内容                │
└──────────────────────────────────────────────┘
```

---

## 五、核心工具实现详解

### 5.1 BashTool — Shell 命令执行

**文件**: `src/tools/BashTool/` (18 个文件)

```
maxResultSizeChars: 30,000
strict: true
searchHint: 'execute shell commands'
```

**执行流程**:
```
BashTool.call(input)
  │
  ├─ 命令语法验证
  ├─ shouldUseSandbox() → 是否需要沙箱
  │    ├─ 检查 dangerouslyDisableSandbox 标记
  │    ├─ 检查用户配置的排除命令
  │    └─ 固定点算法剥离 env/timeout/sudo 包装
  ├─ 沙箱模式: sandbox.exec(command)
  ├─ 非沙箱模式: child_process.exec(command)
  ├─ Git 操作追踪
  │    ├─ 正则检测: git commit / git push / gh pr create
  │    ├─ 提取元数据: SHA / Branch / PR URL
  │    └─ OTLP 计数器 + Analytics 事件
  └─ 返回 stdout/stderr
```

**智能行为检测**:
```typescript
isReadOnly(input) {
  // 分析命令，判断是否只读
  // find, grep, cat, ls, git log, git status → true
  // rm, mv, git push → false
}

isDestructive(input) {
  // rm, mv, truncate, > (重定向) → true
}
```

### 5.2 FileEditTool — 文件编辑（字符串替换）

**文件**: `src/tools/FileEditTool/`

```typescript
// 输入 Schema
{
  file_path: string,     // 绝对路径
  old_string: string,    // 要替换的子串
  new_string: string,    // 替换后的内容
  replace_all?: boolean  // 是否替换所有匹配
}
```

**核心机制 — 过期检测**:

```
validateInput()
  │
  ├─ 读取文件内容 + 时间戳
  ├─ 与 readFileState 缓存对比
  │    ├─ 时间戳匹配? → 未修改
  │    ├─ 时间戳不匹配?
  │    │    └─ 内容匹配? → 未修改（Windows 时间戳精度问题）
  │    │    └─ 内容不匹配? → ⚠️ 文件已被修改，拒绝编辑
  │    └─ 从未读取过? → ⚠️ 必须先 Read 才能 Edit
  └─ 查找 old_string
       ├─ findActualString() — 带引号归一化
       ├─ 弯引号 (' ") → 直引号 (' ")
       └─ 多个匹配且 replace_all=false → 拒绝
```

> **为什么需要过期检测?** 防止模型在不知道文件已被（用户或其他工具）修改的情况下覆盖更改。这是一种乐观并发控制。

**引号归一化**:
```
模型输出: "don't" → 搜索时归一化为 "don't"
文件内容: "don't" → 两者匹配
替换结果: 保留原始文件的引号风格
```

**原子性保证**:
```
⚠️ 在过期检查和写入之间没有 await
→ 防止异步操作在检查和写入之间修改文件
→ 同步的 check-then-write
```

### 5.3 AgentTool — 子 Agent 派生

**文件**: `src/tools/AgentTool/AgentTool.tsx` (~3900 行)

这是最复杂的工具之一，负责创建独立的子 Agent：

```typescript
// 输入 Schema
{
  agent_type: string,    // 'general-purpose' | 'Explore' | 'Plan' | ...
  prompt: string,        // Agent 的任务描述
  timeout_seconds?: number  // 最大 600 秒
}
```

**内置 Agent 类型**:
| 类型 | 用途 |
|------|------|
| `generalPurposeAgent` | 通用多任务 Agent |
| `exploreAgent` | 代码库探索专家 |
| `planAgent` | 规划与架构设计 |
| `verificationAgent` | 方案验证 |
| `claudeCodeGuideAgent` | Claude Code 使用指南 |

**执行模式**:
- `runAgent()` — 标准执行（阻塞直到完成）
- `forkSubagent()` — 从父上下文分叉，共享 Prompt Cache
- `resumeAgentBackground()` — 后台恢复之前的会话

### 5.4 ToolSearchTool — 延迟工具发现

**文件**: `src/tools/ToolSearchTool/`

**两种搜索模式**:

```
1. 精确选择: "select:WebFetch,WebSearch"
   → 直接按名称加载
   → 逗号分隔多选
   → 快速路径

2. 关键词搜索: "fetch web page content"
   → 工具名拆分: FileEditTool → ['file', 'edit', 'tool']
   → MCP 工具拆分: mcp__server__action → ['server', 'action']
   → 评分:
     - 完整部分匹配: 10 分（MCP: 12 分）
     - 子串匹配: 5 分（MCP: 6 分）
     - searchHint 匹配: 4 分
     - 描述匹配: 2 分
   → 返回 Top N（默认 5）
```

### 5.5 GrepTool & GlobTool

**GrepTool**: 基于 ripgrep 的内容搜索
```
特性:
├─ 完整正则语法
├─ 输出模式: content / files_with_matches / count
├─ 分页: head_limit + offset
├─ 排除 VCS 目录: .git, .svn, .hg, .bzr, .jj, .sl
├─ 尊重权限文件忽略模式
└─ isConcurrencySafe: true, isReadOnly: true
```

**GlobTool**: 文件名模式匹配
```
特性:
├─ 通配符和 glob 模式
├─ 默认限制: 100 个结果
├─ 相对路径转换（节省 Token）
└─ isConcurrencySafe: true, isReadOnly: true
```

### 5.6 WebFetchTool & WebSearchTool

```
WebFetchTool:
├─ shouldDefer: true（延迟加载）
├─ 预批准主机免权限提示
├─ 重定向处理（返回包含重定向 URL 的特殊消息）
├─ 二进制内容（PDF）持久化到磁盘
├─ maxResultSizeChars: 100,000
└─ 支持 LLM 提示处理内容

WebSearchTool:
├─ shouldDefer: true
├─ 使用原生 API 搜索（server_tool_use 机制）
├─ 进度追踪: 查询更新 + 结果计数
├─ maxResultSizeChars: 100,000
└─ 支持域名白名单/黑名单（二选一）
```

---

## 六、ToolUseContext — 执行环境

每个工具调用都收到一个丰富的上下文对象：

```typescript
interface ToolUseContext {
  options: {
    commands: Command[]              // 可用命令
    tools: Tools                      // 可用工具
    mcpClients: MCPServerConnection[] // MCP 连接
    mcpResources: Record<string, ServerResource[]>
    agentDefinitions: AgentDefinitionsResult
    mainLoopModel: string             // 当前模型
    maxBudgetUsd?: number             // 预算上限
  }

  abortController: AbortController    // 取消控制器
  readFileState: FileStateCache       // 文件读取状态缓存
  messages: Message[]                 // 当前对话消息

  getAppState(): AppState             // 全局应用状态
  setAppState(f: (prev) => AppState)  // 更新状态

  // UI 回调
  setToolJSX(): void                  // 更新执行中 UI
  addNotification(): void             // 触发通知
  sendOSNotification(): void          // 系统通知

  // 限制
  fileReadingLimits?: { maxTokens?, maxSizeBytes? }
  globLimits?: { maxResults? }
}
```

### 文件状态缓存

```typescript
// readFileState 追踪每个文件的读取状态
type FileStateCache = Map<string, {
  content: string           // 文件内容
  timestamp: number         // 读取时间戳
  offset?: number           // 部分读取偏移
  limit?: number            // 部分读取行数
  isPartialView?: boolean   // 是否为部分视图
}>
```

这个缓存是 FileEditTool 过期检测的基础——它知道模型上次读到的文件内容是什么。

---

## 七、结果处理与持久化

### 大结果持久化

```
工具输出
  │
  ├─ 长度 ≤ maxResultSizeChars?
  │    └─ 直接作为文本返回给模型
  │
  └─ 长度 > maxResultSizeChars?
       ├─ 写入 /tmp/ 临时文件
       └─ 返回文件路径给模型
           "结果已保存到 /tmp/claude-xxx.txt"
```

| 工具 | maxResultSizeChars | 说明 |
|------|-------------------|------|
| BashTool | 30,000 | 命令输出可能很长 |
| FileReadTool | Infinity | 自身通过 limit 控制 |
| GrepTool | 30,000 | 搜索结果 |
| WebFetchTool | 100,000 | 网页内容 |
| WebSearchTool | 100,000 | 搜索结果 |

---

## 八、工具一览表

| 工具 | 类型 | 并发安全 | 只读 | 延迟 | 用途 |
|------|------|---------|------|------|------|
| Bash | 执行 | ❌ | 分析 | ❌ | Shell 命令 |
| FileRead/Read | 文件 | ✅ | ✅ | ❌ | 读取文件 |
| FileEdit/Edit | 文件 | ❌ | ❌ | ❌ | 字符串替换编辑 |
| FileWrite/Write | 文件 | ❌ | ❌ | ❌ | 创建/覆盖文件 |
| Grep | 搜索 | ✅ | ✅ | ❌ | 内容搜索 |
| Glob | 搜索 | ✅ | ✅ | ❌ | 文件名搜索 |
| Agent | 调度 | ❌ | ❌ | ❌ | 派生子 Agent |
| SendMessage | 通信 | ❌ | ❌ | ❌ | 向 Agent 发消息 |
| WebFetch | 网络 | ✅ | ✅ | ✅ | 抓取网页 |
| WebSearch | 网络 | ✅ | ✅ | ✅ | 网络搜索 |
| ToolSearch | 系统 | ✅ | ✅ | ❌ | 发现延迟工具 |
| Skill | 扩展 | ❌ | ❌ | ❌ | 执行技能 |
| MCPTool | 扩展 | ❌ | ❌ | ❌ | MCP 工具代理 |
| TaskCreate | 任务 | ❌ | ❌ | ✅ | 创建任务 |
| TaskUpdate | 任务 | ❌ | ❌ | ✅ | 更新任务 |
| AskUserQuestion | 交互 | ❌ | ✅ | ✅ | 向用户提问 |
| NotebookEdit | 文件 | ❌ | ❌ | ✅ | Jupyter 编辑 |
| EnterPlanMode | 模式 | ❌ | ✅ | ✅ | 进入计划模式 |
| ExitPlanMode | 模式 | ❌ | ✅ | ✅ | 退出计划模式 |
| TodoWrite | 任务 | ❌ | ❌ | ✅ | 待办事项 |

---

## 九、关键设计模式总结

### 1. 三层校验（Defense in Depth）
```
validateInput  → 工具级校验（格式、存在性、大小）
checkPermissions → 权限系统校验（规则、Hook、分类器、用户）
call           → 实际执行（可能再次内部校验）
```

### 2. 文件状态追踪（Optimistic Concurrency）
```
Read → 缓存 {content, timestamp}
Edit → 对比缓存 → 匹配则写入 → 不匹配则拒绝
Write → 必须先 Read → 过期检测同上
```

### 3. 延迟工具池（Lazy Tool Pool）
```
初始: 30 个工具在 Prompt 中
运行时: 模型按需 ToolSearch → 动态加载
效果: 节省 ~数千 Token 的 Prompt 空间
```

### 4. 安全默认值（Fail-Closed）
```
新增工具忘记设置 isConcurrencySafe?  → 串行执行（安全）
新增工具忘记设置 isReadOnly?         → 需要权限（安全）
新增工具忘记设置 checkPermissions?   → 默认 allow（委托给通用系统）
```
