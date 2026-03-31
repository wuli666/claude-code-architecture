# 07 - 基础设施与服务

> 支撑 Claude Code 运转的底层能力：从 Git 到 OAuth 到记忆系统

---

## 一、Git 集成

**文件**: `src/utils/git/`

### 1.1 无子进程 Git 读取

Claude Code 实现了直接读取 `.git/` 目录的轻量级 Git 解析器，**避免频繁 spawn `git` 子进程**：

**文件**: `src/utils/git/gitFilesystem.ts`

```
传统方式:
  child_process.exec('git branch --show-current')
  → fork 进程 → exec git → 读取 stdout → 解析
  → 延迟: ~20-50ms

Claude Code 方式:
  readFile('.git/HEAD')
  → 解析 "ref: refs/heads/main\n"
  → 返回 "main"
  → 延迟: ~1-2ms
```

**核心函数**:

| 函数 | 作用 | 替代的 git 命令 |
|------|------|----------------|
| `readGitHead(gitDir)` | 读取 HEAD（分支/detached） | `git rev-parse HEAD` |
| `resolveRef(gitDir, ref)` | 解析 ref 到 SHA | `git rev-parse ref` |
| `getCachedBranch()` | 获取当前分支 | `git branch --show-current` |
| `getCachedRemoteUrl()` | 获取 origin URL | `git remote get-url origin` |
| `getCachedDefaultBranch()` | 获取默认分支 | `git symbolic-ref refs/remotes/origin/HEAD` |
| `isShallowClone()` | 是否浅克隆 | `git rev-parse --is-shallow-repository` |
| `getWorktreeCountFromFs()` | worktree 数量 | `git worktree list` |

### 1.2 文件监视缓存

```typescript
// GitFileWatcher — 文件变更驱动的缓存
class GitFileWatcher {
  private dirty = false
  private cached: T | null = null

  watch('.git/HEAD')
  watch('.git/refs/heads/<branch>')
  watch('.git/config')

  // 当文件变更时:
  onFileChange() {
    this.dirty = true   // 标记脏
  }

  // 当读取缓存时:
  async get(): T {
    if (this.dirty || !this.cached) {
      this.dirty = false        // ① 清除脏标记（先清后算）
      this.cached = await compute()  // ② 异步计算
      // 如果在 ② 期间文件又变了 → dirty 重新变为 true
      // → 下次 get() 会重新计算
    }
    return this.cached
  }
}
```

> **竞态条件处理**: 先清除脏标记再异步计算，如果计算期间文件变化，脏标记会重新被设置。这是一种常见的乐观并发模式。

### 1.3 安全防护

```typescript
// 分支名安全检查
isSafeRefName(name: string): boolean
  ├─ 只允许: [a-zA-Z0-9/._+-@]
  ├─ 拒绝: .. (路径遍历)
  ├─ 拒绝: 以 - 开头（参数注入: git checkout --evil）
  ├─ 拒绝: shell 元字符（|;&$(){}...）
  └─ 应用于所有用户控制的 git ref

// SHA 验证
isValidGitSha(s: string): boolean
  ├─ 仅 40 字符十六进制（SHA-1）
  └─ 或 64 字符十六进制（SHA-256）
```

### 1.4 Git 配置解析器

**文件**: `src/utils/git/gitConfigParser.ts`

完全按照 Git 源码 `config.c` 实现的配置解析器：

```
parseGitConfigValue(gitDir, section, subsection, key)

支持:
  ├─ 节标题: [remote "origin"]
  ├─ 键值对: url = https://github.com/...
  ├─ 引号字符串: "hello \"world\""
  ├─ 转义序列: \n \t \\ \"
  ├─ 行内注释: key = value  # comment
  ├─ Section 大小写不敏感, Key 大小写不敏感
  └─ Subsection 大小写敏感
```

---

## 二、OAuth 认证

**文件**: `src/services/oauth/`

### 2.1 认证流程

Claude Code 使用标准 OAuth 2.0 + PKCE 流程：

