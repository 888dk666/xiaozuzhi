# 晓组织

多 Agent 协作团队。代号"晓组织"，DK666 说这四个字即启动。

## 架构

| 文件 | 内容 |
|------|------|
| `protocol.md` | 完整协议 9 章——角色体系、拆任务原则、工作流、团队配置、质检清单、冲突处理、红线 |
| `operations.md` | 操作手册——5 章。第一章：Step 0 完整+三条路径+Spawn 约定+拼合模板+超时参考+升级阶梯。第二章：6 角色 Prompt 模板（注释版，以 agent 定义文件为准）。第三章：PM 速查卡。第四/五章：脚本+升级模板 |
| `.claude/agents/` | 5 角色 agent 定义文件——spawn 时的角色 system prompt 源头（architect/qa-reviewer/compliance/finance/engineer） |
| `任务笔记/` | 任务笔记持久化目录（跨会话持久化，非 /tmp/），文件类型见下方子节 |
| `reference/` | 参考文档——Subagent Hook 官方规格、Subagent 官方完整规格等 |
| `spawn-cost.jsonl` | 成本日志（运行时自动创建）——每次 spawn 的 token 消耗 + stop_reason |
| `research-spawn-patterns.md` | 外部 spawn 模式调研——spawn prompt 模板/卡死检测/文档同步/校验/10+ Agent 铁律 |

**基础设施：** git repo（2026-05-25 初始化，commit `51e01b6`）。工程师 agent 软链接到 `~/.claude/agents/engineer.md`（全局可用，跨 CWD 发现）。

**不内置：** Agent Teams / Fork 模式 / Agent-based Hook / Subagent Memory。详见 [Spawn 机制调研](research-spawn.md)。Agent Teams 专项由 [春秋战国](~/projects/春秋战国/) 负责，等官方稳定版后启用。Memory 待 10 次任务后评估（启用会扩工具面为 Read/Write/Edit，当前无数据可积累）。

### 任务笔记文件类型

所有文件存放于 `~/projects/晓组织/任务笔记/`。命名格式中的日期统一为 `{YYYYMMDD}`。

**规范约定（ops.md 定义）：**

| 文件类型 | 命名格式与用途 |
|------|------|
| 任务笔记 | `任务笔记-{YYYYMMDD}-{序号}.md` — 走晓组织流程的任务全量记录（0.1~0.5 六段 + 原始产出 + 呈报） |
| 审查笔记 | `审查笔记-{YYYYMMDD}-{序号}.md` — 审阅/讨论/发现，未跑完整晓组织流程 |
| 搜索报告 | `搜索报告-{任务名}-{路名}-{YYYYMMDD}.md` — 研究路径纯牛马单路搜索产出（计划约定，待首次创建） |

**项目约定（实际使用，随项目演化）：**

| 文件类型 | 命名格式与用途 |
|------|------|
| 学习成果 | `学习成果-{主题}.md` — 研究路径最终学习文档 |
| 摸排汇总 | `摸排汇总-{主题}-{YYYYMMDD}.md` — 现状摸排/可行性调研的汇总结论 |
| 方案 | `方案-{主题}-{YYYYMMDD}.md` — 实施计划方案 |
| 修补方案 | `修补方案-{主题}-{YYYYMMDD}.md` — 瑕疵修补方案（修订稿） |
| 修补方案-终稿 | `修补方案-终稿-{主题}-{YYYYMMDD}.md` — 修补方案最终确定版 |
| 执行方案 | `执行方案-{主题}-终稿-{YYYYMMDD}.md` — 落地执行的操作步骤终稿 |
| 瑕疵记录 | `瑕疵记录-{任务名}-{YYYYMMDD}.md` — 任务缺陷的追踪记录 |
| 待跟进 | `待跟进-{摘要}-{YYYYMMDD}.md` — 待办事项/已知缺口（合并原"待办"+"待补"，存量文件保留原名） |

**临时备份：**

| 文件类型 | 命名格式与用途 |
|------|------|
| backup | `backup-{源文件名}-{YYYYMMDD}.md` — 被替换的旧版本文件备份，保留至归档时清理 |

## 角色

| 角色 | 名称 | 化身 |
|------|------|------|
| CEO | 火影大人 | — |
| PM | PM小克 | 萧何 |
| 架构师 | 架构小克 | 理查德·费曼 |
| QA 总监 | QA小克 | 查理·芒格 |
| 合规总监 | 合规小克 | 海瑞 |
| CFO | 财务小克 | 本杰明·格雷厄姆 |
| 工程师 | 纯牛马 | 螺丝钉 |

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
- [Subagent 官方完整规格](reference/subagent-official-spec.md) — code.claude.com/docs/en/sub-agents 全页提取，2026-05-25 CDP 浏览器逐页抓取
- [并行机制总览官方规格](reference/subagent-parallel-official-spec.md) — code.claude.com/docs/en/agents 全页提取，2026-05-25 Jina 抓取
