# 上下文经济学：Cache、ToolSearch 与延迟加载的合一架构

> 基于 `@anthropic-ai/claude-code@2.1.88` 源码分析
> 源码路径: `restored-src/src/`
> 横切章节：本章覆盖 [02-tool-system](./02-tool-system.md)、[05-extension-system](./05-extension-system.md)、[09-memory-system](./09-memory-system.md) 三章共有的"上下文经济"主题，从 prompt cache 的视角重构对 ToolSearch / 延迟加载 / Skills frontmatter 三个机制的理解。

---

## 一、为什么单独开一章

前面 02/05/09 三章已经分别讲清了 Tool 系统、扩展系统、Memory 系统的**机制**——"怎么做"。但有一个跨章节的核心问题没有被显式回答：

> **为什么 Claude Code 要这样做？**

答案是 **Anthropic Prompt Cache 的经济模型**。所有那些看起来"很复杂"的设计——deferred 工具、tool_reference 协议、Skills frontmatter-only 加载、`cache_control` breakpoint——本质上都在解同一个问题：**让缓存命中率最大化、缓存失效范围最小化**。

理解这一章后，再回头读 02/05/09，会从"功能手册"读成"架构论证"。

---

## 二、三层冷启动

"冷启动"在 LLM agent 场景指三种完全不同的开销，必须分清楚才能讨论优化方向。

### 2.1 Token 冷启动 — Context 窗口的占座成本

```
问题：50 个 MCP 工具 × 1k schema = 50k token 占着窗口
即便不调用，这些 token 也永远在 system prompt 里
```

这是最直观的成本：上下文窗口是稀缺资源，工具 schema 占满了，真正的"工作记忆"（用户代码、日志、对话历史）就被挤压。

**对策（CC 实现）**：deferred 工具**只暴露 name**（约 5-10 token/个，连描述都不给），完整 schema 按需加载。详见 §四。

### 2.2 Latency 冷启动 — 首次发现工具的延迟

```
问题：模型必须先调 ToolSearch 才能用新工具，多一轮往返
代价：约 300-400ms（一次模型推理 + 检索 + 服务端展开）
```

这是用户能感知的延迟。每次模型遇到"未加载的工具"，都必须先发起一次工具搜索，等结果回来才能真正调用工具。

**对策（CC 实现）**：
- 高频/通信类工具加 `alwaysLoad: true` 白名单，跳过搜索（`tools/ToolSearchTool/prompt.ts:62-108`）
- `getToolDescriptionMemoized()` 缓存搜索结果（`ToolSearchTool.ts:66`）
- 工具描述里塞同义词，BM25/关键词检索一次命中

### 2.3 Cache 冷启动 — Prompt Cache 失效（最阴险的一层）

```
问题：Anthropic prompt cache 是连续前缀匹配
后果：tools 数组、system prompt、messages 历史任何位置变动
      → 那个位置之后的所有 token 全部 cache miss
价格：cache miss 比 hit 贵 12.5x（$3.75 vs $0.30 / 1M）
```

这是延迟加载架构成立的真正理由。**不是省 token，是保前缀缓存命中率**。

**对策（CC 实现）**：见 §五、§六。

### 2.4 三层冷启动的优先级

| 层 | 用户感受 | 频率 | 优化收益 |
|---|---|---|---|
| Token | 窗口被占 | 每次请求 | 中（影响其他内容） |
| Latency | 等待 | 每次首调用工具 | 高（用户直接感知） |
| **Cache** | **看不见但贵 + 慢** | **每次工具集变动** | **极高（钱 + latency 复合）** |

`💡 关键洞察`：Token 是窗口经济，Latency 是体验经济，**Cache 是钱经济 + 长期 latency 经济**。第三层是 production agent 必须重视的。

---

## 三、Anthropic Prompt Cache 的本质

### 3.1 cache_control 是"沿前缀切的刀"

**关键转一下脑子**：`cache_control` 不是说"这段被缓存"，而是说"**到这里为止的所有内容，合起来形成一个缓存单元**"。

API 请求示例：

```typescript
{
  "system": [
    { "type": "text", "text": "你是助手..." },          // block 0
    { "type": "text", "text": "工具说明...",
      "cache_control": { "type": "ephemeral", "ttl": "1h" } }  // BR1 ← 切一刀
  ],
  "tools": [...],
  "messages": [
    { "role": "user", "content": [
      { "type": "text", "text": "用户问题",
        "cache_control": { "type": "ephemeral", "ttl": "5m" } }  // BR2 ← 又切一刀
    ]}
  ]
}
```

服务端看到 2 个 breakpoint，**生成 2 个 cache entry**：

```
cache_A = hash(从 token 0 到 BR1 的所有内容)，TTL 1h
cache_B = hash(从 token 0 到 BR2 的所有内容)，TTL 5m
```

注意 cache_B 是**包含 cache_A 内容的更长前缀**。这是嵌套关系，不是并列。

### 3.2 命中逻辑

```
下一轮请求：
  服务端从前往后逐块匹配 cache_control 标记
  → 匹配到 BR1：cache_A 命中 → 按 cache_read 价格($0.30/1M)
  → 匹配到 BR2：cache_B 命中 → 同上
  → BR2 之后的新内容：miss → 按正常 input ($3/1M) 处理
```

**实际计费效果**：1M token 的请求里，如果 950k 是 cache 命中，只有 50k 是新内容，**总价约 $0.435**（vs 全 miss $3）。

