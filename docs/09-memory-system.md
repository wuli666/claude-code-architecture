# 09 - Memory 系统深度解析

> Claude Code 如何"记住你"——从分类、存储、写入到召回注入的完整链路

---

## 一、Memory 解决什么问题？

Claude Code 每次启动都是一个新进程，模型本身没有记忆。但用户希望：

- "上次我说了不要 mock 数据库，你怎么又 mock 了？"
- "我是做数据科学的，解释代码的时候别用前端术语"
- "这个项目下周四要 code freeze，别给我提大重构"

**Memory 系统的目标**：让 Claude Code 跨会话记住关于你、你的项目、你的偏好的信息，而且越用越懂你。

---

## 二、两套完全不同的东西：CLAUDE.md vs AutoMem

很多人把这两个混在一起，但它们是完全不同的机制：

```
CLAUDE.md（指令型）                    AutoMem（知识型）
├─ 人手写的                            ├─ AI 自动提取的
├─ 像项目的 README/规范                ├─ 像 AI 的笔记本
├─ 每次全量注入                        ├─ 按需选择性注入
├─ 内容: 规则、约定、偏好              ├─ 内容: 学到的事实、反馈、背景
└─ 你控制写什么                        └─ AI 决定记什么
```

**类比**：CLAUDE.md 像你给新同事的入职文档，AutoMem 像这个同事自己记的工作笔记。

---

## 三、分类体系

### 3.1 按来源分（谁写的、存在哪）

**源码**: `src/utils/memory/types.ts`

```
┌────────────┬─────────────────────────┬───────────────────────────────────┐
│ 类型        │ 谁写                   │ 存在哪                             │
├────────────┼─────────────────────────┼───────────────────────────────────┤
│ Managed    │ 企业 IT 管理员           │ /etc/claude-code/ 或 macOS plist │
│ User       │ 用户手写                │ ~/.claude/CLAUDE.md               │
│ Project    │ 团队手写                │ 项目根目录/CLAUDE.md              │
│ Local      │ 个人手写（gitignored）   │ CLAUDE.local.md                  │
├────────────┼─────────────────────────┼───────────────────────────────────┤
│ AutoMem    │ AI extraction agent     │ ~/.claude/projects/<路径>/memory/ │
│ TeamMem    │ AI extraction agent     │ 同上 /memory/team/               │
└────────────┴─────────────────────────┴───────────────────────────────────┘
```

上面四行是 **CLAUDE.md 系列**——人写的指令。
下面两行是 **AutoMem 系列**——AI 写的知识。

### 3.2 AutoMem 的语义分类（四类 taxonomy）

**源码**: `src/memdir/memoryTypes.ts`

AI 写记忆时不是随便写的，必须归入四类之一：

| 类型 | 一个问题概括 | 举例 |
|------|------------|------|
| **user** | 这个用户是谁？ | "用户是数据科学家，擅长 Python，正在学 Rust" |
| **feedback** | 我以后该怎么做？ | "不要 mock 数据库，因为上次 mock 导致线上事故" |
| **project** | 项目当前是什么状态？ | "下周四 code freeze，移动端要发版" |
| **reference** | 去哪查外部信息？ | "pipeline bug 在 Linear 的 INGEST 项目里" |

**关键设计**：这四类的核心原则是"**不存能从代码推出的东西**"。代码结构、Git 历史、文件路径——这些 Claude 可以 grep/git log 获取，不需要记忆。Memory 只存那些代码里看不到的信息。

**源码中的明确排除列表**（`memoryTypes.ts` WHAT_NOT_TO_SAVE_SECTION）：
```
不要存:
  ❌ 代码模式、架构、文件路径、项目结构
  ❌ Git 历史、最近改动
  ❌ Debug 方案、修复方法
  ❌ CLAUDE.md 里已经写了的
  ❌ 临时任务状态、当前会话上下文
```

### 3.3 feedback 类型为什么特殊

feedback 要求正文必须有 **Why** 和 **How to apply** 两行：

