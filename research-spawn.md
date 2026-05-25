---
name: spawn-mechanisms-research
description: Spawn 机制深度调研——手动 Agent() vs Agent Teams vs Coordinator Mode vs Fork，2026-05-24 补充
metadata:
  type: reference
  project: 晓组织
  originSessionId: 630c2a48-1791-4ce8-95f9-050d4a44c361
---

# Spawn 机制深度调研

2026-05-24，逐条过操作手册 spawn 约定时发现信息缺口，补充搜索。与 [research.md](research.md) 互补——research.md 覆盖多 Agent 生态全景，本文件聚焦 spawn 机制的决策依据。

相关：[operations.md](operations.md) spawn 约定 | [小小克使用指南](~/projects/小小克/subagent-guide.md) Subagent 技术细节

---

## 一、四套并行机制全景

Claude Code 里能并行干活的办法目前有四套：

| 机制 | 怎么用 | 状态 | 启用方式 |
|------|--------|------|---------|
| 手动 Agent() spawn | PM 用 Agent 工具一个一个派人 | ✅ 正式功能，始终可用 | 无需额外配置 |
| Agent Teams | 建团队 + 共享任务列表，队友自组织领活、互发消息 | ⚠️ 实验性（Research Preview，2026-02） | `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` |
| Coordinator Mode | 协调者只保留 Agent/SendMessage/TaskStop 三个工具，不能读文件改代码 | ⚠️ 实验性 | `CLAUDE_CODE_COORDINATOR_MODE=1` |
| Fork 模式 | 子 Agent 继承对话历史，复用 prompt cache，v2.1.117+ | ⚠️ 实验性 | `CLAUDE_CODE_FORK_SUBAGENT=1` |

来源：Anthropic 官方文档 + morphllm.com 2026 年深度分析 + 社区实测

---

## 二、官方判断标准

来源：docs.anthropic.com，code.claude.com/docs

> "Use subagents when you need quick, focused workers that report back. Use agent teams when teammates need to share findings, challenge each other, and coordinate on their own."

翻译：需要快速专注的工人把结果报回来 → Subagents。需要队友之间互相分享发现、质疑对方、自己协调分工 → Agent Teams。

### Subagents vs Agent Teams 五维对比（官方）

| 维度 | Subagents | Agent Teams |
|------|-----------|-------------|
| 上下文 | 独立上下文，结果返回给调用者 | 独立上下文，完全独立运行 |
| 通信 | 只向主 Agent 汇报结果 | 队友之间直接互发消息 |
| 协调 | 主 Agent 管理所有工作 | 共享任务列表 + 自协调 |
| 最适合 | 专注任务，只要结果 | 复杂工作，需要讨论协作 |
| Token 成本 | 较低：结果摘要返回主上下文 | 较高：每个队友是独立 Claude 实例 |

来源：docs.anthropic.com，research.md §15.3 已验证

---

## 三、Coordinator Mode——萧何的工具箱

### 是什么

`CLAUDE_CODE_COORDINATOR_MODE=1` 开启后，协调者 Agent 只保留三个工具：

- **Agent** — 派人干活
- **SendMessage** — 给队友发消息
- **TaskStop** — 叫停任务

不能读文件、不能改代码、不能执行 shell 命令。工人保留完整工具集（减去协调工具）。

### 跟晓组织的对应

萧何的定义就是"不写代码、不深入执行"。Coordinator Mode 从工具层面强制了这个边界——不是靠 prompt 约束"你别写代码"，而是根本不给他读写文件的工具。

来源：Claude Code 源码分析（github.com/alejandrobalderas/claude-code-from-source）+ Anthropic 工程博客

---

## 四、Orchestrator-Workers 模式的实测数据

来源：Anthropic 内部研究 + LangChain 评测

### 性能

- Opus 4 指挥 + Sonnet 4 子代理，在内部研究基准上比单人 Opus 4 高 **90.2%**
- Subagent 模式在多域查询场景下比 skills 模式省 **67%** token（LangChain 实测）

### 成本

| 模式 | Token 倍数（相对单人聊天） |
|------|--------------------------|
| 手动 Subagents | ~1.75x |
| 手动 Panes | ~2x |
| Agent Teams | ~3.5x |
| 重度多 Agent | ~15x |

