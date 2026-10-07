# OryxOS 多实例扩展文档（阶段二：底座分布式）

> 本目录是 OryxOS **阶段二（底座分布式）** 的四篇文档，仿照 `docs/oryxos-docs/` 的《业界调研 / 需求文档 / 技术方案 / AI 编程指南》四篇结构组织，回答同一个问题的四个面：**为什么要多实例（Why）、多实例要满足什么（What）、多实例怎么做（How）、多实例怎么用 AI 编程做出来（How to build）**。
>
> 前置阅读：`docs/oryxos-docs/oryxos.md`（项目定位）、`docs/oryxos-docs/IndustryResearch.md` 第六章（从单机到分布式的方向）、`docs/oryxos-docs/TechnicalSolution.md`（阶段一单机技术方案）、项目根目录 `CLAUDE.md`（项目宪章八原则 + 工作区存储约束）。

---

## 1. 文档定位

阶段一交付的是**单机运行时内核**：一个 OryxOS 实例，装在企业自己的一台服务器或一个容器里，跑起一组 Agent，服务一个部门或一个场景。这件事已经做扎实——五大核心能力、多 Agent 并存、审计 day one 落库、渠道入站、知识库、人格库、Tool 治理、定时任务。

阶段二要解决的是阶段一刻意没有解决的问题：**当一个企业从"一个部门试点"走向"服务全公司"时，单实例会同时撞上三件事**——扛不住量、扛不住故障、扛不住规模化治理。阶段二把"一个实例服务全公司"变成"一组实例服务全公司"。

这套文档覆盖的特性范围（与项目特性账本对应）：多副本部署、文件面分布式、容器交付、工作区存储插件化。四篇文档把它们当作**本周期的设计目标**从头完整论证，而不是当作已有实现的记录。

| 文档 | 回答 | 一句话内容 |
|------|------|-----------|
| [IndustryResearch.md](IndustryResearch.md) | Why | 业界分布式 Agent 底座与分布式协调的成熟做法，以及 OryxOS 从中选了什么、拒了什么 |
| [DemandAnalysis.md](DemandAnalysis.md) | What | 阶段二的功能需求、非功能需求、数据模型、验收标准，不展开 How |
| [TechnicalSolution.md](TechnicalSolution.md) | How | 共享状态层、恰好一次三件套、渠道属主、共享工作区、知识代次、Helm 交付的完整设计 |
| [AiProgrammingGuide.md](AiProgrammingGuide.md) | How to build | 用 Spec-Kit + 手动提示词混合模式把阶段二做出来，7 个 user story 的拆解与红线 |

**阅读顺序**：IndustryResearch → DemandAnalysis → TechnicalSolution → AiProgrammingGuide。只想上手写代码的读者可以直接读 TechnicalSolution 第 1 章（决断表）+ AiProgrammingGuide 第 4 章（拆解）。

---

## 2. 范围边界

阶段二是一条清晰的线：**让一组无状态实例共享同一份状态，对外表现成一个 OryxOS**。线内是要做的，线外明确不做。

**线内（本套文档覆盖）：**

1. 集群开关与运行档位（单机档 / 集群档）、实例标识、误配拒启动
2. 实例注册与心跳，实例可查（`GET /api/v1/instances`）
3. 会话轮次跨副本恰好一次（同会话串行、跨副本排队）
4. 定时任务到点跨副本恰好一次（幂等键取理论触发时刻）
5. 渠道入站事件的回执去重跨副本生效（替换进程内 Map）
6. 渠道长连接单属主、永不互踢、故障接管、停机先释放
7. `.oryxos/` 工作区放共享卷、存储插件化、原子落盘、版本号总线变更感知
8. 知识库索引在集群档的代次化与重建认领
9. 崩溃语义（标失败不重放）与租约接管时序
10. 官方 Helm Chart 交付、探针、滚动升级、优雅停机
11. 跨副本可观测（traceId 同源、审计带实例、OTel 可选导出）
12. 集群档的密钥与安全（主密钥多副本一致、加密字段、工作区权限）

**线外（明确不做，见 TechnicalSolution 最后一章）：**

