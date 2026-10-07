---
description: "Task list for feature 001-llm-provider-abstraction"
---

# Tasks: 核心能力一 · 对接 LLM（Provider 抽象）

**Input**: Design documents from `/specs/001-llm-provider-abstraction/`

**Prerequisites**: [plan.md](./plan.md) · [spec.md](./spec.md) · [research.md](./research.md) · [data-model.md](./data-model.md) · [contracts/](./contracts/) · [quickstart.md](./quickstart.md)

**Tests**: **包含测试任务**。理由：本特性 spec 的 5 个用户故事全部以 `Independent Test` 定义验收，SC-001~SC-010 均为可执行断言，且宪法要求 `mvn verify`（Spotless/P3C/Checkstyle/SpotBugs/OWASP）全绿方可合并——测试不是可选项，而是本特性的交付判据。

**Organization**: 按用户故事分阶段，每个故事可独立实现、独立验证、独立交付。

## Format: `[ID] [P?] [Story] Description`

- **[P]**: 可并行（不同文件、无未完成依赖）
- **[Story]**: `US1`~`US5`，对应 spec.md 的用户故事
- 所有任务含确切文件路径

## Path Conventions

Maven 多模块，路径相对仓库根（`/Users/frank/Developer/research/oryx-os-practice`）：

```text
pom.xml · .mvn/ · config/ · tools/stub-llm-server/ · .github/workflows/
oryxos-core/src/{main,test}/java/io/oryxos/core/{provider,agent}/
oryxos-provider/src/{main,test}/java/io/oryxos/provider/{,adapter,tool}/
oryxos-storage/src/{main,test}/java/io/oryxos/storage/ · oryxos-storage/src/main/resources/db/migration/{sqlite,postgresql}/
oryxos-cli/src/{main,test}/java/io/oryxos/cli/
oryxos-boot/src/{main,test}/java/io/oryxos/boot/ · oryxos-boot/src/main/resources/
```

**约定**：所有 `mvn` 命令必须前置 `JAVA_HOME=/Library/Java/JavaVirtualMachines/temurin-21.jdk/Contents/Home`（本机 `mvn` 默认 JDK 27）；依赖/插件版本一律按 [research.md](./research.md) §1 钉定，**不得**猜测版本号。

**⚠️ 生成任务时发现并已修正的设计漏洞**：原 data-model 把 `vendor` 设为"受支持厂商枚举"，这会让"同一协议族下接入第三家厂商"必须改代码（加枚举常量），违反 FR-019/US5。已修正为 `protocol`（协议族枚举，决定适配器）+ `vendor`（自由标签，未知厂商须显式 `base_url`）。`protocol`/`vendor` 的校验组合即 FR-007"不受支持的厂商"的落点：无法确定接入方式（协议族未实现，或未知厂商且未给端点）→ 加载期报错。详见 [data-model.md](./data-model.md) §1、[contracts/providers-yaml.md](./contracts/providers-yaml.md)。

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: 绿地首建——可编译、可门禁、可运行验收桩的多模块工程骨架

- [ ] T001 Create parent `pom.xml`（packaging=pom；Java 21 `maven.compiler.release=21`；parent=spring-boot-starter-parent 3.5.16；`dependencyManagement` 导入 spring-ai-bom 1.1.2 与 spring-ai-alibaba-bom 1.1.2.4-security-fix；聚合 5 个子模块）in `pom.xml`
- [ ] T002 [P] Configure workspace Maven repo and mirror（`-Dmaven.repo.local=.m2repo -s config/maven-settings.xml`；镜像 `mirrorOf=*` → `https://maven.aliyun.com/repository/public`）in `.mvn/maven.config`, `config/maven-settings.xml`
- [ ] T003 [P] Create 5 module POMs with dependency direction `core ← {provider,storage,cli}`、`boot → {provider,storage,cli}`（provider 不依赖 storage；core 不依赖任何实现模块）in `oryxos-core/pom.xml`, `oryxos-provider/pom.xml`, `oryxos-storage/pom.xml`, `oryxos-cli/pom.xml`, `oryxos-boot/pom.xml`
- [ ] T004 [P] Add repository baseline files in `.gitignore`（含 `.m2repo/`、`.oryxos/`、`target/`）, `README.md`, `.editorconfig`
- [ ] T005 Wire all quality gates into `mvn verify`（spotless-maven-plugin 3.10.3 + google-java-format 1.37.0；maven-pmd-plugin 3.28.0 + `com.alibaba.p3c:p3c-pmd:2.1.1`；maven-checkstyle-plugin 3.6.0；spotbugs-maven-plugin 4.10.4.1 + `com.h3xstream.findsecbugs:findsecbugs-plugin:1.14.0`；dependency-check-maven 13.0.0 绑定 `security-scan` profile，数据目录 `.dependency-check-data`）in `pom.xml`
- [ ] T006 [P] Author gate configuration files in `config/checkstyle/checkstyle.xml`, `config/p3c/pmd-ruleset.xml`, `config/spotbugs/exclude.xml`
- [ ] T007 [P] Configure structured JSON logging（Logback，禁用 `System.out`）in `oryxos-boot/src/main/resources/logback-spring.xml`
- [ ] T008 [P] Add minimal CI（`JAVA_HOME=21` + 工作区 repo + `mvn verify`）in `.github/workflows/ci.yml`
- [ ] T009 [P] Implement acceptance stub（`com.sun.net.httpserver.HttpServer`，零依赖；`--id/--shape openai|dashscope/--mode text|tool|401|429|404|400|500|badjson/--slow/--no-usage/--with-tools`；`GET /stats` 返回 `{"id","requests"}`；`GET /healthz`）in `tools/stub-llm-server/StubLlmServer.java`
- [ ] T010 Verify toolchain and record it：`JAVA_HOME=<temurin-21> mvn -q -DskipTests verify` 必须通过；把命令与 Java 21 要求写入 `README.md` in `README.md`

