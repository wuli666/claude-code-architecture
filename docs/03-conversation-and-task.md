# 03 - 对话循环与 Task 系统

> 核心运转引擎：从用户输入到模型响应的完整消息流

---

## 一、系统总览

对话系统由三个核心组件构成：

```
QueryEngine（高层编排器）
     │
     ├─ 管理多轮对话状态
     ├─ 系统提示词构建
     ├─ 用户输入处理
     └─ 调用 ↓

query()（核心对话循环 — AsyncGenerator）
     │
     ├─ Compaction 检查
     ├─ API 调用（withRetry 包装）
     ├─ Tool 执行
     ├─ 继续/终止判定
     └─ 每次迭代 yield 事件

Task 系统（后台任务管理）
     │
     ├─ LocalAgentTask — 本地 Agent
     ├─ RemoteAgentTask — 远程 Agent
     ├─ DreamTask — 记忆整合
     └─ LocalShellTask — 后台 Shell
```

---

## 二、query() — 核心对话循环

**文件**: `src/query.ts` (1729 行)

这是整个 Claude Code 最核心的函数。它是一个 **AsyncGenerator**，每次迭代处理一轮模型交互：

```typescript
export async function* query(
  params: QueryParams,
): AsyncGenerator<
  StreamEvent | RequestStartEvent | Message | TombstoneMessage,
  Terminal  // 返回值：终止原因
>
```

### 循环状态机

```typescript
type State = {
  messages: Message[]                          // 对话消息
  toolUseContext: ToolUseContext                // 工具执行上下文
  autoCompactTracking: AutoCompactTrackingState // 自动压缩追踪
  maxOutputTokensRecoveryCount: number          // max_output_tokens 恢复计数
  hasAttemptedReactiveCompact: boolean          // 是否已尝试响应式压缩
  pendingToolUseSummary: Promise<...>           // 待处理的工具摘要
  turnCount: number                             // 回合数
  transition: Continue | undefined              // 上次继续原因
}
```

### 单次迭代流程

```
                 ┌──────────────────────────┐
                 │ 0. Skill 预获取 + Memory │
                 └────────────┬─────────────┘
                              │
                              ▼
         ┌────────────────────────────────────────┐
         │ 1. Compaction 检查                      │
         │                                         │
         │  ┌─ Tool Result Budget 应用              │
         │  ├─ Snip Compact（如果启用）             │
         │  ├─ Microcompact                         │
         │  │   ├─ 时间型（上次助手消息间隔过大）   │
         │  │   └─ 缓存型（cache_edits API）        │
         │  ├─ Context Collapse（上下文折叠）       │
         │  └─ Auto-compact（完整摘要重写）         │
         └────────────────┬───────────────────────┘
                          │
                          ▼
         ┌────────────────────────────────────────┐
         │ 2. API 调用                             │
         │                                         │
         │  ├─ System Prompt 组装                   │
         │  ├─ User Context 注入                    │
         │  ├─ withRetry() 包装                     │
         │  │   ├─ 429/529 智能重试                 │
         │  │   ├─ Fast Mode 降级                   │
         │  │   ├─ OAuth 401 刷新                   │
         │  │   └─ 无人值守持久重试                 │
         │  ├─ 流式响应接收                         │
         │  │   ├─ 思维块(thinking)                 │
         │  │   ├─ 文本块(text)                     │
         │  │   └─ 工具调用块(tool_use)             │
         │  └─ 工具并行/串行执行                    │
         └────────────────┬───────────────────────┘
                          │
                          ▼
         ┌────────────────────────────────────────┐
         │ 3. 后处理                               │
         │                                         │
         │  ├─ Post-sampling hooks 执行             │
         │  ├─ Abort 信号检查                       │
         │  ├─ 工具使用摘要                         │
         │  └─ yield 所有消息                       │
         └────────────────┬───────────────────────┘
                          │
                          ▼
         ┌────────────────────────────────────────┐
         │ 4. 继续/终止判定                        │
         │                                         │
         │  ├─ 有 tool_use 块 → 继续循环            │
         │  ├─ stop_reason=stop → 返回 Terminal     │
         │  └─ 异常恢复:                            │
         │      ├─ Collapse Drain → 排空折叠后重试  │
         │      ├─ Reactive Compact → 紧急压缩重试  │
         │      └─ Truncation → 截断头部后重试      │
         └────────────────────────────────────────┘
```