来源：Anthropic 工程博客 + morphllm.com 2026 年汇总

### 核心原则

> "Start with the simplest solution, only add complexity when actually needed."
> — Anthropic 官方建议

> "Token budget IS architecture."
> — 社区共识，research.md §7 已收录

> "Don't add an agent unless you have to."
> — Anthropic 工程团队经验

---

## 五、Agent Teams 详细画像

### 架构四件套

| 组件 | 职责 |
|------|------|
| Team Lead | 主会话，建团队、分任务、合成结果 |
| Teammates | 独立 Claude Code 实例，各有 1M token 上下文 |
| Task List | 磁盘 JSON 文件，文件锁防竞态，三状态流转 |
| Mailbox | 内存消息系统，支持 1:1 和广播，自动投递 |

### 内置质量闸

**Plan Approval**：队友先出计划 → 发批准请求给 lead → lead 批准/拒绝 → 拒绝则队友留在 plan mode 修改 → 批准后执行。Lead 可设置批准标准。

### 团队 Hook 事件

- TeammateIdle — 队友空闲通知
- TaskCreated — 任务创建通知
- TaskCompleted — 任务完成通知

均不支持 matcher 过滤。

### 已知限制（官方确认）

1. 不支持 `/resume` — 会话中断后队友不恢复，可能重复工作
2. 僵尸队友 — 配置里存在但已失效，无法强制终止
3. 关闭缓慢 — 需逐个关队友再清团队，顺序不可逆
4. 不支持嵌套团队 — 队友不能再建子团队
5. Lead 不可转让
6. 分屏显示需要 tmux 或 iTerm2，不支持 VS Code/Windows Terminal/Ghostty
7. 队友状态行始终显示 0 工具调用/0 token
8. spawn 报告成功但进程未实际创建（偶发 bug）
9. 生产部署不推荐

### 最佳实践（官方 8 条）

1. 给队友足够的上下文
2. 从 3-5 人开始
3. 每人 5-6 个任务
4. 任务粒度适中
5. 等队友完成——lead 有时不等队友就开始自己干
6. 从调研和审查练手——新人先做边界清晰的不写代码任务
7. 避免文件冲突——每人负责不同的文件集
8. 监控和引导——随时检查进度，纠偏，随时合成发现

### 成本参考

- Claude Max ($200/月) 计划内基本"免费"
- API 模式：5 人团队跑 2 小时约 $15-30
- 模型建议：Lead 用 Opus，队友用 Sonnet，可降本 40-50%

来源：docs.anthropic.com + morphllm.com + dev.to 实战系列

---

## 六、手动 Agent() spawn 详细画像

### 生命周期

```
AgentTool 调用
  → resolveAgentTools（按允许/禁止列表过滤工具）
  → runAgent（设置上下文、MCP 服务器、系统提示词）
  → queryLoop（标准 agentic 循环，在独立上下文中）
  → 结果提取并返回给父会话
  → 清理（上下文丢弃）
```

来源：Claude Code 源码分析（github.com/alejandrobalderas/claude-code-from-source）

### 自定义子 Agent

定义在 `.claude/agents/` 下，Markdown + YAML frontmatter。关键字段：

| 字段 | 作用 |
|------|------|
| `model` | 模型指定（sonnet/opus/haiku） |
| `tools` / `allowed-tools` | 工具白名单 |
| `disallowedTools` | 工具黑名单 |
| `permissionMode` | 权限模式 |
| `maxTurns` | 最大轮数限制 |
| `skills` | 加载的 skills |
| `background` | 后台运行（v2.1.49+，后台子代理权限请求自动拒绝，可 Ctrl+B 中途切后台） |
| `isolation` | worktree 隔离 |
| `memory` | 持久记忆配置 |

作用域优先级（从高到低）：Managed settings > `--agents` CLI > `.claude/agents/`（项目级）> `~/.claude/agents/`（用户级）> Plugin `agents/`

### 硬限制

- **子代理不能 spawn 子代理**（深度=1 强制）
- 所有 spawn 必须从主会话发起
- Explore/Plan 不读 CLAUDE.md

