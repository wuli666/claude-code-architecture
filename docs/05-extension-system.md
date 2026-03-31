# 05 - 扩展系统（MCP / Plugins / Skills）

> 三种扩展机制，让 Claude Code 的能力无限延伸

---

## 一、扩展体系总览

Claude Code 提供三种粒度的扩展机制：

```
┌─────────────────────────────────────────────────────────┐
│                                                          │
│  MCP Server（服务级）                                    │
│  ├─ 运行独立的服务器进程                                 │
│  ├─ 暴露: 工具(Tools) + 资源(Resources) + 提示(Prompts) │
│  ├─ 通信: stdio / SSE / HTTP / WebSocket / SDK          │
│  └─ 配置: .mcp.json                                     │
│                                                          │
│  Plugin（包级）                                          │
│  ├─ 一个完整的扩展包                                     │
│  ├─ 包含: 命令 + Agent + 技能 + Hook + MCP Server       │
│  ├─ 分发: Marketplace / Git                              │
│  └─ 配置: plugin.json manifest                           │
│                                                          │
│  Skill（文件级）                                         │
│  ├─ 单个 Markdown 文件                                   │
│  ├─ 可复用的 Prompt 片段                                 │
│  ├─ 支持: 模型覆盖 / 工具限制 / Hook                    │
│  └─ 路径: .claude/skills/                                │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

三者的集成关系：
```
Plugin 可以包含 → MCP Server + Skill + Command + Hook
MCP Server 可以暴露 → Prompt（作为 MCP Skill）
Skill 可以限制 → 使用的 Tool 集合
DXT 包可以包含 → 预编译的 MCP Server 二进制
```

---

## 二、MCP（Model Context Protocol）

### 2.1 什么是 MCP？

MCP 是 Anthropic 定义的 **模型上下文协议**，让外部服务器可以向 AI 模型暴露工具、资源和提示。可以类比为 AI 时代的 "USB 接口"——任何实现了 MCP 协议的服务器都可以被 Claude Code 使用。

### 2.2 核心架构

**关键文件**:
- `src/services/mcp/client.ts` (33KB) — 客户端实现
- `src/services/mcp/config.ts` (15KB) — 配置加载
- `src/services/mcp/types.ts` — 类型定义
- `src/tools/MCPTool/MCPTool.ts` — 工具代理

### 2.3 传输类型

| 类型 | 通信方式 | 使用场景 |
|------|---------|---------|
| `stdio` | 标准输入/输出 | 本地进程，最常用 |
| `sse` | Server-Sent Events | HTTP 远程服务器 |
| `sse-ide` | SSE + IDE 桥接 | VS Code / JetBrains |
| `http` | HTTP 请求/响应 | REST API |
| `ws` | WebSocket | 全双工远程 |
| `sdk` | 进程内调用 | SDK 嵌入式服务器 |

### 2.4 服务器配置

```json
// ~/.claude/.mcp.json 或 .mcp.json
{
  "mcpServers": {
    "github": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "ghp_xxx"
      }
    },
    "remote-api": {
      "type": "http",
      "url": "https://api.example.com/mcp",
      "headers": {
        "Authorization": "Bearer xxx"
      }
    }
  }
}
```

### 2.5 配置来源优先级

```
dynamicMcpConfig   → 运行时 API 动态添加
policySettings     → 企业管理策略（managed-mcp.json）
localSettings      → --plugin-dir 会话级
projectSettings    → .mcp.json（项目级）
userSettings       → ~/.claude/.mcp.json（用户级）
built-in/plugin    → 内置或插件提供
```

### 2.6 服务器生命周期

```
配置发现
  │
  ▼
创建 Transport（stdio/SSE/HTTP/WS/SDK）
  │
  ▼
MCP Client 连接
  │
  ├─ Initialize 握手
  │    ├─ 协议版本协商
  │    └─ 能力声明
  │
  ├─ ListTools → 工具发现
  │    ├─ 名称规范化: mcp__<server>__<tool-name>
  │    └─ JSON Schema → Zod Schema 转换
  │
  ├─ ListPrompts → 提示发现（作为 MCP Skill）
  │
  └─ ListResources → 资源发现
      └─ 用于 ReadMcpResource 工具

运行中...
  ├─ 工具调用: MCPTool.call() → RPC
  ├─ 通知接收: channel notifications
  └─ 资源变更: resourcesChanged

