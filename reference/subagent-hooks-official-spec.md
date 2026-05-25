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

SubagentStop 的 `additionalContext`：文本作为系统提醒注入。超过 10,000 字符→保存到文件，替换为文件路径预览。

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
    "hookEventName": "SubagentStop",
    "additionalContext": "...",
    "decision": "block",
    "reason": "..."
  }
}
```

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

- **SubagentStart**：`additionalContext` 出现在子 agent 对话开头，第一条 prompt 之前。多个 hook 的值拼接。
- **SubagentStop + additionalContext**：文本作为系统提醒注入 Claude 上下文。超 10,000 字符则保存到文件，替换为文件路径预览。

## 实测 stdin 字段（2026-05-25 调试 hook 抓取）

### SubagentStart（实测 + 官方对照）

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

### SubagentStop（实测 + 官方对照）

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

## 截断说明（官方文档）

官方文档中 SubagentStart 和 SubagentStop 的完整 per-event input schema 段落在提供的页面内容中被截断。以上字段表已通过 2026-05-25 调试 hook 实测填补，原始数据已提取记录于此。

## 对 Gap #8 的影响

- **Part A（成本日志）**：完全可行。SubagentStart 记时间+类型，SubagentStop 记完成时间。
- **Part B（活跃数硬挡）**：原方案（SubagentStart exit 2 拒绝 spawn）**不可行**。SubagentStart 只能观察+警告（stderr + additionalContext），不能硬挡。需要替代方案。
