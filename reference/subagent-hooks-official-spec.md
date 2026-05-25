---
name: subagent-hooks-official-spec
description: SubagentStart/SubagentStop 官方规格——stdin 字段、exit code 行为、decision 模式，2026-05-25 从 code.claude.com/docs/en/hooks 抓取
metadata:
  type: reference
  project: 晓组织
  source: https://code.claude.com/docs/en/hooks
  fetched: 2026-05-25
---

# SubagentStart / SubagentStop 官方 Hook 规格

来源：https://code.claude.com/docs/en/hooks，2026-05-25 抓取。

## 事件概述

| 事件 | 触发时机 |
|------|---------|
| SubagentStart | 子 agent 被 spawn 时 |
| SubagentStop | 子 agent 完成时 |

两者都在 agentic loop 内触发，与 tool-call 事件同级。

## Matcher 语法

按 **agent type 名称** 匹配：

| Matcher 值 | 匹配对象 |
|-----------|---------|
| `general-purpose` | 默认子 agent |
| `Explore` | 内置 Explore agent |
| `Plan` | 内置 Plan agent |
| 自定义 agent 名称 | 自定义 agent frontmatter 中的 `name` 字段（不是文件名） |

支持：精确字符串、`|` 分隔列表、JS 正则（含非字母数字/下划线字符时）。

## 所有 Hook 事件的通用 stdin JSON 字段

以下字段在所有 hook 事件（含 SubagentStart/SubagentStop）的 stdin JSON 中都存在：

| 字段 | 说明 |
|------|------|
| `session_id` | 当前会话标识符 |
| `transcript_path` | 对话 JSON 文件路径 |
| `cwd` | hook 调用时的当前工作目录 |
| `permission_mode` | 当前权限模式（`"default"`, `"plan"`, `"acceptEdits"`, `"auto"`, `"dontAsk"`, `"bypassPermissions"`） |
| `effort` | 含 `level` 字段的对象（`"low"`, `"medium"`, `"high"`, `"xhigh"`, `"max"`）——在 tool-use 上下文中存在 |
| `hook_event_name` | 触发事件名（如 `"SubagentStart"`） |

## Subagent 专属额外字段

当以 `--agent` 运行或在子 agent 内部时，额外增加两个字段：

| 字段 | 说明 |
|------|------|
| `agent_id` | 子 agent 唯一标识符。仅在子 agent 调用内存在 |
| `agent_type` | Agent 名称（如 `"Explore"`）。对自定义 agent 是 frontmatter 的 `name` 字段，非文件名 |

## Exit Code 行为

| 事件 | 能 Block？ | Exit code 2 行为 |
|------|-----------|-----------------|
| **SubagentStart** | **不能** | 仅向用户显示 stderr（纯观察性质） |
| **SubagentStop** | **能** | 阻止子 agent 停止 |

### SubagentStart 详细行为

- **不能阻止子 agent spawn。** Hook 只是运行，可以通过 stderr 向用户显示信息。
- Exit code 0：抑制输出。
- 其他非零 exit code：stderr 作为 hook 错误通知显示。

### SubagentStop 详细行为

- Exit code 2：阻止子 agent 停止（使其保持运行）。
- Exit code 0 + 无输出：允许停止继续。
- 其他非零 exit code：显示 stderr 但不阻止。

## SubagentStart —— Context-Only 模式（无 Block）

```json
{
  "hookSpecificOutput": {
    "hookEventName": "SubagentStart",
    "additionalContext": "Make sure to follow the security review checklist"
  }
}
```

- `additionalContext`：字符串，添加到子 agent 对话上下文开头，位于第一条 prompt 之前。多个 hook 的 additionalContext 会拼接。
- **无 decision/block 控制可用。**

## SubagentStop —— Decision 控制

支持顶层 `decision` 模式：

```json
{
  "decision": "block",
  "reason": "Subagent must finish validation before stopping"
}
```

- `"decision": "block"`：阻止子 agent 停止
- `"reason"`：显示阻止原因的反馈
- 不传 `decision`（或 exit 0 无 JSON）：允许子 agent 停止

**SubagentStop 不支持 `additionalContext`。** 要向父会话注入上下文，改用 PostToolUse hook + Agent 工具 matcher。

## 通用 JSON Stdout 格式

所有 command 类型 hook 在 exit 0 时通过 stdout 返回 JSON：

### 通用字段（所有事件）

| 字段 | 默认 | 说明 |
|------|------|------|
| `continue` | `true` | 设为 `false` 则 Claude 在 hook 运行后完全停止处理 |
| `stopReason` | 无 | `continue` 为 `false` 时显示给用户的消息 |
| `suppressOutput` | `false` | 设为 `true` 则在 transcript 中隐藏 hook 的 stdout |
| `systemMessage` | 无 | 显示给用户的警告消息 |
| `terminalSequence` | 无 | 允许列表中的终端 escape 序列（OSC 0/1/2/9/99/777, BEL），用于桌面通知 |

