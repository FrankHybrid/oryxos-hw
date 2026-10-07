# OryxOS 与 Hermes Agent 对比分析报告

> **视角**：功能、技术架构、企业价值
> **比对时间**：**2026-10-07**（文档会持续演进，结论以文中标注的具体来源为准）

---

## 0. 结论摘要（TL;DR）

### 0.1 一句话总纲

**两者是同一层（Agent 运行底座）的两个反向答案**：Hermes（v0.20.0）把 Agent 做进**人的工作流**——终端、聊天窗口、编辑器、桌面端、任务板；OryxOS 把 Agent 做进**企业的治理体系**——REST API 之后、审计表里、白名单沙箱里、现有 Java 工具链里。由此，三个角度的差异可以各用一句话概括：

| 角度 | 一句话结论 |
|------|-----------|
| **功能** | 两者的差异是**能力面取向**不同：Hermes 把能力做进工具集与渠道（多模态、20+ 渠道、Skill Hub、A2A、Kanban、桌面端），OryxOS 把能力做进底座机制（一个目录一个 Agent、审计表、工具策略层、知识库）。按"什么都能干"衡量前者更宽，按"干什么都能留下痕迹与约束"衡量后者更完整。 |
| **技术架构** | 分歧在"**状态与边界放在哪一层**"：Hermes 让每个 worker 独立成进程、各自持有状态（隔离强、统一治理弱）；OryxOS 让多 Agent 共享一个进程与一套存储（统一治理强、执行边界依赖白名单）。两种取向各自带来一组可预期的约束。 |
| **企业价值** | 两者的设计意图锚在不同约束上：OryxOS 锚在"数据留在域内 + 纳入既有技术栈的治理流程"，Hermes 锚在"能力完整度 + 快速可用"。**这些是各自声明的主张，不是第三方验证的结论**；企业场景的实际满足度取决于该企业关注哪一类约束（§4 按约束对称展开，不做推荐）。 |

### 0.2 三个角度的关键差异点

**角度一：功能**

- **宽度 vs 收窄**：Hermes 内置工具覆盖网络/终端/浏览器/媒体/编排/记忆/自动化/集成八类，并有视觉、图像生成、TTS、语音模式、LSP、Curator；OryxOS 核心阶段只做 9 个内置工具，把能力扩展权交给 MCP 生态。
- **生态开放度相反**：Hermes 的 Skill Hub 有 7 个外部来源、四级信任分级、`--force` 可覆盖非危险策略，且 Agent 能自建/改写技能（程序性记忆）；OryxOS 明确"Skill 和 Tool 来源受控，不做无约束的公开市场"，市场列为社区共建。
- **"多 Agent"是两种东西**：OryxOS 是**同实例多 Agent 共享一套底座**（一个目录 = 一个 Agent）；Hermes 是**多进程 worker + Kanban 持久任务板**（`delegate_task` 并发 ≤3、深度 ≤2；`kanban swarm` 支持 root/worker/verifier/synthesizer 拓扑）。
- **v0.20.0 新增的加分项**：A2A v1.0 插件、HMAC 签名 Outbound Webhook、`grounded-citations` 可核验引用技能、桌面端 Artifacts + 插件 SDK、实时语音与唤醒词（本地检测、可按唤醒词路由到不同 Profile）。
- **OryxOS 的独占项**：独立知识库模块（切分/向量化/双路召回 + RRF 融合）、`NotifyTools` 出站通知抽象、审计表从 day one 写入、Profile 级 Tool Policy 设计。

**角度二：技术架构**

- **Agent 循环**：OryxOS 自实现 ReAct（约数十行 Java，明确禁用 Spring AI 的自动 tool 执行以防重复调用），核心阶段**不并行、不流式**；Hermes 已支持工具并行、流式、**默认轮数 500**（v0.20.0 起，原 90）、逐轮微压缩与渐进式工具结果裁剪。
- **隔离强度**：OryxOS 核心阶段是**应用层白名单**（文档自认"劝阻级，防不住蓄意绕过"），容器/microVM 为扩展阶段；Hermes 已可用 Docker（只读根文件系统、移除全部 capabilities、禁提权、PID 上限 256、命名空间隔离）/ SSH / Singularity / Modal / Daytona。
- **演进性 vs 即用性**：OryxOS 的 `Sandbox` 接口刻意不含"容器镜像/VM 配置"字样，用 microVM 反向套签名验证中立性——**接口不为某一档实现绑死**；Hermes 是**今天就能跑不可信代码**的强隔离，但没有同类显式的隔离演进抽象。
- **状态与扩展路径**：OryxOS 把"实例无状态、状态外置"作为前提，扩展阶段迁 Redis/PostgreSQL/对象存储，分布式协调留 Nacos/ETCD；Hermes 状态在 `~/.hermes/`（`state.db` / `kanban.db` / `memories/` / `.env`），Kanban 为**同机多进程共享本地 SQLite**，文档未见跨节点调度。
- **分发**：OryxOS 单 fat JAR → GraalVM Native；Hermes v0.20.0 起为 shell 安装器 / Docker / Nix（Node.js 26 为目标运行时），Homebrew 与 pip/PyPI 渠道已退出。
- **审计的实现方式**：OryxOS 落**结构化表**（`tool_invocations` / `llm_calls`，Sandbox 违规复用同一审计路径）；Hermes 是**六类插件钩子**（`pre/post_tool_call`、`pre/post_llm_call`、`on_session_start/end`）+ 日志 + FTS5 会话检索——完整度取决于是否写钩子。

**角度三：企业价值**

> 说明：本节采用"设计意图 → 适用约束"的描述方式，不采用评分或推荐（原因见 §0.3）。两家的企业主张均以**其自身文档为据**，凡属厂商自设标准者就地标注。

- **两家的自我主张**（**厂商声明，非第三方验证**）：OryxOS 主张"数据不出域 + 完全可审计 + 纳入既有治理流程 + 技术栈对齐"；Hermes 主张"能力丰富度 + 开箱即用 + 生态"。前者出自 `IndustryResearch.md` 3.3–3.4，后者出自其功能概览页的自我描述。
- **设计意图与已实现的落差**（需按证据等级阅读）：OryxOS 的审计写入、工具策略层、凭证加密、多副本集群等在仓库宪法中已作为**工程约束**存在（§4.2 表内标等级）；Hermes 的能力以 **v0.20.0 已交付**为准。两者论据性质不同，不宜并列打分。
- **可预期的适用边界**：OryxOS 的单进程统一设计使"一处改策略、全 Agent 生效"成为自然结果，代价是执行边界依赖白名单而非进程/容器隔离；Hermes 的多进程设计使"隔离与并行"成为自然结果，代价是凭证与配置分散在多份 `.env`/`config.yaml` 中。
- **生态依赖的位置不同**：OryxOS 把扩展能力押在 MCP 生态与社区接力上；Hermes 把部分高价值能力（Tool Gateway、Modal）放在 Nous Portal 付费订阅上。**两者都存在外部依赖，区别只在依赖的对象**（社区 vs 商业服务）。
- **共同未覆盖处**：两边的公开材料均未见"租户级 RBAC + SSO + 租户维度审计报表"的完整组合。这是本报告基于**可获取文档**的观察，不等于两者内部不存在（§6 已声明该局限）。

---

### 0.3 文档版本与比对口径（重要）

**本节回答"这份对比到底比的是哪个版本"**——因为两家文档的性质不同，版本口径直接决定结论的有效性。