**Checkpoint**: 工程可编译、门禁可跑、桩可启动

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: 全部用户故事共享的契约、持久化地基、禁用自动执行的守卫、boot 骨架

**⚠️ CRITICAL**: 本阶段完成前不得开始任何用户故事

**为什么审计持久化在地基阶段**：宪法原则 V 要求每次调用尝试 day-one 落库，而 US1 的 `invoke` 一落地就会写审计。故 `llm_calls` 迁移 + 实体/仓库 + 基础 append 属"任何故事都依赖"的地基；US4 只负责记录**保真度与韧性**（未知量语义、凭证擦除、截断、写失败降级、完整性证明）。

### core 契约与值对象

- [ ] T011 [P] Implement `ProviderName`（非空、`^[a-z0-9][a-z0-9._-]{0,63}$`）in `oryxos-core/src/main/java/io/oryxos/core/provider/ProviderName.java`
- [ ] T012 [P] Implement `ProtocolFamily`（`OPENAI_COMPATIBLE`/`DASHSCOPE` 已实现；`ANTHROPIC` 声明未实现，构造即拒并给出明确信息）与 `ProviderVendor`（自由标签，长度 1..64，携带已知厂商默认端点表）in `oryxos-core/src/main/java/io/oryxos/core/provider/ProtocolFamily.java`, `oryxos-core/src/main/java/io/oryxos/core/provider/ProviderVendor.java`
- [ ] T013 [P] Implement `SamplingParameters`（`temperature`/`maxTokens`/`topP` 可空；`null` 语义="未设置"；`merge(defaults, overrides)` 逐字段：覆盖值 > 默认值 > 不发送）in `oryxos-core/src/main/java/io/oryxos/core/provider/SamplingParameters.java`
- [ ] T014 [P] Implement `TokenUsage`（三字段可空 `Integer`；`null` 表示厂商未返回，**不等价 0**；提供 `unknown()`）in `oryxos-core/src/main/java/io/oryxos/core/provider/TokenUsage.java`
- [ ] T015 [P] Implement `FailureCategory`（11 类：`PROVIDER_NOT_REGISTERED, CREDENTIAL_MISSING, AUTH_FAILED, RATE_LIMITED, MODEL_NOT_FOUND, BAD_REQUEST, TIMEOUT, UNREACHABLE, MALFORMED_RESPONSE, UPSTREAM_ERROR, INTERNAL`，各带稳定字符串名供审计聚合）in `oryxos-core/src/main/java/io/oryxos/core/provider/FailureCategory.java`
- [ ] T016 [P] Implement `ToolSpec`（`name`/`description`/`inputSchemaJson`，唯一形态=声明，不含执行体）、`ChatMessage`（`role`/`content`/`toolCallId`）、`ToolCallRequest`（`id`/`name`/`argumentsJson` 原样透传）in `oryxos-core/src/main/java/io/oryxos/core/provider/ToolSpec.java`, `oryxos-core/src/main/java/io/oryxos/core/provider/ChatMessage.java`, `oryxos-core/src/main/java/io/oryxos/core/provider/ToolCallRequest.java`
- [ ] T017 [P] Implement `LlmRequest`（`ProviderName`/`messages`/`tools`/`overrides`/`sessionId`/`traceId`；**不得**含 model、api_key、base_url）、`LlmResult`（sealed）、`LlmSuccess`（`content`/`toolCalls`/`usage`/`provider`/`model`/`durationMs`）、`LlmFailure`（`category`/`provider`/`model`/`message`/`durationMs`，message 不含凭证）in `oryxos-core/src/main/java/io/oryxos/core/provider/LlmRequest.java`, `oryxos-core/src/main/java/io/oryxos/core/provider/LlmResult.java`, `oryxos-core/src/main/java/io/oryxos/core/provider/LlmSuccess.java`, `oryxos-core/src/main/java/io/oryxos/core/provider/LlmFailure.java`
- [ ] T018 [P] Implement `ProviderEntry`（`name`/`protocol`/`vendor`/`model`/`credentialEnvVar`/`baseUrl(可空)`/`timeoutMs`/`defaults`）、`ProviderRegistry`（只读：`find`/`all`/`contains`，无写方法）、`ProviderConfigException`（人类可读、含条目名与字段名）in `oryxos-core/src/main/java/io/oryxos/core/provider/ProviderEntry.java`, `oryxos-core/src/main/java/io/oryxos/core/provider/ProviderRegistry.java`, `oryxos-core/src/main/java/io/oryxos/core/provider/ProviderConfigException.java`
- [ ] T019 [P] Implement `ProviderService`（`invoke(LlmRequest)` 同步 + `requireEntry(ProviderName)`）、`LlmCallAudit`、`LlmCallAuditStore`（`append`）、`ProviderStatus`（`lastSuccessAt`/`lastFailureAt`/`lastFailureCategory`/`consecutiveFailures`/`observedCalls`/`windowSaturated`）、`ProviderStatusReader`（`read`/`readAll`）in `oryxos-core/src/main/java/io/oryxos/core/provider/ProviderService.java`, `oryxos-core/src/main/java/io/oryxos/core/provider/LlmCallAudit.java`, `oryxos-core/src/main/java/io/oryxos/core/provider/LlmCallAuditStore.java`, `oryxos-core/src/main/java/io/oryxos/core/provider/ProviderStatus.java`, `oryxos-core/src/main/java/io/oryxos/core/provider/ProviderStatusReader.java`
- [ ] T020 [P] Implement `AgentProviderReference`（`agentName`/`providerName(可空)`/`overrides`/`problem(可空)`）in `oryxos-core/src/main/java/io/oryxos/core/agent/AgentProviderReference.java`
- [ ] T021 [P] Implement `AgentProviderReferenceScanner`（**只读** `.oryxos/agents/*/AGENT.md` frontmatter 的 `provider.name`/`temperature`/`max_tokens`/`top_p`；不读正文、不建 Profile、不缓存、不写文件；frontmatter 缺失/坏 YAML → 产出 `problem` 而非抛错）in `oryxos-core/src/main/java/io/oryxos/core/agent/AgentProviderReferenceScanner.java`
- [ ] T022 [P] Unit test core value objects（命名正则边界、`SamplingParameters.merge` 优先级逐字段、`TokenUsage.unknown()` 三字段为 null、sealed `LlmResult` 消费穷尽、`ProtocolFamily.ANTHROPIC` 构造被拒）in `oryxos-core/src/test/java/io/oryxos/core/provider/ProviderValueObjectsTest.java`