1. **跨节点 Agent 协作**（Agent 之间跨实例发现、委托、A2A 协同）——那是阶段三，属于另一层「Agent 通信底座」，不混进阶段二
2. 多区域 / 多活 / 跨地域部署
3. 多主数据库、分区容忍（本方案是 CP 取向：单写者 + 中心协调，不做冲突合并）
4. 分布式事务（跨多张表的业务级原子性）
5. 引入新中间件（Redis、ZooKeeper、etcd、消息队列）——阶段二只在 PostgreSQL 与共享文件系统上做
6. Agent 级调度亲和 / 会话粘性路由——本方案的前提恰恰是**任意请求落到任意副本都正确**
7. 阶段三及以外的容量模型（十万级 Agent、跨组织联邦）

---

## 3. 统一术语

四篇文档共用以下术语，含义以此表为准（阶段一已有术语见 `docs/oryxos-docs/DemandAnalysis.md` 第 2 章，此处只列阶段二新增）。

| 术语 | 定义 |
|------|------|
| **实例 / 副本（instance / replica）** | 一个 OryxOS 进程。集群档下每个实例有全局唯一的 `instance-id`，实例之间完全对等，没有主从之分 |
| **单机档 / 集群档** | 由 `oryxos.cluster.enabled` 决定的两个运行档位。单机档是默认值，走阶段一的全部代码路径；集群档打开协调原语。同一个 jar、同一个镜像、同一份配置结构，唯一区别是这个开关和它所要求的存储面 |
| **共享状态层** | 集群档下多实例共读写的状态载体，本方案里是 PostgreSQL（关系型状态 + 协调原语）和共享卷（文件面状态）两部分 |
| **认领（claim）** | 用一条带条件的写语句（CAS）把某件事在某一时刻唯一归属到某个实例的动作。认领失败即代表别人持有 |
| **租约（lease）** | 认领的凭据加过期时间。持有方必须周期性续约；持有方崩溃后，租约自然过期，由其他实例接管 |
| **属主（owner）** | 某个渠道长连接或某项在跑工作的当前唯一持有实例 |
| **理论触发时刻（theoretical fire time）** | cron 表达式算出来的应触发时刻，由调度器（`CronTrigger`）给出，**不是墙钟时间**。它是定时任务跨副本去重的幂等键 |
| **幂等键（idempotency key）** | 用来判定"这件事是不是同一件"的键。会话轮次用（会话 + 轮次序号），定时任务用（任务 + 理论触发时刻），入站事件用（渠道 + 事件 id），知识重建用（后端 + 作用域） |
| **事件回执去重** | 入站渠道对同一条消息事件只处理一次。阶段一用进程内 Map，集群档必须落到共享状态层 |
| **版本号总线（version bus）** | `workspace_versions` 表按域维护单调递增的版本号。管理面写路径落盘后 bump 对应域，各副本轮询比对自身已加载版本，落后即重载 |
| **域（domain）** | 版本号总线上的一类可独立重载的工作区数据：agents、skills、personas、knowledge、channels、bootstrap |
| **代次（generation）** | 知识索引的一次已提交快照版本。检索**恒读已提交代次**，重建期间旧代次继续服务 |
| **本机执行视图（native path）** | 共享工作区里的一个路径在本实例上真实可执行的本地路径。Tool（文件、Shell）执行必须用本机执行视图，不能把逻辑路径直接当文件路径用 |
| **逻辑路径** | 相对于工作区根的位置描述（例如 `agents/ops-agent/AGENT.md`）。它在各副本之间一致、可比较、可落库、可作为版本号总线的键；但它**不是**本机可直接打开的文件路径，也不能用来做真实路径（软连接越界）校验 |
| **管理写互斥预约** | 管理面对某类工作区数据的写操作前提。它是人工介入的闸门，**不自动过期**；恢复前必须先停掉全部写者 |
| **读写者停机预约** | 停机流程里的写者退场确认，保证"最后一个写者关灯"之后才允许做离线恢复动作 |
| **保留路径** | `.workspace-*` 与 `.staging`。工作区里以它们开头的路径由底座内部占用，业务 Agent 不得使用 |
| **逃生舱（escape hatch）** | `POST /api/v1/workspace/refresh`。运维绕过版本号总线直接改盘后，用它强制各副本重载 |
| **崩溃轮次** | 实例在跑某个会话轮次或某个定时触发时崩溃留下的半成品记录。**标失败、不重放**，由用户重发恢复 |
| **误配拒启动** | 集群档下遇到本方案不支持的状态组合（如 SQLite、markdown 记忆后端、内存知识后端）时，启动直接失败并给出恢复指引，不静默降级 |

