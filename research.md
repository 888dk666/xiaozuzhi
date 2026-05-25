---
name: multi-agent-ecosystem-research
description: 多Agent协作生态全面调研——官方方案+业界框架+开源项目+实战经验+新兴工具+文章启发，2026-05-23
metadata:
  type: reference
  originSessionId: 630c2a48-1791-4ce8-95f9-050d4a44c361
---

# 多 Agent 协作生态全面调研

2026-05-23，7 个并行 agent + 1 篇公众号文章，覆盖多 Agent 协作全貌。

---

## 一、Claude Code 官方多 Agent 能力

来源：docs.anthropic.com、code.claude.com/docs、anthropic.com/engineering

### 三个层级

**1. Subagents（正式功能）**

单会话内，主 Agent 通过 Agent 工具派生子代理。每个子代理有独立的上下文窗口、系统提示词、工具白名单、模型和权限。内置 Explore（Haiku、只读）、Plan、General-purpose、statusline-setup、claude-code-guide 五个子代理。用户可自定义，用 Markdown + YAML frontmatter 定义，支持 Managed settings / CLI 参数 / 项目级 / 用户级 / 插件级五种作用域。子代理之间不通信，只向主 Agent 回报结果。支持 hooks、MCP 服务器、持久记忆、skills 预加载、worktree 隔离。

**2. Agent View（正式功能）**

通过 `claude agents` 打开仪表盘，管理和派发多个独立的后台会话。每个会话是完整的 Claude Code 实例，独立运行不相通信。可在 terminal 关闭后继续后台跑。

**3. Agent Teams（实验性功能，v2.1.32+）**

多个 Claude Code 实例组成团队直接通信。设 `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` 启用。

核心四件套：
- **Lead Agent**：主会话，创建团队和协调
- **Teammates**：独立 Claude Code 实例，各自有独立上下文窗口
- **Shared Task List**：pending/in-progress/completed 三态，支持任务依赖
- **Mailbox**：队友间直接消息，自动投递，无需轮询

存储于 `~/.claude/teams/{name}/config.json` 和 `~/.claude/tasks/{name}/`。

两种显示模式：in-process（单终端内 Shift+Down 切换队友）和 split panes（需 tmux 或 iTerm2）。队友每人独立上下文窗口，不继承 Lead 的对话历史，但自动加载 CLAUDE.md、MCP server 和 skills。

协调机制：共享任务列表 + 文件锁防竞态 + 任务依赖自动管理。队友完成后可自取（self-claim）下一个未分配、无阻塞的任务。队友间通过名称互发消息（SendMessage），空闲时自动通知 Lead。支持 Plan Approval 流程，可用 subagent 定义复用为 teammate 角色。

三个专属 hook 事件（不支持 matcher）：
- **TeammateIdle**：队友即将空闲时触发，exit code 2 发反馈让队友继续工作
- **TaskCreated**：创建任务时触发，exit code 2 阻止创建
- **TaskCompleted**：标记完成时触发，exit code 2 阻止完成

已知限制与 Bug（来自 GitHub Issues）：
- #60199 队友接受 shutdown 后进程不终止
- #55666 spawn 报告成功但没创建实际进程
- #47396 僵尸队友——在 config 中存在但已失效，无法强制终止
- #42005 队友状态行始终显示 0 tool uses / 0 tokens
- #58145 Planning mode 文档描述错误
- #52097 model 参数不支持版本锁定（只能 sonnet/opus/haiku）
- 仅一个 Lead，不能嵌套团队，不能转让领导权
- /resume 无法恢复 in-process 队友

**Subagents vs Teams 选择：**
- Subagents：只需结果、不需要队友间沟通、token 预算敏感、任务简单独立
- Teams：需要队友互相讨论辩论、多视角并行探索、竞争性假设验证、跨层协调

### Managed Agents（2026.4）

Anthropic 的托管长时 Agent 服务。核心设计是"大脑与双手解耦"——Agent harness（大脑）不在沙箱容器内，通过 `execute(name, input) -> string` 接口调用沙箱。Session 是独立于上下文窗口的持久事件日志，支持断点续跑和多脑多手扩展。P50 TTFT 降低 60%。

### 官方推荐模式（Building Effective Agents, 2024.12）

五种模式：Prompt Chaining（链式）、Routing（路由分流）、Parallelization（并行分段/投票）、Orchestrator-Workers（编排-工人）、Evaluator-Optimizer（评估-优化）。核心理念：**从简单开始，只有简单方案不够时才加复杂度。**

多 Agent 研究系统核心发现：Multi-agent 比单 agent 性能提升 90.2%；token 消耗解释 80% 性能方差；多 agent 比普通聊天多用约 15 倍 token。

---

## 二、业界六大框架对比

来源：Gurusup、OpenAgents、Medium/Jose Sosa、各框架 GitHub

| 框架 | 核心协调机制 | 角色定义方式 | 质量控制手段 | 最佳场景 | 关键局限 |
|------|-------------|-------------|-------------|---------|---------|
| **LangGraph** | 有向图+共享状态，节点=Agent，边=条件路由，内置checkpoint | State Schema 定义输入输出 | human-in-the-loop（暂停等审批）、time-travel调试、LangSmith可观测性 | 复杂分支流程、需审批的合规场景（金融/医疗） | 学习曲线陡，简单流程也需定义图结构 |
| **CrewAI** | 角色隐喻（role/goal/backstory），三种模式：顺序/层级/协商 | 自然语言定义 role+goal+backstory | 层级模式中 manager agent 负责审查输出 | 快速原型、业务流程自动化（20行代码即可跑） | 缺少内置checkpoint，细粒度控制弱，规模大了常迁到LangGraph |
| **AutoGen / AG2** | 多轮对话驱动，GroupChat+GroupChatManager选谁发言，v0.4改为事件驱动+异步 | system_message定义专长，按对话上下文分配发言权 | Agent间互相辩论+批判（如writer+reviewer+fact-checker三轮对话） | 代码生成+审查、需要多Agent辩论迭代的任务 | token成本高（4 Agent×5轮=20次LLM调用），微软已转向维护模式 |
| **OpenAI Agents SDK** | 显式Handoff：分流Agent→专业Agent→可回传或再转 | 每个Agent定义instructions+model+tools+可handoff列表 | 内置Guardrails（输入/输出校验）+Tracing全链路追踪 | OpenAI生态团队、客服分流类场景（8-10个Agent以内） | 锁定OpenAI模型，Agent多了handoff变复杂 |
| **MetaGPT** | SOP编码协作：产品经理→架构师→项目经理→工程师，按软件公司流程串行+评审 | 预设SOP角色（PM/架构师/工程师等），每个角色有固定输出物和评审门 | 结构化SOP自带质量门（需求→设计→代码，每步有评审） | 需求到代码全流程自动生成、软件项目脚手架 | 领域绑定软件工程，灵活性低 |
| **Camel** | 角色扮演社群：两个Agent互相扮演，对话驱动任务推进 | Role-Playing模式定义双方角色和任务，支持大规模Agent社群（声称百万级） | Critic Agent专门负责审查；stateful memory保留历史上下文 | 学术研究、合成数据生成、大规模Agent行为实验 | 偏研究导向，生产落地案例少，社区较小 |

### 哪个框架最接近三省六部制？

**第一梯队：LangGraph 和 MetaGPT**
- LangGraph：图结构天然分离"执行节点"和"审查节点"，conditional edge 充当协调者路由；human-in-the-loop checkpoint 就是"审查关"。这是对该模式支持最完整的框架。
- MetaGPT：SOP 里直接嵌入了"产出物→评审→下一阶段"的门禁，是唯一把"软件公司三省六部"直接写进代码的框架。缺点是领域绑死软件工程。

**第二梯队：CrewAI（层级模式）和 AutoGen**
- CrewAI 层级模式：manager agent = 协调者，worker agents = 执行者，manager 天然承担审查职责。但缺少正式的"审查节点"，审查是隐式而非显式的。
- AutoGen：多Agent 辩论天然自带"互审"机制，但 coordinator（GroupChatManager）是个黑箱 LLM。

**结论**：如果要做泛化的"三省六部制"框架（不局限于软件工程），LangGraph 是当前最合适的底座。MetaGPT 则提供了 SOP 角色分离的最佳参考模板。

---

## 三、Accio Work（阿里国际站）

来源：Accio Work 官网、百度百科、OFweek、PR Newswire

### 产品概况

2026 年 3 月发布，基于 Electron 构建，支持 macOS 和 Windows。前身是 2024 年 11 月上线的 B2B AI 搜索引擎 Accio。底层采用多智能体架构，模块分为 AI 搜索、商品页面重构、端到端电商平台三层。模型层整合 Qwen2.5 与自研算法，2025 年 2 月接入 DeepSeek 推理模型。数据层覆盖 25 万商家、3000 万跨境贸易企业信息。截至 2026 年 4 月，已支撑超 23 万在线店铺。海外企业用户突破 1000 万。

### 多智能体协作机制

"团队"（Team，Beta 阶段）是核心功能：用户创建包含 Team Leader 和成员 Agent 的团队。TL 负责任务委派、群聊协调、多 Agent 工作流编排。Agent 之间通过"handoff protocols"传递任务，在团队工作区内共享上下文。Skill 和 Agent 可通过 Team Hub 上传下载实现复用。每个 Agent 可自定义角色（role）、指令（instructions，即 system prompt）和能力集（capability set，控制可访问的工具范围）。支持按 Agent 独立切换模型（Gemini / GPT-4o / Claude / Qwen）。

### 安全与运行模式

本地优先（local-first）：编排在设备端完成，LLM 请求通过用户自己的代理路由。四级权限模型：自动批准 / 每次询问 / 记住选择 / 始终拒绝。沙箱隔离机制：每个工具独立环境运行，目录白名单 + 符号链接逃逸检测。

### 与 Claude Code 技术对比

| 维度 | Accio Work | Claude Code |
|------|-----------|-------------|
| 形态 | Electron GUI 桌面应用 | CLI 终端工具 |
| 用户定位 | 中小企业经营者，零代码 | 开发者 |
| 多 Agent | 显式 TL+成员团队结构，群聊协调 | Subagents + Agent Teams（实验性） |
| 模型 | 网关路由，免配 API Key | 直接配置 API Key |
| 工具生态 | 内置连接器市场 + MCP | hooks/skills |
| 浏览器 | CDP 内嵌 | 无原生浏览器 |
| 自动化 | Cron 式定时任务 | 无内置定时 |

### 局限

团队功能仍为 Beta；不支持 Linux；底层多 Agent 通信协议的工程细节（如消息格式、状态同步机制）未公开；无开源代码；LLM 调用仍需经阿里网关。

---

## 四、卡兹克文章核心内容（微信公众号，2026-05）

来源：https://mp.weixin.qq.com/s/iBCNkORwkff3PL1EZD3zqw

文章用 Accio Work 搭了两套多 Agent 系统：

### 4.1 三省六部制（流程驱动，适合执行）

角色映射：
- 中书省 → 出方案（多方案并行）
- 门下省 → 挑毛病、审议（质检）
- 尚书省 → 汇总决策、往下派活（协调）
- 六部（吏户礼兵刑工）→ 各管一摊并行执行

实操案例——让这个团队做一款心流码字器产品：
- 中书令一次出了三个不同方向的方案（情绪共鸣免费路线 / SaaS 付费墙 / 出海买断 Tauri + Product Hunt $14.9）
- 门下侍中审议：方案一条件放行+警示，方案二直接打回（核心矛盾："心流是反分析的，你量化它就消失了"），甩出三个跨方案共性问题（护城河在哪？创始人生存周期？用户真实痛点验证？）
- 尚书令综合审议做决断：方案一做主线，方案三留作第二阶段，方案二砍掉，把评估任务派给六部
- 六部并行开工：
  - 兵部：扫了 30+ 竞品，发现完整组合目前无竞品，但 V2EX 上有人在做类似方向，时间窗口约 6 周
  - 礼部：起名"墨沉"，正面回答"凭什么让用户主动晒截图"——用户晒的不是产品，是自己凌晨三点还在写的人设
  - 刑部：合规评估——必须注册个体工商户，个人收款码不能收经营款，字体只能用思源黑体和阿里巴巴普惠体
  - 户部：财务模型
  - 工部：技术架构
  - 吏部：人力规划
