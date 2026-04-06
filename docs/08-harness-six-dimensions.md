# 08 - Harness 六维评估

> 以 AI Agent Harness 的六大核心维度评估 Claude Code 的架构成熟度

---

## 评估总览

```
┌──────────────┬────────┬────────────────────────────────────────┬──────────────────────────────────────┐
│     模块     │ 成熟度 │              核心实现                   │              主要缺口                │
├──────────────┼────────┼────────────────────────────────────────┼──────────────────────────────────────┤
│ ① 上下文工程 │ ★★★★★  │ 三级 Compaction + Memory 系统          │ Memory 检索靠关键词，无向量语义检索   │
│              │        │ + Prompt Cache 复用 + CLAUDE.md 层级    │                                      │
├──────────────┼────────┼────────────────────────────────────────┼──────────────────────────────────────┤
│ ② 工具编排   │ ★★★★☆  │ 43 工具 + 并发标记 + 子 Agent          │ 无工具依赖声明图                     │
│              │        │ + Worktree 隔离 + 延迟工具池            │ 执行顺序完全靠模型自行决定            │
├──────────────┼────────┼────────────────────────────────────────┼──────────────────────────────────────┤
│ ③ 验证机制   │ ★★★☆☆  │ Verification Agent（对抗式测试）       │ 不在生成循环内                       │
│              │        │ + 多级错误恢复 + Hook 反馈              │ 无自动修复循环、无生成-评估分离       │
├──────────────┼────────┼────────────────────────────────────────┼──────────────────────────────────────┤
│ ④ 状态管理   │ ★★★☆☆  │ Session 持久化 + /rewind 回退          │ 无自动检查点、无 auto-commit          │
│              │        │ + DreamTask 记忆整合 + Shell 快照       │ 回退靠用户手动触发                    │
├──────────────┼────────┼────────────────────────────────────────┼──────────────────────────────────────┤
│ ⑤ 可观测性   │ ★★★☆☆  │ OpenTelemetry 全链路 + BigQuery 导出   │ 无质量评分、无异常检测                │
│              │        │ + 分布式 Span 追踪 + 成本归因           │ 无自动归因分析                       │
├──────────────┼────────┼────────────────────────────────────────┼──────────────────────────────────────┤
│ ⑥ 人类接管   │ ★★★★★  │ 五级权限模式 + Hook 阻断 + 沙箱        │ 基本完整                             │
│              │        │ + 竞速决策 + 分类器降级 + MDM 策略      │                                      │
└──────────────┴────────┴────────────────────────────────────────┴──────────────────────────────────────┘
```

---

## ① 上下文工程 — ★★★★★

**Claude Code 在这个维度是行业标杆。**

### 能力矩阵

| 能力 | 实现方式 | 源码位置 |
|------|---------|---------|
| 动态 System Prompt | 基础提示 + Git Status + CLAUDE.md 多级合并 + 日期注入 | `context.ts` |
| 三级压缩 | Micro → Auto → Reactive 三级递进 | `services/compact/` |
| Prompt Cache | `ttl: '1h'` + `scope: 'global'` + 分叉共享 | `services/api/claude.ts` |
| Session Memory | 对话中周期提取（10K Token 初始化，5K 更新间隔） | `services/SessionMemory/` |
| Auto-Memory | 对话结束时一次性提取持久化信息 | `services/extractMemories/` |
| Dream 整合 | 跨会话记忆整合（24h + 5 session 门控 + PID 锁） | `services/autoDream/` |
| 延迟工具池 | ToolSearch 按需加载大体积工具定义 | `tools/ToolSearchTool/` |
| 上下文折叠 | Context Collapse 读投影 + 批量折叠 | `services/compact/` |

### 三级 Compaction 详解

```
Token 使用量增长 →

├─ Microcompact（轻量修剪）
│   ├─ 时间型: 距上次助手消息间隔过大 → 清除旧工具结果
│   │   替换为 "[Old tool result content cleared]"
│   └─ 缓存型: cache_edits API → 服务端删除旧缓存，本地不修改
│   影响: 几乎无感，不丢失语义
│
├─ Auto-compact（完整摘要）
│   ├─ 阈值: contextWindow - maxOutputTokens - 13,000
│   ├─ 行为: 全部消息摘要重写，保留边界标记
│   ├─ 熔断: MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3
│   └─ 抑制: session_memory/compact 查询、context-agent 内不触发
│   影响: 丢失细节但保留要点
│
└─ Reactive compact（紧急压缩）
    ├─ 触发: API 返回 prompt_too_long 错误
    ├─ 级联: Collapse Drain → Reactive Compact → 截断头部
    └─ 策略: 先尝试恢复，恢复失败再暴露错误
    影响: 有损但保证不崩溃
```