### 持久化地基

- [ ] T023 Create Flyway dual-track V1 for `llm_calls`（SQLite：`id INTEGER PRIMARY KEY AUTOINCREMENT`、`created_at TIMESTAMP`；PostgreSQL：`id BIGSERIAL`、`created_at TIMESTAMPTZ`；两文件同构且均含 `session_id/provider/model/prompt_tokens/completion_tokens/total_tokens/trace_id/duration_ms/success/error_category/error_message`；索引 `(provider, created_at)` 与 `(trace_id)`；同版本号 V1）in `oryxos-storage/src/main/resources/db/migration/sqlite/V1__create_llm_calls.sql`, `oryxos-storage/src/main/resources/db/migration/postgresql/V1__create_llm_calls.sql`
- [ ] T024 [P] Implement `LlmCallEntity`（字段与迁移一一对应）与 `LlmCallJpaRepository` in `oryxos-storage/src/main/java/io/oryxos/storage/LlmCallEntity.java`, `oryxos-storage/src/main/java/io/oryxos/storage/LlmCallJpaRepository.java`
- [ ] T025 Implement `LlmCallAuditStoreAdapter`（基础 `append`：保存一行；失败时 ERROR 结构化日志且**不抛**）in `oryxos-storage/src/main/java/io/oryxos/storage/LlmCallAuditStoreAdapter.java`
- [ ] T026 [P] Test storage foundation（SQLite 迁移可执行建表建索引；一行写入可读回；`success=false` 与 `error_category` 可持久化）in `oryxos-storage/src/test/java/io/oryxos/storage/LlmCallPersistenceTest.java`

### 禁用自动执行与显式映射

- [ ] T027 Implement 禁用自动执行三件套：`NoOpToolCallingManager`（`executeToolCalls` 原样返回入参 `ChatResponse`）、`NeverExecuteEligibilityPredicate`（`isToolExecutionRequired` 恒 `false`）、`NoRetryTemplate`（`maxAttempts=1`）in `oryxos-provider/src/main/java/io/oryxos/provider/tool/NoOpToolCallingManager.java`, `oryxos-provider/src/main/java/io/oryxos/provider/tool/NeverExecuteEligibilityPredicate.java`, `oryxos-provider/src/main/java/io/oryxos/provider/NoRetryTemplate.java`
- [ ] T028 [P] Implement `ProviderChatModelRegistry`（`Map<ProviderName, ChatModel>` 显式映射；按需构造并缓存；**禁止**注入 `List<ChatModel>`/`Map<String, ChatModel>`/`ObjectProvider<ChatModel>`）in `oryxos-provider/src/main/java/io/oryxos/provider/ProviderChatModelRegistry.java`

### boot 骨架与守卫

- [ ] T029 Create boot skeleton：`OryxOsApplication` + `application.yml`（`spring.threads.virtual.enabled=true`、`spring.jpa.hibernate.ddl-auto=none`、`spring.flyway.locations=classpath:db/migration/{vendor}`、`oryxos.workspace-root`、`oryxos.provider.*`、SQLite datasource + `hibernate-community-dialects` dialect；`spring.autoconfigure.exclude` 的类名**从依赖 jar 的 `AutoConfiguration.imports` 读取后填写**，不得猜写）in `oryxos-boot/src/main/java/io/oryxos/boot/OryxOsApplication.java`, `oryxos-boot/src/main/resources/application.yml`
- [ ] T030 [P] Test zero-credential boot + constitution guards（未设置任何 `*_API_KEY` 时应用启动到 ready 且容器内**无 `ChatModel` Bean**；ArchUnit：`oryxos-*` 源码不得 import `reactor.*`/`org.springframework.web.reactive.*`、不得依赖 `spring-boot-starter-webflux`、`oryxos-provider` 不得依赖 `org.springframework.ai.autoconfigure..`）in `oryxos-boot/src/test/java/io/oryxos/boot/ZeroCredentialBootTest.java`, `oryxos-boot/src/test/java/io/oryxos/boot/ConstitutionGuardTest.java`

**Checkpoint**: 地基就绪——契约冻结、迁移可跑、零凭证可启动、自动执行已关闭

---

## Phase 3: User Story 1 - 全局注册表 + 按名称显式路由：Agent 只保留选择权 (Priority: P1) 🎯 MVP

**Goal**: 运维在 `.oryxos/providers.yaml` 注册多个条目；Agent 只写一个 Provider 名称；调用精确落到该条目对应的模型连接，多条目并存零串台。

**Independent Test**: 3 个条目分别指向 3 个身份可识别的桩端点，3 个 Agent 各声明一个名称发起调用；响应可归因、非声明端点零流量、Agent 定义内不出现模型名/凭证/端点。

### Tests for User Story 1

