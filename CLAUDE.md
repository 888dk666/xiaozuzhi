# 晓组织

多 Agent 协作团队。代号"晓组织"，DK666 说这四个字即启动。

## 架构

| 文件 | 内容 |
|------|------|
| `protocol.md` | 完整协议 9 章——角色体系、拆任务原则、工作流、团队配置、质检清单、冲突处理、红线 |
| `operations.md` | 操作手册——5 章。第一章：Step 0 完整+三条路径+Spawn 约定+拼合模板+超时参考+升级阶梯。第二章：6 角色 Prompt 模板（注释版，以 agent 定义文件为准）。第三章：PM 速查卡。第四/五章：脚本+升级模板 |
| `.claude/agents/` | 5 角色 agent 定义文件——spawn 时的角色 system prompt 源头（architect/qa-reviewer/compliance/finance/engineer） |
| `任务笔记/` | 任务笔记持久化目录（跨会话持久化，非 /tmp/） |
| `reference/` | 参考文档——Subagent Hook 官方规格等 |
| `research-spawn-patterns.md` | 外部 spawn 模式调研——spawn prompt 模板/卡死检测/文档同步/校验/10+ Agent 铁律 |

**基础设施：** git repo（2026-05-25 初始化，commit `51e01b6`）。工程师 agent 软链接到 `~/.claude/agents/engineer.md`（全局可用，跨 CWD 发现）。

**不内置：** Agent Teams / Fork 模式 / Agent-based Hook。详见 [Spawn 机制调研](research-spawn.md)。

## 角色

| 角色 | 名称 | 化身 |
|------|------|------|
| CEO | 火影大人 | — |
| PM | PM小克 | 萧何 |
| 架构师 | 架构小克 | 理查德·费曼 |
| QA 总监 | QA小克 | 查理·芒格 |
| 合规总监 | 合规小克 | 海瑞 |
| CFO | 财务小克 | 本杰明·格雷厄姆 |
| 工程师 | 牛马小卡 | 螺丝钉 |

## 触发

"并行"/"多线"/"同时干"/"派人去干"/"晓组织"

## 关联

- [protocol.md](protocol.md) — 完整协议 9 章
- [operations.md](operations.md) — 操作手册 5 章
- [.claude/agents/](.claude/agents/) — 角色 agent 定义文件（spawn system prompt 源头）
- `~/projects/晓组织/任务笔记/` — 任务笔记持久化目录
- [小小克使用指南](~/projects/小小克/subagent-guide.md) — Subagent 通用操作（frontmatter 字段、spawn 规范、限制）
- [信息质量权重体系](../../.claude/projects/-Users-dkhzymini/memory/reference/information-quality-framework.md)
- [多 Agent 生态调研](research.md) — 2026-05-23 初始调研 + 05-24 三轮补充
- [Spawn 机制调研](research-spawn.md) — 2026-05-24——四套并行机制+实测数据+晓组织差距分析
- [Subagent Hook 官方规格](reference/subagent-hooks-official-spec.md) — SubagentStart/SubagentStop 官方文档抓取，2026-05-25
