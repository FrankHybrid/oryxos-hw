# Phase 0 Research: 核心能力一 · 对接 LLM（Provider 抽象）

**Feature**: `001-llm-provider-abstraction` | **Date**: 2026-10-07

> **核实方法说明**：本文件的所有版本号与 API 签名均**实证**取得，不依赖记忆：
> - 版本号来自 `https://repo1.maven.org/maven2/<path>/maven-metadata.xml`（Maven Central 官方元数据）
> - 依赖钉定关系来自对应 `.pom` 文件
> - API 签名来自**下载 jar 后 `javap`**（JDK 21）
> - 类是否存在来自 **jar 条目列表**（`unzip -l`）
>
> 标记含义：✅ 已实证 · ⚠️ 未实证（明确写出，留待实现阶段确认）· ❌ 已排除

---

## 1. 版本与坐标钉定 ✅

### Decision

| 组件 | 钉定版本 | 依据 |
|------|---------|------|
| Java | **21**（`maven.compiler.release=21`） | 宪法；本机 Temurin 21.0.12.1 已安装 |
| Spring Boot | **3.5.16** | Maven Central `spring-boot-starter-parent` 元数据中 3.x 最新补丁（3.5.5…3.5.16）。**不取 4.x**（宪法限定 Spring Boot 3.x，尽管 4.0.x/4.1.x 已发布） |
| Spring AI BOM | **1.1.2** | Spring AI Alibaba 1.1.2.4-security-fix 的 autoconfigure 依赖精确钉在 `spring-ai-*:1.1.2` |
| Spring AI Alibaba BOM | **1.1.2.4-security-fix** | 该线构建于 Spring Boot **3.5.10**（与 Boot 3.5.16 同 minor）；`-security-fix` 后缀表明含安全修复。版本列表见 `spring-ai-alibaba-bom` 元数据 |
| DashScope 实现 artifact | `com.alibaba.cloud.ai:spring-ai-alibaba-dashscope` | 1.1.x 线**改名**（1.0.x 叫 `spring-ai-alibaba-core`；`spring-ai-alibaba-core:1.1.2.4-security-fix` 实测 404，`spring-ai-alibaba-dashscope:1.1.2.4-security-fix` 实测 200） |
| 协议 client | `org.springframework.ai:spring-ai-openai`、`spring-ai-anthropic`（仅登记，不实现） | 元数据中 1.1.x 线存在（1.1.5…1.1.8 等）；1.1.2 实测 200 |
| Flyway | **Boot 托管的 11.7.2** | `spring-boot-dependencies:3.5.16` 的 `<flyway.version>11.7.2` |
| SQLite JDBC | **Boot 托管的 3.49.1.0** | `spring-boot-dependencies:3.5.16` 的 `<sqlite-jdbc.version>` |
| Hibernate SQLite 方言 | `org.hibernate.orm:hibernate-community-dialects:6.6.53.Final` | 与 Boot 托管的 Hibernate **6.6.53.Final** 对齐；类 `org.hibernate.community.dialect.SQLiteDialect` 实测存在于该 jar |
| SnakeYAML | Boot 托管 2.4 | `<snakeyaml.version>2.4` |
| 测试 | JUnit 5.12.2 / AssertJ 3.27.7 / Mockito 5.17.0（Boot 托管）+ `com.squareup.okhttp3:mockwebserver:4.12.0` + `com.tngtech.archunit:archunit-junit5:1.5.1` | 元数据实测 |
| 质量门禁 | spotless-maven-plugin **3.10.3** + google-java-format **1.37.0** · maven-pmd-plugin **3.28.0** + `com.alibaba.p3c:p3c-pmd:2.1.1` · maven-checkstyle-plugin **3.6.0** · spotbugs-maven-plugin **4.10.4.1** + `com.h3xstream.findsecbugs:findsecbugs-plugin:1.14.0` · dependency-check-maven **13.0.0** | 各自元数据实测 |