### 3.3 硬限制：每个请求最多 4 个 breakpoint

这是 Anthropic 的硬限制，逼着你做架构选择。CC 怎么花这 4 个：

```
BR1 (1h):  system 末尾 ──────── 极稳定段，长 TTL
BR2 (1h):  tools 末尾 ────────── 稳定段(只有 eager 工具)
BR3 (5m):  对话历史中段 ──────── turn 间隔通常 < 5min
BR4 (5m):  当前消息末尾 ──────── 立刻就被下一轮覆盖
```

每个 breakpoint 都是一个"防护栅栏"：栅栏之前的内容如果完全一致，就命中长 TTL 的缓存；栅栏之后的变化只影响后续 breakpoint。

### 3.4 TTL 选择的经济学

| TTL | 写入价格 | 读取价格 | 适用场景 |
|---|---|---|---|
| **5min**（默认） | 1.25× base | 0.1× base | 密集对话 |
| **1h**（opt-in） | 2× base | 0.1× base | 间歇使用 |

**关键特性：TTL 是 sliding**——每次缓存命中，TTL 会重置刷新。"5 分钟"实际是"距上次使用 5 分钟"。

**反直觉计算**：1 小时内 10 次查询，平均间隔 6 分钟

```
5min TTL 的命运（间隔 6min > 5min，每次都过期）：
  第 1 次：写入 $0.0375
  第 2-10 次：每次都 miss + 重写 = $0.0375 × 9 = $0.3375
  总计：$0.375

1h TTL 的命运（6min < 1h，每次都命中 + 刷新 TTL）：
  第 1 次：写入 $0.060（贵 60%）
  第 2-10 次：每次命中 = $0.003 × 9 = $0.027
  总计：$0.087

→ 1h TTL 省 77%
```

`💡 关键洞察`：**"贵 2 倍的 1h TTL，反而比便宜的 5min 总成本低"**——因为命中的次数足够多，长 TTL 的写入溢价被多次低价读取摊平。**写入价格不是关键，命中率才是**。

---

## 四、tools 数组永远不变的秘密

`@02-tool-system § 5.4` 讲了 ToolSearchTool 的两种模式（精确选择 + 关键词搜索）。但有一个细节没有展开：**deferred 工具加载之后，到底去了哪里？**

### 4.1 错误的直觉

很容易以为：deferred 工具加载后，schema 被追加到 tools 数组（Codex 就是这样做的）。

**这个直觉是错的。**

### 4.2 源码证据

`tools/ToolSearchTool/prompt.ts:44` 描述：

> *"Until fetched, only the name is known — there is no parameter schema, so the tool cannot be invoked. ... Once a tool's schema appears in that result, it is callable exactly like any tool defined at the top of the prompt."*

`tools/ToolSearchTool/ToolSearchTool.ts:462-469` 返回值：

```typescript
return {
  type: 'tool_result',
  tool_use_id: toolUseID,
  content: content.matches.map(name => ({
    type: 'tool_reference' as const,
    tool_name: name,
  })),
}
```

`utils/messages.ts:1912`：

> *"When a tool_result contains tool_reference, **the server expands it to a functions block**."*

### 4.3 真相 — 三种存在形态

> ⚠️ **【本节有已知错误，先读 §11.3】** 下文"从头到尾不进 tools 数组"以及"形态 2：schema 进 messages 流，不进 tools 数组"的说法**不准确**：deferred 工具一旦经 ToolSearch 发现，**会进 tools 数组**（带 `defer_loading: true`）。修正后的完整路径与三态表见 **§11.3**（源码实证 `claude.ts:1160-1167` + `api.ts:224-225`）。本节原文保留仅作"错误直觉"对照。

deferred 工具的"存在感"以三种形式存在，**从头到尾不进 tools 数组**：

#### 形态 1：未加载 — 只在 system reminder 里以 name 出现

```xml
<available-deferred-tools>
WebFetch
WebSearch
mcp__github__create_issue
mcp__slack__list_messages
...
</available-deferred-tools>
```

**只有工具名，不带任何描述或参数 schema**。每个约 5-10 token。

源码 `tools/ToolSearchTool/prompt.ts:115-117`：

```typescript
export function formatDeferredToolLine(tool: Tool): string {
  return tool.name   // 就这一行
}
```

注释 `prompt.ts:111-113` 透露了背景：CC 团队做过 A/B 实验（`exp_xenhnnmn0smrx4`），在名单里加 `searchHint`（简短描述）对模型工具发现能力**没有提升**，2026 年 3 月 21 日下线了。所以最终就是裸 name 列表。

模型只能靠工具名猜功能，要知道细节必须调 ToolSearch。这是个比"name + desc"更激进的"信任模型命名直觉"策略——也意味着**工具命名规范化比写 description 更重要**。`mcp__server__action` 这种结构化命名比自然语言描述更有效，因为模型在 ToolSearch 之前**只有名字可以判断**。

#### 三个位置看 schema 到底在哪

很容易混淆"deferred 工具有没有 schema"——答案要分三个位置：

| 工具状态 | 客户端代码 registry | tools 数组（API 请求） | system reminder（LLM 看到） | messages 流 |
|---|---|---|---|---|
| **Eager** | ✓ 完整 schema | ✓ 完整 schema | — | — |
| **Deferred 未加载** | ✓ 完整 schema | ✗ | ✓ 仅 name | ✗ |
| **Deferred 已加载** | ✓ 完整 schema | ✗ | ✓ 仅 name | ✓ 完整 schema（server 展开） |