- [ ] T031 [P] [US1] Registry load & validation tests（12 条规则逐条：空注册表、名称非法/重复、`protocol` 未实现、`vendor` 为空、未知 vendor 无 `base_url`、`model` 空、`api_key` 字面量/多占位、`base_url` 非法、`timeout_ms` 非正、`defaults` 越界、未知字段；均须给出条目名或字段名）in `oryxos-provider/src/test/java/io/oryxos/provider/ProviderRegistryLoaderTest.java`
- [ ] T032 [P] [US1] Routing integration test（3 桩 + `/stats`；交叉发起 ≥10 次调用，断言每个桩 `requests` 恰等于打向它的次数、零错投；同名模型不同条目的端点与凭证各自生效）in `oryxos-boot/src/test/java/io/oryxos/boot/ProviderRoutingIT.java`
- [ ] T033 [P] [US1] Agent name-only contract test（构造合法/含禁止字段/frontmatter 损坏三类 `AGENT.md` 样本，断言：解析只取 `provider.name` 与采样覆盖；出现 model/api_key/base_url 时产出漂移提示且**不**用于路由）in `oryxos-core/src/test/java/io/oryxos/core/agent/AgentReferenceContractTest.java`

### Implementation for User Story 1

- [ ] T034 [US1] Implement `ProviderRegistryLoader`（SnakeYAML 严格解析；实现 [data-model.md](./data-model.md) §1 的 12 条校验；`api_key` 必须是单个 `${VAR}`；工作区路径经 `toRealPath()` 校验必须位于工作区根内；**加载期只记录环境变量名，不读值**）in `oryxos-provider/src/main/java/io/oryxos/provider/ProviderRegistryLoader.java`
- [ ] T035 [US1] Implement `ImmutableProviderRegistry`（启动构建、运行期只读、确定性顺序）in `oryxos-provider/src/main/java/io/oryxos/provider/ImmutableProviderRegistry.java`
- [ ] T036 [US1] Implement `SyncRestClientFactory`（按条目 `base_url` + `timeout_ms` 构造**同步** `RestClient`：`HttpClient.newBuilder().connectTimeout(...)` → `JdkClientHttpRequestFactory#setReadTimeout(Duration)` → `RestClient.builder().requestFactory(...)`；**绝不调用** `webClientBuilder`）in `oryxos-provider/src/main/java/io/oryxos/provider/SyncRestClientFactory.java`
- [ ] T037 [US1] Implement `ChatModelFactory`（`ProtocolFamily.OPENAI_COMPATIBLE` → `OpenAiApi.builder().baseUrl().apiKey().restClientBuilder().build()` + 6 参 `OpenAiChatModel` 构造器，注入 `NoOpToolCallingManager`/`NoRetryTemplate`/`ObservationRegistry.NOOP`/`NeverExecuteEligibilityPredicate`；`ANTHROPIC` 明确抛"协议族未实现"）in `oryxos-provider/src/main/java/io/oryxos/provider/ChatModelFactory.java`
- [ ] T038 [US1] Implement `CredentialResolver`（**每次调用时**读环境变量；缺失或空串 → `CREDENTIAL_MISSING`，且不发出任何外部请求；错误信息只含变量名与条目名）in `oryxos-provider/src/main/java/io/oryxos/provider/CredentialResolver.java`
- [ ] T039 [US1] Implement `FailureMapper`（11 类纯函数：401/403→`AUTH_FAILED`、429→`RATE_LIMITED`、404/模型不被接受→`MODEL_NOT_FOUND`、400→`BAD_REQUEST`、连接/DNS/TLS→`UNREACHABLE`、超时→`TIMEOUT`、解析失败→`MALFORMED_RESPONSE`、其它 5xx/未识别→`UPSTREAM_ERROR`、内部异常→`INTERNAL`）in `oryxos-provider/src/main/java/io/oryxos/provider/FailureMapper.java`
- [ ] T040 [US1] Implement `DefaultProviderService.invoke`（解析条目 → 未注册则 `PROVIDER_NOT_REGISTERED`（不外呼）→ 取凭证 → 合并采样参数 → 取/建 ChatModel → `chatModel.call(new Prompt(messages, options))` → 成功归一为 `LlmSuccess` → 追加一条 `llm_calls` → 返回；全路径无未捕获异常）in `oryxos-provider/src/main/java/io/oryxos/provider/DefaultProviderService.java`
- [ ] T041 [US1] Implement `ProviderStartupValidator`（注册表加载失败 → 抛 `ProviderConfigException` 使上下文启动失败并退出非 0；错误信息含条目名/字段名）in `oryxos-boot/src/main/java/io/oryxos/boot/ProviderStartupValidator.java`
- [ ] T042 [US1] Implement `ProviderWiring`（手工 `@Bean`：`ProviderRegistryLoader` → `ProviderRegistry` → `SyncRestClientFactory`/`ChatModelFactory`/`ProviderChatModelRegistry` → `DefaultProviderService`；禁止组件扫描到 Spring AI 自动装配）in `oryxos-boot/src/main/java/io/oryxos/boot/ProviderWiring.java`
- [ ] T043 [US1] Implement CLI `oryxos provider list`（无 TTY 纯文本定列输出：`NAME/VENDOR/MODEL/CREDENTIAL/READY/LAST_OK/LAST_FAIL/FAILS`；`READY` 只反映环境变量是否存在，**永不打印值**；`list` 不发外部请求、不写审计）in `oryxos-cli/src/main/java/io/oryxos/cli/ProviderListCommand.java`
- [ ] T044 [US1] Implement CLI `oryxos provider show <name>`（条目详情 + 生效采样参数；未注册名称 → 退出码 2 且提示可用名称）in `oryxos-cli/src/main/java/io/oryxos/cli/ProviderShowCommand.java`
- [ ] T045 [US1] Wire Picocli CLI into boot（`oryxos` 主入口、命令树 `provider list|show`、Spring 上下文装配、退出码约定 0/1/2）in `oryxos-cli/src/main/java/io/oryxos/cli/OryxOsCommand.java`, `oryxos-cli/src/main/java/io/oryxos/cli/ProviderCommand.java`
- [ ] T046 [US1] Run quickstart S2 and record results（3 条目交叉路由零错投；新增第 4 个条目仅改配置即完成一次成功调用 = SC-002）in `specs/001-llm-provider-abstraction/quickstart.md`