关闭
  └─ shutdown → 进程退出
```

### 2.7 MCPTool — 工具代理

每个 MCP 服务器暴露的工具都通过 `MCPTool` 模板实例化：

```typescript
// 模板定义（单个）
MCPTool = {
  name: 'mcp__<server>__<tool>',
  isMcp: true,
  inputSchema: passthrough,    // 任意 JSON 透传
  maxResultSizeChars: 100_000,
  call: (input) => mcpClient.callTool(toolName, input)
}
```

### 2.8 高级功能

**OAuth 支持**:
```
MCP 服务器配置可以包含 OAuth:
  ├─ 自动 OAuth 发现
  ├─ Token 刷新
  ├─ 回调处理
  └─ Cross-App Access (XAA/SEP-990) 支持
```

**Elicitation（用户输入收集）**:
```
MCP 服务器返回错误码 -32042 时:
  ├─ 向用户请求额外输入
  ├─ 支持: string, number, boolean, enum, date-time
  ├─ 自然语言日期解析（via Haiku LLM）
  └─ URL 或队列模式处理
```

**Channel Notifications**:
```
MCP 服务器推送消息:
  ├─ notifications/claude/channel → 用户消息注入
  ├─ XML <channel> 标签包装
  ├─ 支持: thread_id, user 等元数据
  └─ 需要 GrowthBook gate + 订阅检查
