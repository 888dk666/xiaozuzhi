---
name: subagent-parallel-official-spec
description: Claude Code 并行机制总览官方规格——code.claude.com/docs/en/agents，2026-05-25 Jina 抓取
metadata:
  type: reference
  project: 晓组织
  source: https://code.claude.com/docs/en/agents
  fetchMethod: Jina (r.jina.ai)
  fetchDate: 2026-05-25
---

# Claude Code 并行机制总览官方规格

> 来源：https://code.claude.com/docs/en/agents，2026-05-25 Jina 抓取。

---

## 五种并行方式对比

| 方式 | 是什么 | 什么时候用 |
|------|--------|-----------|
| **Subagents** | 一个会话内的委派工人，在自己的上下文中做侧任务，返回摘要 | 侧任务会产生大量搜索结果/日志/文件内容，你不想淹没问题上下文 |
| **Agent view** | 一个屏幕派发和监控后台运行的会话，通过 `claude agents` 打开。Research preview | 你有几个独立任务，想分发出去、随时查看状态、只在需要时介入 |
| **Agent teams** | 多个协调会话，共享任务列表+agent 间消息，由 lead 管理。实验性，默认关闭 | 你想让 Claude 拆分项目、分配任务、保持工人同步 |
| **Worktrees** | 独立 git checkout，并行会话互不触碰文件 | 你自己跑多个会话，或子代理编辑重叠文件 |
| **`/batch`** | 把一个大型变更拆成 5-30 个 worktree 隔离的子代理，每个开 PR | 可以用一条指令描述的仓库级迁移或机械重构 |

所有方式的工人都 Claude 会话。要引入不同工具，通过 MCP server 暴露给 Claude。

可以组合这些方式。

---

## 选择方式的三问框架

1. **谁协调工作？** — 你想让 Claude 在一个对话内委派和收集结果 → Subagents。你要分发独立任务然后检查回来 → Agent view。你要 Claude 规划、分配、监督一组工人 → Agent teams（实验性，默认关闭）。

2. **工人之间需要通信吗？** — Subagents 只向 spawn 它们的会话报告结果。Agent view 会话只向你报告。Agent teams 的队友共享任务列表并互相直接发消息。

3. **任务触碰同一文件吗？** — 用 worktrees 隔离。Subagents 和你自己跑的会话可以各用独立 worktree。Agent teams 不隔离队友到 worktree 中，所以要分区工作——每人负责不同文件集。

---

## 检查运行中的工作

- 后台会话：`claude agents` 打开 agent view
- 当前会话的子代理：`/agents` 打开面板（Running 标签=活着的子代理，Library 标签=创建/编辑自定义子代理）
- 当前会话后台运行的任何东西：`/tasks` 列出每项

---

## 关联

- [Subagent 官方完整规格](subagent-official-spec.md) — 同一次调研，子代理详情页
- [Subagent Hook 官方规格](subagent-hooks-official-spec.md) — 同一次调研，hook 事件