**关键点**：Schema 在客户端代码里**永远存在**（Tool 对象的 `input_schema` 字段），但 LLM **看不到**——除非被加载到上下文。`ToolSearch` 是把"知道工具名"升级为"能调用工具"的唯一通道。

#### 形态 2：加载 — schema 进 messages 流，不进 tools 数组

```
[messages 历史]
  user: 帮我查 PG 慢查询
  assistant: tool_use { name: "ToolSearch", input: {query: "postgres"} }
  user: tool_result [
    { type: "tool_reference", tool_name: "mcp__postgres__query" }
  ]
  ────── 服务端在这里 inline 展开 ──────
  user (服务端注入): <functions>
                       <function>{ "name": "mcp__postgres__query",
                                   "description": "...",
                                   "parameters": {...} }</function>
                     </functions>
  assistant: tool_use { name: "mcp__postgres__query", ... }  ← 可以调用
```

**关键事实**：展开的 `<functions>` 块在 `messages` 数组里，**不在 `tools` 字段里**。

#### 形态 3：tools 数组 — 只有 eager 工具，session 内基本固定

```json
{
  "tools": [
    { "name": "Read",  "input_schema": {...} },     // eager
    { "name": "Write", "input_schema": {...} },     // eager
    { "name": "Bash",  "input_schema": {...} },     // eager
    { "name": "ToolSearch", "input_schema": {...} } // eager，作为"门票"
    // ← 几十个 MCP 工具一个都不在这
  ]
}
```

### 4.4 为什么这个设计这么聪明

`💡 关键洞察`：**把工具 schema 当成"对话内容"而不是"工具声明"，是这套架构的真正天才之处。**

对话内容**天然是追加式的**——messages 末尾增加新内容，不影响前面的 cache prefix。而 tools 数组是个**集合**，任何位置的变化都让从那以后的 token 全 miss。CC 用 messages 的增量特性"驮"工具加载，等于免费搭车。

### 4.5 tools 数组什么时候真的会变

| 触发 | tools 数组变化 |
|---|---|
| session 启动 | eager 工具一次性进入 |
| 安装新插件且新插件提供 **eager** 工具 | 新 eager 进入 |
| 安装新插件且新插件提供 **deferred** 工具（常见，MCP 默认 deferred） | **tools 数组不变**，只是 `<available-deferred-tools>` 名单加一行 |
| 加载 deferred 工具（通过 ToolSearch） | ⚠️ **tools 数组会新增该工具**（带 `defer_loading: true`），该工具**首次**进入时击穿一次 cache、之后稳定；schema 再由服务端就地展开进 messages 流给模型看。详见 §十一·三 |
| session 结束 | 无关 |

> ⚠️ **§4.3 / §4.5 源码订正**：把 deferred 工具加载描述成"schema 永不进 tools 数组"并不准确。真相分两态——**未被发现**的 deferred 工具确实不在 tools 数组（只在 `<available-deferred-tools>` 列名字）；但**一旦经 ToolSearch 发现**，下一轮请求会用 `extractDiscoveredToolNames` 扫历史里的 `tool_reference`，把该工具的完整 schema 带 `defer_loading: true` **加进 tools 数组**（`claude.ts:1160-1167` + `api.ts:223-225`），再由服务端就地展开进 messages 给模型看。省的是"不预声明全部"，不是"发现后也不进数组"。完整机制与缓存代价见 **§十一·三**。

---

## 五、CC 的防御纵深 — 四层叠加

回到核心问题：Anthropic prompt cache 是连续前缀匹配，任何位置变动都让那之后全 miss。CC 怎么应对？

### 5.1 防御层 1：架构隔离（让易变的远离稳定的）

```
tools 数组（易变） ─────► 改造为"只放 eager 工具，session 内基本静态"
deferred 工具 schema ──► 改走 messages 流（server-side tool_reference 展开）
高频 MCP 工具 ────────► alwaysLoad 白名单强制 eager
```

源码证据：`services/api/claude.ts:1329` 注释直说——

> *"ephemeral prepend (which busts cache whenever the pool changes)"*

他们**明知**直接在 system prompt 前置 deferred 工具名单会破坏 cache，所以做了 `isDeferredToolsDeltaEnabled()` 特性开关（`claude.ts:1330`），把名单改为**持久化的 delta 附件**而不是每轮重新 prepend。

### 5.2 防御层 2：cache_control breakpoint 显式切段

```typescript
// services/api/claude.ts:603, 615, 648, 663
content: [
  { type: 'text', text: '...', cache_control: getCacheControl({...}) }
]
```

每个用户消息和助手消息的**最后一个 block** 都带 `cache_control`。CC 用它把 prompt 切成多段独立的缓存单元，**一段 miss 不连累其他段**。

特别是 `claude.ts:1388` 这段注释，展示了**故意把动态部分放在缓存边界之后**：

> *"Server tools must be in the tools array by API contract. **Appended after toolSchemas (which carries the cache_control marker) so toggling /advisor only churns the small suffix, not the cached prefix.**"*

——意思是：`/advisor` 这种用户可切换的功能会让 tools 数组变。CC 故意把它放在 `cache_control` 标记**之后**，这样切换 advisor 只让那一小段失效。

### 5.3 防御层 3：TTL 分级 + 双 ephemeral 桶