### Memory 三层架构

```
Session Memory（短期）
  ├─ 当前对话的实时笔记
  ├─ 触发: Token ≥ 10,000 且距上次 ≥ 5,000
  └─ 分叉子 Agent 后台提取

Auto-Memory（中期）
  ├─ 对话结束时提取持久化信息
  └─ 存储: memory/ 目录（Markdown + MEMORY.md 索引）

Dream 整合（长期）
  ├─ 跨会话记忆整合优化
  ├─ 24h + 5 session 双门控 + PID 锁
  └─ 失败时 rollbackConsolidationLock() 回退
```

### 缺口

- Memory 检索是纯文件名/关键词匹配（`MEMORY.md` 索引），无向量语义检索
- 记忆条目多时可能出现相关记忆不被召回
- 无记忆衰减机制（旧记忆不会降低优先级）

---

## ② 工具编排 — ★★★★☆

### 能力矩阵

| 能力 | 实现方式 | 源码位置 |
|------|---------|---------|
| 工具数量 | 43+ 内置 + MCP 动态扩展 | `tools.ts` |
| 并发控制 | `isConcurrencySafe` 逐工具声明 | `Tool.ts` |
| 子 Agent | 5 种内置 + 自定义 Agent，分叉共享缓存 | `tools/AgentTool/` |
| 物理隔离 | Git Worktree + tmux session | `setup.ts` |
| 延迟发现 | ToolSearch 关键词/精确匹配双模式 | `tools/ToolSearchTool/` |
| Coordinator | 多 Worker 编排：研究→合成→实施→验证 | `coordinator/` |
| 行为标记 | isReadOnly / isDestructive / interruptBehavior | `Tool.ts` |

### 并发模型

```typescript
// 每个工具自声明并发安全性
Grep, Glob, FileRead, WebFetch  → isConcurrencySafe: true   // 可并发
Bash, FileEdit, Agent            → isConcurrencySafe: false  // 必须串行

// 默认值: false（失败关闭 — 新工具忘记设置则串行执行）
```

### Coordinator 模式

```
Coordinator（只有 Agent + SendMessage + TaskStop 工具）
  │
  ├─── Worker A（读取 — 可并行）
  ├─── Worker B（写入 — 必须串行）
  └─── Worker C（验证 — 按文件区域独立）

四阶段: Research（并行）→ Synthesis → Implementation（串行）→ Verification
```

### 延迟工具池

```
初始 Prompt: 30 个工具定义（常用工具）
模型需要时: ToolSearch("web fetch")
  ├─ 精确: "select:WebFetch" → 直接加载
  └─ 关键词: 完整匹配 10 分, 子串 5 分, searchHint 4 分
→ 每次调用节省几千 Token
```

### 缺口

- **无工具依赖图** — 不能声明 "Edit 依赖 Read"，靠模型推理
- **无执行计划优化** — 不会分析最优执行顺序
- **无批量事务** — 不能声明 "这 3 个 Edit 是一个原子操作"
- 但 `validateInput` 兜底：未读就编辑 → 拒绝，保证安全

---

## ③ 验证机制 — ★★★☆☆

> Claude Code 最大的成长空间。

### 能力矩阵

| 能力 | 实现方式 | 源码位置 |
|------|---------|---------|
| Verification Agent | 对抗式独立验证，PASS/FAIL/PARTIAL | `tools/AgentTool/built-in/verificationAgent.ts` |
| 三级错误恢复 | Collapse Drain → Reactive Compact → Max Tokens 调整 | `query.ts` L1086-1199 |
| 错误扣留 | withheldByCollapse / withheldByReactive | `query.ts` L1065-1183 |
| Hook 反馈 | PostToolUse / PostToolUseFailure / Stop Hooks | `utils/hooks.ts` |
| 熔断器 | 连续 3 次 autocompact 失败 → 停止 | `services/compact/autoCompact.ts` |
| 模型降级 | 连续 3 次 529 → 切备选模型 | `services/api/withRetry.ts` |
| 乐观并发 | FileEdit 过期检测（timestamp + content） | `tools/FileEditTool/` |

### Verification Agent

```
门控: feature('VERIFICATION_AGENT') + GrowthBook tengu_hive_evidence

提示词核心: "你是对抗式验证者。目标是尝试破坏实现，而不是确认它能工作。"

触发条件（由主 Agent 判断）:
  ├─ 3+ 文件编辑
  ├─ 后端/API 变更
  └─ 非平凡任务

输出: VERDICT: PASS / FAIL / PARTIAL + 证据
```

### 三级错误恢复