- 尚书令汇总输出最终 PRD——产品名称、战略路径、技术架构、财务模型、品牌方案、合规清单、人力规划，总共 12 份交付文件
- 整个过程：从需求到 PRD 签发，**40 分钟**

### 4.2 AI 圆桌（讨论驱动，适合想透问题）

6 个不同模型 + 1 个主持 agent，五轮流程：
1. 主持抛出议题，每个 agent 独立分析
2. 交叉质疑环节，互相挑对方的毛病
3. 回应修正，承认被打中的漏洞，反驳不成立的质疑
4. 共识收集，看哪些点全场认了，哪些点还有分歧
5. 投票选举意见领袖，由他写最终报告

跑了 10 场，Opus 4.6 赢了 6 场，累计 36 票，碾压级别。GPT-5.4 赢了 2 场，剩下两场打平。

### 4.3 核心洞察

- **制度本身就是最好的 prompt**：三省六部制能跑出效果，不是靠长篇提示词，而是靠角色定义本身——"你是门下侍中，你的职责就是挑毛病"。角色自带边界和行为预期。
- **多视角并行思考是 AI 对 OPC 最大的赋能**：一个人配上一套 AI 工具栈，能产出过去 10 人团队的工作量。
- **制衡的价值**：出主意的人永远会觉得自己的主意好，每个人都需要一个专门挑毛病的角色。
- **Accio Work 极限状态下上下文工程没 Claude Code Agent Team 好**：把 9 个 Opus 4.6 放进去时，多轮对话后质量不太稳定。

---

## 五、开源项目：当皇上（danghuangshang）

来源：GitHub wanikua/danghuangshang，2656 Star，249 Fork，MIT 协议

### 架构

基于 OpenClaw（Node.js 多 Agent 网关框架），将明朝内阁制/唐朝三省制/现代企业制三种制度映射为 Agent 协作流程。默认明朝制：用户=皇帝，18 个 Agent = 满朝文武。

角色分三层：
- **调度层**：司礼监接旨 + 内阁 Prompt 优化 + 都察院审查
- **执行层**：六部各司其职（兵部编码/户部财务/礼部营销/工部运维/吏部项目/刑部法务）
- **辅助层**：翰林院/国子监/太医院等

角色定义：每个 Agent 通过 `.md` 文件写入 SOUL 人设（身份、职责边界、工作流规则）。司礼监明确规定"禁止写代码、禁止查数据、禁止写文案"——自己只调度不执行。

通信机制：Discord 多 Bot 模型——每个 Agent 一个独立 Discord Bot 账号，用户 @谁谁响应。Agent 间通信用 `sessions_send`/`sessions_spawn`（OpenClaw 原生），内部有 MessageBus（EventEmitter pub/sub）、task-store（JSON 状态机跟踪多步任务依赖）、permission-guard（权限矩阵，禁止越级调用）。

技术栈：OpenClaw Gateway + React/Vite GUI + SQLite 记忆 + Docker 沙箱 + GitHub Webhook 触发代码审查。模型层支持任何 OpenAI 兼容 API。v3.7.0 起支持 Nous Research Hermes Agent 作平行 runtime。

### 创新点

1. **制度性审核而非简单委派**：门下省有"封驳"权——可以否决不合理方案，这在 AutoGPT/CrewAI 中没有对应物
2. **完全可观测**：Discord 频道公开所有流转，GUI 提供 Kanban、时间线、健康监控
3. **零代码安装**：一行脚本 5 分钟部署，默认自带 60+ Skill
4. **多制度可切换**：唐/明/现代企业三种模式一键切换

### 局限性

- **强依赖 OpenClaw**：不是独立框架，本质是 OpenClaw 的配置模板+人设注入
- **Discord 绑定**：多 Bot 架构导致需要多个 Discord 应用，国内用户接入飞书配置复杂
- **Token 消耗大**：一次全流程任务可能消耗 100 次模型调用
- **无共享记忆**：Agent 记忆是独立 SQLite 文件，缺乏向量检索或跨 Agent 知识共享
- **GUI 只读**：看板只能监控不能操作，实际指挥仍在 Discord 频道
- **任务状态机简陋**：JSON 文件而非数据库，无事务保证

### 对我们的启发

1. **角色边界硬约束**值得借鉴——司礼监"禁止写代码"这种负向指令比正向提示更有效
2. **审核前置**思路：任务派发前先经"内阁"优化 Prompt，减少执行层返工
3. **模型分层**：重活用强模型、轻活用快模型，兼顾质量与成本
4. **制度化流程优于自由对话**：固定"接旨-优化-派发-审查"流水线，比让 Agent 自己商量更可控

### 同类项目

- Edict（cft0808/edict，带完整 Kanban 看板的分支）
- Become CEO（同作者的英文企业版）
- TianGong（终端版）

---

## 六、新兴工具与被忽略的方案

来源：WebSearch + WebFetch，2026-05-23

### 轻量级替代方案

- **OpenClaw**（MIT 许可）：最被低估的自托管 agent 运行时。单个 Node.js 进程即可运行多个隔离 agent，配置靠 Markdown 文件（AGENTS.md、SOUL.md、HEARTBEAT.md）。亮点是 Heartbeat Daemon——每 30 分钟自动检查任务清单并执行，无需人工提示。
- **Claude Agent SDK**：极简路线，agent 循环就是 prompt + 工具调用，子 agent 本身也是工具。不设编排层，依赖模型自身推理能力。
- **Mastra**（19.1K stars）：TypeScript 生态的轻量选择，与 React/Next.js 深度集成。
- **Cline 2.0**：已从"人机结对"重新架构为"终端 AI agent 控制平面"——一个控制面管理多个 agent。

### Agent 间通信协议：四层协议栈

| 协议 | 所属 | 层级 | 状态 |
|------|------|------|------|
| MCP | Anthropic | Agent-工具层 | 9700 万下载，事实标准 |
| A2A | Google → Linux Foundation | Agent-to-Agent 发现+任务委派 | v1.0，Agent Card 安全机制 |
| ACP | IBM + Linux Foundation | Agent 间商业交易 | 发展中 |
| UCP | Google | Google 生态内商业层 | 发展中 |

四协议互补不互斥。Cisco 的 agntcy 在此基础上构建"Agent 互联网"框架。

### 中国开源生态

- **DeerFlow 2.0**（字节跳动）：GitHub Trending 第一，4.9 万 Star。核心设计：Lead Agent + 11 层中间件链 + 动态子 agent（最多 3 并行，Docker 沙箱隔离）+ 渐进式技能加载
- **MetaGPT**：6.25 万 Star，模拟软件公司 SOP
- **Agno**：3.66 万 Star，框架+运行时+控制平面的统一栈
- **Open-AutoGLM**（智谱）：2.08 万 Star，移动端自动化

### IDE 多 Agent 现状

Cursor、Windsurf、Copilot 目前仍是单 agent 辅助编码模式，未见真正多 agent 协作功能。CLI 工具（Claude Code、Codex CLI、Aider）在多 agent 场景更成熟——可通过 pipe 组合、headless 运行、CI 集成。

### 值得关注的方向

OpenClaw 的自主后台 agent 模式、DeerFlow 的沙箱隔离+中间件链架构、A2A 跨框架互通、Mastra/Goose 的多模型同时运行，代表了从"单个对话 agent"到"自主执行基础设施"的范式转移。

---

## 七、实战经验与最佳实践

来源：Anthropic 官方博客、LangChain 架构指南、orq.ai MAST 失效分类学、Datagrid 成本优化策略

### 被验证有效的模式

**第一原则：先用单 agent + 好工具。** 所有来源最一致的结论。Anthropic 原话："从最简单的方案开始，只在实际需要时增加复杂度。"

**Orchestrator-Workers（编排-工人）**：中心 LLM 动态分解任务、分发子 agent、汇总结果。Anthropic 内部研究系统用此模式：Claude Opus 4 做主控 + Sonnet 4 子 agent，在内部研究评测上比单 agent Opus 4 高出 90.2%。关键设计：子 agent 各自拥有独立上下文窗口，并行搜索不同方向，压缩后返回结果给主控。

**Evaluator-Optimizer（评估-优化循环）**：一个生成、一个评估反馈、循环迭代。适用于有明确评估标准且迭代能产生可度量改进的场景。

**Subagent 模式的上下文隔离是杀手锏**：LangChain 实测，多领域查询场景下 subagent 模式比 skills 模式少消耗 67% token——每个子 agent 只用相关上下文，避免了上下文累积膨胀。

**工具设计比 prompt 设计更重要**：Anthropic 在 SWE-bench agent 上花在优化工具上的时间超过优化整体 prompt。仅仅改进工具描述就让后续 agent 任务完成时间下降 40%。

### 最常见的失败模式

**协调成本指数增长。** orq.ai 的 MAST 框架归纳为三层：
- 规范缺陷：角色定义模糊、任务分解太粗或太细
- Agent 间失配：A 输出 YAML、B 期望 JSON；重复劳动；无限循环
- 验证终止缺口：没有裁判 agent 把关、无终止条件、agent 目标达成后继续空转

**Anthropic 第一手踩坑记录**：早期 agent 会为简单查询生成 50 个子 agent、全网搜寻不存在的来源、子 agent 相互用冗余更新干扰对方。解决方案——在 prompt 里嵌入"按查询复杂度缩放投入"规则：简单事实查询 1 agent + 3-10 次工具调用，直接对比 2-4 agent，复杂研究 10+ agent。

**基准测试的冷水**：多 agent 系统经常跑不过强单 agent 基线（如 o1）。MetaGPT、AG2 等框架在真实部署中频繁出现任务交接失败、agent 误解共享目标、重复生成代码等问题。

**误差复合效应**：Anthropic 指出 agent 系统是有状态的，一个小错误会级联放大。传统软件的"小 bug"对 agent 可能是致命伤——一个工具调用失败会导致 agent 走上完全不同的轨迹。

### 成本真相

- agent 交互比普通 chat 多消耗 **~4x** token
- 多 agent 系统比 chat 多消耗 **~15x** token
- token 用量单独解释 **80%** 的性能方差
- 但升级模型（如 Sonnet 3.7 → Sonnet 4）带来的性能提升，大于把 token 预算翻倍

八条成本管控策略：上下文压缩（传递摘要而非全文）、动态模型路由（简单任务走便宜模型）、并行替代串行、工具调用缓存+限流、按 agent/任务/业务线做粒度成本归因。

### 构建可靠多 Agent 系统的关键结论

1. **不到万不得已不加 agent。** 先用单 agent + 好工具 + 好 eval。只有上下文窗口不够用、团队边界需要隔离、或确实需要大规模并行时才上多 agent。
2. **Token 预算即架构。** 多 agent 的本质是通过并行化 token 花费来突破单 agent 上限。
3. **可观测性是刚需。** Agent 非确定性、误差复合——没有全链路追踪，出了问题完全没法排查。
4. **用结果而非过程做评估。** 不要评估 agent 是否走了"正确步骤"，评估最终状态是否正确。
5. **原型到生产的鸿沟比想象中大得多。** 需要状态化执行、彩虹部署、断点恢复、检查点系统等全套工程基础设施。
6. **护栏分层布防。** agent 间的熔断器、重试上限、超时阈值、人工升级路径——缺一不可。
7. **编码类任务目前不适合重度多 agent。** 原因：编码任务可并行化的部分比研究任务少，且 LLM agent 目前不擅长实时协调和委派其他 agent。

---

## 八、总结：我们的定位

### 不需要第三方框架

Claude Code 原生的 Subagents + Agent Teams 已经覆盖核心需求。我们缺的不是新工具，是把已有的能力用好。

### 能力对照表

| 已有 | 对标三省六部 | 状态 |
|------|------------|------|
| Subagents | 六部执行 | ✅ 在用 |
| 质检 Agent | 门下省 | ✅ 已写入 [晓组织协议](protocol.md) |
| 多方案并行+质检 | 中书省+门下省 | ✅ 已写入 [晓组织协议](protocol.md) |
| Agent Teams | 完整三省六部制 | 未开（实验性，可试） |
| role-over-rules | 角色定义优先 | ✅ 已写入 feedback/rules |

