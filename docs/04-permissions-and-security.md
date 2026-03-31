# 04 - 权限与安全系统

> 确保 AI 不会"搞砸"的守门人：从规则匹配到沙箱隔离

---

## 一、为什么需要权限系统？

Claude Code 赋予 AI 执行 Shell 命令、编辑文件、访问网络的能力。如果没有权限控制，一个错误的 `rm -rf /` 就可能造成灾难。权限系统是 **安全与效率之间的平衡器**：
- 太严格 → 用户每次操作都要确认，体验极差
- 太宽松 → AI 可能执行危险操作
- 理想状态 → 已知安全的操作自动通过，未知的询问用户

---

## 二、权限模式

**文件**: `src/utils/permissions/PermissionMode.ts`

| 模式 | 行为 | 适用场景 |
|------|------|---------|
| `default` | 敏感操作需确认 | 日常使用 |
| `acceptEdits` | 文件编辑自动通过 | 信任 AI 编辑能力时 |
| `plan` | 暂停执行，等待审阅 | 大型重构前的规划 |
| `bypassPermissions` | 跳过所有权限检查 | 管理员/沙箱环境 |
| `auto` | AI 驱动的自动审批 | 仅内部使用 |

**模式切换循环**（Shift+Tab）:
```
外部用户: default → acceptEdits → plan → bypassPermissions → default
内部用户: default → bypassPermissions → auto → default
```

---

## 三、权限决策流程

**文件**: `src/utils/permissions/permissions.ts`

```
工具请求 tool_use
  │
  ▼
┌───────────────────────────────────────────────────────┐
│ 1. Allow 规则匹配                                      │
│    alwaysAllowRules 中是否有精确匹配?                   │
│    ├─ 匹配 → ✅ 直接通过                               │
│    └─ 不匹配 → 继续 ↓                                  │
├───────────────────────────────────────────────────────┤
│ 2. Deny 规则匹配                                       │
│    alwaysDenyRules 中是否有精确匹配?                    │
│    ├─ 匹配 → ❌ 直接拒绝                               │
│    └─ 不匹配 → 继续 ↓                                  │
├───────────────────────────────────────────────────────┤
│ 3. Ask 规则匹配                                        │
│    alwaysAskRules 中是否有匹配?                         │
│    ├─ 匹配 → ❓ 强制询问用户                           │
│    └─ 不匹配 → 继续 ↓                                  │
├───────────────────────────────────────────────────────┤
│ 4. 危险权限过滤（Auto 模式）                            │
│    剥离"危险"的 Allow 规则                              │
│    （代码解释器、Agent allow 等）                       │
│    → 继续 ↓                                            │
├───────────────────────────────────────────────────────┤
│ 5. 分类器路径（Auto 模式）                              │
│    classifyBashCommand() AI 分析                       │
│    ├─ 安全 → ✅ 通过                                   │
│    ├─ 不安全 → ❌ 拒绝（追踪拒绝次数）                 │
│    └─ 连续拒绝 3 次 / 总拒绝 20 次 → 降级到询问       │
├───────────────────────────────────────────────────────┤
│ 6. 交互式对话                                          │
│    向用户展示确认对话框                                 │
│    用户选择: Allow / Deny / Allow Always                │
└───────────────────────────────────────────────────────┘
```

---

## 四、权限规则系统

### 4.1 规则格式

**文件**: `src/utils/permissions/PermissionRule.ts`

```typescript
type PermissionRule = {
  source: PermissionRuleSource   // 来源
  ruleBehavior: 'allow' | 'deny' | 'ask'
  ruleValue: {
    toolName: string             // 工具名
    ruleContent?: string         // 匹配内容（可选）
  }
}
```

**规则语法**:

| 格式 | 含义 | 示例 |
|------|------|------|
| `ToolName` | 该工具所有操作 | `Bash` → 允许所有 Bash 命令 |
| `ToolName(content)` | 精确内容匹配 | `Bash(npm install)` |
| `ToolName(prefix*)` | 通配符匹配 | `Bash(git *)` → `git push`, `git commit`... |
| `ToolName(domain:X)` | 域名匹配（WebFetch） | `WebFetch(domain:github.com)` |