```
用户执行 /login
  │
  ├─ 1. 生成 PKCE Code Verifier + Challenge
  │      code_challenge_method: 'S256'
  │
  ├─ 2. 构建授权 URL
  │      ├─ Console: console.anthropic.com/oauth/authorize
  │      └─ Claude.ai: claude.ai/oauth/authorize
  │
  ├─ 3. 打开浏览器 → 用户授权
  │
  ├─ 4. 本地 HTTP 服务器监听回调
  │      localhost:PORT/callback?code=xxx
  │
  ├─ 5. Code → Token 交换
  │      POST /oauth/token
  │      + code + code_verifier
  │
  └─ 6. 存储 Token
         ├─ Access Token → ~/.claude/credentials
         └─ Refresh Token → 系统 Keychain
```

### 2.2 双通道认证

| 通道 | URL | Scope | 用途 |
|------|-----|-------|------|
| Console | console.anthropic.com | ALL_OAUTH_SCOPES | API 调用 + 账户管理 |
| Claude.ai | claude.ai | CLAUDE_AI_INFERENCE_SCOPE | 推理调用 |

### 2.3 Profile 获取

```typescript
// API Key 模式
getOauthProfileFromApiKey()
  → GET /api/claude_cli_profile
  → Header: x-api-key: sk-xxx

// OAuth Token 模式
getOauthProfileFromOauthToken(accessToken)
  → GET /api/oauth/profile
  → Header: Authorization: Bearer xxx

// 返回
OAuthProfileResponse = {
  accountUuid, email, name,
  subscriptionType, billingType,
  rateLimitTier, ...
}
```

---

## 三、LSP 集成

**文件**: `src/services/lsp/`

### 3.1 什么是 LSP？

LSP（Language Server Protocol）让 Claude Code 可以利用语言服务器提供的 **代码智能**——语法检查、类型信息、引用查找等。

### 3.2 架构

```
LSPServerManager（管理器）
  │
  ├─ 维护: servers Map, extensionMap, openedFiles Map
  │
  ├─ initialize()
  │    └─ 从配置加载所有 LSP 服务器
  │
  ├─ getServerForFile(filePath)
  │    └─ 按文件扩展名路由到对应服务器
  │
  ├─ ensureServerStarted(filePath)
  │    └─ 按需启动服务器进程
  │
  └─ 文件操作通知
       ├─ openFile(path, content)  → didOpen
       ├─ changeFile(path, content) → didChange
       ├─ saveFile(path)           → didSave
       └─ closeFile(path)          → didClose
```

### 3.3 LSP Client

```typescript
LSPClient:
  ├─ start(command, args) → 通过 stdio 启动 LSP 进程
  ├─ initialize(params) → LSP 初始化握手
  ├─ sendRequest(method, params) → JSON-RPC 请求
  ├─ sendNotification(method, params) → JSON-RPC 通知
  ├─ onNotification(method, handler) → 注册通知处理
  └─ stop() → 关闭服务器

特性:
  ├─ 延迟处理器注册（连接就绪前排队）
  ├─ 崩溃检测 + onCrash 回调
  └─ Trace.Verbose 日志
```

### 3.4 与 FileEditTool 集成

当 FileEditTool 编辑文件时，会通知 LSP：

```
FileEditTool.call()
  ├─ 写入文件内容
  ├─ lspManager.changeFile(path, newContent)  → didChange
  └─ lspManager.saveFile(path)                → didSave

→ LSP 服务器重新分析 → 返回诊断信息
→ 可用于后续的代码质量反馈
```

---

## 四、Analytics 系统

**文件**: `src/services/analytics/`

### 4.1 架构：队列排空模式

```
事件产生                     Sink 附加
  │                            │
  ▼                            ▼
logEvent(name, data)     attachAnalyticsSink(sink)
  │                            │
  ├─ Sink 已附加?              ├─ 排空积压事件
  │   ├─ 是 → 直接发送         │    queueMicrotask(() => {
  │   └─ 否 → 入队 eventQueue  │      queue.forEach(e => sink.log(e))
  │                            │    })
  └─                           └─ 幂等（只排空一次）
```

> **为什么用队列排空?** Analytics 模块不能依赖任何其他模块（避免循环依赖），但事件可能在 Sink 附加前就产生了。队列确保零事件丢失。

### 4.2 事件路由

```
logEvent(name, metadata)
  │
  ├─ Datadog 路由（如果 gate 启用）
  │    └─ trackDatadogEvent(name, metadata)
  │
  └─ 1P 路由（第一方事件日志）
       ├─ shouldSampleEvent(name) → 采样率检查
       │    ├─ 返回 0 → 丢弃（采样掉了）
       │    ├─ 返回 null → 不采样（全部记录）
       │    └─ 返回 N → 记录并附加 sample_rate=N
       └─ logEventTo1P(name, metadata)
```

