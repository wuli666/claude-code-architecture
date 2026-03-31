# 06 - 终端 UI 与状态管理

> 在终端中构建一个完整的 React 应用——从像素级渲染到事件驱动的状态管理

---

## 一、技术选型：为什么是 Ink？

Claude Code 的终端 UI 不是传统的 `console.log` 打印——它是一个完整的 **React 应用**，运行在终端中。

```
传统 CLI:  console.log("Hello") → stdout
Claude Code: <Box><Text>Hello</Text></Box> → React → Yoga → ANSI → stdout
```

选择 Ink（React 终端渲染器）的原因：
- **声明式** — 描述 UI 应该是什么样的，而非如何更新
- **组件化** — 复用消息渲染、权限对话框等组件
- **状态驱动** — 状态变更自动触发 UI 更新
- **Flexbox 布局** — Yoga 引擎提供真正的 CSS Flexbox

但原版 Ink 无法满足 Claude Code 的需求，因此做了 **深度 fork**。

---

## 二、渲染管线

```
React 组件树
  │
  ▼
┌──────────────────────────────────────────┐
│ Ink Reconciler（自定义 React 协调器）     │
│ ├─ createElement / updateElement         │
│ ├─ 属性 Diff                             │
│ └─ 事件处理器注册                        │
└───────────────┬──────────────────────────┘
                │
                ▼
┌──────────────────────────────────────────┐
│ DOM Tree（虚拟 DOM）                      │
│ ├─ ink-root / ink-box / ink-text         │
│ ├─ 7 种节点类型                          │
│ ├─ Dirty 标记（增量渲染）                │
│ └─ 滚动状态管理                          │
└───────────────┬──────────────────────────┘
                │
                ▼
┌──────────────────────────────────────────┐
│ Yoga Layout（Flexbox 引擎 — WASM）       │
│ ├─ 每个 DOM 节点 ↔ 一个 Yoga 节点        │
│ ├─ Flexbox 计算: direction, wrap, align  │
│ └─ 文本宽度测量函数                      │
└───────────────┬──────────────────────────┘
                │
                ▼
┌──────────────────────────────────────────┐
│ Screen Buffer（字符网格）                 │
│ ├─ 二维字符数组 + 样式数组               │
│ ├─ 超链接映射                            │
│ └─ 高度限制（Alt-Screen 模式）           │
└───────────────┬──────────────────────────┘
                │
                ▼
┌──────────────────────────────────────────┐
│ Diff & Patch                              │
│ ├─ 逐行对比前后帧                        │
│ ├─ 跳过未变更区域（Blit 优化）           │
│ ├─ 生成 ANSI 转义序列                    │
│ └─ 超链接 OSC 8 序列                     │
└───────────────┬──────────────────────────┘
                │
                ▼
┌──────────────────────────────────────────┐
│ Terminal stdout                           │
│ ├─ 光标定位                              │
│ ├─ 样式应用                              │
│ └─ 实际字符输出                          │
└──────────────────────────────────────────┘
```

---

## 三、Ink Fork 的关键增强

### 3.1 与原版 Ink 的区别

| 特性 | 原版 Ink | Claude Code Fork |
|------|---------|-----------------|
| 渲染 API | `render()` 阻塞 | `render()` + `createRoot()` |
| 鼠标支持 | 无 | Mode-1003 hover + click |
| 键盘协议 | 基本 | Kitty 键盘协议 + modifyOtherKeys |
| 屏幕模式 | 普通 | Alt-Screen 管理 |
| 文本选择 | 无 | 应用内文本选择覆盖层 |
| 搜索高亮 | 无 | 位置追踪 + 高亮渲染 |
| 超链接 | 无 | OSC 8 协议支持 |
| 焦点管理 | 简单 | DOM-like 焦点栈（32 层） |
| 性能 | 全量渲染 | 增量 Blit + 样式池 + 字符缓存 |

### 3.2 帧渲染循环

**文件**: `src/ink/ink.tsx`

```typescript
scheduleRender()
  │
  ├─ 节流: ~60fps（FRAME_INTERVAL_MS）
  │
  ▼
renderFrame()
  │
  ├─ React commit phase（Yoga 布局计算）
  ├─ Output 生成（Screen Buffer）
  ├─ Diff 计算（与前一帧对比）
  ├─ Patch 优化（合并相邻修改）
  ├─ 光标 + 选择覆盖层
  └─ Terminal 写入
```

**双缓冲**:
```
frontFrame (当前显示)  ←→  backFrame (正在计算)
  └─ 交换后：旧 front 变为下一帧的 back
  └─ 字符/超链接池在帧间复用（减少 GC）
```

### 3.3 终端控制码

