---
name: subagent-official-spec
description: Claude Code Subagent 官方完整规格——code.claude.com/docs/en/sub-agents，2026-05-25 CDP 浏览器逐页提取
metadata:
  type: reference
  project: 晓组织
  source: https://code.claude.com/docs/en/sub-agents
  fetchMethod: CDP browser
  fetchDate: 2026-05-25
---

# Claude Code Subagent 官方完整规格

> 来源：https://code.claude.com/docs/en/sub-agents，2026-05-25 通过 CDP 浏览器完整提取。
> 配套页面：https://code.claude.com/docs/en/agents（Agents and parallel work 总览）
> Hook 相关：见 [Subagent Hook 官方规格](subagent-hooks-official-spec.md)

---

## 一、概述

Subagents are specialized AI assistants that handle specific types of tasks. Use one when a side task would flood your main conversation with search results, logs, or file contents you won't reference again: the subagent does that work in its own context and returns only the summary. Define a custom subagent when you keep spawning the same kind of worker with the same instructions.

Each subagent runs in its own context window with a custom system prompt, specific tool access, and independent permissions. When Claude encounters a task that matches a subagent's description, it delegates to that subagent, which works independently and returns results.

Subagents work within a single session. To run many independent sessions in parallel and monitor them from one place, see background agents. For sessions that communicate with each other, see agent teams.

Subagents help you:
- Preserve context by keeping exploration and implementation out of your main conversation
- Enforce constraints by limiting which tools a subagent can use
- Reuse configurations across projects with user-level subagents
- Specialize behavior with focused system prompts for specific domains
- Control costs by routing tasks to faster, cheaper models like Haiku

Claude uses each subagent's description to decide when to delegate tasks. When you create a subagent, write a clear description so Claude knows when to use it.

---

## 二、内置子代理（Built-in subagents）

Claude Code includes built-in subagents that Claude automatically uses when appropriate. Each inherits the parent conversation's permissions with additional tool restrictions.

**Explore and Plan skip your CLAUDE.md files and the parent session's git status to keep research fast and inexpensive.** Every other built-in and custom subagent loads both.

### Explore
- **Model:** Haiku (fast, low-latency)
- **Tools:** Read-only tools (denied access to Write and Edit tools)
- **Purpose:** File discovery, code search, codebase exploration
- **Thoroughness levels:** quick (targeted lookups), medium (balanced exploration), very thorough (comprehensive analysis)

### Plan
- Plan mode 下的代码库研究
- 不能嵌套生成子代理
- 只读工具

### General-purpose
- 继承主会话模型
- 所有工具
- 复杂多步骤任务，需要探索+修改

### Other built-in
- **statusline-setup**: 运行 `/statusline` 时自动调用，模型 Sonnet
- **claude-code-guide**: 回答 Claude Code 功能相关问题时自动调用，模型 Haiku

---

## 三、子代理作用域（Scope）

When multiple subagents share the same name, the higher-priority location wins.

| 优先级 | 作用域 | 位置 | 说明 |
|--------|--------|------|------|
| 1 (最高) | Managed settings | 组织管理员部署 | 所有项目强制生效 |
| 2 | `--agents` CLI flag | 命令行传入 JSON | 仅当前会话 |
| 3 | `.claude/agents/` | 项目级 | 可纳入版本控制，递归扫描。从 CWD 向上遍历发现 |
| 4 | `~/.claude/agents/` | 用户级 | 所有项目可用，递归扫描 |
| 5 (最低) | Plugin's `agents/` directory | 插件目录 | 随插件安装，递归扫描 |

**同名冲突规则：** 同一作用域内两个同名文件，保留一个丢弃另一个，无警告。子目录路径不影响身份——身份仅来自 `name` frontmatter 字段。name 值在整个树中必须唯一。

**插件子代理特殊规则：** 插件 `agents/` 目录的子文件夹会成为 scoped identifier 的一部分。例如 `agents/review/security.md` 在插件 `my-plugin` 中注册为 `my-plugin:review:security`。