### 权限模式

| 模式 | 行为 |
|------|------|
| `default` | 标准权限提示 |
| `acceptEdits` | 自动批准编辑 |
| `auto` | 全自动 |
| `bypassPermissions` | 绕过权限检查 |
| `dontAsk` | 不询问 |
| `plan` | 先计划后执行 |

### 五种调用方式

1. 自然语言："用 code-reviewer 审查 auth.ts"
2. @-mention：`@agent-code-reviewer`
3. `--agent` CLI flag：`claude --agent code-reviewer`
4. 自动委派：Claude 根据任务描述和子代理 description 自动判断
5. 显式 Agent 工具调用：PM 直接用 Agent tool spawn

### 通信渠道

| 渠道 | 机制 | 场景 |
|------|------|------|
| 前台生成器 | 直接 async iterator | 父会话等待完成 |
| 磁盘输出文件 | JSONL 转录 + outputOffset | 父会话或观察者增量读取 |
| 任务通知 | XML 注入父会话上下文 | 完成提醒 + 摘要 + token 统计 |
| 待处理消息队列 | pendingMessages 数组 | SendMessage 给运行中的 agent |

来源：Claude Code 源码分析 + morphllm.com + tembo.io

### 后台执行

v2.1.49+ 支持 `background: true`（YAML frontmatter 或 Agent 工具参数）：
- 后台子代理与父会话并行运行
- 启动前未预批准的权限请求自动拒绝
- 可通过 `Ctrl+B` 中途切后台
- 全局禁用：`CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=1`
- 适用场景：长时间任务（测试套件、安全审计、依赖更新），父会话不被阻塞

### 广播/对等模式

SendMessage 的 `*` 通配符支持广播。Agent 之间使用结构化消息类型（`shutdown_request`、`plan_approval_response`）。晓组织暂不适用（以 PM 为中心的星型拓扑，非对等网）。

---

## 七、社区实战参考

### 16-Agent C 编译器案例

Anthropic 工程团队：
- 16 个 Agent，2 周
- 100K+ 行 Rust
- $20K API 费用
- 20 亿 input token
- ~2000 个会话
- GCC torture test 99% 通过率
- 能编译 Linux kernel/QEMU/FFmpeg/SQLite/PostgreSQL/Redis
- 在 x86/ARM/RISC-V 上启动 Linux 6.9

来源：Anthropic 工程博客

### 三种并行方式选择框架

| 方式 | 通信 | Token 成本 | 最适合 |
|------|------|-----------|--------|
| Subagents | 只向 parent 汇报 | 最低（~1.75x） | 独立调研、并行文件分析、快速专注任务 |
| Agent Teams | 队友直接互发消息 | 最高（~3.5x） | 多文件协调功能、跨层改动、竞争假设调试 |
| 手动 Panes | 你手动传递信息 | 中等（~2x） | 你想要最大控制权时；4-8 个并行任务 |

### 决策框架：两个核心问题

1. **Agent 之间需要在中途共享发现吗？**
   - 需要 → Agent Teams
   - 不需要 → Subagents（最便宜最简单）

2. **多少并行任务？**
   - 1-3 个 → Subagents
   - 4-8 个 → 手动 Panes 或 Agent Teams
   - 8+ 个 → Agent Teams（没人能同时追踪 8 个 pane）

来源：morphllm.com 2026

### 常见翻车模式

1. **为独立任务用 Agent Teams** — 花 ~2x 额外成本买你用不上的协调功能
2. **需要协作用 Subagents** — 父会话变成瓶颈，在 agent 之间当传话筒
3. **Agent 太多** — 更多 agent ≠ 更快；3-5 是实测最优区间
4. **跳过规划** — 没想清楚就建团队，前 10-15 分钟浪费在探索而不是执行

### 开源项目教训

- 47 次 handoff，$9.12 账单（OpenAI SDK 真实反模式，2026）
- 一个全流程任务可能消耗 100 次模型调用
- 多 Agent 系统经常打不过强单 Agent 基线（如 o1）
- MetaGPT、AG2 实际部署中频繁出现任务交接失败、Agent 误解共享目标、重复生成代码