### 核心原则（来自调研 + 实践）

1. 制度本身就是最好的 prompt —— 角色定义 > 规则堆砌
2. 不到万不得已不加 agent —— 简单任务单 agent 够了
3. 多方案并行出 + 质检挑 —— 方向不确定时出 2-3 个方案互相比
4. 角色边界硬约束 —— 负向指令（"禁止做X"）比正向更有效
5. 可观测性是刚需 —— 没有追踪就没法排查
6. Token 预算即架构 —— 多 agent 本质是用 token 换并行能力

---

## 九、三省六部制制度精髓（两轮深度调研）

2026-05-23，两轮 agent 调研，提取可迁移到 Agent 系统的制度设计原则。

### 9.1 第一轮：三省核心机制

来源：学习时报、澎湃、兰州大学、西安交大学术论文

**五项核心原则：**

1. **强制串行流程，不是简单分权** — 一道政令必经三省流转，缺一环不具法律效力。精髓：不是让权力互相制衡，是让流程本身成为不可绕过的约束。

2. **封驳——低位高权的否决机制** — 门下省给事中仅正五品上，却能封存退回或直接涂改中书诏书。魏征以此多次驳回唐太宗敕令。武则天宰相刘祎之死前喊"不经凤阁鸾台，何名为敕？"。精髓：否决权行使者品级低于决策者，杜绝"自己审自己"。

3. **五花判事——竞争性方案 + 署名追责** — 六位中书舍人就同一议题各自独立起草方案并署名，中书令择优选用，分歧时另附"商量状"保留记录。《资治通鉴》评："由是鲜有败事。"精髓：方案来自竞争而非共识，每份可追溯到具体人。

4. **制度基因可变形继承** — 唐后三省名存实亡，明清废三省存六部，但决策-审核-执行的分层制衡逻辑被内阁、司礼监、军机处以不同形态继承。张国刚："以规则代人事、以分权防专权"的基因持续不灭。

5. **假设每个环节都会出错** — 三省六部不靠选出"好人"来保证质量，而是假设每个人都会出错，用流程强制纠错。

### 9.2 第二轮：六部制度、谏官、冲突仲裁、明清演变

来源：知乎、简书、网易号、豆瓣读书、百度百科

**六部制度设计：**
- 专业化分工遵循"职能聚类、互不重叠"。每部辖四司，全国共二十四司
- 尚书（正三品）+ 侍郎（1-2人）+ 郎中（各司主管），三层结构
- 六部之间不设横向制衡（那是三省的事），跨部靠尚书省统辖或政事堂协调
- 御史台六察官对口监察六部——每个御史对口一个部，外部审计闭环
- 迁移：执行层 Agent 按"业务域完整性"切分，每个执行 Agent 有明确的尚书/侍郎/郎中三层。独立质检 Agent 对口审查每个执行 Agent

**谏官制度——双重监察体系：**
- 门下省封驳：决策链条内的事前审核，管"政策对不对"
- 御史台/谏官：决策链条外的事后监察，管"人对不对+事落没落实"
- 封驳讲规则合规性，监察讲行为合规性
- 两套独立不可合并——一个拦坏政策，一个查坏执行
- 迁移：Agent 系统需要"流程内自动校验"（对应封驳）+ "独立审计 Agent"（对应御史台，不参与执行流，只做异步巡视）

**冲突仲裁——三级升级：**
- 一级：政事堂合议（同级协商）
- 二级：御前裁定（上级仲裁）
- 三级：权力集中（衰变案例——唐玄宗撤议事环节导致杨国忠专权）
- 迁移：Agent 需三级仲裁——Agent 委员会协商 → 调度 Agent 裁决 → 但设硬约束防止仲裁退化为集权

**明清演变——废相后的制度重构：**
- 明朝：内阁票拟 + 司礼监批红，两个互不信任的集团各拿半个权限
- 核心洞察：制衡不是靠明文规定，是靠"让两组互不信任的人各拿半个权限"
- 清朝军机处消灭了所有制衡，效率极高但完全依附皇权
- 六部千年保留——执行层的专业化边界是最抗颠覆的结构
- 迁移：多 Agent 系统的决策权不应由一个 Agent 独占。"半权限分派"比明文规则更强——让两个功能互补但信任模型不同的 Agent 各持半个决策权

### 9.3 可迁移的五条制度设计原则

1. **强制串行流程** — Plan → Review → Execute，每环节不可绕过
2. **独立审查 + 一票打回权** — Review Agent 不向 Plan Agent 汇报，有封驳权
3. **多方案竞争而非单方案优化** — 同一任务多 Agent 独立出方案，择优
4. **双重质检** — 流程内校验（封驳）+ 流程外审计（御史台）
5. **半权限分派** — 决策权分给两个互不信任的角色，各拿半个

> **注意：以上五条为迁移假说，按 [information-quality-framework](../../.claude/projects/-Users-dkhzymini/memory/reference/information-quality-framework.md) 的假说验证路径处理，未经验证前不作为正式规则使用。**

---

## 十、现代公司制度（四维深度调研）

2026-05-23，4 个 agent 并行调研现代企业制度中可迁移到多 Agent 系统的设计原则。

### 10.1 公司治理与制衡机制

来源：Wikipedia（Corporate Governance、SoD、Independent Director、RACI）、Spotify Engineering Culture、COSO框架

**权责分离（Separation of Duties）：**
公司内控体系的基石——授权(authorization)、执行(custody)、记录(record keeping)、核对(reconciliation)四职能必须由不同主体承担。RACI 矩阵进一步细化：执行者(R)与审批者(A)严格分离，每项任务只有一个 A。
- 迁移：Agent 系统中生成输出的 Agent 不得自审自批，审计 Agent 不得参与执行链

**独立监督：**
独立董事不入管理层、不受执行链管辖。NYSE/Nasdaq 要求上市公司董事会多数为独立董事(S&P 500 已达 72%)。审计委员会直接向董事会报告，不经过 CEO。这与门下省"封驳"内核完全一致——在权力运行链条中嵌入一道独立关口。
- 迁移：Agent 系统应设独立于执行链的审计 Agent，只做合规校验

**单一问责（RACI 核心）：**
每个可交付物有且仅有一个 Accountable，防止"都负责=都不负责"的责任扩散。
- 迁移：多 Agent 协作中每任务一个主责 Agent

**小单元自治：**
Amazon two-pizza team（6-8 人）将沟通代价控制在可承受范围。Spotify Squad 端到端拥有产品全程——"程序员必须承受自己设计决策的后果"。Chapter/Guild 通过横向社区而非纵向指令做能力对齐。
- 迁移：执行 Agent 自治完成闭环，跨队一致性靠协议而非集中调度

**矩阵双线：**
一个人同时向功能主管(管能力成长)和项目主管(管交付进度)汇报，两条线天然形成制衡——功能线防止为速度牺牲质量，项目线防止为完美拖延交付。
- 迁移：Agent 系统需要"能力维度+任务维度"并行管理

### 10.2 冲突升级与质量控制

来源：Wikipedia（Disagree and Commit、Andon、ISO 9000、Six Sigma/DMAIC、Code Review、PDCA、Internal Audit）、Intel/Andrew Grove

**冲突升级机制：**

Amazon "disagree and commit"：决策阶段充分辩论，一旦决策落定，所有人无条件执行。Intel Grove "建设性对抗"：知识权力高于职位权力，对事不对人。冲突升级模型核心洞见：每一级有明确的触发条件和应对手段，不越级也不拖延。

丰田"安灯"两段式拉绳：第一拉是求助信号（不停线），第二拉才是停线。避免"一有问题就停摆"的极端。
- 迁移：Agent 检测到异常先发"求助信号"，一定时间内未解决才中断流水线。三级升级——L1 Agent间协商、L2 仲裁Agent介入、L3 人类接管

**质量控制体系：**

ISO 9001 七大原则中，"过程方法"和"循证决策"最具迁移价值——把活动当过程管、用数据而非假设做决策。

PDCA 循环（计划-执行-检查-改进）是持续改进的元模式，每次迭代拉近目标一步，永不停在"差不多"。

Six Sigma DMAIC（定义-测量-分析-改进-控制）："控制"阶段最关键——改完不代表完，要有持续监控确保不回退。

Google Code Review 数据：75% 的审查意见关乎可维护性而非功能 bug，最优审查速度是每小时 200-400 行。审查的真正价值在知识传递和标准对齐，而非抓错。
- 迁移：Agent 每次输出经过 PDCA 包装；关键输出必须经过 reviewer Agent 审查，审查重点不只是正确性，还有可维护性（输出结构是否清晰、日志是否完整）

**独立审计：**

内部审计向董事会（而非管理层）汇报。外部审计（第三方认证）提供第二层验证。SOX 法案的 SoD 用四个不可合并的职能定义安全边界。
- 迁移：关键操作走四眼原则——一个 Agent 提出、另一个审批、第三个执行、第四个验证。同一条链上不可复用同一 Agent

### 10.3 组织架构与层级设计

来源：Wikipedia（span of control）、McKinsey（5 managerial archetypes, 2017）、Remote.com（4 org structures）、李平/华夏基石（字节三台架构, 2020）、WhatMatters/John Doerr、IIA Three Lines Model (2020)

**层级与管理幅度：**

麦肯锡基于数千岗位将管理者分为 5 种原型：玩家/教练（3-5人，战略复杂岗）、教练（6-7人）、监督者（8-10人）、协调者（11-15人）、标准化协调者（15+人）。核心变量：任务同质性、下属成熟度、流程标准化程度。

1980 年代 IT 技术将平均幅度从 1:4 推向 1:10——信息流转成本骤降。扁平化虽快但有混乱风险。
- 迁移：Agent supervisor 调度上限取决于子任务同质性——同质任务可批量分发（15+），异构复杂任务需窄幅管理（3-5）。Agent间通信带宽越高，越可减少中间协调层

**部门划分——四种基本模式：**

- 职能型：专业分工，易生筒仓
- 事业部型：独立核算，资源重复
- 矩阵型：双线汇报，灵活但复杂
- 扁平型：敏捷但需强自驱

**字节跳动"三台"模型：**
- 前台：产品，面向客户、精兵作战
- 中台：技术基础设施，可复用能力平台（如"黑土地"）
- 后台：文化治理，战略/风控/长期投入

IIA "三线模型"：一线（业务）管执行，二线（风控/合规）管监督，三线（内审）提供独立鉴证。财务/法务/HR 必须独立于业务线——被监督者不能管监督者。
- 迁移：Agent 系统分三层——执行 Agent（前台，直接面向任务）、能力 Agent（中台，可复用工具/知识/模型）、治理 Agent（后台，审计/合规/资源管控）。监督 Agent 不可被被监督 Agent 管辖

**跨部门协作：**

OKR 最大陷阱是照搬组织架构写目标——各部门各自定义，互相看不到关联。正解是 Intel "Operation Crush"模式：公司级北极星目标先行，再通过级联与对齐形成跨部门目标网络，多人共担一个 KR。

协作机制：内部 SLA（服务等级协议）、联合项目组、三层例会（日站会/周复盘/月战略对齐）。
- 迁移：Agent 系统目标不应按 Agent 类型拆解——先定全局目标，再按"谁需要一起协作"分配。内部 SLA 即 Agent 间接口契约（超时重试/降级）。三层例会即心跳检查+状态同步+优先级重排

### 10.4 激励、绩效与问责

来源：Wikipedia（Goodhart定律、代理人问题）、HBR（信任机制, 2025）、Gartner（3500+样本, 2024）、Handgraaf et al. 同行评审论文（公开奖励实验, 2013）、James Reason & Sidney Dekker（Just Culture）、Roger Martin（责任病毒）、Google re:Work、字节跳动公开资料

**绩效管理体系：**

OKR vs KPI 核心区别：OKR 设定"我们要去哪"的雄心目标+可衡量的关键结果，驱动变革；KPI 衡量"我们现在是否健康"，监控运营。Google 采用 0-1 分制评分，60%-70% 即视为成功——满分=目标太保守。

**Goodhart 定律**："当一个度量变成目标，它就不再是好的度量。"一旦 Agent 知道评估标准，就会优化该标准而非真实目标。医院缩短住院天数指标导致仓促出院后返院率飙升。