### Rationale

- **对齐而非追新**：Spring AI 版本取 Alibaba 实际编译依赖的 1.1.2（而非更高的 1.1.8），避免"Alibaba 的 autoconfigure 与更高 Spring AI 混用"这一未测试组合。这是本特性唯一愿意为稳定性牺牲补丁版本的取舍。
- **Boot 3.5.16 而非 Alibaba 编译所用的 3.5.10**：同一 minor 的补丁升级是 Boot 的兼容承诺范围，且 3.5.16 含更多修复。
- **用 library 而非 starter**：`spring-ai-openai` / `spring-ai-alibaba-dashscope` 是纯库；starter 会带入 autoconfigure（见 §3，直接冲突）。

### Alternatives considered

| 备选 | 否决理由 |
|------|---------|
| Spring Boot 4.x（4.1.x 已发布） | 宪法明文限定 Spring Boot 3.x；且 Spring AI 2.x/Spring AI Alibaba 2.0.0-M1.1 才对齐 Boot 4，等于同时升级两个大版本 |
| Spring AI 1.1.8 + Alibaba 1.1.2.4-security-fix | 组合未经上游测试；补丁收益（bug 修复未知）小于兼容风险 |
| Spring AI Alibaba 1.0.0.4（Boot 3.4.8 线） | 落后一个 minor 且非 security-fix 线 |
| Flyway 13.9.0 | 13.x 已把 SQLite 移出 core（`flyway-database-nc-sqlite` 最早 13.6.0），引入 Boot 版本管理之外的复杂度；11.7.2 已验证内置 SQLite |

---

## 2. 手工构造多个 ChatModel，并**禁用**自动工具执行 ✅

这是本特性技术风险最高的一处（宪法原则 I/II 的落点），已用 `javap` 核实全部签名。

### Decision

**装配路径（OpenAI 兼容协议族）**：

```java
// 每个注册表条目 → 一个 ChatModel（惰性构造 + 缓存）
OpenAiApi api = OpenAiApi.builder()
        .baseUrl(entry.baseUrlOrDefault())      // ✅ 已核实
        .apiKey(resolvedKey)                    // ✅ apiKey(String) 与 apiKey(ApiKey)
        .restClientBuilder(syncRestClient)      // ✅ 只传同步 RestClient.Builder
        .build();

ChatModel model = new OpenAiChatModel(
        api,
        OpenAiChatOptions.builder()
                .model(entry.model())                     // ✅ 已核实
                .temperature(...).maxTokens(...).topP(...) // ✅ 已核实
                .build(),
        NoOpToolCallingManager.INSTANCE,                  // ① 永不执行工具
        noRetryTemplate(),                                // ② maxAttempts=1
        ObservationRegistry.NOOP,
        (options, response) -> false);                    // ③ 资格判定恒 false
```

**禁用自动执行用"双保险"**，两条都在 6 参构造器里显式注入：
1. `ToolExecutionEligibilityPredicate` 恒返回 `false`（`isToolExecutionRequired` 默认实现即调用它）→ 模型层根本不会尝试执行；
2. 传入自实现的 `ToolCallingManager`，其 `executeToolCalls(prompt, response)` **原样返回** `response`（不执行、不回填）。

> ✅ 已核实：`ToolCallingManager` 接口方法为 `resolveToolDefinitions(ToolCallingChatOptions)` 与 `executeToolCalls(Prompt, ChatResponse)`；`ToolExecutionEligibilityPredicate extends BiPredicate<ChatOptions, ChatResponse>` 且有 `isToolExecutionRequired(...)` 默认方法；`OpenAiChatModel` 与 `DashScopeChatModel` 都提供含该 predicate 的 6 参公开构造器。