```
API 返回错误
  │
  ▼
Stage 1: Collapse Drain — 排空已暂存的上下文折叠
  │ 失败
  ▼
Stage 2: Reactive Compact — 全对话摘要压缩 + 剥离图片
  │ 失败（hasAttemptedReactiveCompact 门控，只试一次）
  ▼
Stage 3: Max Tokens 调整 — 解析错误消息，动态调小 max_tokens
  │ 全部失败
  ▼
CannotRetryError → 暴露给用户
```

### 错误扣留模式

```
关键设计: 不立即暴露错误，先尝试恢复

prompt_too_long → withheldByCollapse = true → 先试 collapse drain
media_too_large → withheldByReactive = true → 先试 reactive compact
恢复成功 → 用户完全无感
恢复失败 → 才暴露错误
```

### 缺口：与理想架构的差距

```
当前 Claude Code:
  生成代码 → 执行完毕 → [可选] 手动调 Verification Agent → 报告结果
                              ↑ 需要模型主动决定调用
                              ↑ 同一模型自己验证

理想架构:
  Generator → Executor → Evaluator ──┐
       ↑                              │ 失败
       └────── 自动反馈修复 ──────────┘

缺失:
  1. 无自动测试执行 — 不会自动 npm test / pytest
  2. 无修复循环 — 验证失败后不自动把失败原因喂回重试
  3. 无独立评估器 — 没有用不同模型做独立评估
  4. 无置信度分数 — 不会输出 "我对这次修改 80% 有信心"
```

---

## ④ 状态管理 — ★★★☆☆

### 能力矩阵

| 能力 | 实现方式 | 源码位置 |
|------|---------|---------|
| Session 持久化 | `~/.claude/history.jsonl` JSON-L，100 条上限 | `history.ts` |
| /rewind 回退 | 用户手动回退到对话中任意消息点 | `commands/rewind/` |
| Shell 快照 | 启动时捕获完整 shell 环境（10s 超时） | `utils/bash/ShellSnapshot.ts` |
| Config 快照 | Hook 配置启动时冻结 | `utils/hooks/hooksConfigSnapshot.ts` |
| DreamTask | 后台记忆整合 + PID 锁 + mtime 回退 | `services/autoDream/` |
| Worktree | Git worktree 物理隔离副本 | `setup.ts` |
| 成本持久化 | 按会话保存到项目配置 | `cost-tracker.ts` |

### 快照体系

```
启动时捕获:
  Shell Snapshot — aliases, functions, PATH（10s 超时）
  Hooks Config  — 冻结 Hook 配置，防中途修改影响
  Teammate Mode — auto / tmux / in-process 模式快照

运行时:
  readFileState — 文件 {content, timestamp} 缓存（乐观并发基础）
  Task 输出    — 每个后台任务输出写入独立文件
```

### /rewind 机制

```
/rewind (别名: /checkpoint)
  → 消息选择器 UI → 截断消息列表到该点 → 继续对话

限制: 只回退对话状态，不回退文件系统
```

### 缺口

```
缺失 1: 无自动检查点
  理想: 每批工具执行后自动保存 {对话快照 + 文件差异}

缺失 2: 无 auto-commit
  理想: 重要变更后自动 git commit（WIP），验证失败可 git reset

缺失 3: 无事务性
  理想: BEGIN → 多个编辑 → COMMIT/ROLLBACK

缺失 4: /rewind 对话与文件不一致
  理想: /rewind 同时回退文件到对应状态
```

---

## ⑤ 可观测性 — ★★★☆☆

> 基础设施是生产级的，但"智能"层几乎为零。

### 能力矩阵

| 能力 | 实现方式 | 源码位置 |
|------|---------|---------|
| OpenTelemetry | Trace + Metric + Log 三大支柱 | `utils/telemetry/instrumentation.ts` |
| 分布式追踪 | Interaction → LLM → Tool → Hook 四层 Span | `utils/telemetry/sessionTracing.ts` |
| BigQuery 导出 | 企业/团队用户指标导出 | `utils/telemetry/bigqueryExporter.ts` |
| 成本归因 | 按模型分 input/output/cache_read/cache_creation | `cost-tracker.ts` |
| 活动统计 | 日热力图、连续天数、峰值分析 | `utils/stats.ts` |
| 性能剖析 | 20+ profileCheckpoint + FPS + Perfetto | 启动流程各处 |
| PII 安全 | 类型级标记 + `_PROTO_*` 前缀 + 按通道剥离 | `services/analytics/index.ts` |

### 指标对比