**Checkpoint**: US1 独立可用——多 Provider 显式路由成立，Agent 侧只有名称一个字段

---

## Phase 4: User Story 2 - 统一调用契约：厂商差异被吸收，工具定义随请求下发 (Priority: P1)

**Goal**: 上层拿到统一结构结果（文本 / 工具调用请求 / token 用量）；工具定义随请求下发但**永不被系统执行**；第二个协议族（DashScope）接入而不触碰上层。

**Independent Test**: 同一份请求（含 2 个工具声明）打到两种协议形态的桩端点，响应结构同构、可被同一段解析逻辑处理；断言桩仅被请求一次、工具未被执行。

### Tests for User Story 2

- [ ] T047 [P] [US2] Contract suite（参数化覆盖：纯文本、单/多工具调用、`--no-usage` 用量缺失、工具名不在声明内、`argumentsJson` 非法结构；逐形态断言 `LlmResult` 字段与审计恰好一行）in `oryxos-provider/src/test/java/io/oryxos/provider/ProviderServiceContractTest.java`
- [ ] T048 [P] [US2] Tool declaration never executes test（断言：请求体中含工具名称/说明/参数 schema；桩 `/stats` 的 `requests` 恰为 1（无二次回灌）；DashScope 族的 stub `ToolCallback.call()` **未被进入**（计数器为 0））in `oryxos-provider/src/test/java/io/oryxos/provider/ToolDeclarationNoExecutionTest.java`
- [ ] T049 [P] [US2] Protocol adapter test with `--shape dashscope`（同一 `LlmRequest` 经 DashScope 原生适配器得到与 OpenAI 族同构的 `LlmSuccess`）in `oryxos-boot/src/test/java/io/oryxos/boot/ProviderProtocolAdapterIT.java`

### Implementation for User Story 2

- [ ] T050 [US2] Define protocol adapter SPI（`ProtocolAdapter`：`supports(ProtocolFamily)` + `createChatModel(ProviderEntry, String apiKey, RestClient)`；新增协议族只需实现该接口 + 注册，核心不改）in `oryxos-provider/src/main/java/io/oryxos/provider/adapter/ProtocolAdapter.java`
- [ ] T051 [US2] Implement `OpenAiCompatibleAdapter`（把 T037 的装配逻辑归位到 SPI 实现；`toolCallbacks` 路径不用于 OpenAI 族）in `oryxos-provider/src/main/java/io/oryxos/provider/adapter/OpenAiCompatibleAdapter.java`
- [ ] T052 [US2] Implement `DashScopeAdapter`（`DashScopeApi.builder().baseUrl().apiKey().restClientBuilder().build()` + 6 参 `DashScopeChatModel`；**实现前先 `javap` 核对 `DashScopeChatOptions.Builder` 实际签名**（research.md G2），不得凭记忆写）in `oryxos-provider/src/main/java/io/oryxos/provider/adapter/DashScopeAdapter.java`
- [ ] T053 [US2] Implement `ToolSpecConverter`（OpenAI 族 → schema-only `OpenAiApi.FunctionTool`（无回调）；DashScope 族 → 计数器可观测的 stub `ToolCallback`，`call()` 抛异常）in `oryxos-provider/src/main/java/io/oryxos/provider/ToolSpecConverter.java`
- [ ] T054 [US2] Implement `ResponseNormalizer`（`ChatResponse.getResult().getOutput().getText()` → `content`；`getToolCalls()` → `List<ToolCallRequest>`（字段 `id/name/arguments` 原样）；`getMetadata().getUsage()` → `TokenUsage`，**缺失即 null ≠ 0**）in `oryxos-provider/src/main/java/io/oryxos/provider/ResponseNormalizer.java`
- [ ] T055 [US2] Implement `SamplingParametersResolver`（三级优先级贯通：Agent 覆盖 > 条目默认 > 不发送，逐字段独立；同一 Agent 同一名称在注册表不变时可复现）in `oryxos-provider/src/main/java/io/oryxos/provider/SamplingParametersResolver.java`
- [ ] T056 [US2] Implement CLI `oryxos provider invoke --name <n> [--prompt <p>] [--temperature <t>] [--with-demo-tool]`（成功：stdout 文本 + stderr 结构化摘要；失败：stderr `FAILED category=... provider=... model=... reason=...` + 退出码 1；`--with-demo-tool` 只下发一个 echo 工具声明并把模型返回的工具调用请求作为**结果**打印，绝不执行）in `oryxos-cli/src/main/java/io/oryxos/cli/ProviderInvokeCommand.java`
- [ ] T057 [US2] CLI output contract test（定列格式可断言；`--verbose` 下仍不打印凭证值；`FAILED` 行含 category/provider/model/reason）in `oryxos-cli/src/test/java/io/oryxos/cli/ProviderCliOutputTest.java`
- [ ] T058 [US2] Run quickstart S7 and record results（工具只下发、不执行；桩仅 1 次请求）in `specs/001-llm-provider-abstraction/quickstart.md`

**Checkpoint**: US1 与 US2 同时成立——多协议族下契约同构，工具边界锁死

---

## Phase 5: User Story 3 - 弱依赖：故障快速失败、态势可见，绝不静默换 Provider (Priority: P2)

**Goal**: 失败以 11 类稳定类别返回且可定位；一家不健康不阻塞启动、不连带影响其它条目；态势从真实调用被动累积且读取时不发外呼。

**Independent Test**: 构造 5 类失败逐一调用（凭证缺失/限流/超时/不可达/模型不存在），每次得到带条目名与类别的失败结果、失败条目之外零外呼；让至少一个条目不可达，验证应用仍启动、其它条目调用不受影响、态势即时反映。

### Tests for User Story 3