**工具声明"只描述、不执行"的两种协议路径**：
- OpenAI 兼容：`OpenAiChatOptions.Builder.tools(List<OpenAiApi.FunctionTool>)` ✅ 已核实；`FunctionTool(Function)` 仅携带名称/说明/参数 schema，**没有任何回调可执行**——这是最干净的满足 FR-012 的路径。
- DashScope：走 `toolCallbacks(List<ToolCallback>)`，传入一个 **`call()` 抛异常且永不被调用**的 stub；由上面 ①③ 双保险保证不可达，并由契约测试断言"stub 的 `call()` 未被进入"。

**重试策略**：`RetryTemplate` 配 `maxAttempts=1`（关闭 Spring AI 内建重试）。理由：一次"调用尝试"必须恰好对应一次外呼，否则 SC-003"每次尝试恰好一条审计记录"与成本归属都不成立。

**响应归一映射**（✅ 全部核实）：
| 我们的字段 | Spring AI 来源 |
|-----------|---------------|
| 最终文本 | `ChatResponse.getResult().getOutput().getText()` |
| 工具调用请求 | `ChatResponse.getResult().getOutput().getToolCalls()` → `AssistantMessage.ToolCall` 记录，字段 `id()/type()/name()/arguments()` |
| token 用量 | `ChatResponse.getMetadata().getUsage()` → `Usage.getPromptTokens()/getCompletionTokens()/getTotalTokens()`（**返回可空 `Integer`**，与 `TokenUsage` 的 null=未知 语义天然一致） |
| 是否含工具调用 | `ChatResponse.hasToolCalls()` |

### Alternatives considered

| 备选 | 否决理由 |
|------|---------|
| 用 starter + `ChatClient.prompt().call()` | ❌ 宪法原则 I/II 明文禁止；默认 `ToolCallingManager` 会**执行**工具，导致工具被调两次 |
| 只靠"传入空 `ToolCallback` 列表"来避免执行 | 不足：模型仍可能返回工具调用，默认 manager 仍会尝试执行 |
| 只注入 no-op `ToolCallingManager`，不加 predicate | 可行但少一层保险；两者都在构造器签名里，成本为零 |
| 自己拼 HTTP 调各家 API | 违反"复用管道"（宪法：协议转换交给 Spring AI），且要自己维护各家协议 |

---

## 3. 规避 eager 自动装配（保证零凭证可启动）✅

### Decision

1. **依赖层**：只引 `spring-ai-openai` / `spring-ai-alibaba-dashscope` 等**库**，不引任何 `*-spring-boot-starter` / `*-autoconfigure`。注册表的每个条目由我们手工 `new`（§2），因此**启动期不需要任何 API key**。
2. **兜底层**：若某依赖间接带入 autoconfigure，在 `oryxos-boot` 显式 `spring.autoconfigure.exclude` 排除（DashScope 的自动装配类名以实际 jar 的 `AutoConfiguration.imports` 为准——⚠️ 未逐字实证，实现阶段从依赖 jar 读取后写入，不用猜测的类名）。
3. **验证层**：集成测试断言"未设置任何 `*_API_KEY` 时上下文中**没有** `ChatModel` 类型的 Bean"，且应用能启动到 ready。

### Rationale

自动装配的三宗罪在本项目都成立：(a) 无 key 时阻断启动（与弱依赖/零凭证可启动冲突）；(b) 靠容器里"有哪些 ChatModel Bean"来区分 Provider，正是原则 III 禁止的类型扫描；(c) 它默认装配会执行工具的 manager。

### Alternatives considered

| 备选 | 否决理由 |
|------|---------|
| 引 starter 但把 `api-key` 设为占位假值 | 会伪装出"凭证已配置"的假象，且密钥语义被污染 |
| 引 starter 仅用其 API 类型、不依赖装配 | 无法保证；autoconfigure 仍在 classpath 上参与条件装配 |
| 用 `@ConditionalOnProperty` 自己控制 | 我们本就要手工装配，多此一举 |

---

## 4. Flyway 与 SQLite ✅（原 R3 风险解除）

### Decision

**使用 Boot 托管的 Flyway 11.7.2 + `flyway-core`，不引入任何额外 database 模块。**