### 4.3 PII 安全

```typescript
// 两种标记类型确保元数据安全
AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS
  → 通用安全字符串

AnalyticsMetadata_I_VERIFIED_THIS_IS_PII_TAGGED
  → PII 标记值（_PROTO_* 前缀的键）
  → stripProtoFields() 在发送到 Datadog 前剥离
  → 1P 保留（因为 1P 有数据保护协议）
```

### 4.4 禁用条件

```typescript
isAnalyticsDisabled() = true 当:
  ├─ 测试环境
  ├─ 第三方 Provider（Bedrock / Vertex / Foundry）
  └─ 遥测被用户禁用
```

---

## 五、Telemetry 遥测

**文件**: `src/utils/telemetry/`

### 5.1 OpenTelemetry 集成

```
bootstrapTelemetry()
  │
  ├─ Trace Provider → 分布式追踪
  │    └─ OTLP / Prometheus 导出器
  │
  ├─ Metric Provider → 指标收集
  │    └─ Delta 时间性（非累积）
  │
  ├─ Log Provider → 结构化日志
  │    └─ 事件发射器
  │
  ├─ Resource Detection → 资源信息
  │    └─ env, host, OS
  │
  └─ 支持:
       ├─ 代理 / mTLS / CA 证书
       ├─ Console 导出器（调试）
       └─ 超时: 15s
```

### 5.2 事件记录

```typescript
logOTelEvent(eventName, metadata)
  ├─ 添加 getTelemetryAttributes() 上下文
  ├─ 递增 event.sequence 计数器
  ├─ 可选: prompt.id（来自 SDK）
  ├─ 工作区路径（CLAUDE_CODE_WORKSPACE_HOST_PATHS）
  └─ eventLogger.emit('claude_code.' + eventName, ...)
```

### 5.3 PII 保护

```typescript
redactIfDisabled(content: string): string
  ├─ OTEL_LOG_USER_PROMPTS=true → 返回原文
  └─ 否则 → 返回 '<REDACTED>'
```

---

## 六、记忆系统

### 6.1 架构总览

```
┌─────────────────────────────────────────────────────┐
│                    记忆系统                           │
│                                                      │
│  Session Memory（会话记忆）                          │
│  ├─ 当前对话的实时笔记                              │
│  ├─ 后台子 Agent 周期性提取                         │
│  └─ 存储: ~/.claude/projects/<path>/session-memory/ │
│                                                      │
│  Auto-Memory（自动记忆）                             │
│  ├─ 持久化跨会话的重要信息                          │
│  ├─ 对话结束时一次性提取                            │
│  └─ 存储: ~/.claude/projects/<path>/memory/          │
│                                                      │
│  Dream Task（记忆整合）                              │
│  ├─ 后台整合和优化已有记忆                          │
│  └─ UI: 状态栏小药丸 + 任务对话框                   │
│                                                      │
└─────────────────────────────────────────────────────┘
```

### 6.2 Session Memory

**文件**: `src/services/SessionMemory/`

```
触发条件（全部满足才触发）:
  ├─ GrowthBook 启用 tengu_session_memory
  ├─ 消息 Token ≥ 10,000（初始化阈值）
  ├─ 距上次提取 Token ≥ 5,000（更新阈值）
  └─ 距上次提取工具调用 ≥ 3（工具阈值）

提取流程:
  1. 检查阈值 → 满足
  2. markExtractionStarted() → 记录开始时间
  3. 分叉子 Agent（共享 Prompt Cache）
  4. Agent 分析新消息 → 提取要点
  5. 写入 session-memory 文件
  6. markExtractionCompleted()

等待机制:
  waitForSessionMemoryExtraction()
  → 最多等 15s（stale 阈值 1min）
  → 用于会话结束时确保最后一次提取完成
```

### 6.3 Auto-Memory

**文件**: `src/services/extractMemories/`

```
触发条件:
  ├─ 完整查询循环结束（模型最终响应，无工具调用）
  ├─ 自上次提取后有足够新消息
  └─ 自上次提取后没有 AutoMem 写入

提取流程:
  1. handleStopHooks 中触发
  2. 分叉 Agent（完美分叉 + 共享缓存）
  3. Agent 分析对话 → 提取持久化信息
  4. 写入 memory/ 目录
  5. 以 Markdown 格式存储

存储格式:
  ~/.claude/projects/<path>/memory/
  ├── user_role.md          # 用户角色信息
  ├── feedback_testing.md    # 测试相关反馈
  ├── project_auth.md        # 项目认证信息
  └── MEMORY.md              # 索引文件
```