---

## 三、Compaction — 三级上下文压缩

Claude 模型有上下文窗口限制。当对话变长时，Compaction 系统确保不超出窗口：

### 3.1 层级结构

```
             Token 使用量增长 →

  ┌──────────┬──────────────────┬──────────────────┬─────────┐
  │ 正常区间  │  Microcompact    │  Auto-compact    │ 阻塞限制 │
  │          │  (轻量修剪)      │  (完整摘要)       │         │
  └──────────┴──────────────────┴──────────────────┴─────────┘
  0%                                              90%    100%
```

### 3.2 Microcompact — 轻量修剪

**文件**: `src/services/compact/microCompact.ts`

两种触发方式：

**时间型 Microcompact**:
```
条件: 距上次助手消息间隔超过阈值
行为: 直接修改消息内容
      旧工具结果 → "[Old tool result content cleared]"
影响: 缓存失效（内容已变）
```

**缓存型 Microcompact（cache_edits API）**:
```
条件: 模型支持 + GrowthBook 启用 + 主线程
行为: 不修改本地消息内容
      通过 cache_edits API 指示服务端删除旧缓存
影响: 本地消息不变，缓存层修剪
```

**可压缩的工具类型**:
- File Read/Edit/Write, Bash, Grep, Glob, Web Search/Fetch
- **不可压缩**: Agent, SendMessage, TaskStop, TeamCreate/Delete

### 3.3 Auto-compact — 完整摘要

**文件**: `src/services/compact/autoCompact.ts`

```typescript
// 关键阈值
MAX_OUTPUT_TOKENS_FOR_SUMMARY = 20_000    // 摘要最大 Token
AUTOCOMPACT_BUFFER_TOKENS = 13_000         // 有效窗口缓冲区
WARNING_THRESHOLD_BUFFER_TOKENS = 20_000   // UI 警告阈值
MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3   // 熔断器

// 计算公式
effectiveWindow = contextWindow - min(maxOutputTokens, 20_000)
autoCompactThreshold = effectiveWindow - 13_000
```

**触发抑制场景**:
- `session_memory` / `compact` 查询（分叉 Agent）→ 抑制
- `marble_origami`（上下文 Agent）→ 抑制
- 仅响应式压缩模式 → 抑制
- Context Collapse 模式（90% 提交 / 95% 阻塞）→ 抑制

### 3.4 Reactive Compact — 紧急压缩

当 API 返回 `prompt_too_long` 错误时触发：

```
API 返回 prompt_too_long
  │
  ├─ 1. 尝试 Collapse Drain（如果有待排空的折叠）
  │      └─ 成功 → 重试
  │
  ├─ 2. 尝试 Reactive Compact（完整摘要压缩）
  │      └─ 成功 → 重试
  │
  └─ 3. 最后手段: Truncation（截断消息头部）
         └─ 重试
```

### 3.5 Compaction 结果结构

```typescript
interface CompactionResult {
  boundaryMarker: SystemMessage       // 标记压缩边界的合成消息
  summaryMessages: UserMessage[]       // 压缩后的摘要
  attachments: AttachmentMessage[]     // 重新注入的附件
  hookResults: HookResultMessage[]     // 压缩后 Hook 结果
  messagesToKeep?: Message[]           // 保留的消息后缀
  preCompactTokenCount?: number        // 压缩前 Token 数
  postCompactTokenCount?: number       // 压缩后 Token 数
}

// 最终消息序列
buildPostCompactMessages(result) = [
  boundaryMarker,
  ...summaryMessages,
  ...messagesToKeep,
  ...attachments,
  ...hookResults,
]
```

---

## 四、QueryEngine — SDK 编排器

**文件**: `src/QueryEngine.ts` (600+ 行)

QueryEngine 是面向 SDK/无头模式的高层封装，管理跨轮次的对话状态：