实证：`flyway-core-11.7.2.jar` 内含 `org/flywaydb/core/internal/database/sqlite/` 下的 `SQLiteDatabaseType`、`SQLiteParser`、`SQLiteSchema`、`SQLiteTable`、`SQLiteConnection`、`SQLiteDatabase` 共 6 个类。
迁移目录用 `classpath:db/migration/{vendor}`（`{vendor}` 由 Flyway 解析为 `sqlite` / `postgresql`）。

> ⚠️ 演进提醒：Flyway 13.x 起 SQLite 已移出 core（`org.flywaydb:flyway-database-nc-sqlite`，最早 13.6.0）。未来升级 Flyway 主版本时必须重新核实本结论——写入 plan 的文档同步任务。

### Alternatives considered

| 备选 | 否决理由 |
|------|---------|
| `flyway-database-nc-sqlite` | 11.x 不存在该 artifact（实测元数据只有 13.x）；且 Boot 3.5 不管理它 |
| Flyway 13.9.0 | 需同时引入 nc-sqlite 并脱离 Boot 版本管理；无收益 |
| 自实现迁移执行器 | 仅有 1 张表的场景看似够用，但会与后续所有特性的表结构管理分叉，且违反宪法"表结构由 Flyway 管理" |

---

## 5. 超时控制 ✅

### Decision

每个条目一个**同步** `RestClient`，其 request factory 用 JDK HttpClient 实现：

```java
HttpClient http = HttpClient.newBuilder().connectTimeout(connectTimeout).build();
JdkClientHttpRequestFactory f = new JdkClientHttpRequestFactory(http);
f.setReadTimeout(Duration.ofMillis(entry.timeoutMs()));   // ✅ setReadTimeout(Duration) 已核实
RestClient.Builder b = RestClient.builder().requestFactory(f);
```

超时（连接/读取）→ `FailureCategory.TIMEOUT`；连接被拒/DNS 失败 → `UNREACHABLE`。由于超时是**条目级**的，ChatModel/RestClient 也按条目缓存，与 §2 的"一条目一实例"一致。

### Alternatives considered

| 备选 | 否决理由 |
|------|---------|
| `SimpleClientHttpRequestFactory` | 只有连接/读取的粗粒度设置，且默认实现细节更多 |
| Boot 的 `ClientHttpRequestFactoryBuilder` | 存在于 Boot 3.5（`ClientHttpRequestFactorySettings` 已实证），但带来额外抽象层；直接用 JDK 工厂更透明 |
| WebClient 超时 | ❌ 违反原则 VII（响应式） |
| Resilience4j 熔断/超时 | 熔断属扩展阶段（FR-015 明确核心阶段不做） |
| 靠线程中断（`Future.get(timeout)`） | 需要额外线程与句柄，违反同步模型；且无法保证释放底层连接 |

---

## 6. 注册表解析器选型 ✅

### Decision

**SnakeYAML 2.4（Boot 托管）直接解析 + 手写严格校验**，产出不可变 `ProviderRegistry`。校验规则见 [data-model.md §1](./data-model.md)（含未知字段报错、`api_key` 必须是 `${VAR}`、`base_url` 合法性、名称唯一性、vendor 白名单）。

### Rationale

- 注册表是**工作区文件**（`.oryxos/providers.yaml`），不是 Spring 配置源，用 Spring 的绑定机制反而要把它"塞进" Environment（并可能与 `application.yml` 的优先级/覆盖语义纠缠）。
- 手写校验能给出**精确到字段与条目序号**的中文报错，这是 FR-007 与运维体验的要求；`@ConfigurationProperties` 的校验错误信息在嵌套集合场景下可读性差。
- SnakeYAML 已在项目技术栈中（宪法/CLAUDE.md 列出），不新增依赖。

### Alternatives considered

