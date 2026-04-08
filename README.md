# Claude Code 源码架构深度分析

[![linux.do](https://img.shields.io/badge/linux.do-huo0-blue?logo=linux&logoColor=white)](https://linux.do)

> 基于 [`@anthropic-ai/claude-code@2.1.88`](https://www.npmjs.com/package/@anthropic-ai/claude-code) sourcemap 还原源码的架构分析，源码还原来自 [ChinaSiro/claude-code-sourcemap](https://github.com/ChinaSiro/claude-code-sourcemap)

---

## 这是什么？

这是对 [Anthropic Claude Code](https://claude.com/product/claude-code) CLI 工具的 **源码级架构分析文档**。

Claude Code 是 Anthropic 官方推出的 AI 编程助手 CLI，它不是一个简单的命令行工具，而是一个完整的 **终端原生 AI IDE**——在终端中运行了一个完整的 React 应用（深度 fork 的 Ink 框架），集成了对话管理、工具调用、权限控制、插件系统、终端 UI 渲染、多 Agent 协调等能力。

本仓库通过分析其 npm 包中 sourcemap 还原出的源码（约 **4500+ 文件、1900+ TypeScript 源文件**），深入剖析了整个系统的架构设计与实现细节。

> **注意**：本仓库 **不包含** Claude Code 的源代码。Claude Code 是 Anthropic 的商业软件，受其 [Commercial Terms of Service](https://www.anthropic.com/legal/commercial-terms) 约束。本仓库仅包含基于公开分发的 npm 包进行的原创架构分析文档。

---

## 分析来源

| 项目 | 信息 |
|------|------|
| 分析对象 | `@anthropic-ai/claude-code` |
| 版本 | `2.1.88` |
| 来源 | npm 包内置 sourcemap (`cli.mjs.map`) |
| 还原方式 | 通过 sourcemap 中的 `sourcesContent` 字段还原 |
| 源码规模 | ~4,500 文件 / ~1,900 TypeScript 源文件 |
| 分析时间 | 2025-03 |

---

## 文档目录

| # | 文档 | 内容 | 关键词 |
|---|------|------|--------|
| 00 | [架构总览](./docs/00-architecture-overview.md) | 整体架构鸟瞰、目录结构、数据流、七大子系统 | 全景图、技术栈、模块关系 |
| 01 | [启动流程](./docs/01-startup-and-bootstrap.md) | 从 `node cli.js` 到 REPL 就绪的 4 阶段启动 | Bootstrap、init、setup、传输层 |
| 02 | [Tool 工具系统](./docs/02-tool-system.md) | 30+ 工具的定义、注册、权限校验、执行生命周期 | Tool 接口、三层校验、延迟加载 |
| 03 | [对话循环与 Task](./docs/03-conversation-and-task.md) | 核心对话循环、消息流、上下文压缩、多 Agent 协调 | query()、Compaction、Coordinator |
| 04 | [权限与安全](./docs/04-permissions-and-security.md) | 权限模型、Hooks、沙箱隔离、Settings 层级 | 规则匹配、Auto 模式、MDM |
| 05 | [扩展系统](./docs/05-extension-system.md) | MCP 协议、Plugin 系统、Skills 技能、DXT 格式 | MCP Server、Marketplace、Skill |
| 06 | [终端 UI](./docs/06-terminal-ui-and-state.md) | Ink 渲染管线、状态管理、输入处理、快捷键系统 | React+Terminal、Yoga、Store |
| 07 | [基础设施](./docs/07-infrastructure-and-services.md) | Git、OAuth、LSP、Analytics、Memory、成本追踪 | 文件监视缓存、PKCE、遥测 |
| 08 | [Harness 六维评估](./docs/08-harness-six-dimensions.md) | 上下文工程、工具编排、验证机制、状态管理、可观测性、人类接管 | 成熟度评级、缺口分析、雷达图 |
| 09 | [Memory 系统深度解析](./docs/09-memory-system.md) | 分类体系、文件落盘、写入链路、召回注入、Memory 幻觉、Dream 整合 | AutoMem、extraction agent、Sonnet 召回 |

---

## 核心发现

### 1. 终端原生 IDE，不是 CLI 工具

Claude Code 在终端中运行了一个完整的 **React 应用**，使用深度 fork 的 Ink 框架 + Yoga (WASM) Flexbox 布局引擎。支持鼠标追踪、Alt-Screen、Kitty 键盘协议、应用内文本选择、超链接等高级终端特性。

### 2. 四阶段渐进初始化

```
Phase 0: 快速路径分发（--version 零依赖返回）
Phase 1: 全局初始化（TLS/OAuth/代理，API TCP+TLS 预连接）
Phase 2: 会话初始化（CWD/Hooks/Worktree/Analytics）
Phase 3: REPL 启动（React 渲染 + MCP 非阻塞连接）
```

### 3. 三级上下文压缩防御

```
Microcompact（轻量修剪）→ Auto-compact（完整摘要）→ Reactive compact（紧急压缩）
```

确保无论对话多长都不会超出上下文窗口——这是 LLM 应用工程中极其关键但很少被公开讨论的实现。

### 4. 失败关闭的安全设计

工具系统默认 `isConcurrencySafe: false`、`isReadOnly: false`——新增工具忘记设置标记时，会串行执行且需要权限确认，而非并发执行或静默通过。

### 5. 三种粒度的扩展机制

| 扩展类型 | 粒度 | 能力 |
|---------|------|------|
| MCP Server | 服务级 | 工具 + 资源 + Prompt |
| Plugin | 包级 | 命令 + Agent + 技能 + Hook + MCP |
| Skill | 文件级 | 可复用 Prompt 片段 |

---

## 技术栈

| 层面 | 技术 |
|------|------|
| 运行时 | Node.js 18+ / Bun |
| 语言 | TypeScript |
| UI 框架 | Ink (深度 fork) + React |
| 布局引擎 | Yoga (WASM Flexbox) |
| CLI 框架 | Commander.js |
| API 通信 | Anthropic SDK |
| Schema 验证 | Zod |
| 搜索引擎 | ripgrep (内嵌) |

---

## 如何自行还原源码

如果你想自己分析源码，可以从 npm 包中还原：

```bash
# 1. 下载 npm 包
npm pack @anthropic-ai/claude-code

# 2. 解压
tar -xzf anthropic-ai-claude-code-*.tgz

# 3. 从 sourcemap 还原（需要自行编写提取脚本）
# sourcemap 位于 package/cli.mjs.map
# 解析 sourcesContent 字段即可还原源文件
```

---

## 声明

- 本仓库 **不包含** Anthropic Claude Code 的源代码
- Claude Code 是 Anthropic 的商业软件，所有权利归 Anthropic 所有
- 本仓库仅包含基于公开分发的 npm 包进行的 **原创架构分析文档**
- 分析目的为技术学习与研究，如有侵权请联系删除

---

## License

本仓库中的分析文档采用 [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/) 许可证发布。

---

## Star History

如果这个分析对你有帮助，请给个 Star 支持一下！