```typescript
// services/api/emptyUsage.ts:16-17
ephemeral_1h_input_tokens: 0,
ephemeral_5m_input_tokens: 0,
```

Anthropic API 支持两种 ephemeral cache：

- **5 分钟**：便宜的写入（1.25x），用于会话内频繁变化的内容
- **1 小时**：更贵的写入（2x），用于跨会话或长会话的稳定段

CC 给**稳定段**（eager tools + 核心 system prompt）用 1h，**易变段**（用户对话尾部）用 5m。两段独立失效，**长 TTL 的段长期保持命中率**。

### 5.4 监控层：promptCacheBreakDetection — "破窗探测器"

最让人意外的是 CC 有个**专门的诊断模块** `services/api/promptCacheBreakDetection.ts` 追踪 cache break。每次请求都记录 16+ 个维度的 hash：

```typescript
type PreviousState = {
  systemHash: number              // system prompt 内容
  toolsHash: number               // tools 数组内容
  cacheControlHash: number        // cache_control 配置(scope/TTL)
  perToolHashes: Record<...>      // 每个工具的 schema hash
  toolNames: string[]
  betas: string[]                 // beta 头列表
  model: string
  fastMode: boolean
  globalCacheStrategy: string     // 'tool_based' | 'system_prompt'
  autoModeActive: boolean
  // ... 还有更多
}
```

任何维度变化时，**自动生成 diff 文件 + 上报 analytics**，在 BigQuery 里聚合分析。注释里出现的真实数据（`promptCacheBreakDetection.ts:35-37`）：

> *"diffed to name which tool's description changed when toolSchemasChanged but added=removed=0 (**77% of tool breaks per BQ 2026-03-22**). AgentTool/SkillTool embed dynamic agent/command lists."*

`💡 关键洞察`：**77% 的 cache break 不是工具增删，而是某个工具的 description 字段变了**——比如 AgentTool / SkillTool 描述里嵌入了动态的 agent 列表 / command 列表，那些列表每次会话不一样。CC 团队靠生产数据发现了这个反直觉的事实，然后针对性修复（把动态部分从 description 里剥离）。

这个监控模块本身就比那些机制更值得学——**Anthropic 把 cache hit rate 当成核心产品指标，跟 latency / 错误率同等重要**。

---

## 六、对照：Codex 的差异化设计

OpenAI Codex CLI 在同一个问题上做了不同的选择。对比有助于理解 CC 设计的独到之处。

### 6.1 启动策略

| 维度 | Codex | Claude Code |
|---|---|---|
| **触发延迟加载的阈值** | MCP 工具 ≥ 100 才强制 defer | MCP 工具**默认全部 defer**，opt-out 模式 |
| **Eager 工具决策** | 由 `exposure.is_direct()` 标记 | `alwaysLoad: true` 优先级最高 |
| **检索机制** | BM25 索引（`tool_search.rs`） | 名称匹配 + `select:<name>` 直选 |
| **Schema 投递格式** | `defer_loading: true` 占位 spec | `tool_reference` 块（API 原生支持） |

### 6.2 Schema 投递的关键差异

**Codex 流程**：

```
turn N: 模型调用 tool_search → 返回工具列表
turn N+1: 客户端必须把找到的工具 schema 加进 tools 数组
        → tools 数组变了 → cache miss
```

**CC 流程**：

```
turn N: 模型调用 ToolSearch → 返回 tool_reference 块
turn N+1: 客户端把 tool_reference 作为 tool_result 发回
        → tools 数组不变 → 服务端在 messages 流里展开
        → cache 不受影响
```

### 6.3 Cache 控制粒度

| 能力 | Codex (OpenAI API) | Claude Code (Anthropic API) |
|---|---|---|
| 用户可控的 breakpoint | ❌（按 1024-token block 自动） | ✅（最多 4 个 `cache_control`） |
| TTL 选择 | ❌（统一约 5min） | ✅（5min / 1h 两档） |
| 缓存粒度 | 整个请求作为 unit | 块级 |
| 用户可观测的 cache 状态 | 仅 cache_creation_input_tokens | cache_creation + cache_read + 双 ephemeral 桶 |

`💡 关键洞察`：Anthropic 把 `cache_control` 做成**用户可控的细粒度 breakpoint**，这是 API 设计层面的差异化竞争。CC 的整套 ToolSearch + tool_reference 架构，**本质上是为了"最大化利用这个 API 特性"——它和 API 是协同设计的**。Codex 在 OpenAI API 上即使想抄 CC 的架构也学不全，因为底层 API 没给那么细的控制。

### 6.4 哲学差异

```
Codex 假设：大多数任务工具少、prompt 稳定 → 默认 eager 简单可靠
CC 假设：MCP 生态会爆炸式增长、用户工具集会持续变动 → 默认 defer + cache breakpoint
```

CC 的设计**前瞻性更强**，但也**复杂度更高**。Codex 是"现在友好，未来加补丁"；CC 是"现在过度工程，赌未来"。

> 📎 本节只对照"工具暴露策略"。Codex 与 CC 更底层的差异在**传输是否有状态**——Codex 走 OpenAI Responses，HTTP 路径全发、WebSocket 路径用 `previous_response_id` 只发增量；CC 走 Anthropic 无状态 API 永远全发 + cache。四方完整对照见 **§11.5**，统一到"前缀复用第一定律"的抽象见 **§11.6**。

---