---

## 4. 统一设计决断（D1–D14）

四篇文档在论证时引用这些编号，含义以此表为准。

| # | 决断 | 一句话理由 |
|---|------|-----------|
| **D1** | 单机档零变化、集群档零共享假定 | 同一个 jar 同一个镜像；集群档才打开协调原语，单机档代码路径不新增协调调用，避免为非目标场景付复杂度 |
| **D2** | 协调基质复用 PostgreSQL，不引入新中间件 | OryxOS 已经支持 PostgreSQL 部署档，协调所需的能力（条件写、唯一约束、TTL 判断、行级可见性）它都有；少一个中间件就少一套高可用、监控、备份、审计面 |
| **D3** | 恰好一次只承诺在五个粒度上 | 会话轮次、定时触发、入站事件、渠道属主、知识重建。其余链路（审计落库、通知推送）是「至少一次 + 幂等键」，文档必须写清边界而不是笼统宣称 exactly-once |
| **D4** | 一切跨副本互斥走「CAS 单语句 + TTL 租约」 | 不用应用层读-改-写（有竞态）、不用悲观长事务（会拖垮连接池）。续约由专用调度线程同步阻塞完成，遵守宪法的同步执行模型 |
| **D5** | 定时任务幂等键取理论触发时刻 | 墙钟会被漂移、GC、NTP 影响；用 `CronTrigger` 的理论触发时刻做唯一键，多个副本算出同一个值，才能在数据库上做唯一约束 |
| **D6** | 崩溃轮次标失败、不重放 | 重放一个已经产生过外部副作用（发过消息、调过工单系统）的轮次会造成重复副作用；标失败 + 用户重发是唯一安全的语义 |
| **D7** | 文件面只依赖「读写可见 + rename 原子」 | 共享卷（NFS/CIFS/云盘 RWX）不保证文件锁、inotify、内存映射一致；把不保证的能力从设计假设里删掉，才能跨存储实现都正确 |
| **D8** | 变更感知走版本号总线 + 轮询，集群档不装文件 watcher | 单机档的 `WatchService` 在共享卷上不可靠（事件丢失、跨节点不可见）；版本号是唯一真相源，轮询有确定上限，逃生舱兜底 |
| **D9** | 工作区访问统一走存储抽象 + 本机执行视图 | 业务代码只用 `WorkspaceStorage.root()/resolve()`；共享模式启动先校验 `.workspace-id`，不匹配就拒启动，**绝不回退本地**（否则数据分裂） |
| **D10** | 渠道长连接单属主、永不互踢 | IM 平台侧对同渠道多连接的行为不一致（有的会互踢到全部掉线）；属主租约只允许在过期后接管，接管前先让旧连接走完停机链路 |
| **D11** | 集群档误配启动即拒 | 错的存储组合会让多实例静默读写各自的数据副本，故障发生在几天之后且不可审计；宁可在启动时报错。**启动校验七类**：SQLite 档、markdown 记忆后端、内存（memory）知识后端、`oryxos.workspace.storage=local`、工作区标识缺失或不匹配、主密钥缺失或跨副本不一致、实例标识重复。**另有一类运行期 fail-closed**（不占启动校验名额）：协调基质（数据库）不可用时集群档拒服务，不降级为单机 |
| **D12** | 交付走官方 Helm Chart + 容器原生面 | 企业不是手搓 docker run；chart 自带探针、滚动升级、优雅停机、卷与密文引用，才能被现有 K8s 治理流程接管 |
| **D13** | 跨副本可观测与审计同源 | 一次消息处理跨多个副本时，审计行、日志、trace span 必须能拼回同一条链路（同 traceId、同计时区间、带实例标识） |
| **D14** | 边界：不做阶段三、不做多活、不做多主、不加中间件 | 每一条都是"等真实数据证明必要"的事，写在文档里是为了让评审时不必反复讨论 |

---

## 5. 统一状态表清单

集群档新增以下状态表（字段级设计见 TechnicalSolution 第 4 章与 DemandAnalysis 第 10 章；结构按第 5 章末段的双轨纪律随 Flyway 迁移落到 PostgreSQL，SQLite 轨保留结构等价最小实现）：