| 项 | OryxOS | Hermes Agent |
|----|--------|--------------|
| **比对版本** | 本地文档当前版本，含 **014 / 015 / 017 / 025 / 026 / 028 节**演进修订 | **v0.20.0**（官方标签 `v2026.8.3`），**发布日期 2026-08-03**，代号 **"The Herald Release"** |
| **文档形态** | **规划/设计文档**：`oryxos.md` 项目篇、`DemandAnalysis.md` 需求（What）、`TechnicalSolution.md` 技术方案（How）、`IndustryResearch.md` 业界调研、`AiProgrammingGuide.md` 实施指南 | **已交付功能的用户手册**（社区中文文档站，上游为 `NousResearch/hermes-agent` 的 `website/docs/`） |
| **版本依据** | 文档内嵌的节号演进标注（如"015 起""025 演进""第 28 节补齐"） | 侧边栏「版本更新」以 v0.20.0 为最新，该页载明"官方标签 `v2026.8.3`，发布日期 2026 年 8 月 3 日" |
| **版本识别说明** | 无独立版本号；核心阶段目标产物为 **OryxOS 1.0** | 功能文档页显示 `DOCS / CURRENT` **不带版本号**，版本号须由发布说明页与侧边栏版本目录确认 |
| **发布节奏参考** | 核心阶段 **4 周 × 3 小时 = 12 小时**（教学/最小内核节奏） | v0.19.0 → v0.20.0 一个周期内：**约 3,650 次提交、1,400 个合并 PR、5,200 个文件变更、约 559,000 行新增、1,200 个 issue 关闭、650+ 位贡献者** |
| **来源** | `docs/oryxos-docs/` | [v0.20.0 发布说明](https://hermesagent.org.cn/docs/releases/v0-20-0)、[安装](https://hermesagent.org.cn/docs/getting-started/installation) 及各功能页 |

> **口径提示**：OryxOS 文档写的是"要做什么、分几期做"，Hermes 文档写的是"已经有什么、怎么用"。因此本报告在涉及 OryxOS 未交付项时一律显式标注「核心阶段 / 扩展阶段 / 社区共建」，**不做"规划 vs 已交付"的等同比较**。

> **⚠️ 版本敏感点：本报告在 v0.20.0 口径下修正的三处判断**（初版报告基于未带版本信息的功能页，遗漏了发布说明中的内容）：
> 1. **A2A 已落地**：v0.20.0 新增 **A2A v1.0 插件**——"让 Hermes 能够发现、调用或接受其他兼容 Agent 的调用"。初版"Hermes 文档未见 A2A"的判断就此作废（见 §2.1、§3.1、§3.3）。
> 2. **审批与治理控制已存在**：v0.20.0 新增**智能审批**（从历史记录生成 allowlist 建议、可配置策略、连续拒绝熔断）与 **HMAC 签名的 Outbound Webhook**。初版将 Hermes 治理一概描述为"钩子制、需自行接审计"过于绝对（见 §3.2、§4.2）。
> 3. **默认循环上限已变**：v0.20.0 将**默认工具调用轮数从 90 提高到 500**。初版据环境变量页写的"默认 90"是 v0.20.0 **之前**的口径（见 §2.1、§7）。
>
> 另需注意 v0.20.0 的两条**分发渠道变更**：**Node.js 26** 成为安装器/修复/升级流程的目标运行时；**Homebrew 与 pip / PyPI wheel 渠道已退出**，官方当前支持 **shell 安装器 / Docker / Nix**。这使初版"pip/uv 安装"的表述需要按版本区分（见 §3.1）。

---

## 1. 产品定位与设计哲学

### 1.1 OryxOS 的自我定位

OryxOS 把自己定义为**企业私有部署的 Agent 操作系统（Agent Harness OS）**：

> "你用一句自然语言发布一个任务 → 底座把它拆解 → 组织一支 Agent 团队 → 多个 Agent 分工协作 → 交付一个结果。让每一家公司，都能用自然语言跑起来自己的 Agent。"
> —— `docs/oryxos-docs/oryxos.md`

其北极星公式是：**自然语言(md) + Memory + Tool + MCP(Connector) + Skill + 知识库 + Notify = 一个 Agent**；一个目录定义一个 Agent，一个底座运行一群 Agent，私有部署，数据不出域。

OryxOS 文档中最关键的一条概念切分是 **agent runtime ≠ Agent OS**：

> "runtime 让一个 Agent 跑起来，Agent Harness OS 让一群 Agent 被运行和管理起来。OryxOS 是后者。"
> —— `docs/oryxos-docs/oryxos.md`

它明确承认与业界项目的关系是**同类不同定位**：

> "三者都是 Agent OS，OpenClaw 偏个人、Hermes 偏个人到小团队，OryxOS 直接定位严监管企业场景。"
> —— `docs/oryxos-docs/IndustryResearch.md`

**OryxOS 的设计原则**（`oryxos.md`）：底座优先于 Agent、自实现核心可控优先、配置即 Agent、对接开放标准（MCP/A2A/Agent Skills 形态）、无状态实例状态外置、安全是地基不是补丁、分阶段克制。

### 1.2 Hermes Agent 的定位

Hermes Agent 由 NousResearch 维护，文档自我描述为一个"强大的自主助手"：

> "Hermes Agent 拥有一系列丰富的功能，远超基础聊天能力。从持久化记忆和文件感知上下文，到浏览器自动化和语音对话，这些功能协同工作，使 Hermes 成为一个强大的自主助手。"
> —— [功能概览](https://hermesagent.org.cn/docs/user-guide/features/overview)

Hermes 的定位特征是**"个人助手 → 团队协作 → 企业平台"三连跳**：

- **个人层**：CLI 交互（`hermes chat`）、桌面版、个人记忆（`MEMORY.md`/`USER.md`）、皮肤与主题、人格 `SOUL.md`；
- **团队层**：消息网关（Telegram/Discord/Slack/飞书/企微/钉钉/微信/邮件…）、Kanban 多代理协作板、profile 隔离、`hermes profile` 团队共享；
- **企业层**：OpenAI 兼容 API Server、Nous Portal 团队账号、Tool Gateway 订阅、enterprise control-plane（本报告第三章"证据边界"处说明该条为仓库提交标题，非文档页内容）。

关键差异：Hermes 文档里**没有**"Agent OS"这个自我标签，也**没有**"一个目录 = 一个 Agent"这种统一底座叙事；它的组织单元是 **profile（配置文件/隔离实例）** 与 **skill（能力文档）**。

> 来源：[配置文件命令参考](https://hermesagent.org.cn/docs/reference/profile-commands)、[技能系统](https://hermesagent.org.cn/docs/user-guide/features/skills)

### 1.3 定位对照表

| 维度 | OryxOS | Hermes Agent |
|------|--------|--------------|
| 自我标签 | 企业级 Agent OS / Agent Harness OS | 自主助手（autonomous assistant） |
| 组织单元 | `.oryxos/agents/<name>/` 目录（= 一个 Agent） | `profile`（= 一套隔离配置/实例）+ `skill`（= 能力文档） |
| 主入口 | REST API（Web Service） | CLI / 消息平台 / Web Dashboard |
| 目标用户 | 严监管企业（银行/政府/电信/能源/医疗）的业务方与 IT 部门 | 个人开发者 → 小团队 → 企业（Nous Portal 付费用户） |
| 与对方关系 | 同类不同定位，生态互补 | 文档未提 OryxOS（单向认知） |
| 实现语言 | Java（JDK 21） | Python |
| 许可 | Apache 2.0 | 文档站显示"OPEN SOURCE"（具体协议以仓库为准） |

---

## 2. 功能对比

### 2.1 功能总览对照

| 功能区 | OryxOS | Hermes Agent |
|--------|--------|--------------|
| **Agent 定义** | 一个目录 = 一个 Agent：`AGENT.md`（frontmatter=Profile，正文=指令）+ `skills/` 软连接 + `scripts/` + `REFERENCE.md` | `profile` + `SOUL.md`（人格）+ 上下文文件（`.hermes.md`/`AGENTS.md`/`CLAUDE.md`/`SOUL.md`/`.cursorrules`）+ skills |
| **LLM 接入** | Provider 抽象（基于 Spring AI Alibaba），provider name → ChatModel **显式映射**；核心阶段**无 fallback** | 20+ provider（含 OpenRouter/Nous/Anthropic/DeepSeek/Kimi/GLM/MiniMax/阿里云/AI Gateway…）+ provider routing（按价格/吞吐/延迟排序）+ fallback provider + 凭证池 |
| **Agent 循环** | 自实现 ReAct loop，约数十行 Java，`MAX_ITERATIONS` 默认 10 | 内置 agent loop；`agent.max_turns` 环境变量页默认 90，**v0.20.0 已将默认工具调用轮数提高到 500**；子 Agent `max_iterations` 默认 50 |
| **工具并行** | 核心阶段**不并行**（一次响应内多个 Tool 调用按顺序执行） | 支持（hooks 文档明确"如果模型并行调用 3 个工具，`pre_tool_call` 会触发 3 次"） |
| **流式响应** | 核心阶段**不做**，扩展阶段 SSE | 支持（流式 TUI、流式超时参数、Web Dashboard 流式 Markdown） |
| **记忆** | 三层设计：会话 + 长期（`MEMORY.md`，核心/归档两分区）+ 情景（扩展）；统一 `MemoryService` 门面；三档后端（Markdown / SQLite / 自托管 Mem0） | 两层内置：`MEMORY.md`（2200 字符，约 800 token）+ `USER.md`（1375 字符，约 500 token）；**冻结快照**注入；+ `session_search`（SQLite FTS5 + LLM 摘要）；+ **8 个外部记忆提供者**（Honcho/OpenViking/Mem0/Hindsight/Holographic/RetainDB/ByteRover/Supermemory） |
| **记忆安全** | 文档未提及记忆内容安全扫描 | **记忆写入前扫描注入与数据外泄模式**（prompt injection、凭证外泄、SSH 后门、不可见 Unicode 字符直接拦截） |
| **知识库** | `oryxos-knowledge` 模块：解析/切分/向量化索引流水线、双路召回 + RRF 融合、`retrieve_knowledge` 工具 | 文档未见独立知识库模块（可由 MCP server 或记忆提供者承担） |
| **技能体系** | 公共 Skill 库 `.oryxos/skills/`，Agent 用相对软连接绑定；三层渐进式披露；核心阶段**无技能市场**（市场为社区共建） | `~/.hermes/skills/`（唯一真相源）+ **Skill Hub**（official / skills.sh / well-known / GitHub / ClawHub / LobeHub / claude-marketplace 七个来源）+ 信任等级 + 安全扫描 + Agent 自建技能（`skill_manage`） |
| **工具接入** | 三档：零代码（Agent 目录 + 复用 MCP）→ 轻代码（自写 MCP server）→ 重代码（Java `@Tool` Bean） | 内置工具集开箱即用 + MCP（stdio / HTTP）+ 插件（通用插件/记忆提供者/上下文引擎） |
| **内置工具广度** | 核心阶段 9 个：`read_file`/`write_file`/`list_dir`/`shell`/`http_get`/`http_post`/`save_memory`/`recall_memory`/`notify` | 极广：web、terminal/process、browser、media（vision/image_gen/TTS）、编排（todo/clarify/execute_code/delegate_task）、memory/session_search、cronjob/send_message、homeassistant、rl_*、LSP、Curator… |
| **浏览器自动化** | 未提及（可通过 MCP 接） | 原生多后端：Browserbase / Browser Use 云 + 本地 Chrome(CDP) / 本地 Chromium |
| **多模态** | 未提及 | 视觉分析、剪贴板图像粘贴、图像生成（FLUX 2 Pro + 2× 超分）、语音模式、TTS（5 家供应商） |
| **多渠道接入** | 核心阶段 CLI；已实现飞书/企微/钉钉三个入站渠道模块（长连接，免公网回调）；Slack 等在扩展阶段 | 20+ 平台：Telegram/Discord/Slack/WhatsApp/Signal/SMS/Email/Matrix/Mattermost/BlueBubbles/飞书/企微/钉钉/微信/Home Assistant 等 |
| **出站通知** | `NotifyTools`：`notify(content, channel)`，适配器接口 + Webhook 实现（企微/飞书/钉钉群机器人），扩展阶段 SMTP | `send_message` 工具 + 各平台 `HOME_CHANNEL` 默认推送目标 + cron 交付目标 |
| **定时任务** | `AgentScheduler`（Spring `ThreadPoolTaskScheduler` + `CronTrigger`），cron 定义在 `AGENT.md` frontmatter；`scheduled_tasks`/`task_executions` 两表持久化；管理端点可查历史/立即执行/启停 | `hermes cron`（list/create/edit/pause/resume/run/remove/status/tick），可附加技能、交付到任意平台；Web Dashboard 提供 Cron 页面与 REST API；也支持脚本型 pre-run |
| **多 Agent 协作** | 同实例多 Agent 并存（多个目录同时可用）；扩展阶段 A2A、跨节点协作 | `delegate_task`（**并行上限 3**、**深度上限 2**、子 Agent 隔离子会话与独立终端、禁用递归委派）；`execute_code`（Python RPC 编排）；**Kanban 多代理协作板**（SQLite 持久任务板、worker 认领/心跳/阻塞/恢复、`hermes kanban swarm` 支持 root/worker/verifier/synthesizer 拓扑）；**A2A v1.0 插件**（v0.20.0 新增：可发现、调用或接受其他兼容 Agent 的调用）；**HMAC 签名 Outbound Webhook**（把会话/轮次/工具生命周期事件推送到外部系统） |
| **对外服务** | REST API 核心 10 端点 + OpenAPI/Swagger UI；扩展阶段 Agent CRUD、SSE、Webhook、Metrics | OpenAI 兼容 API Server（默认 `127.0.0.1:8642`，Bearer Token + CORS 白名单）+ Web Dashboard 自身 REST API + Webhook 平台 + ACP（编辑器集成） |
| **IDE 集成** | 文档未提及 | ACP（VS Code / Zed / JetBrains），工具活动、文件 diff、终端命令在编辑器内渲染 |
| **管理界面** | 核心阶段无（Web 仪表板为扩展功能） | **Web Dashboard**（`hermes dashboard`，默认 127.0.0.1:9119）：Status / Chat(内嵌 TUI) / Config(150+ 字段表单) / API Keys / Sessions(全文搜索) / Logs / Analytics(成本) / Cron / Skills |
| **数据持久化** | SQLite + Spring Data JPA（5 张表）+ Flyway 双轨迁移（PostgreSQL 为部署选项）+ 文件系统 | `~/.hermes/state.db`(SQLite, FTS5)、`kanban.db`、`config.yaml`、`.env`、`memories/`、`skills/` |
| **可观测性** | 结构化日志 + SQLite 审计表（day one 写入）+ 扩展阶段 Micrometer/Prometheus/Grafana | 三类日志文件（agent/errors/gateway）+ `hermes logs` 过滤 + `hermes insights` token/成本分析 + Dashboard Analytics 页（token、缓存命中率、按模型成本） |

### 2.2 功能差异的四个要点

**要点一：Hermes 的功能宽度显著更大，但宽度来自"工具集"而非"底座治理"。**

Hermes 的工具注册表覆盖网络、终端与文件、浏览器、媒体、Agent 编排、记忆与召回、自动化与交付、集成八大类（[工具与工具集](https://hermesagent.org.cn/docs/user-guide/features/tools)）。OryxOS 核心阶段只做 9 个内置工具，理由是"覆盖让 Agent 能读写文件、跑命令、调外部 API、记事、往外推通知的最短链路"（`TechnicalSolution.md` 6.2）。这不是能力差距，而是**设计取舍**：OryxOS 选择把能力扩展权交给 MCP 生态，Hermes 选择把常用能力内置。

**要点二：两者对"技能"的开放程度截然相反，这决定了工具能力的供应链形态。**

Hermes 的技能系统是**开放式供应链**：七个外部来源、`--force` 可覆盖非危险策略阻断、Agent 可以自己创建/修改/删除技能（`skill_manage`，"程序性记忆"）。它有信任分级（builtin / official / trusted / community）和安全扫描，但默认姿态是"可用优先"。

OryxOS 明确走相反路线：

> "**Skill 和 Tool 来源受控，不做无约束的公开市场。** 企业内的 Skill 和 Tool 要经过注册、审核、签名、版本管理，来源可追溯。"
> —— `docs/oryxos-docs/IndustryResearch.md` 5.6

**要点三：多 Agent 是两种完全不同的东西。**

| | OryxOS | Hermes |
|---|---|---|
| 单元 | Agent 目录（Profile） | profile（隔离实例）/ Kanban worker |
| 共享 | 共享同一进程、同一套 Provider/Memory/Tool/Sandbox/审计 | 各 worker 有**自己的身份、模型、运行环境**，共享一张 SQLite 任务板 |
| 协作方式 | 同实例并存（进程内多 Profile）+ 扩展阶段 A2A 跨节点协作 | 任务板认领/心跳/阻塞/完成 + delegate_task 父子摘要回传 + **A2A v1.0 插件**（v0.20.0 已交付） |
| 治理面 | 统一（一个底座管所有 Agent） | 分布式（每个 worker 独立配置，板子只管任务） |

OryxOS 的 Kanban 对应物是"扩展阶段的跨节点协作"愿景；Hermes 的"统一底座"对应物是"HERMES_HOME 多实例 + profile 隔离"。**两者互为镜像的空白**。

> **注意 A2A 上的口径变化**：OryxOS 把 A2A 列为**阶段三愿景**（当前尚未实现），Hermes 已在 **v0.20.0 以插件形式交付 A2A v1.0**。所以准确表述不是"Hermes 没有 A2A"，而是——**Hermes 先有了协议接入能力，但缺的仍是"跨节点的底座级调度"**（Kanban 仍是同机多进程共享本地 SQLite）；OryxOS 缺的是协议实现，但它的路线图把"单节点运行时"与"跨节点通信底座"显式拆成两件分开演进的事（`IndustryResearch.md` 6.5）。

**要点四：Hermes 已交付 OryxOS 仍在规划的功能，但方向和形态不同。**

| OryxOS 扩展阶段规划 | Hermes 当前状态 |
|---|---|
| Web 仪表板（Profile 管理/Session/审计查询） | 已有，但定位是**单机本地管理台**（无认证，默认仅 127.0.0.1） |
| SSE 流式响应 | 已有 |
| API Key + JWT | 已有 API Server 的 Bearer Token（单 token，非用户体系） |
| Provider fallback / circuit breaker / hedge racing | 已有 fallback provider + provider routing + 凭证池 |
| 向量检索 / 情景记忆 | 已有 8 个外部记忆提供者（语义/知识图谱/自动抽取） |
| 渠道扩展（企微/飞书/钉钉/Slack） | 已有 20+ 平台 |
| MCP Server 暴露 | 已有（`hermes mcp serve`） |

**但反向也成立**：Hermes 文档中未见 OryxOS 已有或已设计的这几项——
- 审计表（`tool_invocations` / `llm_calls`）作为**结构性数据**落库并从 day one 写入（Hermes 是钩子 + 日志的观察式路径）；
- `Sandbox` 接口先行 + 按信号驱动升级到容器/microVM 的**显式隔离演进路径**（Hermes 是六种后端即选即用，无同类演进抽象）；
- Profile 级 **Tool Policy**（allow/deny）作为**底座内建**的治理能力（Hermes 的智能审批 allowlist 更偏"交互式危险命令审批"而非"Agent 可用工具集的静态授权"）；
- 三级租户模型（组织/部门/项目）+ RBAC 粒度到 Agent/Tool/Skill 的规划；
- 跨节点 Agent 协作的底座级路线图（Hermes 已有 A2A 协议插件，但未见跨节点调度）。

---

## 3. 技术架构对比

### 3.1 架构基线对照

| 维度 | OryxOS | Hermes Agent |
|------|--------|--------------|
| 语言/运行时 | Java（JDK 21）+ Spring Boot 3.x | Python 3.11+（`hermes dump` 显示 Python 版本行与 OpenAI SDK 版本）；v0.20.0 起 **Node.js 26** 成为安装器/升级流程的目标运行时 |
| 进程模型 | Spring MVC 单体 + Java 21 virtual thread（同步阻塞，单机撑千级并发）；三入口（CLI / Web Service / Scheduler）汇入同一 `AgentService` | CLI 进程 / 网关守护进程（systemd 或 launchd 用户服务）/ Dashboard(FastAPI+Uvicorn) / API Server；worker 为**独立 OS 进程** |
| 分发形态 | 单可执行 fat JAR（`java -jar`），扩展阶段 GraalVM Native Image | **v0.20.0 起：shell 安装器 / Docker / Nix**（`install.sh` 或 `install.ps1`）；**Homebrew 与 pip / PyPI wheel 渠道已退出**；可选 extras（web/pty/mcp/acp/messaging/voice/termux…）；Android/Termux 有独立路径 |
| Agent 循环实现 | **自实现 ReActLoop**，Spring AI 仅作协议适配 + schema 生成，显式禁用其自动 tool 执行（决策二，防 tool 被调两次） | 内置 agent loop（`agent.max_turns`，**v0.20.0 默认轮数由 90 提高到 500**），工具分发 `handle_function_call()`，可被插件钩子插桩 |
| LLM 抽象 | Spring AI `ChatClient` 上包 `ProviderService`；provider name → ChatModel **显式映射**（不用类型扫描） | 多 provider 抽象 + OpenRouter 聚合 + 自定义 OpenAI 兼容端点（vLLM/SGLang）+ OAuth 提供商 |
| 工具抽象 | `OryxTool`（`getName`/`getDescription`/`getInputSchema`/`execute` → `ToolResult`），统一内置/注解/MCP 三种来源 | 工具集（toolsets）按平台启停；MCP 工具加 `mcp_<server>_<tool>` 前缀注册；插件钩子可拦截 |
| 上下文组装 | `PromptBuilder` 四段拼接：system prompt（`AGENT.md` 正文 + Bootstrap + 当前时间）→ Memory 注入 → 对话历史（`maxHistoryTurns` 截断，默认 20）→ 可用 Tool 列表 | 上下文文件自动发现 + skills 三级渐进披露（`skills_list` ~3k token → `skill_view` → 引用文件）+ 记忆冻结快照 + **compression**（阈值可配，`summary_model`/`summary_provider` 可指定） |
| 上下文压缩 | 核心阶段**简单截断**（裁字符串 / SQL `LIMIT`），扩展阶段才做总结压缩 | **已有且 v0.20.0 加强**：`compression.threshold`（可按模型或绝对 token 数配置）+ 独立摘要模型/provider + 渐进式工具结果裁剪 + 逐轮微压缩 + 保留最近若干条用户消息 |
| 审批与授权 | 核心阶段靠 `application.yaml` 白名单（文件/命令/域名/SMTP）+ 无认证（内网假设）；扩展阶段 Tool Policy、RBAC、API Key+JWT | **v0.20.0 智能审批**：从历史记录生成 allowlist 建议、可配置策略、连续拒绝熔断；容器重定向命令也纳入审批门；`--yolo` 可跳过；per-platform 用户白名单 + `GATEWAY_ALLOWED_USERS` |
| 状态管理 | **实例无状态、状态外置**为设计前提；核心阶段状态落本地 SQLite + 文件；扩展阶段 Redis(会话) / PostgreSQL(记忆+Skill) / 对象存储(审计+大文件) | 状态在 `~/.hermes/` 下：`state.db`、`kanban.db`、`memories/`、`skills/`、`config.yaml`、`.env`；实例即状态持有者 |
| 持久化迁移 | Flyway `db/migration/{vendor}/` 双轨，SQLite 零配置默认 / PostgreSQL 按 url 自动识别 | 未见数据库迁移框架描述；`hermes config migrate` 针对配置项而非 schema；v0.20.0 加入 `hermes sessions optimize` 与面向中日韩文本的双字索引 |
| 隔离实现 | `Sandbox` 接口 + `WhitelistSandbox`（应用层 Path/Pattern/命令/域名/SMTP 端点白名单）；扩展阶段容器 → microVM，**接口不变只新增实现类** | `terminal` 六种后端：local / docker / ssh / singularity / modal / daytona；容器加固：只读根文件系统、移除全部 Linux capabilities、禁提权、PID 上限 256、完整命名空间隔离；v0.20.0 补强 SSRF（DNS 固定）、凭据读取范围限制、压缩边界脱敏 |
| 凭证管理 | `ConfigLoader` 统一加载，`${ENV_VAR}` 占位不落明文；扩展阶段加密存储、轮转、KMS/Vault | 22 个 provider/tool 密钥存 `.env`，Web Dashboard 可读写；凭证池支持多密钥轮换与冷却重置；API Server Bearer Token；v0.20.0 收紧凭据读取范围 |
| 动态发现 | `WorkspaceWatcher`（JDK `WatchService`）+ `AgentLifecycleService.register()`，API 上传与手工丢目录**同一段注册代码**，免重启上线 | MCP `notifications/tools/list_changed` 动态刷新；`/reload-mcp`；技能热切换（下一会话生效）；`/reload` 重载 `.env`；v0.20.0 支持 MCP Server 按缓存 schema 延迟启动 |
| 分布式就绪度 | 架构上预留（无状态实例 + 状态外置 + `ReentrantLock` 明确标注"非分布式锁"）；阶段三引入 A2A 通信底座，扩展阶段 Nacos/ETCD 协调 | **已有 A2A v1.0 插件（协议层接入）**，但**未见跨节点调度**；Kanban worker 是**同机多进程**（共享本地 SQLite 板） |

### 3.2 三个本质性的架构分歧

#### 分歧一：单体统一底座 vs 多进程隔离实例

OryxOS 的架构核心是"**所有能力收敛到一个引擎、一套存储、一个进程内**"，理由是符合"单二进制、装好就跑"的定位，外部依赖（LLM API、MCP server）都在应用边界之外（`TechnicalSolution.md` 第 2 章）。它的多 Agent 是**进程内多 Profile 对象**，"每个 Agent 是内存里的 Profile 对象加 Session 列表占用极少"（第 14 章）。

Hermes 的多 Agent 走的是**进程隔离 + 共享任务板**：Kanban 的"每个 worker 都是独立 OS 进程，有自己的身份、模型和运行环境"（[Kanban 多代理协作板](https://hermesagent.org.cn/docs/user-guide/features/kanban)）。

两者各有代价：

| | 单体（OryxOS） | 多进程（Hermes） |
|---|---|---|
| 资源开销 | 低（Profile 是内存对象） | 高（每 worker 一个进程 + 可能一个终端会话） |
| 隔离强度 | 弱（同进程，靠应用层白名单） | 强（进程/容器边界） |
| 统一治理 | 天然（一个审计表、一套 Policy） | 难（每个 worker 独立配置，需额外控制平面） |
| 横向扩展 | 需先做状态外置（OryxOS 明确列为阶段二） | 天然多进程，但共享本地 SQLite 板反而成了单点 |

**这是一道"先易后难"与"先难后易"的分岔**：OryxOS 先吃统一治理的红利，把分布式留成规划；Hermes 先吃隔离与并行的红利，把统一治理留成空白（外部报道称其正在补 enterprise control-plane，见 §6 证据边界）。

#### 分歧二：Adapter 式沙箱（可演进）vs Backend 式沙箱（可即用）

OryxOS 的 `Sandbox` 设计有一条非常克制的原则：

> "先抽象出一个 `Sandbox` 接口，表达'在受控环境里执行一个动作'这个意图，不携带任何一档实现特有的概念（不出现'容器镜像''VM 配置'字样）……用最重的 microVM 实现去反向套这个签名，也应该能干净套入，这是校验接口是否中立的办法。"
> —— `docs/oryxos-docs/TechnicalSolution.md` 6.7

同时它对核心阶段的强度有**清醒的自我否定**：

> "应用层白名单是'劝阻级'防线，防的是模型犯傻误操作，防不住蓄意绕过，核心阶段不建议用它跑完全不可信的代码或对外做多租户。"
> —— 同上，6.7 要点一

Hermes 是**当下可用的强隔离**：Docker 后端只读根文件系统、移除全部 capabilities、禁提权、PID 限制、完整命名空间隔离（[工具与工具集](https://hermesagent.org.cn/docs/user-guide/features/tools)），并额外提供 SSH（"推荐用于安全性——Agent 无法修改其自身代码"）、Singularity/HPC、Modal、Daytona 云沙箱。

**结论**：两者在这一轴上的取舍方向相反——Hermes 在"今天就能安全地跑不可信代码"上已经可用（容器后端开箱即用）；OryxOS 在"未来平滑演进到 microVM 而不改调用方"上做了更明确的接口设计（`Sandbox` 签名刻意中立，用 microVM 反向套签名验证）。后者同时也是可验证的工程判断：OryxOS 的技术方案专门论证了 `SecurityManager` 在 JDK 17 起废弃、JDK 21 不可用，因此不用它。

#### 分歧三：审计是"数据地基" vs 审计是"钩子扩展"

OryxOS 把审计做成**架构内的表结构**：

> "`tool_invocations` 和 `llm_calls` 在核心阶段就做写入（不一定做查询接口），因为'可审计'是 OryxOS 的差异化卖点之一，审计数据的地基应该 day one 就立起来，纯靠日志后期要做审计还得反解析返工。"
> —— `docs/oryxos-docs/TechnicalSolution.md` 9.2

并且 Sandbox 违规**复用既有审计路径**（抛异常 → `tool_invocations` 记 `success=false` + `error_message`），不新增审计逻辑。

Hermes 提供的是**两层组合的审计与治理**：一层是 `pre_tool_call` / `post_tool_call` / `pre_llm_call` / `post_llm_call` / `on_session_start` / `on_session_end` 六个插件钩子，文档明示用途包括"日志记录、审计追踪、工具调用计数、阻止危险操作（打印警告）、速率限制"（[事件钩子](https://hermesagent.org.cn/docs/user-guide/features/hooks)）；另一层是 v0.20.0 引入的**智能审批**（历史记录生成 allowlist 建议、可配置策略、连续拒绝熔断）与 **HMAC 签名 Outbound Webhook**（把会话/轮次/工具事件推给外部系统，适合接 CI、监控面板或内部工作流）。

它的会话可以用 FTS5 全文搜索、日志可分级过滤、成本可按模型统计——**观察能力其实相当完整**。但它**没有把"谁调了什么工具、花了多少 token、依据哪条策略放行"固化为一等公民的结构化表 + 强制写入**：审计完整度取决于你是否写了钩子、是否配了 webhook。OryxOS 则相反：表结构与写入路径是架构的一部分，查询接口可以后补。

**技术含义**：这体现的是两种审计集成路径——OryxOS 选择"写入默认发生，缺的是查询与报表"；Hermes 选择"观察原料齐全（钩子/审批/签名事件/全文检索），缺的是开箱的合规视图"。**这是路径差异，不是能力高下**：前者省掉"要不要记"的决策，后者把"记什么、怎么用"留给集成方。

### 3.3 两者共同承认的空白

值得指出的是，两者在文档层面**都没有**声称已经具备完整企业治理：

- OryxOS 反复声明"核心阶段是地基，企业级治理是终局"，多租户/SSO/完整审计/Tool 治理都在扩展阶段；
- Hermes **v0.20.0** 文档站的功能概览、Web Dashboard 页与环境变量页均**未见**多租户 RBAC、SSO/OIDC/SAML 集成、租户级审计报表的描述；身份控制仍是"per-platform 用户白名单 + API Server Bearer Token"这一层级；Dashboard 甚至明确"**仪表板本身没有身份验证机制**"，绑定 `0.0.0.0` 时"网络上的任何人都可以查看和修改你的凭据"。v0.20.0 的智能审批解决的是"危险命令要不要放行"，**不是**"哪个租户/角色能用哪些 Agent 与工具"。

也就是说，在**公开可获取的材料**范围内，两家的治理层都还缺少"租户级 RBAC + SSO + 租户维度审计报表"这一完整组合：OryxOS 在文档中把它整体列为扩展阶段（仓库约束显示分布式底座已在推进），Hermes 在 v0.20.0 文档中未见对应条目，其身份控制止于 per-platform 白名单与单一 Bearer Token。

**这是基于可获取材料的观察，不等于两者内部不存在**——尤其对 Hermes，本报告无法访问其仓库，无法排除该能力已在代码中落地。另需注意这个"缺失"的性质也有区别：OryxOS 是**把它写进了规划**（承认应该有），Hermes 是**未在文档中提出该需求**（可能另有一层未公开的设计）。把两者都记作"空白"是简化，读者宜按各自的证据等级理解。

---

## 4. 设计意图与适用约束（原"企业价值"）

### 4.1 两家自设的评价标准（先声明来源属性）

OryxOS 的调研文档为"企业为什么需要私有底座"立了四条标准（`IndustryResearch.md` 3.3）：

1. 核心业务数据不能出企业；
2. 系统必须完全可审计；
3. 任何新组件都要过现有安全合规流程；
4. 技术栈要跟现有体系对齐。

**必须声明**：这四条是 **OryxOS 为论证自身定位而设的标准**，`IndustryResearch.md` 与 `oryxos.md` 中反复出现的"Agent OS 五大能力""Agent OS vs agent runtime"同样是 OryxOS 的概念装置。Hermes 从未以这四条为目标，其文档强调的是能力完整度、多渠道触达与开箱即用。

因此本报告**不采用评分或推荐**——用一方自设的尺子给双方打分，结论会带上同义反复（"Hermes 在严监管标准上失分"几乎等价于"Hermes 不是为严监管设计的"）。下面改为陈述两组**可验证的成对约束**，由读者按自己的约束条件判断。

### 4.2 两组成对约束（对称呈现）

#### 约束对一：执行边界强 ⟷ 治理视图统一（二者不可同时最大化）

| | OryxOS | Hermes Agent |
|---|---|---|
| 隔离机制 | 应用层白名单（`SandboxChecker`：路径/可执行文件/域名/SMTP 端点）+ 独立工具策略层（`tool_policy_rules`：全局/Agent 级 allow-deny，管理台可热更新；策略拒绝照写审计并标 `blocked_by='policy'`）〔宪法 020，仓库约束文档〕 | 6 种终端后端；Docker 容器加固（只读根 FS、移除全部 capabilities、禁提权、PID 上限 256、命名空间隔离）〔v0.20.0 文档〕 |
| 边界性质 | **逻辑边界**：白名单是"劝阻级"防线（OryxOS 文档自述"防不住蓄意绕过"），解释器一旦列入白名单即等于授予宿主进程权限 | **物理边界**：进程/容器级隔离，可跑更不可信的代码 |
| 治理视图 | **单点**：策略、审计在同一库，一处改全 Agent 生效 | **分散**：各 worker 独立配置；治理靠插件钩子 + 智能审批 + 签名 Webhook 自行拼装 |
| 结构性代价 | 同进程意味着一次白名单配置失误的影响面是**整个底座** | 每 worker 独立 `.env`/`config.yaml` 意味着凭证与策略的**一致性靠人工维持** |

**可验证推论**：两者在这条轴上是**反向取舍，不存在同时占优**。要"一处改策略、全量生效"就得接受逻辑边界；要"进程级隔离跑不可信代码"就得接受配置分散。

#### 约束对二：域内闭环 ⟷ 能力外延（能力来源不同）

| | OryxOS | Hermes Agent |
|---|---|---|
| 能力来源 | 自建机制（审计、策略、知识库、通知抽象）+ 社区 MCP 生态 | 自建工具集 + 7 个 Skill Hub 来源 + 云服务（Tool Gateway→Firecrawl/FAL/OpenAI TTS/Browser Use；Modal/Daytona 沙箱） |
| 默认数据流向 | 全部本地；扩展阶段即外部 LLM API 也在企业自有 key 下 | 可全本地，但**部分高价值能力默认经 Nous 云**，且 Tool Gateway 属付费订阅 |
| 结构性代价 | 能力面窄，缺口需自己用 MCP 补；兑现依赖社区接力（OryxOS 文档自认"社区接力的不确定性"风险） | 能力面宽，但引入**外部服务依赖 + 商业订阅绑定 + 开放技能供应链**三类风险 |

**可验证推论**：两者都存在外部依赖，**区别只在依赖对象**（社区生态 vs 商业云服务）。把前者说成"可控"、后者说成"锁定"，是把依赖对象当成了依赖强度——这不成立。

#### 版本成熟度：两边的可比性有限

| | OryxOS | Hermes Agent |
|---|---|---|
| 版本标识 | 文档无独立版本号；节号演进至 042；仓库有 Helm Chart、集群档、Docker/Release 流水线 | `v0.20.0`（`v2026.8.3`），有发布说明与版本侧边栏 |
| 能力分布 | 部分能力以**仓库宪法约束**形式存在（已实现）；部分在 `DemandAnalysis.md` 第 6 章仍为扩展阶段规划 | 功能以**用户文档**形式呈现，v0.20.0 已交付 |
| 可比性 | **低**：一边是"设计文档 + 仓库约束"的混合体，难以逐条判断落地状态 | **高**：有版本号、发布说明、贡献者规模 |

> 本报告据工作区根目录 `CLAUDE.md`（OryxOS 项目宪法）补充了若干"已实现"级证据（工具策略层、凭证加密、多副本集群、Helm 交付）。**该文件是仓库工程约束文档，不是已发布的产品文档**——它比 `docs/oryxos-docs/` 更接近实现状态，但仍不等于第三方验证。

### 4.3 两处把厂商叙事与实现事实分开看的地方

前两版报告在三处把**厂商叙事当成了分析结论**，此处显式分开：

| 议题 | 厂商叙事（需标注来源） | 可验证的实现事实 |
|---|---|---|
| "企业级治理" | OryxOS：核心阶段是地基、治理是终局，并已写入路线图；Hermes 未在文档中提出同类主张 | OryxOS 侧：审计表、工具策略层、凭证加密、集群 CAS 已作为工程约束存在；Hermes 侧：智能审批、签名 Webhook、per-platform 白名单已交付，**双方公开材料均未见"租户级 RBAC + SSO + 租户维度审计报表"的完整组合** |
| "安全是 day one 的设计" | OryxOS 文档自述"从第一天就把安全做进架构"（`oryxos.md`） | OryxOS：白名单 + 策略层 + 凭证加密确已落地；但核心阶段**无认证**、白名单为逻辑边界——"day one 设计"指机制就位，不等于强度达标 |
| "Hermes 偏个人/小团队" | 此判断出自 **OryxOS 的** `IndustryResearch.md` 2.1 | Hermes 自身文档呈现的是个人→团队→企业的连续能力（网关、Kanban、API Server、Team 账号）；"偏小团队"是竞品定位判断，非中立事实 |

### 4.4 OryxOS 自列风险的读法

OryxOS 文档自己列出了这些风险（`DemandAnalysis.md` 第 12 章），这份自我披露值得记录：

| 已识别风险 | 技术含义（非采购含义） |
|-----------|-----------------|
| 核心功能范围风险（4 周 12 小时极紧约束） | 核心阶段的节奏是教学/最小内核节奏 |
| Tool 执行安全风险（应用层白名单非完整 Sandbox） | 文档明言"核心阶段不建议在生产环境跑高敏感场景"——即承认逻辑边界的上限 |
| 定位被误读的风险 | 文档要求"明确说明核心阶段是地基，差异化是终局" |
| 生态关系风险 | 通过 markdown + frontmatter 目录形态与 OpenClaw/Hermes 互通 |
| 社区接力的不确定性 | 不少治理能力依赖社区贡献者推进 |

**技术读法**：这些是"设计意图与阶段目标之间已知的差距"，不是"两家的能力对比结论"。把它当作给 OryxOS 加分（"文档诚实"）或给 Hermes 减分都不成立。

### 4.5 生态接口层的实际收敛

两边在**资产格式**上是收敛的，这比能力对比更值得注意：

> "三者都采用 markdown + frontmatter 的目录形态，社区的优质 Skill 经过企业审查后理论上可以导入 OryxOS 的全局 Skill 库。"
> —— `docs/oryxos-docs/IndustryResearch.md` 5.7（**OryxOS 方判断**）

同时 OryxOS 的需求文档明确：

> "**Skills Marketplace**：社区贡献的 Skill 共享平台，兼容 agentskills.io 开放标准，跟 OpenClaw 和 Hermes Agent 兼容。"
> —— `docs/oryxos-docs/DemandAnalysis.md` 第 7 章

两者都遵循 **agentskills.io** 规范（Hermes 技能系统页明确"与 agentskills.io 开放标准兼容"；OryxOS 在术语与需求文档中同样引用该标准），也都以 **MCP** 作为工具互操作协议。这意味着**在技能与工具资产层面，两者是可迁移的**——差异主要在底座、治理与运行时。

---

## 5. 两者共同提出的 Agent 设计问题（本报告认为最有价值的部分）

前面四章都是"差异"。但把两份材料放在一起读，真正有信息量的不是谁强谁弱，而是**两家在互不通气的情况下，被同一批工程问题逼到了相似的答案上**。这些问题本身就是"什么是 agent"的可操作回答。

### 5.1 Agent 的"记忆"是上下文预算工程，不是存储问题

| | 做法 | 目的 |
|---|---|---|
| Hermes | `MEMORY.md` 硬限 **2200 字符**（≈800 token）、`USER.md` 硬限 **1375 字符**（≈500 token）；满了就要求 Agent 合并/替换条目 | 控制系统提示的固定开销 |
| OryxOS | 长期记忆分**核心区/归档区**：核心区永不截断、每轮全量注入；归档区才截断和检索 | 保证"最重要的事永远在场" |

两家的机制不同，但都在处理同一个约束：**上下文窗口是有限的，所以"记住"的本质是决定什么值得常驻**。Hermes 用字符配额把决策权交给 Agent（让它自己权衡、自己合并）；OryxOS 用分区把决策权交给调用时机（写核心还是写归档由 Agent 显式指定，系统不猜）。

**推论**：任何 Agent 的记忆设计都可以被还原为三个参数——**预算（多大）、驻留策略（什么常驻）、写入时机（何时决定）**。这比"支持几种记忆类型"更能说明一个 Agent 架构的水平。

### 5.2 上下文经济学决定了两家不约而同地对"技能"做渐进披露

Hermes 的 `skills_list()` 只暴露 `{name, description, category}`（约 3k token），正文要 `skill_view()` 才加载，引用文件再下一层。OryxOS 同样是三层：`AGENT.md` 正文常驻 → 绑定 Skill 的 name/description/路径每轮注入 → SKILL.md 正文经 `read_file` 按需取。

**两边独立收敛到同一个模式**，原因不是互相抄，而是同一个经济学约束：**要让能力可无限扩展，就必须让 token 占用不随能力数量线性增长**。这也解释了为什么两家的 `AGENT.md`/`SKILL.md` 都不做流程解析——它们只声明"什么时候该用我"，具体步骤留给模型在运行时读。

**推论**：Skill 不是"插件"或"函数"，而是**一段可被按需拉进上下文的文本指令**。它和 Tool 的根本差别在于：Tool 消耗的是执行能力，Skill 消耗的是上下文预算。

### 5.3 多 Agent 的本质是"上下文隔离 + 状态共享"，粒度可以不同

| | 隔离单位 | 共享什么 | 粒度 |
|---|---|---|---|
| Hermes `delegate_task` | 一次全新会话（子 Agent 对父对话**一无所知**，只有 `goal`+`context`，只回传最终摘要） | 无（避免污染父上下文） | 单次任务 |
| Hermes Kanban | 独立 OS 进程（各有身份/模型/环境） | 一张 SQLite 任务板（认领/心跳/阻塞/完成） | 跨会话、跨长时间 |
| OryxOS | 同进程内多个 Profile（各自 `AGENT.md`、各自 Session） | 检索结果（各自检索） | 常驻、共享底座 |

三者在实现上差别很大，但都在回答同一个问题：**当一个任务超出一个上下文窗口能承载的复杂度时，怎么把它切开**。Hermes 的答案是"切开上下文，只让摘要回流"——它甚至明确禁止子 Agent 写入共享记忆、禁止递归委派（深度上限 2），因为这些都会破坏隔离；OryxOS 的答案是"切开身份，共享底座"——Agent 之间不共享会话，但共享同一套记忆机制与策略。

**推论**：多 Agent 的收益来自**上下文隔离**（每个 Agent 只装自己需要的东西），代价是**信息在边界上必然损耗**（只有摘要能穿过）。这就解释了为什么 `delegate_task` 要求调用方把子任务所需信息**全部写进 `context`**——这不是 API 设计缺陷，而是隔离的必然结果。

### 5.4 "Agent 能干什么"由工具集与边界共同决定，且边界更关键

一个反直觉但两边的材料都支持的现象：

- Hermes 单版本内工具能力增幅极大（默认轮数 90→500、工具自恢复、grounded-citations），但它的安全讨论始终围绕**边界**：6 种终端后端、容器加固、SSRF、凭据读取范围、审批 allowlist。
- OryxOS 核心阶段只有 9 个内置工具，却把大量工程投在边界上：白名单四类动作、独立的工具策略层、凭证加密、解释器白名单的信任边界说明。

**两家的投入分布指向同一个判断**：Agent 与聊天机器人的分水岭不在"能生成什么"，而在"**能对世界做什么、以及做错了会怎样**"。工具集定义了能力上限，边界定义了可托付程度；而后者才是决定 Agent 能否进入真实环境的那一项。

**推论**：评估一个 Agent 架构，看它的沙箱与策略层比看它的工具数量更有信息量——工具数量可以靠 MCP 一夜补齐，边界设计不行。

### 5.5 三种上下文压缩策略，对应三种"什么可以丢"的判断

| | 策略 | 隐含判断 |
|---|---|---|
| OryxOS 核心阶段 | 简单截断（保留最近 `max_history_turns` 轮，默认 20） | 近期的比远期的重要 |
| Hermes 记忆 | 冻结快照（会话开始注入一次，会话中不更新，以保前缀缓存） | 稳定的前缀比实时的准确性更值钱（换性能） |
| Hermes 长会话（v0.20.0） | 渐进式工具结果裁剪 + 逐轮微压缩 + 保留最近用户消息 | 工具输出可弃、用户意图不可弃；且压缩要摊平到每一轮而非一次性做 |

这三条路线没有对错，但它们把"什么是 Agent 的上下文"这个问题具体化了：**上下文不是对话记录，而是被反复裁剪的一份工作集**。谁的裁剪策略更贴近任务结构，谁在长任务上就更稳。

### 5.6 本节小结：从两份材料能读出的"什么是 agent"

综合以上五条，一个可操作的描述是：

> **Agent = 一个受预算约束的上下文管理器 + 一组被边界约束的执行能力 + 一个能跨会话留存的状态区。**

三家（加上被两家共同参照的 OpenClaw）在具体实现上差异巨大，但这三件事一件都不能少：没有上下文管理就退化成单轮问答；没有执行能力就退化成文本生成；没有跨会话状态就永远需要人重新交代背景。**而"Harness/OS"这一层的全部工程，几乎都是在给这三件事设预算、划边界、定策略。**

这也是为什么 OryxOS 反复强调"瓶颈不在模型，在运行环境"，以及为什么 Hermes 会把大量版本预算花在压缩、审批、隔离与恢复上——**两家的实践都在说明：Agent 的能力上限由模型决定，但 Agent 的可靠性上限由它外面的那层决定。**

---

## 6. 证据边界与不确定性声明

本报告严格区分事实与推断。以下四条是本报告结论强度的边界，建议连同结论一并阅读。

**（1）两边的证据标准不对称，本报告已在 §4.2 显式标注但无法完全消除。**

Hermes 侧的证据是**用户文档 + 版本化发布说明**（可核验到 v0.20.0）；OryxOS 侧的证据是**设计文档 + 仓库工程约束**的混合体，难以逐条判断落地状态。因此本报告对"OryxOS 是否具备某能力"的判断，强度普遍低于对 Hermes 的同类判断。**凡文中写"文档未见/公开材料未见"，均指证据缺失，不等于该能力不存在。**

**（2）Hermes 侧的"企业化"叙事，本报告未采信为对比依据。**

公开检索显示存在 `feat(enterprise): control-plane core — resource model, store, audit, …` 的仓库提交、以及第三方媒体关于"Nous Research 将 Hermes Agent 做成 Teams 企业平台"的报道。但本报告**无法访问 GitHub 与多数第三方站点**（域名解析到非公开 IP），无法核实，故未采信。若该 control-plane 已落地，§4.3 中"双方公开材料均未见租户级 RBAC + SSO + 租户维度审计报表"一句需要相应修订。

**（3）OryxOS 侧存在"文档阶段"与"仓库实现"的落差，本报告可能仍有低估。**

`docs/oryxos-docs/TechnicalSolution.md` 把多租户、SSO、Tool Policy、Web 仪表板等列为扩展阶段；但工作区根目录的 **`CLAUDE.md`（OryxOS 项目宪法）** 显示：工具治理层（`tool_policy_rules` 表，020）、AES-GCM 凭证加密（022）、Provider 主备切换（023）、多副本集群与 CAS 认领（026）、共享卷工作区（027）、Helm/K8s 交付与 OTel trace（039）、工作区存储约束（042）等**已作为工程约束存在**。

本报告据 `CLAUDE.md` 修正了两处先前的低估（工具策略层、分布式就绪度），但**该文件是仓库工程约束文档，不是产品发布文档**，同样不等于第三方验证；且本报告未通读 OryxOS 全部代码与文档，**可能仍有其他能力被低估**。此前两版报告分别出现过一次口径偏差（Hermes 方向、OryxOS 方向各一次），读者宜按此提示复核关键结论。

**（4）Hermes 的许可与商业边界未在本次可访问的文档页中明确。**

文档站标注"OPEN SOURCE / COMMUNITY MAINTAINED"，但具体开源协议（OryxOS 调研文档称其 MIT，`IndustryResearch.md` 2.1）未在本次抓取的功能文档页中出现。而 Tool Gateway 页明确其为**付费 Nous Portal 订阅功能**，Modal 为"订阅的可选附加组件"——这是已核实的事实。正式引用前建议直接核对仓库 LICENSE。

---

## 7. 一页速查对照表

| 对比项 | OryxOS | Hermes Agent |
|--------|--------|--------------|
| 一句话定位 | 企业私有部署的 Agent OS 底座 | 功能最全的开源自主助手 |
| **比对版本** | 文档含 014/015/017/025/026/028 节修订，核心阶段目标 **1.0** | **v0.20.0（`v2026.8.3`，2026-08-03，"The Herald Release"）** |
| 语言/栈 | Java 21 + Spring Boot 3.x + Spring AI Alibaba | Python 3.11+（v0.20.0 起安装链路需 **Node.js 26**） |
| 分发 | 单 fat JAR → GraalVM Native | **v0.20.0 起：shell 安装器 / Docker / Nix**（Homebrew 与 pip/PyPI 已退出） |
| Agent 定义 | 一个目录 = 一个 Agent（`AGENT.md`） | profile + SOUL.md + skills + 上下文文件 |
| 多 Agent | 同进程多 Profile（统一底座） | 多进程 worker + Kanban 任务板；delegate_task 并发≤3/深度≤2；**A2A v1.0 插件已交付** |
| 主入口 | REST API（10 端点 + Swagger） | CLI / 20+ 消息平台 / 本地 Dashboard |
| 循环上限 | 10 次（可配） | **v0.20.0 默认 500 轮**（原 90，可配）；子 Agent 50 |
| 工具并行 | 核心阶段不并行 | 支持 |
| 流式 | 扩展阶段（SSE） | 已支持 |
| 记忆 | 三层设计；核心/归档分区；3 档可插拔后端 | 两层内置（2200+1375 字符，冻结快照）+ 8 个外部后端 + FTS5 会话搜索 |
| 记忆安全扫描 | 未见提及 | 有（注入/外泄/后门/不可见字符） |
| 技能生态 | 受控、无公开市场（市场为社区共建） | Skill Hub 7 来源 + 信任分级 + Agent 自建技能 |
| 知识库 | 有独立模块（RRF 融合检索） | 未见独立模块 |
| 多模态 | 未见提及 | 视觉/图像生成/TTS/**实时语音+唤醒词** |
| 浏览器自动化 | 未见提及 | 原生（云+本地多后端） |
| IDE 集成 | 未见提及 | ACP（VS Code/Zed/JetBrains） |
| 沙箱 | 白名单（`SandboxChecker`）+ 独立工具策略层（020，仓库约束）；→ 容器 → microVM〔声明〕 | local/docker/ssh/singularity/modal/daytona；容器强加固〔v0.20.0 已交付〕 |
| 审计 | `tool_invocations`/`llm_calls` 表写入 + `trace_id` 串联（021）〔仓库约束〕 | 6 类插件钩子 + 智能审批 + HMAC Outbound Webhook + 日志/成本分析〔v0.20.0 已交付，需自行接入〕 |
| 认证 | 核心阶段无（内网假设）；API Key + JWT〔扩展阶段〕；落库凭证 AES-GCM 加密（022）〔仓库约束〕 | API Server Bearer Token；per-platform 用户白名单；Dashboard 无认证〔v0.20.0 文档〕 |
| 多租户/SSO/RBAC | 文档列为扩展阶段（三级租户模型）；多副本集群已具 CAS 认领与实例心跳（026）〔仓库约束〕 | 公开材料未见 |
| 可观测 | 结构化 JSON 日志 + 审计表；OTel trace 可选导出（039）〔仓库约束〕 | 日志三文件 + `hermes insights` + Dashboard Analytics〔v0.20.0〕 |
| 跨节点协作 | 文档列为阶段三愿景（A2A 通信底座 + 跨节点调度）；分布式部署（026/027/039）已有仓库约束 | A2A v1.0 协议接入〔v0.20.0 已交付〕；跨节点调度未见（Kanban 为同机多进程） |
| 商业模式 | Apache 2.0 开源，目标 Apache 基金会顶级项目 | 开源 + Nous Portal 付费订阅（Tool Gateway/Modal 等） |

> **状态标注法**：`〔v0.20.0 已交付〕`＝有版本化发布说明可核验；`〔仓库约束〕`＝见于 `CLAUDE.md` 工程约束，接近实现状态但非产品文档；`〔声明〕`/`〔扩展阶段〕`＝仅见于设计文档。**三类证据强度依次递减，读者不宜跨列直接比较。**

---

## 8. 结语

OryxOS 与 Hermes Agent 的差异，**不是功能多少的差异，而是对"Agent 应该长在哪里"这个问题给出的两个不同答案**：

- **Hermes 的答案**：Agent 长在**人的工作流里**——长在终端、长在聊天窗口、长在编辑器、长在桌面端、长在任务板上。所以它把功能做宽、把生态做活、把体验做顺，代价是底座治理让位于个体体验。
- **OryxOS 的答案**：Agent 长在**企业的治理体系里**——长在 REST API 后面、长在审计表里、长在白名单沙箱里、长在现有 Java 运维工具链里。所以它把接口做窄、把边界做硬、把治理写进架构，代价是能力扩展让位于治理确定性。

从 **v0.20.0** 这个横切面看，前一条路线的加速度很惊人——一个版本内约 3,650 次提交、650+ 位贡献者、A2A v1.0、实时语音、智能审批与签名 Webhook 同时落地。后一条路线在公开材料上呈现出的节奏是分阶段的：先用 4 周跑通运行时内核，再按信号补齐治理与分布式（仓库约束中 020/021/022/026/027/039 等编号即这一过程留下的刻度）。

**两种节奏各有代价**：快速铺开意味着能力面在短期内更完整，但每一项的治理深度取决于是否有人随之补齐；分期推进意味着内核与治理的边界始终清晰，但在补齐之前能力面是窄的。**这不是优劣，是两种不同的工程节奏。**

一个值得注意的现象：**按"OS"这个词的本义（管进程、管资源、管边界）衡量，OryxOS 的自我定位更贴近它，Hermes 的功能面更宽但组织单元是 profile 与 skill。** 但这个观察的效力有限——"OS"是 OryxOS 自己选用的隐喻（`IndustryResearch.md` 1.1 用"进程与操作系统"作类比），用它的隐喻去评价不做这个隐喻的项目，本身就是一种标准移植。**更稳妥的表述是：OryxOS 把治理机制作为产品的组织中心，Hermes 把能力供给作为产品的组织中心。**

对读者而言，真正可迁移的收获不在两家谁更合适，而在 §5 归纳的那几条设计问题——**上下文预算、渐进披露、隔离粒度、边界与能力的关系、压缩策略**。这些问题的答案不随产品的版本号变化，理解了它们，再读任何 Agent 项目的文档都会有判断力。

---

## 附录 A：来源清单

**OryxOS（本地文档）**
- `docs/oryxos-docs/oryxos.md` — 项目定位、愿景、五大核心能力、路线图、设计原则
- `docs/oryxos-docs/DemandAnalysis.md` — 需求文档（What）：术语、核心/扩展/社区共建功能分级、非功能需求、里程碑、风险
- `docs/oryxos-docs/TechnicalSolution.md` — 技术方案（How）：7 项关键技术决策、整体架构、五大能力模块、14 个 Maven 模块、数据持久化、性能
- `docs/oryxos-docs/IndustryResearch.md` — Agent OS 定义、OpenClaw/Hermes 格局分析、Java 生态缺位论证、安全对标、分布式路线
- **`CLAUDE.md`（工作区根目录，OryxOS 项目宪法）** — ⚠️ **证据等级与上述文档不同**：这是仓库工程约束文档，用于修正 §4.2 与 §7 中若干"低估"项（工具治理层 020、trace_id 串联 021、凭证 AES-GCM 加密 022、Provider 主备 023、多副本集群 CAS 归属 026、共享卷工作区 027、Helm/K8s 交付与 OTel 039、工作区存储 042）。**它不是产品发布文档，也不等于第三方验证**——引用时须保持与 `docs/oryxos-docs/` 的区分
- `docs/oryxos-docs/AiProgrammingGuide.md` — Spec-Kit + 手动提示词混合实施模式

**Hermes Agent（官方中文文档站；比对基线 = v0.20.0）**
- **版本与发布**：[v0.20.0 发布说明](https://hermesagent.org.cn/docs/releases/v0-20-0)（`v2026.8.3`，2026-08-03，"The Herald Release"）← **本报告版本信息的主要依据**
- [安装](https://hermesagent.org.cn/docs/getting-started/installation) ← Python 3.11+ 前提、extras 清单、v0.20.0 分发渠道变更
- [功能概览](https://hermesagent.org.cn/docs/user-guide/features/overview)
- [Nous 工具网关](https://hermesagent.org.cn/docs/user-guide/features/tool-gateway)
- [工具与工具集](https://hermesagent.org.cn/docs/user-guide/features/tools)
- [技能系统](https://hermesagent.org.cn/docs/user-guide/features/skills)
- [持久记忆](https://hermesagent.org.cn/docs/user-guide/features/memory)
- [MCP（模型上下文协议）](https://hermesagent.org.cn/docs/user-guide/features/mcp)
- [代码执行](https://hermesagent.org.cn/docs/user-guide/features/code-execution)
- [子 Agent 委派](https://hermesagent.org.cn/docs/user-guide/features/delegation)
- [Kanban 多代理协作板](https://hermesagent.org.cn/docs/user-guide/features/kanban)
- [事件钩子](https://hermesagent.org.cn/docs/user-guide/features/hooks)
- [Web 仪表板](https://hermesagent.org.cn/docs/user-guide/features/web-dashboard)
- [CLI 命令参考](https://hermesagent.org.cn/docs/reference/cli-commands)
- [配置文件命令参考](https://hermesagent.org.cn/docs/reference/profile-commands)
- [环境变量参考](https://hermesagent.org.cn/docs/reference/environment-variables)

> **关于版本标注方式**：Hermes 的功能文档页显示 `DOCS / CURRENT`，**页面本身不带版本号**；本报告的 v0.20.0 归属依据来自（a）侧边栏「版本更新」将 v0.20.0 置于最新、（b）v0.20.0 发布说明页的"官方标签 `v2026.8.3`，发布日期 2026 年 8 月 3 日"。因此**凡下文引用环境变量页（默认 90 轮、Node.js/Python 版本示例等）与 v0.20.0 发布说明（默认 500 轮、A2A、智能审批）存在冲突之处，一律以发布说明（更新）为准，并已在正文中显式标注**。这也是第一版报告出现口径偏差的原因，特此说明。

> 说明：Hermes 中文文档站为社区维护的镜像/汉化站点，上游为 NousResearch/hermes-agent 官方仓库的 `website/docs/`。个别页面存在导航发现但正文未完整抓取的情况（如 `/docs/user-guide/features/api-server`、`/docs/user-guide/messaging`、`/docs/integrations`、`/docs/user-guide/features/a2a`、`/docs/user-guide/checkpoints-and-rollback`、`/docs/user-guide/features/plugins`），本报告对这些章节的引用均来自已成功抓取的页面内交叉信息（如环境变量表中的 `API_SERVER_*`、CLI 命令表中的 `hermes mcp serve` / `hermes acp` / `hermes plugins`、发布说明中的 A2A v1.0 与智能审批），未作超出证据的推断。

## 附录 B：待补充核实的项

如需进一步核实本报告的判断，建议补齐以下项（本节为**证据补强清单**，非选型建议）：

1. **Hermes 的 enterprise control-plane 现状** —— 公开检索到的 `feat(enterprise): control-plane core — resource model, store, audit, …` 提交与第三方"Teams 企业平台"报道，本报告因无法访问 GitHub/多数第三方站点而**未采信**。v0.20.0 发布说明中亦未见多租户/RBAC/SSO 条目，建议直接核对仓库与最新 release notes 确认。
2. **Hermes 的 LICENSE** —— 确认是否为 MIT，以及 Nous Portal 相关组件（Tool Gateway、订阅代理）是否在同一许可下。
3. **Hermes 文档页与发布说明的口径差** —— 环境变量页中"默认 90 轮"、`hermes dump` 示例里的旧版本号（0.8.0 / 2026.4.8）等均为 v0.20.0 之前的内容，说明该文档站部分页面的更新滞后于发布说明。正式引用前建议以对应版本的 release notes 复核数值。
4. **OryxOS 的实现状态（本报告最薄的一环）** —— 本报告主要依据 `docs/oryxos-docs/` 设计文档与根目录 `CLAUDE.md` 工程约束，**未通读代码**。已知 `CLAUDE.md` 提到的 020（工具治理层）/021（trace_id）/022（凭证加密）/023（Provider 主备）/026（多副本集群）/027（共享卷）/039（Helm+OTel）/042（工作区存储）等均指向"已实现"，与设计文档中的阶段划分存在落差。建议以代码、Release 记录与 Helm Chart 为准逐项确认，**尤其是认证与多租户的实际状态**。
5. **OryxOS 的隔离强度实测** —— 白名单在真实攻击面下（路径穿越、解释器逃逸、SSRF）的有效性，以及容器/microVM 实现是否已落地。
6. **API Server 与 Dashboard 的认证矩阵** —— 核实 Hermes `API_SERVER_KEY` 的粒度（单 token 还是多用户），以及 Dashboard 是否存在绑定非 localhost 时的认证方案。
7. **两者的 A2A 语义差异** —— Hermes 的 A2A v1.0 为插件形态（发现/调用/被调用）；OryxOS 的 A2A 在文档中为阶段三"跨节点通信底座"愿景（是否有仓库级实现需核实）。需对比二者在"跨节点可靠委托与异步协同"上的实际能力边界。
8. **两者的 MCP 服务端能力对比细节** —— OryxOS 的 MCP Server 暴露在文档中列为扩展阶段，Hermes 已有 `hermes mcp serve`，需对比暴露面的语义差异。
9. **Hermes 的架构设计文档** —— 本报告对 Hermes 的机制理解主要来自**用户功能页**，而非其架构/开发者文档（`hermes-agent-source-wiki` 一类第三方源码解读因网络受限未能获取）。若要深入"什么是 agent"这一层，应直接读源码与开发者指南。
