# Implementation Plan: 核心能力一 · 对接 LLM（Provider 抽象）

**Branch**: `001-llm-provider-abstraction` | **Date**: 2026-10-07 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/001-llm-provider-abstraction/spec.md`

## Summary

实现 OryxOS 的 LLM 接入层：一份**全局 Provider 注册表**（工作区文件 `.oryxos/providers.yaml`，一个名称 = 一个「厂商 + 模型」组合）+ 一个**同步调用契约**（`ProviderService.invoke`，归一返回文本 / 工具调用请求 / token 用量，失败归一为 11 类稳定类别）+ **显式映射装配**（按条目惰性构造模型连接，禁止类型扫描与自动装配）+ **审计即写**（每次尝试恰好一条 `llm_calls`）+ **弱依赖**（零凭证可启动、单家不健康不扩散）+ **被动态势**（态势由审计表派生，无主动探测）。

本特性是**绿地首建**：当前工作区只有文档与 `.specify/`，无任何构建文件，因此本特性同时建立 Maven 多模块骨架与质量门禁。技术取向由宪法与已澄清的 spec 决策共同锁定：Java 21 + Spring Boot 3.x + Spring AI（Alibaba）**仅作协议转换与工具 schema 承载**，循环与调度不在此层；全程同步阻塞 + 虚拟线程，**不引入响应式编程模型**（`reactor-core` 仅作为 Spring AI API 签名的传递依赖被动存在，我们的代码不使用它，见 Constraints 与 research.md §16）。

## Technical Context

**Language/Version**: Java 21（`maven.compiler.release=21`；`mvn` 必须以 `JAVA_HOME=<temurin-21>` 运行——本机 `mvn` 默认跑在 JDK 27 上，已实测）

**Primary Dependencies**（全部经 Maven Central 元数据 + 下载 jar 后 `javap` 实证，详见 [research.md](./research.md) §1）:
- Spring Boot **3.5.16**（3.x 最新补丁；Boot 4.x 已发布但宪法限定 3.x）
- Spring AI BOM **1.1.2**（精确对齐 Spring AI Alibaba 的编译依赖）
- Spring AI Alibaba BOM **1.1.2.4-security-fix**（构建于 Boot 3.5.10；DashScope 实现 artifact 在 1.1.x 更名为 `spring-ai-alibaba-dashscope`）
- 协议 client 用**库**而非 starter：`spring-ai-openai`；`spring-ai-anthropic` 仅登记为扩展点不实现
- 持久化：Boot 托管 Flyway **11.7.2**（✅ 已实证 core 内置 SQLite 支持，无需额外 artifact）+ Boot 托管 `sqlite-jdbc` **3.49.1.0** + `hibernate-community-dialects` **6.6.53.Final**（对齐 Boot 托管 Hibernate 6.6.53）
- 其它：SnakeYAML 2.4（Boot 托管）· Picocli · Logback/SLF4J（JSON）
- 测试：JUnit 5.12.2 / AssertJ 3.27.7 / Mockito 5.17.0（Boot 托管）+ `mockwebserver` 4.12.0 + `archunit-junit5` 1.5.1
- 质量门禁：spotless 3.10.3 + google-java-format 1.37.0 · maven-pmd-plugin 3.28.0 + `p3c-pmd` 2.1.1 · maven-checkstyle-plugin 3.6.0 · spotbugs-maven-plugin 4.10.4.1 + `findsecbugs-plugin` 1.14.0 · dependency-check-maven 13.0.0

**Storage**: SQLite（默认零配置，`jdbc:sqlite:${workspace}/oryxos.db`）/ PostgreSQL 14+（部署选项，按 url 自动识别）；表结构由 Flyway 双轨迁移目录 `db/migration/{vendor}/` 管理，`ddl-auto=none`（宪法原则 VIII）。本特性建 1 张表 `llm_calls`。

**Testing**: `mvn verify` 全绿为准 → 单元（加载校验/采样合并/类别映射）· 契约套件（参数化覆盖文本/工具调用/11 类失败）· 集成（本地桩端点，覆盖 quickstart S1~S8）· 架构守卫（ArchUnit：我们的代码禁 import `reactor.*`/`org.springframework.web.reactive.*`、禁依赖 `spring-boot-starter-webflux`、禁依赖 Spring AI autoconfigure、禁注入 `ChatModel` 集合）· 质量门禁（Spotless + P3C/PMD + Checkstyle + SpotBugs/FindSecBugs + OWASP Dependency-Check）

**Target Platform**: JVM 21 服务端（Linux/macOS 开发机 → 后续容器）；单可执行 fat JAR，零外部服务依赖（LLM 厂商在应用边界之外）

**Project Type**: Maven 多模块（契约库 + 实现模块 + 持久化 + CLI + Spring Boot 装配应用）

**Performance Goals**:
- 包装层自身开销可忽略：不含厂商网络与推理时间的前提下，单次调用路径只增加"注册表查表 + 凭证读取 + 参数合并 + 审计一次写入"。
- 20 并发调用（同/异 Provider 混合）零响应错配、零凭证混用（SC-006）。
- 失败/超时在 `timeout_ms` 上限内返回（SC-008），不无限等待。

**Constraints**:
- 核心阶段全程同步阻塞 + 虚拟线程；**不引入** `spring-boot-starter-webflux`，绝不调用 `webClientBuilder(...)`，我们的代码/契约中不出现 `Flux`/`Mono`/`CompletableFuture`（宪法原则 VII）。⚠️ 澄清：`ChatModel.stream()` 的返回类型使 `reactor-core` 成为 Spring AI 的**传递依赖，无法移除**；这不是"引入异步编程模型"，但**使用**它即违规——守卫落在"我们新增的代码与依赖"上（详见 [research.md](./research.md) §16，需同步进 `CLAUDE.md`）。
- 禁用 Spring AI 自动工具执行：6 参构造器注入 **eligibility predicate 恒 false + no-op `ToolCallingManager`** 双保险；禁用 eager 自动装配（只引库、不引 starter，兜底 `spring.autoconfigure.exclude` 的类名从依赖 jar 的 `AutoConfiguration.imports` 读取）。模型连接由 `Map<ProviderName, ChatModel>` 显式维护（原则 I/II/III）。
- Spring AI 内建重试关闭（`RetryTemplate` maxAttempts=1），保证"一次调用尝试 = 一次外呼 = 一条审计"。
- 零凭证可启动；任何 Provider 不健康不得阻塞启动或影响其它条目（FR-020）。
- 凭证只以 `${ENV_VAR}` 占位出现（字面量加载期报错）；配置、日志、错误信息、审计均不得出现凭证明文（FR-002 / SC-007）。
- 运行期注册表只读；改动重启生效。
- 工作区文件路径必须经真实路径校验，禁止软连接逃逸出工作区根（宪法原则 VI）。
- 构建：`~/.m2` 在沙箱下不可写（实测）→ 工作区内 `.m2repo` + 国内镜像；`repo1.maven.org` 实测可达，**无需放宽沙箱权限**。

**Scale/Scope**: 注册表条目 ~10 量级（本轮验收 ≥3）；6 个 Maven 模块；11 个失败类别；**协议族实现 2 个**（OpenAI 兼容 / DashScope 原生）+ 声明 1 个扩展点（Anthropic，不实现）；2 份迁移文件（sqlite + postgresql 各 1）。

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| 原则 | 本特性的落实方式 | 结论 |
|------|-----------------|------|
| I 自实现 ReAct 循环 (NON-NEGOTIABLE) | 本特性不实现循环，但**不得**引入 Spring AI 的 Agent/自动工具执行能力：只调 `chatModel.call(new Prompt(...))`（✅ 签名已实证），返回的 tool call 原样交给上层。落点：6 参构造器显式注入 **eligibility predicate 恒 false + no-op `ToolCallingManager`**；ArchUnit 守卫"provider 模块不得依赖 `org.springframework.ai.autoconfigure..`"。 | PASS |
| II Spring AI 仅做协议转换与 Schema 生成 (NON-NEGOTIABLE) | 用途仅限：各家协议转换 + 工具声明的 schema 承载。只引**库**（`spring-ai-openai` / `spring-ai-alibaba-dashscope`），不引 starter，零 key 可启动；工具**永不执行**——OpenAI 族走 schema-only `OpenAiApi.FunctionTool`（✅ 已实证该构造器无回调），DashScope 族走永不被调用的 stub 并由契约测试断言 `call()` 未被进入。 | PASS |
| III Provider 显式映射 | 核心即"名称 → 条目 → 模型连接"的显式映射表（`Map<ProviderName, ChatModel>`，一条目一实例，惰性构造）；注册表为唯一真相源，无类型扫描、无按序回退、无隐式默认。测试：同名模型不同端点分别命中各自端点。 | PASS |
| IV 一个目录 = 一个 Agent | 不引入 `.oryxos/profiles/`；Agent 侧新增内容仅"只读 frontmatter 中的 `provider.name` + 采样覆盖"，不建第二份绑定索引、不缓存正文。最小读取器标注 provisional，由后续 `AgentLoader` 取代（见 Complexity Tracking #1）。 | PASS（带声明） |
| V 审计 Day One 落库 (NON-NEGOTIABLE) | 每次调用尝试（含失败/超时/凭证缺失/未注册）恰好一条 `llm_calls`；审计写失败只告警不改判结果。本特性只建 `llm_calls`——本特性不存在工具执行，`tool_invocations` 待能力四落地（见 Complexity Tracking #2）。 | PASS（带声明） |
| VI 安全是地基 (NON-NEGOTIABLE) | 凭证仅 `${ENV_VAR}` 占位（字面量加载期报错）；日志/审计/CLI 输出过滤凭证；注册表文件路径经 `toRealPath()` 校验必须位于工作区根内。不使用 `SecurityManager`。 | PASS |
| VII 同步执行 + 虚拟线程 | 全程同步阻塞：`ProviderService.invoke` 同步返回，超时用同步 `JdkClientHttpRequestFactory.setReadTimeout(Duration)`（✅ 已实证），只传 `RestClient.Builder`；`spring.threads.virtual.enabled=true`。⚠️ **精确化**（已实证 `ChatModel.stream()` 返回 `Flux` ⇒ `reactor-core` 为不可移除的传递依赖）：原则禁止的是**引入并使用**异步编程模型，故守卫为"我们的代码不出现 `Flux`/`Mono`/`CompletableFuture`/`WebClient`，不依赖 `spring-boot-starter-webflux`"，而非"classpath 上不得存在 reactor-core"。已写入 `research.md` §16，需同步进 `CLAUDE.md`。 | PASS（带精确化说明） |
| VIII 目录配置即 Agent / 状态外置 | Agent 与 Provider 均由文件定义，无需写 Java；运行实例无状态——态势数据由审计表派生重建，不依赖进程内计数器；表结构由 Flyway 管理，`ddl-auto=none`。 | PASS |

**质量门禁（宪法"开发流程与质量门禁"）**：Spotless(Google) + 阿里 P3C(PMD) + Checkstyle + SpotBugs/FindSecBugs + OWASP Dependency-Check 全部接入 `mvn verify`；提供最小 CI（`mvn verify`）。OWASP DC 首次运行需下载 NVD 数据，见风险 R6。

**Phase 1 后复检结论**：设计未引入新的原则冲突；`contracts/` 中"工具只下发不执行"、"失败不切换 Provider"、"态势不主动探测"三条不变量均由可执行断言覆盖。**通过**。

## Project Structure

### Documentation (this feature)

```text
specs/001-llm-provider-abstraction/
├── plan.md              # 本文件
├── spec.md              # 已澄清的需求规格
├── research.md          # Phase 0：技术决策与备选（版本唯一真相源）
├── data-model.md        # Phase 1：配置/领域/持久化/派生视图数据形态
├── quickstart.md        # Phase 1：S1~S8 可运行验收指南
├── contracts/           # Phase 1：对外与模块间契约
│   ├── providers-yaml.md
│   ├── provider-service.md
│   ├── llm-calls-schema.md
│   ├── cli.md
│   └── agent-md-provider-reference.md
├── checklists/
│   └── requirements.md  # 规格质量清单
└── tasks.md             # Phase 2 输出（/speckit-tasks 生成，本命令不创建）
```

### Source Code (repository root)

```text
pom.xml                                  # parent：打包 pom，Java 21，依赖/插件版本集中管理
.mvn/maven.config                        # -Dmaven.repo.local=.m2repo -s config/maven-settings.xml
config/
├── maven-settings.xml                   # 镜像（国内源；本环境 Maven Central 不可达）
├── checkstyle/checkstyle.xml
├── p3c/pmd-ruleset.xml
└── spotbugs/exclude.xml
.github/workflows/ci.yml                 # mvn verify（最小 CI）
tools/stub-llm-server/StubLlmServer.java # 验收桩：JDK21 单文件、零依赖、OpenAI 兼容