**通配符匹配逻辑**:
```typescript
// "git *" 模式
matchWildcardPattern("git *", "git push")    // ✅ 匹配
matchWildcardPattern("git *", "git")         // ✅ 匹配（尾部 * 可选）
matchWildcardPattern("git *", "gitconfig")   // ❌ 不匹配（空格很重要）
```

### 4.2 规则来源与优先级

```
优先级（高 → 低）:
  1. CLI 参数        --allowed-tools / --denied-tools
  2. 会话内规则      动态修改
  3. Local Settings  .claude/settings.local.json（gitignored）
  4. Project Settings .claude/settings.json（共享）
  5. User Settings   ~/.claude/settings.json（全局）
  6. Managed/Policy  /etc/claude-code/managed-settings.json
                     macOS plist / Windows Registry
```

> **设计洞察**: Local Settings 优先于 Project Settings，因为 Local 是 gitignored 的——开发者可以有自己的本地覆盖而不影响团队配置。

### 4.3 危险模式检测

**文件**: `src/utils/permissions/dangerousPatterns.ts`

在 Auto 模式下，以下 Allow 规则会被自动剥离：

**危险 Bash 模式**:
```
python, python3, node, ruby, bash, sh, zsh, perl, php
→ 代码解释器，可执行任意代码
```

**危险 PowerShell Cmdlet**:
```
Invoke-Expression, Invoke-Command     → 字符串求值
Start-Process, Start-Job              → 进程生成
Add-Type, New-Object                  → .NET 逃逸
pwsh, powershell, cmd, wsl            → 嵌套 Shell
```

**危险文件系统路径**:
```
.gitconfig, .gitmodules               → Git 配置
.bashrc, .zshrc, .profile             → Shell 配置
.mcp.json, .claude.json               → Claude 配置
.git/, .vscode/, .idea/               → 敏感目录
```

---

## 五、Hooks 系统

**文件**: `src/utils/hooks.ts` (4000+ 行)

Hooks 是用户定义的 **Shell 命令**，在特定生命周期点执行。

### 5.1 Hook 事件类型

| 事件 | 触发时机 | 用途示例 |
|------|---------|---------|
| `SessionStart` | 会话开始 | 加载环境变量 |
| `SessionEnd` | 会话结束 | 清理资源 |
| `PreToolUse` | 工具执行前 | 日志记录 |
| `PostToolUse` | 工具执行后（成功） | 自动格式化 |
| `PostToolUseFailure` | 工具执行后（失败） | 错误报告 |
| `PermissionRequest` | 请求权限时 | 自动审批/拒绝 |
| `PermissionDenied` | 权限被拒绝 | 审计日志 |
| `UserPromptSubmit` | 用户提交输入 | 输入预处理 |
| `FileChanged` | 文件变更 | 自动重新加载 |
| `CwdChanged` | 工作目录变更 | 环境切换 |
| `ConfigChange` | 配置变更 | 热重载 |
| `InstructionsLoaded` | CLAUDE.md 加载 | 注入额外指令 |

### 5.2 PermissionRequest Hook

最强大的 Hook 类型——可以程序化地控制权限决策：

**输入**:
```json
{
  "hook_event_name": "PermissionRequest",
  "tool_name": "Bash",
  "tool_input": { "command": "npm install lodash" }
}
```

**输出**:
```json
// 允许（可选修改输入）
{
  "decision": "allow",
  "reason": "npm install is safe",
  "updatedInput": { "command": "npm install lodash --save-exact" },
  "updatedPermissions": [
    { "type": "addRules", "rules": ["Bash(npm install *)"], "behavior": "allow" }
  ]
}

// 拒绝
{
  "decision": "deny",
  "reason": "This command is not allowed in production"
}

// 阻塞（触发 abort）
{
  "decision": "block",
  "reason": "Critical security violation detected"
}
```

**退出码含义**:
| 退出码 | decision | 行为 |
|--------|----------|------|
| 0 | allow | 权限通过 |
| 0 | (无输出) | 放行（继续后续检查） |
| 2 | deny/block | 权限拒绝 |

### 5.3 Hook 超时

```
工具 Hook: 10 分钟（TOOL_HOOK_EXECUTION_TIMEOUT_MS）
SessionEnd Hook: 1.5 秒（可通过 CLAUDE_CODE_SESSIONEND_HOOKS_TIMEOUT_MS 配置）
```