360 度评估：多来源反馈（上级、平级、下级、自评）弥补单一评估盲区。字节跳动将 360 评估嵌入 OKR 周期，任何人都可公开评价他人 OKR。
- 迁移：Agent 系统区分"探索性任务"（OKR式，60-70%=成功）和"运维性任务"（KPI式，100%达标）。避免单一指标考核，设置多维度交叉验证。Goodhart 防御：指标公式对 Agent 不可见，或用不可直接优化的复合指标

**激励设计：**

代理人问题（Principal-Agent Problem）：执行者掌握更多信息，可能为自己而非委托人做事。解法：利益对齐（利润分成、递延补偿）+ 监控机制。

短期激励（奖金）vs 长期激励（股权/期权）：纯短期激励诱发短视行为。

Handgraaf (2013, 300+引用) 实验：公开奖励效果持续优于私下奖励，社会性奖励（认可）优于金钱奖励。表扬批评比建议 5:1。
- 迁移：Agent 奖励函数应同时包含即时任务完成分（短期）和项目整体成功权重（长期）。将 Agent 的行为日志对同级 Agent 部分可见（类似公开表扬），形成社会性反馈

**问责与信用体系：**

Roger Martin "责任病毒"："英雄型"领导包揽过多责任，"被动型"下属逃避责任，双方互相强化形成恶性循环。

**Just Culture（公正文化）**：航空/医疗领域的无惩罚报告制度。核心是区分三类行为：(1) 无意失误——安慰+学习，不追责；(2) 风险行为——教练式纠正；(3) 鲁莽违规——追责。关键原则："局部合理性"——人在当下情境中做出的决定对自己而言是合理的。

Gartner 2024 调研（3500+员工）：仅 48% 员工信任高层。信任破坏行为 Top3：隐瞒信息(-20%)、甩锅(-30%)、推翻决策(-20%)。信任建设：解释决策逻辑（信任度提升 4.3 倍）+ 重视员工反馈。
- 迁移：Agent 出错误时区分"系统设计问题"(修复框架) vs "策略选择失误"(调整prompt) vs "违规绕过"(硬限制)。建立 Agent 信任分：新 Agent 从小任务开始积累信用，高信用 Agent 获更大自主权。引入"正向追责"：每个任务完成时记录决策链，"为什么这样做"比"谁做的"更重要

### 10.5 四维交汇——三省六部 × 现代公司 = Agent 系统设计原则

| 制度原则 | 三省六部 | 现代公司 | 对 Agent 系统 |
|---------|---------|---------|-------------|
| 权责分离 | 中书出令≠门下审核 | SoD 四职能不同人 | 生成≠审查≠批准 |
| 独立监督 | 御史台对六部六察官 | 内审向董事会汇报 | 独立审计 Agent |
| 竞争方案 | 五花判事（六舍人各自起草） | Amazon 多方案辩论 | 多 Agent 并行出方案 |
| 低位否决 | 给事中五品能驳回宰相 | 合规部门可否决业务 | 质检 Agent 一票打回权 |
| 问责锚定 | 署名追责 | RACI 单一 A | 每任务一个主责 Agent |
| 冲突升级 | 政事堂→御前 | disagree and commit→上级裁定 | Agent协商→仲裁→人接管 |
| 信任累积 | 科举+考课 | 信用分+晋升阶梯 | Agent 信用分阶梯 |
| 激励对齐 | 考课+监察 | OKR+长期股权 | 短期准确+长期成功双权重 |
| 三台分离 | 三省+六部 | 前台/中台/后台 | 执行/能力/治理 Agent |
| 安灯求助 | — | 丰田两段式拉绳 | 异常先求助，超时再中断 |
| Goodhart防御 | 封驳独立审核 | 指标不透明 | 质检标准对执行Agent不可见 |

> **注意：以上所有"迁移"条目均为假说，按 [information-quality-framework](../../.claude/projects/-Users-dkhzymini/memory/reference/information-quality-framework.md) 的假说验证路径处理，未经验证前不作为正式规则使用。**

---

## 十一、第一性原理 → 已移至 [思维框架·第一性原理](../../.claude/projects/-Users-dkhzymini/memory/framework/methods.md#first-principles-thinking)

---

## 十二、2026-05-24 补充调研：操作方案深度

上次覆盖：架构总览 + 框架对比 + 案例。本次补三个方向的实操细节。

### 12.1 Claude Code Agent Teams — 操作细节

来源：docs.anthropic.com、morphllm.com、GitHub FlorianBruniaux/claude-code-ultimate-guide、SitePoint、dev.to

#### 架构四件套

| 组件 | 说明 |
|------|------|
| Team Lead | 主会话，创建团队、派活、合成结果 |
| Teammates | 独立 Claude Code 实例，各有 1M token 上下文窗口，加载 CLAUDE.md/MCP/skills 但不继承 Lead 对话历史 |
| Task List | 磁盘 JSON 文件，`~/.claude/tasks/{team-name}/`，三态 pending/in-progress/completed，支持依赖自动解阻塞，文件锁防竞态 |
| Mailbox | 内存消息系统，Agent 间 1:1 消息 + 全员广播，自动投递无需轮询 |

#### 两种显示模式

- **In-process（默认）**：单终端内 Shift+Down 切换队友
- **Split panes**：tmux 或 iTerm2 分屏，每个队友独立窗格。不支持 VS Code/Windows Terminal/Ghostty

#### 任务协调机制

- 文件锁防竞态——同一任务不会被两个队友同时认领
- 依赖自动管理——阻塞任务在依赖完成后自动解阻塞
- Plan Approval——队友先出计划给 Lead 批，批之前只读模式
- 三个专属 hook：TeammateIdle（exit code 2 发反馈让队友继续）、TaskCreated（exit code 2 阻止创建）、TaskCompleted（exit code 2 阻止完成）
- 权限预批——队友的权限请求冒泡到 Lead，常用操作预先批准减少摩擦

#### Worktree 隔离

`--worktree` flag 给每个队友独立 git worktree——完整仓库副本在独立目录，独立分支和 git index。消除文件冲突，最后 git merge 合并。

#### 最佳实践（来自官方 + 社区）

1. **给足够上下文**——队友不继承 Lead 对话历史，任务描述必须自包含
2. **控制团队规模**——3-5 个队友起步，每人 5-6 个任务。3 个聚焦的 agent 胜过 5 个分散的
3. **任务粒度适当**——自包含单元，产出明确
4. **避免文件冲突**——分配每个队友独立文件所有权，或用 worktree 隔离
5. **预批权限**——减少摩擦

#### 适用/不适用场景

| 适用 | 不适用 |
|------|--------|
| 并行代码审查（安全/性能/测试） | 严格串行依赖的任务 |
| 新模块/功能开发（前后端+测试并行） | 同文件编辑（合并冲突不可避免） |
| 竞争性假设调试 | 简单任务（单会话更快更省） |
| 跨层协调（API+UI+测试） | 预算紧张（成本按队友线性增长） |
| 多角度研究和综合 | 协调成本超过收益的场景 |

#### 已知限制（GitHub Issues 汇总）

- `/resume` 和 `/rewind` 不恢复队友——resume 后重新 spawn
- 队友有时忘记标记任务完成
- 关闭慢——队友完成当前请求后才停
- 每会话只能一个团队
- 不能嵌套团队——队友不能创建子团队
- Lead 不可转让
- 不能按角色选模型（Opus 给 Lead/Sonnet 给 Worker/Haiku 给测试）——此限制已被官方文档否定，队友可通过 spawn prompt 指定不同模型
- Split panes 不支持 VS Code 终端

#### 实战案例：16 Agent 写 C 编译器

Anthropic 工程团队用 16 个 Claude Agent 花 2 周从零构建 Rust C 编译器（10 万+行）：

- API 花费 $20K
- 输入 20 亿 token，约 2000 个会话
- GCC torture 测试 99% 通过率
- 能编译 Linux 内核、QEMU、FFmpeg、SQLite、PostgreSQL、Redis
- 在 x86/ARM/RISC-V 上启动 Linux 6.9

---

### 12.2 OpenAI Agents SDK — 操作模式

来源：openai/openai-agents-python DeepWiki、dev.to、c-sharpcorner.com

#### 两种核心编排原语

| 维度 | Handoffs（对等委托） | Agents-as-Tools（编排者模式） |
|------|---------------------|---------------------------|
| 控制流 | 转给专业 agent | 编排者持有 |
| 最终回复归属 | 链上最后一个专业 agent | 编排者合成 |
| 上下文可见性 | 完整对话历史（可配置过滤） | 子 agent 只看到传入参数 |
| 最佳场景 | 分流→专业路由、线性管道、客服 | 结构化合成、并行执行、稳定输出 schema |
| 底层机制 | `Agent.handoffs` / `handoff()` | `Agent.as_tool()` |

#### Handoff 模式——分流架构

```python
triage_agent = Agent(
    name="Triage Agent",
    instructions="Route requests to the appropriate specialist.",
    handoffs=[billing_agent, refund_agent, faq_agent],
)
```

底层 handoff 对 LLM 呈现为工具调用（`transfer_to_billing_agent`），Runner 自动处理转换。

**关键配置：**
- `input_filter` — 控制传给下个 agent 的历史内容（剥离工具调用、截断历史、删敏感数据）
- `nest_handoff_history` — 把先前对话压缩为单条摘要消息
- `on_handoff` — handoff 触发时的回调（日志、状态更新）
- `input_type` — Pydantic 模型，LLM 必须提供的结构化元数据（如原因、优先级）
- `is_enabled` — 基于上下文/权限动态开关

**Handoff Contract 模式（防无限循环）：**

2026 年记录的反模式：两个 agent 互相 handoff 直到预算耗尽——真实案例 47 次 handoff，$9.12 账单。

解药——所有权边界 + 显式"完成"信号：
- 只有 owner 写状态，其他人只读
- Handoff 单向（research → writer → done，无反向边）
- MAX_HANDOFFS=4——两个 agent 四次 handoff 太多了，超过说明分解有问题
- 如果需要反向边，加独立的 critic agent 而非让 writer handoff 回 researcher

#### Agents-as-Tools——并行执行

```python
async def run_all_specialists(request: str) -> str:
    specialists = [lynx, wildfire, bedrock, leverage, sentinel, prism, razor]
    async def run_one(agent):
        result = await Runner.run(starting_agent=agent, input=request)
        return result.final_output
    results = await asyncio.gather(*[run_one(a) for a in specialists])
    return aggregate_results(results)
```

墙钟时间从 ~70s 降到 ~15s。

#### Guardrails——作用域感知放置

| 类型 | 运行位置 |
|------|---------|
| Input guardrails | 工作流第一个 agent |
| Output guardrails | 产出最终输出的 agent |
| Tool guardrails | 仅作用于 function_tool（不适用 Handoff 和 Hosted tools） |

核心原则：验证放在离风险动作最近的地方。

#### LLM-as-a-Judge——迭代精炼

Generator → Evaluator 循环：评估通过→退出，不通过→反馈→重新生成。适用于有明确质量标准的产出。

#### 企业级能力（2026）

- 内置沙箱——agent 限定在隔离工作区、特定文件、特定工具
- 通过 provider-agnostic 架构支持多种模型（LiteLLM 为可选 beta 适配器之一）
- 追踪/护栏/可观测性内建在基础设施层
- Gartner 预测：到 2026 年底 40% 企业应用含任务专用 AI agent（2025 年 <5%）

---

### 12.3 Google ADK + A2A 协议

来源：Google Cloud Blog、Google Codelabs、ThoughtWorks Technology Radar、developers.googleblog.com

#### ADK 多语言生态（2026 状态）

| 语言 | 状态 |
|------|------|
| Python | 最早发布，最成熟 |
| Java | 2026-03-30 发布 1.0.0 |
| Go | 可用 |
| TypeScript | 可用 |
| Kotlin | 可用 |

ThoughtWorks Technology Radar（2026-04）：部分组件 pre-GA，但在 Google Cloud 团队中已在生产成功运行。

#### 三种内置 Template Workflow Agents（ADK 2.0 标记 superseded，推荐 graph-based/dynamic workflows）

| 组合器 | 用途 |
|--------|------|
| SequentialAgent | 管道式串行：A→B→C |
| ParallelAgent | 并行执行互不依赖的 agent |
| LoopAgent | 质量循环：agent 反复执行直到条件满足，`max_iterations` 设上限 |