oryxos-core/                             # 契约与值对象（依赖倒置的唯一入口）
└── src/main/java/io/oryxos/core/
    ├── provider/  ProviderName, ProviderEntry, ProviderVendor, ProviderRegistry,
    │              SamplingParameters, ToolSpec, ChatMessage, LlmRequest,
    │              LlmResult(sealed), LlmSuccess, LlmFailure, ToolCallRequest,
    │              TokenUsage, FailureCategory, ProviderService,
    │              LlmCallAudit, LlmCallAuditStore, ProviderStatus, ProviderStatusReader,
    │              ProviderConfigException
    └── agent/     AgentProviderReference, AgentProviderReferenceScanner
└── src/test/java/io/oryxos/core/...      # 值对象/合并优先级/类别枚举 单测

oryxos-provider/                         # 能力一实现
└── src/main/java/io/oryxos/provider/
    ├── ProviderRegistryLoader            # SnakeYAML 严格加载 + 校验 + 真实路径校验
    ├── ImmutableProviderRegistry
    ├── ChatModelFactory                  # vendor→协议→按条目惰性构造 ChatModel
    ├── ProviderChatModelRegistry         # Map<ProviderName, ChatModel> 显式映射（唯一注入点）
    ├── ToolSpecConverter                 # ToolSpec → Spring AI 工具声明（只描述，不执行）
    ├── FailureMapper                     # 上游信号 → FailureCategory（纯函数）
    └── DefaultProviderService            # invoke：解析→合并→调用→归一→审计