> **为什么 SessionEnd 这么短?** 用户按 Ctrl+C 退出时不应该等太久。如果 Hook 需要做耗时操作，应该在后台进行。

---

## 六、权限处理器

**文件**: `src/hooks/toolPermission/`

### 6.1 Interactive Handler（交互模式）

```
并行竞速:
  ├─ 用户对话框（本地终端）
  ├─ CCR Bridge（claude.ai 远程）
  ├─ Channel（Telegram/iMessage）
  ├─ Hooks
  └─ 分类器（Auto 模式）

第一个 claim() 成功的赢得决策权
其余自动 abort

200ms 宽限期:
  → 分类器指示器在 200ms 后才显示
  → 如果用户在 200ms 内点击，不显示分类器
  → 避免"分类器闪烁"

✅ 自动通过指示器:
  → 终端聚焦: 显示 3 秒
  → 终端失焦: 显示 1 秒
  → Esc 可提前关闭
```

### 6.2 Coordinator Handler（协调器模式）

```
串行执行:
  1. Hooks → 决策? → 返回
  2. 分类器 → 决策? → 返回
  3. 无决策 → 返回 null → 降级到对话框
```

### 6.3 权限日志

```typescript
// 审批事件
'tengu_tool_use_granted_in_config'          // 规则自动通过
'tengu_tool_use_granted_by_classifier'      // 分类器通过
'tengu_tool_use_granted_in_prompt_permanent' // 用户通过 + 保存规则
'tengu_tool_use_granted_in_prompt_temporary' // 用户通过（仅会话）
'tengu_tool_use_granted_by_permission_hook'  // Hook 通过

// 拒绝事件
'tengu_tool_use_denied_in_config'           // 规则拒绝
'tengu_tool_use_rejected_in_prompt'          // 用户拒绝 / Hook 拒绝
```

---

## 七、沙箱系统

**文件**: `src/utils/sandbox/sandbox-adapter.ts`

沙箱通过 `@anthropic-ai/sandbox-runtime` 限制 Bash 命令的文件系统和网络访问。

### 文件系统沙箱

```typescript
// Settings 中配置
sandbox: {
  filesystem: {
    allowRead: ['/home/user/project', '/usr/lib'],
    denyRead: ['/etc/shadow'],
    allowWrite: ['/home/user/project'],
    denyWrite: ['/home/user/project/.env']
  }
}
```

**路径解析规则**:

| 格式 | 权限规则解析 | 沙箱配置解析 |
|------|------------|------------|
| `//path` | 绝对路径 `/path` | 绝对路径 |
| `/path` | 设置相对 `${SETTINGS_DIR}/path` | 绝对路径（原样） |
| `~/path` | 展开到 home | 展开到 home |
| `./path` | 透传 | 设置相对 |

### 网络沙箱

```typescript
sandbox: {
  network: {
    allowManagedDomainsOnly: true  // 仅允许策略中的域名
  }
}
```

### 沙箱绕过检测

当使用 `--dangerously-skip-permissions` 时的安全检查：

```
检查清单:
  ├─ 是否 root/sudo? → 拒绝（除非在沙箱容器中）
  ├─ 是否在 Docker 中? → 允许
  ├─ 是否在 Bubblewrap 中? → 允许
  ├─ IS_SANDBOX 环境变量? → 允许
  └─ 无网络访问? → 允许
```

---

## 八、Settings 系统

**文件**: `src/utils/settings/`

### 8.1 设置源层级

```
┌─────────────────────────────────────────────────────────┐
│ 最高优先级                                               │
│                                                          │
│  1. CLI Arguments                                        │
│     --allowed-tools, --denied-tools                      │
│                                                          │
│  2. In-Session Changes                                   │
│     权限对话框中的"Always Allow"                          │
│                                                          │
│  3. Local Settings                                       │
│     .claude/settings.local.json（gitignored）             │
│                                                          │
│  4. Project Settings                                     │
│     .claude/settings.json（共享）                          │
│                                                          │
│  5. User Settings                                        │
│     ~/.claude/settings.json（全局）                        │
│                                                          │
│  6. Managed/Policy Settings                              │
│     ├─ /etc/claude-code/managed-settings.json  (Linux)   │
│     ├─ /etc/claude-code/managed-settings.d/*.json        │
│     ├─ /Library/Managed Preferences/...plist  (macOS)    │
│     └─ HKLM\SOFTWARE\Policies\ClaudeCode     (Windows)  │
│                                                          │
│ 最低优先级                                               │
└─────────────────────────────────────────────────────────┘
```