**文件**: `src/ink/termio/`

```
CSI（Control Sequence Introducer）:
  ├─ 光标移动: cursorPosition(row, col)
  ├─ 屏幕操作: ERASE_SCREEN, CURSOR_HOME
  ├─ 滚动区域: setScrollRegion(top, bottom)
  ├─ 键盘模式: Kitty protocol, modifyOtherKeys
  └─ 鼠标追踪: ENABLE_MOUSE_TRACKING

DEC（Digital Equipment Corporation）:
  ├─ Alt-Screen: ENTER_ALT_SCREEN, EXIT_ALT_SCREEN
  ├─ 光标可见性: SHOW_CURSOR, HIDE_CURSOR
  └─ 括号粘贴模式

OSC（Operating System Command）:
  ├─ 剪贴板: setClipboard(content)
  ├─ Tab 状态: setTabStatus(text)
  ├─ 超链接: link(url) → \x1b]8;;url\x07text\x1b]8;;\x07
  └─ 终端复用器检测: tmux / screen
```

---

## 四、状态管理

### 4.1 Store 实现

**文件**: `src/state/store.ts`

一个极简的 Pub/Sub Store：

```typescript
class Store<T> {
  private state: T
  private listeners: Set<() => void>

  getState(): T

  setState(updater: (prev: T) => T): void {
    const next = updater(this.state)
    if (Object.is(this.state, next)) return  // 引用相等跳过
    this.state = next
    this.listeners.forEach(l => l())         // 通知订阅者
  }

  subscribe(listener: () => void): () => void
}
```

> **设计洞察**: 没有使用 Redux、Zustand 等库，而是自己实现了最简 Store。因为终端 UI 的状态管理需求比 Web 简单得多——不需要中间件、时间旅行调试等功能。

### 4.2 AppState 类型

**文件**: `src/state/AppStateStore.ts` (450+ 行)

AppState 是一个 **巨大的 DeepImmutable 类型**，包含整个应用的状态：

```typescript
type AppState = DeepImmutable<{
  // ── UI 状态 ──
  expandedView: boolean
  footerSelection: number
  viewSelectionMode: boolean
  coordinatorTaskIndex: number

  // ── 权限 ──
  toolPermissionContext: {
    mode: PermissionMode
    alwaysAllowRules: RulesBySource
    alwaysDenyRules: RulesBySource
    alwaysAskRules: RulesBySource
  }

  // ── 任务管理 ──
  tasks: { [taskId: string]: TaskState }
  foregroundedTaskId: string | null
  viewingAgentTaskId: string | null

  // ── MCP ──
  mcp: {
    clients: MCPServerConnection[]
    tools: Tool[]
    commands: Command[]
    resources: Record<string, ServerResource[]>
  }

  // ── 插件 ──
  plugins: {
    enabled: LoadedPlugin[]
    disabled: LoadedPlugin[]
    errors: PluginError[]
  }

  // ── 团队 ──
  teamContext: {
    leader: TeammateInfo
    teammates: TeammateInfo[]
    colors: Map<string, string>
  }

  // ── 远程会话 ──
  remoteSessionUrl: string | null
  remoteConnectionStatus: string
  remoteBackgroundTaskCount: number

  // ── 更多... ──
  authVersion: number
  tungstenActiveSession: string | null
  bagelActive: boolean
  // ... 共 50+ 字段
}>
```

### 4.3 状态变更钩子

**文件**: `src/state/onChangeAppState.ts`

当状态变更时，自动同步到外部系统：

```
状态变更 → onChangeAppState()
  │
  ├─ permissionMode 变更 → 外部化到 CCR/SDK
  ├─ mainLoopModel 变更 → 持久化到 settings.json
  ├─ expandedView 变更 → 持久化到全局配置
  ├─ tmux panel 可见性 → 持久化到全局配置
  ├─ 设置变更 → 清除 auth 缓存（API Key, AWS, GCP）
  └─ 环境变量 → 传播到 process.env
```

---

## 五、屏幕与组件架构

### 5.1 主 REPL 屏幕

**文件**: `src/screens/REPL.tsx` (~875KB)

REPL 是应用的核心屏幕——一个巨大的 React 组件，连接所有子系统：

```
<REPL>
  ├─ 虚拟消息列表
  │    ├─ UserPromptMessage
  │    ├─ AssistantTextMessage
  │    ├─ AssistantThinkingMessage
  │    ├─ AssistantToolUseMessage
  │    ├─ SystemTextMessage
  │    ├─ RateLimitMessage
  │    └─ ...
  ├─ PromptInput（输入框）
  ├─ Footer（状态栏）
  ├─ PermissionDialogs（权限确认）
  ├─ DiffDialog（差异查看器）
  ├─ TaskView（任务面板）
  └─ Notifications（通知）
```