### hookSpecificOutput 包装

事件特定的控制字段嵌套在 `hookSpecificOutput` 中，必含 `hookEventName`：

```json
{
  "hookSpecificOutput": {
    "hookEventName": "SubagentStart",
    "additionalContext": "Follow security guidelines for this task"
  }
}
```

```json
{
  "decision": "block",
  "reason": "Please also verify the edge cases before finishing"
}
```

> 注意：`hookSpecificOutput` 和顶层 `decision` 是两种不同的输出模式。SubagentStart 用前者（additionalContext），SubagentStop 用后者（decision: block）。SubagentStop 不支持 additionalContext。

### Exit Code 规则总结

| Exit Code | 含义 |
|-----------|------|
| **0** | 成功——stdout 上的 JSON 被解析用于决策 |
| **2** | Blocking 错误——stderr 显示给 Claude，操作被阻止（如果事件支持 block） |
| **其他** | Non-blocking 错误——stderr 第一行作为通知显示，执行继续 |

## 子 Agent Frontmatter 中的 Hooks

在 agent YAML frontmatter 中可定义 hooks。对子 agent，`Stop` hooks 会**自动转换为 `SubagentStop`**，因为那是子 agent 完成时触发的事件：

```yaml
---
name: security-reviewer
description: Review code for security issues
hooks:
  SubagentStop:
    - matcher: ""
      hooks:
        - type: command
          command: "./scripts/post-review.sh"
---
```

## 上下文行为

- **SubagentStart**：`additionalContext` 出现在子 agent 对话开头，第一条 prompt 之前。多个 hook 的值拼接。✅ 已实测确认（2026-05-25）。
- **SubagentStop**：**不支持 `additionalContext`**。要向父会话注入内容，需改用 PostToolUse hook（matcher: Agent 工具）代替。

## 官方 stdin 字段表（完整版，2026-05-25 Jina 抓取）

### SubagentStart 官方字段

在通用字段之外，SubagentStart 接收：

| 字段 | 说明 |
|------|------|
| `session_id` | 当前会话标识符 |
| `transcript_path` | 对话 JSON 路径 |
| `cwd` | hook 调用时的当前工作目录 |
| `hook_event_name` | 固定为 `"SubagentStart"` |
| `agent_id` | 子 agent 唯一标识符 |
| `agent_type` | Agent 名称（如 `"Explore"`、`"general-purpose"`、`"Plan"`，或自定义名） |

官方示例：
```json
{
  "session_id": "abc123",
  "transcript_path": "/Users/.../.claude/projects/.../xxx.jsonl",
  "cwd": "/Users/...",
  "hook_event_name": "SubagentStart",
  "agent_id": "agent-abc123",
  "agent_type": "Explore"
}
```

### SubagentStop 官方字段

在通用字段之外，SubagentStop 接收：

| 字段 | 说明 |
|------|------|
| `session_id` | 当前会话标识符 |
| `transcript_path` | **父会话**的转录路径 |
| `cwd` | 当前工作目录 |
| `permission_mode` | 当前权限模式 |
| `hook_event_name` | 固定为 `"SubagentStop"` |
| `stop_hook_active` | 布尔值，是否有 Stop hook 正在运行 |
| `agent_id` | 子 agent 唯一标识符 |
| `agent_type` | Agent 名称（用于 matcher 过滤） |
| `agent_transcript_path` | **子 agent 自己的转录文件路径**，在父转录目录的 `subagents/` 子目录下 |
| `last_assistant_message` | 子 agent 最终回复的文本内容（无需解析转录文件） |
| `background_tasks` | 父会话的后台任务数组（v2.1.145+） |
| `session_crons` | 父会话的 cron 任务数组（v2.1.145+） |
| `effort` | 含 `level` 字段的对象，当前 turn 的活跃 effort 级别 |

官方示例：
```json
{
  "session_id": "abc123",
  "transcript_path": "~/.claude/projects/.../abc123.jsonl",
  "cwd": "/Users/...",
  "permission_mode": "default",
  "hook_event_name": "SubagentStop",
  "stop_hook_active": false,
  "agent_id": "def456",
  "agent_type": "Explore",
  "agent_transcript_path": "~/.claude/projects/.../abc123/subagents/agent-def456.jsonl",
  "last_assistant_message": "Analysis complete. Found 3 potential issues...",
  "background_tasks": [],
  "session_crons": []
}
```

## 实测 stdin 字段（2026-05-25 调试 hook 捕获，与官方对照一致）

### SubagentStart 实测（含官方示例未列的 permission_mode + effort）