## 七、面对超多工具的优雅设计指南

把前面的机制凝练成可执行的设计原则。

### 7.1 七条 Recipe

```
1. tools 数组只留 eager 工具，deferred 全部不进数组
2. deferred 工具的存在感通过 "name + 1行 desc" 在 system reminder 里露出
3. 模型按需调 ToolSearch，工具 schema 通过 messages 流注入（不是 tools 数组）
4. 高频工具走 alwaysLoad 白名单强制 eager（避免每次都搜）
5. 工具描述写**搜索友好的关键词**（因为 BM25 / 关键词匹配靠这个吃饭）
6. 用 plugin:tool 命名空间避免冲突，realpath 去重避免重复加载
7. 条件激活：跟文件路径绑定的工具（如 frontend-design），只在触及匹配文件时进上下文
```

### 7.2 工具数量决策矩阵

| 工具数 | 建议 |
|---|---|
| < 20 | 全 eager，别折腾 |
| 20–100 | 高频 eager + 长尾 defer |
| > 100 | 全 defer + 必须上检索索引（BM25/名称匹配） |

### 7.3 三层冷启动的对策清单

```
Token 冷启动:
  • deferred 工具只暴露 name，完整 schema 按需加载
  • 加载后放 messages 而非 tools
  • 触发阈值：工具 schema 总和 > 上下文窗口的 5%

Latency 冷启动:
  • 高频/通信类工具加 alwaysLoad 白名单 → 跳过搜索
  • 搜索结果缓存（getToolDescriptionMemoized 模式）
  • 工具描述里塞同义词，让搜索一次命中
  • 触发阈值：延迟 > 500ms

Cache 冷启动:
  • 稳定段（system + eager tools）放 breakpoint A，长 TTL（1h）
  • 动态段（已加载 deferred 工具）走 messages，无需独立 breakpoint
  • 安装新插件时，deferred 工具加 stub name → 影响最小
  • 触发阈值：平均 cache hit rate < 70% 就要重新切分 breakpoint
```

### 7.4 综合决策矩阵

```
                 工具集稳定？           工具集动态？
              ─────────────────────  ─────────────────────
  少量工具    全 eager，简单粗暴      全 eager + breakpoint
  (< 20)     (Codex 默认)           保护

  中量工具    高频 eager + 长尾       全 defer + ToolSearch
  (20–100)    defer                 (CC 默认)

  大量工具    必须 defer + 索引       defer + 索引 +
  (> 100)     (Codex 阈值)         breakpoint 切分
                                    (CC 完整方案)
```

---

## 八、贯穿例子：Incident Commander Agent

用一个 80 工具的事故响应 agent 展示整套架构落地。

### 8.1 场景设定

```
公司：某 B2C SaaS，oncall 工程师用 agent 辅助诊断
工具集：80 个，平均 schema ~800 token，全 eager 要 64k token
  ├─ 内置          10 个(Read/Write/Bash/Grep/Edit/...)
  ├─ GitHub MCP    10 个(list_prs/get_commits/create_issue/...)
  ├─ Slack MCP     15 个(post_message/search_messages/...)
  ├─ Datadog MCP   12 个(query_metrics/log_search/...)
  ├─ PagerDuty MCP  8 个(ack_incident/get_oncall/...)
  ├─ Kubernetes MCP 15 个(kubectl_logs/kubectl_describe/...)
  └─ PostgreSQL MCP 10 个(query/explain/schema/...)
```

### 8.2 启动时 tools 数组的真实形状

```json
{
  "tools": [
    // ───── 10 个内置 eager ─────
    { "name": "Read", "input_schema": {...} },
    { "name": "Bash", "input_schema": {...} },
    ...
    // ───── 4 个 alwaysLoad 白名单 ─────
    { "name": "datadog_query_metrics",   "input_schema": {...} },
    { "name": "slack_post_message",      "input_schema": {...} },
    { "name": "pagerduty_ack_incident",  "input_schema": {...} },
    { "name": "kubectl_logs",            "input_schema": {...} },
    // ───── 1 个门票工具 ─────
    { "name": "ToolSearch", "input_schema": {...} }
  ]
}
```

**总共 15 个工具进 tools 数组，约 12k token**。剩下 66 个 defer。

白名单选取依据：拿过去 3 个月的 incident 处理记录，统计工具调用频次，**top 5% 频率的工具进白名单**。

### 8.3 上下文形状变迁

```
turn 1: [system+名单 ~1.3k][tools 12k][msg: 用户问题 50]
turn 2: [system+名单 ~1.3k][tools 12k][msg: + datadog_query_metrics 调用]
turn 3: [system+名单 ~1.3k][tools 12k][msg: + ToolSearch + tool_ref +
                                          server展开 functions(postgres_query) 600 +
                                          assistant 调用 + 结果]
turn 4: [system+名单 ~1.3k][tools 12k][msg: + 同上结构，加 postgres_explain 600]
turn 5: [system+名单 ~1.3k][tools 12k][msg: + github_list_prs 700]
turn 6: [system+名单 ~1.3k][tools 12k][msg: + slack post]
turn 7: [system+名单 ~1.3k][tools 12k][msg: + pagerduty ack]

观察：
  ├─ 第一段 [system+名单] 2k：7 turn 全程不变，长 TTL cache 100% hit
  ├─ 第二段 [tools] 12k：7 turn 全程不变，长 TTL cache 100% hit
  └─ 第三段 [messages]：每 turn 都追加，正常对话增量，本来就要算
```