```python
research_loop = LoopAgent(
    name="research_loop",
    sub_agents=[researcher, judge],
    max_iterations=3,  # 与晓组织 QA 打回 3 轮上限同构
)
root_agent = SequentialAgent(
    name="pipeline",
    sub_agents=[research_loop, content_builder],
)
```

#### A2A 协议——Agent 间"通用语言"

开放标准，不同框架/厂商/域的 agent 互操作：

- **Agent Card**（`/.well-known/agent-card.json`）：每个 agent 发布的 JSON 文档，描述身份、能力、技能、端点、认证要求——类似微服务的服务发现
- **Client ↔ Remote Agent 模型**：编排者通过 HTTP JSON-RPC 发现并调用远程 agent
- **安全优先**：HTTPS/TLS、JWT、OIDC、API Key
- **异步通信**：SSE 流式传输
- **类比**："微服务对 agent 的意义"——服务发现、RPC 调用、负载均衡

#### 三种编排架构

**1. Orchestrator 模式（Google Cloud Blog 推荐）：**

```
Frontend → Orchestrator → [Researcher Agent] → [Judge Agent] → [Content Builder]
                ↑               ↓ A2A              ↓ A2A
           (RemoteA2aAgent)  (Cloud Run)        (Cloud Run)
```

前端只调一个端点（orchestrator），agent 独立扩缩容，多语言混合。

**2. 事件驱动（Eventarc）：**

Agent 通过消息总线异步通信，完全解耦——新订阅者无需修改发布者。

**3. 远程 Agent 组合（A2A）：**

```python
researcher = RemoteA2aAgent(
    name="researcher",
    agent_card="http://researcher-service:8000/.well-known/agent.json",
)
```

#### 部署：Vertex AI Agent Engine

全托管服务——会话管理（短期记忆）+ Memory Bank（长期记忆）+ 安全沙箱代码执行。兼容 ADK/LangChain/LangGraph/CrewAI/AutoGen。部署后自动暴露 A2A Agent Card。

#### 2026 Agent 全栈

```
Orchestrator（SequentialAgent/LoopAgent/Parallel）
    ↓ A2A
Agent A (Python) + Agent B (Java) + Agent C (Go)
    ↓ MCP
Tools / APIs / Data
    ↓
Vertex AI Agent Engine / Cloud Run（托管部署+状态管理）
```

---

### 12.4 对晓组织的可操作启发

1. **Worktree 隔离应启用**——`--worktree` flag 是 Claude Code 原生能力，能让"不碰同一文件"从软约束变硬隔离
2. **Plan Approval 是原生对应物**——我们的 QA 审拆法在 Claude 原生层面有 hook 支持（TaskCreated hook exit code 2 可以阻止任务创建）
3. **MAX_HANDOFFS / max_iterations / QA 3轮打回**——三个独立方案在同一问题上收敛到同一数字（3-4），说明这个上限有跨框架共识
4. **Orchestrator 单端点模式**——Google ADK 和 OpenAI SDK 都把 orchestrator 设计为唯一入口，跟晓组织 PM 是唯一协调点一致
5. **Agent Card 概念值得借鉴**——每个晓组织角色可以用一段简短声明描述"我是谁、能做什么、不能做什么"，类似 A2A Agent Card 的精简版。我们的角色 Prompt 模板已经接近这个形态
6. **Handoff Contract 的 ownership 机制**——"只有 owner 写状态，其他人只读"可以迁移到晓组织：任务笔记只有 PM 能写，其他角色只读任务笔记

---

## 十三、2026-05-24 第二轮补充：三大框架完整操作流程

上次补充了架构细节，本次搜的是"具体怎么操作"——创建、配置、执行、清理的完整步骤。

### 13.1 Claude Code Agent Teams——完整操作流程

来源：morphllm.com、GitHub FlorianBruniaux/claude-code-ultimate-guide、SitePoint、腾讯云、博客园、CSDN

#### Step 1：启用

settings.json 加 `"CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"` 或 `export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`。重启后 `/config` 确认 "Agent Teammate mode" 选项出现。需要 Claude Pro/Max + Opus 4.6，CLI 必须（浏览器版不支持）。

#### Step 2：自然语言创建团队

不需要写配置文件——直接对话描述团队：

```
> Create a team with 3 teammates to build the user settings page.
  - Teammate 1: backend API routes and database schema
  - Teammate 2: frontend React components
  - Teammate 3: integration tests
```

```
> Create an agent team to review PR #142. Spawn three reviewers:
  - One focused on security implications
  - One checking performance impact
  - One validating test coverage
```

Claude 自动建团队、spawn 队友、分发任务列表、启动协调。

#### Step 3：任务协调——两种分配方式

| 方式 | 机制 |
|------|------|
| Lead 分配 | 告诉 lead 哪个任务给哪个队友 |
| 自取（self-claim） | 队友完成后自动领下一个未分配、未阻塞的任务 |

任务三态：`pending → in_progress → completed`。任务可设依赖——阻塞任务在依赖完成后自动解阻塞。文件锁防竞态——同一任务不会被两个队友同时认领。

#### Step 4：Plan Approval

```
> Spawn an architect teammate to refactor the authentication module.
  Require plan approval before they make any changes.
```

队友在 Lead 批准前只读模式——对应我们的 QA 审拆法，但是 Claude 原生 hook 层面实现。

#### Step 5：队友间通信——Mailbox

队友的文本输出**对团队不可见**——必须显式用 `write` 发消息。支持 1:1 私信和全员广播。消息自动投递无需轮询。Lead 自动收到队友空闲通知。

#### Step 6：键盘操作

| 操作 | 快捷键 |
|------|--------|
| 下一个队友 | Shift+Down |
| 上一个队友 | Shift+Up |
| 展开队友视图 | Enter |
| 中断当前回合 | Escape |
| 回到 Lead 视图 | Escape |
| 切换任务列表 | Ctrl+T |

#### Step 7：关闭——顺序不能反

1. 先逐个关队友：`> Ask the security-reviewer teammate to shut down`
2. 所有队友停后再清理：`> Clean up the team`

倒过来会失败——Lead 清理时检查活跃队友，有活的就拒绝。

#### 完整生命周期

```
1. Create Team → 2. Create Tasks (含依赖) → 3. Spawn Teammates (并行)
→ 4. Work (claim→execute→complete) → 5. Coordinate (消息/重分配/解阻塞)
→ 6. Shutdown Teammates (逐个) → 7. Cleanup Team Resources
```

#### 文件结构

```
~/.claude/
├── teams/{team-name}/
│   ├── config.json          # Members, agent IDs
│   └── inboxes/{agent}.json # 每个 agent 的 mailbox
└── tasks/{team-name}/
    ├── 1.json
    ├── 2.json
    └── 3.json
```

#### 最佳实践

1. 任务尽量独立——最小化依赖以最大化并行
2. 团队 3-5 人起步——超过 5 人协调成本上升
3. 给自包含的任务描述——队友不继承 Lead 对话历史
4. Lead 用 Opus，队友用 Sonnet——省钱
5. 成本按队友线性增长——每个队友是独立 Claude 实例

#### 适用/不适用

| 适用 | 不适用 |
|------|--------|
| 并行代码审查 | 严格串行依赖 |
| 全栈功能开发（前后端+测试并行） | 同一文件同时编辑（除非 worktree 隔离） |
| 竞争性假设调试 | 简单单文件改动 |
| 大规模独立模块重构 | 复杂交织依赖的任务 |

#### 已知限制

- `/resume` 和 `/rewind` 不恢复队友——resume 后需重新 spawn
- 队友有时忘记标记任务完成
- 关闭慢——队友完成当前请求后才停
- 每会话只能一个团队
- 不能嵌套团队
- Lead 不可转让
- 队友可各自指定模型（官方确认不强制同模型），但每会话只能一个团队
- Split panes 不支持 VS Code/Windows Terminal/Ghostty

#### C 编译器案例

Anthropic 工程团队，16 Agent，2 周，Rust 写 C 编译器（10 万+行）：$20K API 花费、20 亿输入 token、约 2000 会话、GCC torture 测试 99% 通过、能编译 Linux 内核/QEMU/FFmpeg/SQLite/PostgreSQL/Redis、在 x86/ARM/RISC-V 上启动 Linux 6.9。

---

### 13.2 CrewAI——完整操作流程

来源：DigitalOcean、GUVI、百度开发者、ZeroToMastery

#### 四件套心智模型

| 组件 | 作用 |
|------|------|
| Agents | 角色化 LLM 工人，role+goal+backstory+tools+memory |
| Tasks | 具体工作单元，description+expected_output+context（通过 context 传依赖，非 depends_on 参数） |
| Tools | Agent 可调用的能力/API |
| Crew | 编排器，组装 agents+tasks+process，执行 kickoff |

#### Step 1：定义 Agent——Role/Goal/Backstory/Tools

```python
researcher = Agent(
    role="Senior Research Analyst",
    goal="Identify emerging AI trends and their business implications",
    backstory="You have 10 years of experience in AI research and market analysis.",
    tools=[web_search, FileReadTool()],
    memory=True,
    verbose=True,
    allow_delegation=False
)
```

设计原则：单一职责、最小权限、状态隔离（避免 Agent 间共享可变状态）。

#### Step 2：定义 Task——Description/Expected Output/Dependencies

```python
research_task = Task(
    description="Research the topic: {{topic}}. Gather key facts, statistics, and expert opinions.",
    expected_output="A structured research document with key points and sources.",
    agent=researcher,
    async_execution=False,
    output_file="research_findings.md"
)

write_task = Task(
    description="Write a summary report on '{{topic}}' using the research findings.",
    expected_output="A well-structured article with introduction, body, and conclusion.",
    agent=writer,
    context=[research_task],        # 传 researcher 输出给 writer
    context=[research_task],        # 通过 context 传依赖（CrewAI 无 depends_on 参数）
    output_file="final_report.md"
)
```

**Task 生命周期：** Parse（自然语言→结构化任务对象）→ Plan（PDDL 规划器生成执行路径 DAG）→ Execute（消息队列分发子任务）→ Validate（断言库检查输出正确性）。

#### Step 3：选择协作模式（Process Type）

| 模式 | 行为 | 适用 |
|------|------|------|
| Sequential | 线性串行，A→B→C | 简单管道、确定性工作流 |
| Hierarchical | Manager Agent 动态委派 | 复杂工作流需协调 |
| Consensual（规划中，尚未实现） | 投票解决冲突 | 高风险决策需可靠性 |

除这三种模式外，还支持显式协作拓扑：Pipeline（串行阶段）、Blackboard（共享中间结果并行处理）、Negotiation（投票冲突解决）。

#### Step 4：组装 Crew

```python
crew = Crew(
    agents=[researcher, writer],
    tasks=[research_task, write_task],
    process=Process.SEQUENTIAL,
    memory=True,        # Agent 间共享记忆
    verbose=True
)
```

Hierarchical 模式需指定 manager_agent：

```python
crew = Crew(
    agents=[researcher, writer, analyst],
    tasks=[research_task, write_task, analysis_task],
    process=Process.HIERARCHICAL,
    manager_agent=manager,
    memory=True
)
```

#### Step 5：执行

```python
result = crew.kickoff(inputs={"topic": "AI Safety in Enterprise 2026"})
```

底层：Sequential 模式下逐个执行任务；Hierarchical 模式下 manager agent 动态编排；memory=True 时 agent 间共享上下文/输出。

#### 高级特性

**动态任务分解：**
```python
complex_task = Task(
    description="Handle customer complaint",
    # 注意：DecompositionStrategy 类在 CrewAI 中不存在，任务分解靠 planning=True + Process.hierarchical
    subtasks=[
        {"name": "classify_complaint", "required_skills": ["NLP"]},
        {"name": "assign_specialist", "required_skills": ["routing"]}
    ]
)
```

**三层重试策略：**
1. 即时重试（同 agent，最多 3 次）
2. 延迟重试（换备选 agent）
3. 人工升级（通知管理员）