| 备选 | 否决理由 |
|------|---------|
| Jackson YAML | 可做 unknown-property 失败，但引入第二套 YAML 解析栈 |
| Spring `@ConfigurationProperties` | 与"工作区文件 + 运行期只读 + 加载期拒绝"的语义不贴；错误信息弱 |
| Spring profiles 承载多套注册表 | 让"注册表"变成"环境配置"，与"注册表是工作区资产、Agent 按名引用"冲突 |

---

## 7. 运行态势数据来源 ✅（架构决策）

### Decision

**态势数据由 `llm_calls` 审计表派生**（窗口 100 行 + 应用层计算连续失败），**不设进程内计数器、不建独立状态表**。

### Rationale

1. **单一真相源**：SC-010 要求"连续失败次数与审计中的失败记录数一致"——若另存计数器，二者必然有漂移窗口。
2. **宪法原则 VIII（实例无状态、状态外置）**：进程内计数器是运行状态；派生方案让实例可随时重启、未来可多副本。
3. **零额外写路径**：审计写入是本特性已有的强制动作（原则 V），派生读取不再新增写入面，也就不会再新增一类"写失败怎么办"。
4. **审计数据本来就在**：这正好把"审计 day one 落库"从合规负担变成产品能力（运维可见的态势）。

### Alternatives considered

| 备选 | 否决理由 |
|------|---------|
| 进程内 `ConcurrentHashMap` 计数器 | 双真相源、重启即丢、多副本不一致、违反原则 VIII |
| 独立 `provider_status` 表 | 与审计表信息重复，且需要第二条写路径与失败处理；无新增信息 |
| 实时主动探测生成态势 | 核心阶段明确不做（FR-022，且探测有成本/限流风险） |

---

## 8. 失败类别体系 ✅

### Decision

固定 **11 类** `FailureCategory`（见 [data-model.md §4](./data-model.md)），由**纯函数** `FailureMapper` 从"HTTP 状态码 / 异常类型 / 响应解析结果"映射得到；未识别信号一律 `UPSTREAM_ERROR`。

映射输入的可测性：桩端点可精确制造 401/403/429/404/400/5xx/坏 JSON/延迟/连接拒绝，因此 11 类中的 10 类都能被自动化覆盖；`INTERNAL` 用依赖注入的故障注入覆盖。

### Alternatives considered

| 备选 | 否决理由 |
|------|---------|
| 直接透传厂商原始 error code | 厂商各异、无法聚合（SC-004 要"可区分且稳定"） |
| 把框架异常类型直接抛给上层 | 上层会耦合 Spring AI 异常层次，契约失去稳定性（FR-010/014） |
| 只分"成功/失败"两类 | 无法支撑"故障可诊断"与运维定位（SC-004） |

---

## 9. 审计写入失败的处理 ✅

### Decision

审计写入失败 → **ERROR 结构化日志告警 + 调用结果照常返回**；不重试、不把该次调用改判为失败、不静默。

### Rationale

FR-018 的原文要求；同时避免"审计系统故障放大成业务不可用"。日志是告警通道（本特性不引入告警平台）。

### Alternatives considered

| 备选 | 否决理由 |
|------|---------|
| 抛出异常 | 会让可审计性变成可用性的单点，违反 FR-018 |
| 内存排队后重试 | 引入状态与生命周期（违反原则 VIII），且核心阶段无此必要 |
| 完全静默 | 违反原则 V 的精神（"不得以日志够了为由跳过"同理不适用于反向） |

---

## 10. 凭证占位严格性 ✅

### Decision

`api_key` **只接受**单个 `${ENV_VAR}` 占位；字面量、多个占位、带默认值的形式一律**加载期报错**。加载期只记录**变量名**，值在**每次调用时**读取；缺失/空串 → `CREDENTIAL_MISSING`（不外呼）。

### Alternatives considered