**CLI 定义示例：**
```json
claude --agents '{
  "code-reviewer": {
    "description": "Expert code reviewer. Use proactively after code changes.",
    "prompt": "You are a senior code reviewer. Focus on code quality, security, and best practices.",
    "tools": ["Read", "Grep", "Glob", "Bash"],
    "model": "sonnet"
  }
}'
```
CLI 定义仅存在于该会话，不保存到磁盘。`prompt` 字段等同于文件形式中的 markdown body。

**`--add-dir` 添加的目录仅授予文件访问权限，不扫描子代理。**

---

## 四、子代理文件格式

子代理文件使用 YAML frontmatter + Markdown body：

```yaml
---
name: code-reviewer
description: Reviews code for quality and best practices
tools: Read, Glob, Grep
model: sonnet
---
You are a code reviewer. When invoked, analyze the code and provide
specific, actionable feedback on quality, security, and best practices.
```

- Frontmatter 定义元数据和配置
- Body 成为子代理的 system prompt（不是完整 Claude Code system prompt，仅此内容+环境补充细节）
- 子代理在父会话的当前工作目录中启动
- 子代理内 `cd` 命令不在 Bash/PowerShell 调用之间持久化，不影响父会话工作目录
- **重启会话才能加载磁盘上的修改**（通过 `/agents` 界面创建的立即生效）

---

## 五、全部 16 个 Frontmatter 字段

仅 `name` 和 `description` 是必填的。

| 字段 | 必填 | 说明 |
|------|------|------|
| `name` | **是** | 唯一标识符，小写字母+连字符。Hook 接收此值作为 `agent_type`。文件名不需要与此匹配 |
| `description` | **是** | 告诉 Claude 何时委派给此子代理 |
| `tools` | 否 | 可用工具白名单。省略则继承所有工具。要预加载 Skill 用 `skills` 字段而非在此列出 Skill |
| `disallowedTools` | 否 | 要移除的工具。从继承或指定的列表中移除。如果同时设置 tools 和 disallowedTools，先应用 disallowedTools |
| `model` | 否 | `sonnet` / `opus` / `haiku` / 完整 model ID（如 `claude-opus-4-7`）/ `inherit`。默认 `inherit` |
| `permissionMode` | 否 | `default` / `acceptEdits` / `auto` / `dontAsk` / `bypassPermissions` / `plan`。插件子代理忽略此字段 |
| `maxTurns` | 否 | 最大 agentic turns，达到后子代理停止 |
| `skills` | 否 | 启动时预加载到上下文的 skill 列表。注入完整 skill 内容，不只是描述。不能预加载设置 `disable-model-invocation: true` 的 skill。子代理仍可通过 Skill 工具调用未列出的 skill |
| `mcpServers` | 否 | 可用 MCP 服务器列表。每项可以是引用已配置服务器的字符串（如 `"github"`），也可以是内联定义的完整 MCP 服务器配置。插件子代理忽略此字段 |
| `hooks` | 否 | 作用域为此子代理的生命周期钩子。插件子代理忽略此字段。支持所有 hook 事件，最常用：PreToolUse / PostToolUse / Stop（运行时自动转为 SubagentStop） |
| `memory` | 否 | 持久记忆范围：`user` / `project` / `local`。启用后自动启用 Read/Write/Edit 工具。系统提示会包含记忆目录 MEMORY.md 的前 200 行或 25KB |
| `background` | 否 | 设为 `true` 则始终作为后台任务运行。默认 `false` |
| `effort` | 否 | 努力级别：`low` / `medium` / `high` / `xhigh` / `max`。覆盖会话 effort 级别。默认继承会话。可用级别取决于模型 |
| `isolation` | 否 | 设为 `worktree` 则在临时 git worktree 中运行——从默认分支（非父会话 HEAD）分支出的独立副本。未产生更改的 worktree 自动清理 |
| `color` | 否 | 显示颜色：`red` / `blue` / `green` / `yellow` / `purple` / `orange` / `pink` / `cyan` |
| `initialPrompt` | 否 | 当此 agent 作为主会话运行时（通过 `--agent` 或 `agent` 设置），自动提交为首个 user turn。命令和 skill 会被处理。追加到用户提供的 prompt 之前 |

---

## 六、模型选择（Model resolution）

`model` 字段控制子代理使用的 AI 模型：
- **Model alias:** `sonnet` / `opus` / `haiku`
- **Full model ID:** 如 `claude-opus-4-7` 或 `claude-sonnet-4-6`
- **inherit:** 使用与主会话相同的模型
- **省略:** 默认 `inherit`