```json
{
  "session_id": "ae02e71c-...",
  "transcript_path": "/Users/dkhzymini/.claude/projects/.../xxx.jsonl",
  "cwd": "/Users/dkhzymini",
  "permission_mode": "bypassPermissions",
  "effort": {"level": "max"},
  "hook_event_name": "SubagentStart",
  "agent_id": "a5c75a0caebd1caf7",
  "agent_type": "Explore"
}
```

### SubagentStop 实测（与官方字段完全一致）

**比 SubagentStart 多 5 个字段：**

```json
{
  "session_id": "ae02e71c-...",
  "transcript_path": "/Users/dkhzymini/.claude/projects/.../xxx.jsonl",
  "cwd": "/Users/dkhzymini",
  "permission_mode": "bypassPermissions",
  "effort": {"level": "max"},
  "hook_event_name": "SubagentStop",
  "agent_id": "a851c8b0359675bc4",
  "agent_type": "Explore",
  "agent_transcript_path": "/Users/dkhzymini/.claude/projects/.../subagents/agent-xxx.jsonl",
  "last_assistant_message": "子 agent 的最终回复全文",
  "stop_hook_active": false,
  "background_tasks": [],
  "session_crons": []
}
```

**新增字段说明：**

| 字段 | 说明 | 对成本日志的价值 |
|------|------|----------------|
| `agent_transcript_path` | 子 agent 独立转录文件路径 | **可直接读取提取 token 消耗，无需 PM 手动报** |
| `last_assistant_message` | 子 agent 最终回复全文 | 可用于自动记录产出摘要 |
| `stop_hook_active` | 是否已有 stop hook 在运行 | 防止递归 |
| `background_tasks` | 后台任务列表 | — |
| `session_crons` | 会话 cron 列表 | — |

## 获取说明

初抓 https://code.claude.com/docs/en/hooks 时 WebFetch 小模型截断了页面末尾的 per-event input schema。2026-05-25 通过 Jina（r.jina.ai）补充抓取获得完整文档。以下 stdin 字段表为官方文档 + 实测双重验证。

## 实测验证清单（2026-05-25 五项摸排）

| # | 场景 | 结论 | 证据 |
|---|------|------|------|
| 1 | 后台 agent 触发 hook？ | ✅ SubagentStart/SubagentStop 均触发，字段与前台完全一致 | 实测 general-purpose 后台 agent |
| 2 | 失败/崩溃场景 | ⚠️ 工具错误不阻断 SubagentStop，转录 `stop_reason` 可区分正常/截断/异常 | 实测 agent 执行失败 Bash（exit 1），SubagentStop 正常触发 |
| 3 | worktree agent | ⚠️ 无法实测（需从晓组织 git repo 内 spawn）。机制推断：hook 与 worktree 无关，`agent_transcript_path` 不在 worktree 内 | 晓组织已初始化为 git repo（2026-05-25），待下次从项目目录内实测 |
| 4 | 转录文件写入时序 | ✅ SubagentStop 触发时文件已完整写入（实测 31,907 字节/8 行），无竞态 | hook 内 `wc -c` 实时检测 |
| 5 | additionalContext 注入 | ✅ 确认注入到子 agent 上下文，agent 能精确引用注入的标记文本 | Explore agent 回显了标记内容 |

## 对 Gap #8 的影响

- **Part A（成本日志）**：完全可行。SubagentStart 记时间+类型，SubagentStop 通过 `agent_transcript_path` 读取 token 消耗。转录 `stop_reason` 可区分正常/截断/异常。
- **Part B（活跃数软提醒）**：SubagentStart 不可 block spawn（平台限制）。只做计数+stderr 警告+additionalContext 提醒。SubagentStop 减计数。需僵尸清理（SubagentStop 崩溃时不触发）。

## subagent 转录 `stop_reason` 完整取值（Messages API）

来源：https://platform.claude.com/docs/en/api/messages，2026-05-25 抓取。

| 值 | 含义 | 成本日志处理 |
|---|------|------------|
| `end_turn` | 自然停止 | 正常完成 |
| `max_tokens` | 超过 token 上限被截断 | **截断**——产出不完整，标注 |
| `tool_use` | 中间消息（调用了工具，非最终） | 忽略，读最后一条消息的 stop_reason |
| `stop_sequence` | 自定义停止序列被触发 | 异常，标注 |
| `refusal` | 安全策略拦截 | 被拒绝，标注 |
| `pause_turn` | 长任务被暂停 | 极少见，标注 |

## subagent 转录 `usage` 完整字段

| 字段 | 说明 |
|------|------|
| `input_tokens` | 输入 token |
| `output_tokens` | 输出 token |
| `cache_creation_input_tokens` | 创建 cache 消耗的 input token |
| `cache_read_input_tokens` | 从 cache 读取的 input token |
| `service_tier` | `standard` / `priority` / `batch` |

> 总输入 token = `input_tokens` + `cache_creation_input_tokens` + `cache_read_input_tokens`