- [ ] T059 [P] [US3] Failure end-to-end test（5 类失败 + 反例：失败后**未**向其它条目发起补偿调用——用其它桩 `/stats` 增量断言为 0）in `oryxos-boot/src/test/java/io/oryxos/boot/ProviderFailureIT.java`
- [ ] T060 [P] [US3] Weak-dependency test（注册表含不可达条目与凭证未就绪条目时：应用成功启动；引用其它条目的调用成功率与全健康时一致；连带失败 0 次）in `oryxos-boot/src/test/java/io/oryxos/boot/WeakDependencyIT.java`
- [ ] T061 [P] [US3] Provider status query test（窗口 100 内 `lastSuccessAt`/`lastFailureAt`/`lastFailureCategory`；连续失败计数与表中连续失败行数一致；窗口取满且最新为失败 → `windowSaturated=true`；`read`/`readAll` 不触发外部调用、不写行）in `oryxos-storage/src/test/java/io/oryxos/storage/ProviderStatusQueryTest.java`
- [ ] T062 [P] [US3] Agent reference startup test（`AGENT.md` 引用未注册名称 → 启动日志出现 WARN 含 agent 名与缺失名称、实例照常启动、其它 Agent 不受影响；该 Agent 调用在外部请求前以 `PROVIDER_NOT_REGISTERED` 失败）in `oryxos-boot/src/test/java/io/oryxos/boot/AgentReferenceStartupIT.java`

### Implementation for User Story 3

- [ ] T063 [US3] Implement `ProviderStartupChecks`（分层校验：注册表结构性错误 → 启动即拒（T041 已有）；`scan-agent-references=true` 时扫描并 WARN 未注册引用与漂移字段；凭证缺失**不**在启动期校验；任何 Provider 的可达性**不**在启动期探测）in `oryxos-boot/src/main/java/io/oryxos/boot/ProviderStartupChecks.java`
- [ ] T064 [US3] Implement `ProviderStatusQueryAdapter`（按 `provider` 倒序取最近 `oryxos.provider.status-window`（默认 100）行 → 应用层计算四要素 + `windowSaturated`；无记录返回全空对象而非错误；**不依赖进程内计数器**）in `oryxos-storage/src/main/java/io/oryxos/storage/ProviderStatusQueryAdapter.java`
- [ ] T065 [US3] Harden `LlmCallAuditStoreAdapter` degradation path（写入失败 → ERROR 结构化日志 + 调用结果照常返回；不重试出第二次外部调用；不把调用改判为失败）in `oryxos-storage/src/main/java/io/oryxos/storage/LlmCallAuditStoreAdapter.java`
- [ ] T066 [US3] Extend CLI `provider list` with status columns（`LAST_OK`/`LAST_FAIL`/`FAILS`/`READY`；无记录显示 `-`；`windowSaturated` 显示 `≥100`；`list` 仍不发外部请求）in `oryxos-cli/src/main/java/io/oryxos/cli/ProviderListCommand.java`
- [ ] T067 [US3] Wire timeout end-to-end（条目 `timeout_ms` → 读取超时 → `FailureCategory.TIMEOUT`；`duration_ms ≈ timeout_ms`；返回耗时不超过超时上限）in `oryxos-provider/src/main/java/io/oryxos/provider/SyncRestClientFactory.java`
- [ ] T068 [US3] Enforce laziness in `ChatModelFactory`（仅在首次调用时构造并缓存；构造失败只影响该条目；启动路径**不得**触发任何网络访问或凭证读取）in `oryxos-provider/src/main/java/io/oryxos/provider/ChatModelFactory.java`
- [ ] T069 [US3] Run quickstart S1/S3/S5/S6 and record results（零凭证启动、9 类失败类别、弱依赖不扩散、Agent 引用告警不阻塞）in `specs/001-llm-provider-abstraction/quickstart.md`

**Checkpoint**: 弱依赖与态势成立——不健康不扩散、态势可读且即时

---

## Phase 6: User Story 4 - 调用留痕：每次调用都留下可审计的用量与结果 (Priority: P2)

**Goal**: 每次调用尝试恰好一条审计记录，字段与实际调用一致；用量未知显式标记而非 0；凭证与内容不入库。

**Independent Test**: 连续 3 次成功 + 1 次失败 → 审计恰好新增 4 行；字段与响应一致，失败行含类别与摘要；`--no-usage` 调用三列 token 为 NULL；任何列无凭证明文。

> 迁移、实体、仓库与基础 append 已在 Phase 2 完成（宪法原则 V 的 day-one 要求）；本阶段专注记录的**保真度、脱敏与证明**。

### Tests for User Story 4

- [ ] T070 [P] [US4] Audit completeness test（3 成功 + 1 失败 → 恰好 4 行；逐字段比对 provider/model/success/error_category/duration_ms；`session_id`/`trace_id` 透传正确；发起时刻合理）in `oryxos-boot/src/test/java/io/oryxos/boot/AuditCompletenessIT.java`
- [ ] T071 [P] [US4] Credential zero-leak test（把哨兵密钥注入环境变量后跑成功/失败/超时三类调用，断言：配置文件、结构化日志输出、`llm_calls` 全表、CLI stdout/stderr 中哨兵串出现次数为 0）in `oryxos-boot/src/test/java/io/oryxos/boot/CredentialLeakageTest.java`
- [ ] T072 [P] [US4] Token unknown semantics test（`--no-usage` 响应 → 三列均 NULL；有 usage → 三列等于响应值；**断言不存在 0 冒充未知**）in `oryxos-storage/src/test/java/io/oryxos/storage/TokenUnknownSemanticsTest.java`

### Implementation for User Story 4