Claude Code 按以下优先级解析子代理模型：
1. `CLAUDE_CODE_SUBAGENT_MODEL` 环境变量（如设置）
2. 每次调用的 model 参数
3. 子代理定义的 `model` frontmatter
4. 主会话模型

---

## 七、权限模式（Permission modes）

子代理继承父会话的权限上下文，可以覆盖模式，但父模式优先时除外。

| 模式 | 行为 |
|------|------|
| `default` | 标准权限检查+提示 |
| `acceptEdits` | 自动接受文件编辑和工作目录/common filesystem 命令 |
| `auto` | 后台分类器审查命令和受保护目录写入 |
| `dontAsk` | 自动拒绝权限提示（显式允许的工具仍可用） |
| `bypassPermissions` | 跳过所有权限提示（根/家目录删除仍提示） |
| `plan` | Plan mode（只读探索） |

**关键规则：**
- 父用 `bypassPermissions` 或 `acceptEdits` 时，子代理**不能覆盖**——父模式优先
- 父用 `auto` mode 时，子代理继承 `auto` mode，其 `permissionMode` frontmatter 被忽略——分类器以与父会话相同的 block/allow 规则评估子代理工具调用
- `bypassPermissions` 使用时需谨慎——允许子代理无审批执行操作，包括对 `.git`、`.claude`、`.vscode`、`.idea`、`.husky` 的写入

---

## 八、工具控制

### 可用工具
子代理默认继承父会话的所有工具。通过 `tools`（允许列表）或 `disallowedTools`（拒绝列表）限制。

```yaml
# 只允许这些工具
tools: Read, Grep, Glob, Bash

# 或：继承所有，只移除写入能力
disallowedTools: Write, Edit
```

如果同时设置两者，先应用 `disallowedTools`，再从剩余池中解析 `tools`。在两者中都列出的工具被移除。

### 限制可 spawn 的子代理类型（Agent tool type restriction）
使用 `Agent(agent_type)` 语法在 tools 字段中限制可 spawn 的子代理类型：

```yaml
---
name: coordinator
description: Coordinates work across specialized agents
tools: Agent(worker, researcher), Read, Bash
---
```

这是允许列表——只能 spawn `worker` 和 `researcher`。尝试 spawn 其他类型会失败。
- 允许所有子代理不加限制：`tools: Agent, Read, Bash`
- 完全禁止 spawn：省略 `Agent`
- 此限制仅对通过 `claude --agent` 运行的主线程生效。子代理不能 spawn 其他子代理，因此 `Agent(agent_type)` 在子代理定义中无效果

### 通过 permissions.deny 禁用特定子代理
```json
{
  "permissions": {
    "deny": ["Agent(Explore)", "Agent(my-custom-agent)"]
  }
}
```
也可用 CLI flag：`claude --disallowedTools "Agent(Explore)"`

---

## 九、MCP 服务器作用域

使用 `mcpServers` 字段为子代理提供 MCP 服务器访问。支持两种形式：

```yaml
mcpServers:
  # 内联定义：仅此子代理可用，启动时连接，完成时断开
  - playwright:
      type: stdio
      command: npx
      args: ["-y", "@playwright/mcp@latest"]
  # 字符串引用：复用父会话的连接
  - github
```

内联定义使用与 `.mcp.json` 相同的 schema（stdio/http/sse/ws），以服务器名称为 key。主会话不会获取内联定义的 MCP 工具。

`mcpServers` 字段在两个上下文中都生效：
- 作为子代理（通过 Agent 工具或 @-mention spawn）
- 作为主会话（通过 `--agent` 或 `agent` 设置启动）

---

## 十、Memory（持久记忆）

`memory` 字段为子代理提供跨会话持久化的记忆目录：

```yaml
---
name: code-reviewer
description: Reviews code for quality and best practices
memory: user
---
```

| Scope | 位置 | 用途 |
|-------|------|------|
| `user` | `~/.claude/agent-memory/<name>/` | 跨所有项目积累知识 |
| `project` | `.claude/agent-memory/<name>/` | 项目特定，通过版本控制共享 |
| `local` | `.claude/agent-memory-local/<name>/` | 项目特定，但不应纳入版本控制 |