### 5.2 组件树

```
<KeybindingSetup>
  <App>
    ├─ AppContext（终端尺寸、输入事件、焦点）
    ├─ StdinContext（原始模式控制）
    ├─ TerminalSizeContext（动态缩放）
    ├─ TerminalFocusProvider（终端聚焦/失焦）
    ├─ ClockProvider（动画时钟）
    ├─ ThemeProvider（主题：light/dark/auto）
    │
    └─ <REPL>
        ├─ 消息渲染区
        ├─ 输入区
        ├─ 状态栏
        └─ 对话框层
  </App>
</KeybindingSetup>
```

### 5.3 输入组件

**文件**: `src/components/PromptInput/PromptInput.tsx`

```
PromptInput:
  ├─ 多模式文本输入（insert / replace / search）
  ├─ 历史记录补全
  ├─ Vim 模式支持
  ├─ 图片粘贴处理
  ├─ Suggestion 系统
  │    ├─ Typeahead（输入预测）
  │    ├─ Prompt Suggestion（提示建议）
  │    └─ History Search（历史搜索）
  ├─ 模式指示器
  ├─ 权限模式切换（Shift+Tab）
  ├─ Fast Mode 切换
  ├─ Effort Level 指示
  └─ IDE Selection 处理
```

### 5.4 消息渲染

**文件**: `src/components/messages/`

每种消息类型有专门的渲染组件：

**用户消息**:
```
UserPromptMessage     → 用户输入（含附件）
UserBashInputMessage  → Bash 命令输入
UserBashOutputMessage → Bash 输出
UserImageMessage      → 内联图片
UserToolResultMessage → 工具结果（成功/错误/拒绝/取消）
UserCommandMessage    → Slash 命令
UserTeammateMessage   → 跨队友消息
```

**助手消息**:
```
AssistantTextMessage           → 纯文本 + Markdown
AssistantThinkingMessage       → 思维过程（扩展思维）
AssistantRedactedThinkingMessage → 隐藏思维
AssistantToolUseMessage        → 工具调用 + 执行进度
GroupedToolUseContent          → 多个连续工具调用分组
```

**系统消息**:
```
SystemTextMessage      → 系统提示
SystemAPIErrorMessage  → API 错误
RateLimitMessage       → 速率限制通知
CompactBoundaryMessage → 压缩边界标记
HookProgressMessage    → Hook 执行进度
```

### 5.5 Diff 查看器

**文件**: `src/components/diff/DiffDialog.tsx`

```
DiffDialog:
  ├─ 列表视图 + 详情视图
  ├─ Git Diff + 回合 Diff
  ├─ 文件选择 + Hunk 导航
  └─ 快捷键: ↑↓ 导航, Enter 详情, Esc 关闭
```

---

## 六、输入处理管线

### 6.1 从原始字节到语义事件

```
stdin（原始字节流）
  │
  ▼
useInput Hook（ink/hooks/use-input.ts）
  ├─ 启用 raw mode（禁用行缓冲 + 回显）
  ├─ 监听 stdin EventEmitter
  │
  ▼
parseKey()（ink/events/input-event.ts）
  ├─ CSI u 解析（Kitty 键盘协议）: [codepoint;modifiers u
  ├─ xterm modifyOtherKeys: [27;modifiers;keycode~
  ├─ 应用键盘模式: O<letter>（数字键盘）
  ├─ 功能键 / 方向键 / 特殊键
  ├─ Shift 检测（大写字母 A-Z）
  ├─ 修饰符归一化（alt/meta 合并, super 独立）
  └─ 返回 [Key, input] 元组

  Key = {
    upArrow, downArrow, leftArrow, rightArrow,
    pageUp, pageDown, wheelUp, wheelDown,
    home, end, return, escape,
    ctrl, alt, shift, meta, super, fn,
    tab, backspace, delete
  }
  │
  ▼
KeyboardEvent 包装
  │
  ▼
useKeybinding 解析
  └─ 见下文 §7
```

### 6.2 鼠标事件

```
Mode-1003 鼠标追踪:
  ├─ CSI 序列: \x1b[M + 3 字节（button, x, y）
  ├─ SGR 扩展: \x1b[<button;x;y;M/m
  │
  ├─ onClick → Box 组件的点击处理
  ├─ onMouseEnter / onMouseLeave → 悬停效果
  └─ wheelUp / wheelDown → 滚动事件
```

---

## 七、快捷键系统

**文件**: `src/keybindings/`

### 7.1 架构