- [ ] T073 [US4] Implement `SecretScrubber`（从任意待落库/待打印文本中剔除当前注册表所有凭证环境变量的**值**；同时剔除 `Bearer <token>` 形态；不得依赖固定前缀）in `oryxos-provider/src/main/java/io/oryxos/provider/SecretScrubber.java`
- [ ] T074 [US4] Implement `AuditRecordFactory`（从 `LlmRequest` + `LlmResult` + 起止时刻组装 `LlmCallAudit`：成功/失败两态字段就位、`duration_ms` 实测、`success` 与结果一致、`error_message` 经 scrub + 截断）in `oryxos-provider/src/main/java/io/oryxos/provider/AuditRecordFactory.java`
- [ ] T075 [US4] Apply scrubbing and truncation at the persistence boundary（`error_message` 截断上限常量 + scrub 兜底；确保任何调用方都不可能写入未脱敏文本）in `oryxos-storage/src/main/java/io/oryxos/storage/LlmCallAuditStoreAdapter.java`
- [ ] T076 [US4] Run quickstart S4 and record results（审计行数、NULL≠0、连续失败计数、`grep` 明文检查）in `specs/001-llm-provider-abstraction/quickstart.md`

**Checkpoint**: 审计可信——完整性、脱敏、未知量语义三者均有断言

---

## Phase 7: User Story 5 - 接入新厂商不动核心链路 (Priority: P3)

**Goal**: 在既有协议族已被支持的前提下，新增厂商只改配置与凭证；新增协议族只需实现一个 SPI 实现。

**Independent Test**: 用此前未在本环境出现过的 `vendor`（同一协议族）新增条目，零代码改动完成一次成功调用；既有 Agent 行为与审计表现不变。

### Tests for User Story 5

- [ ] T077 [P] [US5] Zero-code vendor onboarding test（注册表新增 `vendor: minimax` + 显式 `base_url`（OpenAI 兼容桩）→ 条目可用于调用；断言仓库中无 `minimax` 相关 Java 代码被新增，且核心模块测试全绿）in `oryxos-boot/src/test/java/io/oryxos/boot/ZeroCodeVendorOnboardingIT.java`
- [ ] T078 [P] [US5] Protocol extensibility architecture test（`oryxos-core` 与 `DefaultProviderService` 不 import 任何具体适配器类；新增适配器只需实现 `ProtocolAdapter` 并注册）in `oryxos-provider/src/test/java/io/oryxos/provider/adapter/ProtocolAdapterArchitectureTest.java`

### Implementation for User Story 5

- [ ] T079 [US5] Complete known-vendor default endpoint table（`deepseek`/`moonshot`/`zhipu`/`dashscope`/`openai` 及各自默认端点；未知 vendor 必须显式 `base_url` 的规则与提示文案）in `oryxos-core/src/main/java/io/oryxos/core/provider/ProviderVendor.java`
- [ ] T080 [US5] Document the extension point（如何新增协议族：实现 `ProtocolAdapter` + 注册 + 加迁移无关的测试；以及"未知厂商零代码接入"的操作步骤）in `oryxos-provider/README.md`
- [ ] T081 [US5] Run quickstart S2 extension step and record results（新增第 5 个条目 `minimax-abab`，零代码完成一次成功调用）in `specs/001-llm-provider-abstraction/quickstart.md`

**Checkpoint**: 全部 5 个用户故事独立可用

---

## Phase 8: Polish & Cross-Cutting Concerns

**Purpose**: 宪法要求的文档同步、部署选项验证、门禁全绿与缺口声明

- [ ] T082 [P] Sync `CLAUDE.md`（工作区树加 `providers.yaml`；`AGENT.md` 示例改为 `provider.name` + 采样覆盖（删除内联 model/api_key/fallback）；`llm_calls` 表加 `success`/`error_category`/`error_message` 三列；模块表标注"当前已创建 6 个模块，其余按各自特性创建"；陷阱表加"Agent 内联 Provider 实现细节"；原则 VII 条目加 reactor-core 传递依赖的精确化说明）in `CLAUDE.md`
- [ ] T083 [P] Sync `docs/oryxos-docs/TechnicalSolution.md`（§3.1 注册表位置与"Agent 仅引用名称"；§3.2 补 `providers.yaml` + 手工装配做法；§8.2 Profile/Agent 配置段；§9.2 `llm_calls` 增列；§10 模块结构标注当前 6 模块；§3.3 为 023 `fallback` 留"Provider 名称列表"语义）in `docs/oryxos-docs/TechnicalSolution.md`
- [ ] T084 [P] Sync `docs/oryxos-docs/DemandAnalysis.md`（§5.3 Provider 抽象补"全局注册表 + Agent 仅引用名称"；§10 `LLM Call` 数据模型补三列）in `docs/oryxos-docs/DemandAnalysis.md`
- [ ] T085 [P] Verify PostgreSQL deployment option（可用 Docker 时用 Testcontainers 跑 `postgresql/V1` 迁移与一次写入读回；不可用则在 [research.md](./research.md) 缺口 G1 记录未验证状态与理由）in `oryxos-storage/src/test/java/io/oryxos/storage/PostgresMigrationIT.java`
- [ ] T086 Run OWASP Dependency-Check first pass（NVD 数据落入 `.dependency-check-data`；若 NVD 主机不可达，按 [plan.md](./plan.md) R6 处置并在交付报告标红）in `pom.xml`
- [ ] T087 Run full `mvn verify` until green（Spotless + P3C/PMD + Checkstyle + SpotBugs/FindSecBugs + 全部测试；记录命令与结果）in `README.md`
- [ ] T088 [P] Walk quickstart S1~S8 end to end and record actual results（每个场景标注通过/偏差，含 SC-001~SC-010 对照）in `specs/001-llm-provider-abstraction/quickstart.md`
- [ ] T089 [P] Verify offline rebuild reproducibility（`JAVA_HOME=<temurin-21> mvn -o -q verify` 在有缓存的 `.m2repo` 下可复现；记录 JAVA_HOME 与命令）in `README.md`
- [ ] T090 [P] Record residual gaps and deferrals（research.md 缺口 G1~G5 的最终状态；Anthropic 协议族"已声明未实现"；`tool_invocations` 归属能力四；注册表运行期管理与主动探测属扩展阶段）in `specs/001-llm-provider-abstraction/research.md`