来源：research.md §7 + §12.2 + morphllm.com

---

## 八、跟晓组织现状的差距分析

| # | 发现 | 晓组织现状 | 动作 |
|---|------|-----------|------|
| 1 | Coordinator Mode 工具限制天然匹配萧何 | spawn 约定全靠 prompt 约束 PM 不写代码 | **不开**：等 10 次任务后评估，2026-05-25（详见 operations.md spawn 约定） |
| 2 | 手动 spawn 是正确主轴 | operations.md 第一章全是手动 Agent 调用 | **验证通过**，不需要改方向 |
| 3 | Agent Teams 不适合晓组织 | 曾在 spawn 约定里混提 Plan Approval | **关掉**：.zshrc env var 关掉 + 文档统一"不内置"，2026-05-25 |
| 4 | 3-5 队友上限多源验证一致 | operations.md 主流程写"最多 8 个" | **修**：改为最多 5 个 |
| 5 | Subagent 模式 67% 省 token（vs skills） | 无相关数据 | **标注**：成本优势数据，已记录 research-spawn §四，2026-05-25 |
| 6 | 子代理可复用为 Agent Teams 队友 | 不知道 | **标注**：已知，暂不启用，2026-05-25 |
| 7 | Fork 模式实验性，prompt cache 复用 | spawn 约定标注"暂不内置" | **标注**：等稳定后评估，2026-05-25 |
| 8 | Subagent memory 字段可配置 | 没提 | **标注**：暂不配置，等 10 次任务后评估，2026-05-25 |

---

## 九、结论

### 晓组织的 spawn 方向

**手动 Agent() spawn 为主轴。** 理由：
1. 晓组织 PM（萧何）是指挥型——调度每一步、审每一步产出、决定下一步
2. Agent Teams 是自组织型——扔任务列表让大家自己领活协调
3. 两种模型本质上不兼容，不存在"二选一"
4. 官方建议：简单、专注任务 → Subagents。恰好是晓组织的场景

### 可选的增强（2026-05-25 更新）

- **Coordinator Mode**：物理强制萧何不写代码 → **暂不开**，等 10 次任务后评估
- **自定义子 Agent 定义文件**：`.claude/agents/` 下 5 角色已建 ✅（architect/qa-reviewer/compliance/finance/engineer）
- **工作树隔离**：engineer 已配 `isolation: worktree` ✅，其余角色不需要

### 暂不内置的

- Agent Teams（实验性、不适合指挥型协作、不支持 /resume）
- Fork 模式（实验性）
- Agent-based Hook（等 QA 流程跑熟后考虑自动化）

---

## 参考来源

- [Agent Teams vs Subagents in Claude Code (morphllm.com, 2026)](https://www.morphllm.com/agent-teams-vs-subagents)
- [Claude Code Subagents: Why the Best Coding Agents Delegate (morphllm.com, 2026)](https://www.morphllm.com/claude-code-subagents)
- [AI Agent Orchestration: Claude Code Agent Teams & Multi-Agent Coding (morphllm.com, 2026)](https://www.morphllm.com/ai-agent-orchestration)
- [Claude Code from Source — Ch10 Coordination (github.com)](https://github.com/alejandrobalderas/claude-code-from-source/blob/main/book/ch10-coordination.md)
- [Claude Code Subagents: A 2026 Practical Guide (tembo.io)](https://www.tembo.io/blog/claude-code-subagents)
- [Scaling Claude Code agents across your engineering team (portkey.ai)](https://portkey.ai/blog/claude-code-agents/)
- [Claude Code 多 Agent 协作：Subagents 和 Agent Teams 怎么选？(腾讯云)](https://cloud.tencent.com.cn/developer/article/2652960)
- [Turning Solo Development into Team Development with Claude Code Agent Teams (dev.to)](https://dev.to/kochan/turning-solo-development-into-team-development-with-claude-code-agent-teams-part-8-589i)
- [Thoughtworks Technology Radar: Team of coding agents](https://www.thoughtworks.com/radar/techniques/team-of-coding-agents)
- Anthropic 官方文档（docs.anthropic.com, code.claude.com/docs）
- Anthropic 工程博客（anthropic.com/engineering）