### 8.4 三层冷启动的实际数字

```
Token 冷启动：
  全 eager：tools 数组 64k token
  现方案：  tools 数组 12k + 系统主体 ~700 + deferred 名单 ~600 ≈ 13.3k
  节省：    78%（50k token 让出来给对话和日志用）

Latency 冷启动（7 turn 完整事故响应）：
  全 eager：每 turn ~3.5s = 24.5s
  现方案：  4 个 alwaysLoad turn × 1.2s + 3 个 ToolSearch turn × 1.5s = 9.3s
  节省：    62%

Cache 冷启动（5 分钟内来第二个事故）：
  全 eager：cache 完美命中，但 MCP 抖动会全部失效
  现方案：  系统段 1h TTL 命中，对话段是新对话不复用
  优势：    对 MCP 抖动免疫，新事故的 cache 不依赖旧事故的 messages
```

`💡 关键洞察`：重点不在"省了多少 token"，而在**"哪段在变"和"哪段不变"被清晰切开**。前两段是"配置层"，第三段是"执行层"。配置层永远在 cache 里，执行层正常计费——这就是 production-grade agent 的稳态。

---

## 九、Skills 加载的对应模式

`@09-memory-system` 已经覆盖了 Skills 的 frontmatter-only 加载机制。这里从 cache 经济学的视角补充：

### 9.1 Skills 复用同一套延迟加载思想

| 阶段 | 加载内容 | Token 成本 | 位置 |
|---|---|---|---|
| 启动时 | name + description + whenToUse | ~30-100 token/skill | system prompt |
| 触发时 | 完整 markdown body | 几百到几千 token/skill | messages 流 |
| 条件激活 | `paths` frontmatter 匹配的 skill | 0（不触及就不加载） | 不进 prompt |

### 9.2 与工具加载的一致性

Skills 和 deferred tools **共用同一套架构原理**：

```
工具：name + desc 进 system reminder，schema 进 messages
Skills：name + desc 进 system reminder，body 进 messages
```

两者都遵循"**配置层留 stub，执行层走 messages 增量**"的总原则。

### 9.3 条件激活（conditionalSkills）

`skills/loadSkillsDir.ts:788-790` 显示，带 `paths` frontmatter 的 skill 仅在触及匹配文件时激活：

```typescript
if (skill.frontmatter.paths) {
  conditionalSkills.set(skill.name, skill)  // 不进 prompt
}
```

这是 Skills 加载的"第三态"——比 frontmatter-only 更省。例如 `frontend-design` skill 只在编辑 `.tsx/.vue` 文件时才进入 context。

---

## 十、源码引用一览

本章涉及的关键源码位置，便于回查：

### ToolSearch 与 deferred 工具

| 文件 | 内容 |
|---|---|
| `tools/ToolSearchTool/ToolSearchTool.ts:304-471` | ToolSearch 主体实现 |
| `tools/ToolSearchTool/ToolSearchTool.ts:442` | "client-side tool_reference expansion" 注释 |
| `tools/ToolSearchTool/ToolSearchTool.ts:462-469` | 返回 tool_reference 块 |
| `tools/ToolSearchTool/prompt.ts:44` | "Until fetched, only the name is known" |
| `tools/ToolSearchTool/prompt.ts:62-108` | `isDeferredTool()` 决策逻辑 |
| `tools/ToolSearchTool/prompt.ts:107` | `tool.shouldDefer === true` |
| `utils/messages.ts:1912` | "the server expands it to a functions block" |
| `utils/messages.ts:1933-1963` | `relocateToolReferenceSiblings` 防 stop pattern |
| `utils/toolSearch.ts:101-152` | `tst-auto` 自动启用判定 |

### Cache 控制

| 文件 | 内容 |
|---|---|
| `services/api/claude.ts:603, 615, 648, 663` | `cache_control` 的实际写入位置 |
| `services/api/claude.ts:1329-1330` | "ephemeral prepend busts cache" 注释 + `isDeferredToolsDeltaEnabled` |
| `services/api/claude.ts:1349-1356` | Chrome tool 注入的 cache 考量 |
| `services/api/claude.ts:1388-1389` | "Appended after toolSchemas ... only churns the small suffix" |
| `services/api/emptyUsage.ts:16-17` | 双 ephemeral 桶（5m / 1h） |
| `services/api/promptCacheBreakDetection.ts` | 整个监控模块 |
| `services/api/promptCacheBreakDetection.ts:31-37` | 16+ 维度 hash 追踪 + BQ 数据 |

### Skills 加载

| 文件 | 内容 |
|---|---|
| `skills/loadSkillsDir.ts:96-105` | frontmatter-only 加载 |
| `skills/loadSkillsDir.ts:118, 753-761` | realpath + seenFileIds 去重 |
| `skills/loadSkillsDir.ts:788-790` | conditionalSkills + paths 触发 |
| `tools/SkillTool/SkillTool.ts:81-94` | Skill body 按需加载 |

### Subagent 工具继承

| 文件 | 内容 |
|---|---|
| `tools/AgentTool/forkSubagent.ts:60-72` | `useExactTools: true` 锁定父级工具集 |

---

## 十一、无状态重提交 与 前缀复用第一定律