```markdown
---
name: Testing policy
description: Integration tests must use real database
type: feedback
---

Integration tests must hit a real database, not mocks.

**Why:** Prior incident where mock/prod divergence masked a broken migration.
**How to apply:** Any test file touching DB operations.
```

为什么？因为没有 Why 的规则会变成盲目遵守。有了 Why，模型在边界情况下可以自行判断"这条规则在这个场景下是否适用"。

---

## 四、存储：文件怎么落盘的

### 4.1 目录结构

```
~/.claude/projects/<sanitized-project-root>/memory/
├── MEMORY.md                  ← 索引文件（人可读的目录）
├── user_role.md               ← 一个主题 = 一个文件
├── feedback_testing.md
├── project_release_freeze.md
├── reference_linear_board.md
└── team/                      ← 团队共享记忆（可选）
    ├── MEMORY.md
    └── feedback_ci_policy.md
```

### 4.2 两种文件的区别

**MEMORY.md**（索引）：
```markdown
- [User role](user_role.md) — User is a data scientist focused on observability
- [Testing policy](feedback_testing.md) — Integration tests must hit a real database
- [Release freeze](project_release_freeze.md) — Non-critical merges freeze before mobile release
```

- 每行一条，不超过 150 字符
- 上限 200 行（超过会被截断）
- **始终全量注入** system prompt
- 不存正文，只存指针

**Topic 文件**（正文）：
```markdown
---
name: Testing policy
description: Integration tests must use real database
type: feedback
---

Integration tests must hit a real database, not mocks.

**Why:** Prior incident where mock/prod divergence masked a broken migration.
**How to apply:** Any test file touching DB operations.
```

- frontmatter 三个字段：name、description、type
- description 是**召回时的关键**——Sonnet 模型根据它判断要不要注入
- **不是每次都注入**，只有被选中才注入

### 4.3 为什么一个主题一个文件？

不用数据库，不用 JSON，直接用 Markdown 文件。好处：

1. 用户可以手动编辑（`vim ~/.claude/projects/.../memory/feedback_testing.md`）
2. Git 友好（能 diff、能 commit）
3. 文件名即语义（`user_role.md` 一看就懂）
4. extraction agent 用 Edit/Write 工具操作，和编辑源码一样

---

## 五、写入：AI 怎么提取记忆

### 5.1 什么时候触发

**源码**: `src/services/extractMemories/extractMemories.ts`

```
用户和 Claude 对话
  │
  模型返回最终回答（stop，没有更多工具调用）
  │
  ▼
handleStopHooks() → executeExtractMemories()
```

不是每条消息都触发，是**一轮对话结束时**才触发。

### 5.2 触发前的检查

```
executeExtractMemories()
  │
  ├─ feature flag 启用？
  ├─ 用户没禁用 auto memory？
  ├─ 不是远程模式？
  ├─ 距上次提取够了一定 turn 数？
  │
  ├─ 关键: 主 agent 自己写过 memory 了吗？
  │   └─ hasMemoryWritesSince()
  │       检查对话中是否有 Write/Edit 操作指向 memory 目录
  │       ├─ 有 → 跳过（避免重复提取）
  │       └─ 没有 → 继续
  │
  └─ 全部通过 → 启动 extraction agent
```

**互斥设计**：用户说"记住这个" → 主 agent 直接写文件 → extraction agent 检测到已经写过 → 跳过。防止重复。

### 5.3 Extraction Agent 是怎么工作的

这不是主模型自己提取的，而是**分叉出一个独立的子 agent**：