```typescript
class QueryEngine {
  private mutableMessages: Message[]        // 跨轮次持久化
  private abortController: AbortController
  private totalUsage: NonNullableUsage      // 累计 Token 使用
  private discoveredSkillNames: Set<string>
}
```

### submitMessage() 生命周期

```
submitMessage(userInput)
  │
  ├── 1. 解析模型覆盖、思维配置
  ├── 2. 获取系统提示词
  │      ├─ fetchSystemPromptParts()
  │      ├─ Git 状态
  │      ├─ CLAUDE.md 内容
  │      └─ 记忆机制注入
  ├── 3. 用户输入处理
  │      ├─ processUserInput() — Slash 命令展开
  │      ├─ 推入 mutableMessages
  │      └─ 持久化到 transcript
  ├── 4. 技能 & 插件加载
  │      └─ yield buildSystemInitMessage()
  │          （工具/命令/Agent/技能元数据）
  ├── 5. 本地命令路径（如果是 Slash 命令）
  │      └─ 直接返回，不调用 API
  └── 6. 调用 query() 循环
         ├─ yield 所有 SDK 消息
         ├─ 累计 usage
         └─ 处理订阅/PR 活动事件
```

### QueryEngine vs query() 的关系

```
QueryEngine = "有状态的对话管理器"
  ├─ 维护消息历史（跨轮次）
  ├─ 处理用户输入（Slash 命令解析）
  ├─ 系统提示词构建
  └─ 调用 query() 处理每一轮

query() = "无状态的单轮执行器"
  ├─ 接收消息 → 调用 API → 执行工具 → 返回结果
  ├─ Compaction 管理
  └─ 不知道"对话"的概念，只知道"消息列表"
```

---

## 五、API 服务层

### 5.1 Claude API 封装

**文件**: `src/services/api/claude.ts` (2000+ 行)

```typescript
// 关键函数
getExtraBodyParams()      // 组装 anthropic_beta 数组
getPromptCachingEnabled() // 按模型判断是否启用缓存
getCacheControl()         // 返回缓存配置 {type, ttl, scope}
queryModelWithStreaming()  // 主 API 调用
```

**Prompt Cache 配置**:
```typescript
{
  type: 'ephemeral',
  ttl: '1h',         // 符合条件时（Bedrock/1P + GrowthBook）
  scope: 'global'    // 全局缓存（跨会话共享）
}
```

### 5.2 重试逻辑

**文件**: `src/services/api/withRetry.ts` (400+ 行)

```typescript
export async function* withRetry<T>(
  getClient,
  operation,
  options: {
    maxRetries?: number,           // 默认 10
    model: string,
    fallbackModel?: string,        // 过载时降级模型
    fastMode?: boolean,
    signal?: AbortSignal,
  },
): AsyncGenerator<SystemAPIErrorMessage, T>
```

**重试策略矩阵**:

| 错误类型 | 策略 |
|---------|------|
| 429 (Rate Limit) | 读取 Retry-After → 等待 → 重试 |
| 529 (Overloaded) | 指数退避 → Fast Mode 降级 |
| 401 (Token Expired) | 刷新 OAuth Token → 重试 |
| 403 (Token Revoked) | 刷新 → 重试 |
| ECONNRESET | 禁用 keep-alive → 重连 |
| prompt_too_long | 不重试 → 交由 query() 处理 |

**Fast Mode 降级**:
```
Fast Mode 429/529:
  ├─ retry-after 短（< SHORT_RETRY_THRESHOLD_MS）
  │    → sleep → 重试（保留缓存）
  ├─ retry-after 长
  │    → 进入冷却期（切换到标准速度模型）
  └─ 配额超限
       → 永久禁用 Fast Mode
```

**无人值守持久重试**:
```
当 CLAUDE_CODE_UNATTENDED_RETRY 设置时（仅内部）:
  ├─ 429/529 无限重试
  ├─ 最大退避: 5 分钟
  ├─ 重置上限: 6 小时
  ├─ 心跳: 每 30 秒 yield 消息
  └─ 用于长时间无人值守的批量任务
```