### 8.2 设置 JSON 结构

```json
{
  "permissions": {
    "allow": ["Bash(git *)", "Edit"],
    "deny": ["Bash(rm -rf *)"],
    "ask": ["Bash(npm publish *)"],
    "defaultMode": "default",
    "additionalDirectories": ["/shared/libs"]
  },
  "hooks": {
    "PreToolUse": [{
      "command": "echo $TOOL_NAME >> /tmp/tool-log.txt"
    }],
    "SessionEnd": [{
      "command": "git stash",
      "timeout": 5000
    }]
  },
  "sandbox": {
    "filesystem": {
      "allowRead": ["/project"],
      "allowWrite": ["/project/src"]
    }
  },
  "env": {
    "NODE_ENV": "development"
  }
}
```

### 8.3 MDM（企业管理）设置加载

```
加载优先级（First-Source-Wins）:
  1. Remote Managed Settings（远程配置）
  2. HKLM/plist（系统级策略）
  3. /etc/claude-code/managed-settings.json（文件）
  4. HKCU（用户级策略）

Drop-ins 支持:
  /etc/claude-code/managed-settings.d/
  ├── 01-security.json    ← 按字母序合并
  ├── 02-network.json
  └── 03-custom.json
```

### 8.4 CLAUDE.md 加载

```
从 CWD 向上遍历:
  /Users/dev/project/src/components/
  └─ .claude/                     ← 最深层
  /Users/dev/project/src/
  └─ .claude/
  /Users/dev/project/             ← Git 根目录
  └─ .claude/                     ← 通常在这里
  ~/.claude/                      ← 全局配置

文件搜索:
  ├─ 默认: ripgrep --files --hidden --follow --no-ignore --glob '*.md'
  ├─ 回退: 原生 Node.js 遍历
  └─ 超时: 3 秒

Worktree 回退:
  如果 worktree 缺少 .claude/ → 使用主仓库的拷贝
```

---

## 九、技能作用域保护

```
.claude/skills/my-skill/
  └─ SKILL.md

当技能 my-skill 获得文件访问权限时:
  ├─ Allow 规则范围: /.claude/skills/my-skill/**
  ├─ 不能访问: /.claude/skills/other-skill/**
  └─ 不能访问: /.claude/ 根目录

→ 防止一个技能的迭代过程授予对整个 .claude/ 的访问权限
```

---

## 十、关键安全设计模式

### 1. 原子决策（ResolveOnce）
```
并行竞速中的决策必须是原子的:
  claim() → 内部检查 + 标记，原子操作
  → 防止多个源同时"赢得"决策权
  → 只有第一个 claim 成功的有效
```

### 2. 危险模式自动剥离
```
Auto 模式进入时:
  findDangerousClassifierPermissions()
  → 扫描所有 Allow 规则
  → 剥离代码解释器模式（python:*, node:*）
  → 剥离 Agent allow 规则
  → 确保分类器不被绕过
```

### 3. 拒绝限流
```
分类器连续拒绝 3 次 → 降级到询问用户
分类器总拒绝 20 次 → 降级到询问用户
→ 防止分类器 bug 导致无限拒绝
→ 成功时重置连续拒绝计数
```

### 4. 文件路径安全
```
UNC 路径阻止:
  \\server\share → 拒绝
  → 防止 Windows NTLM 凭据泄露

路径遍历防护:
  ../../../etc/passwd → 沙箱阻止
  → 文件系统权限检查

参数注入防护:
  isSafeRefName() → 只允许 [a-z0-9/._+-@]
  → 防止 git 命令注入
```

### 5. 设置验证
```
Zod Schema 验证:
  ├─ 权限规则过滤无效条目
  ├─ 详细错误报告（含修复建议）
  ├─ 不拒绝整个文件（fail-open 单条）
  └─ 文档链接引导
```