**Flows 条件分支（非 "CrewFlow"，官方无此名称）：**
```python
patent_flow = Flow(
    tasks=[
        {"id": "prior_art_search", "agent": "search_agent", "input": "${query}"},
        {"id": "novelty_assessment", "agent": "analysis_agent",
         "conditions": [
             {"if": "${novelty_score} < 0.7", "then": "reject"},
             {"else": "proceed"}
         ]}
    ]
)
```

#### 操作总结

| 步骤 | 动作 |
|------|------|
| 1 | 定义 Agent——role, goal, backstory, tools, memory |
| 2 | 定义 Task——description, expected output, dependencies |
| 3 | 选 Process——Sequential / Hierarchical / Consensual（规划中） |
| 4 | 组装 Crew——agents + tasks + process type |
| 5 | 执行——crew.kickoff(inputs={...}) |
| 6 | 迭代——verbose=True 调试，精炼 agent prompt，加 guardrails |

---

### 13.3 LangGraph——图结构工作流+Checkpoint+Human-in-the-Loop

来源：LangGraph 官方文档、GitHub zhang-juntai/langgraph-multi-agent-analytics、MachineLearningMastery、百度开发者

#### 三个原语

| 组件 | 角色 |
|------|------|
| State | 类型化共享数据对象（TypedDict 或 Pydantic），携带跨步骤上下文——消息、工具输出、元数据、迭代计数器 |
| Nodes | Python 函数或类（LLM 调用、工具调用、人工审查门），读当前状态返回部分更新 |
| Edges | 节点间过渡——静态（A→B）、条件（路由函数检查状态返回下一个节点名）、循环（回环） |

与线性 Chain 不同，LangGraph 原生支持循环、分支、回环。

#### Supervisor 模式——完整六步

```
Entry → Supervisor → [Router Decision] → Specialist Agent → Supervisor → ... → END
```

**1. 定义多 Agent 状态：**
```python
class MultiAgentState(TypedDict):
    messages: Annotated[List, operator.add]
    next_agent: str
    research_results: list
    final_output: str
```

**2. Supervisor 节点——每次 agent 返回后重判路由：**
```python
def supervisor_node(state: MultiAgentState) -> dict:
    system = SystemMessage(content="""You are a supervisor.
    Route to: 'researcher', 'analyst', 'writer', or 'FINISH'.""")
    response = llm.invoke([system] + state["messages"])
    return {"next_agent": response.content.strip()}
```

**3. 专业节点——各司其职：**
```python
def researcher_node(state): return {"research_results": [...], "messages": [...]}
def analyst_node(state): return {"messages": [...]}
def writer_node(state): return {"final_output": "...", "messages": [...]}
```

**4. 连线建图：**
```python
workflow = StateGraph(MultiAgentState)
workflow.add_node("supervisor", supervisor_node)
workflow.add_node("researcher", researcher_node)
workflow.add_node("analyst", analyst_node)
workflow.add_node("writer", writer_node)
workflow.set_entry_point("supervisor")

def route_to_agent(state): return state["next_agent"]
workflow.add_conditional_edges("supervisor", route_to_agent, {
    "researcher": "researcher", "analyst": "analyst",
    "writer": "writer", "FINISH": END
})
for agent in ["researcher", "analyst", "writer"]:
    workflow.add_edge(agent, "supervisor")  # 所有人执行完回 supervisor

app = workflow.compile(checkpointer=memory)
```

**5-6. 一个生产级企业分析链的实际节点序列：**
```
planner → memory_hydrate → clarification_gate → approval_gate
→ prepare_query → lint → execute_query → validate → report
→ persist → context_checkpoint → memory_persist → alerts
```
每步是独立、可审计、可恢复的节点——不是单体 LLM 调用。

#### Checkpoint——断点续跑的工程基础

每个 **super-step**（一次图"滴答"，可含多个并行节点）执行后**自动保存完整状态快照**，支持暂停/恢复、故障回滚、跨服务器重启的长运行工作流、完整状态转换审计追踪。

**开发环境（内存）：**
```python
from langgraph.checkpoint.memory import MemorySaver
memory = MemorySaver()
app = workflow.compile(checkpointer=memory)
config = {"configurable": {"thread_id": "session-001"}}
```

**轻量/单机环境（官方不推荐生产）：**
```python
from langgraph.checkpoint.sqlite import SqliteSaver
with SqliteSaver.from_conn_string("agent_memory.db") as checkpointer:
    app = workflow.compile(checkpointer=checkpointer)
```
**生产环境：** PostgresSaver（推荐）、RedisSaver（高吞吐）。也支持 S3 兼容对象存储。

**崩溃恢复：**
```python
current_state = app.get_state(config)
print(f"Next node: {current_state.next}")  # 知道停在哪
for event in app.stream(None, config):      # 从断点恢复
    pass
```

#### Human-in-the-Loop——真正的暂停（不是"请批准"提示）

图**真的停**在指定节点，状态持久化到 checkpointer，通过 API 恢复。

**Step 1——定义含审批字段的状态：**
```python
class AgentState(TypedDict):
    draft: str
    approved: bool
    sent: bool
```

**Step 2——定义节点：**
```python
def draft_node(state): return {"draft": "Email draft...", "approved": False, "sent": False}
def send_node(state):
    if state.get("approved"): return {"sent": True}
    else: return {"sent": False}
```

**Step 3——在风险动作前设置暂停点：**
```python
app = workflow.compile(checkpointer=memory, interrupt_before=["send_message"])
```

**Step 4——运行——图会在 send_node 前自动停下：**
```python
for event in app.stream(initial_state, config):
    pass
state = app.get_state(config)
print(f"Paused at: {state.next}")  # ('send_message',)
```

**Step 5——人工审查并更新状态：**
```python
app.update_state(config, {"approved": True})
```

**Step 6——从暂停点恢复：**
```python
for event in app.stream(None, config):  # None = "从上次停的地方继续"
    pass
```

#### 程序化 HITL（`interrupt()` 函数）

```python
@tool
def human_assistance(query: str) -> str:
    interrupt_data = interrupt({
        "type": "human_review",
        "query": query,
        "context": {"pending_actions": [...]}
    })
    return interrupt_data.get("response", "")
```

#### HITL 触发策略

| 触发条件 | 示例 |
|---------|------|
| 置信度阈值 | 模型置信度 < 0.85 → 暂停 |
| 敏感域检测 | 金融/医疗/法律关键词 → 暂停 |
| 业务规则匹配 | 交易金额 > $10K → 暂停 |
| 强制门禁 | 澄清节点、审批节点永远暂停 |

#### 两种 HITL 模式

1. **Clarification Gate**——信息不充分，图暂停，用户补充，图恢复
2. **Approval Gate**——高风险动作需授权，图暂停，批准记录在 ApprovalStore，只在批准后恢复

#### 三层记忆架构

> LangSmith 是可观测性/追踪平台，非记忆层本身。

| 层级 | 存储 | 用途 |
|------|------|------|
| 短期 | Checkpointer 层（Redis/Postgres/SQLite 均可） | 当前对话状态、暂停恢复 |
| 长期 | Store API（Postgres+pgvector / Redis Vector / Milvus） | 结构化知识、用户偏好、跨会话记忆 |
| 程序性 | Reflection Node → 写回 Store | 执行日志、决策路径、策略更新 |

#### 适用对照表

| 需求 | LangGraph 功能 |
|------|---------------|
| 多专业 agent 协作 | Supervisor 模式+条件边 |
| 循环、重试、迭代精炼 | 循环图+状态驱动路由 |
| 关键步骤人工审批暂停 | interrupt_before / interrupt() |
| 崩溃和服务器重启存活 | 数据库持久化 checkpointer |
| 从确切失败点恢复 | app.get_state() + app.stream(None, config) |
| 审计每个决策 | LangSmith 追踪 + checkpointer 历史 |
| 跨小时/天的长运行工作流 | 持久化状态快照 + WorkflowPauseStore |

2026 年业界预测：超过 60% 企业 AI 应用将采用图结构工作流框架。**[经官方文档验证，此数字无可追溯一手来源。参考：Gartner 预测 40% 企业应用 2026 年底前集成任务专用 AI Agent；远期 80% 企业 2027 年前使用 AI 工作流编排工具]**

---

### 13.4 晓组织与三家操作层面对比

| 能力 | Claude Code Agent Teams | CrewAI | LangGraph | 晓组织 |
|------|------------------------|--------|-----------|--------|
| 团队创建 | 自然语言 | Python 代码 | Python 代码 | 自然语言（当前 Claude 实例做 PM） |
| 任务分配 | Lead 派 + 自取 | 代码声明 depends_on | 图路由函数 | PM 手动拆+派 |
| 角色定义 | spawn 时的自然语言描述 | Agent(role, goal, backstory) | Node 函数 | Prompt 模板（第二章） |
| 质量控制 | Plan Approval hook | Hierarchical manager 审查 | interrupt_before 暂停+人工 | QA 审查+合规巡视 |
| 循环上限 | TaskCompleted hook | 三层重试（3次→换人→人工） | LoopAgent max_iterations | QA 打回最多 3 轮 |
| 断点恢复 | /resume 不恢复队友 | 无 | checkpoint + resume | 无 |
| 执行中途分支 | 手动叫停重派 | Flows @router 条件装饰器 | 条件边（conditional edge） | 判错止损→升级路径 |
| 并行 | 多个队友并行 spawn | async_execution | ParallelAgent | 互不依赖的子任务并行派 |
| 通信 | Mailbox（1:1 + 广播） | context/depends_on 传输出 | 共享 State | PM 是唯一协调点（不直接通信） |
| 隔离 | worktree（独立 git 分支） | 进程级 | 进程级 | 无硬隔离 |
| 记忆 | 独立上下文窗口 | memory=True（共享） | Checkpointer + Store + LangSmith 追踪 | 任务笔记（临时文件） |

---

## 十四、2026-05-24 第三轮补充：MetaGPT——最接近三省六部制的框架

来源：MetaGPT 官方文档、SegmentFault、CSDN、GitCode、百度开发者

### 14.1 核心理念

`Code = SOP(Team)`——把软件公司的标准作业程序编码到多 Agent 系统中。不是让一个 Agent 黑箱式生成代码，而是模拟真实软件公司的角色化流水线。

### 14.2 五角色 SOP 流水线

```
用户需求 → ProductManager → Architect → ProjectManager → Engineer(n_borg) → QaEngineer → 交付
```

> **2026-05-24 验证：** 当前默认流水线已变更——ProjectManager、Engineer(n_borg)、QaEngineer 在源码中被注释，替换为 Engineer2 + DataAnalyst。以下 Step 1-5 描述的是原始设计，非当前默认行为。

#### Step 1 — ProductManager：PRD 生成

- 动作：`WritePRD`、竞品研究、四象限图生成
- 2026 新增：`_research_competitors` 在起草 PRD 前实时搜索竞品情报
- 产出 `PRD.md` 含：产品目标+成功 KPI、用户画像+用户故事、竞品分析（自动生成四象限图）、功能需求（P0/P1/P2 优先级）、非功能需求（性能/安全）、MVP 范围+迭代路线图
- 代码路径：`metagpt/roles/product_manager.py` → `metagpt/actions/write_prd.py`

#### Step 2 — Architect：系统设计+技术蓝图

- 读 PRD → 技术选型 → 模块划分 → API 设计 → 时序流程图
- 产出：`design.md`（文件结构+类层次+数据流）、Mermaid 格式时序流程图、`TechStack.md`（前后端/数据库选型+理由）、OpenAPI 格式 API 规范
- 代码路径：`metagpt/roles/architect.py`

#### Step 3 — ProjectManager：任务拆解

- 解析设计模块 → 拆成文件级子任务 → 构建依赖 DAG（有向无环图）
- 产出 `tasks.json` 含：任务 ID+描述、分配工程师槽位、依赖图（拓扑排序）、每任务预估工作量
- **DAG 并行调度**：互不依赖的任务自动标记为可并行，交给 n_borg 工程师池同时执行
- 代码路径：`metagpt/roles/project_manager.py`

#### Step 4 — Engineer(n_borg)：并行开发