---

## Dependencies & Execution Order

### Phase Dependencies

- **Phase 1 Setup（T001–T010）**: 无依赖，立即开始
- **Phase 2 Foundational（T011–T030）**: 依赖 Phase 1；**阻塞全部用户故事**
- **Phase 3–7 用户故事**: 依赖 Phase 2 完成
  - US1 与 US2 均为 P1；US1 是 MVP，US2 依赖 US1 的 `invoke` 骨架（T040）与装配（T042）
  - US3 依赖 US1（失败路径与启动校验）与 US2（`ResponseNormalizer` 的用量字段）
  - US4 依赖 US1 的审计写入路径（T040）与 US3 的降级路径（T065）
  - US5 依赖 US2 的适配器 SPI（T050–T052）
- **Phase 8 Polish**: 依赖全部目标故事完成

### Critical Path

`T001 → T005 → T010 → T011–T030 → T034–T042 → T040 → T046`（US1 闭环）`→ T050–T056 → T058`（US2）`→ T063–T067 → T069`（US3）`→ T073–T075 → T076`（US4）`→ T079–T081`（US5）`→ T082–T090`（收尾）

### Within Each User Story

1. 先写测试任务（`[P]`）并确认其**失败**
2. 再按"值对象 → 加载/装配 → 服务 → CLI/boot 装配 → 集成"顺序实现
3. 最后跑该故事的 quickstart 场景并记录结果

### Parallel Opportunities

- **Phase 1**：T002/T003/T004 并行；T006/T007/T008/T009 并行（不同文件）
- **Phase 2**：T011–T022 两两文件无关，可大规模并行；T024 与 T026 并行；T028 与 T029 需在 T011–T019 之后
- **Phase 3**：T031/T032/T033 三个测试并行；T043/T044（两个 CLI 命令类）并行
- **Phase 5**：T059–T062 四个测试并行；T064（storage）与 T066（cli）并行
- **Phase 6**：T070–T072 三个测试并行
- **Phase 8**：T082/T083/T084 三份文档并行；T085/T088/T089/T090 并行

---

## Parallel Example: User Story 1

```bash
# 三个验收测试并行编写（先失败）
Task: "Registry load & validation tests in oryxos-provider/src/test/java/io/oryxos/provider/ProviderRegistryLoaderTest.java"
Task: "Routing integration test in oryxos-boot/src/test/java/io/oryxos/boot/ProviderRoutingIT.java"
Task: "Agent name-only contract test in oryxos-core/src/test/java/io/oryxos/core/agent/AgentReferenceContractTest.java"

# 两个 CLI 命令类并行实现
Task: "Implement ProviderListCommand in oryxos-cli/src/main/java/io/oryxos/cli/ProviderListCommand.java"
Task: "Implement ProviderShowCommand in oryxos-cli/src/main/java/io/oryxos/cli/ProviderShowCommand.java"
```

## Parallel Example: Foundational

```bash
# core 值对象与契约可并行（不同文件）
Task: "Implement ProviderName in oryxos-core/src/main/java/io/oryxos/core/provider/ProviderName.java"
Task: "Implement SamplingParameters in oryxos-core/src/main/java/io/oryxos/core/provider/SamplingParameters.java"
Task: "Implement TokenUsage in oryxos-core/src/main/java/io/oryxos/core/provider/TokenUsage.java"
Task: "Implement FailureCategory in oryxos-core/src/main/java/io/oryxos/core/provider/FailureCategory.java"
```

---

## Implementation Strategy

### MVP First（仅 US1）

1. Phase 1 Setup → 2. Phase 2 Foundational → 3. Phase 3 US1
4. **STOP & VALIDATE**：跑 T046（quickstart S2）——3 条目零错投 + 新增条目零代码即接入
5. 此时已具备可交付价值：**多 Provider 显式路由 + 审计落库 + 零凭证启动**

### Incremental Delivery

| 增量 | 内容 | 交付后可验证 |
|------|------|-------------|
| 1（MVP） | Setup + Foundational + US1 | SC-001/002/009（部分） |
| 2 | + US2 | SC-005 + 工具边界 |
| 3 | + US3 | SC-004/008/009/010 |
| 4 | + US4 | SC-003/007 |
| 5 | + US5 | SC-002 完整 |
| 6 | Polish | SC-001~010 全量 + 门禁全绿 |

### Parallel Team Strategy

Phase 1 + Phase 2 全体共同完成（契约必须先冻结）。之后：
- 开发者 A：US1 → US4（审计路径）
- 开发者 B：US2（契约/适配器）
- 开发者 C：US3（弱依赖/态势）+ CLI
- US5 与 Polish 在 US1/US2 稳定后介入

---

## Notes

- **[P] 含义**：不同文件、无未完成依赖；同一文件的任务（如 `ChatModelFactory` 在 T037/T068，`ProviderListCommand` 在 T043/T066，`LlmCallAuditStoreAdapter` 在 T025/T065/T075）**必须串行**
- **版本纪律**：任何依赖/插件版本取自 [research.md](./research.md) §1，禁止猜测；`DashScopeChatOptions.Builder` 与 `spring.autoconfigure.exclude` 类名是仅有的两处需现场核实项（G2/G3）
- **宪法守卫**：T030 的 ArchUnit 守卫必须保持绿；它同时守住原则 II（不依赖 Spring AI autoconfigure）、III（不注入 ChatModel 集合）、VII（我们的代码不用 Reactor/WebFlux）
- **零凭证**：任何任务都不得引入"启动期需要 API key"的行为（T030/T060 是回归防线）
- **审计不可省**：`invoke` 的任何新分支（含新增失败类别）都必须写且只写一条审计
- **提交纪律**：每个任务或逻辑任务组后提交；每个 Checkpoint 停下来独立验证
- **避免**：模糊任务、同文件并行、跨故事的破坏性依赖