> 本节是对全章的"地基补完"：前面讲的所有 cache 技巧，都站在一个前提上——**Anthropic API 是无状态的，agent loop 每一轮都把全量上下文重发一遍**。理解这一点，才知道 cache 到底在省什么；理解它的对立面（Codex 的有状态增量），才知道两套设计共享同一条物理定律。全部结论有源码实证（CC = `claude-code-sourcemap`，Codex = `codex-rs`）。

### 11.1 前提：agent loop 是"无状态重提交"

主循环就是一个 `while (true)`，每转一圈调一次模型，每次调用都把 `system + tools + 全量 messages` 重新发出去：

```
query.ts:307   while (true) { ...                       // 每圈 = 一次 API 调用
query.ts:663     callModel({ messages, systemPrompt, tools })  // 三者每次都传
query.ts:1716    messages: [...prev, ...assistant, ...toolResults]  // 历史 append-only 增长
query.ts:1659    refreshTools()                          // turn 之间唯一会换 tools 的入口
```

**Anthropic Messages API 没有"会话记忆"**：服务端处理完即忘，下一个请求必须自带一切。教学版 `claw0/s06` 把这点暴露得最干净——它 `client.messages.create(system=…, tools=…, messages=…)` 每次全发，且**没有任何 `cache_control`**，所以每轮全价。生产 CC 和它的唯一区别，就是加了缓存。

### 11.2 为什么"每轮全发"还不贵：重发 ≠ 重算 ≠ 全价

"重发"只是**字节上传**这一层是全量的。重复前缀（system + tools + 老 messages）发生三件不同的事：

| 层面 | 重复前缀发生什么 |
|---|---|
| 传输（HTTP body） | 确实每次发全量字节（无状态的代价） |
| 服务端计算 | 命中前缀 KV-cache → 跳过重新 attention |
| 计费 | 走 `cache_read`（约 0.1×），不是全价 |

算笔账：某轮输入 system ~2.5k + tools ~12k + messages ~80k ≈ 95k token。带缓存后等效计费 ≈ `90k×0.1×(缓存读) + 5k×1×(本轮新增) ≈ 14k 全价 token`，省约 85%。`buildSystemPromptBlocks`（`claude.ts:1376`）给每个 system 块打 `cache_control`，正是为此。**结论：省钱靠缓存命中，不靠少发。**

### 11.3 源码订正：deferred 工具加载的真实路径（补 §4.3 / §4.5）

ToolSearch **不传 schema**，只返回 `tool_reference`（光一个工具名）。schema 在**下一次请求**才进 tools 数组：

```
请求 N-1   deferred 工具被 filter 掉，不在 tools 数组（claude.ts:1160-1167）
           模型只在 <available-deferred-tools> 看到名字（5-10 token/个）
   │
   │  模型调 ToolSearch("select:mcp__pg__query")
   │  → 返回 tool_result[ {type: tool_reference, tool_name} ] 进消息历史
   ▼
请求 N     extractDiscoveredToolNames(messages) 扫到该 tool_reference（toolSearch.ts:545-580）
           → filter 放行 → 该工具完整 schema 加进 tools 数组，带 defer_loading:true（api.ts:223-225）
           → 服务端凭 defer_loading schema + 历史里的 tool_reference，就地展开进 messages 流
           → 模型可调用
```

两个要点纠正旧表述：

- **schema 有两个位置**：传输载体 = tools 数组（`defer_loading:true`，只放**已发现**的）；可见位置 = messages 流（服务端展开的 `<functions>`）。`defer_loading:true` = "schema 在请求里但默认别给模型看"，直到 `tool_reference` 激活。两者缺一不可。
- **用过即焊死，单调增长**：`extractDiscoveredToolNames` 扫整个历史，且**跨 compact 保留**（compact boundary 用 `preCompactDiscoveredTools` 把发现集搬过边界，`toolSearch.ts:553-559`）。所以一旦用过，该工具整个 session 都在 tools 数组里。
- **缓存代价 = 台阶式**：每个新工具**首次**进入 tools 数组那一轮击穿一次前缀缓存（tools 在最前面，往后 system+messages 全重建），之后稳定命中。省的是"**不预声明全部 200 个**（用不到的只占名字）"，不是"发现后也不进数组"。这也是为什么"`select:` 精确选 + 工具命名规范化"直接省钱：搜得准 = 只跳一级台阶，乱搜 = 台阶跳一路、还把没用的工具永久焊进数组。

### 11.4 对立面：有状态 vs 无状态

| | 无状态（stateless） | 有状态（stateful） |
|---|---|---|
| 服务端 | 处理完即忘 | 记得你之前的事 |
| 请求 | **自带一切**，可独立处理 | 只带"在上次基础上加一点"，**依赖服务端那份记忆** |
| 记忆放哪 | **客户端**（每轮重传 messages） | **服务端**（按句柄存上下文） |
| 状态丢了 | 无所谓（本来就全发了） | **塌**：必须回退全量或报错 |
| 类比 | 每说完一句就失忆的店员（你每句重述全部背景） | 记得你的店员（你只说"加份薯条"；他换班了就懵） |

无状态不代表"没记忆"，而是**记忆放在客户端**——模型本身无记忆，多轮对话全靠 loop 每轮把增长的 messages 喂回去。HTTP/REST 选无状态图的是**健壮 + 可水平扩展**（任何一台机器都能处理任何请求），代价是啰嗦，于是用 prefix cache 把"啰嗦"变便宜。

### 11.5 Codex 对照：HTTP 全发 vs WebSocket 增量