- `n_borg=5` 创建 5 个并发的 "Alex" 工程师
- 动作：`WriteCode`、`WriteCodeReview`、`SummarizeCode`、`FixBug`
- 产出：`workspace/src/` 源代码文件、代码摘要（供下游迭代理解）、QA 打回时的重构代码
- **代码审查机制**（`use_code_review=True`）：1) 工程师生成代码 → 2) WriteCodeReview 动作检查代码 → 3) 问题自动修正后才存仓库 → 4) 生成代码摘要供后续工程师快速理解已有代码
- **并行效率**：n_borg=1 时总时间=Σ(每任务时间)；n_borg=5 时总时间≈max(每任务时间)
- 代码路径：`metagpt/roles/engineer.py`

#### Step 5 — QaEngineer：测试+Bug 修复闭环

- 动作：`WriteTest`、`RunTests`、`AnalyzeFailure`、`FixBug`（触发 Engineer）
- 产出：`tests/` 测试代码、测试执行报告、Bug 报告（以消息发送给 Engineer agent）
- **Bug 修复闭环**：读代码→生成测试→执行测试→分析失败→Bug 报告→Engineer 修→再测
- 测试策略：边界值（自动识别参数边缘 0/null/最大值）、异常路径（练习错误分支）、集成测试（基于 Architect 的 API 设计）、覆盖率导向（可配阈值，默认 80%）
- 代码路径：`metagpt/roles/qa_engineer.py`

### 14.3 关键运行参数

| 参数 | 作用 | 典型值 |
|------|------|--------|
| `n_borg` | 并行工程师数量 | 3-5 |
| `use_code_review` | 启用自动代码审查 | True |
| `investment` | API 费用上限（美元） | 3.0-10.0 |
| `n_round` | 最大迭代轮次 | 5-10 |
| `max_auto_summarize_code` | 几轮后自动摘要 | 3 |

### 14.4 一条命令跑全程

```bash
pip install metagpt
metagpt --init-config
metagpt "Build a CLI 2048 game with scoring, difficulty levels, and save/load"
```

执行日志：
```
[INFO] Boss: Received requirement, analyzing...
[INFO] ProductManager: Writing PRD...
[INFO] Architect: Designing system architecture...
[INFO] ProjectManager: Tasks decomposed, assigning to Engineers
[INFO] Engineer: Writing code... (5 parallel instances)
[INFO] QA: All tests passed. Project generated at ./workspace/
```

### 14.5 代码级启动

```python
import asyncio
from metagpt.team import Team
from metagpt.roles import ProductManager, Architect, ProjectManager, Engineer, QaEngineer

async def start_software_company():
    company = Team()
    company.hire([
        ProductManager(),       # Alice - 产品经理
        Architect(),            # Bob - 架构师
        ProjectManager(),       # Eve - 项目经理
        Engineer(n_borg=5, use_code_review=True),
        QaEngineer(),           # Edward - QA工程师
    ])
    company.invest(investment=3.0)
    company.run_project("开发一个支持OAuth2.0的待办事项Web应用")
    await company.run(n_round=5)
```

### 14.6 记忆三层架构

| 层级 | 存储 | 用途 |
|------|------|------|
| 短期 | 对话上下文窗口 | 当前任务上下文 |
| 工作 | 共享项目产物（PRD/设计文档/代码） | 角色间信息传递 |
| 长期 | 向量数据库 | 历史成功项目经验，新任务参考 |

### 14.7 任务调度算法

基于 DAG 的最大匹配度调度——保证依赖关系正确的同时，将子任务分配给最匹配的角色。互不依赖的任务自动并行。

### 14.8 核心交付物一览

| 阶段 | 角色 | 交付物 |
|------|------|--------|
| 需求分析 | ProductManager | PRD.md（用户画像+功能列表+竞品分析） |
| 系统设计 | Architect | 设计文档+时序图+技术栈说明 |
| 任务管理 | ProjectManager | tasks.json+依赖关系图 |
| 代码实现 | Engineer(n_borg) | 完整源代码+代码摘要+审查报告 |
| 质量保障 | QaEngineer | 测试代码+测试报告+Bug 报告 |

### 14.9 MetaGPT vs 晓组织——结构对比

| 维度 | MetaGPT | 晓组织 |
|------|---------|--------|
| PM 做什么 | 竞品搜索+写 PRD | 查历史+理解需求+脑中拆+判路径（Step 0 全部） |
| PM 拆任务 | ProjectManager 单独角色，DAG 自动调度 | PM 手动拆，标依赖关系 |
| 方案设计 | Architect 出系统设计+API 规范+时序图 | 架构小克 出方案（概述+技术路线+风险+工作量） |
| 并行执行 | n_borg 自动并行，DAG 调度 | PM 手动判断"互不依赖→并行派" |
| 代码审查 | Engineer 内置 WriteCodeReview，自动修正 | QA小克 独立审查，打回→工程修改 |
| Bug 修复 | QA→Engineer 自动消息闭环 | QA 打回→工程修改→再审（手动） |
| 测试 | QaEngineer 自动生成+执行+分析失败 | 整体验证（PM 手动跑） |
| 额外角色 | 无 | 合规小克 / 财务小克 / 秘书小克 |
| 领域绑定 | 软件工程 | 通用（任何任务类型） |
| 启动方式 | Python 代码 `company.hire([...])` | 自然语言，PM 按操作手册手动 spawn |
| 记忆 | 三层（短期/工作/长期向量） | 任务笔记（临时文件） |
| 成本控制 | `investment=3.0` 硬上限 | 财务小克 事前估算+事后 ROI |

### 14.10 启发

1. **n_borg 并行模式**可直接借鉴——我们的牛马小卡并行派发跟 n_borg 是同构的，但我们没有 DAG 自动调度，靠 PM 手动判依赖
2. **Bug 修复闭环的自动化**——QA→Engineer 消息自动触发 FixBug，我们的 QA 打回靠 PM 手动中转。Agent Teams 的 mailbox 可以实现这个
3. **代码审查内建到工程师**——WriteCodeReview 在存仓库前自动跑，我们的 QA 审查是独立的、事后的。两种思路各有道理：内建更快，独立更可靠
4. **investment 硬上限**——MetaGPT 设了美元硬上限，我们的财务小克只标注不否决。"不否决"是设计选择，但 MetaGPT 证明硬上限可行
5. **DAG 任务调度**——ProjectManager 自动构建依赖图+拓扑排序，我们的 PM 手动标依赖——差距在自动化程度，不在架构理念
6. **MetaGPT 绑死软件工程**——晓组织的通用性是真正的差异化优势，但 MetaGPT 的 SOP 角色固化程度更深（角色有固定代码文件，不是 prompt 模板）

---

## 十五、2026-05-24 官方文档验证——Claude Code 多 Agent 能力全貌

三个 Agent 并行访问官方文档（CDP 浏览器直读 docs.anthropic.com 和 code.claude.com/docs），一手来源。

### 15.1 总体架构：五种并行方式

来源：https://code.claude.com/docs/en/agents.md

官方定义了五种让多个 Agent 并行工作的方法：

| 方式 | 本质 | 适用场景 |
|------|------|----------|
| **Subagents** | 单个会话内委派的工作者，在自己的上下文中执行任务，返回摘要 | 子任务产生大量搜索结果/日志/文件内容，不希望占用主对话上下文 |
| **Agent View** | 一个屏幕 dispatch 和监控后台运行的多个会话 | 多个独立任务，分派出去，扫一眼状态，只在需要时介入 |
| **Agent Teams** | 多个协调的会话，共享任务列表+Agent 间消息，由 leader 管理（实验性，默认关闭） | 希望 Claude 自己拆分项目、分配任务、保持工作者同步 |
| **Worktrees** | 独立的 git checkout，确保并行会话互不干扰 | 手动运行多个会话，或子代理需要编辑重叠的文件 |
| **`/batch` 命令** | 将一个大型变更拆分为 5-30 个工作树隔离的子代理，每个开一个 PR | 仓库级迁移或机械化重构 |

所有方式中的"工作者"都是 Claude 会话，这些方式可以组合使用。

### 15.2 Subagents——最核心的原生能力

> 详细使用指南见 [小小克使用指南](~/projects/小小克/subagent-guide.md)（frontmatter 字段、spawn 规范、权限模式、已知限制等）。本节聚焦 Subagent 在多 Agent 体系中的定位。

来源：https://code.claude.com/docs/en/sub-agents.md

#### 官方定义

> Subagents are specialized AI assistants that handle specific types of tasks. Use one when a side task would flood your main conversation with search results, logs, or file contents you won't reference again: the subagent does that work in its own context and returns only the summary.

每个子代理在**自己的上下文窗口**中运行，拥有**自定义系统提示**、**特定工具访问权限**和**独立权限**。

#### 五种内置子代理

| 名称 | 模型 | 工具 | 用途 |
|------|------|------|------|
| **Explore** | Haiku | 只读 | 快速搜索和分析代码库。thoroughness: quick/medium/very thorough |
| **Plan** | 继承主会话 | 只读 | Plan mode 下的代码库研究。不能嵌套生成子代理 |
| **General-purpose** | 继承主会话 | 所有 | 复杂多步骤任务，需要探索+修改 |
| **statusline-setup** | Sonnet | - | 运行 `/statusline` 时自动调用 |
| **claude-code-guide** | Haiku | - | 回答 Claude Code 功能相关问题时自动调用 |

Explore 和 Plan 是唯一跳过 CLAUDE.md 和 git status 的子代理（为速度和成本）。

#### 自定义子代理配置

用 Markdown + YAML frontmatter 定义：

```yaml
---
name: code-reviewer
description: Reviews code for quality and best practices
tools: Read, Glob, Grep
model: sonnet
---
(system prompt in markdown body)
```

**所有 frontmatter 字段：**

| 字段 | 必填 | 说明 |
|------|------|------|
| `name` | 是 | 唯一标识符，小写+连字符 |
| `description` | 是 | 告诉 Claude 何时委派给此子代理 |
| `tools` | 否 | 可用工具白名单。省略则继承所有工具 |
| `disallowedTools` | 否 | 要移除的工具 |
| `model` | 否 | sonnet/opus/haiku/full model ID/inherit（默认 inherit） |
| `permissionMode` | 否 | 权限模式（插件子代理忽略此字段） |
| `maxTurns` | 否 | 最大 agentic turns |
| `skills` | 否 | 启动时预加载到上下文的技能列表 |
| `mcpServers` | 否 | 可用的 MCP 服务器（插件子代理忽略此字段） |
| `hooks` | 否 | 作用域为此子代理的生命周期钩子（插件子代理忽略此字段） |
| `memory` | 否 | 持久化记忆范围：user/project/local |
| `background` | 否 | 是否始终作为后台任务运行 |
| `effort` | 否 | 努力级别：low/medium/high/xhigh/max |
| `isolation` | 否 | 设为 `worktree` 则在临时 git worktree 中运行 |
| `color` | 否 | 显示颜色 |
| `initialPrompt` | 否 | 作为主会话 agent 运行时自动提交的第一个用户 turn |

#### 五种作用域优先级（从高到低）

1. **Managed settings**（组织级，管理员部署）
2. **`--agents` CLI flag**（当前会话）
3. **`.claude/agents/`**（项目级，可纳入版本控制）
4. **`~/.claude/agents/`**（用户级，所有项目可用）
5. **Plugin 的 `agents/` 目录**（插件级）

同名时高优先级覆盖低优先级。项目级和用户级目录递归扫描，插件级子目录成为 scoped identifier。

#### 调用方式

- **自然语言**：提示中提名字，Claude 决定是否委派
- **@-mention**：`@"agent-name"` 确保该子代理一定运行
- **会话级**：`claude --agent <name>` 整个会话使用该子代理的系统提示+工具+模型
- **自动委派**：Claude 根据 description 字段自动匹配

#### 上下文加载详情

子代理启动时加载：自己的系统提示+环境的补充细节、委派提示词、各级 CLAUDE.md 和 memory、父会话启动时的 git 状态快照、skills 字段指定的 skill 完整内容。

不加载：父会话的对话历史和工具结果。Explore 和 Plan 是唯一跳过 CLAUDE.md 和 git status 的子代理，无 frontmatter 字段可改此行为。

#### 关键限制