---

## 六、上下文管理

**文件**: `src/context.ts`

### System Context（系统上下文）

```typescript
// memoized — 对话期间缓存
getSystemContext() = {
  gitStatus: string,    // 分支、状态、最近提交
  cacheBreaker?: string // 缓存打破器（仅内部）
}
```

Git Status 收集（并行执行）:
```
Promise.all([
  getBranch(),              // 当前分支
  getDefaultBranch(),       // 默认分支（main/master）
  'git status --short',     // 工作区状态（截断 2000 字符）
  'git log -n 5',           // 最近 5 次提交
  'git config user.name',   // 用户名
])
```

### User Context（用户上下文）

```typescript
// memoized — 对话期间缓存
getUserContext() = {
  claudeMd: string,      // CLAUDE.md 文件内容（所有层级合并）
  currentDate: string     // "Today's date is 2026-03-31."
}
```

CLAUDE.md 发现流程:
```
从 CWD 向上遍历到 Git 根目录
  ├─ 每级目录收集 .claude/ 配置
  ├─ Worktree 回退: 如果 worktree 缺少 .claude/ → 使用主仓库的
  └─ 合并: managed > user > project
```

---

## 七、Task 系统

### 7.1 Task 类型与状态

**文件**: `src/Task.ts`

```typescript
// 任务类型
type TaskType =
  | 'local_bash'           // 本地 Shell
  | 'local_agent'          // 本地 Agent
  | 'remote_agent'         // 远程 Agent
  | 'in_process_teammate'  // 进程内队友
  | 'local_workflow'       // 本地工作流
  | 'monitor_mcp'          // MCP 监视器
  | 'dream'                // 记忆整合

// 任务状态
type TaskStatus = 'pending' → 'running' → 'completed' | 'failed' | 'killed'
```

**任务 ID 生成**:
```typescript
// 前缀 + 8 随机字符（36^8 ≈ 2.8 万亿种组合）
// 前缀: a=agent, b=bash, d=dream, s=main-session
generateTaskId('a') → "a7x9k2m1"
```

### 7.2 Dream Task — 记忆整合

**文件**: `src/tasks/DreamTask/DreamTask.ts`

Dream Task 是后台运行的记忆整合任务：

```
DreamTaskState:
  ├─ phase: 'starting' → 'updating'
  ├─ sessionsReviewing: number     // 已审阅的会话数
  ├─ filesTouched: number          // 触及的文件数
  ├─ turns: RecentTurn[]           // 最近助手回合（最多 30）
  └─ abortController               // 取消控制器
```

> **本质**: Dream Task 本身只是 UI 展示（状态条小药丸 + 任务对话框）。实际的记忆整合工作由分叉 Agent 在其他地方执行。

### 7.3 Main Session Task — 后台化

**文件**: `src/tasks/LocalMainSessionTask.ts`

当用户后台化当前会话时（如 `Ctrl+Z`），主查询循环变成一个 Task：

```
前台对话
  │
  ├─ 用户按 Ctrl+Z 或切换到其他 Agent
  │
  ▼
registerMainSessionTask()
  ├─ 生成 's' 前缀任务 ID
  ├─ 链接输出到独立 transcript 文件
  ├─ 标记 isBackgrounded: true
  └─ 在 AsyncLocalStorage 隔离中运行

startBackgroundSession()
  ├─ runWithAgentContext() 隔离
  ├─ 增量写入 transcript
  ├─ UI 进度更新: {tokenCount, toolUseCount, recentActivities}
  ├─ Abort 处理: 原子检查 notified 标志
  └─ 完成: completeMainSessionTask(success)

foregroundMainSessionTask()
  └─ 交换前台/后台状态，恢复之前的会话
```

---

## 八、Coordinator 模式 — 多 Agent 编排

**文件**: `src/coordinator/coordinatorMode.ts`

Coordinator 模式将 Claude Code 转变为一个 **团队协调器**，管理多个 Worker Agent：

### 激活条件

```typescript
function isCoordinatorMode(): boolean {
  return feature('COORDINATOR_MODE') &&
         isEnvTruthy(process.env.CLAUDE_CODE_COORDINATOR_MODE)
}
```