```
用户配置
  ~/.claude/keybindings.json
  │
  ▼
loadUserBindings()
  │
  ├─ 解析快捷键字符串: "ctrl+k ctrl+s" → ParsedBinding
  ├─ 验证上下文: Global / Editor / Chat / Search
  └─ 合并默认绑定 + 用户覆盖
  │
  ▼
Resolver（解析引擎）
  │
  ├─ resolveKey() — 单键匹配
  └─ resolveKeyWithChordState() — 和弦匹配
       ├─ 追踪 pending: ParsedKeystroke[]
       ├─ 前缀检测 → chord_started
       ├─ 完整匹配 → match
       ├─ Esc → chord_cancelled
       └─ 最后匹配的绑定获胜（用户覆盖默认）
```

### 7.2 和弦快捷键

```
示例: Ctrl+K Ctrl+S（保存所有）

第一次按键: Ctrl+K
  → 检测到和弦前缀
  → 返回 'chord_started'
  → UI 显示 "Ctrl+K..."

第二次按键: Ctrl+S
  → 与待处理和弦组合
  → 完整匹配 → 触发动作
  → 返回 'match'

第二次按键: Esc
  → 取消和弦
  → 返回 'chord_cancelled'
```

### 7.3 上下文（Context）

快捷键在不同上下文中有不同行为：

| 上下文 | 激活条件 | 示例 |
|--------|---------|------|
| `Global` | 始终 | Ctrl+C 退出 |
| `Editor` | 输入框聚焦 | Ctrl+A 全选 |
| `Chat` | 对话模式 | Enter 提交 |
| `Search` | 搜索模式 | Enter 确认搜索 |

---

## 八、焦点管理

**文件**: `src/ink/focus.ts`

Claude Code 实现了类浏览器的焦点管理系统：

```typescript
FocusManager:
  ├─ activeElement: DOMElement | null    // 当前焦点
  ├─ focusStack: DOMElement[]            // 焦点栈（最多 32 层）
  │
  ├─ focus(node) → 设置焦点 + 派发 blur/focus 事件
  ├─ blur() → 清除焦点
  ├─ handleNodeRemoved() → 节点删除时清理
  ├─ handleAutoFocus() → React mount 时自动聚焦
  ├─ handleClickFocus() → 鼠标点击聚焦
  └─ enable() / disable() → 暂停/恢复
```

**Box 组件的焦点属性**:
```typescript
<Box
  tabIndex={0}        // Tab 循环顺序（≥0）
  autoFocus           // 自动聚焦
  onClick={...}       // 点击事件
  onFocus={...}       // 获得焦点
  onBlur={...}        // 失去焦点
  onKeyDown={...}     // 键盘事件
  onMouseEnter={...}  // 鼠标进入
  onMouseLeave={...}  // 鼠标离开
/>
```

---

## 九、React Compiler 优化

Claude Code 的组件使用了 **React Compiler** 编译：

```typescript
// 编译后的组件模式
const $ = _c(N)  // 创建 N 个缓存槽

function MyComponent(props) {
  // 条件缓存
  if ($[0] === Symbol.for("react.memo_cache_sentinel")) {
    // 首次计算
    $[0] = expensiveComputation()
  }
  const cached = $[0]

  // 使用缓存值
  return <Text>{cached}</Text>
}
```

> **效果**: 避免闭包重新创建成本，减少不必要的重新渲染。相当于自动的 `useMemo`/`useCallback`，但更精细。

---

## 十、性能优化策略

| 策略 | 实现 | 效果 |
|------|------|------|
| 帧率节流 | ~60fps 调度 | 避免过度渲染 |
| 增量 Blit | 逐行对比，跳过未变更 | 减少 stdout 写入 |
| 样式池代际重置 | 分代 GC 样式对象 | 降低内存压力 |
| 字符缓存 | 帧间复用字符对象 | 减少分配 |
| 双缓冲 | front/back frame 交换 | 消除闪烁 |
| Dirty 标记 | 只重新布局脏节点 | 加速 Yoga 计算 |
| React Compiler | 自动 memo 化 | 减少重新渲染 |
| 延迟组件加载 | `lazy()` 导入 App/REPL | 加速启动 |

---

## 十一、关键设计模式总结

### 1. React + 终端 = 最佳组合
```
声明式 UI（React）+ 低级终端控制（ANSI/CSI/DEC/OSC）
= 既有 React 的开发效率，又有终端的精确控制
```

### 2. 极简 Store
```
不用 Redux/Zustand/MobX
一个 100 行的 Store + 引用等性检查
够用就好
```

### 3. 深度终端集成
```
不只是"在终端中打印文字"
而是: 鼠标追踪 + Alt-Screen + Kitty 键盘 + 超链接 + 文本选择
= 一个真正的终端原生 IDE 体验
```