| 表 | 承担的事 | 主键 / 幂等键 | 关键列 |
|----|---------|--------------|--------|
| `instances` | 实例注册与心跳 | `instance_id` | `host`、`port`、`version`、`status`、`started_at`、`last_heartbeat_at` |
| `session_turn_leases` | 会话轮次跨副本恰好一次 | `session_id`（同一时刻一个持有者） | `owner_instance_id`、`lease_id`、`turn_seq`、`state`、`acquired_at`、`expires_at`、`last_error` |
| `schedule_claims` | 定时任务到点跨副本恰好一次 | `(schedule_id, fire_time)` 唯一 | `owner_instance_id`、`lease_id`、`state`、`claimed_at`、`expires_at`、`finished_at`、`error_message` |
| `event_receipts` | 入站事件回执去重 | `(channel, event_id)` 唯一 | `instance_id`、`received_at`、`expires_at` |
| `channel_ownerships` | 渠道长连接属主 | `channel_name` | `owner_instance_id`、`lease_id`、`acquired_at`、`expires_at`、`last_renew_at` |
| `workspace_versions` | 版本号总线 | `domain` | `version`（单调递增）、`updated_at`、`updated_by` |
| `admin_write_reservations` | 管理写互斥预约（不自动过期） | `scope` | `owner_instance_id`、`reserved_at`、`note` |
| `knowledge_build_claims` | 知识索引重建认领 | `(backend, scope)` | `owner_instance_id`、`lease_id`、`state`、`acquired_at`、`expires_at` |
| `knowledge_generations` | 知识索引已提交代次 | `generation_id` | `backend`、`scope`、`state`（`building` / `committed` / `retired`）、`chunk_count`、`created_at`、`committed_at` |

状态枚举统一：`session_turn_leases.state` ∈ {`claimed`, `running`, `failed`}；`schedule_claims.state` ∈ {`claimed`, `running`, `succeeded`, `failed`, `skipped`}；`knowledge_build_claims.state` ∈ {`claimed`, `running`, `failed`}（"已提交"由 `knowledge_generations.state=committed` 表达，认领记录不承担代次语义）；`instances.status` ∈ {`starting`, `ready`, `draining`, `stale`}。

索引与约束命名规范（四篇文档与迁移脚本统一）：唯一约束 `uq_<表名缩写>_<列名>`，普通索引 `idx_<表名缩写>_<列名>`，例如 `uq_schedule_claims_schedule_fire(schedule_id, fire_time)`、`idx_session_turn_leases_expires(expires_at)`、`idx_event_receipts_expires(expires_at)`、`uq_knowledge_generations_committed(backend, scope) WHERE state = 'committed'`（部分唯一索引）。

**四张认领表的栅栏语义（`lease_id`）**：`session_turn_leases`、`schedule_claims`、`channel_ownerships`、`knowledge_build_claims` 都带 `lease_id`。续约与释放必须**同时匹配** `owner_instance_id` 与 `lease_id`；到期接管由接管方写入**新的** `lease_id`。这样旧持有者即使从长暂停（GC/STW/网络隔离）中醒来，其续约语句也只会影响 0 行，无法用过期凭据"复活"——这是双属主窗口能被量化、可接受的前提。`schedule_claims` 的接管是**带新 `lease_id` 的 CAS 改写**（不是插入新行，因为 `(schedule_id, fire_time)` 是唯一键也同时是幂等键）。

审计表增量：`tool_invocations`、`llm_calls` 增加 `instance_id` 列（与既有 `trace_id` 一起构成跨副本链路追溯），`task_executions` 增加 `instance_id` 与 `fire_time`。

创建与演进走 Flyway 双轨目录（`db/migration/sqlite/`、`db/migration/postgresql/`）：**同一变更两 vendor 各写一份、同版本号、只增不改**——集群档新增的 9 张表同样在两轨各写一份（SQLite 轨保留结构等价的最小实现，保证迁移序列对称、单机档与集群档的表结构可对照），是否允许启用这些表由集群档启动校验决定，而不是靠"只在一个 vendor 里建表"。`hibernate.ddl-auto` 保持 `none`。

---

## 6. 统一量级目标