启用后：
- 子代理系统提示包含读写记忆目录的指令
- 子代理系统提示包含记忆目录 MEMORY.md 的前 200 行或 25KB（以先到者为准），并附有超限时整理 MEMORY.md 的指令
- Read/Write/Edit 工具自动启用

官方推荐 `project` 为默认 scope。

---

## 十一、Skills 预加载

```yaml
skills:
  - api-conventions
  - error-handling-patterns
```

- 注入完整 skill 内容到子代理上下文，不只是描述
- 子代理仍可通过 Skill 工具调用未列出的 skill
- 不能预加载设置了 `disable-model-invocation: true` 的 skill
- 如果列出的 skill 缺失或禁用，跳过并记录警告到 debug log
- 要阻止子代理调用 skill，从 tools 中移除 Skill 或加入 disallowedTools

---

## 十二、Hooks（子代理级）

在子代理 frontmatter 中定义 hooks——仅在子代理活跃时运行，完成时清理。支持所有 hook 事件。

```yaml
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/validate-command.sh"
  PostToolUse:
    - matcher: "Edit|Write"
      hooks:
        - type: command
          command: "./scripts/run-linter.sh"
```

Stop hooks 在运行时自动转换为 SubagentStop 事件。

**Frontmatter hooks 也适用于主会话模式（`--agent` 或 `agent` setting）——会与 settings.json 中定义的 hooks 并肩运行。**

**项目级 SubagentStart/SubagentStop hooks**（在 settings.json 中配置）：

| 事件 | Matcher 输入 | 触发时机 |
|------|-------------|---------|
| SubagentStart | Agent type 名称 | 子代理开始执行时 |
| SubagentStop | Agent type 名称 | 子代理完成时 |

---

## 十三、Worktree 隔离

`isolation: worktree` 让子代理在临时 git worktree 中运行——完整仓库副本在独立目录，独立分支和 git index。

- 从默认分支（非父会话 HEAD）分支
- 未产生更改的 worktree **自动清理**
- 支持非 git VCS 通过 `WorktreeCreate`/`WorktreeRemove` hooks

---

## 十四、上下文加载规则

### 非 Fork 子代理启动时加载：
1. **System prompt:** agent 自身 prompt + Claude Code 追加的环境细节（不是完整 Claude Code system prompt）
2. **Task message:** Claude 委派时写的委派 prompt
3. **CLAUDE.md 和 memory:** 主会话加载的每一级记忆层级（`~/.claude/CLAUDE.md`、项目规则、CLAUDE.local.md、托管策略文件）。**Explore 和 Plan 跳过此项**
4. **Git status:** 父会话启动时拍摄的快照（非 git 仓库或 `includeGitInstructions: false` 时不存在）。**Explore 和 Plan 跳过此项**
5. **Preloaded skills:** agent 的 `skills` 字段列出的 skill 的完整内容

**Explore 和 Plan 是唯一跳过 CLAUDE.md 和 git status 的代理。没有 frontmatter 字段或 per-agent 设置可以更改。**

### 子代理不加载：
- 父会话对话历史
- 父会话已调用的 skill
- 父会话已读取的文件

### Fork 子代理例外：
Fork 继承完整对话历史，不启动全新上下文。详见 §十六。

---

## 十五、调用方式

三种方式，从建议到强制逐级升级：

1. **自然语言** — 在 prompt 中提到子代理名称，Claude 决定是否委派
2. **@-mention** — 输入 `@` 从自动完成中选择子代理，**保证**该子代理一定运行
3. **会话级** — `claude --agent <name>` 整个会话使用该子代理的 system prompt、工具限制和模型。也可在 settings.json 中设置 `"agent": "code-reviewer"`

@-mention 语法：
- 本地子代理：`@agent-<name>`
- 插件子代理：`@agent-<scoped-name>`，如 `@agent-my-plugin:code-reviewer`

---

## 十六、Fork 模式（实验性）

需要 Claude Code v2.1.117+，设置 `CLAUDE_CODE_FORK_SUBAGENT=1`。

Fork 是一种继承完整对话历史的子代理——不启动全新上下文。