- **子代理不能 spawn 其他子代理**——硬限制。官方："Subagents cannot spawn other subagents. Don't include Agent in a subagent's tools array."
- 文件修改需重启会话生效（通过 `/agents` 界面创建的立即生效）
- 同名冲突无警告——同一作用域内两个同名文件，保留一个丢弃另一个
- 插件子代理不支持 hooks/mcpServers/permissionMode 字段（安全原因）
- 父级权限模式优先——父用 bypassPermissions 或 acceptEdits 时子代理不能覆盖
- 子代理后台运行时自动拒绝需提示的权限请求

#### 子代理定义可复用为 Agent Teams 队友

> When spawning a teammate, you can reference a subagent type from any subagent scope. The teammate honors that definition's tools and model, and the definition's body is appended to the teammate's system prompt as additional instructions.

skills 和 mcpServers frontmatter 在作为队友运行时**不生效**——队友从项目和用户设置加载。

#### 权限模式

| 模式 | 行为 |
|------|------|
| default | 标准权限检查+提示 |
| acceptEdits | 自动接受文件编辑和常见文件系统命令 |
| auto | 后台分类器审查命令和受保护目录写入 |
| dontAsk | 自动拒绝权限提示 |
| bypassPermissions | 跳过所有权限提示（.git/.claude/.vscode 等保护目录也放开，根/家目录删除仍提示） |
| plan | Plan mode（只读探索） |

#### Fork 模式（实验性）

`CLAUDE_CODE_FORK_SUBAGENT=1` 启用。Fork 子代理继承完整对话历史而非从零开始，解决"给子代理太多背景信息"的问题。限制：Fork 不能再 spawn Fork。需要 Claude Code v2.1.117+。

| | Fork | Named subagent |
|---|------|----------------|
| 上下文 | 完整对话历史 | 全新上下文+传入的 prompt |
| 系统提示和工具 | 同主会话 | 来自于代理定义文件 |
| 模型 | 同主会话 | 来自于代理的 model 字段 |
| 权限 | 提示出现在终端 | 后台运行自动拒绝 |
| Prompt cache | 与主会话共享 | 独立 cache |

---

### 15.3 Agent Teams——实验性团队协作

来源：https://code.claude.com/docs/en/agent-teams.md

#### 官方定义

> Agent teams let you coordinate multiple Claude Code instances working together. One session acts as the team lead, coordinating work, assigning tasks, and synthesizing results. Teammates work independently, each in its own context window, and communicate directly with each other.

#### 与 Subagents 官方对比

| 维度 | Subagents | Agent Teams |
|------|-----------|-------------|
| 上下文 | 独立上下文；结果返回给调用者 | 独立上下文；完全独立 |
| 通信 | 只向主代理报告结果 | 队友之间直接互发消息 |
| 协调 | 主代理管理所有工作 | 共享任务列表+自协调 |
| 最适合 | 只需结果的聚焦任务 | 需要讨论和协作的复杂工作 |
| Token 成本 | 较低：结果摘要返回主上下文 | 较高：每个队友是独立的 Claude 实例 |

> Use subagents when you need quick, focused workers that report back. Use agent teams when teammates need to share findings, challenge each other, and coordinate on their own.

#### 架构四件套

| 组件 | 角色 |
|------|------|
| **Team lead** | 主 Claude Code 会话，创建团队、生成队友、协调工作 |
| **Teammates** | 独立的 Claude Code 实例，各自有 1M token 上下文窗口 |
| **Task list** | 共享工作项列表，三态 pending/in progress/completed，支持依赖自动解阻塞 |
| **Mailbox** | Agent 间消息系统，自动投递无需轮询 |

存储于 `~/.claude/teams/{team-name}/config.json` 和 `~/.claude/tasks/{team-name}/`。Claude 自动生成和管理，不可手动编辑。

#### 启用

`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`，需 Claude Code v2.1.32+。

#### 自然语言创建

告诉 Claude 创建团队+描述任务+团队结构，Claude 自动建团队、spawn 队友、分发任务、协调工作。

两种启动方式：1) 用户请求团队 2) Claude 判断任务适合并行→提议创建→用户确认后执行。两种情况下 Claude 不会未经批准创建团队。

#### 显示模式

- **In-process**（默认）：所有队友在主终端内，Shift+Down 切换
- **Split panes**：每个队友独立 tmux/iTerm2 窗格，不支持 VS Code/Windows Terminal/Ghostty

#### 任务协调

两种分配：Lead 指派 / 自取（self-claim，完成后自动领下一个未分配未阻塞任务）。文件锁防竞态。依赖自动管理——阻塞任务在依赖完成后自动解锁。

#### Plan Approval

队友先 plan→发送审批请求给 lead→lead 批准/拒绝→被拒绝后队友留在 plan mode 修改重提→批准后开始执行。可给 lead 审批标准："只批准含测试覆盖的计划""拒绝修改数据库 schema 的计划"。

#### 队友间通信

每个队友独立上下文窗口，不继承 lead 对话历史。加载 CLAUDE.md、MCP 服务器、skills，收到 spawn prompt。必须显式用 SendMessage 发消息。1:1 或全员（逐个发）。消息自动投递。空闲自动通知 lead。

#### 团队 Hook 事件

| Hook | 触发时机 | 行为 |
|------|---------|------|
| **TeammateIdle** | 队友即将空闲 | exit code 2 发反馈让队友继续 |
| **TaskCreated** | 任务创建 | exit code 2 阻止创建 |
| **TaskCompleted** | 任务标记完成 | exit code 2 阻止完成 |

#### 完整已知限制

1. `/resume` 和 `/rewind` 不恢复 in-process 队友——resume 后重新 spawn
2. 任务状态可能滞后——队友有时忘记标记完成
3. 关闭慢——队友完成当前请求后才停
4. **一个会话只能一个团队**——清理当前团队后才能建新的
5. **不能嵌套团队**——队友不能创建子团队
6. **Lead 不可转让**——创建团队的那个会话永远是 lead
7. **权限 spawn 时设定**——所有队友以 lead 权限模式启动，spawn 后可按个改但不能 spawn 时指定
8. Split panes 需 tmux 或 iTerm2
9. Agent teams 实验性，默认关闭

#### 最佳实践（官方）

1. 给队友足够上下文——队友不继承 lead 对话历史，spawn prompt 必须自包含
2. 3-5 人起步——无硬上限但协调成本指数增长。3 个聚焦的胜过 5 个分散的
3. 每人 5-6 个任务——防频繁上下文切换
4. 任务粒度适当——太小协调成本>收益，太大风险高。自包含单元+明确可交付物
5. 等队友干完——lead 有时自己上手不等队友，发现了要提醒"等队友完成再继续"
6. 从研究和审查起步——新人先用有明确边界的非编码任务练手
7. 避免文件冲突——每个队友拥有不同文件集
8. 监控和引导——查看进度、重定向不工作的方案、随时合成发现。长时间无人看管增加浪费风险

#### 适用场景

- 研究和审查：多队友同时探索不同维度，互相分享和质疑
- 新模块/功能：各管一块互不踩踏
- 竞态假设调试：多队友并行测试不同理论
- 跨层协调：前端/后端/测试各由一个队友负责

不适用：严格串行任务、同文件编辑、多依赖的工作

#### 清理团队

先逐个关队友（"Ask the researcher teammate to shut down"），再让 lead 清理（"Clean up the team"）。清理时检查活跃队友，有活的就失败。必须用 lead 清理——队友不能清，因为团队上下文可能解析不正确。

#### Team Config 说明

`~/.claude/teams/{team-name}/config.json` 含 members 数组（名称、agent ID、agent type），队友可读此文件发现其他成员。运行时状态（session ID、tmux pane ID），不可手动编辑。团队配置没有项目级对应物。

---

### 15.4 Agent View——会话管理面板

来源：https://code.claude.com/docs/en/agent-view.md

`claude agents` 打开，一个屏幕显示所有后台会话。按状态分组：Working（动画）、Needs input（黄色）、Idle（暗色）、Completed（绿色）、Failed（红色）、Stopped（灰色）。可在面板中直接输入提示创建新后台会话，可用自定义子代理作为会话主代理。需要 Claude Code v2.1.139+。

---

### 15.5 Worktrees——工作树隔离

来源：https://code.claude.com/docs/en/worktrees.md

- `--worktree` 标志创建独立的 git checkout
- 子代理可通过 `isolation: worktree` frontmatter 字段配置
- `.worktreeinclude` 文件控制哪些 gitignored 文件被复制到新工作树
- 未产生更改的工作树自动清理
- 支持非 git VCS 通过 `WorktreeCreate`/`WorktreeRemove` hooks
- 注意：Agent Teams 队长无法给队友加 `--worktree`——不是 Agent Teams 的内置机制，是单独的手动方案

---

### 15.6 Hook 系统在 Agent 场景中的角色

来源：https://code.claude.com/docs/en/hooks.md

#### Agent 专属 Hook 事件

| Hook 事件 | 触发时机 | Matcher 支持 |
|-----------|----------|-------------|
| **SubagentStart** | 子代理被生成时 | agent type 名称 |
| **SubagentStop** | 子代理完成时 | agent type 名称 |
| **TeammateIdle** | 队友即将空闲时 | 不支持 |
| **TaskCreated** | 任务被创建时 | 不支持 |
| **TaskCompleted** | 任务被标记完成时 | 不支持 |

#### Agent-based Hooks（实验性）

`type: "agent"` 的 hook——生成一个子代理来验证条件，返回决定。子代理可使用 Read、Grep、Glob 等工具做判断。等于官方版"用 Agent 校验 Agent"。

#### 三层次 Hook 嵌套

1. 子代理 frontmatter 内——该子代理专属
2. settings.json——主会话响应 SubagentStart/SubagentStop
3. Agent team 层面——TeammateIdle/TaskCreated/TaskCompleted

---

### 15.7 权限模型在多 Agent 场景下的行为

来源：https://code.claude.com/docs/en/permissions.md

- 子代理继承父会话权限上下文，可通过 `permissionMode` 覆盖
- 例外：父用 bypassPermissions 或 acceptEdits 时子代理不能覆盖
- auto mode 下子代理继承 auto mode，其 permissionMode 被忽略
- 可通过 `Agent(subagent-name)` 权限规则限制哪些子代理可用
- 子代理 tools 字段可用 `Agent(agent_type)` 限制可 spawn 的子代理类型
- Agent Teams 权限：队友继承 lead，不能 spawn 时指定但可 spawn 后改

---

### 15.8 vs 晓组织——差距与修正

| # | 发现 | 晓组织现状 | 动作 |
|---|------|-----------|------|
| 1 | Worktree 隔离无 `--worktree` CLI flag | spawn 约定写了不存在的 flag | **修**：改为 `isolation: worktree`（subagent）或手动 worktree |
| 2 | Subagent 不能 spawn subagent | 流程没写这条硬限制 | **补**：晓组织设计边界——所有 spawn 必须从 PM 发起 |
| 3 | `/batch` 命令——第五种并行方式 | 完全没提 | **标注** |
| 4 | Fork 模式——子代理继承对话历史 | 不知道 | **标注**，等稳定后考虑启用 |
| 5 | Agent-based Hook——用 Agent 校验 Agent | 不知道 | **标注**，QA 跑熟后考虑自动化 |
| 6 | Subagent memory 字段 | 不知道 | **标注**，角色模板可加 memory 配置 |
| 7 | 子代理定义可复用为 Agent Teams 队友 | 不知道 | 设计参考，不需要改 |
| 8 | Agent View + `/batch` 两种额外并行方式 | 不知道 | **标注** |

---

## 十六、补充：Spawn 机制深度调研

2026-05-24 逐条过操作手册时发现 spawn 信息缺口，补充搜索。内容太多，独立为 [research-spawn.md](research-spawn.md)。涵盖：

- 四套并行机制全景（手动 Agent / Agent Teams / Coordinator Mode / Fork）
- 官方判断标准 + Subagents vs Agent Teams 五维对比
- Coordinator Mode 与萧何的对应
- Orchestrator-Workers 实测数据（+90.2% 性能提升）
- Agent Teams 详细画像（架构/限制/最佳实践/成本）
- 手动 Agent() spawn 详细画像（生命周期/配置/硬限制/权限模式）
- 社区实战（16-Agent C 编译器 / 47 handoff $9.12 反模式）
- 跟晓组织现状的差距分析（8 条）
- 结论：手动 spawn 为主轴，Coordinator Mode 可选增强