```
主对话 agent（Opus/Sonnet）
  │
  └─ runForkedAgent()  ← 完美分叉
      │
      ├─ 共享主 agent 的 Prompt Cache（零额外成本）
      ├─ 能看到整个对话历史
      ├─ 但只有受限的工具:
      │   ├─ ✅ Read / Grep / Glob（无限制）
      │   ├─ ✅ 只读 Bash（ls / cat / stat / head / tail）
      │   ├─ ✅ Edit / Write（仅限 memory/ 目录内）
      │   └─ ❌ 其他一切（不能写代码、不能调 MCP、不能派子 agent）
      │
      ├─ 最多 5 个 turn（防止它去"验证"记忆内容）
      │
      └─ 收到的 prompt:
          ├─ "分析最近 ~N 条消息"
          ├─ "只用消息内容更新记忆，不要去 grep 源码验证"
          ├─ "这是已有的记忆文件列表:（manifest）"
          ├─ "先 Read 要改的文件，再 Edit/Write"
          └─ "四种类型的定义和示例"
```

### 5.4 Manifest 预注入

**源码**: `src/memdir/memoryScan.ts`

提取前，系统先扫描已有的 memory 文件，生成一个 manifest 给 extraction agent 看：

```typescript
// scanMemoryFiles() 的逻辑:
// 1. 递归读 memory/ 目录下所有 .md 文件（排除 MEMORY.md）
// 2. 只读 frontmatter（前 30 行）
// 3. 提取: filename, description, type, mtimeMs
// 4. 按修改时间倒序，最多 200 个
```

Manifest 长这样：
```
- [feedback] feedback_testing.md (2026-04-01T10:00:00Z): Integration tests must use real database
- [user] user_role.md (2026-03-28T14:00:00Z): User is a data scientist
- [project] project_release_freeze.md (2026-03-25T09:00:00Z): Merge freeze for mobile release
```

**为什么要预注入**？省 turn。如果不给 manifest，agent 第一个 turn 就要 `ls memory/` + 逐个 `cat` frontmatter，白白浪费 2-3 个 turn（总共只有 5 个）。

### 5.5 高效写入策略

Prompt 中明确教 agent 的策略：

```
Turn 1: 并行 Read 所有要改的文件
Turn 2: 并行 Edit/Write 所有改动
→ 2 个 turn 搞定，剩 3 个 turn 余量
```

不要一个个文件串行 Read→Edit→Read→Edit，那样 5 个 turn 改不了几个文件。

---

## 六、召回：怎么选出相关的记忆

### 6.1 预取机制

**源码**: `src/memdir/findRelevantMemories.ts`, `src/utils/attachments.ts`

```
用户输入新消息
  │
  ▼
startRelevantMemoryPrefetch()  ← 非阻塞，和主模型并行跑
  │
  │  同时...
  │
  ▼                           ▼
Sonnet 选记忆              主模型开始处理用户请求
  │                           │
  └─ 选完了 ─────────────────→ 下一次 API 调用时注入
```

**关键设计**：记忆选择和主模型处理**并行执行**。Sonnet 很快（几百毫秒），通常在主模型的第一个 API 调用返回前就选完了。

### 6.2 选择过程

```
startRelevantMemoryPrefetch()
  │
  ├─ Gate 检查:
  │   ├─ feature flag 启用？
  │   ├─ auto memory 启用？
  │   ├─ 不是单词查询？（上下文太少，无法选择）
  │   └─ 会话累计注入量 ≤ 60KB？（预算控制）
  │
  ├─ 扫描: scanMemoryFiles()
  │   └─ 读 ≤200 个文件的 frontmatter
  │
  ├─ 去重: 过滤本会话已经注入过的文件
  │
  ├─ Sonnet 选择: sideQuery() ← 独立的 API 调用
  │   │
  │   ├─ 不是主模型在选，是另一个 Sonnet 模型
  │   ├─ 输入:
  │   │   ├─ 用户的问题
  │   │   ├─ Memory manifest（文件名 + 描述 + 类型）
  │   │   └─ 最近用了哪些工具（避免推荐已在用的工具的参考文档）
  │   ├─ 输出: ≤5 个文件名
  │   └─ 规则: "只选你确信有用的，不确定就不选"
  │
  ├─ 读取: readMemoriesForSurfacing()
  │   ├─ 每文件上限: 200 行 / 4KB
  │   └─ 如果截断: 附注 "完整文件在 <path>"
  │
  └─ 包装: 创建 relevant_memories attachment
```