| 维度 | Fork | 普通子代理 |
|------|------|-----------|
| 上下文 | 完整对话历史 | 全新上下文+传入的 prompt |
| System prompt 和工具 | 同主会话 | 来自子代理定义文件 |
| 模型 | 同主会话 | 来自 model 字段 |
| 权限 | 提示出现在终端 | 后台运行时自动拒绝 |
| Prompt cache | 与主会话共享 | 独立 cache |

开启 Fork 模式后的三个变化：
1. Claude 用 Fork 代替 general-purpose 子代理
2. 所有子代理 spawn 都在后台运行
3. `/fork` 命令 spawn 一个 Fork

**限制：** Fork 不能再 spawn Fork。

---

## 十七、前后台运行

- **前台（Foreground）：** 阻塞主会话直到完成。权限提示传递给你
- **后台（Background）：** 并发运行，你可继续工作。使用已授予的权限。需要提示的权限自动拒绝

Claude 根据任务决定前台/后台。你也可以：
- 说 "run this in the background"
- 按 Ctrl+B 后台化运行中的任务
- 设 `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=1` 禁用所有后台任务

---

## 十八、子代理恢复（Resume subagents）

每次子代理调用创建新实例（全新上下文）。要继续已有工作而非重新开始，要求 Claude **resume** 子代理：

```
Use the code-reviewer subagent to review the authentication module
[Agent completes]
Continue that code review and now analyze the authorization logic
[Claude resumes the subagent with full context from previous conversation]
```

恢复的子代理保留完整对话历史（所有工具调用、结果、推理），从停止处继续。

需要启用 Agent Teams（`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`）——Claude 使用 SendMessage 工具恢复。

子代理转录文件路径：`~/.claude/projects/{project}/{sessionId}/subagents/agent-{agentId}.jsonl`

转录独立于主会话：
- 主会话压缩时子代理转录不受影响
- 会话持久化时子代理转录保留
- 按 `cleanupPeriodDays` 设置自动清理（默认 30 天）

---

## 十九、自动压缩（Auto-compaction）

子代理使用与主会话相同的逻辑支持自动压缩。默认在 ~95% 容量时触发。设 `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` 调整（如 `50`）。

压缩事件记录在子代理转录文件中：
```json
{
  "type": "system",
  "subtype": "compact_boundary",
  "compactMetadata": {
    "trigger": "auto",
    "preTokens": 167189
  }
}
```

---

## 二十、禁止嵌套 Spawn

**Subagents cannot spawn other subagents.** 所有 spawn 必须从主会话发起。如果需要嵌套委派，使用 Skills 或从主会话串行链接子代理。

---

## 二十一、子代理 vs 主会话 vs Skills

| 用主会话 | 用子代理 | 用 Skills |
|----------|---------|-----------|
| 需频繁来回或迭代优化 | 会产生大量输出不要进主上下文 | 要可复用 prompt/工作流 |
| 多阶段共享重要上下文 | 要强制执行特定工具限制或权限 | 在主会话上下文中运行 |
| 快速、针对性修改 | 工作是自包含的、可返回摘要 | |
| 延迟敏感（子代理需时间收集上下文） | | |

---

## 二十二、插件子代理安全限制

For security reasons, plugin subagents do NOT support:
- `hooks`
- `mcpServers`
- `permissionMode`

这些字段在从插件加载 agent 时被忽略。如需使用，将 agent 文件复制到 `.claude/agents/` 或 `~/.claude/agents/`。

---

## 二十三、子代理定义可复用为 Agent Teams 队友

子代理定义对所有作用域在 Agent Teams 中也可用。spawn 队友时引用子代理类型名称——队友沿用其 tools 白名单和 model，定义的 body 附加到队友的系统提示中。

**注意：** skills 和 mcpServers frontmatter 在作为队友运行时**不生效**——队友从项目和用户设置加载。

---

## 关联文件

- [小小克使用指南](~/projects/小小克/subagent-guide.md) — 基于本文档写的完整使用指南（2026-05-24）
- [Subagent Hook 官方规格](subagent-hooks-official-spec.md) — SubagentStart/SubagentStop 官方文档（2026-05-25）
- [晓组织操作手册](~/projects/晓组织/operations.md) — Spawn 约定
- [晓组织协议](~/projects/晓组织/protocol.md) — 角色体系