四篇文档引用的目标数字以下表为准（压测方法见 TechnicalSolution 第 17 章）。

| 维度 | 单机档（阶段一） | 集群档目标（阶段二） |
|------|----------------|-------------------|
| 副本数 | 1 | 2 ~ 8（推荐 3），本周期验证上限 32 |
| 单集群 Agent 数 | ≥ 10 | ≥ 100 |
| 单集群并发 Session | ≥ 100 | ≥ 1000 |
| 单副本吞吐（mock provider 固定时延） | 参照阶段一 | ≥ 50 消息/秒 |
| 集群加速比 | — | 2 → 4 副本 ≥ 0.8 |
| 工作区变更传播延迟 | 即时（同进程） | 轮询间隔的 2 倍以内（默认 1s → P99 ≤ 3s） |
| 渠道属主接管 | — | 租约过期后 ≤ 30s |
| 会话轮次接管 | — | 租约过期后 ≤ 60s，且崩溃轮次被标失败 |
| 定时触发重复计数 | 不适用 | 每个 `fire_time` 恰好 1 行 `task_executions` |
| 滚动升级 | 停机即中断 | 零消息丢失，单副本不可用窗口 = 在途请求排空时间 |
| 集群可用性目标 | 单点 | 99.9%（3 副本 + PostgreSQL 高可用，可用性下限由数据库侧决定） |

数字口径说明：吞吐与时延指标在**固定时延的 mock provider** 下测量（`-Doryxos.mock.latency-ms`），把 LLM 侧不确定性排除在外；真实吞吐由企业所接 Provider 决定，不作为底座的承诺指标。

**统一默认参数（集群档，四篇文档引用同一套值，不得各写一套）：**

| 参数 | 配置项 | 默认值 |
|------|--------|--------|
| 集群开关 | `oryxos.cluster.enabled` | `false`（单机档） |
| 实例标识 | `oryxos.cluster.instance-id` | 缺省由主机名 + 短随机后缀生成；K8s 下建议显式用 Pod 名 |
| 实例心跳间隔 / 失效判定 | `oryxos.cluster.heartbeat-interval` / `-ttl` | 10s / 30s |
| 会话轮次租约 TTL / 续约间隔 | `oryxos.cluster.session-turn-lease-ttl` | 60s / 20s |
| 会话轮次跨副本排队等待上限 | `oryxos.cluster.session-turn-wait-timeout` | 120s |
| 定时触发认领 TTL | `oryxos.cluster.schedule-claim-ttl` | 120s |
| 入站事件回执保留窗口 | `oryxos.cluster.event-receipt-ttl` | 24h |
| 渠道属主租约 TTL / 续约间隔 | `oryxos.cluster.channel-lease-ttl` | 30s / 10s |
| 知识重建认领 TTL / 续约间隔 | `oryxos.cluster.knowledge-claim-ttl` | 30min / 5min |
| 工作区版本轮询间隔 | `oryxos.workspace.poll-interval` | 1s |
| 工作区存储模式 / 工作区标识 | `oryxos.workspace.storage` / `oryxos.workspace.id` | `local` / 共享卷根下 `.workspace-id` |
| 保留的已提交知识代次 | `oryxos.knowledge.keep-generations` | 2（当前 + 上一代） |
| 管理写互斥预约 | — | **不自动过期**，人工释放 |
| 优雅停机 | `spring.lifecycle.timeout-per-shutdown-phase` / Pod `terminationGracePeriodSeconds` | 10s / 40s |
| 滚动策略 | — | `maxUnavailable: 0` / `maxSurge: 1` |

---

## 7. 统一里程碑与验收场景

阶段二按 **M0 ~ M5** 六个里程碑推进，每个里程碑结束有可演示成果：