└── src/test/java/io/oryxos/provider/...  # 契约套件 + 加载校验 + 映射 + 不执行工具

oryxos-storage/                          # 持久化（实现 core 的审计/态势接口）
├── src/main/java/io/oryxos/storage/
│   ├── LlmCallEntity, LlmCallJpaRepository
│   ├── LlmCallAuditStoreAdapter        # LlmCallAuditStore 实现 + 凭证过滤 + 失败告警
│   └── ProviderStatusQueryAdapter      # ProviderStatusReader 实现（窗口 + 连续失败计算）
├── src/main/resources/db/migration/sqlite/V1__create_llm_calls.sql
├── src/main/resources/db/migration/postgresql/V1__create_llm_calls.sql
└── src/test/java/io/oryxos/storage/...  # 迁移可跑 + 行语义（NULL≠0）+ 态势口径

oryxos-cli/                              # 命令行（Picocli）
└── src/main/java/io/oryxos/cli/
    ├── OryxOsCommand, ProviderCommand
    └── ProviderListCommand, ProviderShowCommand, ProviderInvokeCommand
└── src/test/java/io/oryxos/cli/...       # 输出契约（含"永不打印凭证"）

oryxos-boot/                             # Spring Boot 装配与可执行入口
├── src/main/java/io/oryxos/boot/
│   ├── OryxOsApplication
│   ├── ProviderWiring                   # 手工 @Bean 装配；排除 Spring AI 自动装配
│   └── ProviderStartupChecks            # 注册表拒绝启动 + Agent 引用 WARN
├── src/main/resources/application.yml
└── src/test/java/io/oryxos/boot/...     # 集成：S1~S8 等价物、零凭证启动、弱依赖、并发
```

**Structure Decision**: 采用 **Maven 多模块**，因为它同时满足三条硬约束：(1) 宪法要求跨模块契约放 `oryxos-core`、下游实现（依赖倒置），因此需要"契约模块 vs 实现模块"的物理边界；(2) 宪法禁止 `oryxos-core` 被具体 Provider/持久化实现污染，模块边界让 ArchUnit 守卫可执行；(3) 单可执行 fat JAR 的交付形态要求聚合模块（`oryxos-boot`）与其余模块分离。依赖方向：`core ← provider`、`core ← storage`、`core ← cli`、`boot → {provider, storage, cli}`；provider 不依赖 storage（审计经 core 接口注入）。

### 模块结构声明（宪法要求）

宪法规定"模块划分跟随能力域，允许新建/改名，但 MUST 在对应特性的 plan 里声明理由，并同步更新 `CLAUDE.md` 模块表与 `TechnicalSolution.md` §10"。本特性实际创建 **6 个模块**：parent + `oryxos-core` + `oryxos-provider` + `oryxos-storage` + `oryxos-cli` + `oryxos-boot`。

- **理由**：绿地首建只需本特性可验收的最小闭环；`oryxos-memory` / `oryxos-tool` / `oryxos-channel-*` / `oryxos-knowledge` / `oryxos-persona` / `oryxos-web` 的能力均不在本特性范围，提前建空壳会产生"空模块 + 假依赖"，违背"分阶段克制"。
- **后续同步**：文档同步工作流把 `CLAUDE.md` 模块表与 `TechnicalSolution.md` §10 标注为"当前已创建 6 个模块，其余按各自特性创建"。
- `oryxos-provider` 的定位与 `CLAUDE.md` 模块表一致（能力一），无需改名。

## Implementation Workstreams

> 供 `/speckit-tasks` 展开为任务；W0 必须先于其余工作流，W9 收尾。

| # | 工作流 | 关键产出 | 覆盖需求 |
|---|--------|---------|---------|
| **W0** | 工程地基与质量门禁 | parent pom（Java 21、依赖/插件版本按 research.md §1 钉定）、6 模块骨架、`.mvn/maven.config`（工作区 repo + 镜像）、Checkstyle/P3C/SpotBugs 配置、`mvn verify` 全门禁接线、最小测试栈、Logback JSON、`.gitignore`/`README`、最小 CI | 宪法"质量门禁"、VIII |
| **W1** | core 契约与值对象 | 全部 `io.oryxos.core.provider` 值对象/sealed 结果/异常/接口 + `io.oryxos.core.agent` 引用模型；单测覆盖命名规则、采样合并优先级、`TokenUsage.unknown()` | FR-005/005a/009/010/014、契约冻结 |
| **W2** | 注册表加载与校验 | SnakeYAML 严格加载、12 条校验规则、真实路径校验、不可变注册表；**加载期只记录环境变量名** | FR-001~004/007、SC-007 |
| **W3** | 显式装配与协议覆盖 | `ChatModelFactory`（vendor→协议族 **OPENAI_COMPATIBLE / DASHSCOPE_NATIVE**（Anthropic 声明不实现）→按条目**惰性**构造 + 缓存）、`ProviderChatModelRegistry` 显式 Map、`ToolSpecConverter`（OpenAI 族走 schema-only `FunctionTool`）、**predicate 恒 false + no-op ToolCallingManager 双保险**、`RetryTemplate` maxAttempts=1、**零凭证启动** | FR-004/008/012/019/020、原则 II/III |
| **W4** | 调用链路与响应归一 | 采样合并、凭证解析（调用时）、同步调用 + 条目级 `timeout_ms`（`JdkClientHttpRequestFactory`）、`FailureMapper` 纯函数（11 类）、响应归一（文本/toolCalls/usage，NULL=未知）、工具声明下发不执行 | FR-005a/009~015、契约套件 |
| **W5** | 审计落库 | Flyway 双轨 `V1__create_llm_calls.sql`、实体与仓库、`LlmCallAuditStoreAdapter`（凭证过滤 + 长度截断 + 写失败告警） | FR-016~018、原则 V、SC-003/007 |
| **W6** | 态势读取 | `ProviderStatusQueryAdapter`（窗口 100 + 连续失败 + `windowSaturated`）、零外部调用、无进程内计数器 | FR-021/022、SC-010 |
| **W7** | CLI 与 boot 装配 | Picocli `provider list/show/invoke`、手工 `@Bean` 装配、启动校验（注册表错误→拒绝启动；Agent 引用缺失→WARN）、虚拟线程、`application.yml` | FR-006、US1/US3 场景、契约 `cli.md` |
| **W8** | 验收资产 | `StubLlmServer`、契约套件（11 类失败参数化）、集成测试（S1~S8 等价物：零凭证启动、交叉路由、零错投、弱依赖、并发 20、工具不执行、审计与态势）、ArchUnit 守卫 | SC-001~010、宪法守卫 |
| **W9** | 文档同步（宪法要求） | `CLAUDE.md`（工作区树加 `providers.yaml`；`AGENT.md` 示例改为 `provider.name` + 采样覆盖；`llm_calls` 表加 3 列；模块表标注当前 6 模块；陷阱表加"Agent 内联 Provider 实现细节"）· `TechnicalSolution.md` §3.1/§3.2/§8.2/§9.2/§10 · 为 023 `fallback` 留名称列表语义说明 | 原则 IV/VIII、治理条款 |

### 验收映射

| 验收项 | 由谁证明 |
|--------|---------|
| SC-001 路由 100% | W8 集成（3 桩交叉 ≥10 次 + `/stats` 零错投）+ quickstart S2 |
| SC-002 新 Provider 接入 ≤15 分钟 / 0 行代码 | quickstart S2 新增第 4 个条目即完成一次成功调用 |
| SC-003 审计覆盖 100% | W5/W8（成功/失败/超时/未注册各一行）+ quickstart S4 |
| SC-004 故障可诊断 100% | W4/W8（11 类参数化）+ quickstart S3 |
| SC-005 契约归一 | W8 契约套件（同一段解析逻辑覆盖全部桩形态） |
| SC-006 并发无串扰 | W8 集成（20 并发）+ quickstart S8 |
| SC-007 凭证零泄漏 | W2/W5/W7（日志、审计、CLI 三层断言）+ quickstart S4 |
| SC-008 超时及时 | W4/W8（`--slow` + `timeout_ms`） |
| SC-009 弱依赖不扩散 | W3/W8（不可达条目 + 其它条目成功率）+ quickstart S1/S5 |
| SC-010 态势时效 | W6/W8（失败返回即反映；与审计行数一致）+ quickstart S4 |

## Complexity Tracking

| 偏离/复杂度 | 为什么需要 | 更简方案为何不可行 |
|------------|-----------|-------------------|
| **#1 本特性内实现最小 `AGENT.md` provider 读取器**（跨入"一个目录 = 一个 Agent"领域） | FR-006 要求"启动期对 Agent 引用给出显式告警"，US1.2 要求"Agent 定义只含名称"可验证；而完整 Agent 加载属后续特性，若不实现最小读取器，本特性最核心的"Agent 只持有名称"无法端到端验收 | 纯接口预留（只定义 `requireEntry`）→ 启动期告警无人调用，FR-006 半条落空，验收只能靠 mock；把完整 `AgentLoader` 拉进来 → 越界更多（正文注入、skills 扫描、Profile 派生全都不属本特性） |
| **#2 只建 `llm_calls`，不建 `tool_invocations`** | 原则 V 字面要求两张表 day-one 写入；但本特性**不存在工具执行**（工具只下发、不执行），`tool_invocations` 无写入方，建表即死代码 | 建空表 → 无写入路径、无测试可断言，且其 schema 应由能力四的真实需求定义；改为"能力四落地时建表并写审计"，语义上并未推迟"审计 day one"原则（该原则约束的是**已存在的行为**必须留痕） |
| **#3 `llm_calls` 相对 `CLAUDE.md` 既有 schema 增 3 列**（`success` / `error_category` / `error_message`） | FR-017 要求失败调用留痕必须含失败类别与错误摘要；原 schema 只描述成功语义 | 只写 `success=false` 而丢弃类别 → 无法支撑 SC-004/SC-010（态势需类别）；用日志旁路存类别 → 违背"不得以日志代替落库"的同一原则 |
| **#4 启动校验分层而非全 fail-fast** | 由 spec 明确决策（Clarifications 2026-10-07）：Provider 是弱依赖，零凭证与个别条目不可达不得阻塞启动 | 全 fail-fast → 与"零 key 可 boot"的容器交付形态冲突（CLAUDE.md entrypoint 明确支持零 key 首启）；全延后校验 → 配置错误潜伏到生产首次调用 |
| **#5 工作区本地 Maven 仓库 + 国内镜像** | 环境实测：`~/.m2` 在文件沙箱下不可写（`Operation not permitted`），而 `repo1.maven.org` 可达；不解决则无法编译自证 | 写 `~/.m2` 需要放宽沙箱权限（`danger-full-access`）——实测证明无必要，把仓库放进工作区即可，最小权限优先；不编译只交付代码 → 违反"质量门禁 MUST 全绿方可合并"，且本轮交付不可自证 |

> #1~#3 为**架构/契约层面的显式声明**，#4 为 spec 已批准的取舍，#5 为环境适配（不改变架构）。
> **另需记录一处对宪法的精确化解释**：`reactor-core` 因 `ChatModel.stream()` 的签名成为不可移除的传递依赖，原则 VII 的守卫因此落在"我们的代码不得使用异步模型"而非"classpath 不得存在 reactor-core"（research.md §16）。

## Risks & Mitigations

| # | 风险 | 影响 | 缓解 |
|---|------|------|------|
| R1 | 依赖下载受限（`~/.m2` 不可写） | 无法编译/测试，交付不可自证 | **已实测收敛**：`repo1.maven.org` 与 `repo.spring.io` 可达，唯一障碍是 `~/.m2` 写权限 → `.mvn/maven.config` 指定工作区 `.m2repo` + `config/maven-settings.xml` 走阿里云镜像（[镜像配置依据](https://developer.aliyun.com/mirror/maven)）。**不需要**放宽沙箱权限。残余：`mvn` 默认 JDK 27，所有命令须前置 `JAVA_HOME=<temurin-21>` |
| R2 | Spring AI / Spring AI Alibaba 的 API 与版本漂移 | 装配代码返工 | **已消解大部分**：版本与关键签名（`OpenAiApi.builder()`、6 参构造器、`ToolExecutionEligibilityPredicate`、`OpenAiApi.FunctionTool`、`Usage` 可空 Integer、`setReadTimeout`）均已 `javap` 实证并写入 research.md §1/§2/§5；残余 ⚠️：`DashScopeChatOptions.Builder` 方法签名与 `spring.autoconfigure.exclude` 的确切类名（G2/G3），实现时现场核对 |
| R3 | ~~Flyway 对 SQLite 需额外 artifact~~ | — | **已关闭**：实证 `flyway-core` 11.7.2（Boot 3.5.16 托管版本）内置 `SQLiteDatabaseType`/`SQLiteParser` 等 6 个类，无需额外依赖。演进提醒：Flyway 13.x 起 SQLite 移出 core，主版本升级时需重新核实 |
| R4 | "工具可描述但绝不执行"无理想 API | 可能被迫承载永不调用的回调 | **已消解**：OpenAI 兼容族可走 schema-only `OpenAiApi.FunctionTool`（无任何回调，✅ 已实证）；DashScope 族仍需 stub，但由 predicate+no-op manager 双保险并断言 `call()` 不可达 |
| R5 | PostgreSQL 迁移在本环境无法运行验证（无 PG 实例） | 部署选项路径未实测 | SQL 逐行评审 + 双轨文件同构检查；如需实测用 Testcontainers（需 Docker，按环境可用性决定），否则在交付报告中显式记为**未验证缺口 G1** |
| R6 | OWASP Dependency-Check 首次需下载 NVD 数据 | 门禁无法本地全绿 | 数据目录指向工作区内 `.dependency-check-data`；绑定 `security-scan` profile 并在 CI 运行；实测网络可达，故首次运行应可完成（若 NVD 主机被限，则换镜像/预置数据目录，并在交付报告中标红该门禁未跑） |
| R7 | 最小 `AGENT.md` 读取器与后续 `AgentLoader` 功能重叠/漂移 | 重复实现 | 契约在 `contracts/agent-md-provider-reference.md` 冻结（字段不变）；读取器标注 provisional 并由后续特性替换；W9 在文档中写明移交关系 |
| R8 | 计划声称的协议族覆盖多于实现（声明 3 族、实现 2 族） | 误解为 Anthropic 已可用 | `ProviderVendor` 枚举的协议族映射中 Anthropic 标为"已声明未实现"；文档同步任务在 `CLAUDE.md`/`TechnicalSolution` 注明 |

## Phase 0 / Phase 1 Outputs

- **Phase 0**：[research.md](./research.md) —— 决策 + 理由 + 备选（含版本唯一真相源与全部未确证项标注）
- **Phase 1**：[data-model.md](./data-model.md) · [contracts/](./contracts/)（5 份）· [quickstart.md](./quickstart.md)
- **下一步**：`/speckit-tasks` 依据本 plan 的 W0~W9 生成依赖有序的 `tasks.md`
