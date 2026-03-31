# 01 - 启动流程与 Bootstrap

> Claude Code 从 `node cli.js` 到 REPL 完全就绪的全链路分析

---

## 一、启动全景图

```
$ claude [args...]
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│  Phase 0: cli.tsx — 快速路径分发（零重型导入）                    │
│  ├─ --version → 直接输出，退出                                   │
│  ├─ --daemon-worker → 精简 Worker 启动                           │
│  ├─ remote/bridge/sync → bridgeMain()                            │
│  ├─ daemon → daemonMain()                                        │
│  ├─ ps/logs/attach/kill → 会话管理                               │
│  └─ 默认 → 动态 import main.tsx                                  │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│  Phase 1: main.tsx — Commander.js 解析 + 模式判定                │
│  ├─ 安全设置（SIGINT / PATH 劫持防护）                           │
│  ├─ Deep Link / SSH Remote / Assistant 模式检测                  │
│  ├─ 交互/非交互模式判定                                         │
│  ├─ Entrypoint 分类（cli / sdk-cli / remote / vscode / ...）    │
│  └─ Commander preAction → await init()                           │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│  Phase 2: init.ts — 全局初始化（memoized，整个进程只执行一次）    │
│  ├─ enableConfigs() — 配置验证 & 启用                            │
│  ├─ applySafeConfigEnvironmentVariables()                        │
│  ├─ applyExtraCACertsFromConfig() — TLS 证书                     │
│  ├─ setupGracefulShutdown()                                      │
│  ├─ OpenTelemetry 初始化                                         │
│  ├─ populateOAuthAccountInfoIfNeeded()                           │
│  ├─ detectCurrentRepository() — GitHub 仓库检测                  │
│  ├─ configureGlobalMTLS() — 双向 TLS                             │
│  ├─ configureGlobalAgents() — HTTP 代理                          │
│  ├─ preconnectAnthropicApi() — TCP+TLS 预连接                    │
│  └─ setShellIfWindows() / registerCleanup()                      │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│  Phase 3: setup.ts — 会话级初始化                                │
│  ├─ Node.js 版本检查（≥18）                                     │
│  ├─ UDS 消息服务器启动                                           │
│  ├─ 终端备份恢复（iTerm2 / Terminal.app）                       │
│  ├─ setCwd() — 设置工作目录（必须最先）                          │
│  ├─ Worktree 创建（Git worktree + tmux session）                │
│  ├─ 后台任务注册（Session Memory / Version Lock）                │
│  ├─ Plugin Hooks 预加载                                          │
│  ├─ Analytics Sink 初始化                                        │
│  ├─ API Key 预获取                                               │
│  └─ 权限旁路安全验证                                             │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│  Phase 4: REPL 启动                                              │
│  ├─ React Root 创建                                              │
│  ├─ <App><REPL /></App> 渲染                                     │
│  ├─ MCP 服务器连接（非阻塞）                                    │
│  ├─ 插件加载                                                     │
│  └─ 等待用户输入...                                              │
└─────────────────────────────────────────────────────────────────┘
```

---

## 二、Phase 0: CLI 入口 — 快速路径分发

**文件**: `src/entrypoints/cli.tsx`

这是整个程序的第一个入口文件。它的核心设计理念是 **尽早分流**——在加载任何重型模块之前，先检查是否可以走快速路径。

### 快速路径清单

| 检查条件 | 行为 | 设计原因 |
|---------|------|---------|
| `--version` / `-v` | 直接输出版本号 → `process.exit(0)` | 零模块加载，瞬间返回 |
| `--dump-system-prompt` | 输出系统提示词（仅内部） | 用于敏感性评估 |
| `--daemon-worker=<kind>` | 精简 Worker 启动 | 避免加载完整 CLI 开销 |
| `remote` / `bridge` / `sync` | `bridgeMain()` | 独立子系统，不需要 REPL |
| `daemon` | `daemonMain()` | 长驻后台进程 |
| `ps` / `logs` / `attach` / `kill` | 会话管理命令 | 操作 `~/.claude/sessions/` |
| `--bg` / `--background` | 后台会话 | 同上 |
| `--worktree --tmux` | 在 tmux 中启动 | 必须在完整 CLI 初始化前 exec |

### 关键代码逻辑

```typescript
// 零依赖版本输出 — 构建时宏替换
if (cliArgs.includes('--version') || cliArgs.includes('-v')) {
  process.stdout.write(MACRO.VERSION + '\n')
  process.exit(0)
}

// 所有快速路径都不通过 → 加载完整 CLI
const { cliMain } = await import('./main.js')
await cliMain()
```

> **设计洞察**: `MACRO.VERSION` 是构建时内联的字符串常量，不需要 `require('package.json')`。这让 `--version` 的冷启动时间降到几毫秒。

### 性能剖析检查点