| 里程碑 | 内容 | 可演示成果 |
|--------|------|-----------|
| **M0** | 共享状态基座：集群开关、实例标识、误配拒启动、实例心跳、PostgreSQL 档 | 拉起 2 副本，`GET /api/v1/instances` 看到两个实例与心跳；误配组合启动即拒并给出恢复指引 |
| **M1** | 恰好一次三件套：会话轮次租约、调度到点认领、事件回执去重（AI 编程指南拆成 US-2 → M1a 会话轮次、US-3 → M1b 定时与回执，两段合起来才是 M1） | 同一会话并发打到 3 副本，回复不重不乱；同一 cron 在 3 副本上每个 `fire_time` 只执行一次 |
| **M2** | 渠道属主与停机链路 | 同一渠道多副本启动只有一个是属主；kill 属主后 TTL 内接管且不互踢、不双回复 |
| **M3** | 共享工作区：存储插件、原子落盘、版本号总线、本机执行视图、管理写预约、逃生舱 | 一个副本改 `AGENT.md` / 绑 Skill，其余副本 P99 ≤ 3s 生效；直接改盘后逃生舱强制重载 |
| **M4** | 知识库代次化：重建认领、已提交代次、导入互斥 | 多副本同时触发重建只有一个执行（其余 409 排队），检索全程读已提交代次不断服 |
| **M5** | 交付与验证：Helm Chart、探针、滚动升级、跨副本可观测、HA 演练 | 滚动升级期间消息零丢失；压测报告（加速比、传播延迟、接管时间） |

阶段二的验收是五个端到端场景（详见 DemandAnalysis 第 13 章）：

| Demo | 场景 | 硬性判据 |
|------|------|---------|
| **A** | 滚动升级零中断 | 升级期间 IM 消息全部有且仅有一次回复；定时任务每个 `fire_time` 恰好一次 |
| **B** | 副本猝死接管 | 崩溃轮次标失败不重放；渠道属主在 TTL 内被接管；无重复回复、无消息丢失 |
| **C** | 定时任务跨副本恰好一次 | `task_executions` 每个 `fire_time` 恰好一行；人为阻塞一个副本也不重复 |
| **D** | 工作区变更传播 + 知识重建认领 | 变更 P99 ≤ 3s 生效；重建只执行一次，冲突方排队不丢，检索不断服 |
| **E** | 运维逃生舱与实例视图 | 直接改盘后逃生舱生效；`GET /api/v1/instances` 与审计能对上实例口径 |

---

## 8. 写作纪律（四篇共同遵守）

1. **遵守项目宪章**：八条不可违背原则在集群档下依然成立，尤其是原则二（Spring AI 只用两件事、禁用自动 tool 执行）、原则三（Provider 显式映射）、原则七（同步执行模型，多副本不引入 Reactor/WebFlux/CompletableFuture，续约与轮询都在调度线程上同步阻塞）、原则六（软连接与真实路径校验，共享卷下更严）、原则五（审计 day one 落库，集群档每条审计行带 `instance_id`）。四篇文档都要有专门段落写明本方案如何不违反这些约束。
2. **遵守工作区存储约束**：所有工作区访问走存储抽象，业务代码不出现裸文件 API；本机执行必须取本机执行视图；共享模式校验工作区标识；保留路径与 `.staging` 神圣不可占用；多文件发布写清"可恢复但不是目录原子快照"；运行输出按 Agent/run 隔离。业务语言用中文，代码标识符保留英文原名。
3. **区分口径**：本套文档写的是**阶段二的设计**，不写"已实现"。可以引用阶段一已落地的能力作为基线（五大能力、审计表、渠道适配器、知识库、定时任务、Tool 治理、人格库），但不把阶段一的能力描述成阶段二的成果。
4. **数字要诚实**：目标数字用统一量级目标那一节的口径；不确定的地方做保守表述并标注口径（例如"在固定时延 mock provider 下测得"），不编造外部数据。引用业界项目时只写有把握的事实。
5. **行文风格对齐 `docs/oryxos-docs/`**：中文技术叙述，先给判断再给理由；大量使用 Markdown 表格做对比；关键结论用引用块（`>`）单列；每章末尾用「本方案不做」或「边界」类小节收口；图表用 ASCII 目录树 / 流程图；章节编号用 `## 1.` `### 1.1` 两级为主。不写营销口号，不堆形容词。
6. **术语与编号一致**：所有术语用第 3 章的统一术语表；所有设计决断引用 D1–D14 编号；所有状态表用第 5 章的表名；所有目标数字用第 6 章的口径；所有里程碑用 M0–M5；所有验收场景用 Demo A–E。
7. **互相引用**：四篇文档在需要时用相对路径互相引用（例如"见 [TechnicalSolution.md](TechnicalSolution.md) 第 4 章"），并引用阶段一文档与 `CLAUDE.md` 作为约束来源。