### 6.3 为什么用 Sonnet 做选择而不是关键词匹配？

因为关键词匹配太蠢。比如：

```
用户: "帮我写个数据库迁移测试"

关键词匹配:
  "数据库" → 匹配了 reference_database_dashboard.md ← 不相关
  "测试" → 匹配了 user_role.md（里面提到"正在学测试"）← 勉强相关

Sonnet 选择:
  看到 feedback_testing.md 描述是 "Integration tests must use real database"
  → 直接命中 ← 这才是用户需要的
```

Sonnet 理解语义，能从 description 判断相关性，比关键词精确得多。

### 6.4 预算控制

两个层级的预算防止 memory 吞掉上下文窗口：

```
单文件: ≤ 4KB（截断 + 提示路径）
         → 防止一个超长记忆文件占满空间

会话累计: ≤ 60KB（所有 relevant_memories attachment 加起来）
          → 超过后整个会话不再注入新记忆
          → compact 时重置计数器
```

---

## 七、注入：Memory 怎么进入 API 请求

### 7.1 三条管道

```
发送给 Claude API 的请求:

┌─ system prompt ─────────────────────────────────────────┐
│                                                          │
│  "## When to access memories"                            │  ← 管道 1: 使用规则
│  "## Before recommending from memory"                    │     (始终存在)
│  '"The memory says X exists" is not the same as          │
│   "X exists now."'                                       │
│                                                          │
└──────────────────────────────────────────────────────────┘

┌─ messages[0] (meta user message) ────────────────────────┐
│                                                          │
│  <system-reminder>                                       │  ← 管道 2: CLAUDE.md + 索引
│  # claudeMd                                              │     (每次全量注入)
│  [CLAUDE.md 所有层级合并内容]                              │
│  [MEMORY.md 索引内容]                                     │
│  Today's date is 2026-04-08.                             │
│  </system-reminder>                                      │
│                                                          │
└──────────────────────────────────────────────────────────┘

┌─ messages[1] (meta user message) ────────────────────────┐
│                                                          │
│  <system-reminder>                                       │  ← 管道 3: 选中的记忆正文
│  Memory (saved 3 days ago): feedback_testing.md          │     (按需选择性注入)
│                                                          │
│  ---                                                     │
│  name: Testing policy                                    │
│  description: Integration tests must use real database   │
│  type: feedback                                          │
│  ---                                                     │
│                                                          │
│  Integration tests must hit a real database, not mocks.  │
│  **Why:** Prior incident where mock/prod divergence...   │
│  </system-reminder>                                      │
│                                                          │
└──────────────────────────────────────────────────────────┘

┌─ messages[2] (真正的用户消息) ───────────────────────────┐
│                                                          │
│  帮我写一个数据库迁移的测试                                │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### 7.2 为什么分三条管道？

```
管道 1 (system prompt):
  → 规则，不是内容
  → 告诉模型"怎么对待 memory"，而不是"memory 里有什么"
  → 始终存在，不占选择成本

管道 2 (user context):
  → CLAUDE.md 是项目规则，每次都需要
  → MEMORY.md 索引让模型知道"有哪些记忆文件可以查"
  → 全量注入，不做选择

管道 3 (attachment):
  → 具体的记忆正文
  → Sonnet 选过的才注入（≤5 条）
  → 按需注入，节省 Token
```

### 7.3 `<system-reminder>` 标签的作用

所有 memory 内容都包装在 `<system-reminder>` 标签中。这个标签的作用是：

1. 模型知道这是系统注入的上下文，不是用户说的话
2. 不会被模型当成对话历史的一部分
3. 前端可以选择性地隐藏这些消息

---

## 八、Memory 幻觉——最精妙的设计

**源码**: `memoryTypes.ts` TRUSTING_RECALL_SECTION

Memory 系统最有趣的部分不是"怎么记"，而是"怎么不盲信"。

### 问题

```
3 天前存的记忆: "项目里有 getUserProfile() 函数在 src/utils/auth.ts"
今天的现实:     这个函数昨天被重命名为 fetchUserData() 了
```

如果模型盲信记忆，就会推荐一个不存在的函数。

### 解决方案

System prompt 中明确要求：

```
"The memory says X exists" is not the same as "X exists now."

