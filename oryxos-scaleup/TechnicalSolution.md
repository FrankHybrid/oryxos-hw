# OryxOS 多实例扩展技术方案

本文档定义 OryxOS 阶段二（底座分布式）的技术方案，回答 How 的问题。前置阅读本目录 [README.md](README.md)（总纲：范围边界、统一术语、D1–D14 决断、状态表清单、量级目标、里程碑）、`docs/oryxos-docs/TechnicalSolution.md`（阶段一单机技术方案，本文档与之同构可对照阅读）、`docs/oryxos-docs/DemandAnalysis.md`（阶段一需求）与项目根目录 `CLAUDE.md`（项目宪章八原则 + 042 工作区存储开发约束）。本文档只给契约、状态机、SQL 语义、配置项与模块职责，不展开 Java 代码实现细节。

> 承接阶段一定位：阶段一交付的是**单机运行时内核**——一个实例、一份本地状态、五大核心能力齐备，服务一个部门或一个场景。阶段二要解决的是阶段一刻意没有解决的问题：当一个企业从"一个部门试点"走向"服务全公司"时，单实例会同时撞上**扛不住量、扛不住故障、扛不住规模化治理**三件事。阶段二的边界是一条清晰的线：**让一组无状态实例共享同一份状态，对外表现成一个 OryxOS**。跨节点 Agent 协作（A2A）属于阶段三的 Agent 通信底座，本文档不做设计。

> **文档结构提示：** 全文分两部分。第一部分（第 1–20 章）是**多实例底座**——共享状态层、认领原语、恰好一次三件套、渠道属主、共享工作区、版本号总线、知识代次、数据迁移、可观测、安全、容器交付、性能容量、故障模型、模块结构。第二部分（第 21–24 章）是**验证与约束**——Demo A–E 的端到端时序与断言、M0–M5 实施节奏、与项目宪章的逐条对齐、总结。

---

# 第一部分：多实例底座

## 1. 方案概述

阶段二在同一个 jar、同一个镜像、同一份配置结构上增加一个运行档位：`oryxos.cluster.enabled=true` 时进入**集群档**，打开协调原语；默认 `false` 是**单机档**，走阶段一的全部代码路径，一行运行时行为都不变（D1）。集群档下多个实例完全对等，没有主从，状态全部外置到**共享状态层**——PostgreSQL（关系型状态 + 协调原语）与共享卷（文件面状态）两部分。任意请求落到任意副本都必须正确，这是本方案全部设计的前提假设。

> **技术栈增量一句话总结**：阶段二不新增任何中间件，协调基质复用已支持的 PostgreSQL；文件面只依赖共享卷的"读写可见 + rename 原子"；交付面新增官方 Helm Chart 与容器原生面；可观测面新增 `instance_id` 与可选 OTel 导出。

### 1.1 关键技术决策

README 第 4 章的 D1–D14 是四篇文档共同遵守的决断，下面先用一张表速览，再逐条展开"选择 / 理由 / 代价 / 不这么做会怎样"。

| # | 决策 | 选择 | 一句话理由 |
|---|------|------|-----------|
| **D1** | 运行档位 | 单机档零变化、集群档零共享假定 | 同一个 jar 同一个镜像；集群档才打开协调原语，不为非目标场景付复杂度 |
| **D2** | 协调基质 | 复用 PostgreSQL，不引入新中间件 | 条件写、唯一约束、TTL 判断、行级可见性它都有；少一个中间件少一套高可用、监控、备份、审计面 |
| **D3** | 恰好一次的粒度 | 只承诺会话轮次、定时触发、入站事件、渠道属主、知识重建五个粒度 | 笼统宣称 exactly-once 会在评审和运维上同时破产；其余链路诚实标注"至少一次 + 幂等键" |
| **D4** | 跨副本互斥机制 | CAS 单语句 + TTL 租约 | 应用层读-改-写有竞态；悲观长事务会拖垮连接池 |
| **D5** | 定时幂等键 | 取 `CronTrigger` 的理论触发时刻（fireTime） | 墙钟会被漂移、GC、NTP 影响，各副本算出不同的值就无法在数据库上做唯一约束 |
| **D6** | 崩溃语义 | 崩溃轮次标失败、不重放 | 重放已产生外部副作用的轮次会造成重复副作用；标失败 + 用户重发是唯一安全的语义 |
| **D7** | 文件面依赖能力 | 只依赖读写可见 + rename 原子 | 共享卷不保证文件锁、inotify、内存映射一致；把不保证的能力从设计假设里删掉 |
| **D8** | 变更感知 | 版本号总线 + 轮询，集群档不装文件 watcher | 共享卷上事件会丢、跨节点不可见；版本号是唯一真相源，轮询有确定上限，逃生舱兜底 |
| **D9** | 工作区访问 | 统一走存储抽象 + 本机执行视图 | 业务代码只用 `WorkspaceStorage.root()/resolve()`；共享模式校验 `.workspace-id`，不匹配拒启动、绝不回退本地 |
| **D10** | 渠道长连接 | 单属主、永不互踢 | IM 平台对同渠道多连接的行为不一致（有的互踢到全部掉线）；属主租约只在过期后接管 |
| **D11** | 误配处理 | 集群档误配启动即拒（fail-closed） | 错的存储组合会让多实例静默读写各自的数据副本，故障在几天后发生且不可审计 |
| **D12** | 交付形态 | 官方 Helm Chart + 容器原生面 | 企业不是手搓 `docker run`；chart 自带探针、滚动升级、优雅停机、卷与密文引用 |
| **D13** | 跨副本可观测 | traceId 跨副本同源，审计行带 `instance_id` | 一次处理跨副本时，审计、日志、trace span 必须能拼回同一条链路 |
| **D14** | 边界 | 不做阶段三、不做多活、不做多主、不加中间件 | 每一条都等真实数据证明必要，写下来是为了评审时不必反复讨论 |

**D1 单机档零变化、集群档零共享假定。** 选择：档位由 `oryxos.cluster.enabled` 一个开关决定，默认关闭；集群档才装配协调 Bean，单机档的 Bean 图与阶段一逐点相同。理由：单机档是绝大多数试点部署的形态，为它引入"多副本才需要"的抽象是纯粹的负债；同时集群档必须假设"任何本地内存都不能作为真相源"，否则"任意请求落到任意副本都正确"这条不变量会被悄悄破坏。代价：两条代码路径需要**同一套参数化测试集**覆盖（见 20.4），否则集群档的缺陷要在多副本集成测试里才暴露。不这么做会怎样：要么单机档被协调逻辑污染（启动变慢、本地部署开始要求数据库），要么集群档在某个本地缓存上悄悄退化成"粘性单机"，两者都不可接受。

**D2 协调基质复用 PostgreSQL，不引入新中间件。** 选择：不引入 Redis、ZooKeeper、etcd、消息队列，协调原语全部落在 PostgreSQL 的普通表上。理由：OryxOS 已经支持 PostgreSQL 部署档（url 自动识别），集群档只是把"可选"变成"必需"；协调需要的四件事——条件写（CAS）、唯一约束（幂等键）、基于数据库时间的 TTL 判断、行级可见性——PostgreSQL 全都原生提供，而它本来就承担审计与会话落库，企业侧的高可用、备份、监控、账号体系已经存在。代价：数据库成为协调单点，可用性下限由数据库侧决定（见 17.5、18.2）；协调查询与业务查询共享连接池，需要显式配比（见 17.1）。不这么做会怎样：多一个有状态中间件就多一套必须做高可用、必须备份、必须监控、必须纳入审计的组件；而它带来的能力增量（Redis 的亚毫秒租约、etcd 的 watch 语义）在本方案的量级目标（2–8 副本）下都用不上。

> **对阶段一"未来分布式"设想的明确收敛。** `docs/oryxos-docs/IndustryResearch.md` 第 6.1 节曾设想会话与短期上下文外置 Redis、长期记忆与 Skill 库外置 PostgreSQL、审计与大文件外置对象存储、Agent 配置与租户信息外置配置中心；第 6.4 节进一步设想 Nacos 注册配置、Sentinel 限流熔断、Spring Cloud Gateway、SkyWalking；`docs/oryx-vs-hermes-分析报告.md` 也有"扩展阶段迁 Redis/PostgreSQL/对象存储，分布式协调留 Nacos/ETCD"的复述。**本方案主动收窄这个设想**：保留"状态外置"这个正确原则，去掉"每类状态配一个专用中间件"这个手段。三组对照如下——这也正是 D2 的正面论证。

**收敛对照一：会话与短期上下文不引 Redis。** "同一会话跨副本串行"需要的恰好是**同一个会话键上的一次条件写**——`session_turn_leases` 以 `session_id` 为主键的 CAS 认领正好是它。把会话搬到 Redis 会得到两套状态、两套一致性（Redis 会话 / PostgreSQL 租约与审计），两者无共同事务边界；而且 Redis 在这里并不省事——为了"Redis 说它持有锁"可信，仍要做租约 TTL、续约、过期判定，等于把 D4 的原语重写一遍，还多一份要独立做高可用与持久化策略的有状态组件。**结论：会话留在关系库，串行化复用同一张表上的租约。**

**收敛对照二：记忆 / Agent 配置 / Skill 等文件面不上对象存储。** 对象存储（S3 协议族）没有"同目录临时文件 + rename"的原子替换语义，也没有可预期的目录列举一致性，而 OryxOS 的文件面是**人来编辑、git 跟踪、模型直接按路径读**的数据。搬过去会同时丢掉三件事：原子落盘（D7）、按路径直读（Skill 正文经 `read_file` 现取是原则四的形态）、软连接绑定语义（原则四/六）。**结论：文件面只走共享卷，不依赖对象存储。** 审计与日志的**冷归档**可后续考虑对象存储，属成本优化，不改变运行期语义。

**收敛对照三：实例注册与配置传播不用 Nacos/ETCD。** **实例注册与健康**由 `instances` 表 + 心跳覆盖（第 9 章）；**配置与工作区变更传播**由 `workspace_versions` 版本号总线 + 轮询覆盖（第 11 章）——一个单调递增的版本号比对，不需要 push、长连接、可靠投递。ETCD 相对本方案的能力增量是 watch 与强一致的会话语义；本方案不需要 watch（D8 用轮询换取确定上限），也不需要跨集群一致（D14 不做多活），所以它增量最小而运维面最大。**结论：注册与配置传播各用一张已存在的表，不引注册中心。**

收口到一句话：**阶段二需要的协调语义（条件写、唯一约束、TTL、版本号、行级可见性）关系库都已具备。** 收敛的三条代价（数据库为协调单点、轮询放大读压力、共享卷语义弱于对象存储）已在 D2 与第 17 章写明；什么信号出现才值得引入新中间件，见 19.3。

**D3 恰好一次只承诺在五个粒度上。** 选择：会话轮次、定时触发、入站事件回执、渠道长连接属主、知识索引重建这五件事做跨副本恰好一次；审计落库、通知推送、工作区写盘等其余链路只承诺"至少一次 + 幂等键"。理由：恰好一次从来不是免费属性，它必须有唯一键和一张能落唯一键的表（或一个 CAS 原语）作为承载；没有唯一键可言的链路（比如"发一条 IM 回复"）只能靠上游去重，不能凭空承诺。代价：文档与运维手册必须逐条写清哪条链路是哪种语义，评审时容易被质问"为什么这里不是恰好一次"。不这么做会怎样：笼统宣传 exactly-once 会导致运维在真正需要精确性的地方（定时触发、轮次执行）失去判断依据，也会让实现者以为"重发一次也没关系"。

**D4 一切跨副本互斥走 CAS 单语句 + TTL 租约。** 选择：所有互斥都用一条带条件的写语句完成（见 4.3），失败即代表别人持有；持有时长由 TTL 表达，由持有方周期续约维持。理由：应用层"先查后写"在两个副本之间必然有竞态窗口（见 4.7 的时序反例）；`SELECT ... FOR UPDATE` 加长事务会把数据库连接长期占住，与 virtual thread 下"大量并发但短占用"的模型冲突，连接池会先于业务被打满。代价：TTL 意味着"崩溃后不是立刻可接管"，接管有一个由 TTL 决定的窗口（见 18.4 双属主窗口分析）；续约线程必须是专用调度线程且同步阻塞（见 4.5）。不这么做会怎样：读-改-写会在低概率下产生重复执行，且这种重复只在生产高峰出现，本地测试几乎复现不了。

**D5 定时任务幂等键取理论触发时刻。** 选择：`(schedule_id, fire_time)` 唯一，`fire_time` 是 `CronTrigger` 算出的理论触发时刻。理由：各副本用同一份 cron 表达式与同一定义时区计算，得到**同一个**理论时刻，这个值因此可以做唯一约束；墙钟时间做不到。代价：需要把阶段一 `AgentScheduler` 里依赖"当前时间"的判定逻辑改成显式接受一个由 `CronTrigger` 给出的时刻（见 6.5）。不这么做会怎样：三个副本各自认为"这次是我的"，唯一约束形同虚设，日报发三遍（墙钟反例的完整时序见 6.1）。

**D6 崩溃轮次标失败、不重放。** 选择：实例崩溃后留下的轮次由接管者**先标 `failed` 并写审计**，再释放轮次，用户重发后恢复。理由：崩溃时可能已经发生了外部副作用（已经调用过工单系统、已经发出部分消息），重放会制造重复副作用，而这类副作用往往不可撤销；标失败是唯一不会让企业侧数据变脏的语义，且它把"是否重来"的决策权交回给人。代价：用户感知到一次失败，需要重发；"自动恢复"这个看起来更友好的选项被明确放弃。不这么做会怎样：一个"重启服务重启工单"的 Agent 会在崩溃后重复建单，故障半径从 OryxOS 扩散到业务系统。

**D7 文件面只依赖"读写可见 + rename 原子"。** 选择：共享工作区的所有写入走"同目录临时文件 + rename"，所有变更感知走版本号总线，不依赖文件锁、不依赖 `WatchService`、不依赖 mmap 一致性。理由：NFS/CIFS/云盘 RWX 这几样能力的支持程度差异极大，有的"看起来支持"但在跨节点场景下静默失效；把设计假设收缩到最小公倍数，才能让同一个镜像在所有存储实现上都正确。代价：多文件发布**不是目录原子快照**（见 10.5），需要用写序清单和可重入发布来换；`fsync` 与吞吐之间要做取舍（见 10.4）。不这么做会怎样：在 NFS 上用一个"仅本地缓存生效"的文件锁，会得到"测试环境一直正确、生产环境偶发双写"这种最难查的故障。

**D8 变更感知走版本号总线 + 轮询。** 选择：管理写路径落盘成功后 bump `workspace_versions` 里对应域的版本号；各副本按 `oryxos.workspace.poll-interval`（默认 1s）轮询比对，落后即重载；单机档的 `WatchService` watcher 保持原样，零回归。理由：版本号是唯一真相源，比较是幂等且可重放的；轮询有确定的上限（传播延迟 ≤ 2× 轮询间隔），比"事件一定会到达"这种假设可靠。代价：变更不是即时传播，而是有一个可量化上限（P99 ≤ 3s，见 11.6）；轮询产生固定背景负载（见 17.2）。不这么做会怎样：共享卷上事件丢失意味着某些副本**永远**停在旧版本，且没有任何机制能发现。

**D9 工作区访问统一走存储抽象 + 本机执行视图。** 选择：业务代码只经 `WorkspaceStorage.root()/resolve()` 拿 `Path`，插件以 `WorkspaceStorageProvider.id()` 显式选择；需要真实文件系统路径的动作（文件 Tool、Shell Tool）必须调用 `nativePath`；共享模式启动先校验管理员预置的 `.workspace-id`，不匹配就拒启动，**绝不回退本地**。理由：回退本地会让每个副本写自己容器里的空工作区，产生数据分裂——这比启动失败严重得多，而且发现时往往已经过了几天。代价：部署流程多了一步"预置 `.workspace-id`"（见 3.3）；本地执行视图在共享卷下的映射需要额外校验（见 10.11）。不这么做会怎样：集群档下 `read_file`/`shell` 拿到的路径在本实例不存在，Agent 报一堆"文件不存在"，而写操作悄悄写到本地。

**D10 渠道长连接单属主、永不互踢。** 选择：每条渠道在任一时刻只允许一个活跃长连接，由 `channel_ownerships` 的租约仲裁；只在租约过期后才接管，且接管前先等旧连接走完停机链路（见 8.3）。理由：IM 平台对同渠道多连接的处理不一致，最坏情况是"新连接进来把旧连接踢掉，两边都掉线"，表现为消息间歇丢失；用租约而不是"谁先启动谁赢"才能覆盖崩溃场景。代价：接管要等 TTL（见 8.2 的取值关系），企业侧要接受秒级窗口；启动期多副本会互相竞争，需要抖动抑制（见 8.5）。不这么做会怎样：三个副本各自连一条飞书长连接，飞书侧按"最后连上的生效"，其余两条静默失效，负载均衡把消息路由到没有连接的那个副本时消息直接丢。

**D11 集群档误配启动即拒。** 选择：集群档遇到不支持的组合（**启动校验共 7 类**：SQLite 档、markdown 记忆后端、memory 知识后端、工作区模式非 `shared`、工作区标识缺失或不匹配、主密钥缺失或跨副本不一致、`instance-id` 缺失或重复）直接启动失败并给出恢复指引，不静默降级；**运行期另有一类 fail-closed**：协调基质（数据库）不可达时拒服务，不降级为单机（见 18.3）。完整矩阵见 3.2（11 项，其中 7 项属启动校验）。理由：静默降级的后果是"每个副本各写一份数据"，故障在数天后以数据不一致的形式爆发，且不可审计不可回滚；启动期失败是成本最低的失败。代价：运维需要读懂报错并按指引修正（见 3.2 恢复指引导向具体配置项）；不能靠"先跑起来再说"绕过。不这么做会怎样：生产上出现"用户 A 昨天说的话今天不记得了"，排查两周才发现是四个副本各自的 SQLite。

**D12 交付走官方 Helm Chart + 容器原生面。** 选择：提供 `charts/oryxos/` 官方 chart，集群档默认开启，工作区用 RWX PVC，探针 / 滚动升级 / 优雅停机 / 密文引用全部内置。理由：企业已经有一套 K8s 治理流程（Gatekeeper 策略、审计、变更单），交付物必须能被这套流程直接接管，而不是要求运维手搓 `docker run` 并自行补齐探针与停机参数。代价：chart 需要维护（`values.yaml` 的兼容性、模板渲染的验证链，见 16.8）。不这么做会怎样：每个企业各写一份自己的部署文件，探针与停机参数各错各的，把底座的设计意图丢掉。

**D13 跨副本可观测与审计同源。** 选择：一次处理跨副本时，审计行、结构化日志、可选 trace span 共用同一个 `traceId`，审计行额外带 `instance_id`，且三者使用同一份计时区间。理由：分布式下最贵的成本是排障时间；没有 `instance_id` 就无法回答"是哪个副本做的"和"这个副本下线时影响了多少工作"。代价：每张审计表加一列，所有写入路径都要带上实例标识；`SpanRecorder` 契约必须在 core 且默认 NOOP。不这么做会怎样：出现重复回复时无法判断是两个副本都跑了还是同一个副本跑了两遍，排障只能靠猜（见 14.6 的排障清单即为此设计）。

**D14 边界：不做阶段三、不做多活、不做多主、不加中间件。** 选择：跨节点 Agent 协作、多区域多活、多主数据库与分区容忍、跨表分布式事务、新中间件一律不做。理由：每一条都需要真实使用数据证明必要性；阶段二的核心命题是"单集群内语义自洽"，把命题做纯比做广更重要。代价：企业提出"两地三中心"这类需求时要明确说"不在本周期"。不这么做会怎样：范围蔓延会把一个可验证的里程碑拖成无法交付的框架。

### 1.2 技术栈增量

相对阶段一，阶段二的技术栈增量如下。**一句话原则：新增的都是"运行形态与交付形态"，不是"新的运行时依赖"。**

| 面 | 阶段一 | 阶段二增量 | 说明 |
|----|--------|-----------|------|
| 数据库 | SQLite（默认零配置）/ PostgreSQL（可选部署档） | 集群档 PostgreSQL **必需**，且承担协调基质 | url 仍自动识别；集群档下 SQLite 被启动校验拒绝（见 3.2） |
| 表结构管理 | Flyway 双轨 `db/migration/{vendor}/` | 新增 9 张集群档表**两轨各写一份**（同版本号、只增不改）；SQLite 轨保留结构等价的最小实现 | 迁移序列对称、两 vendor 表结构可对照；"是否允许启用"由集群档启动校验决定（见 13.1）；`hibernate.ddl-auto` 保持 `none` |
| 协调 | 无 | PostgreSQL 上的 CAS 认领原语 + TTL 租约 + 心跳 | 不引入 Redis/etcd/ZooKeeper/MQ |
| 文件面 | 本地目录 + `WatchService` 变更感知 | 共享卷（RWX）+ `WorkspaceStorage` 插件 + 版本号总线轮询 | 只依赖读写可见 + rename 原子 |
| 交付 | Dockerfile + compose + release 流水线 | 官方 Helm Chart（`charts/oryxos/`） | 探针、滚动升级、PDB、密文引用 |
| 可观测 | 结构化日志 + `trace_id` + 审计表 | `instance_id` 贯穿审计与日志；可选 OTel trace 导出 | `oryxos.otel.endpoint` 不配置则零开销 |
| 压测 | 无统一注入点 | mock provider 固定时延（`-Doryxos.mock.latency-ms`） | 把 LLM 侧不确定性从底座指标里排除 |

**明确不新增的东西：** Redis、ZooKeeper、etcd、消息队列、分布式事务框架、工作流引擎、服务网格、独立配置中心；也不引入 Reactor / WebFlux / `CompletableFuture`（宪法原则七）。理由集中在 D2 与 D14。

> **边界：** 1.2 列的是"技术栈增量"，不是"依赖增量承诺"。若阶段三引入 Agent 通信底座，其后端选型属于阶段三的设计范围，本文档不预留。

## 2. 整体架构

阶段二不改变阶段一的分层结构，而是在引擎层与基础层之间插入一个**共享状态层**，并把"触发源"和"渠道连接"从"每实例一份"改成"跨实例仲裁一份"。

### 2.1 单机档 vs 集群档对照

```text
【单机档 oryxos.cluster.enabled=false（默认，阶段一形态）】

      用户 / 业务系统 / 定时钟
              │
              ▼
      ┌───────────────────┐
      │   单个 OryxOS      │  ← TraceId、审计、Provider、Tool、Memory
      │   实例             │
      └───┬───────────┬───┘
          │           │
    ┌─────▼─────┐ ┌───▼──────────────┐
    │ SQLite    │ │ 本地 .oryxos/     │
    │ oryxos.db │ │ 工作区目录        │
    └───────────┘ └──────────────────┘
      进程内 Map 去重、进程内 ReentrantLock、WatchService 变更感知、本渠道唯一长连接
```

```text
【集群档 oryxos.cluster.enabled=true（阶段二）】

        负载均衡 / 网关（无粘性要求）
                 │
   ┌─────────────┼──────────────┬──────────────┐
   ▼             ▼              ▼              ▼
┌────────┐  ┌────────┐    ┌────────┐    ┌────────┐
│ 实例 1 │  │ 实例 2 │    │ 实例 3 │    │ 实例 N │  ← 完全对等，无主从
└───┬────┘  └───┬────┘    └───┬────┘    └───┬────┘
    │           │             │             │
    │  认领/续约/心跳/版本号轮询（PostgreSQL 语义）│
    └───────────┴──────┬──────┴─────────────┘
                       │
        ┌──────────────▼───────────────┐   ┌──────────────────────┐
        │ 共享状态层 A：PostgreSQL      │   │ 共享状态层 B：共享卷   │
        │  instances / 租约 / 认领 /    │   │  RWX：.oryxos/ 工作区  │
        │  去重 / 属主 / 版本号 / 代次  │   │  （.workspace-id 校验）│
        └──────────────────────────────┘   └──────────────────────┘
          渠道长连接：属主租约仲裁，任一时刻仅一个实例持有
```

**差异表：**

| 维度 | 单机档 | 集群档 |
|------|--------|--------|
| 状态位置 | 进程内 + 本地 SQLite + 本地目录 | 全部外置：PostgreSQL（关系型状态与协调）+ 共享卷（文件面） |
| 触发源（人推） | 本实例收到的 CLI / HTTP 请求 | 负载均衡分发到任意副本，请求不保证回到原副本 |
| 触发源（钟推） | 单实例的 `AgentScheduler` 直接执行 | 每副本各自扫描 cron，到点用 `(schedule_id, fire_time)` CAS 认领，只有一个执行 |
| 渠道连接 | 每渠道一条长连接，本进程持有 | 每渠道由 `channel_ownerships` 仲裁出唯一属主，其余副本不连 |
| 事件去重 | 进程内 Map | `event_receipts` 表（`(channel, event_id)` 唯一） |
| 会话并发 | 进程内 `ReentrantLock` 按会话维度 | `session_turn_leases`（`session_id` 唯一）+ 跨副本排队 |
| 文件面 | 本地目录，读写都直接 | 共享卷；写入经 `AtomicFiles`（临时文件 + rename）；本机执行取 `nativePath` |
| 变更感知 | `WatchService` 实时监听 | 版本号总线 + 轮询（默认 1s），集群档不装 watcher |
| 启动校验 | 配置格式与必填项校验 | 追加：数据库必须是 PostgreSQL、记忆/知识后端必须可共享、工作区必须带 `.workspace-id`、主密钥必须一致 → 否则拒启动 |
| 降级行为 | 无协调概念 | 数据库不可用即拒服务（fail-closed），不降级为单机 |

### 2.2 分层视图

```text
┌─ 接入层 ─────────────────────────────────────────────────────────────┐
│  CLI Channel │ Web Service（Spring MVC + virtual thread） │ 定时触发   │
│  渠道长连接（飞书 / 企微 / 钉钉）—— 由 ChannelOwnershipManager 仲裁   │
└───────────────────────────┬─────────────────────────────────────────┘
                            ▼
┌─ 引擎层 ─────────────────────────────────────────────────────────────┐
│  AgentService（统一入口，先认领会话轮次）│ ReActLoop │ PromptBuilder     │
│  ToolExecutor │ AgentScheduler（每副本扫描 + 到点认领）               │
└───────────────────────────┬─────────────────────────────────────────┘
                            ▼
┌─ 能力层 ─────────────────────────────────────────────────────────────┐
│  Provider（显式映射 + fallback）│ Memory │ 内置 9 Tool + MCP │ 知识库  │
│  人格库 │ Tool Policy 治理层 │ 通知推送                                │
└───────────────────────────┬─────────────────────────────────────────┘
                            ▼
┌─ 共享状态层（阶段二新增的核心层）───────────────────────────────────┐
│  A. PostgreSQL：instances │ session_turn_leases │ schedule_claims │    │
│     event_receipts │ channel_ownerships │ workspace_versions │         │
│     admin_write_reservations │ knowledge_build_claims │               │
│     knowledge_generations │ 审计表（+ instance_id）                   │
│  B. 共享卷：.oryxos/ 工作区（agents / skills / personas / knowledge /  │
│     bootstrap / memory / output / logs），经 WorkspaceStorage 抽象访问  │
└───────────────────────────┬─────────────────────────────────────────┘
                            ▼
┌─ 基础层 ─────────────────────────────────────────────────────────────┐
│  JDK 21 virtual thread │ Spring Boot 3.x │ Flyway 双轨 │ ConfigLoader   │
│  主密钥 AES-GCM（enc:v1:）│ 结构化日志 │ SpanRecorder（NOOP 默认）      │
└─────────────────────────────────────────────────────────────────────┘
```

**共享状态层是这一版唯一新增的层。** 它的意义不在于"加了数据库"，而在于它把四类原本存在于实例内部的状态搬到了外面：**"谁在做这件事"**（租约与认领）、**"这件事做过没有"**（幂等键与去重）、**"谁持有这个连接"**（属主）、**"文件面哪一版是当前版"**（版本号）。搬出去之后，实例就真的可以随时被杀掉、拉起来、再加一个。

### 2.3 一个请求在集群档的完整生命周期

以"用户在飞书群里 @机器人说一句话"为例，逐步标注用到哪些原语。

```text
飞书平台 ──消息事件──▶ 负载均衡 ──▶ 副本 B（本实例）
 1. 渠道长连接收事件
    └─ 前置条件：ChannelOwnershipManager 确认本实例是 channel=feishu:ops 的属主
       （属主租约心跳续约中；非属主副本根本没连这条长连接）
 2. 事件回执去重                                        【event_receipts】
    INSERT INTO event_receipts(channel, event_id, ...) ON CONFLICT DO NOTHING
    → 影响行数 = 0 ⇒ 平台重推，直接返回，不重复处理
 3. 路由到 Agent（ContextLoader 读工作区文件；文件面来自共享卷）
    └─ 本副本的已加载工作区版本 vs workspace_versions：已是最新（轮询线程维护）
 4. 认领会话轮次                                        【session_turn_leases】
    turn_seq = 当前 + 1（由唯一约束决定，不由本地自增）
    抢占/插入式认领成功 ⇒ 本副本持有该会话这一轮的租约
    认领失败 ⇒ 同会话已有轮次在跑 ⇒ 同步等待（上限可配）后重试
 5. ReAct 循环执行（同步阻塞、virtual thread 上跑）
    ├─ ProviderService 调 LLM（显式映射 + fallback）→ 写 llm_calls（带 instance_id、trace_id）
    ├─ ToolExecutor 执行 Tool（文件类 Tool 用 WorkspaceStorage.nativePath 取本机视图）
    │   → 写 tool_invocations（带 instance_id、trace_id）
    └─ 失败重试在 virtual thread 上同步退避
 6. 持久化会话（messages_json 追加）→ 释放轮次租约（released）
 7. 出站回复经渠道 Sender（过沙箱白名单）→ 飞书平台
 8. 可选：SpanRecorder 事后补记 turn / llm / tool 三个 span（不配置则 NOOP，零开销）
```

> **注意第 4 步是最关键的一步。** 阶段一"同一会话并发消息"靠进程内锁解决；集群档下同一条消息可能被负载均衡打到两个副本（平台重推、用户重发、健康检查重试都会造成），所以"这轮该谁跑"必须在共享状态层上决定，而不是在任何一个副本的内存里决定。

### 2.4 实例对等性的含义与推论

集群档下所有实例**完全对等**：没有主节点、没有协调者、没有"优先副本"。这一条不是修辞，它有三个可以直接检验的推论：

1. **任意请求落到任意副本都正确。** 会话轮次、定时触发、事件去重三个原语保证同一件事不会被执行两次；共享卷 + 版本号总线保证所有副本看到同一份文件面。因此负载均衡**不需要会话粘性**，也不需要按 Agent 做亲和路由。
2. **杀掉任意一个副本，系统在租约 TTL 内自愈。** 会话轮次、渠道属主、知识重建三类工作在 TTL 到期后被其他副本接管；`instances` 心跳过期只意味着"这个实例不再被信任"，不需要人工干预。
3. **加副本是水平扩容，不是重新配置。** 新副本只需要同一个 `ORYXOS_MASTER_KEY`、同一个数据库、同一个共享卷挂载、同一个 `.workspace-id` 校验通过，就可以直接承担流量；不需要通知其他副本，也不需要改任何既有副本的配置。

> **对等性的代价（诚实说明）**：因为没有主节点，就没有一个"知道全局"的地方——集群视图只能靠 `instances` 表拼出来（见 9.3），"当前有几个副本、谁在跑什么"永远是**最近一次心跳的近似值**而不是实时真相。这是本方案主动接受的取舍：中心协调者会引入选举、主节点故障、脑裂一整套问题，而它们带来的收益（精确的全局视图）对本方案的规模不值。

**本方案不做：** 主节点选举、租约"预投票/多数派"机制、副本角色分化（读副本/协调副本）、跨集群联邦。这些都是多活与阶段三的命题。

---

## 3. 集群开关与配置面

### 3.1 配置项表

所有集群相关配置挂在 `oryxos.cluster.*` 之下，默认值即"单机档语义"。**下表是四篇文档唯一的配置项与默认值来源**（对齐 README 第 6 章"统一默认参数"），不得各章另造一套值。代码标识符保留英文原名。

| 配置项 | 默认值 | 档位 | 含义与取值约束 |
|--------|--------|------|---------------|
| `oryxos.cluster.enabled` | `false` | 两档 | 唯一档位开关。`false` 时以下所有配置项不生效，协调 Bean 不装配 |
| `oryxos.cluster.instance-id` | 缺省由主机名 + 短随机后缀生成；K8s 下建议显式用 Pod 名 | 集群档 | 见下方生成规则；显式设置时必须全局唯一 |
| `oryxos.cluster.heartbeat-interval` | `10s` | 集群档 | 实例心跳写入间隔；必须显著小于 TTL |
| `oryxos.cluster.heartbeat-ttl` | `30s` | 集群档 | 心跳失效判定；建议 ≥ 3 × 心跳间隔 |
| `oryxos.cluster.session-turn-lease-ttl` | `60s` | 集群档 | 会话轮次租约 TTL；需 > 单轮 P99 耗时（含 LLM 与 Tool） |
| `oryxos.cluster.session-turn-lease-renew-interval` | `20s` | 集群档 | 轮次租约续约间隔；必须 < TTL 的 1/2 |
| `oryxos.cluster.session-turn-wait-timeout` | `120s` | 集群档 | 同会话已有在跑轮次时，后来者的同步等待上限；超时给出可读错误（见 5.3） |
| `oryxos.cluster.schedule-claim-ttl` | `120s` | 集群档 | 定时触发认领 TTL；短于任务最长执行时间即会被误判为崩溃（见 18.4） |
| `oryxos.cluster.channel-lease-ttl` | `30s` | 集群档 | 渠道属主租约 TTL |
| `oryxos.cluster.channel-lease-renew-interval` | `10s` | 集群档 | 渠道属主续约间隔 |
| `oryxos.cluster.knowledge-claim-ttl` | `30min` | 集群档 | 知识重建认领 TTL；重建耗时长，取值远大于其他租约 |
| `oryxos.cluster.knowledge-claim-renew-interval` | `5min` | 集群档 | 知识重建认领续约间隔 |
| `oryxos.cluster.event-receipt-ttl` | `24h` | 集群档 | 入站事件回执保留窗口；须 ≥ 渠道平台最大重推窗口（见 7.4） |
| `oryxos.cluster.claim-retry.max-attempts` | `3` | 集群档 | 认领语句同步重试次数上限（见 4.6） |
| `oryxos.cluster.claim-retry.backoff` / `-max` | `200ms` / `2s` | 集群档 | 重试退避初值与上限（指数增长） |
| `oryxos.cluster.event-receipt-purge-batch` | `1000` | 集群档 | 回执清理单批删除行数上限（见 7.2、17.3） |
| `oryxos.cluster.drain-timeout` | `40s` | 集群档 | 优雅下线排空在途工作的上限（对齐 Pod `terminationGracePeriodSeconds: 40s`） |
| `oryxos.cluster.start-jitter-max` | `3s` | 集群档 | 启动期随机抖动上限，抑制多副本同时抢属主（见 8.5） |
| `spring.datasource.url` | 空 | 集群档必需 | PostgreSQL 连接串；集群档下非 `jdbc:postgresql:` 直接拒启动 |
| `spring.datasource.username` / `.password` | 空 | 集群档必需 | 建议由 Secret → 环境变量注入，不落配置文件 |
| `ORYXOS_MASTER_KEY` | 空（缺省文件 `.oryxos/master.key`） | 两档 | 主密钥。集群档强烈建议用环境变量 / Secret 注入，见 15.1 |
| `oryxos.workspace.storage` | `local` | 两档 | `local`（本机目录）/ `shared`（共享卷）。集群档必须是 `shared` |
| `oryxos.workspace.id` | 共享卷根下 `.workspace-id` | 集群档 | 工作区标识的期望值；与卷上文件比对，不匹配拒启动（见 10.2） |
| `oryxos.workspace.poll-interval` | `1s` | 集群档 | 版本号轮询间隔。越小传播越快、背景负载越高；建议 `500ms ~ 5s` |
| `oryxos.workspace.fsync` | `false` | 两档 | 原子落盘是否 `fsync`；开启降低吞吐但增强掉电后一致性（见 10.4） |
| `oryxos.memory.backend` | `markdown` | 两档 | 集群档允许可共享档（如 `sqlite` 走共享库）；`markdown` 拒启动 |
| `oryxos.knowledge.backend` | `local` | 两档 | 集群档要求 chunk 存储落共享库（见 12.6）；`memory` 拒启动 |
| `oryxos.knowledge.keep-generations` | `2`（当前 + 上一代） | 集群档 | 保留的已提交知识代次数；切换后旧代次按此回收（见 12.5） |
| `oryxos.otel.endpoint` | 空 | 两档 | OTel trace 导出端点；不配置则 `SpanRecorder` 走 NOOP，零开销 |
| `spring.lifecycle.timeout-per-shutdown-phase` | `10s` | 两档 | 在途请求排空上限；Pod 侧 `terminationGracePeriodSeconds: 40s`（见 16.6） |
| 管理写互斥预约 | **不自动过期**，人工释放 | 集群档 | 无对应 TTL 配置项——这正是设计（见 10.8） |

**`instance-id` 生成规则。** 优先取显式配置；未显式配置时按 `主机名 + 短随机后缀` 生成，并在启动日志与 `GET /api/v1/info` 中回显。规则要点：

1. **同一实例重启后 ID 可以变化**（新进程即新实例），因此 `instance-id` 不能用来做"恢复上次状态"的依据——恢复靠租约 TTL 与幂等键，不靠 ID。
2. **绝不能出现两个进程共用同一个 `instance-id`**：`instances` 表以 `instance_id` 为主键，同 ID 的第二个实例会覆盖第一条心跳记录，导致运维视图与属主判定失真。K8s 下建议显式注入 Pod 名（StatefulSet 的 `metadata.name` 或 `metadata.uid`），不要用 Deployment 的随机 pod 名作为**稳定**标识去推断历史。
3. **派生值不参与任何唯一约束之外的语义**，只用于审计、日志、实例视图与"这条租约是谁持有的"。
4. **TTL 取值的三条硬约束**（三条一起看，任一条不满足都会产生"活着的持有者被误判为崩溃"）：续约间隔 < TTL；TTL > 单次工作的 P99 耗时；心跳 TTL ≥ 3 × 心跳间隔。定时触发与知识重建两个长任务的 TTL 明显大于其他租约，正是因为第三条约束在它们身上最紧。

### 3.2 配置校验与误配拒启动矩阵

集群档启动时按固定顺序做校验，任一项失败即**拒启动**（D11），退出码非零，错误信息必须同时给出"检测到什么"与"怎么恢复"。校验必须在**打开任何写路径之前**完成，避免"启动到一半才失败"留下半初始化状态。

| # | 误配组合 | 检测手段 | 报错要点 | 恢复指引 |
|---|---------|---------|---------|---------|
| 1 | 集群档 + SQLite | 数据源 url 前缀 / 驱动类型 | "集群档要求共享数据库，检测到 SQLite（`oryxos.db`）" | 切 PostgreSQL（见 3.3），或关闭 `oryxos.cluster.enabled` |
| 2 | 集群档 + 记忆后端 `markdown` | `oryxos.memory.backend` | "`MEMORY.md` 是本地文件，多副本无法共享记忆本体" | 改 `sqlite` 或可共享后端；或回退单机档 |
| 3 | 集群档 + 知识后端 `memory` | `oryxos.knowledge.backend` 能力声明 | "内存知识后端每副本一份索引，检索结果会随请求落到哪个副本而不同" | 切本地后端 + 共享 chunk 存储（见 12.6） |
| 4 | 集群档 + 工作区模式非 `shared` | `oryxos.workspace.storage` | "集群档要求工作区位于共享卷" | 挂载 RWX 卷并置 `storage: shared`（见 16.4） |
| 5 | 共享模式 + 缺 `.workspace-id` | 启动时读 `root/.workspace-id` | "共享工作区缺少标识文件，拒绝初始化" | 由管理员预置 `.workspace-id`（见 3.3 / 10.2），**不要**由实例自动创建 |
| 6 | 共享模式 + `.workspace-id` 不匹配 | 与配置/环境中的期望标识比对 | "本实例期望的工作区标识为 X，卷上为 Y：很可能挂到了别的集群的卷" | 修正挂载或配置；确认无误后由管理员显式改标识 |
| 7 | 集群档 + 主密钥缺失或各副本不一致 | 能解开库中既有 `enc:v1:` 密文才算一致 | "主密钥无法解开已有密文凭证（密钥不匹配）" | 统一注入同一 `ORYXOS_MASTER_KEY`（见 15.1）；**不要**让实例自动重新生成 |
| 8 | 集群档 + `instance-id` 缺失/重复 | 启动时 `INSERT` 心跳记录冲突检测 | "`instance-id` 已被另一活跃实例使用" | 显式设置唯一 ID（见 3.1），或修掉重复的手工覆盖 |
| 9 | 集群档 + 数据库不可达 | 启动时执行一次读校验查询 | "无法连接共享数据库：集群档不提供单机降级" | 修正连接 / 等待数据库恢复（见 18.3 fail-closed） |
| 10 | 集群档 + 共享卷不可写 | 在 `root` 下做一次探针级原子写读回（写保留路径后清理） | "共享卷不可写或未挂载" | 修权限/挂载；只读挂载不适用于集群档写者（见 15.3） |
| 11 | 集群档 + 工作区版本号表不存在 | 读 `workspace_versions` | "数据库缺少集群档迁移，请先执行迁移" | 让它跑 Flyway（见 13.3 迁移顺序） |

> **为什么校验必须"全查完再报"而不是"遇到第一个就退"：** 运维一次修一个错误的成本远高于一次看到全部问题。校验器应收集全部失败项，一次性输出，并在日志中标注每项对应的配置键名。

**本方案不做：** 运行期的"热切换档位"。档位只在启动时决定，运行中改 `oryxos.cluster.enabled` 不生效（也不报错，只记 WARN），因为它会改变 Bean 图与不变量。

### 3.3 从单机档迁移到集群档的运维步骤

迁移的核心纪律是**单机档代码路径在集群档下依然可用**，所以迁移不是"重写"，而是"换存储面 + 加协调"。全程按下列顺序执行，每一步都有校验点，校验不过就停在这一步回滚。

```text
步骤 0  宣布变更窗口，停止所有写入
        - 停掉单机实例（优雅停机，见 9.5），确认没有在跑的轮次与定时触发
        - 确认工作区没有管理写互斥预约残留（admin_write_reservations，见 10.8）

步骤 1  备份（两件一起做，作为回滚点）
        - 数据库：导出 oryxos.db（单机档）或现有 PostgreSQL schema
        - 工作区：整目录快照/打包 .oryxos/（含 agents、skills、personas、knowledge、记忆）

步骤 2  切库：把数据导入 PostgreSQL
        - 建库、建账号（最小权限：业务表 DML + 迁移表 DDL 分权，见 15.4）
        - 导入业务数据（sessions、tool_invocations、llm_calls、scheduled_tasks、
          task_executions、notify_channels、providers、tool_policy_rules、memory_entries 等）
        - 校验：行数、时间范围、抽样比对（尤其审计表）

步骤 3  执行 Flyway 迁移（postgresql 轨）
        - 追加 9 张集群档表与审计增量列（见 13.2）
        - 校验：flyway_schema_history 版本号连续、无失败记录

步骤 4  工作区上共享卷
        - 把 .oryxos/ 内容复制到 RWX 卷（保留相对软连接，不要 dereference）
        - 校验：把软连接解引用后仍能读到 SKILL.md；写一个临时文件并 rename 成功
        - 由管理员**预置** .workspace-id（内容为集群唯一标识，如集群名 + 随机串）

步骤 5  写集群配置
        - oryxos.cluster.enabled=true
        - oryxos.workspace.storage=shared、工作区根指向卷内路径、oryxos.workspace.id 设为集群标识
        - 数据库连接三键（Secret → 环境变量）
        - ORYXOS_MASTER_KEY（与单机档一致！否则已加密凭证解不开）
        - 记忆后端 / 知识后端改为可共享档

步骤 6  启动第一个副本
        - 逐项校验：误配矩阵（3.2）全过 → instances 表出现本实例心跳
        - 校验：GET /api/v1/info 回显 instance-id 与档位；GET /api/v1/instances 看到它

步骤 7  启动其余副本，逐项校验
        - 校验：instances 心跳齐全且互不覆盖；渠道属主唯一（见 8.2）
        - 校验：workspace_versions 六个域都有初始版本号

步骤 8  切流量并做业务校验（对齐 Demo E）
        - 单会话并发打多副本：无重复回复、无乱序
        - 定时任务：task_executions 每个 fire_time 恰好一行
        - 改一次 AGENT.md：其余副本 P99 ≤ 3s 生效
        - 审计：新写入的 tool_invocations / llm_calls 都带 instance_id
```

**回滚点：** 步骤 2/3 之间（数据库侧）、步骤 4 之后（工作区侧）。回滚时只回代码与配置、不回表结构（见 13.3），因为新增表与新列对单机档是惰性的。

**本方案不做：** 在线双向迁移（单机档与集群档同时读写同一份状态）、跨库实时同步、SQLite 与 PostgreSQL 之间的增量 CDC 工具。

## 4. 共享状态层与 CAS 认领原语

### 4.1 为什么是 PostgreSQL

**能力清单（逐项对应本方案的一个用法）：**

| PostgreSQL 能力 | 本方案用它做什么 | 替代品的代价 |
|----------------|----------------|-------------|
| 带条件的单语句写（CAS） | 所有认领：`INSERT ... ON CONFLICT DO NOTHING`、`UPDATE ... WHERE expires_at < ...` | 应用层读-改-写有竞态（见 4.7） |
| 唯一约束 | 幂等键：`(schedule_id, fire_time)`、`(channel, event_id)`、`(backend, scope)`、`session_id` | 没有唯一约束就必须自己写分布式锁，等于重造一遍 |
| 数据库时间源（`CURRENT_TIMESTAMP`） | **所有 TTL 判定**（见 4.3） | 用应用时钟会被 NTP/GC/漂移影响，各副本判定不一致 |
| 行级可见性与多语句事务 | 认领 + 审计写入在一个短事务里保持一致；"落盘 + bump 版本号"的补偿判定 | 无事务的存储会出现"认领成功但审计没写"这类空洞 |
| 单调自增与索引扫描 | 心跳列表、到期扫描、批量清理、指标聚合 | 需要额外组件才能做同样的查询 |
| 现成的高可用 / 备份 / 账号体系 | 企业侧已有 PG 高可用与备份流程可直接复用 | 每个新中间件都要把这四件事再做一遍 |

**为什么不用 advisory lock 作为主机制。** PostgreSQL 的 `pg_advisory_lock` 是**会话级**的，有三个与本方案冲突的性质：①锁没有"过期时间"概念，崩溃但连接还活着的持有者会被无限期 holding，而本方案所有正确性都依赖"持有者崩溃后 TTL 到期即可接管"；②持有状态不落表，**不可查询、不可审计**；③它把协调状态绑在会被复用、归还、回收的连接池连接上。

> **advisory lock 在本方案里的唯一位置：可选弱化辅助。** 允许把它当作"同一实例内多个线程抢同一个渠道时的本地短路优化"（先拿本地锁再走数据库 CAS，减少无意义的数据库往返），但它**绝不作为正确性依据**——即使 advisory lock 完全失效，正确性也必须由数据库表上的 CAS 独立保证。任何"靠 advisory lock 保证跨副本互斥"的写法都视为设计违规。

**本方案不做：** 用数据库做消息队列（轮询队列表分发工作）、用 `LISTEN/NOTIFY` 做变更推送（与 D8 的"版本号 + 轮询"重复且引入长连接依赖）、用两阶段提交做跨表业务原子性（D14 线外）。

### 4.2 表结构与约束

集群档 9 张表的字段级设计如下。字段命名严格对齐 README 第 5 章；表名、列名与状态枚举与 [DemandAnalysis.md](DemandAnalysis.md) 第 10 章一致（`knowledge_build_claims.state` 的具体取值集合由本章 12.3 定稿）。**所有 TTL 列一律以数据库时间为准**。

```text
instances                                  -- 实例注册与心跳（Ch.9）
  instance_id        VARCHAR  PK
  host               VARCHAR
  port               INT
  version            VARCHAR               -- OryxOS 版本，用于滚动升级期观察版本分布
  status             VARCHAR               -- starting | ready | draining | stale
  started_at         TIMESTAMP
  last_heartbeat_at  TIMESTAMP
  UNIQUE (instance_id)                     -- 主键即唯一；不另设业务唯一键
  INDEX (last_heartbeat_at)                -- 支撑"活跃实例列表"与过期扫描

session_turn_leases                        -- 会话轮次跨副本恰好一次（Ch.5）
  session_id         VARCHAR  PK           -- 同一时刻一个持有者（主键即互斥键）
  owner_instance_id  VARCHAR
  lease_id           VARCHAR               -- 每次认领新生成；续约/释放必须带它（栅栏令牌）
  turn_seq           BIGINT                -- 会话内单调递增轮次序
  state              VARCHAR               -- claimed | running | failed
  acquired_at        TIMESTAMP
  expires_at         TIMESTAMP
  last_error         TEXT
  INDEX (owner_instance_id, expires_at)    -- 支撑本实例续约扫描与接管扫描

schedule_claims                            -- 定时触发到点跨副本恰好一次（Ch.6）
  schedule_id        VARCHAR               -- 逻辑外键 → scheduled_tasks
  fire_time          TIMESTAMP             -- 理论触发时刻（D5）
  owner_instance_id  VARCHAR
  lease_id           VARCHAR               -- 认领凭据/栅栏令牌：续约与释放必须同时匹配 owner + lease_id
  state              VARCHAR               -- claimed | running | succeeded | failed | skipped
  claimed_at         TIMESTAMP
  expires_at         TIMESTAMP
  finished_at        TIMESTAMP
  error_message      TEXT
  PRIMARY KEY (schedule_id, fire_time)     -- 幂等键即主键
  INDEX (state, expires_at)                -- 支撑接管扫描

event_receipts                             -- 入站事件回执去重（Ch.7）
  channel            VARCHAR
  event_id           VARCHAR
  instance_id        VARCHAR               -- 首次处理的实例（排障用）
  received_at        TIMESTAMP
  expires_at         TIMESTAMP             -- received_at + event-receipt-ttl
  PRIMARY KEY (channel, event_id)
  INDEX (expires_at)                       -- 支撑 TTL 清理（必须是独立索引）

channel_ownerships                         -- 渠道长连接属主（Ch.8）
  channel_name       VARCHAR  PK
  owner_instance_id  VARCHAR
  lease_id           VARCHAR               -- 栅栏令牌，同 session_turn_leases
  acquired_at        TIMESTAMP
  expires_at         TIMESTAMP
  last_renew_at      TIMESTAMP

workspace_versions                         -- 版本号总线（Ch.11）
  domain             VARCHAR  PK           -- agents|skills|personas|knowledge|channels|bootstrap
  version            BIGINT                -- 单调递增，永不回退
  updated_at         TIMESTAMP
  updated_by         VARCHAR               -- 写路径标识或 instance_id

admin_write_reservations                   -- 管理写互斥预约（Ch.10），不自动过期
  scope              VARCHAR  PK
  owner_instance_id  VARCHAR
  reserved_at        TIMESTAMP
  note               TEXT                  -- 谁为什么事预约；人工恢复时必须读

knowledge_build_claims                     -- 知识索引重建认领（Ch.12）
  backend            VARCHAR
  scope              VARCHAR
  owner_instance_id  VARCHAR
  lease_id           VARCHAR
  state              VARCHAR               -- claimed | running | failed
  acquired_at        TIMESTAMP
  expires_at         TIMESTAMP
  PRIMARY KEY (backend, scope)
  INDEX (state, expires_at)

knowledge_generations                      -- 知识索引代次（Ch.12）
  generation_id      VARCHAR  PK
  backend            VARCHAR
  scope              VARCHAR
  state              VARCHAR               -- building | committed | retired
  chunk_count        BIGINT
  created_at         TIMESTAMP
  committed_at       TIMESTAMP
  UNIQUE (backend, scope) WHERE state='committed'   -- 部分唯一索引：一个作用域只有一个已提交代次
  INDEX (backend, scope, state)
```

**审计表增量（阶段五条原则的延伸）：**

| 表 | 新增列 | 语义 |
|----|--------|------|
| `tool_invocations` | `instance_id VARCHAR` | 执行这次 Tool 调用的实例；与 `trace_id` 一起构成跨副本追溯 |
| `llm_calls` | `instance_id VARCHAR` | 发起这次 LLM 调用的实例（带 fallback 的一次调用有多行，各自记录） |
| `task_executions` | `instance_id VARCHAR`、`fire_time TIMESTAMP` | 哪个实例执行了这次触发、其理论触发时刻；`fire_time` 是 Demo C 的判定依据 |

> **索引取舍原则（贯穿 9 张表）：** ①互斥键必须是主键或唯一约束（不是"普通索引 + 应用判断"）；②每个租约表的 `(state, expires_at)` 或 `(owner_instance_id, expires_at)` 必须有索引，否则过期扫描会全表扫；③TTL 清理列必须**独立建索引**（不能只作为复合索引的第二列），因为它是一个"范围删除"而不是"等值前缀查询"；④不加冗余索引——协调表都很小（副本数 × 会话数级别），索引过多反而拖慢写。

### 4.3 认领原语

集群档的一切互斥收敛到**两条 SQL 形态**。它们是本方案的"公理"，其余章节的机制都是这两条的实例化。

**形态一：插入式认领（`INSERT ... ON CONFLICT DO NOTHING`）。** 用于"从无到有"的认领，判定依据是**受影响行数**。

```sql
-- 语义：抢占（session_id, turn_seq）这个轮次位置；成功当且仅当返回行数 = 1
INSERT INTO session_turn_leases
       (session_id, owner_instance_id, lease_id, turn_seq, state, acquired_at, expires_at)
VALUES (:sessionId, :instanceId, :leaseId, :turnSeq, 'claimed',
        CURRENT_TIMESTAMP, CURRENT_TIMESTAMP + :ttlSeconds)
ON CONFLICT (session_id) DO NOTHING;
-- 返回行数 1 → 认领成功；0 → 别人持有该会话，本副本认领失败
```

```sql
-- 语义：回执去重 / 定时到点认领；成功当且仅当返回行数 = 1
INSERT INTO event_receipts (channel, event_id, instance_id, received_at, expires_at)
VALUES (:channel, :eventId, :instanceId, CURRENT_TIMESTAMP, CURRENT_TIMESTAMP + :ttlSeconds)
ON CONFLICT (channel, event_id) DO NOTHING;
```

**形态二：抢占式认领（`UPDATE ... WHERE expires_at < ...`）。** 用于"接管一个已过期持有者"的认领，判定依据同样是受影响行数。

```sql
-- 语义：接管过期的会话轮次租约；成功当且仅当返回行数 = 1
UPDATE session_turn_leases
   SET owner_instance_id = :instanceId,
       lease_id          = :newLeaseId,      -- 换令牌：旧持有者此后再续约/释放都无效
       state             = 'claimed',
       acquired_at       = CURRENT_TIMESTAMP,
       expires_at        = CURRENT_TIMESTAMP + :ttlSeconds
 WHERE session_id = :sessionId
   AND expires_at < CURRENT_TIMESTAMP;        -- 只抢已经过期的
```

```sql
-- 语义：渠道属主接管（Ch.8）与知识重建接管（Ch.12）用同一条形态
UPDATE channel_ownerships
   SET owner_instance_id = :instanceId, lease_id = :newLeaseId,
       acquired_at = CURRENT_TIMESTAMP, last_renew_at = CURRENT_TIMESTAMP,
       expires_at = CURRENT_TIMESTAMP + :ttlSeconds
 WHERE channel_name = :channelName
   AND expires_at < CURRENT_TIMESTAMP;
```

```sql
-- 续约：必须同时匹配 owner + lease_id（栅栏令牌），避免"已被接管者续约回来"
UPDATE session_turn_leases
   SET expires_at = CURRENT_TIMESTAMP + :ttlSeconds
 WHERE session_id = :sessionId
   AND owner_instance_id = :instanceId
   AND lease_id = :leaseId;
-- 返回行数 0 → 本副本已失去租约（被接管或记录被清理）→ 必须立刻自降级（见 4.5）
```

**三条判定规则，任一条不满足都视为实现错误：**

1. **一切判定以受影响行数为准，不以查询结果为准。** "先 `SELECT` 判断再 `INSERT/UPDATE`"就是 4.7 的竞态。
2. **一切时间比较用数据库时间（`CURRENT_TIMESTAMP`），绝不用应用时钟**（理由见下）。应用侧只出现"时长"（TTL 秒数）。
3. **一切续约/释放必须同时匹配 `owner_instance_id` 与 `lease_id`。** `lease_id` 是栅栏令牌（fencing token）：接管者换了令牌之后，旧持有者的续约与释放都会影响 0 行。

**为什么必须用数据库时间。** 多副本之间的时钟漂移没有上界保证，用应用时钟会让 TTL 判定出现两种对称的错误：时钟快的副本**提前接管**（双属主窗口被放大到漂移量级），时钟慢的副本**延迟接管**（与 README 的"租约过期后 ≤ 30s / ≤ 60s"目标冲突）。用数据库时间把判定收敛到**唯一一个时钟源**。副作用：`expires_at` 列不再由应用赋值，而由 SQL 表达式赋值——认领语句必须内联 TTL 计算。

**为什么不用 `SELECT ... FOR UPDATE` 长事务。** 见 D4（并发模型冲突的连接占用分析见 17.1）。

> **隔离级别口径：** 认领语句在 PostgreSQL 默认隔离级别（`READ COMMITTED`）下即可正确，因为它们都是**单语句**（`UPDATE ... WHERE` 对并发更新同一行会阻塞在行锁上，等前者提交后重新求值 `WHERE`，条件不再成立则影响 0 行）。**不需要**提升到 `REPEATABLE READ` 或 `SERIALIZABLE`。

### 4.4 租约生命周期

```text
                  ┌──────────────────────────────────────────────────┐
                  │                                                  │
                  ▼                                                  │
   [ 无租约 ] ──acquire──▶ [ 持有中 (claimed/running) ] ──renew──┐   │
       ▲                        │                                └───┘
       │                        │ 持有者停止续约（崩溃 / 挂起 / 隔离）
       │                        ▼
       │                  [ 已过期 (expires_at < now) ]
       │                        ├──takeover（另一实例 CAS 成功）──▶ [ 新持有者持有中 ]
       │                        └──purge（清理线程按保留窗口删除）
       └──────────────── release（正常完成 / 主动放弃）◀─────────────┘
```

| 转移 | 触发者 | 条件 | 结果 |
|------|--------|------|------|
| `acquire` | 任意实例 | 目标键上无记录或记录已过期 | 成为持有者，生成新 `lease_id` |
| `renew` | 持有者 | `owner_instance_id` + `lease_id` 匹配 | `expires_at` 延长；返回 0 行则自降级 |
| `expire` | 无（自然发生） | 数据库时间越过 `expires_at` | 记录仍在，但**不再被信任**（状态列不变） |
| `takeover` | 任意实例 | `expires_at < CURRENT_TIMESTAMP` 且 CAS 成功 | 换持有者与令牌；接管者按 D6 处理半成品 |
| `release` | 持有者 | 令牌匹配 | 按业务语义收尾（删行 / 置终态 / 置释放态） |

> **"过期"不是一个写操作。** 没有"过期线程"去改记录——过期是**读取侧对 `expires_at` 的解释**。这样做的好处是：即使某个副本完全卡死、清理线程根本没跑，其他副本依然能正确接管（因为判定只看时间，不看有没有人"宣布过期"）。

**两种收尾语义，按业务选：**

- **删除式释放**（`session_turn_leases`、`event_receipts` 之外的租约表）：任务完成后删掉记录，让下一个认领者能插入成功。
- **置终态式释放**（`schedule_claims` 的 `succeeded/failed/skipped`、`session_turn_leases` 的 `failed`）：记录保留，作为审计与幂等判定依据；"是否可再认领"由唯一键 + 状态共同决定。

**本方案不做：** 租约的"预投票 / 多数派确认"（不由持有者之外的角色确认持有）、租约的自动续期兜底（续约失败就是失败，不静默重试到无限）、多级租约（一个租约派生多个子租约）。

### 4.5 续约线程模型

**为什么必须专用调度线程同步阻塞续约。** 宪法原则七要求全程同步阻塞、不引入 Reactor/WebFlux/`CompletableFuture`。租约续约天然会被误写成"起一个定时任务异步续约"，那会立刻违反原则七。本方案的处理是：

```text
LeaseRenewer（每类租约一个实例，共用一个专用调度线程池）
  │
  ├─ 调度：固定频率 tick（如 1s）扫描"本实例持有的、需要续约的租约"
  │        （数据源：本实例内存中的持有集合，由 acquire 时登记）
  │
  ├─ 续约：对每一个到期的租约，在调度线程上**同步阻塞**地执行一条 UPDATE
  │        - 成功（1 行）→ 延长 expires_at，更新内存中的过期时刻
  │        - 失败（0 行）→ 本实例已失去该租约 ⇒ 登记到"失去集合"并置一个实例内的标志
  │
  └─ 通知：正在跑的 virtual thread 在**下一个检查点**（ReAct 每轮迭代开始、
           每次 Tool 调用前）读该标志；若失去租约 ⇒ 立即中止本轮、走自降级路径
```

关键设计点：

1. **续约线程是"调度线程"，不是"业务线程"。** 它不能调用 LLM、不能执行 Tool、不能读共享卷；它只做数据库往返。这样它的耗时可控（毫秒级），不会因为业务抖动而错过续约窗口。
2. **同步阻塞而非异步回调。** 续约结果在同一个线程上立即判定并落到一个实例内的标志位（`AtomicBoolean` / volatile 字段，这是队列与状态共享，不是异步编程模型）；业务侧在检查点读它。**不使用 Future/回调/事件总线。**
3. **续约失败的自降级行为（这是最容易漏的一条）**：续约返回 0 行意味着"我的租约已经被别人接管了"，此时**继续跑下去就是双写**。所以行为必须是：①立即把该租约移出续约集合；②置失去标志；③业务侧在下一个检查点中止；④**不再尝试释放**（释放必然也是 0 行，且不该去动别人的记录）；⑤记录一条 WARN 日志，带 `instance_id`、`lease_id`、租约键、traceId。
4. **心跳失败的自降级行为不同。** 实例心跳（第 9 章）写失败**不中止任何工作**——心跳只影响运维视图与"是否被信任"，不影响正确性。**租约续约失败 = 必须停；心跳失败 = 只告警。**
5. **续约集合是内存态、可重建。** 实例重启后续约集合为空，不影响正确性——旧租约到期后被接管即可。

**本方案不做：** 用 Quartz/XXL-Job 之类的调度框架承担续约、用响应式定时器、把续约与业务放在同一个线程（会被 LLM 的长阻塞拖垮续约）。

### 4.6 失败与重试

认领与续约语句的失败分三类，处理方式完全不同。**先分类，再决定重不重试**——把"所有失败都重试三次"是反模式，它会把"条件不成立"这种正常结果也变成重试，放大数据库压力。

| 类别 | 典型表现 | 是否重试 | 处理 |
|------|---------|---------|------|
| **认领失败（业务正常结果）** | 影响行数 = 0（别人持有 / 幂等键已存在） | **不重试**（排队语义由上层决定） | 返回"未获得"给调用方；会话轮次走排队（5.3）、定时触发本 tick 直接结束、回执去重直接返回 |
| **瞬时故障（可重试）** | 连接获取超时、连接被重置、死锁（`40P01`）、序列化失败、数据库主备切换 | **重试**，上限 `claim-retry.max-attempts`（默认 3），退避 `200ms → 2s` 指数增长 | 在同一调度/业务线程上同步退避重试；全部失败后按"数据库不可用"处理（见 18.3） |
| **永久错误（不可重试）** | 语法/权限错误（`42xxx`）、表不存在、约束定义不匹配、连接串指向非 PostgreSQL | **不重试**，立即失败 | 启动期发现即拒启动（3.2）；运行期发现即 fail-closed 并告警——这类错误不会自愈 |

**重试的边界：**

- **重试次数是全集群共享的一致口径**，不允许某类租约自己放宽（否则会出现"心跳重试 10 次把连接池占满"）。
- **重试退避发生在我方线程上，不占用数据库连接**（先释放连接再退避，再重新获取）。这一条对连接池容量至关重要，见 17.1。
- **死锁是正常现象而非故障**：PostgreSQL 会杀掉其中一个冲突事务，重试即可，不需要降低隔离级别来"避免"。
- **数据库主备切换期间的失败是成片的**：行为是"短暂停顿后恢复"，而不是"数据错乱"（见 18.3）。

### 4.7 为什么不用应用层"先查后写"

**竞态反例：两个副本同时处理同一会话的两条消息。**

```text
时刻   副本 A                                        副本 B
─────────────────────────────────────────────────────────────────────────────
t0     SELECT ... FROM session_turn_leases
       WHERE session_id='s1'  → 0 行
                                                     SELECT ... WHERE session_id='s1' → 0 行
       （A 认为"没人持有"）                          （B 也认为"没人持有"）
t1     INSERT INTO session_turn_leases
       (session_id='s1', turn_seq=5, owner='A')
                                                     INSERT INTO session_turn_leases
                                                     (session_id='s1', turn_seq=5, owner='B')
t2     → 成功（A 拿到）                              → 主键冲突，抛异常
─────────────────────────────────────────────────────────────────────────────
结果（有唯一约束，只是运气好）：A 跑，B 报错。B 的错误是一条"意外的技术异常"，
                              而不是一条可解释的"这个会话正在被别人处理，请稍等"。
─────────────────────────────────────────────────────────────────────────────
变体：如果 turn_seq 由 "SELECT MAX(turn_seq) + 1" 算出（而不是由唯一约束决定）
t0     A 算出 turn_seq = 5                           B 也算出 turn_seq = 5
t1     A 插入 (turn_seq=5)                            B 插入 (turn_seq=5)
t2     若唯一键是 (session_id, turn_seq) 而不是 session_id：两行都插入成功
       ⇒ A 与 B 同时执行第 5 轮 ⇒ 用户收到两条回复、会话历史里出现两个第 5 轮
       ⇒ 这就是"先查后写"在生产高峰造成的真实故障，本地压测几乎复现不了
```

**正确写法（一条语句，不留窗口）：**

```sql
-- turn_seq 由数据库在认领语句内部推进，不由应用计算
INSERT INTO session_turn_leases (session_id, owner_instance_id, lease_id, turn_seq, state, ...)
VALUES (:sessionId, :instanceId, :leaseId,
        (SELECT COALESCE(MAX(turn_seq), 0) + 1 FROM session_turn_leases WHERE session_id = :sessionId),
        'claimed', ...)
ON CONFLICT (session_id) DO NOTHING;
```

> **规则：应用层永远不做"检查—决策—写入"三步。** 应用层只做一件事：发出一条带条件的语句，看它影响了几行。所有"是否有人持有""是否已处理过""是否轮到我"的判断都发生在数据库的语句求值里。

**本方案不做：** 客户端侧的自旋锁、应用层的心跳表轮询抢锁、基于"最后写入时间"的乐观锁版本号（它在高并发下表现为大量失败重试，而 CAS + 唯一键没有这个问题）。

**边界：** 4.3–4.7 描述的是原语与通用纪律。每个具体场景（会话轮次、定时触发、回执去重、渠道属主、知识重建）在各自章节里说明**用哪种形态、幂等键是什么、收尾语义是什么**，不重复本节的论证。

---

## 5. 恰好一次（一）：会话轮次租约

### 5.1 问题

阶段一的会话并发控制是进程内 `ReentrantLock`（按任务/会话维度），代码里明确标注"非分布式锁"。集群档下它失效的方式有三种：

1. **同一会话的消息被分发到不同副本。** 用户连发两条、负载均衡轮流分发，或 IM 平台重推事件。两个副本各自认为自己独占该会话，于是**同一会话的第 5 轮被执行两遍**：用户收到两条回复、会话历史出现两条 assistant 消息、Tool 副作用发生两次。
2. **进程内锁不跨重启。** 副本 A 崩在轮次中途，新起的副本 B 对 A 的锁一无所知。
3. **进程内锁不提供"排队"语义。** 即使能保证"只有一个人跑"，也要回答"另一个人怎么办"——直接失败还是排队？这是产品决策，必须有明确设计。

> **本节的承诺边界（D3）：** 承诺的是"**同一个会话的同一个轮次在同一时刻只被一个实例执行**"。不承诺"用户消息绝不重复"——那取决于渠道平台是否重推，兜底是回执去重（第 7 章）。也不承诺"轮次一定能跑完"——崩溃轮次按 D6 标失败不重放。

### 5.2 轮次标识与状态机

**幂等键 = `session_id`（互斥维度）+ `turn_seq`（会话内序号）。** 注意 `session_turn_leases` 的主键是 `session_id`——这表示"同一时刻一个会话只有一个持有者"，而不是"一个会话只能跑一轮"。`turn_seq` 是持有者推进的会话内序号，用于审计与"这一轮是第几轮"的判定，不承担互斥职责。

```text
                     acquire (INSERT ... ON CONFLICT DO NOTHING)
                                  │
                                  ▼
                          ┌───────────────┐
                          │    claimed    │  已获得租约，尚未开始执行
                          └───────┬───────┘
                                  │ 开始 ReAct 循环
                                  ▼
                          ┌───────────────┐
        renew ──────────▶ │    running    │ ──────────▶ release（正常完成）
        （续约延长 TTL）    └───────┬───────┘              ⇒ 删除记录（下一个认领者可插入）
                                  │
                                  │ 执行失败（LLM 报错、Tool 失败、超时、中断）
                                  ▼
                          ┌───────────────┐
                          │    failed     │  记录 last_error；写审计；然后释放
                          └───────────────┘
                                  ▲
                                  │ 接管者发现过期租约，且原状态为 claimed/running
                                  │ ⇒ 先置 failed 并写审计（D6），再释放
                          ┌───────┴───────┐
                          │  takeover     │ ← 由另一实例发起（UPDATE ... WHERE expires_at < now）
                          └───────────────┘
```

**枚举口径（逐字对齐 README）：** `session_turn_leases.state` ∈ {`claimed`, `running`, `failed`}。这里**没有** `released`——正常完成是**删除记录**，"失败"才是一个保留的状态：保留是为了让运维与审计看到轮次失败过，而正常完成在 `sessions` 与审计表里已有完整记录。

**字段写入时机：**

| 时机 | 写入 |
|------|------|
| 认领成功 | `owner_instance_id`、`lease_id`、`turn_seq`、`state='claimed'`、`acquired_at`、`expires_at` |
| 进入循环 | `state='running'`（同一条 UPDATE，连同续约的 `expires_at`） |
| 续约 | `expires_at` 延长（每 `session-turn-lease-renew-interval` = 20s） |
| 失败 | `state='failed'`、`last_error`，随后释放 |
| 结束 | 删除该行（**当前持有者主动完成**）/ 释放记录让下一个人插入 |

> **`turn_seq` 的分配纪律：** 由认领语句内部的子查询推进（`COALESCE(MAX(turn_seq),0)+1`），**不由应用计算**。原因见 4.7 的变体反例：应用算 `MAX+1` 会在并发下算出相同的值。缺点是这个子查询让语句变长、且在 `session_id` 上有主键索引所以开销可接受。

### 5.3 跨副本排队语义

**判断：默认"后来者同步等待"，等待上限 `session-turn-wait-timeout = 120s`，超时给用户一条可读的话。** 为什么是等待而不是直接拒绝：

| 方案 | 用户体验 | 系统代价 | 结论 |
|------|---------|---------|------|
| 直接返回 409/429 | 用户在自己没做错事的情况下看到报错（他只是连发了两条） | 无 | **不采用** |
| 静默丢弃后来者 | 用户第二条消息石沉大海 | 无 | **绝不采用**（会丢用户消息） |
| 同步等待 + 上限（本方案） | 用户多等一个轮次的时间；通常几秒到几十秒 | 一个 virtual thread 阻塞等待（便宜） | **采用** |
| 排队进持久化队列 | 体验最好，但要引入队列语义、顺序保证、故障恢复 | 高（等于引入消息队列语义，见 D14） | 超出本周期 |

**等待的实现（严格同步，不引入异步）：**

```text
循环（直到获得租约或超出 :waitTimeout）：
  1. 尝试插入式认领（INSERT ... ON CONFLICT DO NOTHING）
     → 成功 ⇒ 退出循环，开始执行
     → 失败 ⇒ 继续
  2. 读一次当前租约的 expires_at（用于估算还要等多久，仅用于日志与提示语，不用于判定）
  3. 在**当前 virtual thread** 上同步 sleep 一个短间隔（如 500ms，带小幅抖动）
  4. 检查总等待是否超过 :waitTimeout
     → 超过 ⇒ 退出循环，返回可读错误
```

**超时后的用户可见行为：** 返回明确的"上一轮还在处理中"提示，并带上可诊断信息（不暴露内部细节）：

```text
HTTP 409 / 渠道回复文案：
  "这个会话的上一轮处理还没结束，请稍后再发（已等待 120s）。"
  日志与响应头携带 traceId，便于运维按 14.6 的清单排查。
```

**三条纪律：**

1. **等待不占用数据库连接。** sleep 期间连接必须已归还，否则 120s × N 个等待者会打满连接池（17.1）。
2. **等待是同步阻塞，不是异步回调。** 用 `Thread.sleep` 在 virtual thread 上等待，不引入 `ScheduledExecutorService` + Future、不引入事件通知。
3. **等待者不续约、不触碰租约**，只反复尝试认领。**不做"等待者检测到过期就直接改状态"**——那会与接管者的 CAS 竞争。

> **抖动（jitter）的必要性：** 多个等待者（同一会话被三条消息触发）如果以完全相同的间隔重试，会形成周期性冲突并持续失败。等待间隔应加 ±20% 随机抖动。这不是优化，是避免"惊群"式重试撞在同一毫秒。

### 5.4 崩溃接管时序

**这是 D6 落地的地方：接管者必须先标 `failed` 并写审计，再释放。** 顺序不能颠倒。

```text
实例 A（持有 s1 第 5 轮）              共享状态层              实例 B（接管者）
────────────────────────────────────────────────────────────────────────────────
acquire s1 → 成功
state='running'，expires_at=T+60s
   │
   ✗ 进程被 OOM kill（SIGKILL，无任何清理机会）
                                        租约行仍在
                                        owner='A', state='running',
                                        expires_at=T+60s（不再续约）
                                                              （B 的接管扫描每 tick 跑）
                                                              t > T+60s 发现：
                                                              UPDATE ... WHERE expires_at<now
                                                              → 影响 1 行 ⇒ B 是接管者
                                                              ★ ①标 failed、②写审计、③释放（删除该行）
                                                              B 不重放这一轮（D6）
用户重新发一条消息
   → B（或任意副本）认领成功 → 以新的 turn_seq 跑一轮 → 正常回复
```

**为什么必须先标 failed 再释放（顺序反了会怎样）：**

- 若**先释放后标 failed**：释放是删除记录，删完之后没有地方可以写 `failed` 了。结果是"崩溃轮次既没有痕迹也没有终态"，`sessions` 里留下半截 `messages_json`，审计与会话历史无法对应。
- 若**标 failed 但不写审计**：`session_turn_leases` 只有一行状态且很快被删除，无法回答"哪一轮失败过、哪个实例持有、traceId 是什么"。原则五要求审计 day one 落库，崩溃是最需要留痕的事件。

> **接管者不需要知道原持有者是否真的死了。** 它只知道"租约过期了"——**我们不判断对错，只判断时间**。原持有者可能只是被网络隔离或正在长时间 GC；租约过期意味着"不再被信任"，接管是安全的（双属主窗口见 18.4）。

**接管时间目标的构成（对齐 README：租约过期后 ≤ 60s）：**

| 阶段 | 时长 | 说明 |
|------|------|------|
| 崩溃发生 → 租约到期 | ≤ `session-turn-lease-ttl`（60s） | 最坏情况：崩溃发生在刚续约之后 |
| 租约到期 → 接管者发现并 CAS | ≤ 一个扫描 tick（建议 ≤ 5s） | 与心跳/扫描频率同量级 |
| 接管 → 用户可见恢复 | 用户重发即恢复 | **不自动重放**，所以这一段不由系统控制 |
| **合计（可观测部分）** | **≤ 65s 量级** | README 的 ≤ 60s 是"租约过期后"的口径，与上表一致 |

### 5.5 审计与回执留痕

**失败轮次的留痕清单（缺一条都算不合规）：**

| 落点 | 内容 | 目的 |
|------|------|------|
| `session_turn_leases` | `state='failed'`、`last_error`（接管者写入，注明"owner 失联"） | 协调层的即时可见性 |
| `tool_invocations` / `llm_calls` | 崩溃前已发生的调用**已经落库**（day one 写入的自然结果） | 回答"崩溃前跑到哪一步" |
| `sessions.messages_json` | 半截历史（用户消息在、助手回复缺） | 用户可见的真相；**不清理、不回滚**——回滚会掩盖曾发生过的事实 |
| 结构化日志 | 接管事件 WARN：`instance_id`、原 `owner_instance_id`、`lease_id`、`session_id`、`turn_seq`、`traceId` | 排障入口（14.6） |
| 指标 | 接管次数 +1、失败轮次数 +1（14.3） | 聚合视图与告警 |

**`traceId` 的关联规则（对齐 D13 与阶段一 `SpanRecorder` 契约）：** 崩溃轮次的原 `traceId` **不重新使用**。接管者写审计时记录两条关联线索：①原轮次的 `traceId`；②接管动作自身的 `traceId`。用户重发后的新一轮是**新的 traceId**，审计查询时可按 `session_id` + 时间区间把"崩溃的原轮次 → 接管 → 重发的新轮次"拼成一条故事线。**不做跨轮次的 traceId 继承**。

### 5.6 边界

- **多人群聊 / @ 触发的会话口径。** 会话标识沿用阶段一规则（`channel + user + profile` 联合生成，见阶段一技术方案第 9.2 节）。多人群聊里同一条消息可能同时 @ 到多个 Agent，此时**每个 Agent 是一个独立会话**（`session_id` 不同），互不排队——这正是期望行为：两个 Agent 各自回答一次是正确语义。但同一 Agent 在同群里被同一条消息重复触发（平台重推）会落到同一个 `session_id`，由回执去重（第 7 章）拦住，**不靠轮次租约拦**——轮次租约的语义是"串行"而不是"去重"，用错层会把应该排队的两条真实消息误判为重复。
- **与阶段一 `session_id` 生成规则的关系：** 完全不动。集群档不改变会话标识的生成，只改变"谁有权执行这个会话的下一轮"。这一点很重要：如果为了分布式而改 `session_id`，所有历史会话都会失联。
- **`turn_seq` 与 `max_history_turns` 无关。** `turn_seq` 是协调层的轮次序号，`max_history_turns` 是 prompt 组装时的历史截断窗口，两者语义不同、不互相推导。
- **不承诺 FIFO。** 三条消息同时到达时，谁先抢到租约谁先跑，**不保证按到达顺序执行**。真正的顺序保证需要持久化队列（超出本周期）。这是必须对业务方说清楚的边界：如果业务要求严格顺序，应在渠道侧或业务系统侧串行发送消息。

**本方案不做：** 跨副本的会话热迁移（把正在跑的轮次从 A 迁到 B）、轮次优先级队列、会话级并发度 > 1（同一会话允许两条消息并行，靠后续合并）——这三条都需要在会话之上再加一层并发模型，超出阶段二范围。

---

## 6. 恰好一次（二）：定时触发到点认领

### 6.1 幂等键取理论触发时刻（D5）

**核心事实：cron 的"到点"不是一个真实时刻，而是一个被算出来的刻度。** `CronTrigger` 给定（表达式、时区、上一个触发时刻）就会算出下一个触发时刻，这个值是**确定性的**——三个副本用同一份定义、同一个时区、同一个基准，算出的是**同一个值**。这个值就是幂等键。

**反例：如果幂等键取墙钟时间会怎样。**

```text
定义：每天 09:00:00 的日报任务，三副本集群

墙钟方案（错误）：幂等键 = (schedule_id, 触发时的"当前日期 + 当前小时分钟")
────────────────────────────────────────────────────────────────────────
副本 A：tick 在 09:00:00.031 触发 → 墙钟截断到分钟 = 09:00 → 键 (...)
副本 B：tick 在 09:00:00.114 触发 → 墙钟截断到分钟 = 09:00 → 键 (...)   ← 撞上了，
                                                                        只有 A 执行
副本 C：tick 在 08:59:59.998 触发 → 这一 tick 还没到点，跳过；
        下一 tick 在 09:00:01.002 → 墙钟截断到分钟 = 09:00 → 键 (...)   ← 又撞上，
                                                                        还是只有 A 执行
────────────────────────────────────────────────────────────────────────
看起来"撞上"是好事？不。看下一个变体：
────────────────────────────────────────────────────────────────────────
副本 A 的 tick 在 09:00:00.031 → 键 = 09:00
副本 B 的宿主机时钟比 A 快 61 秒（NTP 尚未收敛）→ B 的 tick 在
        实际 09:00:00.5 时它的墙钟显示 09:01:01 → 截断到分钟 = 09:01 → 键 = 09:01
        ⇒ B 用**不同的键**认领成功 ⇒ 同一时刻日报执行两遍
────────────────────────────────────────────────────────────────────────
更常见的变体：tick 间隔与分钟边界不对齐
副本 A tick 落在 09:00:29.9 和 09:01:00.1 —— 两次 tick 分别截断到 09:00 和 09:01
副本 B tick 落在 09:00:59.9 和 09:01:30.2 —— 截断到 09:00 和 09:01
⇒ 谁先谁后取决于 tick 相位，行为不可预测；再叠加"一天一次"的任务在
  夏令时切换日可能被跳过或执行两次
```

**正确方案：幂等键 = `(schedule_id, fire_time)`，`fire_time` 是 `CronTrigger` 算出的理论触发时刻。**

```text
副本 A：CronTrigger.next(lastFire=昨天09:00) → fire_time = 今天 09:00:00
        认领 INSERT (schedule_id, fire_time='今天09:00:00') → 影响 1 行 ⇒ 执行
副本 B：CronTrigger.next(lastFire=昨天09:00) → fire_time = 今天 09:00:00（同一个值！）
        认领 → 主键冲突，影响 0 行 ⇒ 不执行，本 tick 结束
副本 C：同上 ⇒ 不执行

即使三个副本的 tick 相位相差几秒、即使某个副本的墙钟慢 30 秒：
  - 早触发的副本算出的 fire_time 仍是 09:00:00 ⇒ 它执行，其余撞键
  - 晚触发的副本算出同一个 fire_time ⇒ 撞键，不执行
  - 唯一键保证"每个 fire_time 恰好一行 task_executions"（README 第 6 章口径）
```

**三条纪律：**

1. **`fire_time` 必须来自 `CronTrigger` 的计算，不能由任何"当前时间取整"的运算得到。** 时区必须来自任务定义（`scheduled_tasks.zone`），不能默认取宿主时区。
2. **同一定义的基准（`lastFire`）必须可从持久状态重建。** 副本重启后不能靠内存里的"上次触发"——它从 `scheduled_tasks.last_run_at` 或最近一条 `schedule_claims` 反推基准，保证算出的下一个 `fire_time` 与重启前一致。这是本项目里"状态外置"最容易被漏掉的一处。
3. **夏令时与闰秒按 `CronTrigger` 的既有语义处理**，不自行发明规则。在夏令时切换日，"不存在的那个小时"里的任务按触发器的行为可能跳过或顺延一次——这是**已知且有记录**的行为（写进运维手册），不是缺陷。

### 6.2 状态机与落库口径

```text
                 每个副本各自 tick（见 6.4）
                          │
                          │ CronTrigger 算出 fire_time
                          │ INSERT (schedule_id, fire_time, lease_id) ON CONFLICT DO NOTHING
                          ▼
                 ┌──────────────────┐
                 │     claimed      │  影响 1 行 ⇒ 本副本获得这次触发的执行权
                 └────────┬─────────┘
                          │ 开始执行
                          ▼
                 ┌──────────────────┐
                 │     running      │  续约延长 expires_at（TTL=120s）
                 └───┬──────┬───┬───┘
                     │      │   │
        正常完成 ────┘      │   └──── 到点时发现已过期/已跳过
                           │
                    执行失败（异常、超时、被中止）
                     │
                     ▼
        ┌────────────┐  ┌──────────┐  ┌───────────┐
        │ succeeded  │  │  failed  │  │  skipped  │
        └────────────┘  └──────────┘  └───────────┘
```

**枚举口径（逐字对齐 README）：** `schedule_claims.state` ∈ {`claimed`, `running`, `succeeded`, `failed`, `skipped`}。

**栅栏令牌（`lease_id`）的作用与"重新认领"的确切语义。** `schedule_claims` 也带 `lease_id`（与 `session_turn_leases` / `channel_ownerships` / `knowledge_build_claims` 同构）：续约与置终态必须同时匹配 `owner_instance_id` 与 `lease_id`。**接管不是插入新行，而是带新 `lease_id` 的 CAS 改写**——`(schedule_id, fire_time)` 是主键也是幂等键，同一个 `fire_time` 永远只有一行。这样做的意义在于：旧持有者即使从长暂停（GC/STW/网络隔离）中醒来，它那条"续约 / 置终态"语句只会影响 0 行，既不能延长自己的租约、也不能把接管者的 `running` 改成自己的 `failed`；于是 6.6 讨论的"长任务跨越多个周期"与 18.4 的双属主窗口都能被精确界定（详见 5.2 对同一形状的完整论证）。

**落库口径（两张表分工必须清楚，否则会出现重复或漏记）：**

| 表 | 承担 | 写入时机 | 幂等保证 |
|----|------|---------|---------|
| `schedule_claims` | **协调与幂等**：这次触发有没有人认领、谁认领的、什么状态 | 认领时插入（`claimed`），执行时置 `running`，结束置 `succeeded`/`failed`/`skipped` | 主键 `(schedule_id, fire_time)`，**这是"恰好一次"的唯一承载** |
| `task_executions` | **审计与观测**：这次触发实际跑了什么、跑了多久、成功失败 | **与状态置终态在同一个短事务里写入一行** | 无独立唯一键（依赖 `schedule_claims` 的幂等）——但可通过 `(schedule_id, fire_time)` 查询校验"每个 fire_time 恰好一行" |

> **为什么 `task_executions` 不自己做唯一约束？** 它是审计表，阶段一已经存在且有其他写入路径（手动补跑 `POST /api/v2/schedules/{id}/run` 也会写它）。给审计表加一个业务唯一键会把手动补跑也变成"每个 fire_time 只能一次"，而手动补跑在语义上**就是要**再跑一次。所以幂等性只在 `schedule_claims` 上表达，`task_executions` 用 `(schedule_id, fire_time)` 的可查性来支撑 Demo C 的断言。代价：如果有人绕过协调层直接写 `task_executions`，就绕过了幂等——这是运维纪律问题，不是设计缺陷，但要写进手册。

**`schedule_claims` 的清理：** 与 `event_receipts` 不同，`schedule_claims` **不按 TTL 删除**——它是"每个 fire_time 恰好一次"的长期证据，且行数增长有界（任务数 × 触发频率）。保留策略：按 `finished_at` 保留最近 N 天（可配，默认与业务审计口径一致），清理作业必须只在状态为终态（`succeeded`/`failed`/`skipped`）时删除，**绝不能删 `claimed`/`running`**（那会让正在跑的触发失去幂等保护，并在同一 `fire_time` 上被重新认领）。

### 6.3 misfire 策略

**判断：默认 `skipped`（不补跑），可配置为补跑一次。**

| 场景 | 默认行为 | 理由 |
|------|---------|------|
| 副本全部不可用，恢复了 3 小时（跨过 3 个触发点） | 这三个 `fire_time` 各写一行 `state='skipped'`，**不补跑** | 见下 |
| 单个副本 tick 卡住，某次触发到点时没有副本在扫（全集群都卡） | 同上 | 同上 |
| 任务执行中崩溃（认领 TTL 过期） | 接管者标 `failed`，**不重放**（与 D6 一致） | 重放可能重复副作用 |
| 任务被禁用（`enabled=false`）期间跨过的触发点 | 不写任何行（连 `skipped` 都不写），或写 `skipped`（可配） | 禁用是"用户主动关掉"，不是"错过"，不该产生噪音 |

**为什么默认不补跑（这是本节最重要的论证）：**

1. **补跑的语义是错误的。** 一个"每天早上 8 点播报今日天气"的任务，在下午 3 点恢复时补跑，会推一条"suggested clothing for today"到群里——这条消息在下午 3 点毫无意义，且用户会困惑"为什么现在推"。一个"每小时检查一次证书有效期"的任务，恢复后补跑 3 次，是把同一件检查做了 3 遍，产出的通知也是 3 条重复。
2. **补跑会放大故障。** 集群恢复的瞬间，所有错过的触发点会同时被认领并执行，形成一次"任务洪峰"，而此时系统刚刚恢复、数据库连接池还在预热。这个洪峰本身可能触发第二次故障。
3. **补跑需要一条"补偿决策"逻辑，而它的正确性无法通用。** 哪些任务可以补、补几次、补的时候要用哪个时间点的数据（"今天的天气"还是"错过那天的天气"），每个任务答案都不同。把这些差异硬塞进调度层就是把业务语义往底座里塞。
4. **不补跑的代价是可解释的。** 运维看到 `skipped` 行，可以人工判断"要不要手动补一次"——阶段一已有手动补跑端点（`POST /api/v2/schedules/{scheduleId}/run`），**人工决策**这条路径已经存在且语义清晰。

> **可选配置：** `misfire` 策略允许配为 `run-once`（恢复后对最近一个错过的 `fire_time` 补跑一次，更早的仍标 `skipped`）。这是一个显式选择，默认**关闭**，且必须在任务定义上写明理由。

**本方案不做：** misfire 的"按次数补跑"、"按 backfill 区间批量补跑"、任务级优先级与配额（并发上限治理）。

### 6.4 调度形态：每副本各自扫描 + 到点认领

**判断：不做调度中心单点。** 每个副本各自持有一套 `ThreadPoolTaskScheduler` + `CronTrigger`（沿用阶段一实现），各自独立扫描、独立计算 `fire_time`、独立尝试认领；**谁是执行者由数据库上的主键竞争决定，不由调度层指定。**

```text
        ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
        │   实例 A     │   │   实例 B     │   │   实例 C     │
        │ 扫描+算fire  │   │ 扫描+算fire  │   │ 扫描+算fire  │
        └──────┬───────┘   └──────┬───────┘   └──────┬───────┘
               │                  │                  │
               │  INSERT (s, 09:00) ON CONFLICT DO NOTHING
               └──────────────────┼──────────────────┘
                                  ▼
                     ┌──────────────────────────┐
                     │ schedule_claims          │
                     │ PK (schedule_id, fire)   │
                     └────────────┬─────────────┘
                                  │ 影响 1 行 → 该副本执行
                                  │ 影响 0 行 → 本 tick 结束，什么都不做
                                  ▼
                     执行 → 写 task_executions + 置终态
```

**对比：**

| 维度 | 调度中心单点 | 每副本扫描 + 到点认领（本方案） |
|------|-------------|-------------------------------|
| 新增组件 | 需要选主、需要中心服务、中心故障即全集群停调度 | 无（复用副本本身） |
| 副本增减 | 需要重新分配任务分片 | 不需要，加副本自动参与竞争 |
| 空转成本 | 低（只有中心在扫） | N 副本各扫一遍（开销模型见 17.2） |
| 调度精度 | 高（中心统一计时） | 取决于各副本 tick 相位，但**执行唯一性不受相位影响**（D5 保证） |
| 故障半径 | 中心挂 = 所有任务停 | 一个副本挂 = 其余副本继续竞争 |

**代价（必须诚实写明）：**

1. **N 倍空转。** 每个 `fire_time` 上，N−1 个副本的认领是"白跑"的数据库写（影响 0 行）。这是本方案的固有成本，量级估算见 17.2；在 8 副本、100 个任务的量级下是每秒个位数的语句，可接受。
2. **没有"到点精确性"的保证。** 认领成功的那个副本可能是 tick 相位最靠后的，所以实际开始执行的时间可能比 `fire_time` 晚一个 tick 间隔。这对"每天 9 点的日报"完全无所谓，但如果有任务要求"9:00:00.000 精确启动"，本方案不承诺。
3. **容量上限由数据库写竞争决定，不由调度器决定。** 如果任务数增长到每秒数百次触发，`schedule_claims` 的写入会成为热点（缓解手段见 17.3）。

**为什么"到点认领"而不是"分片归属"（每个副本负责一部分任务）：** 分片归属需要在副本增减时重新分片，而重新分片的瞬间正是最容易重复或漏执行的时候；且分片表本身就是一份需要维护的共享状态，等于在协调层里再造一个"谁是主"的问题。到点认领把"归属"变成一次即时竞争，副本增减对它完全透明。

### 6.5 与阶段一的关系与增量字段

**阶段一基线（不动）：** 任务**定义**来自 `AGENT.md` frontmatter 的 `schedules`（cron / zone / message）；运行态与历史在 `scheduled_tasks` / `task_executions`；`reconcile(profileName, key, ...)` 为同一配置任务复用稳定 `scheduleId`；启用/停用、立即执行、历史查询走 `/api/v2` 管理端点。集群档**不改这些语义**。

**增量：**

| 位置 | 增量 | 理由 |
|------|------|------|
| `task_executions.instance_id` | 新增列 | 回答"这次触发是哪个副本跑的"，与审计表的 `instance_id` 口径一致（D13） |
| `task_executions.fire_time` | 新增列 | Demo C 的断言依据："每个 `fire_time` 恰好一行"；同时是"这次执行对应哪个理论触发点"的唯一线索 |
| `scheduled_tasks` | **不加列** | 集群档不需要在任务定义侧记"属主"，归属是每次触发即时决定的（6.4） |
| `schedule_claims` | 新表 | 协调与幂等的承载 |
| `AgentScheduler` | 行为增量 | 触发路径从"直接调 `AgentService.process`"改为"先算 `fire_time` → 认领 → 成功才调"；`fire_time` 随调用链下传，供审计写入 |

**`fire_time` 的下传纪律：** `AgentScheduler` 在认领成功后，把 `fire_time` 与 `schedule_id` 作为**这次处理的上下文**（与 `traceId` 同层）下传到 `AgentService`，最终由审计写入路径记录到 `task_executions.fire_time`。**不允许审计侧"用当前时间近似"**——那会让 Demo C 的断言失去意义（两个副本的可能写入不同的 `fire_time`，就不再能验证"恰好一行"）。

**进程内锁的处置：** 阶段一每个任务一把进程内 `ReentrantLock` 防重叠。集群档下它**保留**，但角色从"正确性保证"降级为"本实例内的短路优化"（避免同一个副本对同一任务重复发起认领）。真正的重叠防护由"任务执行期间租约未过期 ⇒ 认领扫描跳过"承担：扫描时先看该 `schedule_id` 是否有 `state in ('claimed','running')` 且未过期的记录，有则本 tick 不产生新的认领（因为下一个 `fire_time` 本来也还没到；若已到，则由 TTL 决定是"长任务仍在跑"还是"崩溃了需接管"）。

### 6.6 边界

- **秒级 cron 不支持。** cron 表达式最细粒度是分钟（`CronTrigger` 的既有语义）。秒级触发会让 `schedule_claims` 的写竞争与每个副本的扫描负担同时上升一个数量级，而且在"日报/巡检"这类真实场景里没有需求。需要秒级频率的用例应改写为"轮询型 Agent"（Agent 自己 sleep 循环）或由外部系统通过 `POST /agents/{name}/invoke` 触发。
- **长任务跨越多个周期。** 若任务执行时间超过一个 cron 周期：下一个 `fire_time` 到点时，另一个副本会尝试认领并**成功**（因为唯一键不同）——即**同一任务可能重叠执行**。要避免重叠需满足两个条件之一：①任务执行时间 < `schedule-claim-ttl`（120s）且执行期间持续续约（默认行为），此时同一 `schedule_id` 上存在未过期的 `running` 记录，扫描侧据此**跳过**下一个 `fire_time`（写 `skipped`）；②任务定义声明"允许重叠"，此时显式并行。**默认是"不允许重叠、跳过后到的触发点"**，与阶段一"上次没跑完就跳过、不排队不并行"的语义一致——这是刻意的连续性，不是新决定。
- **任务被禁用 / 退役。** `enabled=false` 时调度器不产生新的认领（`skipped` 或静默，见 6.3）；`retired=true` 时既不再扫描也不再认领，但**历史 `schedule_claims` 与 `task_executions` 保留**（退役不删历史，沿用阶段一纪律）。若有 `claimed`/`running` 遗留记录（退役瞬间正在跑），由 TTL 自然过期后由接管者标 `failed`——退役不做特殊处理，避免"退役路径"成为一条没人测的代码分支。
- **不保证执行顺序。** 不同 `fire_time` 的触发之间**不保证按时间顺序执行**（例如 09:00 的触发因崩溃被接管、标 failed，而 10:00 的触发正常完成；此时 09:00 永远没有成功执行）。这是不补跑策略的直接推论，写进运维手册。

**本方案不做：** 跨副本的任务分片、任务优先级与并发配额、任务依赖（A 完成才跑 B）、misfire 的批量回填。

---

## 7. 恰好一次（三）：入站事件回执去重

### 7.1 阶段一进程内 Map 在集群档为什么失效

阶段一的入站去重是一个进程内 Map（渠道 + 事件 id → 已处理）。它在集群档下三个维度同时失效：

| 失效方式 | 具体场景 | 后果 |
|---------|---------|------|
| **不跨副本** | 平台重推同一事件，负载均衡把它分发到另一个副本 | 两个副本各处理一次 ⇒ 用户收到两条回复、Agent 被执行两次 |
| **不跨重启** | 副本 A 处理完事件、返回回执后立刻重启；平台因超时重推 | 新实例的 Map 是空的 ⇒ 重复处理 |
| **内存上限** | Map 无界增长（没有任何淘汰策略），跑几天后 OOM | 更糟：进程重启 ⇒ Map 清空 ⇒ 所有仍在重推窗口内的事件都会被重复处理 |
| **顺序不可靠** | "先查 Map 再写入"在同一个副本内也有竞态（两个线程同时查、同时写） | 单机档就在的隐患，只是概率低 |

> **关键判断：这一条不是"集群档新增的需求"，而是"阶段一的实现选择在集群档下不再成立"。** 阶段一的 `InboundMessageService` 已经把去重作为一个独立职责抽出，集群档要做的是**换掉它的存储**，而不是改它的语义。这也是 D1"集群档只打开协调原语"的一个具体体现。

### 7.2 落库去重表

```sql
-- 幂等键 = (channel, event_id)；主键即唯一约束
INSERT INTO event_receipts (channel, event_id, instance_id, received_at, expires_at)
VALUES (:channel, :eventId, :instanceId,
        CURRENT_TIMESTAMP, CURRENT_TIMESTAMP + :eventReceiptTtlSeconds)
ON CONFLICT (channel, event_id) DO NOTHING;
-- 影响 1 行 → 首次收到，继续处理
-- 影响 0 行 → 重复事件（重推 / 跨副本），直接返回"已收到"，不重复处理
```

**保留与清理：**

- **保留窗口 = `oryxos.cluster.event-receipt-ttl`（默认 `24h`）。** 这个值必须 **≥ 渠道平台的最大重推窗口**：如果平台可能在 72 小时后重推，而我们去重记录只留 24 小时，那么 24 小时后的重推会被当作新事件处理（去重表是唯一防线，见 7.4）。三种渠道的重推窗口不公开承诺，取值纪律是"按观察到的最长窗口 × 2 起算，宁长勿短"。
- **清理线程：** 每个副本各自跑（不需要认领——删除是幂等的，多个副本同时删不产生错误；`DELETE ... WHERE expires_at < ... LIMIT :purgeBatch` 的每次执行都是安全的）。**但要注意：多个副本同时跑同一个清理会互相抢行锁**，所以清理**只在"随机抖动后"执行，且单批行数有上限**（默认 1000），并通过指标观测删除速率（见 17.3）。
- **清理不能删"未过期"记录**，也不能用"最近 N 条"代替 TTL（那会随事件速率变化而改变窗口语义）。
- **膨胀估算与容量：** 单行约 100 字节（含索引），24h 窗口下事件速率 10 条/秒 ⇒ 约 86 万行、约 90MB。这是可接受量级，但**必须设一个"表行数上限"告警**（见 14.3），因为速率失控（例如某个渠道被刷）会让它线性增长。

### 7.3 处理失败时的回执语义

**判断：入站事件处理失败时，删除回执记录（允许平台重推）；但有一个例外——被判定为"已产生部分外部副作用"的失败保留回执。**

这是本节最需要拍板的地方，两个方向都有真实代价：

| 方案 | 语义 | 优点 | 代价 |
|------|------|------|------|
| **失败即删除回执**（允许重推） | 平台重推 = 第二次尝试 | 消息不丢：一次瞬时故障（LLM 超时、工具超时）后平台重推能救回来 | 若第一次已经产生了部分副作用（部分回复已发出），重推会再走一遍 |
| **失败保留回执**（不重推） | 一次失败 = 永久失败 | 不会重复副作用 | **消息静默丢失**：用户没收到任何回复，也没有人知道 |

**本方案的选择与理由：**

```text
处理流程：
  1. 认领回执（INSERT ... ON CONFLICT DO NOTHING）→ 0 行 ⇒ 直接返回
  2. 执行处理（ReAct 循环）
  3. 结果分支：
     ├─ 成功 ⇒ 保留回执（去重生效，平台再推也不会重跑）
     ├─ 失败于"尚未产生任何外部动作"（LLM 首轮调用失败、模型限流、
     │   上下文组装失败）⇒ **删除回执**，返回非 2xx 给平台 ⇒ 平台重推 ⇒ 重新尝试
     └─ 失败于"已产生外部动作之后"（例如回复已分段发出第一段、
         通知已推送、工单已创建）⇒ **保留回执**，写审计 + 告警
         ⇒ 不重推（避免重复副作用），由人工介入
```

**判定"是否已产生外部动作"的手段：** 不看模型输出、不做语义判断，只看**审计事实**——本次处理在 `tool_invocations` 里是否已有成功记录（`success=true`），或渠道 Sender 是否已成功发出至少一段。这是可用且廉价的判据（审计表 day one 落库的红利）。

> **诚实的边界：** 这个判定不是万无一失的。"第一段回复已发出但发送 SDK 返回失败"这种模糊情况无法从审计区分。此时选择**保留回执**（宁可少一次回复，不要两次），并在告警里明确标注"疑似部分发送"——让人来判断，而不是让系统猜。

**返回码的语义（对渠道适配器）：** 删除回执后必须让平台**知道可以重推**：webhook 类渠道返回 5xx；长连接类渠道（飞书/企微/钉钉的 Stream 长连接）没有"告诉平台失败"的通道，此时重推依赖平台自身的超时检测。这是本方案的一个已知弱点，必须在 7.4 写清。

### 7.4 与渠道平台重推行为的关系

**判断：去重表是唯一防线；属主机制只保证"同一时刻一条连接"，不保证"消息只到达一次"。**

```text
渠道平台 ──(1) 事件 e1──▶ 属主副本 A：认领成功 → 处理 → 回复 → 保留回执
           │
           │  A 的回复在路上（或 A 未能及时 ack）
           │
           ├──(2) 平台因未收到 ack 重推 e1 ──▶ 属主副本 A 或新属主副本 B
           │                                   认领 → 0 行 ⇒ 直接返回，不重复处理 ✓
           │
           └──(3) 平台因连接抖动重推 e1 ──▶ 任意副本
                                             找所属主？不。认领 → 0 行 ⇒ 直接返回 ✓
```

**四条必须写清的语义：**

1. **去重与属主是两层独立的保护。** 属主（第 8 章）解决"同一时刻只有一个活跃长连接"；回执去重解决"同一条事件只处理一次"。**属主不是去重的替代品**：即使属主完美工作，平台的重推仍会到达（属主连接会收到平台重推的事件），此时只有回执去重能拦住。
2. **平台重推的触发条件我们无法控制。** 各平台在"多久没 ack 就重推""重推几次""重推间隔"上的行为不同且不公开。因此去重窗口（24h）与"是否存在跨副本事件 id 冲突"是必须实测的两件事（见 7.5）；平台侧的重推行为在 [DemandAnalysis.md](DemandAnalysis.md) 第 5 章有需求侧描述，本文档只负责"我们如何处理重推"。
3. **长连接渠道没有显式的失败反馈通道。** 飞书/企微/钉钉通过 Stream 长连接推事件，事件的"已接收"通常由 SDK 在收到后即返回，而不是等业务处理完。这意味着**业务处理失败时平台不会重推**（它认为事件已被接收）。所以对长连接渠道，"失败即删除回执以允许重推"这条路径**实际上不会发生**——重推只来自平台自身的超时重试。结论：**长连接渠道的一次瞬时失败就是一次真实的用户可见失败**（用户没收到回复），这是本周期接受的代价，缓解手段是审计 + 告警 + 用户重发。
4. **重复处理的唯一兜底是回执去重，因此它不能"为了性能"而弱化。** 任何"先查内存缓存、缓存没有再查库、缓存更新失败也放行"的优化都会在这里制造重复——因为去重失效的直接后果就是用户看到两条回复。若确需缓存，必须是"缓存命中 ⇒ 拒绝（保守）"，而绝不能是"缓存未命中 ⇒ 放行"。

### 7.5 边界

- **保留窗口的取舍。** 窗口越长越安全但表越大；窗口越短越省空间但可能让延迟重推变成新事件。默认 24h，按平台实测调整，且**每次调整都要记入运维手册**。
- **去重表膨胀。** 三个因素会放大它：事件速率、保留窗口、以及**事件 id 的基数**（有的平台对同一条消息的多次重推使用不同的 `event_id`，此时去重表**完全失效**，只能靠业务侧的其他幂等手段）。第三种是**必须在接入新渠道时实测的前置条件**，写进渠道接入清单。
- **跨渠道事件 id 冲突。** 不同渠道的 `event_id` **可能相同**（尤其当它们都是某种全局 UUID 或数字 id 时）。因此唯一键必须是 `(channel, event_id)` 而不是 `event_id` 单列——这不是防御性设计，是必须的：不同渠道的 id 空间没有任何协调，冲突是正常现象而非异常。
- **同一用户的多条真实消息必须都能处理。** 去重键是"事件 id"而不是"内容哈希"或"用户 + 内容"——用户连发两条一样的话是两条事件，必须各回复一次。这一条排除了任何"内容去重"的实现。
- **去重表不做"处理结果"缓存。** 它只记录"这个事件来过"，不记录"回复是什么"。后者看起来可以省一次处理（幂等重放），但会把"回复内容"变成一个需要版本化、需要过期、需要权限管理的第二份状态，超出本周期。

**本方案不做：** 事件的持久化队列（确保"一定能处理完"）、跨渠道的统一事件 id 归一化、重复事件的自动合并回复。

---

## 8. 渠道长连接属主

### 8.1 问题（D10）

阶段一每条渠道在本进程内建一条长连接，天然唯一（一个进程嘛）。集群档下如果每个副本都连，会发生什么取决于平台：

| 平台行为类别 | 表现 | 对本方案的要求 |
|-------------|------|---------------|
| **互踢型**（新连接踢掉旧的） | 每次滚动升级/扩缩容都会把现有连接踢断；最坏情况是"新连接进来 → 旧连接被踢 → 新连接因某原因失败 → 两边都没有连接" | 必须保证**任一时刻只有一个活跃连接**，且接管有确定顺序 |
| **并存型**（同渠道允许多连接，事件随机分发） | 同一事件可能被两个连接各收到一次（平台的保证是"至少一次"） | 属主只能减少概率，最终由回执去重兜底 |
| **拒绝型**（同渠道第二连接直接被平台拒绝） | 副本 B 的连接建立失败 → 该副本需要识别"这是正常的，因为我不是属主"而不是报错重试风暴 | 非属主副本**根本不应该尝试建连**（这是属主机制的主要价值） |

**判断（D10）：以属主租约为准，任一时刻只有一个副本持有某渠道的长连接，永不互踢。** 三种平台行为下这个设计都成立：互踢型不会发生（只有一条连接）、并存型概率大幅降低（连接唯一）、拒绝型不会触发（非属主不连）。

### 8.2 属主租约与心跳续约

```text
channel_ownerships（主键 channel_name，即互斥键）
  owner_instance_id  当前属主
  lease_id           栅栏令牌（接管时更换）
  acquired_at        何时拿到
  expires_at         何时过期（= last_renew_at + channel-lease-ttl）
  last_renew_at      最后一次续约时刻（数据库时间）
```

**取值关系（`channel-lease-ttl = 30s`、`channel-lease-renew-interval = 10s`）：**

| 关系 | 值 | 理由 |
|------|----|------|
| 续约间隔 / TTL | 10s / 30s = 1/3 | 允许**连续两次续约失败**（网络抖动、数据库短暂切换）才过期；若只有 1/2 则一次抖动就过期，接管过于激进 |
| TTL 与"接管 ≤ 30s"目标 | TTL = 30s | 接管时间 ≈ 剩余 TTL（最坏 30s）+ 检测间隔（秒级）⇒ 与 README"租约过期后 ≤ 30s"的口径一致 |
| 续约间隔与心跳间隔 | 10s（属主）vs 10s（实例心跳） | 同频，避免两个线程错峰叠加成额外负载；两者合并到同一个调度 tick 里执行 |

**属主与实例心跳的关系（容易混淆，必须写清）：** 属主租约与 `instances` 心跳是**两件独立的事**。心跳回答"这个实例还活着吗"；属主租约回答"这条渠道归谁"。一个实例可以心跳正常（`ready`）但不是任何渠道的属主（这是常态：3 副本、2 个渠道 → 只有一个渠道属主）。**不要用"实例心跳过期"来推断"渠道属主该换"**——那会把渠道接管时间绑到心跳 TTL（30s）上，虽然数值相同但语义耦合会在调整任一个参数时制造意外。

**非属主副本的行为：** 不建连、不尝试建连。它只做一件事：**定期尝试接管已过期的租约**。这让"3 副本只有 1 个在连渠道"成为默认稳态，也让"属主崩溃后其余副本竞争、恰好一个成功"成为自然结果。

### 8.3 接管时序

```text
t0   实例 A 持有 feishu:ops 属主，长连接活跃，每 10s 续约
t1   A 被 kill -9（或网络隔离 / 宿主机暂停）
     └─ 长连接由平台侧/OS 侧检测断开（时间不可控）
t2   t1 + 最多 30s：租约到期（expires_at 已过，记录仍在）
t3   实例 B 的接管扫描 tick：
       UPDATE channel_ownerships SET owner='B', lease_id=new, expires_at=now+30s
        WHERE channel_name='feishu:ops' AND expires_at < CURRENT_TIMESTAMP
       → 影响 1 行 ⇒ B 成为新属主
t4   ★ 抖动等待（jitter，如 0–2s）：B 不立刻建连
       目的：若 A 其实没死（只是网络分区后又恢复），
             给 A 一个机会在它的下一个续约周期上失败并自行收尾；
             同时避免"多个副本在同一毫秒竞争 → 失败者立刻重试"的惊群
t5   ★ 旧属主收尾检查：B 确认自己不是"从 A 手里抢来的热连接"——
       实际上 B 无法检查 A，所以这一步的语义是：
       "B 在接管前不假设 A 已停止；B 的建连必须是幂等的、
         且失败时可安全丢弃"（连接建立失败 ⇒ B 不释放租约 ⇒
         下一个周期继续尝试，而不是放弃）
t6   B 建立长连接；建立成功后才开始处理事件
t7   A 若还活着：它的下一次续约返回 0 行（lease_id 已变）
       ⇒ 按 4.5 第 3 条自降级：停止续约、置失去标志、
         业务侧在检查点中止、不再释放租约、WARN 日志
```

**"接管前的等待与抖动抑制"为什么必要（三条独立理由）：**

1. **避免"旧属主还在，新属主已连"的双属主窗口被无谓拉长。** 抖动只有 0–2s，但足以让"其实没死的旧属主"在下一次续约时发现自己失去了租约并主动断开——把窗口从"TTL + 建连时间"缩短到"TTL + 抖动 + 建连时间"里的抖动部分不增加，但**旧连接主动断开**这件事显著降低平台侧的双连接时间。
2. **避免惊群。** 3 个副本同时发现租约过期，如果都立刻建连，平台侧会看到 3 个连接尝试（互踢型平台会踢 2 次，拒绝型平台会有 2 个报错）。抖动 + CAS 保证只有 1 个成为属主，但**失败者不应立刻重试**——它们要等到下一个扫描周期。
3. **给"平台侧的旧连接超时"留出时间。** 有的平台在检测到 TCP 断开前需要几十秒；如果新连接立刻建立，平台可能因"同渠道连接数超限"拒绝新连接。抖动几十秒在这个尺度上帮助有限，但配合"建连失败 ⇒ 保持属主资格 ⇒ 下个周期重试"（不释放租约）就能自然收敛——这是比"等待固定时长"更健壮的做法。

> **建连失败不释放租约，这一条要单独记。** 反直觉但正确：如果我刚拿到属主就释放（因为建连失败），那么下一个副本会立刻接管并也建连失败，形成"属主在副本间乒乓"；而持有租约 + 重试会把"平台侧暂时不可用"变成"我们这边安静地重试"，且不给其他副本制造无谓的接管。

### 8.4 停机链路

**顺序是硬性的：先释放属主，再关连接。** 反过来的后果：先关连接、再释放属主，中间有窗口期内"属主仍是我、但我已经没有连接"——这些事件会发到我这里然后无处可去（或者更糟：平台把事件推给一个正在关闭的进程）。而先释放属主则让"下一个属主接管"与"我关闭连接"可以并行，且**接管者不必等 TTL**（这是 `ChannelAdminService.stopAll` 主动释放的价值：把秒级的接管从"等 TTL"变成"立即"）。

```text
优雅停机（SIGTERM）链路的顺序：

 1. spring.lifecycle：进入 draining
    ├─ 从负载均衡摘流（K8s：readiness probe 失败 → 端点摘除）
    └─ instances.status = 'draining'（实例视图立刻可见，见 9.3）

 2. 停止认领新工作
    ├─ 调度扫描停止（不再产生新的 schedule_claims 认领）
    ├─ 会话轮次：拒绝新的认领（但不影响已在跑的轮次）
    └─ 入站渠道：长连接上的新事件**仍然会被收到**（连接还在），
        但处理路径会检查 draining 标志 ⇒ 返回"请重试"让平台重推

 3. 排空在途工作（上限 drain-timeout = 40s，对齐 grace period）
    ├─ 等待已在跑的会话轮次结束（它们在续约，属主资格仍在）
    └─ 等待已认领的定时触发结束

 4. ★ 释放渠道属主（ChannelAdminService.stopAll 的第一件事）
    ├─ 对每个本实例持有的渠道：DELETE FROM channel_ownerships WHERE
    │    channel_name=? AND owner_instance_id=? AND lease_id=?
    └─ 成功后，其他副本可以在**下一个扫描 tick（秒级）**接管，
        不需要等 30s TTL

 5. 关闭长连接（限时 + 守护线程）
    ├─ 每个渠道适配器的 close 必须限时（如 2s）
    ├─ 若 SDK 的 close 阻塞不返回 ⇒ 放守护线程、限时 join 后放弃等待
    │   （阶段一已有先例：飞书 SDK ws.Client.close() 存在阻塞路径，
    │     曾把停机拖满 ~32s；处理办法是 closeQuietly + 限时 join）
    └─ ★ 绝不因为"连接关不掉"而阻塞整个停机链路

 6. 释放数据库资源、注销实例
    └─ instances 记录置 stale（或直接删除，取决于运维口径）
```

**"长连接关闭必须限时、放守护线程等待，不能阻塞停机"是阶段一教训的直接延续。** 阶段一实测过飞书 SDK 的 `close()` 阻塞不返回把停机拖满 32s（exit 143）的问题。集群档下这个风险**更大**，因为停机链路变长（多了属主释放、排空、实例注销），任何一个环节的阻塞都会挤占 40s 的 grace period，最终被 K8s SIGKILL，导致：属主已释放但实例没注销（运维视图有 `draining` 幽灵）、或更糟——属主没来得及释放（要等 30s TTL）。所以纪律是：**停机链路上的每个外部 IO 都必须有超时；只有"释放属主"这一步允许在超时后重试一次（它是最关键的），其余环节超时即跳过并记日志。**

**`ChannelAdminService.stopAll` 的顺序（明确列出）：** ①置 draining；②逐个渠道释放属主（短事务、带超时、失败重试一次）；③逐个渠道调适配器的限时 `close`；④汇总日志（哪个渠道释放成功/失败、哪个 close 超时）。**顺序不能变**，且第 ② 步失败必须被显式记录（因为那意味着要等 TTL）。

### 8.5 启动期竞争的抑制

**问题：** 三个副本在滚动升级/扩容时可能在同一秒启动，都发现"没有属主"（或属主刚被释放），于是同时尝试插入式认领。只有一个成功，但失败的副本如果不小心处理，会立刻重试形成风暴；更糟的是**成功的那个可能还没建好连接就被另一个副本抢先**（不太可能，因为 CAS 唯一，但"属主已插入、连接未建立"这个中间态存在）。

**抑制手段（四条一起用）：**

1. **启动随机抖动。** 启动后等 `0 ~ oryxos.cluster.start-jitter-max`（默认 3s，随机）再开始属主竞争与调度扫描。这让"同时启动"变为"错开启动"，把竞争从"同一毫秒"稀释到"几秒内"。
2. **首轮竞争快速失败即退避。** 认领失败（影响 0 行）**不是错误**，是本副本的正常状态；失败后进入"下一个扫描周期再试"（周期 = 续约间隔量级，如 10s），不做"连续快速重试"。日志级别为 DEBUG/INFO，**不是 WARN**——否则每次扩容都会刷一堆看起来像故障的日志。
3. **"属主已插入但连接未就绪"的处理。** `channel_ownerships` 里只有租约，没有"连接是否已建立"的状态列——这是刻意的（不引入第二个需要维护的状态）。语义上：**持有租约即意味着"我应该去建连并处理事件"**，连接未就绪期间的入站事件由平台侧超时重推 + 回执去重兜底。若建连连续失败（例如平台凭证错误），必须告警（这是配置问题，不是竞争问题）。
4. **健康检查不因"不是属主"而失败。** readiness 只看数据库与工作区（见 9.4），不看"我是不是渠道属主"——否则非属主副本永远不 ready，滚动升级会卡死（这是最容易踩的坑：把"我持有渠道"当成就绪条件）。

### 8.6 三种渠道的差异与统一抽象

**统一抽象的契约（沿用阶段一 `InboundChannelAdapter` 的位置，不新造概念）：** 属主机制**不属于渠道适配器**，而是包在它外面的一个协调层——适配器只负责"连 / 收 / 正常关"，协调层负责"该不该连 / 什么时候可以连 / 停机时先释放"。

```text
        ChannelOwnershipManager（协调层，契约在 core）
                 │  持有时才允许 start()
                 ▼
        InboundChannelAdapter（每渠道一个实现，阶段一已有）
                 │  start() / closeQuietly(限时) / 事件回调
                 ▼
        InboundMessageService（共享编排，阶段一已有）
                 │  ★ 回执去重在这里（见第 7 章）
                 ▼
        AgentService …
```

| 渠道 | 长连接形态 | 差异点 | 统一抽象下的处理 |
|------|-----------|--------|----------------|
| **飞书** | `oapi-sdk` 长连接（`im.message.receive_v1`） | `close()` 存在阻塞不返回的实现路径（阶段一实测） | `closeQuietly` 放守护线程 + 限时 join（2s），超时放弃等待并记 WARN |
| **企微** | WebSocket 长连接（智能机器人） | 断线自动重连（阶段一已实现） | 重连由适配器负责；**但仅当本实例仍是属主时才允许重连**（失去租约时必须停止重连，否则会造成"旧属主偷偷连回来"） |
| **钉钉** | Stream 长连接 | 断线自动重连（对齐企微） | 同上 |

> **"失去租约后必须停止重连"是一条容易漏的纪律。** 企微/钉钉适配器有断线自动重连逻辑（阶段一实现），在集群档下它必须受属主资格约束：租约续约失败 ⇒ 置失去标志 ⇒ 适配器的重连循环在下一次迭代检查该标志并永久退出。否则会出现"旧属主在失去租约后仍然重连成功，与新属主形成双连接"——这会绕过 8.3 的全部设计。

### 8.7 边界

- **属主只保证"同一时刻一个活跃长连接"，不保证"消息零丢失"。** 消息丢失的可能路径有四条：①属主崩溃时尚未 ack 的事件（平台是否重推取决于平台）；②长连接渠道没有显式失败反馈，业务处理失败时平台不会重推（见 7.4 第 3 条）；③属主在"持有租约但连接未就绪"的窗口内收到的事件依赖平台重推；④平台自身的事件丢弃。兜底机制是**平台重推 + 回执去重**，而这两者的有效性都受平台行为约束。**这条边界必须在企业交付文档里明说**——不能承诺"消息不丢"，只能承诺"不重复处理 + 有审计可查"。
- **属主接管不迁移在途工作。** 被接管的是"连接"，不是"正在处理的事件"。原属主正在跑的事件要么跑完（若它在失去租约前完成），要么在新属主的检查点中止（它本来就不是新属主的责任）——这会造成"事件被中断"，由平台重推兜底。
- **属地粒度是渠道。** 一个渠道 = 一条长连接 = 一个属主。不做到"每个会话一个属主"（那会让连接数爆炸）、也不做到"全局一个属主"（那会让所有渠道绑在一个副本上，滚动升级时全部中断）。若某个渠道的消息量极大、成为单副本瓶颈，本方案的处理是**在渠道平台侧把它拆成多个渠道实体**（多机器人），而不是在底座里做渠道内分片。
- **不保证属主的公平性。** 谁先抢到谁就是属主，长期运行下可能出现"3 副本里 1 个持有全部渠道"的偏斜。这是可接受的（连接本身极轻），但**必须可见**——`GET /api/v1/instances` 显示每个实例持有的渠道（见 9.3），运维可以据此判断是否需要重启个别副本以重新分布。

**本方案不做：** 渠道属主的"负载均衡分配"（按副本数均分渠道）、渠道级限流、跨集群的渠道归属（多活场景）。

## 9. 实例注册、心跳与运维视图

### 9.1 `instances` 表与心跳线程

```sql
-- 注册：启动时插入；若 instance_id 已被活跃实例占用则报警（见 3.2 矩阵第 8 项）
INSERT INTO instances (instance_id, host, port, version, status, started_at, last_heartbeat_at)
VALUES (:instanceId, :host, :port, :version, 'starting', CURRENT_TIMESTAMP, CURRENT_TIMESTAMP)
ON CONFLICT (instance_id) DO UPDATE
   SET host = EXCLUDED.host, port = EXCLUDED.port, version = EXCLUDED.version,
       status = EXCLUDED.status, started_at = EXCLUDED.started_at,
       last_heartbeat_at = CURRENT_TIMESTAMP;

-- 心跳：同一个专用调度线程上的同步 UPDATE（间隔 10s）
UPDATE instances SET last_heartbeat_at = CURRENT_TIMESTAMP
 WHERE instance_id = :instanceId;
```

**状态枚举（逐字对齐 README）：** `instances.status` ∈ {`starting`, `ready`, `draining`, `stale`}。

| 状态 | 何时进入 | 语义 |
|------|---------|------|
| `starting` | 注册后、启动校验与装配完成前 | 已有心跳，但还不能接工作；运维视图里可见但不算可服务 |
| `ready` | 启动校验通过、工作路径就绪后 | 正常服务中 |
| `draining` | 收到 SIGTERM 后立即置 | 正在摘流与排空；**仍持有租约与在途工作**，所以不能直接当"已死"处理 |
| `stale` | 由**其他副本**在读取时判定（不写库），或本实例注销时写入 | 心跳过期；不再被信任 |

> **`stale` 的判定纪律：** 心跳过期是**读取侧的解释**（`last_heartbeat_at + heartbeat-ttl < CURRENT_TIMESTAMP`），与租约过期同构（见 4.4）。因此它不需要任何"监控线程去标 stale"，也不会因为某个副本卡死而漏标。允许在优雅下线时由本实例主动写 `stale`（这是"我知道我要走了"的额外信息），但**它不是 stale 判定的必要条件**。

**心跳线程模型：** 与续约共用同一个专用调度线程（`LeaseRenewer` 的 tick 里一起做），同步阻塞、只做数据库往返，不做业务（4.5 的纪律同样适用）。心跳写失败**不中止任何工作**，只记 WARN 并计数（4.5 第 4 条）。

### 9.2 假死与抖动判定

**判断：过期不等于"要杀"，只等于"不再被信任"。** 这是本节最重要的一条口径。`instances` 表里的 `stale` **不触发任何主动动作**——不去 kill 进程、不去抢它的连接、不去抢救它的工作。所有"接管"都是**由需要那项资源的其他副本独立发起的**（渠道属主租约、会话轮次租约各自有自己的 TTL 与接管逻辑），`instances` 只是给人看的视图。

| 现象 | 可能原因 | 本方案的判定 | 是否自动动作 |
|------|---------|-------------|-------------|
| 心跳超时但进程仍在跑 | 数据库连接池耗尽、网络分区、长时间 Full GC、宿主机暂停 | `stale`（视图） | 不动作。它的租约会在各自 TTL 过期后被接管 |
| 心跳超时且进程已死 | 崩溃 | `stale`（视图） | 同上，靠租约 TTL |
| 心跳正常但业务能力丧失 | 工作区不可读、Provider 全挂 | `ready`（视图）+ 指标/告警 | readiness 探针会失败（见 9.4），由 K8s 摘流 |
| 心跳抖动（偶发一次失败） | 瞬时网络抖动 | 不判定为 stale（TTL ≥ 3 × 间隔吸收了抖动） | 告警阈值应为"连续 N 次失败"而不是单次 |

> **为什么不做"过期即主动清理"：** 主动清理需要一个"谁有权清理"的决策者（回到主节点问题），而且在网络分区时它会做出最坏的决策（把实际上还活着、只是连不上数据库的实例的资源抢走）。**让每个资源自己的租约决定归属**，把判定收敛到"一个键上的一次 CAS"，就没有任何全局决策者，也就没有全局误判。

### 9.3 `GET /api/v1/instances` 的字段与语义

```
GET /api/v1/instances
{
  "instances": [
    {
      "instanceId": "oryxos-7d9f4b2c-x8k2p",
      "host": "10.0.3.17", "port": 8080, "version": "2.0.0",
      "status": "ready",                       // starting | ready | draining | stale
      "startedAt": "…", "lastHeartbeatAt": "…",
      "heartbeatAgeMs": 2140,                  // 服务端算的，不信任客户端时钟
      "load": {                                // 尽力而为的近似值，来自该实例自报
        "activeTurns": 3,                      // 本实例正在跑的会话轮次
        "claimedSchedules": 1                  // 本实例当前持有的定时触发认领
      },
      "channels": ["feishu:ops","wecom:hr"],   // 本实例当前持有的渠道属主
      "leases": {                              // 明细可选择性返回（见下）
        "turnLeases": 3, "channelLeases": 2, "knowledgeClaims": 0
      }
    }
  ],
  "generatedAt": "…",
  "staleThresholdMs": 30000
}
```

**字段语义与三条纪律：**

1. **`heartbeatAgeMs` 由服务端用数据库时间（或统一的"接收侧时钟"）计算，不接受客户端上报的相对时间**——否则时钟漂移会污染运维视图（与 4.3 的"用数据库时间"同一条纪律）。
2. **`load` 是"该实例最近一次心跳/自报的近似值"，不是实时真相。** 视图里必须带上"这个数字多旧"（`heartbeatAgeMs`），让人自己判断可信度。**不要**把它当作容量决策的唯一依据（容量决策应看指标聚合，见 14.3）。
3. **`leases` 明细默认只返回计数**（避免暴露内部键名与 session_id），需要明细时用一个显式的运维参数（如 `?detail=true`）并按权限控制。这一条是安全边界的一部分（见 15.4）。

**这个端点的三个真实用途（不是"为了有个端点"）：** ①滚动升级时确认新旧副本的版本分布与摘流是否生效；②排障时回答"消息重复回复是不是两个副本都认为自己持有的"（见 14.6）；③确认渠道属主在接管后确实只在一个实例上（Demo E 的断言）。

### 9.4 与 liveness/readiness 探针的分工

**判断：readiness 必须含数据库校验；liveness 只看进程。** 这两个探针的语义差异在集群档下变成了正确性问题，不只是可用性问题。

| 探针 | 检查什么 | 为什么 | 不这样做的后果 |
|------|---------|--------|---------------|
| **readiness** | ①能否连数据库并成功执行一次读校验查询；②工作区可读（能读到 `.workspace-id`）；③（可选）工作区版本号可读 | 集群档下"连不上数据库"= 不能做任何正确的工作（D11 fail-closed）。这样的副本**必须被摘流**，而不是继续接受请求然后每个都失败 | 流量继续打到一个无法工作的副本，用户看到成片超时，而 K8s 认为它健康 |
| **liveness** | 只看进程是否卡死（如一个由请求线程定期更新的"心跳文件/内存标记"是否在合理时间内被推进） | 如果 liveness 也检查数据库，那么**数据库故障会让全部副本同时被 K8s 重启**——这是最经典的级联故障 | 数据库抖一下 ⇒ 所有副本重启 ⇒ 启动风暴 + 数据库连接雪崩 |

> **分界的一句话口径：** readiness 回答"我现在能不能服务"（依赖外部资源）；liveness 回答"我还需不需要被重启"（只依赖自身）。**任何"外部依赖不可用"都不允许进入 liveness 判据**，这是本方案对`探针设计`最硬的一条要求，必须写进 Helm chart 的注释里（见 16.5）。

**参数建议：** readiness `initialDelaySeconds: 20`、`periodSeconds: 10`、`timeoutSeconds: 3`、`failureThreshold: 3`；liveness `initialDelaySeconds: 60`、`periodSeconds: 20`、`timeoutSeconds: 3`、`failureThreshold: 3`。启动期较宽松（Spring Boot 启动 + 工作区扫描 + Flyway 迁移），稳态较紧。

### 9.5 优雅下线（draining）

```text
① 收 SIGTERM
   └─ instances.status = 'draining'（立即，让运维视图与同伴可见）
② 摘流
   └─ readiness 立即失败 → K8s 从 Service 端点摘除（不再有新请求）
③ 停止认领新工作
   ├─ AgentScheduler 停止扫描（不再产生新的 schedule_claims）
   ├─ 会话轮次：新的认领直接拒绝（快速失败，让负载均衡重试到其他副本）
   └─ 入站事件：处理路径检查 draining ⇒ 返回可重试错误（让平台重推）
④ 排空在途（上限 drain-timeout = 40s，与 terminationGracePeriodSeconds 对齐）
   ├─ 等已在跑的会话轮次结束（继续续约，保持租约）
   ├─ 等已认领的定时触发结束
   └─ 等工作区写入收尾（原子落盘是"要么完成要么没开始"，天然可排空）
⑤ 释放渠道属主（见 8.4：这是停机链路里最关键的一步）
⑥ 关闭长连接（限时 + 守护线程，绝不阻塞）
⑦ instances 置 stale（或删除，按运维口径）；关闭数据库连接池
```

**排空与续约的关系（易错点）：** 排空期间**必须继续续约**。如果一进入 draining 就停止续约，正在跑的轮次会在 TTL 到期时被其他副本接管并标 `failed`——而它其实正在正常收尾。正确语义是："停止**接受新工作**，但继续**守护已有工作**直到它结束或排空超时"。超时后（40s 到）直接退出，未完成的轮次留给接管者按 D6 标失败。

**本方案不做：** 停机时的"工作交接"（把在途轮次主动交给别的副本继续跑）——它需要序列化"半截的 ReAct 循环状态"，而 LLM 会话状态不可无损序列化（模型侧的上下文不在我们手里）。**因此停机语义只能是"排空或标失败"，不存在"迁移中"这一档。**

---

## 10. 共享工作区

### 10.1 存储抽象与插件选择

**契约（在 core，业务代码只认它）：**

```text
WorkspaceStorage
  Path root()                    // 逻辑根（共享卷根 / 本地工作区根）
  Path resolve(String logical)   // 逻辑路径 → NIO Path；业务唯一允许的路径来源
  Path nativePath(Path p)        // 本机执行视图（见 10.3）

WorkspaceStorageProvider
  String id()                    // 显式选择："local" | "shared"（插件标识即配置值）
  WorkspaceStorage create(config)
```

**四条不可违背的使用纪律（对齐 042）：**

1. 业务代码**只能用** `root()` / `resolve()` 得到的 NIO `Path`；**禁止**把它转成字符串后用 `Path.of` 绕回默认存储（那会在共享模式下悄悄指向容器本地路径）。
2. 本机执行（文件 Tool、Shell Tool、脚本运行）**必须**调用 `nativePath`；直接把逻辑路径当文件路径用是违规。
3. 插件选择是**显式的**（`oryxos.workspace.storage` 的值 = `WorkspaceStorageProvider.id()`），不靠"扫描到一个实现就用它"。
4. 契约在 core、实现在下游（存储模块或独立插件模块），**禁止**循环依赖（见 20.1）。

**两种插件的适用面与不支持的情况：**

| 插件 | 存储 | 适用 | 不适用 |
|------|------|------|--------|
| `local` | 实例本地目录 | 单机档、开发、CI | 集群档（各副本一份 ⇒ 数据分裂） |
| `shared` | NFS / CIFS / 云盘 RWX | 集群档的默认 | 只支持 `ReadWriteOnce` 的块存储；跨地域低带宽卷；不支持 rename 语义的对象存储挂载 |

**共享卷的最低能力要求（只依赖两条，与 D7 一致）：** ①一个副本写的文件，其他副本能在有限时间内读到（read-after-write visibility）；②同一文件系统内 `rename` 覆盖是原子的。**明确不依赖：** 文件锁（`flock`/`fcntl`）、`inotify`/`WatchService` 事件、`mmap` 跨节点一致性、目录级原子操作、`fsync` 的跨节点语义保证。

### 10.2 启动校验 `.workspace-id` 与失败不回退本地（D9）

```text
启动序列（共享模式）：
  1. 解析 root
  2. 读 root/.workspace-id
     ├─ 不存在 ⇒ **拒启动**
     │    "共享工作区缺少 .workspace-id：拒绝初始化。
     │     请由管理员显式预置该文件（内容 = 集群标识）。
     │     实例不会自动创建，也不会回退到本地工作区。"
     └─ 存在 ⇒ 读出内容 c
  3. 与期望值 oryxos.workspace.id 比对
     ├─ 不匹配 ⇒ **拒启动**
     │    "期望工作区标识 X，卷上为 Y。很可能挂载到了别的集群的卷。
     │     修正挂载或配置后再启动。"
     └─ 匹配 ⇒ 通过，继续启动校验其余项
```

**为什么必须由管理员预置、不由实例自动创建：** 如果实例在缺失时自动创建，那么"挂错了空卷"这种事故会被**静默变成一个可用的新工作区**——所有 Agent 都不见了，但没有一条错误。让这个文件成为一道人工闸门，误差就从"几天后发现数据不对"变成"启动时报错"。

**为什么失败绝不回退本地（D9 的核心论证）：**

```text
假设允许回退本地：
  t0  运维误把 RWX PVC 的挂载路径写错（或卷未就绪，挂成了 emptyDir）
  t1  实例 1/2/3 各自发现"工作区为空" ⇒ 回退本地 ⇒ **各自初始化一份新工作区**
  t2  三份空工作区开始服务：Agent 目录为空 ⇒ 所有 Agent 都不存在
  t3  运维"修复"挂载、重启实例 ⇒ 读到正确的卷，问题看似消失
  t4  但 t1–t3 期间产生的数据（会话、审计、记忆写入）分散在三个本地目录里，
      **没有任何机制能把它们合并回共享卷**；更糟的是写路径写本地、读路径读共享卷 ⇒
      同一个逻辑路径有两个来源（最难查的一类故障）
```

**结论：回退本地会把"一个可检测的启动失败"变成"一次不可恢复的数据分裂"。** 因此设计上只有两个选项：正确挂载、或启动失败。这一条与 D11（误配拒启动）是同一条原则在文件面上的体现。

**自检的可选增强（不改变结论）：** 允许在 `.workspace-id` 中附带集群标识之外的元信息（如创建时间、管理员备注），并在启动日志中回显，便于运维确认"我连的是哪个卷"。**不允许**把密钥、凭证等敏感信息放进去（这是明文文件）。

### 10.3 本机执行视图（`nativePath`）

**问题：** 共享卷在一个副本上的挂载路径，与它在另一个副本上的挂载路径**可能不同**（容器里的挂载点由部署决定）。工作区里存的是**逻辑路径**（相对 `root()`），而"执行一个文件"或"用 `shell` 跑一个脚本"需要**本实例上真实可用的绝对路径**。

```text
逻辑路径（存进对话/审计/软连接）     本机执行视图（只在执行瞬间使用）
  agents/ops/scripts/check.py   ──►   /mnt/oryxos-workspace/agents/ops/scripts/check.py
                                                    （本实例的挂载点）
```

**规则：**

1. **逻辑路径是唯一真相。** 进审计、进 prompt、进软连接、进 Skill 元数据路径的，一律是逻辑路径（或其派生的一致形式）。**绝不把本机路径持久化**——否则迁移到挂载点不同的集群后，所有记录都会指向不存在的路径。
2. **本机执行视图只在执行瞬间构造，用完即弃。** 不缓存（挂载点可能在实例生命周期内变化，虽然罕见）、不写入任何持久状态。
3. **构造时必须过白名单校验**（原则六）：`nativePath` 的结果必须经 `toRealPath()`（目标存在时）或"最近存在父目录的 `toRealPath()`"（目标不存在时）校验，确认真实路径仍位于白名单根之内。
4. **`nativePath` 找不到本实例可访问的路径时，必须快速失败**，且**绝不回退到本地目录**（理由同 10.2）。
**失效场景与处理：**

| 场景 | 表现 | 处理 |
|------|------|------|
| 共享卷在本实例未挂载 | `nativePath` 无法解析 | 启动校验已拦住（3.2 矩阵第 10 项）；运行期出现则 fail-closed + 告警 |
| 挂载点与其他副本不同 | 逻辑路径一致、本机路径不同 | **正常**，正是 `nativePath` 存在的理由 |
| 目标文件在其他副本刚创建、本实例尚未可见 | 找不到文件 | 这是 NFS 属性缓存导致的短暂不可见；按"文件不存在"处理（业务侧应容忍），并计入指标（见 14.3） |
| 迁移期（单机档 → 集群档）挂载点变化 | 历史审计里的路径若曾是本机路径则失效 | 迁移前审计口径已是逻辑路径（阶段一即用工作区相对路径），迁移不改 |

> **迁移期处理：** 从单机档迁到集群档时，`nativePath` 的语义从"就是本地目录"变成"共享卷的本地挂载点"。因为阶段一的文件访问已经统一走工作区根，**这一步不需要数据迁移**；需要迁移的只有部署配置（挂载点）与 `.workspace-id`（见 3.3）。

### 10.4 原子落盘

**判断：一切工作区写入走"同目录临时文件 + `rename`"，由 `AtomicFiles` 统一提供。**

```text
AtomicFiles.write(target, bytes):
  1. tmp = target 的同目录下、以保留前缀命名的唯一临时文件
     （必须同目录：rename 的原子性只在同一文件系统内保证，跨目录/跨卷会退化为 copy+delete）
  2. 写 tmp（完整写入 + flush）
  3. （可选，由 oryxos.workspace.fsync 控制）fsync(tmp)
  4. rename(tmp, target)          ← 原子替换
  5. （可选）fsync(target 所在目录)
```

**为什么必须同目录：** 把临时文件放在 `/tmp` 再 `rename` 到共享卷，跨文件系统时 `rename` 会失败或退化为"复制 + 删除"（非原子），而这正是要消除的窗口。**临时文件位置是一个硬约束，不是风格选择。**

**`fsync` 的取舍（必须写清，不能让实现者自由发挥）：**

| 配置 | 行为 | 代价 | 适用 |
|------|------|------|------|
| `fsync=false`（默认） | 不强制刷盘 | 掉电/宿主崩溃时，最近几次写入可能丢失（但不会出现"半个文件"，因为 rename 原子） | 绝大多数企业场景；数据可从数据库重建或可接受少量丢失 |
| `fsync=true` | 写后与目录项都刷盘 | 共享卷上 `fsync` 的延迟可能是本地磁盘的 10–100 倍；批量发布（几十个 Agent 文件）会明显变慢 | 有明确"掉电不丢"要求的部署 |

> **关键点：`fsync` 影响的是"掉电后是否丢最近写入"，不影响"是否出现半截文件"。** 后者由 rename 原子性保证，与 fsync 无关。这个区分必须在配置文档里写明，否则运维会以为默认值"不安全"而盲目开启，付出不必要的延迟。

### 10.5 多文件发布的语义

**判断：可恢复，但**不是**目录原子快照。** 这是 042 明确要求写清的一条。

**问题：** 发布一个 Agent（`AGENT.md` + 若干 Skill 软连接 + `scripts/`）是**多个文件**的写入。每个文件单独是原子的（10.4），但**一组文件之间没有原子性**：第 3 个文件写完、第 4 个文件还没写时崩溃，磁盘上就是一个"半个新 Agent"。

**发布协议（写序 + 清单）：**

```text
发布流程：
 1. 把全部新内容写进 .staging/<publish-id>/（保留路径，同一文件系统）
    └─ 逐个文件用 AtomicFiles 落盘
 2. 校验（可选但推荐）：目标状态可在 .staging 内自检
 3. 生成发布清单 .staging/<publish-id>/.publish-manifest
    内容：publish-id、目标逻辑路径列表、每个文件的校验值、开始时间
 4. 按**固定写序**把文件从 .staging rename 到最终位置
    └─ 写序规则：先写"内容文件"（scripts、SKILL.md 等），
        再写"引用/入口文件"（AGENT.md 最后写）
        ⇒ 任何中途崩溃，读者看到的都是"旧入口 + 部分新内容"或"新入口 + 完整新内容"，
          绝不会看到"新入口指向不存在的内容"
 5. 删除 .staging/<publish-id>/ 与清单
```

**崩溃后的恢复流程：**

```text
启动时 / 管理写操作前：
 扫描 .staging/：
   ├─ 无残留 ⇒ 正常
   └─ 有残留（存在 .publish-manifest）⇒ 这是一次未完成的发布
      ├─ 检测：按清单逐项检查目标位置的文件是否存在且校验值匹配
      ├─ 决策：清单记录的写序 + 目标状态决定"继续完成"还是"整批放弃"
      │   （两者都必须可重入：重复执行不产生更坏的结果）
      └─ 记录 WARN 日志与指标：未完成的发布次数
```

**"可检测性"是本节的核心承诺：** 因为清单落盘在 `rename` 之前、且包含每个文件的校验值，所以崩溃后的状态**总是可检测的**（有残留目录 = 有问题）。这一条比"保证原子"更弱但更诚实：我们不承诺不出现"半个新 Agent"，我们承诺"出现时能被发现、能被恢复、且不会让读者看到自相矛盾的状态"。

> **写序规则为什么是"内容先、入口后"：** 读者（`ContextLoader`、`AgentLoader`）总是从入口（`AGENT.md`）开始读。若入口最后写，则在任何中间状态里，读者要么看到旧入口（配旧内容，一致），要么看到新入口（此时新内容已全部就位，一致）。反过来"入口先写"会让读者看到一个指向尚不存在内容的新入口——那是唯一会产生**语义错误**（而非仅仅陈旧）的顺序。

### 10.6 保留路径与命名纪律

**保留路径（底座内部占用，业务 Agent 不得使用）：**

| 路径 | 用途 | 谁写 | 谁读 |
|------|------|------|------|
| `.workspace-id` | 工作区标识（见 10.2） | 管理员预置 | 启动校验 |
| `.workspace-lock` / `.workspace-*` | 管理写互斥预约、读写者停机预约的落盘痕迹（见 10.8、10.9） | 管理面 | 管理面 |
| `.staging/` | 多文件发布的暂存区（见 10.5） | 管理写路径 | 发布协议 |
| `.workspace-versions`（若落盘） | 版本号的本地缓存 | 轮询线程 | 轮询线程 |

**命名纪律（三条）：**

1. **以 `.workspace-` 开头的路径与 `.staging` 是保留的**，业务 Agent 的任何文件、任何 Skill、任何脚本都不得使用；管理写互斥预约在写入前必须**显式检查并拒绝**这类路径。
2. **保留路径不参与版本号总线**（它们的变更是底座内部的，不需要传播重载）；反过来，域范围的 bump 也**不包含**保留路径。
3. **业务数据不得依赖保留路径的持久性**——`.staging` 可在任何时刻被清理，`.workspace-lock` 类文件在恢复流程中会被删除。

### 10.7 运行输出按 Agent/run 隔离路径

**判断：运行输出落在按 Agent + run 隔离的路径下；既不按实例隔离，也不共享同一目录。**

```text
.oryxos/output/
├── <agent-name>/
│   ├── <run-id-1>/          # 一次运行的产物（时间戳 + 短随机）
│   └── <run-id-2>/
└── <another-agent>/
    └── <run-id-1>/
```

**为什么不能按实例隔离：** 请求落到任意副本是前提（D14 线外明确不做会话粘性）。如果产物写在"本实例目录"下，那么同一个 Agent 的产物会散落在 N 个副本各自的目录里，而下一次请求（可能落到另一个副本）读不到上一次的产物——"Agent 有记忆的产出"这件事立刻坏掉。**按实例隔离会把无状态假设破坏掉。**

**为什么不能共享同一目录（所有 run 都写 output/）：** ①不同 Agent 可能产生同名文件（`report.md`），互相覆盖；②同一个 Agent 的两次并发 run 会互相覆盖；③没有清理边界（无法按 run 回收）。**同一目录会让"谁写的"变得不可追溯。**

**为什么是 run 级而不是只到 Agent 级：** Agent 级的并发 run 是常态（两个用户同时问同一个 Agent），run 级隔离让"这次运行的产物"成为一个可整体引用、可整体过期的单元（`read_file` 的路径可被审计精确定位，清理可按 run 粒度过期）。

**清理口径：** 按 run 目录的创建时间做保留策略（可配，默认按业务审计口径），清理是**纯删除**（不涉及版本号总线、不需要 bump）。**不做自动清理的默认值**：本周期只提供"按 run 目录人工/脚本清理"的路径，自动清理放后续（避免误删产物）。

### 10.8 管理写互斥预约

**判断：不自动过期，人工释放。** 这是 D11/D9 同族的一条：宁可让人工多走一步，也不要让"以为独占、其实已过期"的状态出现。

```text
预约流程（管理面对某类工作区数据的写操作，如"离线恢复"、"批量迁移"）：
 1. 申请：INSERT INTO admin_write_reservations (scope, owner_instance_id, reserved_at, note)
          ON CONFLICT (scope) DO NOTHING
    └─ 影响 0 行 ⇒ 已被预约 ⇒ 返回 409，附上持有者与 note（谁在做什么，很重要）
 2. 执行写操作（此时假设自己是唯一写者）
 3. 释放：DELETE FROM admin_write_reservations
          WHERE scope = ? AND owner_instance_id = ?
```

**为什么不自动过期（这是本节的核心论证）：**

- **自动过期会制造"看似独占、实际已过期"的状态。** 一个恢复操作可能耗时超过任何合理的 TTL（几十 GB 的工作区复制、跨存储迁移）。如果预约带 TTL，那么"预约已存在"这个信号在 TTL 之后的含义会从"有人在做，别动"变成"也许有人在做，也许没有"——而后者的信息量为零，运维不得不再去别处确认。与其给一个会骗人的信号，不如给一个永远诚实的信号：**存在即有人持有，直到有人显式释放。**
- **自动过期的失败模式比人工忘记释放严重得多。** 人工忘记释放的代价是"被叫醒一次、删一行"；自动过期导致两个写者同时认为独占的代价是**工作区损坏**（两个发布协议交错、`.staging` 互相清理、软连接指向半成品）。
- **人工闸门需要的信息在 `note` 里。** 因为不自动过期，"怎么恢复"就变得重要：接管者需要知道"上一次是谁、为了什么、什么时候开始的"——这正是 `note` 与 `owner_instance_id`、`reserved_at` 三列存在的理由。

**恢复前必须停全部写者（硬性前置条件）：**

```text
离线恢复的标准流程：
 1. 停掉**全部**可能的工作区写者（不只是持有预约的那个实例）：
    ├─ 所有副本进入 draining 并退出（或缩容到 0）
    └─ 确认：GET /api/v1/instances 中没有 ready/starting 的实例
 2. 确认读写者停机预约已完成（见 10.9）
 3. 执行恢复（从备份恢复、修复损坏、迁移）
 4. 校验：.workspace-id 正确、关键目录结构完整、软连接无 dangling
 5. 显式释放预约（DELETE）
 6. 重启实例；必要时调 POST /api/v1/workspace/refresh（见 11.5）
```

> **为什么"停全部写者"而不是"停持有预约的那个"：** 预约只覆盖"申请过预约的那类写操作"，而**其他副本仍可能在跑正常的业务写**（Agent 通过 `write_file` 写运行输出、记忆写入、会话追加）。恢复动作假设的是"文件面静止"，所以必须真的静止——这只能通过停掉所有实例来保证。

### 10.9 读写者停机预约

**问题：** 停机流程里"我已经不再写工作区了"这个事实需要被确认，否则离线恢复可能在某个副本还在写的窗口内开始。这是"最后一个写者关灯"的语义。

```text
停机时的读写者确认：
 1. 每个副本在 draining 阶段完成后，写一条落盘痕迹：
    .workspace-<instance-id>.stopped（内容：停写时间、实例标识）
 2. 恢复操作开始前，检查：
    ├─ GET /api/v1/instances 无 ready/starting 实例
    └─ 卷上无新的"停写痕迹"缺失（即每个曾经 running 的实例都有对应痕迹）
 3. 恢复完成后，清理全部停写痕迹（它们是本次停机周期的产物，
    不属于工作区数据，不需要版本号 bump）
```

**与"读写者停机预约"术语的关系：** README 定义它为"停机流程里的写者退场确认，保证'最后一个写者关灯'之后才允许做离线恢复动作"。本方案把它实现为**落盘痕迹 + 实例视图双重确认**，而不是一个数据库里的租约——理由：离线恢复的前提恰恰可能是"数据库也不可用"（例如数据库损坏），此时纯数据库侧的预约会失效；落盘痕迹在共享卷上，与要保护的对象在一起，可用性一致。

**边界：** 痕迹文件是**提示性证据**，不是强制互斥。它的价值在于把一个"记得要停全部实例吗"的流程问题变成"卷上有没有缺痕迹"的检查项。

### 10.10 权限与属主

| 项 | 建议 | 理由 |
|----|------|------|
| 多副本同 uid | 全部副本以同一非 root uid（如 1000）运行 | 共享卷上的文件属主必须一致，否则不同副本写的文件之间互相不可读/不可写 |
| 目录属主 | 工作区根目录属主 = 该 uid；权限 `0755`（目录）/ `0644`（文件） | 满足"读写可见"的最低要求，避免"某个副本以别的 uid 写出的文件其他副本读不了" |
| umask | 显式设置（`0022`） | 不同基础镜像的默认 umask 可能不同；不显式设置会让"在某些环境能跑"变成玄学 |
| 可执行位 | 只有明确需要执行的脚本设置 `0755` | 原则六的 Shell 白名单是主要防线，可执行位是辅助；不必要的 `+x` 只是扩大风险面 |
| 只读挂载 | **不适用于集群档写者**。可用于"只读运维工具容器"（例如一个只读副本用于导出/审计查询） | 集群档的每个实例都要写运行输出与记忆；只读挂载会让启动校验的"探针级原子写"失败（3.2 矩阵第 10 项） |
| root squash | 若共享存储支持，开启 | 防止容器内 uid 0 意外以存储侧 root 身份写文件 |

> **K8s 下的一个常见坑：** `securityContext.fsGroup` 会 chown 卷挂载点，在**大规模共享卷**上可能很慢甚至超时；如果卷已经以正确 uid 预置，优先用 `runAsUser`/`runAsGroup` 而不用 `fsGroup`。这条写进 Helm chart 注释（见 16.4）。

### 10.11 软连接与真实路径校验在共享卷下的加固

**宪法原则六的基线（阶段一已有）：** 文件目标存在时必须用 `toRealPath()` 校验真实路径仍在白名单根内；新建路径校验**最近存在父目录**的真实路径；Agent Skill 绑定只允许指向 `.oryxos/skills/` 的相对软连接，拒绝绝对链接与越界链接。

**共享卷下为什么要更严：** 阶段一的软连接校验假设"路径解析结果在本地是稳定的"；共享卷下有三处额外的风险面：

| 额外风险 | 场景 | 加固 |
|---------|------|------|
| **跨副本编辑造成的悬空** | 副本 A 改了软连接指向，副本 B 的缓存/已打开句柄仍指向旧目标；B 尚不可见新目标 ⇒ 出现 dangling 窗口 | 所有软连接**在使用前重新解析**（不缓存解析结果）；dangling 一律拒绝，**不回退到"上一次成功的目标"** |
| **挂载点差异导致的"看起来越界"** | 同一逻辑路径在不同副本的本机路径前缀不同；若比较时混入本机路径，会把合法链接判成越界 | 越界判定**必须在逻辑路径空间完成**（比较解析后的逻辑路径是否仍在 `.oryxos/skills/` 之下），`nativePath` 只用于最终执行，不参与越界判定 |
| **属性缓存导致的 TOCTOU** | 校验时目标存在且合法，执行前目标被其他副本替换成指向外部 | 校验与执行之间不能依赖"文件系统状态不变"；执行前**再校验一次真实路径**（原则六的 `toRealPath()` 本来就是这个作用，共享卷下更要把"二次校验"当默认而不是可选） |

**五类检测（逐项都要在共享卷下成立）：**

```text
1. name-mismatch：软连接名 ≠ 目标 Skill 目录名（或 frontmatter name）
   ⇒ 拒绝。共享卷上这一条尤其重要：其他副本可能改了目标目录名，
     而软连接名未变，此时"看起来绑定成功"但语义已错。
2. dangling：目标不存在
   ⇒ 拒绝。共享卷上出现 dangling 是**正常现象**（跨副本可见性延迟），
     但处理方式只能是"拒绝 + 下次重试"，绝不能"用旧值兜底"。
3. escaped：解析后的真实路径不在 .oryxos/skills/ 之内
   （绝对链接、../ 穿越、指向工作区外）
   ⇒ 拒绝 + 告警（这可能是恶意或严重误配）。
4. invalid-target：目标是文件而不是目录、或缺少 SKILL.md
   ⇒ 拒绝。
5. stale-reference：公共 Skill 被 Agent 引用时被删除/改名
   ⇒ 公共 Skill 的删除默认拒绝（沿用阶段一），
     集群档下"引用检查"必须读**共享卷的当前状态**而不是本副本的缓存。
```

> **一条贯穿性纪律：所有校验的真相源是共享卷的当前状态，不是任何副本的内存/缓存。** 这与第 11 章的版本号总线是同一件事的两面：总线告诉副本"该重载了"，而校验每次都直接看文件系统。

**本方案不做：** 对工作区启用文件锁（D7）、对软连接做"跨副本一致性协议"（无必要：检查即拒绝即可）、对共享卷做扩展属性（xattr）依赖。

---

## 11. 变更感知：版本号总线

### 11.1 为什么集群档不装 `WatchService`（D8）

阶段一用 JDK `WatchService`（inotify 之上）监听 `.oryxos/agents/` 变更，实现"丢目录即上线、全程免重启"。集群档**不装**它，理由三条：

| 理由 | 具体表现 |
|------|---------|
| **共享卷上的事件不可靠** | NFS/CIFS 的 inotify 语义依赖实现：很多实现**完全不上报**别的节点产生的事件；有的只在"写者与读者使用同一客户端缓存"时上报；云盘 RWX 的实现差异更大。结果是"某些副本永远不重载" |
| **事件丢失无补偿机制** | `WatchService` 丢事件时**没有任何信号**（不像"版本号落后"是可检测的）。运维不会知道某个副本停在旧版本，直到用户投诉某个 Agent 改了不生效 |
| **重载语义与"最终一致"不匹配** | 即使事件到达，"收到事件 → 重载"仍需要处理重载失败重试、重载原子性、多个域交织变更（一次发布会 bump 多个域）。把可靠传播的职责交给一个会丢事件的通知机制，等于把可靠性寄托在最弱的一环上 |

**单机档如何做到零回归（D1）：** `WatchService` 的装配被档位开关包住——`oryxos.cluster.enabled=false` 时按阶段一的方式装配 watcher；`true` 时装配轮询器。**同一条 `AgentLifecycleService.register(agentDir)` 入口不变**（阶段一已经做到两条录入路径汇到同一个方法），所以单机档的行为逐点不变，集群档只是换了"什么时候发现变了"的机制。

> **集群档也不做"文件系统轮询全量扫描"。** 版本号总线是**表比对**（读 6 行、比较数字），而不是扫描目录树（可能上千个文件）。这是本方案与"简单粗暴轮询文件 mtime"的分水岭：后者在共享卷上会因属性缓存产生假阴性，且开销随文件数增长。

### 11.2 `workspace_versions` 与域划分

```sql
-- 读：每个副本每 poll-interval（默认 1s）读一次六行
SELECT domain, version FROM workspace_versions;

-- 写（bump）：落盘成功后，对应域 +1
UPDATE workspace_versions
   SET version = version + 1, updated_at = CURRENT_TIMESTAMP, updated_by = :instanceId
 WHERE domain = :domain;
```

| 域 | 覆盖的工作区数据 | 重载动作 |
|----|----------------|---------|
| `agents` | `.oryxos/agents/`（每个 Agent 的 `AGENT.md`、`scripts/`、`REFERENCE.md`、`skills/` 软连接集合） | 重新扫描 Agent 目录、`deriveProfile`、注册/注销、软连接校验 |
| `skills` | `.oryxos/skills/` 公共 Skill 实体 | 刷新 Skill 元数据缓存（name/description/路径）；正文不缓存无需处理 |
| `personas` | `.oryxos/personas/` 自定义人格 | 重载人格库索引 |
| `knowledge` | `.oryxos/knowledge/` 文档源（**不含索引代次**，代次见第 12 章） | 标记"源已变"，是否重建由重建认领决定 |
| `channels` | `.oryxos/channels.yaml` 渠道配置 | 重载渠道配置；**注意：影响长连接配置的变更需要属主配合**（重连是属主的事） |
| `bootstrap` | `AGENTS.md`、`SOUL.md`、`USER.md` | 下一次 `ContextLoader` 组装时自然读到（阶段一无缓存），bump 用于触发指标与一致性观测 |

**域划分的原则：** ①**重载成本相近的归一个域**（`agents` 重扫较贵，`bootstrap` 几乎免费，不应混在一起）；②**变更相关性高的归一个域**（改一个 Agent 的软连接同时影响 `agents`；改公共 Skill 影响 `skills` 且已绑定它的 Agent 下次读取即见）；③**域数保持小**（六个；域太多会让"每 tick 读多行 + 比较"的开销与维护成本上升，域太少会让"改一个小文件导致全量重扫"）。

> **`knowledge` 域的特殊性：** 它 bump 的是"源文档变了"，**不直接触发重建**。重建是一个昂贵的、需要认领的操作（第 12 章），所以源变化只是让各副本知道"该考虑重建了"，实际重建由认领决定谁做、何时做。这条分离很重要——否则一次源变更会让 N 个副本同时启动重建。

### 11.3 写路径的 bump 边界

**判断：落盘成功 + bump 成功，才算"变更可见"。** 两步都必要：落盘成功但没 bump，其他副本永远不知道；bump 成功但落盘失败，所有副本会去重载一个没变（或不存在）的版本。

```text
写路径（所有管理面写操作，不分域都要遵守）：
 1. 通过 WorkspaceStorage 解析路径（逻辑路径）
 2. 用 AtomicFiles 落盘（多文件发布走 10.5 的协议）
    ├─ 失败 ⇒ 整体失败，**不 bump**，向上返回错误
    └─ 成功 ⇒ 继续
 3. bump 对应域（一条 UPDATE）
    ├─ 成功 ⇒ 完成，返回成功
    └─ 失败 ⇒ ★ 见下：落盘成功、bump 失败
```

**"落盘成功、bump 失败"的三种处置（必须选一种并写清）：**

| 处置 | 说明 | 本方案选择 |
|------|------|-----------|
| 回滚落盘 | 把文件改回去 | **不做**。回滚本身也可能失败，且"改回去"在共享卷上对其他副本不可原子 |
| 重试 bump（有限次） | 同步重试（用 4.6 的策略） | **做**：bump 是单条 UPDATE，瞬时故障重试即可 |
| 重试仍失败 ⇒ 记"待补偿"并让轮询自愈 | 定期（如每 tick）检查"本实例最近一次落盘是否有未 bump 的域"，重试 | **做**：这是自愈路径 |

**自愈机制（关键设计）：** 每个实例维护一个**进程内的"未确认 bump 集合"**（域 → 最近一次成功落盘的时间戳）。轮询线程在每次 tick 时：①读 `workspace_versions`；②若"本实例在 T 时刻对域 D 落盘成功但当时 bump 未确认"，则重试 bump；③重试成功后从集合移除。**这样做的代价是"变更可见性可能延迟"（直到数据库恢复），但不会永久丢失**——数据库恢复后自愈自动完成。

> **为什么不把"落盘 + bump"放进一个数据库事务：** 落盘在文件系统上，不在数据库事务里。这是文件面与关系面**无法原子**这一事实的直接推论（也是本方案明确接受的设计约束）。因此这里的正确性口径不是"原子"，而是"**可检测 + 可自愈**"：bump 失败被记录，恢复后自动补。

**外部直接改盘的用途之外的所有写都必须走这条路。** 运维直接用编辑器/git 改盘**不会**经过 bump（见 11.7），此时需要逃生舱（11.5）。

### 11.4 各副本轮询与重载

```text
轮询线程（每副本一个，专用调度线程，同步阻塞）：
 每 oryxos.workspace.poll-interval（默认 1s）：
  1. 一次性读六行 workspace_versions（一条 SELECT）
  2. 与内存中的"我当前已加载的版本"比较
     ├─ 全部相同 ⇒ 什么都不做（这是绝大多数 tick）
     └─ 有域落后 ⇒ 对该域执行重载
  3. 重载成功 ⇒ 更新内存中的版本号
     重载失败 ⇒ **不更新版本号**（下次 tick 会重试）+ WARN + 指标
```

**四条纪律：**

1. **版本号只在重载成功后更新。** 更新了但重载失败会导致"永远停在错误的版本"——这是最隐蔽的一类 bug（现象是"改了不生效，但日志说已重载"）。
2. **重载必须原子（对外可见的切换是一次性）。** 具体做法：重载时先构建新视图（新的 `ProfileRegistry` 内容、新的 Skill 索引），构建成功后再**一次性替换引用**；构建失败则保留旧视图并保留旧版本号。这与阶段一"`ContextLoader` 每次现读不缓存"不冲突——这里是**索引类**结构（Profile 注册表），必须原子替换。
3. **重载失败时的服务行为：继续用旧版本服务，并告警。** 不回滚盘、不停止服务、不把版本号推高。理由：旧版本仍然是**自洽的**（它是上一个成功状态），继续服务比拒绝服务好；而版本号不推进保证了下一次 tick 会重试。
4. **批量比对而非逐域查询。** 一条 `SELECT` 取六行，不是六条查询（见 17.2 的开销模型）。同时"全部相同"这一最常见路径必须**零分配、零重载动作**。

**重载失败的常见原因与处理：** 文件暂时不可见（NFS 属性缓存）⇒ 下次 tick 重试即可，属正常；发布中途（见 10.5）⇒ 若读到不自洽状态，重载应拒绝并重试，等待发布完成；软连接 dangling ⇒ 拒绝该 Agent 的注册并记 WARN（其他 Agent 不受影响，**重载粒度到 Agent 而不是整域失败**——这一点很关键，否则一个坏 Agent 会让整个域卡在旧版本）。

> **重载粒度的取舍：读一个坏 Agent 不应该阻塞其他 Agent 的重载。** 但版本号是域级的，所以"部分成功"时版本号**不推进**（否则永远看不到那个坏 Agent 的修复）。实现上需要区分"重载动作已完成（含坏 Agent 被跳过并记录）"与"重载完全成功"：前者可以推进版本号（因为坏 Agent 的失败已经被记录为状态，下次它被修好会通过 `agents` 域再次 bump 触发），后者不推进。**选择前者**：推进 + 记录失败清单 + 指标暴露"当前有几个 Agent 加载失败"，这样既能前进又不隐藏问题。

### 11.5 逃生舱 `POST /api/v1/workspace/refresh`

**语义：强制本集群所有副本重新加载指定域（或全部域），绕过版本号比对。**

```
POST /api/v1/workspace/refresh
Body: { "domains": ["agents","skills"], "reason": "运维直接改盘后强制重载" }
     （domains 省略 = 全部六个域）

实现（幂等）：
 1. 校验调用方（网络边界内，见 15.4）；reason 必填（用于审计）
 2. 对每个目标域执行一次 bump（version = version + 1）
    ⇒ 复用版本号总线的传播机制，**不新造"刷新通知"通道**
 3. 返回新的版本号
```

**为什么用 bump 实现而不是"广播刷新指令"：** 广播需要一个推送通道（长连接、消息总线），而本方案明确不做（D8）。用 bump 实现逃生舱有三个好处：①复用已验证的传播路径（轮询），不引入新的失败模式；②天然幂等（bump 多次只是多几次无害重载）；③可审计（bump 的 `updated_by` 记录了"这是一次人工刷新"）。

**幂等与代价：** 多次调用会导致多次重载（每次 bump 都会让所有副本重载该域），但重载是幂等的且不产生持久副作用。**代价是"调用它会让全集群做一次重载"**，所以：①REST 层做软性节流（如同一域 5 秒内只接受一次，多余的调用合并并返回同一个版本号）；②调用必须带 `reason` 并落审计（谁在什么时候因为什么刷新了哪个域）。

**什么时候用它（写进运维手册）：** ①运维直接用编辑器/git 改了工作区文件（不经过管理写路径 ⇒ 没有 bump）；②从备份恢复工作区后；③怀疑某个副本的版本号比对失效；④版本号表被人工修改过。

### 11.6 一致性语义

**判断：最终一致；传播上限 = 2 × 轮询间隔；目标 P99 ≤ 3s。**

```text
写者                        workspace_versions            各副本
 t0  落盘完成
 t0+ε  bump（域 D：v → v+1）
                              version[D] = v+1
                            ┌──────────────────────────────────────┐
                            │ 副本 A：下一 tick（≤ 1×interval）读到 │
                            │         ⇒ t0 + 1s 内重载             │
                            │ 副本 B：恰好刚过 tick（最坏情况）      │
                            │         ⇒ 等到下一个 tick             │
                            │         ⇒ t0 + 2×interval = t0 + 2s  │
                            └──────────────────────────────────────┘
 ⇒ 传播延迟上界 = 2 × poll-interval（默认 1s ⇒ 2s）
 ⇒ P99 ≤ 3s 的目标口径包含了"重载动作本身耗时"的余量（重扫 Agent 目录）
```

**三条口径说明（避免被过度解读）：**

1. **"最终一致"不是"最终不一致"。** 上界是**确定的**（2 × 间隔 + 重载耗时），不是"总会有一天一致"。
2. **P99 ≤ 3s 的测量方法**（Demo D 的断言）：在副本 A 上写文件并 bump，记录 t0；在所有副本上探测"新内容可见"的时刻（例如轮询某个 Agent 的 `description` 或一个探针字段），取各副本的延迟分布。**必须用"所有副本"的最大值，而不是平均值**——一致性问题永远由最慢的那个副本决定。
3. **读到旧版本期间的语义是"陈旧但自洽"。** 副本 B 在 t0–2s 之间用的是 v 版本的全部数据（不是"半个 v+1"）——这正是 11.4 第 2 条"原子切换"的收益。**不会出现"新 Agent 列表 + 旧 Skill 索引"这种混合状态**（域的原子性只保证域内，跨域的混合是允许的：`agents` 域先于 `skills` 域传播完是正常的，且阶段一的加载路径每次现读 Skill 元数据，所以这个混合窗口的实际影响很小）。

**传播上限的例外（必须诚实写明）：** ①**长连接相关配置**（`channels` 域）的重载可能不立即改变现有连接——连接的重建需要属主配合，实际生效时间取决于属主何时重建连接；②**knowledge 域**的 bump 只表示"源变了"，真正的检索结果变化要等重建代次切换（第 12 章），这不属于"变更传播延迟"的口径。

### 11.7 边界

- **浏览器/管理台直改文件。** 如果管理台（或某人 `kubectl exec` 进 Pod）直接写工作区文件而不经过管理写路径，**不会 bump**，其他副本看不到变更。这是设计上的必然（无法拦截任意的文件写入），兜底是逃生舱。**运维手册必须把"改盘之后调一次 refresh"写成标准动作。**
- **外部 `git pull` 改盘。** 同上，且更常见（企业可能把工作区当 git 仓库管理）。这种行为与版本号总线是"两套变更来源"，本方案的处理是：**git pull 之后必须调一次 refresh**（可挂到 CI/CD 或运维脚本里）。**不做** git hook 集成（那是运维侧的事，底座不侵入企业的 git 流程）。
- **多人同时改同一 Agent。** 版本号总线**不解决写冲突**——它只传播"变了"。两个管理台同时改同一个 `AGENT.md` 时，后写者覆盖先写者（文件系统的语义），两个副本最终一致地看到后写者的版本。要防覆盖需要乐观锁（版本号比对 + 提交时校验），本周期不做（属于管理台的并发编辑问题，不是底座的一致性问题）。
- **版本号回退。** 如果数据库被从备份恢复，`workspace_versions` 可能**退回**到一个更小的值。此时副本比较"我加载的版本 > 表中版本"⇒ 判定为"落后"（因为不等）⇒ 重载。重载后会加载**当前盘上的内容**，而版本号写入的是表里的旧值。这种情况下可能需要一次 refresh 把人造顺序理顺。**这条写进"工作区损坏恢复流程"（见 18.5）。**
- **版本号不是内容哈希。** 它不校验"盘上的内容是否真的变了"，只表达"有人声称某域变了"。一次 bump 可能对应空变更（落盘内容与原来相同），此时各副本会做一次无谓重载。这是有意的取舍：内容哈希需要读全量文件，成本远高于一次无谓重载。

**本方案不做：** 变更推送（长连接/消息总线）、内容哈希比对、变更历史与回滚（`workspace_versions` 只保留当前值，不保留历史）、按文件的细粒度版本。

---

## 12. 知识库在集群档

### 12.1 阶段一本地后端在集群档的问题

阶段一的知识库是"本地后端 + 向量索引"：文档在 `.oryxos/knowledge/`，解析/切分/向量化后索引落在本地（默认 SQLite 的 chunk 存储）。集群档下四个问题同时出现：

| 问题 | 表现 | 影响 |
|------|------|------|
| **每副本一份索引** | 各副本独立重建 ⇒ N 份索引、N 倍向量化成本（向量化要调 embedding 服务，是**要花钱的**） | 成本与不一致：副本 A 索引了 100 个文档、副本 B 只索引了 98 个（它启动时少了两个），同一问题在两个副本上得到不同答案 |
| **内存后端不可共享** | `memory` 知识后端把索引放在进程内存 ⇒ 每副本一份，重启即丢 | 集群档必须拒绝（见 3.2 矩阵第 3 项） |
| **重建代价大且会重复** | 重建要重新解析、切分、逐段调用 embedding | 多副本同时触发重建 ⇒ N 倍昂贵的 API 调用；且重建期间检索质量不稳定 |
| **导入冲突** | 导入新文档索引段与全量重建可能同时发生 | 两者都在改索引 ⇒ 索引状态不可预期（新文档丢或旧文档重复） |

**判断：集群档下知识索引必须有"代次"概念与"认领"机制。** 代次解决"检索读哪一份快照"，认领解决"谁来重建"。

### 12.2 代次模型

```text
knowledge_generations（一个作用域可以有多个代次，但只有一个处于 committed）
  generation_id  PK      代次标识（如 <backend>:<scope>:<时间戳>:<短随机>）
  backend / scope        作用域（幂等键，scope 如 "default" 或知识库逻辑名）
  state                  building | committed | retired
  chunk_count            该代次的 chunk 数（观测用）
  created_at / committed_at

检索路径（所有副本）：
  SELECT generation_id FROM knowledge_generations
   WHERE backend=? AND scope=? AND state='committed'
  ⇒ 只在这一个代次的 chunk 里检索
  （部分唯一索引 UNIQUE (backend, scope) WHERE state='committed' 保证唯一）
```

**代次状态机：**

```text
     重建认领成功
          │
          ▼
    ┌───────────┐   全部 chunk 写入 + 校验通过
    │ building  │ ──────────────────────────────▶ ┌────────────┐
    └───────────┘        （置 committed 的动作是        │ committed  │ ◀── 检索恒读这个
          │               一次原子切换：旧 committed      └─────┬──────┘
          │               置 retired，新代次置 committed）       │
          │ 认领 TTL 过期 / 重建失败                            │ 回收（保留 keep-generations）
          ▼                                                    ▼
    （标记失败，删除该 building 代次及其 chunk）          ┌────────────┐
                                                          │  retired   │ → 删除
                                                          └────────────┘
```

**三条关键设计：**

1. **检索恒读 `committed` 代次，不读 `building`。** 这是"重建期间检索不断服"的实现方式：重建过程写的是一个**新代次**的 chunk，而检索读的是旧代次，两者物理隔离（靠代次标识在 chunk 存储中的分区）。
2. **代次切换是一次原子的数据库状态变更**（一个短事务里：旧 `committed` → `retired`，新 `building` → `committed`），不是"改一个指针"这种更弱的语义。切换瞬间，正在进行的检索请求要么看到旧代次、要么看到新代次，**不会看到混合**。
3. **`building` 代次的失败必须可清理。** 认领 TTL 过期（实例崩溃）时，接管者不"继续"那个 building 代次（因为不知道它写到哪一步、且 chunk 可能部分写入），而是**判定其失败并删除该代次**，然后从零重建一个**新的 generation_id**。理由：代次是"整体可信"的单位，部分写入的代次无法判定完整性；重新生成一个代次的代价主要是 embedding 调用，而"用一个来源不明的半成品索引"的代价是错误答案。

### 12.3 重建认领

```sql
-- 幂等键 = (backend, scope)；主键即唯一约束
INSERT INTO knowledge_build_claims
       (backend, scope, owner_instance_id, lease_id, state, acquired_at, expires_at)
VALUES (:backend, :scope, :instanceId, :leaseId, 'claimed',
        CURRENT_TIMESTAMP, CURRENT_TIMESTAMP + :claimTtlSeconds)   -- 30min
ON CONFLICT (backend, scope) DO NOTHING;
-- 1 行 ⇒ 本副本负责重建；0 行 ⇒ 已被认领 ⇒ 见下"冲突返回 409"

-- 崩溃接管：只抢已过期的
UPDATE knowledge_build_claims
   SET owner_instance_id = :instanceId, lease_id = :newLeaseId, state = 'claimed',
       acquired_at = CURRENT_TIMESTAMP, expires_at = CURRENT_TIMESTAMP + :claimTtl
 WHERE backend = :backend AND scope = :scope AND expires_at < CURRENT_TIMESTAMP;

-- 续约（间隔 5min）：必须匹配 owner + lease_id
UPDATE knowledge_build_claims SET expires_at = CURRENT_TIMESTAMP + :claimTtl
 WHERE backend = :backend AND scope = :scope
   AND owner_instance_id = :instanceId AND lease_id = :leaseId;
```

**接管时的失败留痕（`state='failed'`）。** 崩溃接管不是"直接改写"：接管者先把过期认领标成 `failed` 并写一条审计（"这次重建没跑完，认领者 X 自 T 起失联"），再把同一行改写为 `claimed` 并换上新 `lease_id`。因为 `(backend, scope)` 是主键，这条失败留痕的生命周期与 `session_turn_leases.failed` **完全同构**——**留到被下一次认领覆盖为止**：运维在"故障发生后、接管发生前"的窗口里能直接看到它，接管之后它的历史意义由审计表承担。这正是 D6「标失败、不重放」在重建链路上的落地：接管者不接着写那个 `building` 代次（12.2 第 3 点），而是删掉它、从零重建一个新代次。

**冲突返回 409 让调用方排队（这是与"会话轮次排队"不同的设计，理由要说清）：**

| 维度 | 会话轮次（5.3） | 知识重建（本节） |
|------|---------------|----------------|
| 冲突时行为 | **同步等待**（≤ 120s） | **立即返回 409**，由调用方决定是否重试 |
| 为什么不同 | 用户就在等回复，等待体验可接受；轮次通常几秒到几十秒 | 重建可能几十分钟，同步等待毫无意义；且调用方通常是管理台/运维脚本，能自己重试 |
| 调用方约定 | 系统内部自动等待 | 调用方按 409 的 `Retry-After`（建议 = `knowledge-claim-ttl` 量级）稍后重试；或改为"查询状态"轮询 |

**"冲突方排队不丢"的落地方式：** 409 响应体带上有用信息（当前持有者、认领开始时间、预计何时可接管），管理台据此显示"重建进行中（由 instance-x 自 10:03 起）"。**不把请求放进队列**（那需要一个队列组件，超出范围）；而是让它**可重试且信息充分**——这就是"排队不丢"在本方案里的含义（README 第 7 章 Demo D 的口径："冲突方排队不丢"= 冲突方不会静默失败）。

**重建的完整流程：**

```text
触发源：①管理面显式请求（POST /api/v1/knowledge/rebuild 类端点）
        ②knowledge 域 bump 后的"该重建了"信号（由某副本尝试认领；
          认领失败即放弃，不重试——因为持有者已经在做）

 1. 认领（上面两条 SQL）
    ├─ 失败 ⇒ 409 或（自动触发路径）直接放弃
    └─ 成功 ⇒
 2. 生成新的 generation_id，插入 knowledge_generations(state='building')
 3. 解析 → 切分 → 逐段向量化 → 写入 chunk 存储（带 generation_id 分区）
    └─ 每个阶段之间做续约检查（租约丢失 ⇒ 立即中止，不再调用 embedding）
    └─ 分批提交（每批一个短事务），避免长事务
 4. 校验：chunk 数 > 0、抽样校验向量可读
 5. ★ 原子切换：一个短事务里
      旧 committed → retired；新 building → committed
    └─ 释放认领（DELETE knowledge_build_claims 的行）
 6. 回收：按 oryxos.knowledge.keep-generations = 2 保留（当前 + 上一代），
    更旧的 retired 代次及其 chunk 由清理作业批量删除
```

**续约检查点为什么在"每个阶段之间"而不是"每段之间"：** 每段（chunk）之间检查会让数据库往返淹没向量化调用；而阶段之间（解析完、切分完、每批向量化后）检查既能有足够快的失败响应，又不增加显著开销。**关键是"租约丢失后必须停止调用 embedding"**——这是成本保护：失去认领还继续向量化，等于在替别人白花钱。

### 12.4 导入索引段与重建的互斥

**问题：** "导入一批新文档并索引"（增量）与"全量重建"（整体）如果并发，会写出一个混合状态：重建正在写新代次，导入往"当前代次"追加 ⇒ 新代次里有的文档、旧代次里也有的文档，切换后语义混乱。

**判断：同一认领原语、同一作用域互斥。** 导入索引段与重建**共用 `knowledge_build_claims` 的 `(backend, scope)` 键**，因此天生互斥。

```text
导入路径：
 1. 尝试认领 (backend, scope)
    ├─ 失败（重建正在进行）⇒ 返回 409，"重建进行中，稍后重试导入"
    │   ⇒ 文档已落盘（源文件写进 knowledge 域），**索引段重试即可**，不丢
    └─ 成功 ⇒
 2. 对**当前 committed 代次**追加 chunk（增量，不新建代次）
    └─ 用较短的认领 TTL 并频繁续约（导入通常比全量重建快）
 3. 更新代次的 chunk_count；释放认领
```

**"排队不丢"的两层含义（写清楚，这是容易被误解的地方）：**

1. **文档源不丢。** 导入的第一步是把文档写进 `.oryxos/knowledge/`（工作区，经原子落盘），这一步**不受认领影响**。所以即使索引段被 409 拒绝，文档已经在了，用户不会"上传了但不见了"。
2. **索引最终会补上。** 索引段被拒后，触发方（管理台/运维）重试；或由 `knowledge` 域的 bump + 重建流程最终覆盖（全量重建会把所有源文档重新索引，包括那批新文档）。**因此本方案不需要一个持久化的"待索引队列"**——工作区里的源文档本身就是队列。

> **代价：** 如果没有人重试，新文档会"在盘上但不在索引里"，直到下一次重建。这是真实存在的窗口，缓解手段是**指标暴露"源文档数 vs 已索引 chunk 覆盖的文档数"差异**（见 14.3）并告警，而不是引入队列。

### 12.5 代次切换与双缓冲

**切换期间的检索行为（三种并发情形，逐一交代）：**

| 情形 | 行为 |
|------|------|
| 检索在切换**之前**开始、切换**之后**才结束 | 按它在开始时读到的 `generation_id` 完成检索（读的是 retired 代次的 chunk）；因为切换不删除旧代次的 chunk（保留 `keep-generations = 2`），所以这次检索**正常完成**，只是返回的是旧代次的结果 |
| 检索在切换**之后**开始 | 读新 committed 代次（新结果） |
| 一个请求内多次检索（ReAct 多轮调 `retrieve_knowledge`） | **可能跨代次**（第一轮旧、第二轮新）。这是允许的：代次切换不保证"一个会话内一致"，只保证"每次检索自洽"。若要会话级一致需把代次固定在会话上（不做，见边界） |

**"双缓冲"的含义与边界：** 旧代次在切换后**保留为 `retired` 且其 chunk 仍在**，直到超出 `keep-generations`（默认 2：当前 + 上一代）才被回收。保留上一代的价值是**快速回滚**：如果新代次发现质量问题时，可以再切换回上一代（把上一代从 retired 置回 committed），代价是"又切了一次"而不是"又重建一次"。回收策略：

```text
回收作业（每个副本可跑，删除是幂等的）：
  对每个 (backend, scope)：
    按 committed_at 倒序排列 committed/retired 代次
    保留前 keep-generations（2）个
    其余状态置 retired 并批量删除其 chunk + 代次行
  ⇒ 批量删除必须分批（与 7.2 同口径），避免大事务
```

**切换失败的处置：** 若"新代次置 committed"这一步失败（数据库不可用），则**不切换**：旧代次仍是 committed，新代次留在 building（等 TTL 过期后被清理并重来）。**绝不出现"两个 committed"**——这由部分唯一索引在数据库层保证（不是靠应用逻辑）。

### 12.6 存储面

**判断：chunk 存储必须放在共享状态层（关系的或文件面的共享存储），不能是进程内存。**

| 存储选项 | 集群档可用性 | 说明 |
|---------|-------------|------|
| **数据库表（推荐，阶段一默认的 SQLite chunk 存储在同构的 PostgreSQL 表里）** | ✅ | 与代次模型天然同源：chunk 带 `generation_id` 列、同一个事务里可校验；备份与恢复与审计一体 |
| **共享卷上的文件式 chunk 存储** | ✅（可行，但要自己实现分区与索引） | 索引元信息仍落库；chunk 本体按代次分目录（保留路径纪律）。适合"chunk 很大、数据库不想扛"的场景 |
| **进程内存** | ❌ | 每副本一份、重启即丢（3.2 矩阵第 3 项拒启动） |
| **本地文件** | ❌ | 每副本一份 |

**容量与清理（必须给出量级意识）：**

```text
chunk 存储量级估算：
  一个文档 50 页（约 5 万 token）→ 切分后约 100 chunk
  向量维度 1024（float32）→ 每 chunk 约 4KB 向量
  1000 个文档 → 约 10 万 chunk → 约 400MB 向量 + chunk 原文（通常更大）
  × keep-generations(2) ⇒ 约 0.8–2GB 量级
⇒ 结论：这个量级数据库能扛，但必须有清理作业，否则代次累积会线性增长。
   清理作业的删除速率必须可控（分批 + 观测），否则一次清理会打满 IO。
```

**索引与查询：** chunk 表必须有 `(generation_id)` 索引（检索按代次过滤是最高频路径）与 `(generation_id, document_id)` 索引（按文档统计覆盖）。**不做**跨代次的联合检索（那会让"恒读已提交代次"失去意义）。

### 12.7 边界

- **跨副本一致性 = 最终一致，且以代次为单位。** 检索在同一时刻在不同副本上可能返回不同代次的结果（副本 A 已看到新 committed、副本 B 还没读到）——这由它们的调用都是"读当前 committed"决定，窗口是轮询/请求时刻的差异。**不承诺"所有副本同一毫秒读到同一代次"**（那需要分布式事务）。
- **超大知识库的重建窗口。** 重建耗时与文档量线性相关（主要是 embedding 调用）：1000 个文档可能几十分钟到几小时。这个窗口内：①旧代次继续服务（检索不断服）；②新文档的导入被 409 挡住（12.4）⇒ 这是明确的代价，运维需要在"重建频率"与"导入及时性"之间取舍。**建议：把大库拆成多个 scope**（每个 scope 独立代次、独立认领）以缩小重建窗口——这是本方案提供的唯一扩容手段（不做分片重建）。
- **重建是昂贵的，因此不做自动频繁重建。** `knowledge` 域 bump 只表示"源变了"，不自动触发重建（11.2 的特殊性）。自动重建策略（如"源变更后 10 分钟自动重建一次"）放在后续，因为它的正确性取决于业务对索引新鲜度的要求。
- **不承诺检索结果的确定性。** 代次切换后，同一查询的结果会变（这是期望行为）；同一代次内、同一副本上，结果应该是确定的（这是阶段一双路召回 + RRF 的既有性质，集群档不改变它）。

**本方案不做：** 索引的增量代次（只做"整代重建"与"对当前代次的增量追加"两种）、跨 scope 的联合检索、向量索引的分布式分片、embedding 结果的跨代次复用缓存。

## 13. 数据持久化增量

### 13.1 Flyway 双轨纪律

**口径（逐字对齐 README 第 5 章末段）：** 集群档新增的 **9 张表必须在 `db/migration/sqlite/` 与 `db/migration/postgresql/` 两轨各写一份**（同版本号、只增不改）。SQLite 轨保留**结构等价的最小实现**，保证迁移序列对称、单机档与集群档的表结构可对照。**是否允许启用这些表由集群档启动校验决定**（3.2 矩阵第 1/2/3 项），而不是靠"只在一个 vendor 里建表"。

| 纪律 | 含义 | 理由 |
|------|------|------|
| 同版本号两 vendor 各一份 | `V7__cluster_tables.sql` 同时出现在 `sqlite/` 与 `postgresql/` 下 | 迁移序列对称；任何 vendor 下 `flyway_schema_history` 的版本集合相同 |
| 只增不改 | 已发布的迁移文件永不修改 | history 表保证恰好一次；改已发布的文件会造成"新环境与老环境的表结构不同" |
| V6 起非幂等干净 SQL | 沿用阶段一（V1–V5 是存量收敛序列，幂等） | 幂等 DDL 会掩盖真实的失败与重复执行 |
| sqlite 轨结构等价最小实现 | SQLite 轨的表结构表达同样的列与约束（受 SQLite 能力限制：部分索引、`ON CONFLICT` 的可用性不同，需用等价写法） | 单机档必须能正常启动与迁移；这批表在单机档**不被使用**，但结构在 |
| `hibernate.ddl-auto` 保持 `none` | 不依赖 Hibernate 自动建表 | 阶段一纪律；SQLite 的 `ALTER TABLE` 能力有限，自动迁移会失败 |
| 集群档表结构由启动校验把关 | 集群档启动时校验 9 张表存在且关键约束正确 | 把"用错了 vendor"变成启动期错误而不是运行期错误 |

> **为什么不做"集群档表只在 postgresql 轨落地"：** 看起来更省事，但它让两轨的迁移序列分叉（版本号相同、内容不同），运维无法用同一套"版本号对照"判断两个部署是否同构；且一旦未来某个集群档特性需要在单机档以退化方式支持，就会缺表。**结构对称 + 使用准入** 比 **结构分叉** 更容易维护，这是本方案的判断。

### 13.2 迁移清单

| 变更 | 对象 | 理由 |
|------|------|------|
| 新建 `instances` | 表 | 实例注册与心跳（第 9 章）；运维视图的唯一数据源 |
| 新建 `session_turn_leases` | 表 | 会话轮次跨副本恰好一次（第 5 章）；`session_id` 主键即互斥键 |
| 新建 `schedule_claims` | 表 | 定时触发跨副本恰好一次（第 6 章）；`(schedule_id, fire_time)` 主键即幂等键 |
| 新建 `event_receipts` | 表 | 入站事件回执去重（第 7 章）；替换进程内 Map |
| 新建 `channel_ownerships` | 表 | 渠道长连接属主（第 8 章）；避免互踢 |
| 新建 `workspace_versions` | 表 | 版本号总线（第 11 章）；六个域的初始行必须在此迁移中插入（否则 11.3 的 bump 会更新 0 行） |
| 新建 `admin_write_reservations` | 表 | 管理写互斥预约（第 10 章） |
| 新建 `knowledge_build_claims` | 表 | 知识重建认领（第 12 章） |
| 新建 `knowledge_generations` | 表 | 知识代次（第 12 章）；含 `UNIQUE (backend, scope) WHERE state='committed'` 部分唯一索引 |
| `tool_invocations` 加 `instance_id` | 列 | D13 跨副本审计追溯；可空（历史行为 NULL） |
| `llm_calls` 加 `instance_id` | 列 | 同上；可空 |
| `task_executions` 加 `instance_id` | 列 | 回答"哪个副本跑的这次触发"；可空 |
| `task_executions` 加 `fire_time` | 列 | Demo C 的断言依据（每个 `fire_time` 恰好一行）；可空（历史与手动补跑可为 NULL） |
| `schedule_claims` 的接管扫描索引 | 索引 | `(state, expires_at)`；没有它接管扫描会全表扫 |
| `session_turn_leases` 的续约/接管索引 | 索引 | `(owner_instance_id, expires_at)` |
| `event_receipts` 的 TTL 清理索引 | 索引 | `(expires_at)`；**必须独立**（范围删除不能靠复合索引第二列） |
| `knowledge_generations` 的作用域索引 | 索引 | `(backend, scope, state)`；支撑"读当前 committed"这条最高频查询 |

**新增列的三条纪律：** ①**必须可空或有默认值**（前向兼容，见 13.3）；②**不加 `NOT NULL` 约束到已有表**（旧行没有值，会失败）；③**不在同一个迁移里既加列又用它做唯一约束**（旧行可能重复）。

### 13.3 迁移顺序与向前兼容

**顺序（硬性）：先加表加列 → 再启动协调。**

```text
阶段 1（迁移，可与阶段一代码共存）：
  执行 V7：两轨各写一份（9 张表 + 新增列 + 索引）
  ⇒ 此时单机档代码路径不受影响（新表不被使用、新列不被写入）
  ⇒ 集群档代码路径**还不可启动**（缺启动校验、缺实现）
  ⇒ 关键：这个阶段可以**先上数据库、后上代码**，回滚时只需回滚代码

阶段 2（代码，逐副本滚动）：
  部署集群档代码；启动校验通过后开始写新表与新列
  ⇒ 新列的写入是"追加"语义，不需要回填历史行

阶段 3（可选，回填）：
  如需给历史审计行补 instance_id ⇒ 由运维脚本按"未知"或留 NULL 处理
  ⇒ 本方案**不做自动回填**（历史数据没有这个信息，编造值是错的）
```

**向前兼容的三条要求：**

1. **旧代码必须能容忍新列与新表的存在**（`ddl-auto=none` 下它根本不看，天然满足）。
2. **新代码必须能容忍新列的空值**（`instance_id` 为 NULL 的历史审计行必须能正常查询——查询与展示逻辑把 NULL 显示为"迁移前"）。
3. **回滚策略：只回代码，不回表。** 新增表与新列对阶段一代码是惰性的，所以回滚代码后系统能立即恢复阶段一行为；回滚表结构（`DROP TABLE`）会丢掉协调状态（这在故障时可能是唯一证据），且存在"回滚过程中有副本还在写"的风险。**因此本方案不支持回滚迁移。**

> **一条必要的例外说明：** 若迁移本身失败（V7 在中途报错），Flyway 会留下失败记录并阻止继续。此时**不能靠改 V7 的 SQL 重试**（违反"只增不改"），而应：①在测试环境复现并修复；②如已进入生产，则人工清理失败记录并**新增 V8 做补偿**（若 V7 部分成功留下半截结构）。这条流程写进运维手册。

### 13.4 连接池、超时与语句超时

**判断：认领语句必须是短事务；所有超时必须显式配置。** virtual thread 让并发线程数可以很大，但数据库连接是真实资源，**连接池是并发上限的真正约束**（见 17.1）。

| 配置 | 建议默认 | 理由 |
|------|---------|------|
| `spring.datasource.hikari.maximum-pool-size` | `20`（可配，按数据库规格与副本数调） | 集群档下**所有副本的连接数之和**必须远小于数据库的 `max_connections`（留出管理与备份连接的余量） |
| `minimum-idle` | `5` | 避免空闲期连接全部关闭后再集体重建 |
| `connection-timeout` | `3s` | 获取不到连接的等待上限；**必须短**，因为 fail-closed 下"等 30 秒"比"快速失败"更糟（请求堆积） |
| `idle-timeout` / `max-lifetime` | `10min` / `30min` | 配合数据库侧与中间件（PGbouncer/云数据库代理）的空闲回收，避免用到一个已被服务端关闭的死连接 |
| `validation-timeout` + `connection-test-query` | `2s` / `SELECT 1` | 防止把死连接交给业务（在共享库+代理环境下这类情况常见） |
| **语句超时**（`statement_timeout`，PostgreSQL 会话级） | `10s`（协调语句可更短，如 `3s`） | 保护：一条因为锁等待而卡住的认领语句会占住连接。**认领类语句必须是最短的那一档** |
| **事务超时**（`idle_in_transaction_session_timeout`） | `30s` | 防止"忘记提交"的连接长期占住行锁（这在实现 bug 时是灾难性的） |
| 数据库侧 `max_connections` | 按 副本数 × `maximum-pool-size` + 余量 规划 | 运维硬约束，写进 Helm chart 的注释与部署文档（16.9） |

> **"认领语句必须短事务"的两条判定：** ①一条认领 = 一条语句（或最多"认领 + 写审计"两条语句在一个事务里）；②**绝不**在事务里做任何 IO（LLM、HTTP、文件读写）。违反第二条会让一个"重建知识库"的事务持有行锁几十分钟——这会直接造成集群级的锁等待雪崩。

### 13.5 备份与恢复的运维要求

**判断：数据库备份与共享卷快照必须作为一对来规划，且"以数据库为准 + 工作区用版本号校准"是恢复的口径。**

| 对象 | 备份方式 | 一致性口径 |
|------|---------|-----------|
| PostgreSQL | 数据库侧既有的物理/逻辑备份（`pg_basebackup` / `pg_dump` 或云数据库快照） | 数据库内部自洽（事务保证） |
| 共享卷 | 存储侧快照（云盘快照/NFS 侧快照）或 `rsync` 到备份位置 | **不是跨文件原子**；可能捕捉到发布中途的状态（见 10.5） |
| **两者之间** | **无法做到原子**（不同系统） | 因此恢复时以**数据库为准**：用 `workspace_versions` 的版本号与盘上内容比对，并**必须调一次逃生舱 `POST /api/v1/workspace/refresh`**（见 11.5） |

**恢复的标准动作（写进运维手册）：**

```text
1. 停全部实例（10.8 的硬性前置条件）
2. 恢复数据库 → 恢复共享卷（顺序不影响，因为停机状态）
3. 校验：.workspace-id；关键目录（agents/skills/personas/knowledge/bootstrap）存在
4. 校验：workspace_versions 六个域都有行（若缺失 ⇒ 说明恢复到了过旧的快照）
5. 启动第一个副本 → 调 POST /api/v1/workspace/refresh（reason="从备份恢复"）
6. 逐项校验：instances 心跳、渠道属主唯一、一次会话消息、一次定时触发
7. 放开流量，观测 15 分钟（接管次数、bump 失败、重载失败三类指标）
```

**备份的一个反直觉要求：`admin_write_reservations` 的行不应该被恢复成"仍然存在"。** 备份里若带着一份陈旧的预约记录，恢复后所有管理写都会被 409 挡住（因为不自动过期）。恢复流程因此必须包含一步"清理陈旧的预约行（并记录原 note 供审计）"——这条来自"不自动过期"的设计，必须在手册里成对出现。

**本方案不做：** 持续归档（PITR）的配置、跨地域备份复制、应用层的"备份/恢复 API"。这些属于企业数据库与存储运维，本方案只规定**一致性口径**与**恢复后的校验动作**。

---

## 14. 可观测与审计

### 14.1 traceId 跨副本同源

**基线（阶段一已有）：** 一次消息处理 = 一个 `trace_id`；审计表（`tool_invocations` / `llm_calls`）带 `trace_id`；`SpanRecorder` 契约在 core，`oryxos.otel.endpoint` 不配置时走 NOOP，零开销；turn/llm/tool 三个 span 与审计同 `traceId`、同计时区间，事后补记；**turn 根 spanId = traceId 前 16 hex 的确定性父子规则**。

**集群档的增量（三条，不改基线）：**

1. **`traceId` 在认领成功后生成，不由请求入口生成。** 为什么：如果入口就生成，那么"认领失败的请求"（排队者、被 409 挡住的重复事件）也会产生一个 `traceId` 并可能写入审计的边角——那会让"一次处理 = 一个 trace"变得含糊，且重复事件会污染同一个 trace。**规则：只有真正开始处理的那一次才生成 `traceId`。**
2. **审计行带 `instance_id`（D13）。** 与 `trace_id` 一起构成"这条记录是谁在哪个副本上、属于哪一次处理"的完整定位。
3. **跨副本不可见的说明：** 一次处理**只在一个副本上完成**（会话轮次租约保证了这一点），所以 `trace_id` **天然不会跨副本**。真正的"跨副本链路"出现在两种情形：①**接管**（原副本做了一半、新副本标失败并写审计）——由 5.5 的两条关联线索拼接；②**同一会话的相邻轮次**落在不同副本——由 `session_id` + 时间区间拼接。**因此本方案不需要"分布式 trace 上下文透传"**（没有跨进程的调用链），只需要"同一副本内 trace 一致 + 审计带实例"。

> **这个判断很重要：** 它说明集群档的可观测需求**不是**"跨进程追踪"，而是"多副本日志/审计的可聚合"。因此本方案不引入 trace 上下文透传的协议细节（W3C traceparent 等），只在需要导出时按 OTel 的既有语义映射（14.5）。

### 14.2 审计行带 `instance_id` 的意义与查询口径

| 查询意图 | 查询写法（语义） | 用途 |
|---------|----------------|------|
| 这个副本在某时段做了多少工作 | `WHERE instance_id = ? AND created_at BETWEEN ? AND ?` | 容量与负载画像；确认"某副本是不是没在干活" |
| 这次处理是谁做的 | `WHERE trace_id = ?` | 单次处理的全量审计（多个副本不会同时出现，除了接管场景） |
| 这个会话的完整历史（跨副本） | `WHERE session_id = ? ORDER BY created_at` | 用户视角的对话审计 |
| 某个副本下线影响了哪些工作 | `WHERE instance_id = ? AND created_at > 下线时刻 - 租约TTL` | 故障影响面评估（Demo B/E 的排障路径） |
| 迁移前 vs 迁移后的数据 | `WHERE instance_id IS NULL` / `IS NOT NULL` | 区分历史行与集群档行 |

**写入口径：** `instance_id` 由审计写入路径从"当前处理的上下文"取（与 `trace_id` 同源），**不由每个 Tool 自己拼**——否则会出现"某个 Tool 忘了写"导致审计缺口。这也是把 `instance_id` 放进与 `traceId` 同一层的理由。

### 14.3 指标清单

| 指标名（语义） | 类型 | 含义 | 告警建议 |
|---------------|------|------|---------|
| `oryxos.claim.attempts`（按租约类型） | Counter | 认领尝试次数 | 无（基线） |
| `oryxos.claim.failures`（按租约类型） | Counter | 认领失败次数（影响 0 行） | 失败率 > 90% 且持续 5min ⇒ 可能有副本卡死在重试（对定时触发而言，N-1 个失败是正常的，所以要按"每副本"看） |
| `oryxos.claim.retries` | Counter | 认领语句的重试次数（瞬时故障） | 突增 ⇒ 数据库侧有问题 |
| `oryxos.lease.renew.failures`（按租约类型） | Counter | 续约失败（返回 0 行 = 已失去租约） | **> 0 即告警**：意味着双写风险窗口出现过 |
| `oryxos.lease.takeovers`（按租约类型） | Counter | 接管次数 | 突增 ⇒ 有副本在频繁假死/退出；持续 > 0 但无部署动作 ⇒ 排查 |
| `oryxos.turn.wait.duration` | Histogram | 会话轮次跨副本排队等待时长 | P99 接近 `session-turn-wait-timeout`（120s）⇒ 需要提高单会话并发或排查长轮次 |
| `oryxos.turn.wait.timeouts` | Counter | 排队超时次数（用户看到 409） | 持续 > 0 ⇒ 用户体验问题，需排查 |
| `oryxos.workspace.reload.duration`（按域） | Histogram | 重载耗时 | P99 远大于轮询间隔（1s）⇒ 轮询线程被拖住，需削减重载成本 |
| `oryxos.workspace.version.lag`（按域） | Gauge | 本副本已加载版本与表中版本的差（0 = 最新） | 持续 > 0 ⇒ 该副本一直落后（重载在失败） |
| `oryxos.workspace.reload.failures`（按域） | Counter | 重载失败次数 | 持续增长 ⇒ 工作区有坏数据（如 dangling 软连接、发布中途） |
| `oryxos.workspace.bump.pending` | Gauge | 落盘成功但 bump 未确认的域数（11.3 自愈机制） | > 0 持续 1min ⇒ 数据库写入异常 |
| `oryxos.channel.ownership.changes`（按渠道） | Counter | 属主变更次数 | 非部署期突增 ⇒ 属主在抖动（检查续约失败与网络） |
| `oryxos.channel.ownership.none` | Gauge | 当前无属主的渠道数 | **> 0 持续 30s 以上即告警**：有渠道没有人在听 |
| `oryxos.instances.active` / `.stale` | Gauge | 活跃/失联实例数 | `active` 小于期望副本数 ⇒ 容量下降；`stale` 持续 > 0 ⇒ 有实例没清理 |
| `oryxos.datasource.pool.usage`（active/idle/pending） | Gauge | 连接池饱和度 | `pending > 0` 持续 ⇒ **真正的并发瓶颈出现**（见 17.1） |
| `oryxos.event.receipts.rows` | Gauge | 回执表行数 | 偏离"速率 × 窗口"预期 ⇒ 清理失效或事件速率失控 |
| `oryxos.knowledge.generation.age` | Gauge | 当前 committed 代次的年龄 | 过老且 `knowledge` 域在 bump ⇒ 重建一直没成功 |
| `oryxos.knowledge.claims.conflicts` | Counter | 重建 409 次数 | 持续 > 0 ⇒ 重建频率过高或有人反复点重建 |
| `oryxos.agents.load.failures` | Gauge | 当前加载失败的 Agent 数 | > 0 即告警（坏 Agent，见 11.4） |

> **指标口径的两条纪律：** ①**所有协调指标必须带"租约类型/域/渠道"这类低基数标签**（标签基数爆炸是指标系统最常见的自伤方式，绝不能把 `session_id` 当标签）；②**"接管次数 > 0"本身不是故障**（滚动升级会造成接管），而**"续约失败 > 0"是故障**——两者的告警级别不同，必须区分。

### 14.4 日志规范

沿用阶段一的结构化 JSON（Logback + SLF4J），集群档下**每条日志必须携带的上下文**：

| 字段 | 何时出现 | 用途 |
|------|---------|------|
| `instance_id` | 所有日志 | 多副本日志聚合后按实例切分（ELK/Loki 的必需维度） |
| `trace_id` | 处理链路内的日志 | 与审计表对齐（14.1） |
| `lease_id` / 租约键 | 认领、续约、接管、失去租约类日志 | 回答"哪一次持有"（同一个键会有多个 lease_id） |
| `domain` | 工作区重载/bump 类日志 | 按域定位 |
| `channel_name` | 渠道属主类日志 | 按渠道定位 |
| `state_from` / `state_to` | 状态机迁移类日志 | 重构时序（见 14.6） |

**级别纪律（集群档新增的三条）：**

1. **"认领失败"是 INFO/DEBUG，不是 WARN 或 ERROR。** 在 N 副本下，定时触发的认领失败率天然是 (N−1)/N；把它记成 WARN 会让日志里 90% 是噪音。真正该 WARN 的是"已知有人应该持有但现在没人持有"（如 `channel.ownership.none`）。
2. **"失去租约（续约返回 0 行）"是 WARN。** 它意味着双写风险窗口出现过，必须显式记录并可在日志系统里告警。
3. **"接管"是 WARN（或 INFO，按团队口径），但必须结构化。** 它解释"为什么某个工作换人了"——这是排障清单的第一入口。

### 14.5 OTel 可选导出与零开销纪律

**判断：OTel 导出是可选增强，不配置则完全零开销（延续阶段一 `SpanRecorder` 的 NOOP 纪律）。**

| 项 | 口径 |
|----|------|
| 开关 | `oryxos.otel.endpoint` 为空 ⇒ `SpanRecorder` 装配 NOOP 实现，**不创建任何 span 对象、不做任何时间测量**（不是"创建了但不导出") |
| 导出内容 | turn / llm / tool 三类 span（阶段一契约不变），属性包含 `instance_id`、`trace_id`、`session_id`（注意：`session_id` 作为 span 属性会带来基数问题，需按团队的采样与存储成本决策，默认只放 `instance_id` 与 `trace_id`） |
| 计时区间 | 与审计表一致（事后补记，见阶段一契约） |
| 父子关系 | turn 根 span 的 `spanId` = `traceId` 前 16 hex（确定性规则，便于按 traceId 反查 span） |
| 不配置时的行为 | 零开销；所有日志与指标不受影响（可观测的两个层次独立） |
| 采样 | 由 OTel 侧配置（底座不做采样决策）；但**审计表不做采样**（原则五：审计 day one 全量落库，可观测的采样与审计的全量是两件事） |

> **为什么 span 与审计是两套而不是一套：** 审计是"必须精确、必须完整、必须落库"的合规资产；span 是"便于可视化、可采样、可丢弃"的观测资产。把两者合并会迫使其中一方让步（要么审计被采样，要么观测被迫承担全量存储成本）。本方案的选择是：**同源（同 traceId、同计时区间），但不合并存储。**

### 14.6 排障手册式清单

**症状一：消息重复回复。**

```text
按顺序查：
 1. event_receipts：这条 (channel, event_id) 有几行？每行的 instance_id 是谁？
    ├─ 2 行 ⇒ 不可能（主键冲突）。若真出现 ⇒ 检查是否有两个库（误配！看 3.2）
    └─ 1 行 ⇒ 去重生效，重复不来自入站重推
 2. session_turn_leases / sessions：这次的 turn_seq 有几条审计？
    ├─ 同一 turn_seq 有两条 assistant 记录 ⇒ 轮次租约失效：查
    │     · 是否有副本的 lease.renew.failures > 0（失去租约后没停）
    │     · 是否有副本的 instance_id 重复（3.2 矩阵第 8 项）
    │     · 是否 TTL 被配置得短于实际轮次耗时（误判崩溃 ⇒ 接管 ⇒ 原副本还在跑）
    └─ 两个不同 turn_seq 各有一条 ⇒ 这是两条真实消息，不是重复（去重不该拦）
 3. channel_ownerships 的历史：属主在这段时间变更过吗？
    └─ 变更时间内的事件可能被两个连接各收一次 ⇒ 由回执去重兜底（应已生效）
 4. 最后查 tool_invocations：重复的副作用（同一工具被调两次）出现在同一个 trace 还是不同 trace？
    ├─ 同一 trace ⇒ 业务代码问题（如重试没做幂等），不是协调层问题
    └─ 不同 trace ⇒ 回到第 2 步
```

**症状二：消息没有回复。**

```text
 1. event_receipts：有这行吗？
    ├─ 无 ⇒ 事件根本没到我们的处理路径：查渠道属主（channel.ownership.none 指标、
    │         GET /api/v1/instances 的 channels 字段）⇒ 可能所有副本都失联
    └─ 有 ⇒ 继续
 2. 这行的 expires_at 是否已过？处理是否失败于"已产生外部动作之前"（7.3 删除了回执）？
    ├─ 回执被删 ⇒ 应等平台重推；若平台不重推（长连接渠道，7.4）⇒ 用户可见失败：
    │            查审计里的错误、查 Provider 可用性
    └─ 回执保留 ⇒ 处理失败于"已产生外部动作之后" ⇒ 查 tool_invocations 里最后一条成功记录
 3. session_turn_leases：这一轮是否处于 failed？
    └─ 是 ⇒ 原副本崩溃，接管者已按 D6 标失败 ⇒ 告知用户重发（设计如此）
 4. sessions.messages_json：用户消息在、助手回复缺 ⇒ 半截历史，与第 3 步一致
 5. 若全无痕迹 ⇒ 检查该副本的 readiness/日志是否有数据库不可用（fail-closed 会拒服务）
```

**症状三：变更不生效（改了 `AGENT.md` / 绑了 Skill，某副本仍用旧版本）。**

```text
 1. workspace_versions：对应域的 version 变了吗？
    ├─ 没变 ⇒ 写入路径没有 bump：
    │     · 是管理面 API 写的吗？若不是（运维直接改盘/git pull）⇒ 用逃生舱 refresh（11.5）
    │     · 是 API 写的 ⇒ 查 bump.pending 指标与写路径日志（落盘成功但 bump 失败，11.3）
    └─ 变了 ⇒ 继续
 2. workspace.version.lag：落后的副本是谁？lag 是几？
    ├─ lag > 0 且持续 ⇒ 该副本的重载在失败：查 workspace.reload.failures 的域
    │     · agents 域失败常见原因：dangling 软连接、发布中途（10.5）、
    │       跨副本可见性延迟（NFS 属性缓存）
    │     · 该副本的 agents.load.failures 指标是否 > 0（坏 Agent 被跳过，11.4）
    └─ lag = 0 ⇒ 版本号推进且已重载，但**内容不对**：
          · 检查重载是否真的读了共享卷（本机执行视图/路径解析问题，10.3）
          · 检查是否读到了本地目录（误配：storage 不是 shared ⇒ 3.2 矩阵第 4 项）
 3. 延迟是否在 P99 ≤ 3s 口径内？超过 2 × poll-interval ⇒ 查 reload.duration 是否异常
```

**本方案不做：** 应用内建的诊断端点（如 `/actuator` 式的完整状态转储）、日志的分布式追踪注入、审计的可视化查询台（继承阶段一口径：审计写入 day one，查询能力后续）。

---

## 15. 安全与密钥

### 15.1 主密钥在集群档必须一致

**基线（阶段一已有）：** 落库凭证（`providers.api_key`、`notify_channels.config` 敏感项）经主密钥 AES-GCM 加密存储，密文带 `enc:v1:` 前缀；`ORYXOS_MASTER_KEY` 环境变量优先，缺省时首启在 `.oryxos/master.key` 自动生成；密钥不匹配时启动即拒并指路恢复。

**集群档的增量与风险：**

| 项 | 口径 |
|----|------|
| **必须一致** | 所有副本必须持有**完全相同**的主密钥；否则副本 B 解不开 A 写的密文（反之亦然），表现为"某个 Provider 在某些副本上可用、在另一些上报解密失败" |
| **偏好 Secret 注入** | 强烈建议 `ORYXOS_MASTER_KEY` 走 K8s Secret → 环境变量（禁止写进 `values.yaml` 明文、禁止写进镜像）；Helm chart 把它列为**必填密文引用**（16.3） |
| **缺省文件的多副本竞态** | 如果依赖"缺省 `.oryxos/master.key` 首启自动生成"，在集群档下会产生**并发竞态**：3 个副本同时首启、同时发现文件不存在、各自生成一份不同密钥并写入同一个共享卷路径 ⇒ 后写覆盖先写 ⇒ **有两个副本的进程内密钥与实际文件不一致**，且它们已经用它加密过数据 |
| **处理方式：原子创建 + 读回校验** | 生成流程必须：①在同目录写临时文件；②用**原子 rename 且不覆盖**的语义（`CREATE_NEW` 等价：若目标已存在则失败）；③失败则放弃自己生成、**读回**已有文件；④读回后用"能否解开库中既有密文"做一致性校验；⑤不一致 ⇒ 拒启动（3.2 矩阵第 7 项） |

> **集群档的推荐做法是"完全不依赖自动生成"**：由管理员显式创建 Secret（内容 = 一个足够随机的密钥），所有副本引用同一个 Secret。自动生成只在单机档与开发环境作为便利路径保留；集群档下它的正确性依赖上述原子创建协议，属于"能工作但不必依赖"的路径。

**轮转边界（本周期范围）：** 主密钥轮转需要"用旧密钥解密、用新密钥重新加密"的全量迁移，且必须在一个所有副本都使用新密钥的时刻切换——这需要一个两阶段流程（引入新密钥、完成重加密、移除旧密钥）与一个协调动作。**本周期不做轮转**：版本号 `enc:v1:` 的存在为未来引入 `enc:v2:` 留了空间，但轮转流程、双密钥并存期、以及"重加密过程中崩溃"的恢复都不在本方案内。**运维含义：主密钥一旦泄露，本周期内的处置方式是"更换所有凭证"而不是"轮转密钥"。**

### 15.2 加密字段在集群档的一致性与轮转边界

| 字段 | 集群档的一致性要求 | 原因 |
|------|------------------|------|
| `providers.api_key` | 同一行对所有副本可解 | Provider 调用可能落在任意副本（无粘性） |
| `notify_channels.config` 敏感项 | 同上 | 通知推送可能由任意副本发起 |
| 工作区里的 `${ENV_VAR}` 占位符 | 所有副本的环境变量必须一致 | 占位符在**读取时**从环境变量解析（阶段一语义），不是落库加密；不同副本的环境差异会造成行为差异 |

> **一条必须写进部署文档的纪律：** 加密字段解密失败时的行为是 **fail-closed**（该 Provider / 渠道不可用 + 告警），**绝不"跳过这个 Provider"**（那会让 LLM 调用悄悄路由到别的 Provider，产生难以察觉的行为变化与成本变化）。

### 15.3 工作区卷的权限模型

见 10.10（多副本同 uid、umask、rwx 卷上的权限实践）。集群档特有的三条补充：①**只读挂载不适用于集群档写者**；②**`fsGroup` 慎用**（大规模共享卷上 chown 可能很慢）；③**卷的属主与权限应在卷预置时确定**，而不是依赖 Pod 启动时修正——因为多副本会并发启动，任何"由应用自己修权限"的方案都会撞在一起。

### 15.4 集群档 API 面的安全边界

**诚实口径：阶段一是"内网无认证"假设，集群档**不改变**这一点，但必须把边界写明。**

| 项 | 本方案口径 |
|----|-----------|
| 认证 | **不做**（仍假设内网/受控网络）。理由：认证是横切所有端点的能力（会话、管理、运维、逃生舱），它与多租户、SSO、RBAC 是一套东西，拆开做一半会造成"部分端点有认证"这种更危险的状态（见阶段一技术方案 7.5 的同类判断） |
| 网络边界 | **必须在部署文档中明确**：OryxOS 的 HTTP 端口不得直接暴露到公网；必须由企业网关/Ingress 前置，并配置网络策略（NetworkPolicy）限制可达来源 |
| 网关/网络策略建议 | ①Ingress 层做 TLS 终止与来源限制；②NetworkPolicy 只允许"网关 → OryxOS 端口"与"OryxOS → 数据库/共享存储/Provider"；③**数据库不得对集群外暴露**；④管理类端点（含逃生舱 `POST /api/v1/workspace/refresh`）建议在网关层按路径做来源白名单 |
| 逃生舱的额外风险 | 它能强制全集群重载，属"运维能力"。**必须写进安全评估**：无认证下任何能访问该端点的调用方都能触发全集群重载（软性节流可缓解，但不能替代认证） |
| 实例视图的信息暴露 | `GET /api/v1/instances` 暴露 host/port/负载/属主。无认证下的对外暴露等于泄露内部拓扑 ⇒ 建议在网关层限制为运维网段 |
| 为什么认证仍属后续阶段 | 与阶段一定位一致：核心阶段/阶段二都假设受控网络；完整鉴权、多租户隔离、SSO 属于"横向能力"（见 `oryxos.md` 路线图），它们需要与租户模型一起设计，而不是在集群档里临时加一层 API Key |

> **这条边界必须显式写进企业交付文档**，否则会出现"因为它是企业级底座，所以它应该自带认证"的误判——本方案主动承认这个缺口，并把它列入后续阶段。

### 15.5 审计不可篡改的诚实边界

| 声称 | 是否成立 | 说明 |
|------|---------|------|
| 审计在应用层是"只追加"的 | ✅ | 底座不提供删除/修改审计行的接口；所有写入路径都是 INSERT |
| 审计带实例与链路标识 | ✅ | `instance_id` + `trace_id`（第 14 章） |
| 审计不可篡改 | ❌ | 任何拥有数据库账号的人都可以 `UPDATE` / `DELETE`；应用层的"只追加"约束在数据库层**没有**强制（没有触发器、没有权限分离、没有 WORM 存储） |
| 审计条目不可抵赖 | ❌ | 无签名、无哈希链，无法证明"这条记录未被删改" |
| 审计可用于合规取证 | ⚠️ | 只能作为**内部运营证据**；要用于合规取证需要额外的存储侧保护（只追加权限、WORM/对象锁、外部 SIEM 同步） |

**本方案的选择与理由：** 本周期不做防篡改加固（哈希链、只追加数据库账号、外部归档），但要**把边界写清**，避免"可审计"被误读为"不可篡改"。可选的低成本加固（写进运维建议、不进本方案的实现范围）：①给应用账号只授 `INSERT`（不给 `UPDATE`/`DELETE` 于审计表）；②定期导出到外部只追加存储；③数据库侧对审计表启用审计日志。**加固建议与实现范围的区分本身就是诚实的一部分。**

**本方案不做：** 审计的哈希链/签名、审计表的行级权限与只追加强制、SIEM 对接、合规报表。

---

## 16. 容器交付

### 16.1 官方 Helm Chart 的结构

```text
charts/oryxos/
├── Chart.yaml                  # 名称、版本、appVersion、kubeVersion 约束
├── values.yaml                 # 全部默认值（集群档默认开启，见 16.2）
├── values.schema.json          # 值的结构校验（缺失/类型错误在 helm 阶段就失败）
├── README.md                   # 安装前置条件、必填项、常见问题
├── templates/
│   ├── _helpers.tpl            # 命名、标签、selectorLabels、校验助手
│   ├── NOTES.txt               # 安装后输出：必填项自检、访问方式、后续步骤
│   ├── deployment.yaml         # 副本数、探针、preStop、grace、资源、反亲和、卷挂载
│   ├── service.yaml            # 集群内访问
│   ├── serviceaccount.yaml     # SA（最小权限）
│   ├── secret.yaml             # 可选：由 chart 生成 Secret（不推荐，见 16.3）
│   ├── configmap.yaml          # application.yaml 的覆盖片段（数据库三键走 Secret）
│   ├── pvc.yaml                # 工作区 RWX PVC（或用已有 PVC，见 16.4）
│   ├── pdb.yaml                # PodDisruptionBudget
│   ├── hpa.yaml（可选，默认关闭） # 水平扩缩（需与"加副本无需改配置"一致，16.7）
│   ├── networkpolicy.yaml（可选） # 见 15.4 的网络策略建议
│   └── tests/
│       └── smoke-test.yaml     # helm test：健康检查 + 实例视图 + 数据库校验
└── ci/                         # 门禁用的 values 组合（见 16.8）
    ├── values-minimal.yaml     # 最小可用（外部 PG + 已有 RWX PVC）
    └── values-kind.yaml        # kind 冒烟用
```

**结构上的两条判断：** ①**不把数据库打包进 chart**（不提供内置 PostgreSQL 子 chart 用于生产）——企业已有数据库与备份流程，chart 内嵌数据库会让"数据在哪、谁备份"变得模糊。测试用的 kind 环境可以在 chart 之外单独起一个 PG。②**工作区 PVC 既能由 chart 创建、也能引用已有 PVC**（生产环境通常由存储管理员预置，见 16.4）。

### 16.2 values 关键项表

| 组 | 键 | 默认 | 说明 |
|----|----|------|------|
| 副本 | `replicaCount` | `3` | 对齐 README 推荐 3 副本；`1` 表示退化为单实例（但集群档仍可选） |
| 镜像 | `image.repository` / `.tag` / `.pullPolicy` | `ghcr.io/oryx-labs/oryxos` / `appVersion` / `IfNotPresent` | 多架构（amd64/arm64），沿用阶段一镜像构建流水线 |
| 集群开关 | `cluster.enabled` | `true` | 官方 chart 的部署形态是集群档（16.3 的必填项与它配套） |
| 集群参数 | `cluster.instanceIdFrom` | `metadata.name`（StatefulSet）/ `metadata.uid` | Pod 名注入 `oryxos.cluster.instance-id`（见 3.1 规则 2） |
| 集群参数 | `cluster.*Ttl` / `*Interval` | 对齐第 3.1 节的统一默认参数表 | **不另造值**；如需调整，chart 的 values 与文档必须同步 |
| 数据库 | `database.url` | `""`（必填） | 指向外部 PostgreSQL；`jdbc:postgresql://…` |
| 数据库 | `database.existingSecret` | `""`（必填） | Secret 名，含 `username` / `password` 键 |
| 数据库 | `database.pool.maxSize` | `20` | 见 13.4；chart 的 NOTES 提示"副本数 × maxSize < 数据库 max_connections" |
| 主密钥 | `masterKey.existingSecret` | `""`（必填） | Secret 名，含 `master-key` 键（15.1） |
| 工作区 | `workspace.storage` | `shared` | 集群档默认共享 |
| 工作区 | `workspace.existingClaim` | `""` | 引用已有 RWX PVC（推荐） |
| 工作区 | `workspace.createClaim` / `.size` / `.storageClass` / `.accessMode` | `false` / `10Gi` / `""` / `ReadWriteMany` | 由 chart 创建时的参数（16.4） |
| 工作区 | `workspace.mountPath` | `/data/.oryxos` | 挂载点即 `nativePath` 的解析基础（10.3） |
| 探针 | `readiness.*` / `liveness.*` | 见 16.5 | readiness 含数据库校验；liveness 只看进程 |
| 资源 | `resources.requests` / `.limits` | requests `500m/1Gi`，limits 不设 CPU limit（见下） | 见 16.7 |
| 滚动 | `strategy.type` / `rollingUpdate.maxUnavailable` / `.maxSurge` | `RollingUpdate` / `0` / `1` | 对齐 README 统一默认参数 |
| 停机 | `terminationGracePeriodSeconds` / `preStop` | `40` / 见 16.6 | 与 `drain-timeout`、`timeout-per-shutdown-phase` 三者一致 |
| PDB | `podDisruptionBudget.minAvailable` | `1`（3 副本时也可用 `maxUnavailable: 1`） | 见 16.7 |
| 反亲和 | `affinity.podAntiAffinity` | 软反亲和（`preferredDuringScheduling`） | 见 16.7 |
| OTel | `otel.endpoint` | `""`（关闭，零开销） | 见 14.5 |
| 网络策略 | `networkPolicy.enabled` | `false`（需按企业环境开启） | 见 15.4 |

### 16.3 必填的两项密文引用

**必填仅两项（这是刻意的克制）：** ①**数据库三键**（url + username + password；url 可放 ConfigMap，用户名/密码必须 Secret）；②`ORYXOS_MASTER_KEY`。两者都走 **K8s Secret → 环境变量**（对齐阶段一的 `ORYXOS_*` 环境变量原生面）。

```text
【错误】把密文写进 values.yaml：
  database: { password: "s3cret" }        # ❌ 明文进 git、进 helm history
【错误】由 chart 自动生成主密钥：
  masterKey: { generate: true }           # ❌ 每次 helm upgrade 可能换密钥 ⇒ 密文全部解不开
【正确】引用已有 Secret：
  database:   { existingSecret: oryxos-db }
  masterKey:  { existingSecret: oryxos-master-key }
```

**缺失时的失败行为（必须快速失败且报错清晰）：**

```text
helm 阶段（首选）：values.schema.json + 模板内 required 校验
  ⇒ 缺少 database.existingSecret 或 masterKey.existingSecret ⇒
     "database.existingSecret is required: 请提供包含 username/password 的 Secret 名"
     （helm install/upgrade 直接失败，不会创建半成品资源）

运行阶段（兜底）：Pod 启动时缺环境变量 / 密钥无法解开库中密文 ⇒
  ⇒ 启动校验失败（3.2 矩阵第 7 项）⇒ 容器退出 ⇒ CrashLoopBackOff
  ⇒ 日志明确指路："提供 ORYXOS_MASTER_KEY（与既有部署一致），
     不要让它自动生成新的密钥"
```

> **绝不允许"找不到主密钥就自动生成一个新的"。** 那会让所有副本在每次重启后都无法解开既有密文，且**症状是"某些 Provider 报解密失败"而不是"启动失败"**——一类很难归因的故障。集群档下这个行为必须是硬失败。

### 16.4 工作区 RWX PVC 与访问模式

| 访问模式 | 集群档可用性 | 说明 |
|---------|-------------|------|
| `ReadWriteMany` | ✅ 必需 | 多副本同时读写同一卷 |
| `ReadWriteOnce` | ❌ | 只有一个节点能挂载 ⇒ 多副本部署必然失败（若副本都被调度到同一节点，则表现为"看起来能用"，一扩容就炸——这是最危险的形态） |
| `ReadWriteOncePod` | ❌ | 语义上排斥多副本 |
| `ReadOnlyMany` | ❌（对写者） | 只读挂载会让启动校验的原子写探针失败（10.10） |

**支持 RWX 的常见存储（只写有把握的事实）：** NFS 系（含各云的文件存储 NFS 协议）、CephFS、部分云的 NAS/文件存储产品。**不支持 RWX 的常见存储：** 大多数块存储（云盘/EBS/云硬盘）默认只支持 RWO。

**不支持 RWX 时的替代路径（按推荐顺序）：**

1. **换用支持 RWX 的文件存储**（首选；企业的 NAS 通常已有）。
2. **单副本 + 保持集群档开关打开**（`replicaCount: 1`，工作区用 RWO）。此时协调原语全部在工作但不产生跨副本竞争——**这不是"退化为单机档"**，而是"以集群档的代码路径跑单副本"，好处是未来加副本只需改一个数字且不需要重新迁移工作区。
3. **明确不做**：把工作区放在对象存储上挂载成文件系统（对象存储没有 rename 原子语义，见 D2 收敛对照二）、用 `hostPath`（节点绑定，多副本不可用）、用多个 RWO 卷各自一份（数据分裂）。

> **K8s 的一个常见坑：** 某些 CSI 驱动会"接受" `ReadWriteMany` 但在底层只支持单挂载，错误在运行时才出现（写入静默失败或第二个 Pod 挂载失败）。因此**部署前必须做一次真实的多副本挂载验证**（chart 的 NOTES 与部署文档要把这一步列为前置检查），不能只看 PVC 是否 `Bound`。

### 16.5 探针

**口径见 9.4（readiness 含数据库校验、liveness 只看进程）。chart 中的参数建议：**

```yaml
readinessProbe:                     # 检查：数据库读校验 + 工作区可读（.workspace-id）
  httpGet: { path: /api/v1/health/ready, port: http }   # 语义化的就绪端点（新端点，见下）
  initialDelaySeconds: 20
  periodSeconds: 10
  timeoutSeconds: 3
  failureThreshold: 3
livenessProbe:                      # 检查：进程是否卡死（不触碰数据库）
  httpGet: { path: /api/v1/health/live, port: http }
  initialDelaySeconds: 60           # 留足启动时间（Spring Boot + 工作区扫描 + Flyway）
  periodSeconds: 20
  timeoutSeconds: 3
  failureThreshold: 3
startupProbe（可选）:                 # 若启动可能较长，用 startup 保护 liveness
  httpGet: { path: /api/v1/health/live, port: http }
  failureThreshold: 30
  periodSeconds: 5
```

**两个端点的语义分工（阶段一只有 `GET /api/v1/health`，集群档需要拆开）：**

| 端点 | 检查 | 谁用 |
|------|------|------|
| `GET /api/v1/health` | 沿用阶段一：进程存活（不动，保持兼容） | 人工、docker healthcheck、旧监控 |
| `GET /api/v1/health/live` | 同 `health`：只反映进程（内部一个由请求线程推进的标记） | liveness probe |
| `GET /api/v1/health/ready` | 追加：数据库可达且 `workspace_versions` 可读；工作区可读且 `.workspace-id` 可读 | readiness probe、LB 健康检查 |

> **踩坑提醒（写进 chart 注释）：** **readiness 绝对不能检查"我是不是渠道属主"**（非属主副本永远不 ready ⇒ 滚动升级卡死，见 8.5 第 4 条）；**liveness 绝对不能检查数据库**（数据库抖动会导致全副本重启，见 9.4）。

### 16.6 滚动升级策略

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate: { maxUnavailable: 0, maxSurge: 1 }   # README 统一默认参数
terminationGracePeriodSeconds: 40
lifecycle:
  preStop:
    exec:
      # 主动前置摘流：先让 readiness 失败，再给 LB/端点控制器时间反应
      command: ["/bin/sh","-c","sleep 5"]
```

**升级窗口内消息的去向（Demo A 的核心，逐条写清）：**

```text
t0  新 Pod 就绪（readiness 通过）→ 加入端点 → maxSurge=1 保证旧 Pod 仍在
t1  旧 Pod 收到 SIGTERM：
    ├─ 立即：instances.status = 'draining'；readiness 失败
    ├─ preStop 的 5s：让 K8s 端点控制器把它从 Service 端点摘掉
    │   （副作用：这 5s 内它仍可能收到少量新请求 —— 这是 preStop 存在的意义：
    │     用 5s 换"摘流完成"，比"立即停服"造成的请求失败更划算）
    ├─ 停止认领新工作（新的会话轮次/定时触发被拒 ⇒ 负载均衡重试到其他副本）
    ├─ 排空在途（≤ 40s）：正在跑的会话轮次继续跑到结束（继续续约）
    ├─ ★ 释放渠道属主 ⇒ 其他副本在秒级内接管长连接（不必等 30s TTL）
    └─ 关闭长连接（限时 2s + 守护线程）→ 退出
t2  旧 Pod 终止；新 Pod 继续服务；重复直到全部替换
```

| 窗口 | 消息会发生什么 | 判据 |
|------|--------------|------|
| 旧 Pod 已摘流、尚未停止认领 | 新请求不会到达（已摘端点） | 无消息受影响 |
| 旧 Pod 停止认领、新 Pod 已就绪 | 请求路由到新 Pod | 正常处理 |
| 渠道长连接切换窗口（≤ 秒级，因为主动释放属主） | 平台在极短窗口内可能连接断开；重推由回执去重兜底 | **零重复回复**；是否有消息丢失取决于平台重推（8.7 的诚实边界） |
| 定时触发 | `(schedule_id, fire_time)` 唯一键保证恰好一次（无论在哪个 Pod 上执行） | `task_executions` 每个 `fire_time` 恰好一行 |

**三个参数的协同（必须一致，否则停机被 SIGKILL）：**

| 参数 | 值 | 关系 |
|------|----|------|
| `terminationGracePeriodSeconds` | 40s | K8s 给的总时间 |
| `oryxos.cluster.drain-timeout` | 40s | 应用内部的排空上限（等于 grace，留余量应更短，建议应用侧 35s） |
| `spring.lifecycle.timeout-per-shutdown-phase` | 10s | Spring 生命周期阶段的单阶段上限（阶段一已有；**注意 destroy-method 回调不受它约束**，长连接类资源必须各自限时关闭——阶段一飞书 SDK 的教训） |

> **口径：** 三者不是"随便配"，而是"应用内部的每一段都必须短于 K8s 给的总时间"。`drain-timeout` 建议配 **35s**（比 grace 少 5s）留出"释放属主 + 关连接 + 进程退出"的时间；若配成等于或大于 grace，K8s 会在应用还没走完第 ⑤ 步时 SIGKILL，**导致渠道属主要等 30s TTL 才被接管**（停机窗口被无谓拉长）。

### 16.7 PDB、反亲和、资源 requests/limits

| 项 | 建议 | 理由 |
|----|------|------|
| **PDB** | 3 副本时 `minAvailable: 2` 或 `maxUnavailable: 1` | 节点维护（drain）时保证至少 2 个副本在服务；注意 PDB **不能阻止** `kubectl delete pod`，它只管自愿中断 |
| **反亲和** | 软反亲和（`preferredDuringSchedulingIgnoredDuringExecution`，按 `topologyKey: kubernetes.io/hostname`） | 3 副本散到 3 个节点能容忍单节点故障；用**软**是因为节点数不足时硬反亲和会 Pending（可用性优先于分布） |
| **`resources.requests.cpu/memory`** | `500m` / `1Gi`（起步值） | 影响调度与 QoS；启动期（Spring + 工作区扫描）内存峰值较高 |
| **`resources.limits.cpu`** | **不设**（或设得很宽） | CPU limit 会造成 throttling，对"virtual thread 大量并发 + 短 IO"的模型伤害明显（表现为延迟抖动而非吞吐下降） |
| **`resources.limits.memory`** | 设为 requests 的 2–3 倍（如 `2Gi`） | 超限即 OOMKill。**注意 OOMKill 是"未优雅停机"**——租约会靠 TTL 过期被接管（而不是靠主动释放），因此内存 limit 不能设得过于贴紧 |
| **HPA** | 默认关闭 | 扩副本的正确性由 D1 与 2.4 保证（加副本无需改配置），但**扩容会触发渠道属主竞争与实例视图变化**，需要配合观测（14.3）；本周期先手动扩缩，HPA 作为可选（`hpa.enabled: false` 默认） |

### 16.8 门禁

```text
必须全绿才算通过（与阶段一 ci 的 docker-build 门禁同一风格）：

1. helm lint charts/oryxos
2. helm template charts/oryxos -f ci/values-minimal.yaml | kubeconform -strict
   └─ 覆盖至少两种 values 组合（最小可用 / kind 冒烟）
3. 模板断言（grep/脚本，防止静默回归）：
   ├─ Deployment 中存在 readiness 与 liveness，且路径分别为 /health/ready 与 /health/live
   ├─ readiness 不引用渠道属主相关路径（8.5 第 4 条）
   ├─ terminationGracePeriodSeconds == 40
   ├─ strategy.maxUnavailable == 0 && maxSurge == 1
   ├─ 主密钥与数据库密码来自 secretKeyRef（不得出现明文 value）
   └─ 工作区卷 accessMode == ReadWriteMany
4. kind 安装冒烟（ci helm job）：
   ├─ kind 起集群 + 起一个 PostgreSQL（chart 之外）
   ├─ 预置 RWX 卷（kind 下用 hostPath 多挂载或 NFS provisioner 模拟，
   │   并明确标注"这是冒烟用，不代表生产存储语义"）
   ├─ helm install → 等 Pod Ready → helm test
   │   ├─ /api/v1/health/ready 通过
   │   ├─ /api/v1/instances 看到 3 个实例且 status=ready
   │   └─ 一次 POST /sessions/{id}/messages 用 mock provider 走通
   └─ 缩容到 0 再升到 3（验证优雅停机与重启无残留）
5. 持久卷的"多副本真挂载"验证（人工/集成环境，见 16.4 的坑）
```

> **门禁的价值在于把"部署形态的约定"变成可执行的断言。** 上面第 3 条的那些断言看起来琐碎，但每一条都对应一个真实会发生的错：readiness 检查属主、grace 配错、accessMode 写成 RWO、密钥明文。**把它们写进 CI 比写进文档可靠。**

### 16.9 部署拓扑示例

```text
                        企业网关 / Ingress（TLS、来源限制）
                                     │
                                     ▼
                    Service: oryxos（ClusterIP，无粘性）
                    ┌────────────┬────────────┬────────────┐
                    ▼            ▼            ▼            │
              ┌──────────┐ ┌──────────┐ ┌──────────┐       │
              │ OryxOS   │ │ OryxOS   │ │ OryxOS   │       │
              │ Pod 1    │ │ Pod 2    │ │ Pod 3    │       │
              │(可能属主) │ │          │ │(可能属主) │       │
              └────┬─────┘ └────┬─────┘ └────┬─────┘       │
                   │            │            │             │
       ┌───────────┴────────────┴────────────┴─────────┐   │
       │                                              │   │
       ▼                                              ▼   │
┌────────────────────────┐              ┌──────────────────────┐
│ PostgreSQL（高可用）     │              │ RWX PVC（NFS/CephFS） │
│ 主 + 备 / 托管高可用     │              │  .oryxos/ 工作区      │
│ 9 张协调表 + 审计 + 会话 │              │  .workspace-id 已预置 │
│ 可用性下限由这一层决定   │              │                      │
└────────────────────────┘              └──────────────────────┘
       ▲                                              ▲
       │ 协调原语（CAS/租约/心跳/版本号）                 │ 文件面（原子落盘 + 轮询）
       └──────────────────────────────────────────────┘

外部依赖（在集群之外，由企业提供）：
  · LLM Provider API（显式映射 + fallback）
  · IM 平台长连接（飞书 / 企微 / 钉钉）—— 每渠道仅一个 Pod 持有
  · 企业监控（指标/日志采集）、Secret 管理、镜像仓库
```

**容量与可用性的下限由哪一层决定（必须对业务方说清）：** 副本数决定吞吐与"实例故障容忍度"；**PostgreSQL 的可用性决定整个集群的可用性下限**（D2 的代价，见 18.2）；共享存储的可用性决定文件面可用性。因此"99.9% 可用性目标（3 副本 + PostgreSQL 高可用）"的达成责任分布在三层，**不是"多起几个 Pod 就高可用"**。

**本方案不做：** 多集群 / 多区域的部署拓扑、服务网格集成、Istio/Gateway API 的具体配置、chart 内嵌数据库或存储 provisioner。

## 17. 性能与容量

### 17.1 virtual thread 与数据库连接池的配比

**判断：连接池是并发上限的真正约束，virtual thread 不是。** 这是集群档最容易误判的一处容量问题。

```text
virtual thread 模型下的真实约束链：

  请求 → virtual thread（便宜，可成千上万）
           │
           └─► 需要数据库连接（昂贵，池上限 20）
                 ├─ 拿到 ⇒ 执行（协调语句毫秒级）
                 └─ 拿不到 ⇒ 等待 connection-timeout（3s）⇒ 失败或重试
                            （此时 virtual thread 只是"挂起等待"，
                              它不消耗 OS 线程，但它确实在占着内存与请求时延）
```

**三条推论：**

1. **"加大 virtual thread 池"不会提升吞吐。** 池不存在（virtual thread 按需创建），能提升吞吐的只有"减少每次请求占用连接的时长"与"增加连接数（受数据库限制）"。
2. **协调查询成为新的连接消费者。** 阶段一每个请求只做"读会话 + 写审计"；集群档追加了"认领租约 + 续约 + 轮询版本号 + 心跳"，每一条都要连接。**这就是为什么续约与轮询必须在专用调度线程上同步完成、且语句必须是短事务**（4.5、13.4）：把 N 个业务线程各自的协调开销收敛到一个线程上的固定频率，总连接占用可预测。
3. **锁等待是最危险的连接占用。** 一条因为行锁等待而卡住 30s 的语句会占住一个连接 30s；20 个连接池只需 20 条这样的语句就完全瘫掉。因此：①所有协调语句设短 `statement_timeout`（3s）；②绝不在事务里做 IO；③**绝不用 `SELECT ... FOR UPDATE`**（4.3）。

**配比建议（经验值，必须按实测调整）：**

```text
maximum-pool-size ≈ 单副本目标并发在处理中的请求数 × 0.3 ~ 0.5
  理由：一个请求的数据库占用是"几次短语句"，不是"全程占用"
        （LLM 调用与 Tool 执行期间不占连接——这是 virtual thread 模型的优势）
  例：目标 50 消息/秒、单请求数据库占用约 30ms ⇒ 平均占用连接数 ≈ 1.5
      ⇒ 池 20 有充足余量，瓶颈会先出现在数据库 CPU 而不是池
运维硬约束：副本数 × maximum-pool-size + 管理余量 < 数据库 max_connections
  例：8 副本 × 20 = 160 ⇒ 数据库 max_connections 至少 200
```

> **观测口径：** `oryxos.datasource.pool.usage` 的 `pending > 0` 持续出现，才是"池成为瓶颈"的判据（14.3）。在那之前优化池是无效的；在那之后优化池（或减少协调语句）才有效。

### 17.2 轮询与心跳的开销模型

**默认参数下的量级估算（N 副本、D = 6 个域）：**

| 来源 | 频率 | 每次的语句数 | N=8 时的语句/秒 | 备注 |
|------|------|------------|----------------|------|
| 版本号轮询 | 每副本 1/s | 1 条 SELECT（一次取 6 行） | 8 | **不以域为单位查询**，这是关键设计 |
| 实例心跳 | 每副本 0.1/s（10s） | 1 条 UPDATE | 0.8 | 可忽略 |
| 租约续约（会话轮次） | 每活跃轮次 0.05/s（20s） | 1 条 UPDATE | 取决于并发轮次 | 1000 并发会话 ⇒ 50 语句/秒 |
| 渠道属主续约 | 每属主 0.1/s（10s） | 1 条 UPDATE | ≤ 渠道数 × 0.1 | 可忽略 |
| 定时触发扫描/认领 | 每副本按 tick | 1 条 SELECT + 触发时 1 条 INSERT | 与任务数相关 | N 副本各扫一遍 ⇒ 空转成本 |
| 回执清理 | 每副本低频 | 1 条 DELETE（LIMIT 1000） | 可忽略 | 需限速以免 IO 抖动 |
| 会话轮次认领 | 每请求 | 1–2 条 | = 请求速率 | 与业务同阶 |
| 审计写入 | 每 LLM/Tool 调用 | 1 条 INSERT | 与调用数同阶 | 阶段一已有 |

```text
结论（默认参数，8 副本）：
  背景协调负载 ≈ 8（轮询）+ 50（1000 活跃轮次的续约）+ 约 10（其他）
             ≈ 70 语句/秒
  对比业务负载（50 消息/秒 × 每消息数次 LLM/Tool 审计写入）
             ≈ 数百语句/秒
  ⇒ 背景协调开销在**同一量级或更低**，可接受。
  ⇒ 主要成本是"轮询的 N 倍放大"与"续约随活跃轮次数线性增长"，
     两者都有明确的扩容路径（17.5）。
```

**三条降低开销的手段（按性价比排序）：** ①**轮询合并成一条查询**（已设计为一次取 6 行）；②**续约合批**（同一个 tick 里对多个租约的续约可以合并为一条 `UPDATE ... WHERE ... IN (...)` 或批量语句——但要保持"每条独立判定"的语义，不能因为合批而混淆哪条成功）；③**增大间隔**（`poll-interval` 从 1s 到 2s 直接减半轮询负载，代价是传播上界从 2s 变 4s，与 P99 ≤ 3s 的目标冲突 ⇒ 需要业务确认）。

### 17.3 数据库热点与索引设计

| 热点 | 表现 | 缓解 |
|------|------|------|
| `session_turn_leases` 的认领 | 高并发会话下主键冲突频繁（同一会话的排队者反复尝试） | 排队间隔带抖动（5.3）；等待者睡眠 500ms 量级 ⇒ 冲突率可控；主键索引使冲突检测极快 |
| `schedule_claims` 的认领 | N 副本在同一 `fire_time` 上写同一主键 ⇒ N−1 次冲突 | 任务数 × 频率决定；冲突是主键级、成本低；串行化风险在于同一毫秒大量任务 ⇒ 建议避免"整点集中大批任务"（写进任务编写建议） |
| `event_receipts` 的清理 | 大批 DELETE 造成 IO 抖动与锁等待 | `purge-batch` 分批（1000）+ 每副本随机错峰 + `(expires_at)` 独立索引 |
| `workspace_versions` 的轮询 | 每副本 1 条/s 读 6 行 | 极小；表只有 6 行，全表在缓存里 |
| 审计表的持续增长 | 查询变慢、清理成本上升 | 按 `created_at` 分区或定期归档（运维侧；阶段一已有增长，不是集群档新增问题） |
| `knowledge_generations` 的检索查询 | 每次检索都要读"当前 committed" | `(backend, scope, state)` 索引使其成为一次索引命中 |
| chunk 表的检索 | 代次过滤 + 向量距离计算 | `(generation_id)` 索引是必须；向量计算是 CPU 热点（不是数据库热点） |

**过期扫描的部分索引（可选但推荐）：** 对"只扫未过期/只扫已过期"的两类查询，用部分索引（`WHERE expires_at < CURRENT_TIMESTAMP` 不能用（非 immutable），但 `WHERE state IN ('claimed','running')` 可以）缩减索引体积。**注意：任何依赖 `CURRENT_TIMESTAMP` 的部分索引在 PostgreSQL 中都不合法**（函数非 immutable），所以过期扫描只能靠普通索引 + 范围条件。

### 17.4 压测方法

```text
环境与工具：
  · mock provider 固定时延：-Doryxos.mock.latency-ms（阶段一已有），
    把 LLM 侧不确定性排除（README 第 6 章口径）
  · 副本数梯度：1 / 2 / 4 / 8（验证加速比）以及上限 32（验证退化点）
  · 后端：真实 PostgreSQL（不用嵌入式替代品）+ 真实 RWX 存储（不用本地目录）

混合负载（三类同时打，比例可配，建议 8:1:1）：
  A. 会话消息：POST /sessions/{id}/messages
     - 两种模式：①大量不同会话（测吞吐）②少数会话高并发（测轮次租约排队与冲突）
  B. 定时触发：注册若干每分钟/每小时的 cron（测到点认领与唯一性）
  C. 渠道事件：向入站接口注入事件（含 10% 重复事件，测回执去重）
  D. 工作区变更：每隔几秒改一次 AGENT.md 并 bump（测传播延迟与重载开销）

观测项（必须都采）：
  吞吐：消息/秒、集群总吞吐、每副本吞吐
  时延：端到端 P50/P99、会话轮次排队等待 P99
  正确性：重复执行计数（应当为 0）、每个 fire_time 的 task_executions 行数（应恒为 1）、
          重复事件被拦下的次数（应等于注入的重复数）
  协调：认领失败率、重试次数、续约失败次数（应为 0）、接管次数
  传播：workspace.version.lag 的分布（所有副本的最大值）
  资源：连接池 active/pending、数据库 CPU/IOPS/锁等待、共享卷 IOPS
```

**方法纪律：** ①**先测单副本建立基线**（否则无法算加速比）；②**每个梯度至少跑 10 分钟稳态**（排除预热与迁移期）；③**正确性指标与性能指标同时采集**——"快但错了"是最坏的结果，且高并发下的正确性缺陷恰恰只在压测里暴露；④**测"副本猝死"**（压测中途 kill 一个副本）与"滚动升级"（压测中途做一次滚动），因为这两个场景才是集群档真正的价值所在。

### 17.5 目标与瓶颈预判

| 目标（README 第 6 章口径） | 预判瓶颈 | 扩容路径 |
|--------------------------|---------|---------|
| 2–8 副本（推荐 3），验证上限 32 | 副本 > 8 后轮询与续约的 N 倍放大开始显著；数据库连接数之和逼近 `max_connections` | 增大 `poll-interval`（换传播延迟）；减少每副本连接池；读副本分流只读查询 |
| 单集群 Agent ≥ 100 | `agents` 域重载成本随 Agent 数线性增长；每个 Agent 的软连接校验有 IO 成本 | 重载增量化为"只重扫变更的 Agent"（需要更细的版本粒度，本周期不做）；减少轮询频率 |
| 单集群并发 Session ≥ 1000 | 续约语句数随活跃轮次线性增长（≈ 50/s）；会话行本身的读写 | 续约合批；连接池与数据库规格提升；会话归档 |
| 单副本 ≥ 50 消息/秒（mock provider） | 主要是数据库（审计写入 + 认领）与 virtual thread 调度 | 数据库规格；审计写入批量化（阶段一已提过"压测发现瓶颈再优化成批量落盘"） |
| 2 → 4 副本加速比 ≥ 0.8 | 共享数据库的写竞争（审计表插入是主要写路径） | **这是最可能的失分点**：审计是"每请求必写"的强共享写入，4 副本时数据库写能力可能成为天花板。缓解：批量插入、审计表分区、必要时把审计写入改为"有界队列 + 批量刷"（然而这会碰"审计 day one 可信"的边界，需谨慎评估） |
| 变更传播 P99 ≤ 3s | `agents` 域重载耗时（大工作区）+ 轮询间隔 | 提高轮询频率（代价见 17.2）；优化重载增量性 |
| 渠道属主接管 ≤ 30s（过期后） | 属主扫描 tick + 建连时间 | 缩短扫描 tick；主动释放（正常停机路径已是秒级） |
| 会话轮次接管 ≤ 60s（过期后） | TTL 是主项（60s） | **TTL 不能无限调小**（必须 > 单轮 P99）；可行的改进是"续约更频繁 + TTL 更短"，但要求单轮 P99 稳定 |
| 可用性 99.9% | **数据库可用性**（D2 的代价） | PostgreSQL 高可用（主备自动切换）、连接池快速失败与重连、fail-closed 避免脑裂 |

> **最需要提前预警的一条：** 审计表的写入是本方案里"共享写入压力最大、又最不能省"的一条路径（原则五）。**4 副本以上的加速比失分最可能出在这里**，因此 17.4 的压测必须把"审计写入速率"作为一等观测项，而不是只看端到端吞吐。

### 17.6 容量规划建议

| 维度 | 建议区间 | 依据 |
|------|---------|------|
| 副本数 | **3**（生产起步）；峰值确认后再加到 4–8 | 3 副本容忍 1 个实例故障且保留多数容量；超过 8 副本的协调开销开始不划算（17.2） |
| 数据库规格 | 起步：4 vCPU / 16GB / SSD，`max_connections ≥ 200`；随副本数与审计量提升 | 协调语句是轻负载但**延迟敏感**；审计写入是主要 IO |
| 数据库高可用 | **必须**（主备 + 自动切换） | 它是集群可用性的下限（18.2） |
| 共享存储 | 支持 RWX、低延迟（同可用区）、容量按"工作区 × 2（含 .staging 与产物）" | 工作区读写是 Agent 每轮的路径（Skill 正文按需读、脚本执行） |
| 连接数规划 | 副本数 × 20 + 50（管理/备份/监控余量） | 13.4 的硬约束 |
| 单 Agent 目录规模 | 保持"每 Agent 数十个文件"量级 | 影响 `agents` 域重载耗时（17.5） |
| 回执表 | 行数告警阈值 = 峰值事件速率 × 24h × 1.5 | 14.3 的指标 |

**本方案不做：** 容量自动伸缩（HPA 的完整策略与预测性扩容）、分库分表、多区域容量模型（D14 线外）。

---

## 18. 故障模型与恢复语义

### 18.1 故障清单

| 故障 | 检测手段 | 影响面 | 恢复方式 | 恢复步骤 |
|------|---------|--------|---------|---------|
| **实例崩溃（kill -9 / OOMKill）** | 心跳过期（`stale`）；租约 TTL 到期 | 该实例持有的：渠道连接、在跑的会话轮次、已认领的定时触发、知识重建 | 自动 | 其余副本按各自 TTL 接管；崩溃轮次标 failed（D6）；渠道属主秒级（若主动释放过）或 ≤ 30s；用户重发恢复 |
| **网络分区（实例 ↔ 数据库）** | 心跳失败、认领与续约失败、连接池 pending | 分区侧的实例**无法做任何正确工作**（fail-closed，18.3） | 自动（分区恢复后） | 分区侧拒服务（readiness 失败被摘流）；分区恢复后自动重连、重新认领；期间它在跑的工作因续约失败被中止，租约被对侧接管 |
| **数据库不可用（主备切换 / 宕机）** | readiness 全副本失败；所有协调操作报错 | **全集群不可服务**（D11 fail-closed） | 自动（数据库恢复后）+ 人工（若需切主） | 应用侧：所有副本 fail-closed，请求快速失败；数据库恢复后自动恢复；**不做单机降级** |
| **共享卷只读 / 未挂载** | 启动探针原子写失败；运行期写入报错 | 该实例无法做工作区相关工作（记忆、文件 Tool、管理写、重载） | 依赖存储侧 + 人工 | 修复挂载/权限；应用侧 fail-closed 并告警；**不回退本地**（10.2） |
| **共享卷延迟抖动（NFS 卡顿）** | 重载耗时上升、文件 Tool 超时、软连接校验变慢 | 用户感知为"Agent 变慢"；可能触发重载失败重试 | 自动（抖动过去后） | 重载失败重试；`workspace.version.lag` 上升是可观测信号；不需人工介入除非持续 |
| **时钟漂移** | **不影响正确性**（一切 TTL 用数据库时间，4.3）；仅影响日志时间戳与 `fire_time` 的显示 | 日志时间线可能错乱 | 自动（NTP 收敛） | 无需动作；排障时以数据库时间与 traceId 为准，不信任应用日志时间 |
| **双属主窗口**（见 18.4） | 渠道侧可能收到两个连接；日志有接管记录 | 极小窗口内同一事件可能被两个连接各收一次 | 自动 | 回执去重兜底（第 7 章）；无需人工 |
| **主密钥不一致** | 启动校验：无法解开库中密文 ⇒ 拒启动（3.2 矩阵第 7 项） | 该副本不能启动 | 人工 | 统一注入同一 Secret；**绝不让它自动生成新密钥**（15.1） |
| **迁移中断（Flyway 失败）** | `flyway_schema_history` 有失败记录；启动被阻止 | 不能升级；若已升级部分结构，新旧代码可能都不满足 | 人工 | 测试环境复现 → 修复 → 若已进生产则人工清失败记录 + 新增补偿迁移（13.3 的例外说明） |
| **`instance-id` 重复** | 启动时心跳冲突检测（3.2 矩阵第 8 项）；实例视图出现"心跳在跳但负载不对" | 运维视图失真；属主/租约归属判断可能混淆 | 人工 | 修正 ID 来源（16.2 的 `instanceIdFrom`）；重启该副本 |
| **工作区损坏（误删/半截发布/软连接成片 dangling）** | `agents.load.failures` 上升、重载失败、用户报错 | 部分或全部 Agent 不可用 | 人工（按 18.5） | 停全部写者 → 恢复 → 校验 → 重启 → refresh |
| **知识重建中途崩溃** | 认领 TTL 过期 ⇒ 接管者判定失败 | 该作用域停留在旧代次（检索仍可用） | 自动 + 人工重试 | 接管者删除失败的 building 代次；重新触发重建（12.3） |

### 18.2 脑裂边界

**判断：本方案不会出现双写脑裂，因为写入是单主串行的——不是因为我们有选举机制，而是因为所有互斥都收敛到"单主数据库上的一次条件写"。**

```text
为什么不会双写（三条论证）：

1. 所有互斥的判定点唯一：PostgreSQL 的某一行。
   两个副本要"都认为自己持有"，必须让同一条 UPDATE/INSERT 对两者都返回 1 行——
   在单主数据库上，同一行的两个并发条件写不可能都成功（行锁 + 重新求值）。
   ⇒ 互斥的正确性来自数据库，而不是来自任何应用层的一致性协议。

2. 我们**没有**"多数派"这个决策点，因此没有"分区两侧各以为自己是多数派"的可能。
   脑裂的经典成因是"两个分区各自选出领导者"；本方案没有选举、没有领导者，
   只有"谁拿到了那一行"。
   ⇒ 网络分区只会造成"有的人拿不到锁（拒服务或排队）"，不会造成"两个人都拿到"。

3. fail-closed 切断了"降级路径"。如果分区侧副本选择"连不上数据库就用本地状态继续跑"，
   那才会产生双写——而 D11/18.3 明确禁止这种降级。
```

**代价（必须承认）：** 这个设计的另一面是**可用性完全绑定数据库**。数据库主库不可写时，整个集群无法认领任何工作（会话轮次、定时触发、渠道属主、重建全部停摆），表现是"全集群同时不可服务"。这就是 D2"少一个中间件"换来"数据库成为协调单点"的诚实代价。缓解手段全在数据库侧：PostgreSQL 高可用（同步/异步流复制 + 自动故障转移）、多可用区部署、以及把连接池配置成"故障时快速失败"（13.4）而不是"长时间阻塞"。**这也是 README 第 6 章"可用性下限由数据库侧决定"的确切含义。**

**本方案不做：** 多主数据库、跨地域同步、分区容忍（D14 线外）；也不做"分区侧降级为只读服务"（只读服务在 Agent 场景里几乎没有价值——读会话历史可以，但那不是主要用途）。

### 18.3 fail-closed 原则（D11）

**判断：数据库不可用时，集群档拒服务，绝不降级为单机。**

```text
行为矩阵：
  数据库可达     ⇒ 正常服务（认领、续约、轮询、审计全部工作）
  数据库不可达   ⇒ ①readiness 失败（被摘流）
                   ②已到达的请求快速失败（返回 503 + 可读原因）
                   ③不再认领新工作（认领会失败，不要重试到打满）
                   ④继续尝试重连（有界退避）
                   ⑤**绝不**使用本地缓存做决策、**绝不**降级为单机模式

  共享卷不可达   ⇒ 同上（工作区是另一半状态）
```

**为什么"降级为单机"是绝对不可接受的（三条）：**

1. **"单机"在集群档里没有明确定义。** 我是 N 个副本中的一个：我该处理全部会话（包括其他副本正在处理的）吗？我该接管所有渠道连接吗？我该假定其他副本已经死了吗？任何回答都会制造一个**和真实拓扑不一致的假象**。
2. **降级期的写入无法合并。** 降级期间我如果用本地 SQLite 接受写入（会话、审计），数据库恢复后这些写没有合并路径（同 10.2 的数据分裂论证）。
3. **降级会掩盖故障。** 用户看到"能用"，运维看到"服务正常"，而实际上系统已经失去了它的全部正确性保证。**fail-closed 的价值就是把"不可见的数据损坏"换成"可见的不可用"**——后者可告警、可回滚、可解释。

**"快速失败"的具体口径（避免实现成"长时间等待"）：** ①连接获取超时 3s；②协调语句 `statement_timeout` 3s；③认领重试上限 3 次、退避到 2s 封顶 ⇒ 最坏路径在 10s 量级内返回失败，而不是挂 30s+。理由：fail-closed 下等待没有收益（数据库不会在 3s 内突然恢复的概率很高，而请求堆积会拖垮恢复后的第一波请求）。

### 18.4 双属主窗口的窗口期分析

**窗口的定义：** "旧持有者还活着但租约已过期"的那段时间。在这段时间里，新持有者会通过 CAS 接管——**因为接管只看时间，不看旧持有者是否真的死了**。因此必然存在一个（通常极小的）时间段，两方都认为自己在持有。

```text
窗口的构成（以渠道属主为例，TTL=30s、续约=10s）：

  t0  旧属主最后一次成功续约（expires_at = t0+30s）
  t1  旧属主因网络抖动/长时间 GC 未能续约（t1 ≈ t0+10s）
  t2  = t0+30s：租约到期。从这一刻起，**新属主有权接管**
  t3  = t2 + 扫描 tick（≤ 秒级）：新属主 CAS 成功
  t4  = t3 + 抖动（0–2s）：新属主开始建连
  t5  = 旧属主从暂停中恢复，它的下一次续约返回 0 行 ⇒ 它发现自己已失去
      （t5 可能远晚于 t3：它可能在 t5 = t0+90s 才醒）

  窗口 = [t2, t5]（最坏情况），通常 = [t2, t3+建连]（旧属主确实死了）
```

**为什么这个窗口可以接受（四条）：**

1. **窗口内旧持有者的行为是受限的。** 它的续约返回 0 行 ⇒ 按 4.5 第 3 条自降级：置失去标志、业务侧在检查点中止。所以窗口内的"双持有"是**旧方的残余执行**（已经跑到一半的那一步），不是"旧方发起新的工作"。
2. **窗口内的后果在渠道场景里是"同一事件可能被两个连接各收一次"**，而这条路径由**回执去重**兜底（第 7 章）——这是本方案特意保留的"第二道防线"，它的存在正是为了吸收这类窗口。
3. **窗口在会话轮次场景里的后果是"两个副本可能都在跑同一个会话"**——但这**不会发生**，因为：旧方的轮次在失去租约后必须在检查点中止；而它已经产生的副作用（已写审计、已发通知）**不会被重放**（D6：接管者标 failed 不重放）。所以最坏结果是"用户收到一条重复的回复"（若旧方恰好在失去租约前完成了发送），而不是"两条不同内容的回复互相覆盖"。
4. **缩短窗口的手段存在，但都不值得本周期做。** ①更短的 TTL（但必须 > 单轮 P99，见 3.1 第 4 条）；②更频繁的续约（更多数据库负载）；③"旧持有者主动放弃"需要它活着且能与数据库通信（恰恰是它出问题时做不到的）；④fencing token 在**下游系统**生效（让收到旧方写入的系统拒绝它）——这是唯一真正严格的解法，但它要求每个下游（IM 平台、工单系统）都支持幂等键，**超出底座能控制的范围**。本方案的做法是：**用 `lease_id` 做数据库侧的栅栏（防止旧方续约/释放成功），用回执去重做业务侧的兜底，并把窗口写清而不是假装它不存在。**

> **诚实结论：** 双属主窗口不可完全消除（在"持有者可能假死"的世界里，任何基于超时的租约都如此）。本方案的承诺是：**窗口内的数据库侧写入被 `lease_id` 栅栏挡住不再生效；窗口内的业务副作用由幂等键与回执去重吸收；窗口的存在可观测（接管次数、续约失败次数指标）**。

### 18.5 工作区损坏恢复流程

```text
前置：按 10.8 停掉**全部**写者（不只是持预约的那个）

 1. 确认静默
    ├─ GET /api/v1/instances：无 ready/starting 实例
    └─ 共享卷上有各实例的停写痕迹（10.9）
 2. 申请管理写互斥预约（scope=workspace-restore，写清 note）
    └─ 被占用 ⇒ 先弄清上一次恢复是否完成（note 里有线索）⇒ 不允许"抢"
 3. 备份当前（即使已损坏）
    └─ 把当前工作区打包留档（调查用；不要在没留档的情况下覆盖）
 4. 恢复内容（按损坏类型）
    ├─ 半截发布 ⇒ 按 .staging 的清单判定"继续完成"或"整批放弃"（10.5）
    │              ⇒ 只影响被发布的那个 Agent，不需要全量恢复
    ├─ 软连接成片 dangling ⇒ 逐个重新绑定（或从备份恢复 skills/ 与 agents/）
    ├─ 误删目录 ⇒ 从备份恢复；缺失的 bootstrap 文件可从模板重建
    └─ 全卷损坏 ⇒ 从备份整体恢复
 5. 校验（必须逐项过）
    ├─ .workspace-id 存在且与 oryxos.workspace.id 匹配
    ├─ 关键目录存在：agents / skills / personas / knowledge / bootstrap 相关
    ├─ 软连接自检：无 escaped、无 name-mismatch（dangling 若有则记录并接受，
    │    因为可能只是可见性延迟 —— 但必须记录并观察）
    └─ workspace_versions 六个域都有行
 6. 释放管理写预约（显式 DELETE）
 7. 启动第一个副本 → 调 POST /api/v1/workspace/refresh（reason="工作区损坏恢复"）
    └─ 为什么必须调：恢复的内容绕过了 bump，表里的版本号与盘上内容不一致
 8. 校验：instances 心跳、agents.load.failures == 0、一次会话消息、一次定时触发
 9. 启动其余副本，放开流量，观测 15 分钟
```

**版本号回退的专门处理：** 若恢复的备份里 `workspace_versions` 的版本号**小于**当前值（例如恢复到较旧的快照），各副本会比较"我加载的 > 表中值" ⇒ 判定为落后 ⇒ 重载 ⇒ 加载盘上内容 ⇒ 版本号被写成表里的旧值。**这个流程能收敛**（重载后的内容与版本号一致）。但若顺序出现异常，**逃生舱是终局手段**：refresh 会把所有域 bump 到"高于任何可能值"的水平（实现上可以取 `max(version)+1`），从而强制所有副本做一次权威重载。

**本方案不做：** 工作区的自动修复（自动补软连接、自动回滚发布）、校验和级别的完整性检查（如全卷哈希清单）、跨备份点的差异合并。

---

## 19. 不做的部分与扩展点

### 19.1 线外清单与理由（D14）

| # | 线外事项 | 理由 |
|---|---------|------|
| 1 | **跨节点 Agent 协作（A2A）** | 它是**另一层**：Agent 通信底座解决"跨节点发现、委托、可靠异步协同"，与"让一组实例共享同一份状态"是两个正交的问题。混在一起会让阶段二既要保证单集群语义自洽、又要设计一套通信协议，两件事都做不扎实（详见 19.2） |
| 2 | **多区域 / 多活 / 跨地域部署** | 需要跨区域一致性（数据库跨区复制延迟）、跨区渠道归属、跨区数据治理三套新问题；而这些问题的解法（多数派、冲突合并、区域亲和）会**推翻**本方案"单主 + CAS"的全部简化。真实需求尚未出现（企业第一阶段是"一个数据中心内多副本"） |
| 3 | **多主数据库、分区容忍（CP 取向）** | 本方案的互斥正确性直接建立在"单主数据库上同一行的条件写不会都成功"这一事实上。多主会引入冲突合并，而"两个副本都认为自己持有渠道属主"这种冲突**无法合并**（只有一个正确答案：谁连了长连接）。D2 与 18.2 的论证全部依赖这一点 |
| 4 | **分布式事务（跨多张表的业务级原子性）** | 阶段二的所有原子性需求都是"单语句 CAS"或"单表短事务"（12.2 的代次切换是一个短事务）。引入跨表分布式事务是为了解决一个本方案没有的问题，而代价是两阶段提交/ Saga 的复杂度与悬挂风险 |
| 5 | **引入新中间件（Redis / ZooKeeper / etcd / MQ）** | 见 D2 的三组收敛对照：每多一个中间件就多一套高可用、监控、备份、权限、审计面，而阶段二需要的协调语义（条件写、唯一约束、TTL、版本号、行级可见性）关系库都已具备。什么信号出现才值得引入见 19.3 |
| 6 | **Agent 级调度亲和 / 会话粘性路由** | 本方案的**前提**恰恰是"任意请求落到任意副本都正确"（2.4）。引入粘性会重新引入"某个副本专属于某些会话"的状态，把水平扩展变成"人工分片"，并在副本故障时产生无法迁移的工作。**粘性不是优化，是回退** |
| 7 | **阶段三及以外的容量模型（十万级 Agent、跨组织联邦）** | 这些量级的瓶颈与本方案不同（会是目录规模、索引分片、组织边界的治理问题）。提前为它们设计只会让本方案的实现背上未验证的假设 |

### 19.2 留给阶段三的边界

**口径：跨节点 Agent 协作是另一层的通信底座，本方案不为其预留运行时耦合，只保证单集群语义自洽。**

```text
阶段二（本方案）            阶段三（另一层）
┌───────────────────┐     ┌──────────────────────────────┐
│  单集群语义自洽     │     │  Agent 通信底座                │
│  · 实例对等        │     │  · 跨节点 Agent 发现           │
│  · 恰好一次（5 个   │     │  · 任务委托与结果回传          │
│    粒度）          │     │  · 可靠异步协同（A2A）         │
│  · 渠道属主        │     │  · 跨节点上下文共享            │
│  · 共享工作区      │     │  · 组织边界与信任模型          │
└───────────────────┘     └──────────────────────────────┘
        ▲                                  ▲
        └── 两者通过**运维与协议**组合，而不是通过共享内部状态 ──┘
```

**明确"不预留"的四件事（写清是为了防止实现者自作主张）：**

1. **不做跨集群的 Agent 目录同步**——每个集群有自己的工作区与 `.workspace-id`；阶段三若要跨集群共享 Agent 定义，那是它的同步机制的责任。
2. **不做"任务可迁移"**——本方案的实例对等是"任意副本能做同一件事"，不是"正在跑的工作可以搬到另一个副本"（9.5 已说明停机语义只有"排空或标失败"）。阶段三的委托是"把一件新工作交给别人"，而不是"把半截工作交出去"。
3. **不做跨集群的租约**——本方案的租约作用域就是"一个数据库 + 一个共享卷"。多集群意味着多套租约，它们之间的协调属于阶段三。
4. **不在审计/协议里预留"目标实例"字段**——本周期的审计回答"谁做的"（`instance_id`），不回答"本来想让谁做"（那需要路由概念，属于阶段三）。

### 19.3 未来若要引入 Redis/MQ 的插入位置与判断信号

**纪律：先有信号，再有组件。** 下表把"什么时候值得引入"与"引入后插在哪"写清，作为未来评审的输入；**本周期一律不做**。

| 组件 | 值得引入的信号（可观测、可量化） | 插入位置 | 替换掉什么 | 引入后新增的运维面 |
|------|-------------------------------|---------|-----------|-----------------|
| **缓存（Redis 或进程内多级）** | ①协调语句占数据库读的比例持续 > 30%（`pg_stat_statements` 可测）；②`workspace_versions` 的轮询成为数据库 QPS 的主项；③数据库读副本无法缓解的读热点出现 | 在 `WorkspaceVersionBus` 与 `ClaimStore` 的**读路径**（版本号读、租约状态查询）；**写路径（CAS）绝不缓存** | 版本号轮询的数据库读、租约状态的展示查询 | Redis 的高可用、持久化策略、容量、监控、权限 |
| **只读副本（先于缓存考虑）** | 同上但以"读多写少"为主 | 审计与实例视图的查询 | 主库的读压力 | 复制延迟的可观测性、"读到旧数据"的语义说明 |
| **推送通道（而非消息队列）** | ①业务明确要求传播延迟 < 2 × poll-interval（即当前 1s 轮询不够快）；②提高轮询频率后数据库 CPU 成为瓶颈 | 在 `WorkspaceVersionBus` 的**通知路径**（bump 后推送"某域已变"，各副本收到后仍按版本号比对——**推送只做加速，不做真相源**） | 部分轮询频率（不能取消轮询：推送会丢，版本号才是真相源，见 D8） | 长连接/消息通道的可用性、断线重连、风暴抑制 |
| **消息队列** | ①要求"入站事件不丢"成为硬承诺（当前只能承诺"平台重推 + 去重"）；②要求跨副本的持久化任务队列与顺序保证；③渠道侧事件速率让"同步处理"不再可行 | 在**入站事件到处理之间**（`InboundMessageService` 之前）与**定时触发的执行**（`schedule_claims` 之前作为分发层） | 回执去重表的"唯一防线"角色（退化为兜底）、每副本扫描的空转 | 队列的高可用、顺序与幂等、积压监控、死信处理、消费位点管理 |
| **注册中心（Nacos/etcd）** | ①出现**跨集群联邦**需求（多集群互发现）；②`instances` 表的注册/健康语义不足以表达"跨集群的服务目录" | 在 `InstanceRegistry` 之下作为跨集群目录 | `instances` 表（仅跨集群部分） | 注册中心的高可用、一致性语义、与 K8s Service 的关系梳理 |
| **对象存储** | ①工作区数据量超过共享卷的容量/成本边界；②审计冷数据归档成为成本问题；③需要跨地域读取工作区（而共享卷做不到） | **只在冷归档侧**（审计历史、运行产物归档），**绝不在热路径**（对象存储没有 rename 原子语义，见 D2 收敛对照二） | 审计表的保留期策略、产物目录的清理 | 对象存储的权限、生命周期策略、读取延迟与一致性说明 |

> **判断速记（写进未来的评审准则）：** ①**写路径（CAS）永远不搬到别的组件**——它是正确性的根；②**真相源永远只有一个**（版本号在表里、租约在表里），任何缓存/推送都只是加速，必须能在被清空后不影响正确性；③**每一层的引入都必须先有可观测的信号**，而不是"为了架构好看"。

---

## 20. 模块结构增量

### 20.1 模块改动表

阶段一有 14 个模块。阶段二的改动遵循三条纪律：**契约放 core、实现在下游、禁止循环依赖**；**模块演进必须在特性 plan 里声明理由并同步阶段一技术方案第 10 章与 `CLAUDE.md` 的模块表**（`CLAUDE.md` 明确要求）。

| 模块 | 改动 | 内容 | 类型 |
|------|------|------|------|
| `oryxos-core` | 修改 | 集群协调契约与值对象：`ClusterCoordinator`、`LeaseStore`、`ClaimStore`、`InstanceRegistry`、`ChannelOwnershipManager`、`WorkspaceVersionBus`、`WorkspaceStorage`/`WorkspaceStorageProvider`、`KnowledgeGenerationStore`、`AdminWriteReservation`；`SpanRecorder` 契约（已在 core，新增 `instance_id` 属性）；`AgentScheduler` 的"算 fireTime → 认领 → 执行"路径增量；`AgentService` 的"先认领会话轮次"增量 | 修改 |
| `oryxos-storage` | 修改 | 9 张新表的 JPA 实体与 repository、Flyway 双轨迁移脚本（V7+）、审计表新列的映射、PostgreSQL 档的确认（沿用阶段一 url 自动识别）、`LeaseStore`/`ClaimStore`/`InstanceRegistry`/`WorkspaceVersionBus`/`KnowledgeGenerationStore` 的数据库实现 | 修改 |
| **新增 `oryxos-cluster`（建议）** | 新增 | 协调实现与线程：`LeaseRenewer`（专用调度线程）、`WorkspaceVersionPoller`、`InstanceHeartbeat`、`ChannelOwnershipCoordinator`、启动校验器（误配矩阵）、`AdminWriteReservationService` | 新增 |
| `oryxos-web` | 修改 | `GET /api/v1/instances`；`GET /api/v1/health/ready` 与 `/health/live`（保留既有 `/health`）；`POST /api/v1/workspace/refresh`；知识重建端点（认领冲突返回 409 + `Retry-After`）；现有端点的"会话轮次认领"接入 | 修改 |
| `oryxos-knowledge` | 修改 | 代次化：chunk 写入带 `generation_id`、检索恒读 committed 代次、重建流程（认领 + 阶段间续约检查 + 原子切换）、导入路径与重建共用认领 | 修改 |
| `oryxos-memory` | 修改 | 集群档校验（markdown 后端拒启动）；共享后端（`sqlite` 走共享库）的确认与文档 | 修改 |
| `oryxos-channel-*`（飞书/企微/钉钉） | 修改 | 适配器受属主资格约束：仅属主启动长连接；失去租约后停止自动重连（8.6）；`closeQuietly` 限时（对齐阶段一飞书教训） | 修改 |
| `oryxos-tool` | 修改 | 文件类 Tool 与 Shell 必须经 `nativePath`（10.3）；`AtomicFiles` 的使用；保留路径拒绝 | 修改 |
| `oryxos-boot` | 修改 | 档位条件装配（单机档不装配协调 Bean，20.3）；启动校验的执行顺序；停机链路顺序（先释放属主，8.4） | 修改 |
| `oryxos-cli` | 修改 | 集群相关诊断子命令（可选：打印档位、实例 ID、版本号、属主） | 修改 |
| `oryxos-persona` / `oryxos-provider` | 基本不动 | Persona 走工作区（版本号总线覆盖 `personas` 域）；Provider 显式映射与 fallback 不变（原则三） | 不动 |
| `oryxos-channel-cli` | 基本不动 | CLI 渠道在集群档下语义为"本副本的本地交互"，不涉及属主 | 不动 |
| `charts/oryxos/` | 新增（非 Maven 模块） | 官方 Helm Chart（16.1） | 新增 |

> **为什么建议新增 `oryxos-cluster` 而不是把协调塞进 `oryxos-core` 或 `oryxos-storage`：** ①**契约与实现分离**（core 放契约、`oryxos-cluster` 放协调线程与装配，`oryxos-storage` 放 SQL 实现）；②**档位隔离更自然**（整个 `oryxos-cluster` 的 Bean 可以被一个 `@ConditionalOnProperty(oryxos.cluster.enabled)` 包住，单机档完全不加载）；③**符合"模块跟随能力域"的演进纪律**，且新增理由可以一句话说清："集群协调是一组有自己线程模型与生命周期的实现，不属于任何现有能力域"。**代价：多一个模块的构建与依赖管理成本**；收益是可测试性与档位隔离的清晰边界。**这一条是建议（需在特性 plan 里最终确认），不是已定的模块表。**

### 20.2 关键契约清单

| 契约 | 职责（一句话） |
|------|--------------|
| `ClusterCoordinator` | 集群档的总装配与生命周期入口：启动校验、启动/停止续约与轮询线程、暴露当前档位与实例状态 |
| `LeaseStore` | 通用租约原语：acquire（插入式/抢占式）/ renew / release / 查询；所有租约表共用同一套语义（4.3） |
| `ClaimStore` | 幂等键认领原语：`claim(key)` 返回"是否获得"；用于事件回执、定时触发、知识重建（4.3 形态一） |
| `InstanceRegistry` | 实例注册、心跳写入、活跃实例列表与过期判定（第 9 章） |
| `ChannelOwnershipManager` | 渠道属主的取得、续约、释放、接管与"本实例是否为属主"的判定；约束渠道适配器的启动与重连（第 8 章） |
| `WorkspaceVersionBus` | 版本号总线的读（六域比对）、bump（域 +1）、以及"未确认 bump"的自愈登记（第 11 章） |
| `WorkspaceStorageProvider` | 存储插件标识与创建（`id()` = `local` / `shared`） |
| `WorkspaceStorage` | 逻辑根、逻辑路径解析、本机执行视图（`root` / `resolve` / `nativePath`）；一切工作区访问的唯一入口（042 约束） |
| `KnowledgeGenerationStore` | 代次的创建、置 committed/retired、读当前 committed 代次、按 `keep-generations` 回收（第 12 章） |
| `AdminWriteReservation` | 管理写互斥预约的申请/查询/显式释放（不自动过期）；恢复流程的闸门（10.8） |
| `SpanRecorder` | trace span 记录契约（阶段一已有，在 core）；集群档新增 `instance_id` 属性，NOOP 默认零开销（14.5） |
| `AtomicFiles` | 同目录临时文件 + rename 的原子落盘工具（10.4）；多文件发布协议的实现基础（10.5） |

> **纪律：上表全部是 core 里的接口/值对象（`AtomicFiles` 可为 core 的工具类），下游模块实现。任何"把契约放到实现模块里"的写法都会造成依赖方向倒置与循环依赖。**

### 20.3 档位隔离与自动装配

```text
oryxos-boot 的条件装配：

  ┌─ 始终装配（两档共有）───────────────────────────────────────┐
  │ 业务 Bean：ReActLoop、ProviderService、MemoryService、        │
  │ ToolRegistry、Channel 适配器、"连接与编排"、Web 层、AgentScheduler│
  │ （注意：AgentScheduler 两档都有；差异在"执行前是否认领"）        │
  └──────────────────────────────────────────────────────────────┘

  ┌─ @ConditionalOnProperty(oryxos.cluster.enabled=false) ───────┐
  │  单机档专属：进程内事件去重 Map、WatchService watcher、          │
  │  进程内 ReentrantLock 会话串行、直接执行（不认领）              │
  │  ⇒ 不装配任何协调 Bean（LeaseStore/ClaimStore/… 都不存在）       │
  └──────────────────────────────────────────────────────────────┘

  ┌─ @ConditionalOnProperty(oryxos.cluster.enabled=true) ────────┐
  │  集群档专属：ClusterCoordinator、LeaseRenewer、VersionPoller、  │
  │  InstanceHeartbeat、ChannelOwnershipManager、启动校验器        │
  │  + 数据库实现（LeaseStore/ClaimStore/… ← oryxos-storage）      │
  └──────────────────────────────────────────────────────────────┘
```

**三条纪律：**

1. **单机档下协调契约的实现类必须不存在于容器里**（不是"存在但不用"）。这样"业务代码不小心调了协调 API"会在单机档启动时或测试中立即暴露，而不是变成"看起来能跑但语义不对"。
2. **业务代码依赖契约而非实现**（构造注入接口），且**两档下调用点的语义必须一致**：单机档的"认领永远成功"（因为无人竞争）与集群档的"认领可能失败"必须在调用点用同一段代码表达——否则会出现"集群档才走的分支没人测"。
3. **测试必须断言档位隔离**（20.4 的参数化测试集）：单机档启动后容器里没有 `LeaseStore` 的 Bean、没有轮询线程在跑（线程名可断言）；集群档必须有。

### 20.4 测试策略

**参数化测试集（同一个测试主体，跑三组参数）：**

```text
维度 1：档位         单机档 / 集群档
维度 2：存储矩阵     本机卷 / NFS / 云盘 RWX（至少覆盖"本机卷"与"一个真实共享卷"）
维度 3：内存后端     markdown / sqlite（知识库：local + 共享 chunk 存储）

⇒ 组合成一套参数化测试；每个业务性测试（会话、定时、渠道、工作区、知识）
  都在这套参数下跑同一段断言。
```

**并发正确性测试：必须多进程/多副本集成测试，不能是单元测试。**

| 测试 | 为什么单元测试无效 | 怎么做 |
|------|------------------|--------|
| 会话轮次恰好一次 | `ReentrantLock` 或 mock 掉的锁在单进程内永远"正确"；竞态只在两个真实的独立数据库连接之间存在 | 起 2–4 个真实进程（或至少真实连接池），同时对同一 `session_id` 发 N 条消息；断言：`turn_seq` 无重复、`task_executions`/审计计数符合预期、回复条数 == 收到处理的轮次数 |
| 定时触发恰好一次 | 单元测试里的时间可以 mock，而缺陷恰恰在"各副本算出的 `fire_time` 是否相同" | 多进程 + 真实 cron 定义 + 人为把某一进程的 tick 相位错开/阻塞；断言每个 `fire_time` 恰好一行 `task_executions` |
| 回执去重 | 单进程 Map 本来就能去重 | 多进程注入同一 `(channel, event_id)`；断言只有一个进程处理；再注入不同渠道的同 id 事件，断言两者都被处理（跨渠道不冲突） |
| 渠道属主唯一 | 单进程只有一条连接 | 多进程同时启动；断言任一时刻只有一个进程持有属主（查询 `channel_ownerships`）；kill 属主进程，断言 ≤ TTL + tick 接管 |
| 崩溃接管 | 无法在单元测试里"杀死"一半的共享状态 | 多进程，处理中途 `SIGKILL` 持有者；断言：接管者把该轮标 `failed`、写审计、不重放（同一 `turn_seq` 不产生第二条成功记录）；用户重发后恢复 |
| 版本号传播 | 单进程里"改了立刻读得到"，测不到传播机制 | 一个进程改文件 + bump；其他进程记录"何时看到新内容"；断言所有副本的最大延迟 ≤ 2 × poll-interval + 重载耗时（Demo D 的度量） |
| 双属主窗口 | 窗口由"假死"造成，单进程无假死 | 用故障注入：让持有者的续约线程暂停（断点/注入延迟）而不停止进程；断言新持有者接管后，旧持有者的后续数据库写入（续约/释放）影响 0 行 |
| 误配拒启动 | 需要真的用错的配置启动 | 参数化：3.2 的**启动误配矩阵全 7 类**各起一次进程，断言非零退出 + 错误信息包含恢复指引的关键词；另加 1 条**运行期 fail-closed** 用例（数据库不可达时拒服务、不降级单机）（Demo M0） |

**工具与基础设施建议：** ①集成测试用真实的 PostgreSQL（Testcontainers 或 CI 内的服务容器），**不用 H2 之类替代品**（认证与并发语义不同，会掩盖真实缺陷）；②共享卷测试在 CI 里用一个真实导出（NFS 容器或 kind 的 NFS provisioner），并**明确标注"CI 的共享卷不保证与生产同语义"**；③测试必须**可重复执行**（每次跑独立 schema/独立工作区），并能**注入故障**（kill 进程、暂停线程、断数据库连接）；④把 17.4 的压测脚本纳入 CI 的 nightly（而不是每次 PR），因为它需要多进程与较长时间。

**本方案不做：** 形式化验证、模型检验（TLA+ 等）、跨数据中心的混沌工程；持续运行的混沌实验（本周期用一次性故障注入覆盖关键场景）。

# 第二部分：验证与约束

## 21. 关键流程端到端

五个场景对齐 README 第 7 章的 Demo A–E（功能验收标准见 [DemandAnalysis.md](DemandAnalysis.md) 第 13 章）。每个场景给出时序、断言清单、失败判定。**断言必须是可执行的**（能写成脚本/测试的），不是"看起来工作正常"。

### 21.1 Demo A：滚动升级零中断

**场景：** 3 副本集群在真实流量下做一次滚动升级（`maxUnavailable: 0` / `maxSurge: 1`，grace 40s），升级期间 IM 消息持续进入、定时任务持续触发。

```text
      流量（IM 消息 + 定时触发）持续不断
                  │
  ┌───────────────┼───────────────┐
  ▼               ▼               ▼
 Pod1(v1,旧)   Pod2(v1,旧)    Pod3(v1→v2,新,已就绪)
  │                                │
  │ 轮换：先起 Pod4(v2) 就绪，再终止 Pod1
  ▼
 Pod1 收到 SIGTERM：
   ① status=draining（实例视图可见）
   ② readiness 失败 → 端点摘除（preStop 的 5s 用于让摘流生效）
   ③ 停止认领新工作（轮次/定时）
   ④ 排空在途（≤ 35s，仍在续约）
   ⑤ 释放渠道属主 → 其他 Pod 秒级接管（不等 30s TTL）
   ⑥ 限时关闭长连接 → 退出
  ▼
 Pod4(v2) 承担流量；重复直到全部替换为 v2
```

**断言清单：**

| # | 断言 | 判定方式 |
|---|------|---------|
| A1 | 升级期间每条 IM 事件**恰好一次**回复 | 按 `(channel, event_id)` 计数：`event_receipts` 一行、对应会话轮次成功一条；对比注入的事件总数 = 成功回复数 + 显式失败数（无"既没成功也没失败"） |
| A2 | 每个 `fire_time` 恰好一行 `task_executions` | `SELECT schedule_id, fire_time, COUNT(*) … HAVING COUNT(*) > 1` 返回空 |
| A3 | 单副本不可用窗口 = 在途请求排空时间 | 监控：Pod1 从 SIGTERM 到退出的时长 ≤ 40s；期间端到端错误率（除主动排空造成的重试）为 0 |
| A4 | 渠道属主切换是秒级（非 TTL 级） | 记录 Pod1 释放属主时刻与其他 Pod 接管时刻，差值 ≤ 扫描 tick（秒级），**不是** 30s |
| A5 | 无残留 | 升级后 `instances` 中只有 v2 的实例且 `status=ready`；无 `draining` 幽灵 |
| A6 | 审计连续 | 升级窗口内的 `tool_invocations`/`llm_calls` 无缺口；`instance_id` 覆盖新旧副本 |

**失败判定（任一即失败）：** ①出现两条相同 `event_id` 的回复；②某个 `fire_time` 有两行 `task_executions`；③Pod 被 SIGKILL（退出码 137）或退出时间 > grace；④属主接管耗时 ≈ 30s（说明主动释放没生效，退化成了 TTL 接管）；⑤升级后仍有 `draining` 状态的实例。

**升级窗口内消息的去向（逐段对应 16.6 的表）：** 摘流完成后新请求不再到达旧 Pod；旧 Pod 停止认领后由新 Pod 承担；渠道切换窗口（秒级）内平台可能短暂断开，重推由回执去重兜底；定时触发的唯一性由 `(schedule_id, fire_time)` 主键保证，与在哪个 Pod 上执行无关。

### 21.2 Demo B：副本猝死接管

**场景：** 3 副本，Pod2 在正在处理某会话轮次时被 `kill -9`（无任何清理机会）。

```text
 Pod2 持有 s1 第 5 轮（state=running, expires_at=T+60s）
   │
   ✗ SIGKILL（无释放、无审计收尾）
   │
   ├─ instances：Pod2 心跳停止 → 其他副本读取时判定为 stale（不主动动作）
   ├─ 渠道属主：Pod2 若持有 → 租约到期（≤30s）→ 其他 Pod CAS 接管 → 建连
   └─ 会话轮次：租约到期（≤60s）→ 其他 Pod CAS 接管
        ├─ ① 该轮置 failed（D6）
        ├─ ② 写审计（原 owner、原 traceId、标失败原因）
        ├─ ③ 释放（删除租约行）
        └─ ④ 不重放
   │
 用户重发同一条内容
   → 任意副本认领成功 → 新 turn_seq → 正常回复
```

**断言清单：**

| # | 断言 | 判定方式 |
|---|------|---------|
| B1 | 崩溃轮次被标 `failed`，**不被重放** | 该 `turn_seq` 在审计中无第二条成功记录；`session_turn_leases` 曾有 `state='failed'` 与 `last_error`（或由审计中的接管记录证明） |
| B2 | 渠道属主在 TTL 内被接管 | 从 SIGKILL 到新属主 `acquired_at` ≤ 30s + 扫描 tick |
| B3 | 无重复回复 | 崩溃轮次前后的回复计数与用户消息数匹配（崩溃轮次没有回复，重发后有一条） |
| B4 | 无消息丢失 | 崩溃后注入的事件全部有 `event_receipts` 行且最终有回复（除显式失败的以外） |
| B5 | 崩溃前已发生的审计保留 | `tool_invocations`/`llm_calls` 中属于原 `traceId` 的行仍在（能回答"跑到哪一步"） |
| B6 | 运维视图可解释 | `GET /api/v1/instances` 中 Pod2 为 `stale`；属主字段已转移到其他实例 |

**失败判定：** ①崩溃轮次被重放（同一 `turn_seq` 出现两次执行）；②属主接管退化为需要人工；③出现消息"既无回复也无失败记录"；④`failed` 的轮次没有 `last_error` 或审计线索（无法归因）。

### 21.3 Demo C：定时任务跨副本恰好一次

**场景：** 同一 cron 定义（如每分钟触发一次）在 3 副本上同时存在；人为阻塞一个副本的调度扫描（模拟卡顿），观察唯一性。

```text
 定义：schedule_id = S1，cron = * * * * *（每分钟）
   每个副本各自：tick → CronTrigger 算出 fire_time（同一值！）
        │
        ├─ Pod1: INSERT (S1, 10:00:00) → 1 行 ⇒ 执行 ⇒ 写 task_executions ⇒ succeeded
        ├─ Pod2: INSERT (S1, 10:00:00) → 0 行 ⇒ 本 tick 结束（什么都不做）
        └─ Pod3: 被阻塞（tick 卡住 90s）⇒ 恢复后算出**同一个** 10:00:00
                 ⇒ INSERT → 0 行 ⇒ 不重复
  10:01:00 下一轮：fixtime 不同 ⇒ 重新竞争 ⇒ 仍恰好一个执行
```

**断言清单：**

| # | 断言 | 判定方式 |
|---|------|---------|
| C1 | 每个 `fire_time` 恰好一行 `task_executions` | 按 `(schedule_id, fire_time)` 分组计数，无 > 1 |
| C2 | 人为阻塞一个副本不产生重复 | 阻塞 Pod3 的扫描线程 90s，跨过至少一个 `fire_time`；C1 仍成立 |
| C3 | `fire_time` 是理论时刻而非墙钟 | `task_executions.fire_time` 的秒位恒为 `00`，且不同副本写入的值相同（对同一 fire_time） |
| C4 | 唯一的执行者带 `instance_id` | 每行 `task_executions` 的 `instance_id` 是三个 Pod 之一，且同 `fire_time` 只有一行 |
| C5 | 阻塞造成的错过按 misfire 策略处理 | Pod3 恢复后不补跑历史 `fire_time`（默认策略）；若有 `skipped` 记录其 `state` 为 `skipped` |
| C6 | 重启不改变基准 | 重启一个副本后，它算出的下一个 `fire_time` 与其他副本一致（基准从持久状态重建，6.1 第 2 条） |

**失败判定：** ①任一 `fire_time` 出现两行 `task_executions`；②`fire_time` 出现非整分钟值（说明用了墙钟）；③重启后副本算出的 `fire_time` 与其他副本不同（说明基准没持久化）；④Pod3 恢复后补跑（违反默认 misfire 策略）。

### 21.4 Demo D：工作区变更传播 + 知识重建认领

**场景：** ①在 Pod1 上改一个 Agent 的 `AGENT.md`（或绑定一个 Skill），测量其他副本的生效延迟；②三个副本同时触发同一知识库重建。

```text
【D1 变更传播】
 Pod1：写文件（AtomicFiles）→ bump agents 域 (v→v+1)
 Pod2/Pod3：各自每 1s 读一次 workspace_versions
        ├─ t0+1s：读到 v+1 ⇒ 重载 ⇒ 新内容可见
        └─ 最坏：t0+2s（恰好在 tick 后发生）
 断言：所有副本的"新内容可见时刻 - t0"的最大值 ≤ 3s（P99 口径）

【D2 知识重建认领】
 Pod1/Pod2/Pod3 同时收到重建请求
   ├─ Pod1: INSERT knowledge_build_claims → 1 行 ⇒ 获得 ⇒ 开建新代次（building）
   ├─ Pod2: 0 行 ⇒ 返回 409（带持有者与开始时间、Retry-After）⇒ 调用方可重试
   └─ Pod3: 0 行 ⇒ 同上
 重建期间：所有副本检索恒读**旧 committed 代次**⇒ 检索不断服
 切换：一个短事务把旧 committed → retired、新 building → committed
 回收：按 keep-generations=2 保留（当前 + 上一代），其余批量删除
```

**断言清单：**

| # | 断言 | 判定方式 |
|---|------|---------|
| D1 | 变更 P99 ≤ 3s 生效（所有副本的最大延迟） | 每个副本探测"新内容可见"的时刻，取最大值；重复 20 次取 P99 |
| D2 | 重建只执行一次 | `knowledge_generations` 中该 `(backend, scope)` 只有一个 `committed`；`building` 代次在成功路径上恰好一个、失败路径上被清理 |
| D3 | 冲突方拿到 409 且不静默失败 | Pod2/Pod3 收到 409，响应体含持有者与可重试提示；**不是** 500 或静默 200 |
| D4 | 冲突方"排队不丢" | 409 之后重试（或在重建完成后重试）成功获得认领或明确得知已完成 |
| D5 | 检索全程不断服 | 重建期间持续检索：无错误、无"代次不存在"；结果在切换前后分别来自旧/新代次（可区分内容） |
| D6 | 代次切换原子 | 不存在"某一刻有两个 committed"（部分唯一索引在数据库层保证；测试查 `COUNT(*) WHERE state='committed'` 恒为 1） |
| D7 | 旧代次按 keep-generations 回收 | 切换并回收后，`retired` 代次数 = keep-generations − 1（即 1） |
| D8 | 导入与重建互斥 | 重建期间发起导入 ⇒ 409；重建完成后导入成功；文档源（盘上的文件）**始终存在**（不丢） |

**失败判定：** ①有副本的延迟 > 3s 且不收敛（版本号没推进或重载一直失败）；②出现两个 `committed`（约束实现错误）；③冲突方拿到 500 或静默成功但没做事；④重建期间检索报错；⑤导入被拒后发现文档源也丢了（违反"排队不丢"的两层含义）。

### 21.5 Demo E：运维逃生舱与实例视图

**场景：** ①运维直接用编辑器改盘（绕过版本号总线），用逃生舱强制重载；②用 `GET /api/v1/instances` 与审计核对实例口径。

```text
【E1 逃生舱】
 运维：直接编辑共享卷上的 .oryxos/agents/ops/AGENT.md（不经过管理写路径 ⇒ 无 bump）
 现象：其他副本不重载（版本号没变）—— 这是**设计的正常行为**
 运维：POST /api/v1/workspace/refresh {domains:["agents"], reason:"手工改盘"}
   → 服务端对 agents 域 bump（v→v+1，updated_by 记录这是一次人工刷新）
   → 各副本轮询到 ⇒ 重载 ⇒ 新内容生效（≤ 2 × poll-interval）
 断言：重载后内容正确；重复调用幂等（不产生错误，只是再重载一次）

【E2 实例视图与审计口径】
 GET /api/v1/instances → 3 个实例，status=ready，各自列出持有的渠道与租约计数
 对该时段做一次业务操作 → 查审计：
    tool_invocations.instance_id / llm_calls.instance_id ∈ 上述 3 个 instance_id
    且与 GET /instances 的 instanceId 集合一致（无"幽灵实例"）
```

**断言清单：**

| # | 断言 | 判定方式 |
|---|------|---------|
| E1 | 直接改盘后不自动生效（预期行为可复现） | 改盘后等待 > 2 × poll-interval，其他副本仍为旧内容 |
| E2 | 逃生舱生效 | refresh 后所有副本 ≤ 2 × poll-interval 内看到新内容 |
| E3 | 逃生舱幂等 | 连续调用 3 次不报错；版本号单调递增；重载次数增加但无副作用 |
| E4 | 逃生舱可审计 | `workspace_versions.updated_by`/`updated_at` 与日志能对应到这次调用（含 reason） |
| E5 | 实例视图完整 | `GET /api/v1/instances` 的实例集合 = 真实运行的副本集合（无遗漏、无幽灵 `stale` 残留） |
| E6 | 视图与审计口径一致 | 审计里的 `instance_id` 全部出现在实例视图中（或可解释为"迁移前的历史行"= NULL） |
| E7 | 实例视图的租约/属主字段与协调表一致 | 视图里某实例的 channels 与该实例在 `channel_ownerships` 中的持有记录一致 |

**失败判定：** ①逃生舱不生效（bump 失败且不报错）；②refresh 报错或产生重复副作用；③实例视图出现"心跳在跳但审计里没有它的任何行"（或反之）——说明实例标识口径不一致；④审计中出现实例视图中不存在的 `instance_id` 且无法解释。

---

## 22. 实施节奏

阶段二按 **M0–M5** 六个里程碑推进（对齐 README 第 7 章）。每个里程碑的目标 / 产出物 / 验收 / 主要风险 / 对应用户故事如下。

### 22.1 M0：共享状态基座

| 项 | 内容 |
|----|------|
| **目标** | 集群档能起来：档位开关、实例标识、误配拒启动、实例心跳、PostgreSQL 档 |
| **产出物** | 两轨迁移 V7（9 张表 + 审计新列）；`oryxos-cluster` 骨架（装配 + 心跳）；`GET /api/v1/instances`；启动校验器（11 项矩阵）；`/health/ready` 与 `/health/live` |
| **验收** | 拉起 2 副本，实例视图看到两个实例与心跳；启动误配矩阵 7 类各自拒启动并给出恢复指引，运行期数据库不可达时拒服务（Demo E2 的前半、Demo M0） |
| **主要风险** | 误配校验不全会让错误配置跑起来（对策：矩阵逐项写成参数化测试，20.4）；两轨迁移的 sqlite 轨写法差异（对策：sqlite 轨只求结构等价并加迁移测试） |
| **对应用户故事** | US-1（集群开关与拒启动）、US-2（实例注册与视图） |

### 22.2 M1：恰好一次三件套

| 项 | 内容 |
|----|------|
| **目标** | 会话轮次租约、调度到点认领、事件回执去重三件事在集群档生效 |
| **产出物** | `LeaseStore`/`ClaimStore` 通用原语与续约线程；`AgentService` 的认领接入（含排队语义）；`AgentScheduler` 的 fireTime 认领路径；`event_receipts` 替换进程内 Map；相关的审计增量（`instance_id`、`fire_time`） |
| **验收** | 同一会话并发打到 3 副本，回复不重不乱；同一 cron 在 3 副本上每个 `fire_time` 只执行一次（Demo A1/A2、Demo C） |
| **主要风险** | 排队实现不当会占满连接池（对策：等待期间不持连接，17.1）；`fire_time` 基准未持久化导致重启后不一致（对策：C6 断言） |
| **对应用户故事** | US-3（会话轮次）、US-4（定时触发）、US-5（事件去重） |

### 22.3 M2：渠道属主与停机链路

| 项 | 内容 |
|----|------|
| **目标** | 渠道长连接单属主、永不互踢、故障接管、停机先释放 |
| **产出物** | `ChannelOwnershipManager`；三个渠道适配器受属主约束（含"失去租约停止重连"）；`ChannelAdminService.stopAll` 的顺序（先释放属主）；启动抖动抑制 |
| **验收** | 同一渠道多副本启动只有一个是属主；kill 属主后 TTL 内接管且不互踢、不双回复（Demo B2/B3） |
| **主要风险** | 适配器的自动重连绕过属主资格（对策：8.6 的显式约束 + 故障注入测试）；停机链路超时挤爆 grace（对策：每步限时，8.4） |
| **对应用户故事** | US-6（渠道属主与停机） |

### 22.4 M3：共享工作区

| 项 | 内容 |
|----|------|
| **目标** | 工作区放共享卷：存储插件、原子落盘、版本号总线、本机执行视图、管理写预约、逃生舱 |
| **产出物** | `WorkspaceStorage`/`Provider`（core 契约 + local/shared 实现）；`AtomicFiles`；多文件发布协议（`.staging` + 清单 + 写序）；`WorkspaceVersionBus` + 轮询重载；`nativePath`；`admin_write_reservations`；`POST /api/v1/workspace/refresh` |
| **验收** | 一个副本改 `AGENT.md` / 绑 Skill，其余副本 P99 ≤ 3s 生效；直接改盘后逃生舱强制重载（Demo D1、Demo E1–E4） |
| **主要风险** | 业务代码残留裸文件 API（对策：042 约束的代码审查 + 测试断言"无 Path.of"）；共享卷不支持 rename（对策：部署前的能力验证，10.1/16.4） |
| **对应用户故事** | US-7（共享工作区）、US-8（变更传播与逃生舱） |

### 22.5 M4：知识库代次化

| 项 | 内容 |
|----|------|
| **目标** | 知识索引在集群档的代次化与重建认领 |
| **产出物** | `knowledge_generations` 代次模型 + `knowledge_build_claims` 认领；chunk 存储带代次分区；重建流程（阶段间续约检查）；导入与重建共用认领（409 + Retry-After）；回收作业（keep-generations） |
| **验收** | 多副本同时触发重建只有一个执行（其余 409 排队），检索全程读已提交代次不断服（Demo D2–D8） |
| **主要风险** | 重建中途崩溃留下半成品代次（对策：12.2 的"删除失败 building 代次、从零重建"）；chunk 存储容量与清理（对策：12.6 的量级估算 + 指标） |
| **对应用户故事** | US-9（知识代次与重建） |

### 22.6 M5：交付与验证

| 项 | 内容 |
|----|------|
| **目标** | 官方 Helm Chart、探针、滚动升级、跨副本可观测、HA 演练 |
| **产出物** | `charts/oryxos/`（含 values.schema、templates、NOTES、tests）；`make helm-lint` + CI 的 helm job（kind 冒烟）；OTel 可选导出 + `instance_id` 贯穿；指标清单落地；压测脚本与报告 |
| **验收** | 滚动升级期间消息零丢失（Demo A）；压测报告（加速比、传播延迟、接管时间，对齐 README 第 6 章数字）；Demo B/D/E 全绿 |
| **主要风险** | 探针写错造成升级卡死或全副本重启（对策：16.5 的断言进 CI）；4 副本加速比不达标（对策：17.5 的审计写入路径优化预案） |
| **对应用户故事** | US-10（容器交付）、US-11（可观测）、US-12（压测与 HA 演练） |

### 22.7 为什么这个顺序不能换

```text
M0 基座 ──► M1 恰好一次 ──► M2 属主 ──► M3 文件面 ──► M4 知识代次 ──► M5 交付验证
 │            │               │            │              │              │
 │            │               │            │              │              └ 需要前面全部
 │            │               │            │              └ 依赖 M0 的代次表/M3 的工作区
 │            │               │            └ 依赖 M0（版本号表）+ M1（认领原语复用）
 │            │               └ 依赖 M0（属主表/心跳）；与 M1 共享租约原语
 │            └ 依赖 M0 的表、启动校验、实例标识（认领需要 instance_id）
 └ 一切的前提：没有共享状态层，后面任何一件事都无处落地
```

| 依赖关系 | 如果换了顺序会怎样 |
|---------|------------------|
| M0 必须第一 | 认领原语的表、实例标识、启动校验是所有后续能力的地基；先做 M1 会导致"没有 instance_id 可写、没有表可认领" |
| M1 先于 M2 | 属主租约与轮次租约**共用同一套 CAS + TTL + 续约实现**。M1 把原语做扎实后，M2 只是换个键与业务语义；反过来先做 M2 会让原语在"渠道"这个更复杂的场景里调试 |
| M2 先于 M3 | 停机链路（先释放属主）属于 M2；M3 的"多文件发布 + 管理写预约"需要停机语义已经稳定（恢复流程的前置条件就是"停全部写者"，10.8） |
| M3 先于 M4 | 知识库的文档源在工作区（共用存储抽象与原子落盘），代次存储需要共享数据库；跳过 M3 会让知识库在共享卷上无立足之地 |
| M5 最后 | chart 的探针与滚动升级参数依赖前四个里程碑的最终运行语义（尤其停机链路与就绪判据）；压测需要全部能力就绪才有意义 |

> **一条纪律：每个里程碑结束都必须能演示，且演示不依赖后续里程碑的代码。** 这是"分阶段克制"（宪法）在实施节奏上的体现——避免"全部做完才第一次跑起来"。

---

## 23. 与项目宪章的约束对齐

项目宪章（`CLAUDE.md`）的八条不可违背原则与 042 工作区存储开发约束，在集群档下依然成立。下表逐条说明：阶段一怎么落地、集群档新增什么风险、本方案怎么保证、违反会怎样。

| 宪法条款 | 阶段一的落地方式 | 集群档的额外风险 | 本方案如何保证 | 若违反会怎样 |
|---------|----------------|----------------|--------------|-------------|
| **原则一：自实现 ReAct Loop**（不得用 Spring AI Agent 抽象） | `ReActLoop` 手写（数十行），Tool 调度由 `ToolExecutor` 控制 | 集群档在循环**外面**加了认领与排队；若实现者图省事把"认领"塞进 `ChatClient` 的调用链（如用 Spring AI 的 advisor 机制做前置拦截），就会把循环控制权交出去 | 认领发生在 `AgentService.process` 的**入口**（循环之前）；`ReActLoop` 本身一行不改；续约检查点插在每轮迭代开始（4.5），仍在自己的循环里 | Tool 被调两次、循环行为不可控；更隐蔽的是"认领与执行不在同一个语义单元里"，故障时无法解释"谁该负责标失败" |
| **原则二：Spring AI 只用两件事**（协议转换 + `@Tool` schema；禁用自动 tool 执行） | 只用 `ChatModel.call(...)`；禁用自动执行 | 集群档新增的 Provider fallback 路径与重试若误用 Spring AI 的重试/advisor，会把 tool 执行也带上 | 集群档**不改** Provider 调用路径（fallback 已有：每次尝试各写一条 `llm_calls`）；认领重试只在协调层（4.6），与 LLM 调用重试完全分离、不共用任何抽象 | tool 被调两次（同一轮内），且审计里出现"一次 tool 调用两条记录"或"绕过 ToolExecutor 的调用" |
| **原则三：Provider 显式映射** | `Map<String, ChatModel>` 显式映射，`provider.name` 引用 | 多副本下若有人"为了容灾"改成按容器扫描 `ChatModel`，会退回到"Bean 类型相同无法区分"的老问题；集群档的 Provider fallback 会让"到底用了哪个 Provider"更需要审计 | 映射机制不变；每次尝试（主/备）各写一条 `llm_calls`，带 `instance_id` 与 `trace_id`，使"哪个副本、用了哪个 Provider"可查（14.2） | 多 Provider 路由错乱；集群档下表现为"同一会话在不同副本上用了不同模型"，输出风格漂移且难以归因 |
| **原则四：一个目录 = 一个 Agent；Skill 走相对软连接 + 渐进披露** | `.oryxos/agents/<name>/` + `skills/` 相对软连接；每轮只注入 Skill 的 name/description/本地路径；正文经 `read_file` 现取 | ①软连接在共享卷上可能 dangling（跨副本可见性）或 escaped；②`nativePath` 与逻辑路径混用会让"本地绝对读取路径"变成某个副本上的路径——一旦落到会话/审计里就不可迁移；③多人同时改一个 Agent 会互相覆盖 | Skill 元数据注入的路径必须是**逻辑路径派生的一致形式**，本机路径只在执行瞬间构造（10.3）；五类检测（name-mismatch/dangling/escaped/invalid-target/stale-reference）在共享卷下每次现检（10.11）；公共 Skill 被引用时删除默认拒绝（沿用阶段一） | ①模型读到别的副本上不存在的路径 ⇒ 大量"文件不存在"，Skill 机制形同虚设；②审计里的路径不可迁移 ⇒ 换部署后无法复现；③软连接逃逸 ⇒ 沙箱白名单被绕过（原则六同时被破） |
| **原则五：审计表 day one 落库** | `tool_invocations`/`llm_calls` 核心阶段即写入 | 集群档审计行**必须带 `instance_id`**；若某条写入路径忘了带（如新增的协调操作自己写审计），会出现"无法归因到副本"的洞；跨副本链路追溯依赖 traceId + instance_id 同源 | D13：`instance_id` 与 `traceId` 同层，由统一的审计写入上下文提供，不由各调用点自己拼（14.2）；崩溃轮次、接管动作都写审计（5.5）；**审计不做采样**（14.5） | 出现无法归因的重复执行/失败 ⇒ 排障只能靠猜；"可审计"这个差异化能力在集群档打折 |
| **原则六：不用 SecurityManager；软连接/真实路径校验**（`toRealPath()`、新建路径查最近存在父目录、只允许相对软连接） | `WhitelistSandbox` + `RealPathBoundary` | 共享卷下三处新增风险：跨副本编辑造成悬空、挂载点差异让逻辑/本机路径混用、属性缓存造成 TOCTOU（校验通过后被替换）。**共享卷上的校验比单机更严**：①越界判定必须在**逻辑路径空间**完成；②执行前**二次校验** `toRealPath()`；③dangling 一律拒绝、**不回退到旧目标**（回退等于用过期授权执行） | 10.11 逐项加固；文件 Tool/Shell 必须经 `nativePath` 且过白名单（10.3）；共享模式启动校验工作区标识（10.2） | 沙箱白名单被绕过（模型或恶意 Skill 读写工作区之外）；或"看起来校验通过"实际执行了越界路径——这是安全地基的破坏，不是功能缺陷 |
| **原则七：同步执行模型**（不引入 Reactor/WebFlux/`CompletableFuture`） | 全程同步阻塞 + virtual thread；无异步模型 | **这是集群档最容易被破坏的一条**：续约、轮询、认领重试、排队等待、渠道接管天然"看起来适合"用定时器/回调/Future；一旦引入，`ReActLoop` 的同步语义会被异步边界切断，"标失败不重放"（D6）也无法在同一个语义单元里保证 | 见下方 23.1 的专门展开 | ①`ReActLoop` 里出现"异步续约失败后业务还在跑"⇒ 双写；②D6 无法实现（无法在"检查点"同步中止）；③异常传播断裂 ⇒ 崩溃轮次无人标失败；④违反宪法，特性 plan 应被拒绝 |
| **原则八：Tool 模块三合一**（内置 Tool + MCP Client 合并，不拆多个模块） | `oryxos-tool` 三合一；`AGENT.md` 加载归 `ContextLoader` | 集群档新增的"文件类 Tool 必须用 `nativePath`"容易被误做成"新建一个 StorageTool 模块"来包装；或把 `WorkspaceStorage` 塞进 `oryxos-tool`（依赖方向错误：契约应在 core） | 存储契约在 `oryxos-core`（20.2），`oryxos-tool` 只**使用**它；不新增 Tool 模块；`AGENT.md`/Skill 加载仍归 `ContextLoader`（不在 Tool 里） | 模块间依赖混乱、循环依赖；"Tool 模块拆成多个"是 `CLAUDE.md` 常见陷阱表里明确列出的错误 |
| **042 工作区存储开发约束** | 见 `docs/oryxos-docs/SharedVolumeGuide.md` 口径 | 集群档把"工作区可能不是本地目录"从假设变成现实：裸文件 API、`Path.of` 绕回、忘记 `nativePath`、自行初始化工作区、占用保留路径、把多文件发布当原子 | 10.1–10.11 逐条落地：只用 `root()/resolve()`；插件以 `id()` 显式选择；契约在 core；`nativePath` 必须调用；共享模式先校验 `.workspace-id` 且**失败不回退本地**；管理写预约**不自动过期**；恢复前停全部写者；`.workspace-*` 与 `.staging` 保留；多文件发布明确"可恢复但非目录原子快照"；运行输出按 Agent/run 隔离 | 数据分裂（各副本读写不同副本的数据）；或静默写到本地目录造成"看起来正常但换请求就丢"；或破坏发布中途的工作区 |
| **分阶段克制**（设计原则之一） | 核心阶段只做运行时内核；治理与分布式基础设施后置 | 集群档会遇到大量"顺手就能做"的诱惑：加上 Redis 会更稳、加 MQ 就不丢消息、加认证更安全、加 Nacos 更"企业级"、加会话粘性能缓解热点 | D2/D14 + 19.1 的线外清单 + 19.3 的"先有信号再有组件"准则；每条不做都给了理由与代价；README 第 2 章的线外七条逐条对应 19.1 | 范围蔓延：里程碑失去可验证性（每个"顺便"都是一个新组件的运维面），阶段二无法在验证周期内交付 |

### 23.1 原则七（同步执行模型）的单独展开

**判断：多副本不引入 Reactor/WebFlux/`CompletableFuture`；续约、轮询、认领重试、排队等待、渠道接管全部在调度线程与 virtual thread 上同步阻塞完成。**

**逐项交代每个"看起来该异步"的地方，在本方案里是怎么同步做的：**

| 场景 | 天然的异步诱惑 | 本方案的同步做法 | 为什么同步是对的 |
|------|--------------|----------------|----------------|
| **租约续约** | `ScheduledExecutorService` + `CompletableFuture`，回调里处理失败 | 专用调度线程，每 tick **同步**遍历本实例持有的租约，逐条同步执行 `UPDATE`，在同一线程上判定结果并置"失去租约"标志（4.5） | 续约是毫秒级数据库往返；同步让"失败 ⇒ 置标志"这个动作与续约判定在同一个时序点上，无需任何跨线程协调 |
| **轮询重载** | 事件驱动（WatchService/队列），回调触发重载 | 专用调度线程按 `poll-interval` **同步**读版本号 → **同步**重载 → 成功后**同步**更新内存版本（11.4） | 重载的原子性（构建新视图后一次性替换引用）在单线程上天然成立；异步并发重载需要额外的互斥与状态机 |
| **认领重试** | 退避后投递到一个重试队列 | 在**发起认领的同一个线程**上同步 `Thread.sleep(退避)` 后重试（4.6） | 重试次数少（≤3）、退避短（≤2s）；同步让"最终失败 ⇒ fail-closed"的传播路径没有断点 |
| **会话轮次排队** | 等待队列 + 事件通知（"轮到你了"） | 在**当前 virtual thread** 上同步 sleep + 抖动 + 重试认领（5.3） | virtual thread 上 sleep 不占 OS 线程；同步等待让"超时 ⇒ 返回 409"的语义直接明了，且等待期间不持数据库连接 |
| **渠道接管** | 定时器 + 回调建连 | 接管扫描线程同步 CAS 成功后，在**该线程**（或一个受控的启动线程）上做抖动等待与建连；建连结果同步记录（8.3） | 建连本身是 IO 阻塞（适用 virtual thread）；同步让"接管失败 ⇒ 保留属主 ⇒ 下周期重试"的状态转移无歧义 |
| **停机链路** | 并行关闭各渠道（`Future.allOf`） | **顺序**逐个渠道：释放属主 → 限时 `closeQuietly`（放守护线程 + `join(2s)`）→ 记录结果（8.4） | 顺序让每一步的失败可归因；"限时 join"本身就是同步等待的原语，不需要异步编排 |
| **Service 层** | 声明式 `@Async` / 返回 `Mono` | Spring MVC + virtual thread，方法签名同步返回（继承阶段一） | 异步会改变异常传播与事务边界，而集群档的正确性严重依赖"异常发生在哪个语义单元里" |

**三条不可让步的底线（写进特性 plan 的检查项）：**

1. **不引入任何新的异步抽象到依赖树**：Reactor、WebFlux、RxJava、`CompletableFuture` 作为业务编排手段一律禁止（`CompletableFuture` 作为 Spring 内部实现细节被动出现不在此列，但业务代码不得使用它做编排）。
2. **"失去租约"的传播必须是同步可见的**：它在同一个线程上被写入一个 volatile/atomic 标志，业务线程在检查点同步读取。**不允许**用事件总线/回调/CompletableFuture 传播——那会让"我已经不该继续跑"这件事变成异步通知，而 D6 依赖它在检查点上**必然**可见。
3. **每个后台动作必须能在地图上被指出来**：本方案只有四个后台调度线程（租约续约 + 心跳、版本号轮询、接管扫描、清理），加上 virtual thread 承载的请求。任何新增的"定时/后台/异步"动作必须在 plan 里说明它是哪一个、为什么不能并入已有的四个。

> **为什么这条原则在集群档下更要守（而不是"分布式了就可以放松"）：** 分布式把状态的真相源外置了，正确性从此依赖"我是否仍然是持有者"这个判定。这个判定只有在**同步、同线程、同一时序点**上读取时才是可靠的；一旦把续约与业务放到不同的执行流里，就会出现"业务已经跑完但续约还没反映失败"或反之的窗口——而 D6 与租约语义的全部价值正是在消除这类窗口。

---

## 24. 总结

**阶段二的核心是：把状态与协调从实例里搬出去。** 阶段一的实例是"进程内状态 + 本地文件 + 本地数据库"；阶段二把它们搬成"PostgreSQL（关系型状态与协调原语）+ 共享卷（文件面）"，于是实例变成真正无状态的、可随时生灭的对等副本，一组实例对外表现成一个 OryxOS。搬出去的四类状态是：**谁在做这件事**（租约与认领）、**这件事做过没有**（幂等键与去重）、**谁持有这个连接**（属主）、**文件面哪一版是当前版**（版本号）。

**一句话技术栈增量：** 不新增任何中间件——协调基质复用已支持的 PostgreSQL，文件面只依赖共享卷的"读写可见 + rename 原子"，交付面新增官方 Helm Chart，可观测面新增 `instance_id` 与可选 OTel 导出。

**恰好一次的边界（必须记住的三条）：** ①只承诺**五个粒度**（会话轮次、定时触发、入站事件、渠道属主、知识重建）；②其余链路（审计落库、通知推送、工作区写盘）是"至少一次 + 幂等键"；③崩溃轮次**标失败、不重放**（D6），由用户重发恢复。加上两条必然存在的窗口：**双属主窗口**（18.4，不可完全消除，靠 `lease_id` 栅栏 + 回执去重吸收）与**变更传播延迟**（P99 ≤ 3s，最终一致但有确定上界）。

**代价的诚实清单（选这个方案就要接受这些）：** ①数据库是协调单点，**集群可用性下限 = 数据库可用性**（99.9% 的达成责任在 PostgreSQL 高可用，不在副本数）；②轮询与心跳是固定背景开销，随副本数放大（17.2），缓解手段有限且彼此冲突（提高频率 vs 降负载）；③共享卷的语义弱于对象存储（容量、跨地域、大文件吞吐），这是不用对象存储的代价；④属主只保证"同一时刻一个活跃长连接"，**不保证消息零丢失**（8.7）——兜底是平台重推 + 回执去重，而两者都受平台行为约束；⑤无认证（15.4）与审计不可篡改（15.5）的缺口被明确写出，而不是被"企业级"这个词掩盖。

**承接阶段三的位置：** 阶段二把**单集群内的语义**做自洽了——任意请求落到任意副本都正确、崩溃有确定语义、变更有一致传播。阶段三要做的是另一层：跨节点的 Agent 发现、委托与可靠异步协同（A2A 通信底座）。两者**分开演进**：本方案不为其预留运行时耦合（19.2 的四条"不预留"），只保证它接上来的时候，下面这一层是稳的。

**核心理念不变：** 无状态实例、状态外置——这条从阶段一第一天就写进 `oryxos.md` 设计原则的决定，到阶段二变成可运行、可验证、有明确边界的一组机制；而"分阶段克制"这条纪律，则决定了本方案拒绝了多少"顺手就能加"的东西（19.1 七条线外 + 19.3 六类组件的引入信号）。

> **一句话收口：** 阶段二不是"把 OryxOS 改成分布式系统"，而是"**把已经写好的一份状态，从实例里搬到一个所有实例都看得见的地方，然后用一条 CAS 语句解决所有争抢**"。