```

---

## 三、Plugin 系统

### 3.1 什么是 Plugin？

Plugin 是一个 **完整的扩展包**，可以包含多种组件类型。相比 MCP Server（只提供工具），Plugin 可以影响 Claude Code 的方方面面。

### 3.2 Plugin Manifest

```json
// plugin.json
{
  "name": "my-awesome-plugin",
  "version": "1.2.0",
  "description": "An awesome Claude Code plugin",
  "author": {
    "name": "Developer",
    "email": "dev@example.com"
  },

  "commands": "./commands/",         // Slash 命令
  "agents": "./agents/",            // 自定义 Agent
  "skills": "./skills/",            // 技能
  "outputStyles": "./styles/",       // 输出样式

  "hooks": {
    "PostToolUse": [{
      "command": "node ./hooks/format.js"
    }]
  },

  "mcpServers": {
    "my-server": {
      "type": "stdio",
      "command": "node",
      "args": ["./servers/main.js"]
    },
    "bundled-server": "./servers/my-tool.dxt"  // DXT 格式
  }
}
```

### 3.3 Plugin 组件类型

| 组件 | 路径 | 功能 |
|------|------|------|
| commands | `./commands/*.md` | Slash 命令（`/my-command`） |
| agents | `./agents/*.md` | 子 Agent 定义 |
| skills | `./skills/*.md` | 可复用提示片段 |
| hooks | manifest 内联 | 生命周期钩子 |
| outputStyles | `./styles/*.md` | 输出格式化 |
| mcpServers | manifest 或 `.dxt` | MCP 服务器 |

### 3.4 Plugin 生命周期

```
安装（首次使用）
  │
  ├─ Marketplace 拉取 / Git Clone
  ├─ Manifest 解析 & 验证
  ├─ 存储到 ~/.claude/plugins/ 或 .claude/plugins/
  │
  ▼
加载
  │
  ├─ Phase 1: 缓存加载（启动时）
  │    └─ 从上次安装的缓存加载
  │
  ├─ Phase 2: 后台调和（非阻塞）
  │    └─ 检查更新，安装新版本
  │
  ├─ Phase 3: 组件激活
  │    ├─ 注册 commands
  │    ├─ 注册 agents
  │    ├─ 注册 skills
  │    ├─ 注册 hooks
  │    └─ 连接 MCP servers
  │
  ▼
运行时
  │
  ├─ 用户可通过 /plugin 管理
  ├─ /reload-plugins 强制重载
  └─ 热重载（文件变更检测）
```

### 3.5 内置 Plugin

```typescript
// src/plugins/builtinPlugins.ts
registerBuiltinPlugin({
  name: 'my-builtin-plugin',
  description: '...',
  skills: [skillA, skillB],
  hooks: { PostToolUse: [...] },
  mcpServers: { ... },
  isAvailable: () => feature('MY_FEATURE'),
  defaultEnabled: true
})
```

内置 Plugin 的 ID 格式: `{name}@builtin`

### 3.6 错误处理

Plugin 使用 **辨别联合类型** 进行类型安全的错误处理：

```typescript
type PluginError =
  | { type: 'path-not-found', ... }
  | { type: 'git-auth-failed', ... }
  | { type: 'plugin-not-found', source, pluginId, marketplace }
  | { type: 'manifest-parse-error', ... }
  | { type: 'manifest-validation-error', ... }
  | { type: 'mcp-config-invalid', ... }
  | { type: 'hook-load-failed', ... }
  | { type: 'mcpb-download-failed', ... }
  // ... 25+ 错误类型
```

---

## 四、Skill 系统

### 4.1 什么是 Skill？

Skill 是最轻量的扩展方式——一个 Markdown 文件就是一个 Skill。它本质是一个 **可复用的 Prompt 片段**，可以被用户通过 `/skill-name` 调用，或被模型通过 SkillTool 调用。

### 4.2 Skill 定义

```markdown
<!-- .claude/skills/code-review/SKILL.md -->
---
description: Review code for bugs and best practices
aliases: [review, cr]
when_to_use: When the user asks for code review or CR
allowed_tools: [FileRead, Grep, Glob]
model: claude-sonnet-4-6
---

You are an expert code reviewer. Analyze the provided code for:
1. Bugs and logic errors
2. Security vulnerabilities
3. Performance issues
4. Code style and best practices

Focus on actionable, specific feedback.
```

### 4.3 Frontmatter 字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `description` | string | 技能描述 |
| `aliases` | string[] | 别名列表 |
| `when_to_use` | string | 何时使用（模型参考） |
| `allowed_tools` | string[] | 允许的工具子集 |
| `model` | string | 模型覆盖 |
| `context` | `'inline' \| 'fork'` | 执行模式 |
| `agent` | string | 关联 Agent |
| `hooks` | HooksSettings | 技能专属 Hook |

### 4.4 Skill 来源

```
优先级（合并顺序）:
  1. 内置 Skill（registerBundledSkill）
  2. 磁盘 Skill（.claude/skills/ + ~/.claude/skills/）
  3. Plugin Skill（plugin.manifest.skills）
  4. MCP Skill（MCP Prompts 转化）
```

### 4.5 执行模式

```
Inline 模式（默认）:
  ├─ Prompt 直接注入当前对话
  ├─ 共享上下文
  └─ 适合: 简短的提示增强

Fork 模式:
  ├─ 创建独立的子 Agent
  ├─ 隔离的 Token 预算
  ├─ 不影响主对话上下文
  └─ 适合: 复杂任务、独立分析
```

### 4.6 变更检测

**文件**: `src/utils/skills/skillChangeDetector.ts`

```
文件监视器:
  ├─ 监视: .claude/skills/ + ~/.claude/skills/
  ├─ 使用 chokidar（polling 回退）
  ├─ 防抖: 300ms（FILE_STABILITY: 1000ms）
  └─ 变更时:
      ├─ 清除命令/技能缓存
      ├─ 通知监听器
      └─ 执行 ConfigChange hooks
```

> **为什么用 polling 回退?** Bun 运行时的 PathWatcherManager 在某些情况下会死锁，chokidar 的 polling 模式是更可靠的备选。

### 4.7 内置 Skill 框架

```typescript
registerBundledSkill({
  name: 'commit',
  description: 'Create a git commit',
  aliases: ['ci'],
  files: {
    // 参考文件（首次使用时提取到磁盘）
    'guidelines.md': '...'
  },
  getPromptForCommand: async (args, context) => {
    // 提取参考文件到 getBundledSkillsRoot()/commit/
    await ensureFilesExtracted()

    return [{
      type: 'text',
      text: `Base directory: ${skillRoot}\n\nCreate a commit...`
    }]
  }
})
```

---

## 五、DXT（Desktop Extension）格式

### 5.1 什么是 DXT？

DXT 是一个 **ZIP 打包格式**，用于分发包含预编译二进制的 MCP 服务器。类似于 macOS 的 `.app` bundle——自包含、跨平台。

### 5.2 文件结构

```
my-server.dxt (ZIP)
├── manifest.json           # 清单文件
├── bin/
│   ├── macos-arm64/       # macOS ARM
│   ├── macos-x64/         # macOS Intel
│   ├── linux-x64/         # Linux
│   └── windows-x64.exe    # Windows
└── resources/             # 可选资源文件
```

### 5.3 Manifest 格式

```json
{
  "name": "my-mcp-server",
  "version": "1.0.0",
  "author": { "name": "Dev" },
  "description": "A bundled MCP server",

  "command": "bin/${platform}/server",
  "args": ["--config", "${user_config.api_key}"],

  "user_config": {
    "api_key": {
      "name": "API Key",
      "type": "string",
      "required": true,
      "sensitive": true,
      "description": "Your API key"
    },
    "max_results": {
      "type": "integer",
      "default": 10
    }
  }
}
```

### 5.4 安全措施

```
ZIP 解压保护:
  ├─ 路径遍历检测（../../ 攻击）
  ├─ 文件数限制: 100,000
  ├─ 最大解压大小: 1GB
  ├─ 单文件上限: 512MB
  ├─ ZIP 炸弹检测: 压缩率 > 50:1 告警
  └─ 可执行权限保留（+x）
```

### 5.5 用户配置存储

```
非敏感配置:
  → settings.pluginConfigs[pluginId].mcpServers[serverName]

敏感配置（sensitive: true）:
  → secureStorage（系统 Keychain）
  → 键格式: {pluginId}/{serverName}
```

---

## 六、扩展集成关系图

```
                    ┌──────────────┐
                    │  SkillTool   │
                    │  (执行技能)  │
                    └──────┬───────┘
                           │ 调用
         ┌─────────────────┼─────────────────┐
         │                 │                 │
    ┌────▼────┐      ┌────▼────┐      ┌────▼────┐
    │ 内置    │      │ 磁盘    │      │  MCP    │
    │ Skill   │      │ Skill   │      │ Skill   │
    └─────────┘      └────┬────┘      └────┬────┘
                          │                 │
                   .claude/skills/    MCP ListPrompts
                          │                 │
                    ┌─────┴─────┐     ┌────▼────┐
                    │  Plugin   │     │  MCP    │
                    │  Skills   │     │ Server  │
                    └─────┬─────┘     └────┬────┘
                          │                 │
                    ┌─────▼─────┐     ┌────▼────┐
                    │  Plugin   │─────│  DXT    │
                    │  Bundle   │     │ Package │
                    └───────────┘     └─────────┘
                          │
           ┌──────────────┼──────────────┐
           │              │              │
      Commands       Hooks         Output Styles
```

---

## 七、AppState 中的扩展状态

```typescript
// MCP 子系统
AppState.mcp = {
  clients: MCPServerConnection[],        // 活跃连接
  tools: Tool[],                          // 已发现工具
  commands: Command[],                    // MCP Prompts + Skills
  resources: Record<string, Resource[]>,  // 资源列表
}

// Plugin 子系统
AppState.plugins = {
  enabled: LoadedPlugin[],                // 已启用插件
  disabled: LoadedPlugin[],               // 已禁用插件
  installationStatus: {                   // 安装状态
    marketplaces: { name, status, error }[],
    plugins: { name, status }[]
  },
  errors: PluginError[]                   // 错误列表
}
```

---

## 八、关键设计模式

### 1. 延迟加载
```
MCP Client 模块 (~33KB): 动态 import
Plugin Loader: 调用 /plugin 时才加载
DXT 验证库 (@anthropic-ai/mcpb, ~700KB): 懒导入
```

### 2. 缓存策略
```
Plugin Manifest: 版本化缓存（内容哈希）
MCP MCPB 文件: .mcpb-cache/ 目录
Marketplace 配置: 本地 JSON 缓存
Skill/Command 发现: memoized + 变更时清除
```

### 3. 去重机制
```
Skills: 按真实文件路径去重（符号链接解析）
Commands: 按名称去重（MCP + 本地合并）
MCP Servers: 按名称去重，冲突检测
Tools: 内置优先，MCP 次之
```

### 4. 注册-发现模式
```
内置 Skill → registerBundledSkill() → 全局注册表
MCP Skill → registerMCPSkillBuilders() → 避免循环依赖
Plugin Skill → loadPluginSkills() → 按 manifest 加载
→ 运行时统一通过 getCommands() 发现所有来源
```