整个启动流程埋入了大量 `profileCheckpoint()` 调用，用于诊断启动性能：

```
profileCheckpoint('main_tsx_entry')           // main.tsx 顶部
profileCheckpoint('main_tsx_imports_loaded')   // 所有导入完成
profileCheckpoint('main_function_start')       // main() 函数开始
profileCheckpoint('run_commander_initialized') // Commander.js 就绪
profileCheckpoint('preAction_start')           // preAction hook 开始
profileCheckpoint('preAction_after_init')      // init() 完成
profileCheckpoint('action_handler_start')      // Action 处理器开始
profileCheckpoint('setup_started')             // setup() 开始
profileCheckpoint('setup_hooks_captured')       // Hooks 快照完成
```

---

## 三、Phase 1: main.tsx — 模式判定与 Commander 设置

**文件**: `src/main.tsx`

### 3.1 安全防护（先于一切）

```typescript
// Windows PATH 劫持防护
process.env.NoDefaultCurrentDirectoryInExePath = '1'

// SIGINT 处理器注册
process.on('SIGINT', () => { /* graceful shutdown */ })
```

### 3.2 模式判定

Claude Code 需要判断当前运行在哪种模式下，这决定了后续的行为：

```typescript
const hasPrintFlag = cliArgs.includes('-p') || cliArgs.includes('--print')
const hasInitOnlyFlag = cliArgs.includes('--init-only')
const hasSdkUrl = cliArgs.some(arg => arg.startsWith('--sdk-url'))

// 非交互模式：print 模式 / SDK / 非 TTY stdout
const isNonInteractive = hasPrintFlag || hasInitOnlyFlag || hasSdkUrl || !process.stdout.isTTY

setIsInteractive(!isNonInteractive)
```

### 3.3 Entrypoint 分类

根据环境变量和调用方式，确定当前入口类型：

| Entrypoint 值 | 触发条件 |
|---------------|---------|
| `cli` | 交互式终端 |
| `sdk-cli` | 非交互式（print / SDK） |
| `github-action` | GitHub Actions 环境 |
| `claude-vscode` | VS Code 扩展 |
| `claude-desktop` | 桌面应用 |
| `local-agent` | 本地 Agent 调用 |
| `remote` | 远程会话 |

### 3.4 Commander.js 注册

`run()` 函数创建 Commander 实例并注册所有选项：

```typescript
const program = createCommand('claude')
  .argument('[prompt]', '要发送给 Claude 的提示')
  .option('-p, --print', '非交互式打印模式')
  .option('--model <model>', '指定模型')
  .option('--permission-mode <mode>', '权限模式')
  .option('--resume <sessionId>', '恢复会话')
  .option('--continue', '继续上次会话')
  // ... 更多选项
```

### 3.5 preAction Hook

**关键机制**: Commander 的 `preAction` 在任何子命令执行前运行：

```typescript
program.hook('preAction', async () => {
  // 1. 等待异步预加载（MDM / Keychain）
  await Promise.all([awaitMdmSettingsLoad(), awaitKeychainPrefetch()])

  // 2. 全局初始化
  await init()

  // 3. Analytics Sinks
  initializeAnalyticsGates()

  // 4. 内联插件
  if (opts.pluginDir) loadInlinePlugin(opts.pluginDir)

  // 5. 数据迁移
  await runMigrations()

  // 6. 远程设置加载（非阻塞）
  loadRemoteManagedSettings()  // fire-and-forget
  loadPolicyLimits()            // fire-and-forget
})
```

---

## 四、Phase 2: init() — 全局初始化

**文件**: `src/entrypoints/init.ts`

`init()` 使用 `memoize()` 确保整个进程只运行一次，即使被多次调用也是幂等的。

### 初始化步骤详解

```
init()
├── 1. enableConfigs()                          配置验证 & 启用
├── 2. applySafeConfigEnvironmentVariables()     安全环境变量
├── 3. applyExtraCACertsFromConfig()             自定义 CA 证书
│      ⚠️ 必须在任何 TLS 连接前（Bun 启动时缓存证书）
├── 4. setupGracefulShutdown()                   优雅关闭注册
├── 5. OpenTelemetry SDK 动态导入 & 初始化       延迟加载（性能）
├── 6. populateOAuthAccountInfoIfNeeded()         OAuth 账户信息
├── 7. JetBrains IDE 检测                        异步填充缓存
├── 8. detectCurrentRepository()                  GitHub 仓库识别
├── 9. 远程设置 & 策略限制 Promise 初始化        并发加载
├── 10. recordFirstStartTime()                    首次启动时间
├── 11. configureGlobalMTLS()                     双向 TLS 证书
├── 12. configureGlobalAgents()                   HTTP 代理 & mTLS
├── 13. preconnectAnthropicApi()                  ⚡ TCP+TLS 预连接
│       ⚠️ 与后续初始化工作重叠执行
├── 14. Upstream Proxy 设置（CCR）                懒导入，失败则跳过
├── 15. setShellIfWindows()                       Windows Shell 设置
├── 16. registerCleanup(shutdownLspServerManager) LSP 清理注册
└── 17. ensureScratchpadDir()                     Scratchpad 目录
```

