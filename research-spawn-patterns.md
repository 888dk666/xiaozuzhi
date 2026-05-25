---
name: spawn-patterns-research
description: 多 Agent Spawn 实战模式外部调研，2026-05-25 小小克搜索，补充 spawn 约定缺口
metadata:
  type: reference
  project: 晓组织
---

# 多 Agent Spawn 实战模式调研

2026-05-25，小小克搜索外部成熟案例，补充 spawn 约定缺口。与 [research-spawn.md](research-spawn.md)（官方机制深度调研）互补——本文件聚焦社区实战操作模式。

## 一、Spawn Prompt 模板字段

社区主流采用 Markdown + YAML frontmatter（`.claude/agents/{name}.md`）。生产环境必需的 spawn 字段：

- **Goal** — 一句话目标
- **Files you own / must not touch** — 文件边界
- **How to verify** — 验证命令 + 通过标准
- **maxTurns** — 迭代上限
- **allowedTools / disallowedTools** — 工具白/黑名单
- **permissionMode** — auto / dontAsk / ask

3-Agent 模板（Orchestrator / Specialist / Evaluator）被广泛引用，Specialist 输出需含不确定性声明和 TDD 标记。

## 二、卡死/超时检测

三层架构：

| 层级 | 机制 | 成本 |
|------|------|------|
| Tier 1 确定性检测 | 墙钟超时、SHA-256 哈希重复检测、A→B→A 周期模式匹配、心跳检查点 | 免费 |
| Tier 2 轻量 LLM（Haiku） | 仅 Tier 1 升级后用于判断"真卡死还是长任务" | 低 |
| Tier 3 全 LLM（Sonnet/Opus） | 生成纠正指令 | 高 |

生产配置：
- 120s 心跳超时
- 5 次相同输出告警
- 失败次数阶梯升级：1次重试 → 2次扩展上下文 → 3次换方案 → 4次人工

Python 工具 **failguard** / **loopguard** 提供零依赖检测。

## 三、Agent 定义 ↔ 文档同步

**agents-doc**（Node.js）：单一真相源目录 → `sync` 生成多工具格式，`check` 做 CI 漂移检测。

**FAF 格式**（IETF draft）：`project.faf` YAML 文件为异构 Agent 提供共享上下文。

**Conductor**（微软开源）：Jinja2 模板路由，零 token 开销的确定性编排。PrompTrek 提案双向文件系统 ↔ Claude Code Agent 同步。

## 四、Spawn 前校验

**自主等级分类**（Tier 0-4）决定证据门槛。

Schema 校验：
- model 枚举
- budget > 0
- prompt 非空 ≤ 100k 字符

必须明确：
- 文件所有权边界
- 验证命令 + 通过标准
- session 追踪标记（SESSION_ID / tracking.md / 时间戳）

输出验证、超时处理、退避逻辑须在 spawn 前确认。

## 五、10+ Agent 团队实操惯例

**CrewAI**：2025 起委派默认关闭，需显式 `allow_delegation=True`。`max_iter=5` 防互踢皮球。

**LangGraph Supervisor 模式**：用 `model.bind_tools([sub_agents])` 动态路由。

核心铁律：
1. 监控与执行分离
2. 所有操作幂等
3. 状态外置化（文件/DB 非内存）
4. 每个外部调用设显式超时
5. 权限按角色最小化 — Planner 只读、Implementer 可写、Reviewer 只读+分析工具

---

## 参考来源

- [Claude Code Subagents Best Practices (pubnub.com)](https://www.pubnub.com/blog/best-practices-for-claude-code-sub-agents/)
- [Why Your Multi-Agent System Fails Silently (dev.to)](https://dev.to/tuomo_pisama/why-your-multi-agent-system-fails-silently-and-how-to-detect-it-f0m)
- [CrewAI Hierarchical Process](https://docs.crewai.com/en/learn/hierarchical-process)
- [Conductor: Deterministic Orchestration (Microsoft)](https://opensource.microsoft.com/blog/2026/05/14/conductor-deterministic-orchestration-for-multi-agent-ai-workflows/)
- [AgentPool YAML Orchestration](https://github.com/phil65/agentpool/)
- [agents-doc Sync Tool (npm)](https://www.npmjs.com/package/agents-doc)
- [FAF Format (IETF Draft)](https://datatracker.ietf.org/doc/html/draft-wolfe-faf-format-00)
- [failguard: Stuck/Loop Detection (PyPI)](https://pypi.org/project/failguard/)