Codex 走 OpenAI Responses API，有两条路：

**路径 A — HTTP（默认/兜底）**：和 Anthropic 一样全发。请求结构体 `ResponsesApiRequest`（`client.rs:746-764`）每轮带 `instructions + 全量 input + tools`；`store = is_azure_responses_endpoint()`（普通端点 = false，不靠服务端存）；靠 `prompt_cache_key`（按 `thread_id`）命中 OpenAI 前缀缓存。

**路径 B — WebSocket 增量（真·有状态）**：`prepare_websocket_request`（`client.rs:1037-1062`）只发 `previous_response_id + 增量项`，历史不重发，服务端凭 id 接上下文。但触发条件严格（`get_incremental_items`，`client.rs:988-1024`）：

1. websocket 开着且有上一个非空 `response_id`；
2. **所有非 input 字段（instructions、tools…）必须和上轮完全相同**；
3. 本轮 input 必须是上轮 baseline 的**严格前缀扩展**（只在末尾追加）。

任一不满足 → 回退全量。

| | claw0/s06 | Claude Code | Codex (HTTP) | Codex (WS 增量) |
|---|---|---|---|---|
| 底层 | Anthropic | Anthropic | OpenAI Responses | OpenAI Responses |
| system/instructions | 每轮发 | 每轮发 | 每轮发 | **不发**（靠 prev_id） |
| 历史 messages/input | 每轮全发 | 每轮全发 | 每轮全发 | **只发 delta** |
| tools | 每轮发 | 每轮发 | 每轮发 | **不发**（靠 prev_id） |
| 省钱机制 | 无 | cache_control | prompt_cache_key | 服务端状态 + 增量 |
| 重复部分计费 | 全价 | cache_read | cache_read | 几乎不发 |

### 11.6 前缀复用第一定律

把 prefix cache（无状态）和 previous_response_id 增量（有状态）抽到底，**底层是同一块东西：服务端的 prefix KV-cache**。`previous_response_id` 本质就是一个指向被保留 KV 状态的句柄。两者复用的**条件完全一样**：

> **能省的永远只有那段没变的前缀，且只能往末尾追加。**

根因在 transformer 的**因果注意力**：位置 N 的 KV 张量依赖位置 0..N-1 的全部内容。**改了第 N 个 token，N 之后每个位置的 KV 都得重算。** 这不是哪个 API 的策略，是物理约束。无状态/有状态只是对"前缀变了怎么办"给出不同答案：

| | 前缀逐字节相同 | 前缀变了 |
|---|---|---|
| 无状态 prefix cache | 命中省钱 | **静默重算**，client 已全发 → 仍正确 |
| 有状态增量 | 只发 delta | **退全量或报错**，需自己兜底 |

所以 prefix cache 是"**有状态的收益 + 无状态的兜底**"——KV 丢了不影响正确性，只影响速度和钱；Codex 的增量是真·有状态，所以要在 `get_incremental_items` 里手写回退逻辑，等于用代码补上 prefix cache 天生就有的安全网。

**可迁移推论（设计 agent 上下文的第一性原则）：**

1. **能 append 就别 insert，能放末尾就别放开头**——判断一个改动贵不贵，只问"它动的是前缀，还是只在尾巴加"。
2. **任何插入到中间的操作都是缓存杀手**：system prompt 塞动态时间戳、tools 数组中段插工具、改写历史消息，都会让那一点之后的 KV 全废。
3. **compact 天然贵**：它是"改写中段历史"，等于在前缀中部动土，那一刻后面全重算/重发——所以 compact 是"故意付一次满额、换之后上下文变短"的权衡，不是免费的。
4. CC 的所有招数（tools 稳定、schema 走 tool_reference 往 messages 末尾追加、description 别塞动态内容、breakpoint 切在变化边界、deferred 工具发现后才进数组）——**全是这条定律的不同推论**，目标只有一个：别动前缀，只在尾巴长。

`💡 一句话`：**无状态 vs 有状态 = 记忆放客户端还是服务端；但两者都受同一条"前缀不变才能复用"的因果律约束。CC 把缓存当看不见的加速器（无状态 + 兜底），Codex 在 websocket 上把它当承重墙（有状态 + 手写回退）。看懂这条定律，全章的技巧都成了它的推论。**

---

## 十二、一句话结案

> **CC 的 prompt cache 问题不是"用一个技巧解决"的，是用架构隔离（让易变的远离稳定的）+ 协议利用（cache_control 显式标记）+ 持续监控（自动检测+数据驱动修复）三件套永久控制的。它不是"消除"cache miss，是"把 miss 限制在最小范围 + 持续发现新的 miss 源"。**

这套设计的可迁移规则只有 3 条：

1. **配置层与执行层分开**：tools 数组放稳定的，messages 流放动态的
2. **高频走白名单，长尾走检索**：用 28 法则，5% 工具承担 80% 调用就值得 eager
3. **breakpoint 切在变化边界**：不变的段独立缓存，变的段独立失效，影响不串场

把这 3 条带到任何多工具 agent 项目，都成立。区别只是工具具体名字，**架构是同一个**。

---

## 相关章节

- [02 - Tool 系统](./02-tool-system.md) — ToolSearch 的功能描述与评分算法
- [05 - 扩展系统](./05-extension-system.md) — Skills 与 Plugin 的加载流程
- [09 - 记忆系统](./09-memory-system.md) — Memory 与 Skills frontmatter 的注入机制