### 关键设计决策

**1. 预连接 API（步骤 13）**
```typescript
preconnectAnthropicApi()
// 在 init() 还在做其他工作时，TCP 三次握手 + TLS 握手已经开始
// 到真正调用 API 时，连接已经建立好了
```

**2. 上游代理失败静默（步骤 14）**
```typescript
try {
  const { setupUpstreamProxy } = await import('./upstreamproxy/setup.js')
  setupUpstreamProxy()
} catch {
  // fail-open: 代理设置失败不影响启动
}
```

**3. CA 证书时序要求（步骤 3）**
```
⚠️ Bun 运行时在启动时缓存 CA 证书列表。
如果在此之后才设置自定义 CA → TLS 连接会失败。
因此 applyExtraCACertsFromConfig() 必须在所有网络操作前执行。
```

---

## 五、Phase 3: setup() — 会话级初始化

**文件**: `src/setup.ts`

`setup()` 负责每次会话的特定初始化，与 `init()` 不同，它可能在不同参数下被多次调用。

### 函数签名

```typescript
export async function setup(
  cwd: string,                         // 工作目录
  permissionMode: PermissionMode,       // 权限模式
  allowDangerouslySkipPermissions: boolean,
  worktreeEnabled: boolean,             // 是否使用 Git Worktree
  worktreeName: string | undefined,
  tmuxEnabled: boolean,                 // 是否使用 tmux
  customSessionId?: string | null,
  worktreePRNumber?: number,
  messagingSocketPath?: string,
)
```

### 初始化流程

```
setup()
├── 1. Node.js 版本检查 ≥ 18
├── 2. Custom Session ID 设置
├── 3. UDS 消息服务器
│      ├─ 创建 Unix Domain Socket
│      ├─ 路径: tmpdir 或显式指定
│      └─ 导出 $CLAUDE_CODE_MESSAGING_SOCKET
├── 4. 终端备份恢复
│      ├─ iTerm2: 检查备份（Swarms 启用时）
│      └─ Terminal.app: 检查备份
│      ⚠️ 仅交互模式，非交互跳过
├── 5. setCwd(cwd) ← 🔴 关键时序点
│      ⚠️ 必须在任何 CWD 依赖代码之前
│      ├─ 捕获 Hooks 配置快照
│      └─ 初始化 FileChanged Hook Watcher
├── 6. Worktree 创建（如果启用）
│      ├─ 检查 Git 仓库
│      ├─ 解析到主仓库根目录
│      ├─ 生成 tmux 会话名
│      ├─ createWorktreeForSession()
│      ├─ createTmuxSessionForWorktree()
│      ├─ 更新 CWD & projectRoot
│      └─ 清除记忆文件缓存
├── 7. 后台任务注册
│      ├─ Session Memory 初始化（同步）
│      ├─ Context Collapse 初始化
│      └─ 锁定当前版本号
├── 8. Plugin Hooks 预加载
│      └─ 热重载设置
├── 9. Analytics
│      ├─ 仓库分类
│      ├─ Commit 归因 Hook
│      ├─ Session 文件访问 Hook
│      ├─ Team Memory 同步监视器
│      └─ initSinks() — 附加错误日志 + Analytics Sinks
├── 10. API Key 预获取
│       └─ 信任确认后安全预获取
├── 11. 权限旁路验证
│       ├─ --dangerously-skip-permissions 安全检查
│       ├─ 拒绝 root/sudo（除非在沙箱中）
│       ├─ 检查 Docker / Bubblewrap / IS_SANDBOX
│       └─ 无网络访问则允许
└── 12. 上次会话退出事件记录
```

### CWD 设置的时序依赖

```
setCwd() 是整个系统的关键锚点：
                    setCwd()
                       │
         ┌─────────────┼─────────────┐
         │             │             │
    Hooks 加载    FileChanged    权限规则
    (需要 CWD     监视器        (路径解析
     查找配置)    (需要 CWD     需要 CWD)
                   确定监视路径)
```

---

## 六、传输层架构

**文件**: `src/cli/transports/`

Claude Code 支持多种通信传输方式，用于不同的部署场景：

### 传输选择优先级

```
CLAUDE_CODE_USE_CCR_V2 ?
  → SSETransport（SSE 读 + HTTP POST 写）
    └─ 路径: ${url}/worker/events/stream

CLAUDE_CODE_POST_FOR_SESSION_INGRESS_V2 ?
  → HybridTransport（WebSocket 读 + HTTP POST 写）
    └─ 用于 Bridge 模式远程控制

默认:
  → WebSocketTransport（WebSocket 双向通信）
```