Before recommending from memory:
- If the memory names a file path: check the file exists.
- If the memory names a function or flag: grep for it.
- If the user is about to act on your recommendation, verify first.
```

**记忆不是事实源（source of truth），是线索源（source of hints）。**

这和人类记忆很像——你记得某个文件在某个位置，但得打开看一眼确认它还在那儿。

### 陈旧检测

注入记忆时会附带时间戳：
```
Memory (saved 3 days ago): feedback_testing.md
Memory (saved 47 days ago): project_release_freeze.md
```

模型看到"47 days ago"会自然地降低信任度——一个快两个月前的项目状态可能早就变了。

---

## 九、Dream——跨会话记忆整合

**源码**: `src/services/autoDream/autoDream.ts`

除了每次 turn 结束时的 extraction，还有一个更慢的后台整合机制：

```
触发条件（全部满足）:
  ├─ 距上次整合 ≥ 24 小时
  ├─ 自上次整合后 ≥ 5 个新会话
  └─ PID 文件锁可获取（防并发）

整合行为:
  ├─ 读取多个会话的记忆
  ├─ 合并重复条目
  ├─ 更新过时信息
  └─ 优化 MEMORY.md 索引

失败处理:
  └─ rollbackConsolidationLock() → 回退 mtime → 下次重试
```

**类比**：extraction 像你每天记的零散笔记，Dream 像你周末把笔记整理归档。

---

## 十、完整数据流总览

```
═══════════════ 写入路径 ═══════════════

对话结束
  │
  ▼
executeExtractMemories()
  ├─ 检查 gate + 互斥
  ├─ 扫描已有文件 → manifest
  ├─ 分叉 extraction agent
  │   ├─ 工具受限: Read/Grep/Glob + 只读 Bash + memory 内 Edit/Write
  │   ├─ 最多 5 turn
  │   └─ 策略: Turn 1 并行 Read → Turn 2 并行 Write
  └─ 文件落盘: memory/*.md + 更新 MEMORY.md


═══════════════ 召回路径 ═══════════════

用户输入
  │
  ├─ 并行 ──→ Sonnet sideQuery（选记忆）
  │            ├─ 读 manifest
  │            ├─ 匹配用户问题
  │            └─ 返回 ≤5 个文件名
  │
  ▼
主模型 API 调用
  ├─ system: memory 使用规则
  ├─ user[0]: CLAUDE.md + MEMORY.md 索引
  ├─ user[1]: 选中的记忆正文 ← Sonnet 结果注入
  └─ user[2]: 真正的用户消息


═══════════════ 整合路径 ═══════════════

后台定时（24h + 5 sessions）
  │
  ▼
Dream agent
  ├─ 合并重复
  ├─ 清理过时
  └─ 优化索引
```

---

## 十一、设计模式总结

| 模式 | 实现 | 为什么 |
|------|------|--------|
| **文件即数据库** | 每个记忆是一个 .md 文件 | 可 diff、可手动编辑、可 Git |
| **分叉写入** | 独立 agent 提取，不是主模型 | 不占主模型 Token，工具受限更安全 |
| **旁路召回** | Sonnet sideQuery 并行预取 | 不阻塞主模型，更快更便宜 |
| **预算控制** | 4KB/文件，60KB/会话 | 防止 memory 吞上下文窗口 |
| **线索非事实** | "verify before recommending" | 防止 memory 幻觉 |
| **语义选择** | Sonnet 按 description 选 | 比关键词匹配精确 |
| **互斥写入** | 主 agent 写过则跳过 extraction | 防重复 |
| **manifest 预注入** | 告诉 agent 已有什么 | 省 turn |