---

## 七、成本追踪

**文件**: `src/cost-tracker.ts`

### 7.1 追踪指标

```typescript
// 实时追踪（存储在 bootstrap/state.ts）
getTotalCostUSD()                    // 总成本（美元）
getTotalInputTokens()                // 输入 Token 总数
getTotalOutputTokens()               // 输出 Token 总数
getTotalCacheReadInputTokens()       // 缓存读取 Token
getTotalCacheCreationInputTokens()   // 缓存创建 Token
getTotalDuration()                   // 总耗时
getTotalAPIDuration()                // API 调用总耗时
getTotalLinesAdded()                 // 新增代码行数
getTotalLinesRemoved()               // 删除代码行数
getTotalWebSearchRequests()          // 网络搜索次数
getModelUsage()                      // 按模型分类的使用量
hasUnknownModelCost()                // 是否有未知模型成本
```

### 7.2 持久化

```typescript
// 保存到项目配置
saveCurrentSessionCosts(fpsMetrics?)
  → 写入 ~/.claude/projects/<path>/config.json

// 从项目配置恢复
getStoredSessionCosts(sessionId)
  → 读取上次会话的成本数据
```

### 7.3 React Hook 集成

```typescript
// useCostSummary Hook
useCostSummary(getFpsMetrics?)
  ├─ 注册退出处理器
  ├─ 退出时显示成本摘要
  ├─ 退出时保存成本到项目配置
  └─ 仅在 hasConsoleBillingAccess() 时显示
```

---

## 八、命令系统（Slash Commands）

**文件**: `src/commands.ts`

### 8.1 命令类型

```typescript
Command = CommandBase & (
  | PromptCommand      // 内容型：展开为对话内容
  | LocalCommand       // 本地型：执行异步 CLI 操作
  | LocalJSXCommand    // JSX 型：渲染 React 组件
)

// 区分
PromptCommand: type = 'prompt'
  → 生成 ContentBlockParam[]
  → 注入对话 → 可能触发 API 调用

LocalCommand: type = 'local'
  → async (args, context) => LocalCommandResult
  → 纯本地操作（不调用 API）

LocalJSXCommand: type = 'local-jsx'
  → 渲染 React 组件
  → 完全接管 UI（如 /help, /diff）
```

### 8.2 命令注册

```typescript
// 条件导入 80+ 命令
getCommands() = memoize(() => {
  const commands = [
    helpCommand,           // /help
    clearCommand,          // /clear
    compactCommand,        // /compact
    modelCommand,          // /model
    costCommand,           // /cost
    diffCommand,           // /diff
    loginCommand,          // /login
    permissionsCommand,    // /permissions
    // ... 80+ 更多
  ]

  // Feature-gated 命令
  if (feature('VOICE_MODE')) commands.push(voiceCommand)
  if (feature('PROACTIVE')) commands.push(proactiveCommand)

  // 动态加载
  commands.push(...getSkillDirCommands())  // .claude/skills/
  commands.push(...getBundledSkills())     // 内置技能
  commands.push(...getPluginCommands())    // 插件命令

  // 过滤内部命令
  if (!isAntOnly()) {
    return commands.filter(c => !INTERNAL_ONLY_COMMANDS.has(c.name))
  }

  return commands
})
```

### 8.3 命令源

| 来源 | 类型 | 示例 |
|------|------|------|
| `builtin` | 硬编码 | `/help`, `/clear` |
| `bundled` | 编译内置 | `/commit`, `/review` |
| `plugin` | 插件提供 | `/my-plugin-cmd` |
| `mcp` | MCP 提示 | `/weather:forecast` |
| `managed` | 企业部署 | `/company-tool` |
| `skills` | 用户创建 | `.claude/skills/my-skill` |

---

## 九、会话历史

**文件**: `src/history.ts`

### 9.1 存储格式

```
~/.claude/history.jsonl（全局，跨项目）

每行一条 JSON:
{"sessionId":"abc","timestamp":1711900000,"type":"user","message":"..."}
{"sessionId":"abc","timestamp":1711900001,"type":"assistant","message":"..."}
```