### WebSocket 传输

```typescript
// 状态机
type State = 'idle' → 'connected' → 'reconnecting' → 'closing' → 'closed'

// 特性
- 自动重连（指数退避）
- 睡眠检测（60s 阈值 → 系统休眠/唤醒）
- 心跳机制（10s ping/pong）
- Keep-alive 帧（5min 间隔）
- 永久关闭码（1002, 4001, 4003）跳过重试
```

### Hybrid 传输

在 WebSocket 读取的基础上，写入使用 HTTP POST：

```
write(message)
  → 如果是 stream_event: 缓冲（最多 100ms）
  → 否则: 立即刷新缓冲 + 入队
  → SerialBatchEventUploader
      ├─ 串行批量 POST（防止并发写冲突）
      ├─ 指数退避（500ms 基础，8s 上限）
      ├─ 无限重试
      └─ 反压：maxQueueSize = 100k 事件
```

### SSE 传输

```
SSE 帧解析器:
  ├─ 增量解析文本缓冲区
  ├─ 双换行分隔帧
  ├─ 字段提取: event, id, data
  └─ 注释处理（以 : 开头的行）

特性:
  ├─ 重连 + 指数退避
  ├─ 存活超时（45s 无数据 = 连接已死）
  └─ 永久 HTTP 状态码（401, 403, 404）
```

---

## 七、Bootstrap State — 全局状态隔离

**文件**: `src/bootstrap/state.ts`

Bootstrap 模块维护进程级全局状态，有严格的 **导入隔离规则**：

```
⚠️ state.ts 不允许导入任何非 bootstrap 模块
   → 防止循环依赖
   → 确保在最早期可用
```

### 关键状态函数

| 函数 | 作用 |
|------|------|
| `getSessionId()` / `switchSession()` | 会话 ID 管理 |
| `setIsInteractive()` | 交互模式标记 |
| `setClientType()` | 客户端类型（cli/sdk/remote/...） |
| `setSessionSource()` | 会话来源标记 |
| `setInitialMainLoopModel()` | 默认模型设置 |
| `setAllowedSettingSources()` | 设置源控制 |
| `setAllowedChannels()` | MCP 通道白名单 |

---

## 八、REPL 启动

**文件**: `src/replLauncher.tsx`

REPL 是最终的交互界面，通过 React 组件树启动：

```tsx
// 延迟加载（减少初始 bundle 解析时间）
const App = lazy(() => import('./components/App.js'))
const REPL = lazy(() => import('./screens/REPL.js'))

// 组合渲染
<KeybindingSetup>
  <App>
    <REPL
      messages={[]}
      tools={tools}
      commands={commands}
      permissionMode={permissionMode}
      model={model}
      // ... 更多 props
    />
  </App>
</KeybindingSetup>
```

### REPL 启动条件

在 REPL 渲染前，以下条件必须满足：

- [x] Root React 组件已创建
- [x] 所有预获取完成（API Key / Release Notes）
- [x] Worktree 已创建（如果请求）
- [x] Session ID 已建立
- [x] 模型已选择
- [x] 工具和命令已加载
- [x] 权限模式已确定
- [ ] MCP 服务器连接（**非阻塞**，后台进行）

---

## 九、启动性能优化策略

| 策略 | 实现 | 效果 |
|------|------|------|
| 零导入快速路径 | `--version` 不加载任何模块 | 毫秒级返回 |
| 动态导入 | `await import()` 延迟加载重型模块 | 减少初始解析 |
| 构建时宏替换 | `MACRO.VERSION` 编译内联 | 无运行时开销 |
| API 预连接 | `preconnectAnthropicApi()` | TCP+TLS 重叠 |
| MDM 子进程预读 | `startMdmRawRead()` | 并行 I/O |
| Keychain 预获取 | `startKeychainPrefetch()` | 并行 I/O |
| memoized init | `init = memoize(async () => ...)` | 去重执行 |
| 非阻塞 MCP | MCP 连接后台进行 | 不阻塞 REPL |
| fire-and-forget | 远程设置/策略限制 | 不阻塞启动 |

---

## 十、关键时序依赖图

```
startMdmRawRead() ──┐
                     ├── preAction 等待 ──→ init()
startKeychainPrefetch()──┘                    │
                                              │
               applyExtraCACertsFromConfig()──┤ (TLS 必须最先)
                                              │
               preconnectAnthropicApi() ──────┤ (与其他初始化重叠)
                                              │
                                              ▼
                                         setup()
                                              │
                                setCwd() ─────┤ (锚点：后续全部依赖 CWD)
                                              │
                                  Hooks ──────┤
                                              │
                               initSinks() ──┤
                                              │
                                              ▼
                                        REPL 启动
```