| 备选 | 否决理由 |
|------|---------|
| 允许 `${VAR:default}` | 默认值会变成"事实上的明文凭证来源"，与原则 VI 的意图相悖 |
| 允许字面量（便于本地调试） | 一旦允许，明文密钥必然进入工作区文件与 git 历史（原则 VI） |
| 允许 `${VAR1}${VAR2}` 拼接 | 无真实需求，增加解析歧义 |

---

## 11. 构建环境策略 ✅（实测）

### 实测事实

| 事实 | 实测结果 |
|------|---------|
| `~/.m2` 可写性 | ❌ `mkdir: /Users/frank/.m2: Operation not permitted`（文件沙箱 workspace-write） |
| `repo.maven.apache.org` | ❌ `SSL_ERROR_SYSCALL`（该主机不可达） |
| `repo1.maven.org` | ✅ HTTP 200（**可达**） |
| `repo.spring.io/release` | ✅ HTTP 200 |
| `maven.aliyun.com/repository/public` | ✅ 可达（返回 404 属仓库根的正常响应）；镜像配置见[阿里云官方文档](https://developer.aliyun.com/mirror/maven) |
| `mvn` 默认 JDK | ⚠️ JDK **27**（Homebrew），而 `java` 是 21 → 构建必须显式 `JAVA_HOME=<temurin-21>` |
| JDK 21 单文件源码模式 + `com.sun.net.httpserver` | ✅ 实测可运行（验收桩方案成立） |

### Decision

- 仓库根放 `.mvn/maven.config`：`-Dmaven.repo.local=.m2repo -s config/maven-settings.xml`
- `config/maven-settings.xml`：`mirrorOf=*` → `https://maven.aliyun.com/repository/public`（含 `spring` 代理仓以备扩展）
- `.gitignore` 忽略 `.m2repo/`
- 所有构建命令前置 `JAVA_HOME=<temurin-21>`（写进 quickstart 与 CI）

### Rationale

`~/.m2` 不可写是环境事实，不是设计偏好；工作区内的 `.m2repo` 既满足沙箱约束，也让依赖缓存随工作区可见、可清理。**不需要** wider access：实测网络可达，唯一障碍是写路径。

### Alternatives considered

| 备选 | 否决理由 |
|------|---------|
| 申请 `danger-full-access` 写 `~/.m2` | 实测证明无必要——把仓库放进工作区即可；最小权限优先 |
| 用系统 `settings.xml` 配镜像 | 位于工作区外，不可写 |
| 不编译，只交付代码 | 交付无法自证，违反质量门禁 |

---

## 12. 最小 `AGENT.md` provider 读取器（范围决策）

### Decision

本特性实现**只读 frontmatter** 的最小读取器（`provider.name` + 采样覆盖），用于启动期引用校验与 US1.2 验收；**不读正文、不派生 Profile、不扫 skills、不缓存**。契约冻结在 [contracts/agent-md-provider-reference.md](./contracts/agent-md-provider-reference.md)，标注 provisional，由后续 `AgentLoader` 取代。

### Rationale 与备选

见 [plan.md](./plan.md) 的 Complexity Tracking #1（不重复展开）。

---

## 13. 厂商 → 协议覆盖表 ✅（文档级核实）

| 厂商（注册表 `vendor`） | 推荐接入方式 | 默认端点（示例） | 依据 |
|------------------------|-------------|-----------------|------|
| `deepseek` | OpenAI 兼容 | `https://api.deepseek.com` | [DeepSeek API Docs](https://api-docs.deepseek.com/) 明确提供 OpenAI SDK 用法 |
| `moonshot`（Kimi） | OpenAI 兼容 | `https://api.moonshot.cn/v1` | [Kimi API 开放平台](https://platform.kimi.com/docs/api/overview) |
| `zhipu`（GLM） | OpenAI 兼容 | `https://open.bigmodel.cn/api/paas/v4` | [智谱 OpenAI 兼容文档](https://docs.bigmodel.cn/cn/guide/develop/openai/introduction) |
| `dashscope`（通义） | **两条都支持**：原生（`spring-ai-alibaba-dashscope`）或 OpenAI 兼容模式 | 兼容模式 `https://dashscope.aliyuncs.com/compatible-mode/v1` | [阿里云 Base URL 总览](https://www.alibabacloud.com/help/tc/model-studio/base-url)、[OpenAI-compatible 说明](https://www.alibabacloud.com/help/en/model-studio/qwen-vl-compatible-with-openai) |
| `openai` | OpenAI 兼容 | 厂商默认 | — |
| 自建 vLLM / 网关 | OpenAI 兼容 + `base_url` | 运维提供 | spec US1 场景 4 |

> ⚠️ 上表端点为**文档级**确认；各厂商路径细节（是否需要 `/v1` 后缀）在实现阶段由桩端点之外的联调确认，不阻塞本特性（本特性验收全部走桩端点）。

### Decision

核心阶段实现**两个协议族适配器**：`OPENAI_COMPATIBLE`（覆盖 deepseek / moonshot / zhipu / openai / dashscope 兼容模式 / 自建）+ `DASHSCOPE_NATIVE`（通义原生）。`ANTHROPIC` 作为**已声明的第三个扩展点**登记在 `ProviderVendor` 枚举的协议族映射里，但**不实现适配器**（超出核心阶段需求）。

### Alternatives considered

| 备选 | 否决理由 |
|------|---------|
| 只做 OpenAI 兼容一个协议族 | 会退化成"Spring AI Alibaba 依赖无用"，也与技术方案中"用 Spring AI Alibaba 吸收 DashScope 协议差异"的表述不符；且失去"多协议族扩展点"的真实验证 |
| 同时实现 Anthropic 第三个适配器 | 无企业侧需求证据，属"分阶段克制"应砍掉的范围 |
| 每个厂商一个适配器 | 大量厂商共用 OpenAI 兼容协议，按厂商而非按协议组织会重复实现 |

---

## 14. 测试与验收资产选型 ✅

### Decision

- **验收桩**：`tools/stub-llm-server/StubLlmServer.java`——JDK 21 单文件源码模式运行（✅ 已实测可行），`com.sun.net.httpserver.HttpServer`，零依赖；`--id` 身份归因、`--mode` 制造 7 种响应、`--slow` 制造超时、`/stats` 支持"零错投"断言。
- **单元级 HTTP 断言**：`mockwebserver`（4.12.0）用于失败类别映射与超时的细粒度测试。
- **架构守卫**：`archunit-junit5`（1.5.1）断言"我们的代码不依赖 WebFlux/WebClient/Flux/CompletableFuture""provider 模块不依赖 `org.springframework.ai.autoconfigure..`""不存在对 `ChatModel` 集合的注入"。
- **契约套件**：参数化覆盖"文本 / 工具调用 / 11 类失败"，并对每个形态断言审计恰好一行。

### Alternatives considered

| 备选 | 否决理由 |
|------|---------|
| WireMock | 功能过剩、依赖重；本特性只需可控的假厂商响应 |
| 用真实厂商做验收 | 需要账号/密钥/网络，不可复现（spec Assumptions 明确用桩） |
| Testcontainers 起 PG 验证双轨迁移 | 需 Docker；PG 路径记为未验证缺口（见 §15） |

---

## 15. 已知未验证缺口（显式声明，不隐藏）

| # | 缺口 | 影响 | 处理 |
|---|------|------|------|
| G1 | PostgreSQL 迁移与运行路径**未实测**（本环境无 PG 实例） | 部署选项档的正确性无运行时证据 | 双轨 SQL 逐行同构评审；如需实测用 Testcontainers（需 Docker）；在 tasks 中作为独立任务并标注"可选/需环境" |
| G2 | DashScope 原生适配器的 `DashScopeChatOptions.Builder` 方法签名 ⚠️ 未逐方法实证（`javap` 检索未命中，可能是命名不同） | 该适配器实现细节需现场核对 | 实现阶段先 `javap` 该 Builder 再落码；`DashScopeApi.builder()`（`baseUrl/apiKey/restClientBuilder`）与 `DashScopeChatModel` 6 参构造器 ✅ 已实证 |
| G3 | `spring.autoconfigure.exclude` 的具体类名 ⚠️ 未逐字实证 | 兜底排除项写错会静默无效 | 实现阶段从依赖 jar 的 `AutoConfiguration.imports` 读取真实类名；同时用"无 key 下无 ChatModel Bean"的测试作为最终判据 |
| G4 | 各厂商 base_url 路径细节（`/v1` 后缀等） | 真实联调可能需微调 | 注册表支持自定义 `base_url`，运维可覆盖；不阻塞桩端点验收 |
| G5 | Reactor 无法从 classpath 彻底移除 | 与原则 VII 的字面表述存在张力 | 见 §16，写为精确化解释 + ArchUnit 守卫 |

---

## 16. Reactor 的现实与原则 VII 的精确化 ⚠️（重要澄清）

### 实证事实

`org.springframework.ai.chat.model.ChatModel` 接口签名包含 `Flux<ChatResponse> stream(Prompt)`，因此 **`reactor-core` 是 Spring AI 的传递依赖，无法移除**；`OpenAiApi.Builder` / `DashScopeApi.Builder` 也各有一个 `webClientBuilder(WebClient.Builder)` 方法（`spring-webflux` 类型出现在签名里）。

### Decision（对宪法原则 VII 的精确化，而非豁免）

1. **不引** `spring-boot-starter-webflux`；若 `spring-webflux` 因传递依赖出现，在 `oryxos-boot` 显式 `<exclusion>` 排除（除非证明会破坏类装载）。
2. **绝不调用** `webClientBuilder(...)`；只传 `restClientBuilder(...)`（✅ §5 已实证签名）。
3. 我们的代码、契约、返回值中**不出现** `Flux` / `Mono` / `CompletableFuture` / `WebClient`；`ProviderService.invoke` 是同步返回。
4. ArchUnit 守卫针对"**我们新增的代码与依赖**"：禁止 import `reactor.*` / `org.springframework.web.reactive.*`，禁止依赖 `spring-boot-starter-webflux`。

### Rationale

原则 VII 的意图是"不引入异步编程模型"（不让复杂度与调试难度进入我们的代码与架构）。`reactor-core` 作为第三方 API 签名的一部分被动存在于 classpath，不构成"我们引入了异步编程模型"；但**使用**它就是违规——因此守卫落在"我们的代码"这一侧，并有可执行断言。此澄清需写入 plan 的 Constitution Check 说明与 `CLAUDE.md` 的对应条目，避免后续特性误读为"Reactor 可用"。

---

## 17. 结论：对本特性设计的影响

1. 依赖清单可**精确钉定**（§1），pom 无需 `LATEST` 或猜测版本；Boot 3.5.16 满足宪法的 Spring Boot 3.x 约束。
2. 装配走"**库 + 手工 new + 一条目一实例**"（§2/§5），既满足原则 III 的显式映射，又天然满足零凭证启动与条目级超时。
3. 自动工具执行由 **eligibility predicate 恒 false + no-op manager** 双重关闭（§2），且在 6 参构造器上是显式参数——这是宪法原则 I/II 的可审计落点。
4. 工具声明在 OpenAI 兼容族可走 **schema-only `FunctionTool`**，从根源上不存在可执行的回调（§2）。
5. Flyway 11.7.2 内置 SQLite（§4），**不需要**额外 artifact——R3 风险关闭。
6. 态势数据从审计表派生（§7），避免双真相源，同时让"审计 day one"变成运维可见能力。
7. 构建环境问题用**工作区内 Maven 仓库**解决（§11），不需要放宽沙箱权限。
8. Reactor 的传递存在问题已澄清并给出可执行守卫（§16），后续特性不得据此认为"响应式可用"。