### 9.2 粘贴内容管理

```typescript
// 长文本粘贴被替换为引用
getPastedTextRefNumLines(text)   // 计算行数
formatPastedTextRef(id, lines)   // "[Pasted text #1 +42 lines]"
formatImageRef(id)               // "[Image #1]"

// 展开引用
expandPastedTextRefs(input, pastedContents)
  // "[Pasted text #1]" → 实际内容

// 限制
MAX_PASTED_CONTENT_LENGTH = 1024  // 内联存储上限
MAX_HISTORY_ITEMS = 100           // 历史记录上限
```

### 9.3 历史读取

```typescript
makeHistoryReader()
  → AsyncGenerator<HistoryEntry>

makeLogEntryReader()
  → 从 history.jsonl 逆序读取
  → 按 sessionId 过滤
  → 跳过标记条目
```

---

## 十、消息格式转换

**文件**: `src/utils/messages/`

### 10.1 SDK ↔ 内部格式

```
SDK 消息格式（外部）     ←→     内部消息格式
SDKMessage                       Message
  │                               │
  ├─ role: 'user'|'assistant'     ├─ type: 'user'|'assistant'|'system'
  ├─ content: string|blocks       ├─ message: { content, usage }
  └─ tool_use_id?                 ├─ uuid, timestamp
                                  ├─ isMeta, toolUseResult
                                  └─ isCompactSummary
```

### 10.2 System Init 消息

每次 SDK 会话开始时发送的元数据消息：

```typescript
buildSystemInitMessage() = {
  type: 'system/init',
  cwd: '/path/to/project',
  session_id: 'session_abc',
  tools: ['Bash', 'Edit', 'Read', ...],
  mcp_servers: ['github', 'linear'],
  model: 'claude-opus-4-6',
  permissionMode: 'default',
  slash_commands: ['/help', '/model', ...],
  skills: ['commit', 'review', ...],
  agents: ['general-purpose', 'Explore', ...],
  plugins: ['my-plugin@marketplace'],
  claude_code_version: '2.1.88',
  output_style: 'explanatory',
  fast_mode: { enabled: false, state: 'off' }
}
```

---

## 十一、模型选择

**文件**: `src/utils/model/`

### 11.1 模型解析优先级

```
1. /model 命令覆盖（运行时）
2. --model CLI 参数
3. ANTHROPIC_MODEL 环境变量
4. settings.json 配置
5. 默认模型（Claude Opus 4.6）
```

### 11.2 多 Provider 支持

```typescript
// 每个模型有多个 Provider 的配置
ModelConfig = {
  firstParty: 'claude-opus-4-6-20250624',    // api.anthropic.com
  bedrock: 'anthropic.claude-opus-4-6-v1',   // AWS Bedrock
  vertex: 'claude-opus-4-6@20250624',        // Google Vertex
  foundry: 'claude-opus-4-6',                // Foundry
}

// Provider 特定默认模型
getDefaultSonnetModel():
  ├─ 1P: claude-sonnet-4-6
  └─ 3P: claude-sonnet-4-5（因为 3P 上线有延迟）
```

### 11.3 模型能力检测

```typescript
// modelCapabilities.ts
isModelSupported(model, capability): boolean
  ├─ 'thinking'     → 扩展思维
  ├─ 'cache_edits'  → 缓存编辑 API
  ├─ 'fast_mode'    → 快速模式
  └─ 'tool_search'  → 工具搜索
```

---

## 十二、关键架构模式总结

| 模式 | 使用位置 | 核心思想 |
|------|---------|---------|
| 文件监视缓存 | Git 集成 | dirty flag + 异步计算 + 竞态防护 |
| 队列排空 | Analytics | 零依赖 + 零丢失 + 幂等排空 |
| 分叉 Agent | Memory 系统 | 共享缓存 + 异步隔离 + 后台执行 |
| PKCE OAuth | 认证 | 标准 OAuth 2.0 + 本地回调服务器 |
| 多 Provider 桥接 | 模型选择 | 统一接口 + Provider 特定配置 |
| JSON-L 持久化 | 历史记录 | 追加写入 + 逆序读取 + 跨项目 |
| PII 标记 | Analytics | 类型级安全 + 按通道剥离 |
| 延迟 Schema 加载 | 设置验证 | Zod 编译推迟 + 打破导入循环 |