### Coordinator 架构

```
Coordinator（指挥官）
  │
  ├─ 拥有: Agent Tool + SendMessage Tool + TaskStop Tool
  ├─ 不直接执行代码
  ├─ 负责: 理解需求 → 分解任务 → 派发 Worker → 合成结果
  │
  ├─── Worker A（代码探索）
  │     └─ 拥有: File/Grep/Glob/Bash（只读）
  │
  ├─── Worker B（代码实现）
  │     └─ 拥有: File/Edit/Write/Bash
  │
  └─── Worker C（验证测试）
        └─ 拥有: Bash/File/Grep
```

### 工作流阶段

```
1. Research（研究）
   ├─ 多个 Worker 并行探索
   └─ Coordinator 合成发现

2. Synthesis（合成）
   ├─ Coordinator 理解全局
   └─ 制定具体实施方案

3. Implementation（实施）
   ├─ Worker 按方案编写代码
   └─ 写入操作串行执行

4. Verification（验证）
   ├─ 每个文件区域独立验证
   └─ Worker 证明代码可工作
```

### Worker 能力暴露

```typescript
// Coordinator 的 User Context 中包含 Worker 可用工具清单
getCoordinatorUserContext() = {
  workerToolsContext: `
    Workers spawned via the Agent tool have access to these tools:
    Bash, Edit, FileRead, FileWrite, Glob, Grep, ...

    Workers also have access to MCP tools from connected MCP servers:
    github, linear, slack

    Scratchpad directory: /tmp/claude-scratchpad/
    Workers can read and write here without permission prompts...
  `
}
```

---

## 九、消息类型体系

```typescript
// 用户消息
UserMessage = {
  type: 'user',
  message: { content: string | ContentBlockParam[] },
  uuid, timestamp,
  isMeta?: boolean,           // 合成消息标记
  toolUseResult?: ToolResult,  // 如果是工具结果
  isCompactSummary?: boolean,  // 如果是压缩摘要
}

// 助手消息
AssistantMessage = {
  type: 'assistant',
  message: { content: ContentBlockParam[], usage },
  uuid, timestamp,
  apiError?: string,           // 错误恢复
}

// 系统消息
SystemMessage = {
  type: 'system',
  content: string,
  subtype?: 'compact_boundary' | 'local_command',
  compactMetadata?: {          // 压缩边界元数据
    preservedSegment: {
      headUuid, anchorUuid, tailUuid
    }
  }
}

// 附件消息（压缩后重新注入）
AttachmentMessage = {
  type: 'attachment',
  attachment: { type: 'skill_discovery' | ... }
}

// 墓碑消息（标记删除）
TombstoneMessage = {
  type: 'tombstone',
  message: ...
}
```

---

## 十、关键设计模式

### 1. AsyncGenerator 驱动的流式处理
```
query() 是一个 AsyncGenerator:
  ├─ yield 消息 → 调用方（QueryEngine/REPL）逐个处理
  ├─ return Terminal → 终止原因
  └─ 天然支持背压（调用方不消费就不继续生产）
```

### 2. 三级压缩防御
```
Microcompact  → 轻量修剪（不影响语义）
Auto-compact  → 完整摘要（语义压缩）
Reactive      → 紧急压缩（API 返回错误时）

三级递进，从轻到重，确保永不超出上下文窗口
```

### 3. 分叉 Agent 模式
```
主对话分叉:
  ├─ 共享 Prompt Cache → 零额外缓存成本
  ├─ AsyncLocalStorage 隔离 → 状态不污染
  └─ 后台执行 → 不阻塞用户

用于: Session Memory / Dream / Agent / Background Query
```

### 4. 智能重试策略
```
不是简单的"失败就重试":
  ├─ 429: 尊重 Retry-After 头
  ├─ 529: Fast Mode 降级后重试
  ├─ 401: 刷新 OAuth Token 后重试
  ├─ prompt_too_long: 不重试，交给压缩系统
  └─ ECONNRESET: 禁用 keep-alive 后重连
```