```
有的（确定性指标）:
  ✅ Token 计数: input / output / cache_read / cache_creation
  ✅ 成本: USD 精确到分
  ✅ 耗时: API / 工具 / 总时长 / 不含重试
  ✅ 代码行: lines_added / lines_removed
  ✅ 会话: count / 连续天数 / 峰值小时
  ✅ UI: FPS 帧率 / 渲染耗时

没有的（智能指标）:
  ❌ 质量评分: "这次代码质量 B+"
  ❌ 异常检测: "连续 5 次 Edit 被拒绝"
  ❌ 用户归因: "用户 revert 3 次 → 方向不对"
  ❌ 预测: "类似任务平均需要 8 轮工具调用"

验证: 全局搜索 quality|score|rating|grade → 零结果
```

### Span 层次

```
Session
  └─ Interaction（用户请求→Claude 响应，序列号递增）
       ├─ LLM Request（model, tokens, thinking）
       ├─ Tool Execution / Tool Blocked on User
       └─ Hook Execution
```

### 缺口

有管道没智能——OpenTelemetry 基础设施已是生产级（BigQuery、Perfetto），但上面没有质量评分、异常检测、归因分析。"只差最后一公里"。

---

## ⑥ 人类接管 — ★★★★★

> 所有 AI Agent 工具中最完善的。

### 能力矩阵

| 能力 | 实现方式 | 源码位置 |
|------|---------|---------|
| 五级权限模式 | default / acceptEdits / plan / bypass / auto | `utils/permissions/PermissionMode.ts` |
| 规则引擎 | allow/deny/ask + 通配符 + 6 级来源优先级 | `utils/permissions/` |
| Hook 阻断 | PermissionRequest Hook: allow/deny/block | `utils/hooks.ts` |
| 竞速决策 | 本地 / CCR / Channel / Hook / 分类器并行 | `hooks/toolPermission/handlers/` |
| 分类器降级 | 连续 3 次拒绝 → 用户询问，总 20 次 → 降级 | `utils/permissions/autoModeState.ts` |
| 沙箱 | 文件系统 R/W 白名单 + 网络域名白名单 | `utils/sandbox/` |
| MDM 策略 | 文件 / plist / Registry / 远程策略 | `utils/settings/mdm/` |
| 危险剥离 | Auto 模式自动移除代码解释器 Allow 规则 | `utils/permissions/dangerousPatterns.ts` |
| 原子决策 | `claim()` 防并行竞速多重获胜 | `hooks/toolPermission/PermissionContext.ts` |

### 权限决策流程

```
工具请求 → ① Allow 规则匹配? → ✅ 通过
         → ② Deny 规则匹配?  → ❌ 拒绝
         → ③ Ask 规则匹配?   → ❓ 强制询问
         → ④ 危险模式过滤（Auto 模式剥离代码解释器）
         → ⑤ 分类器（Auto 模式，超限降级）
         → ⑥ 交互式对话（竞速决策）
```

### 竞速决策架构

```
并行启动:
  用户本地对话框 ─┐
  CCR Bridge     ─┤
  Channel        ─┼─→ claim() 原子竞争 → 第一个获胜
  Hooks          ─┤
  分类器          ─┘

200ms 宽限期: 分类器指示器延迟显示，避免 UI 闪烁
自动通过: 终端聚焦 3s / 失焦 1s ✅，Esc 关闭
```

### 六级规则来源

```
1. CLI Arguments         最高优先级
2. In-Session Changes
3. Local Settings        .claude/settings.local.json（gitignored）
4. Project Settings      .claude/settings.json（共享）
5. User Settings         ~/.claude/settings.json（全局）
6. Managed/Policy        /etc/ / plist / Registry    最低优先级
```

### 缺口

基本完整。极少数 edge case：
- 无时间维度权限（"仅工作时间允许"）
- 无上下文动态权限（"仅在 test 目录允许 rm"）

---

## 雷达图

```
          上下文工程
            ★★★★★
              │
    人类接管   │   工具编排
    ★★★★★ ────┼──── ★★★★☆
              │
    可观测性   │   验证机制
    ★★★☆☆ ────┼──── ★★★☆☆
              │
          状态管理
           ★★★☆☆
```

---

## 核心洞察

**1. 两极分化明显**

上下文工程和人类接管 ★★★★★，验证/状态/可观测 ★★★☆☆。前两者直接决定产品可用性（上下文 = AI 能力上限，接管 = 安全下限），优先级最高。

**2. 验证是最大增长点**

Verification Agent 是"可选的、事后的、同模型的"。进化为"自动的、内联的、独立模型的"修复循环后，能力会有质的飞跃。

**3. 可观测性"有管道没智能"**

OpenTelemetry + BigQuery + Perfetto 已是生产级基础设施，但上面没有质量评分/异常检测/归因分析。只差最后一公里。

**4. 状态管理的哲学选择**

选择"快照式"而非"事务式"——捕获环境但不保证一致性回退。这是对复杂度的有意取舍：文件系统事务很难做对，不如让用户自己 `git stash` / `git reset`。
