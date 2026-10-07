# Data Model: 核心能力一 · 对接 LLM（Provider 抽象）

**Feature**: `001-llm-provider-abstraction` | **Date**: 2026-10-07 | **Spec**: [spec.md](./spec.md)

本文件定义本特性的**数据形态**：配置文件 schema、领域值对象、持久化表结构、只读派生视图。实现细节（类的方法体）不在此展开。

---

## 1. 配置数据：`.oryxos/providers.yaml`（全局 Provider 注册表）

**位置**：`${oryxos.workspace-root}/providers.yaml`（默认 `<cwd>/.oryxos/providers.yaml`）。与 `.oryxos/mcp_servers.yaml` 同域，属工作区资产。

**生命周期**：启动时一次性加载 → 校验 → 构建不可变注册表 → 运行期只读。运行期修改文件不生效（重启生效）。

**Schema**：

```yaml
providers:                       # 必填，非空列表
  - name: deepseek-chat          # 必填；全局唯一；^[a-z0-9][a-z0-9._-]{0,63}$
    protocol: openai-compatible  # 必填；已实现的协议族枚举（决定用哪个适配器）
    vendor: deepseek             # 必填；自由标签（1..64），用于默认端点查表与运维展示
    model: deepseek-chat         # 必填；该厂商侧的模型标识
    api_key: ${DEEPSEEK_API_KEY} # 必填；只接受单个 ${ENV_VAR} 占位（字面量一律加载期报错）
    base_url: https://api.deepseek.com   # 可选；已知 vendor 缺省用默认端点；**未知 vendor 时必填**
    timeout_ms: 60000            # 可选；缺省用 oryxos.provider.default-timeout-ms
    defaults:                    # 可选；采样参数默认值
      temperature: 0.7
      max_tokens: 4096
      top_p: 1.0
```

> **为什么 `vendor` 是自由标签而不是枚举**（FR-019 / US5 的关键）：`protocol` 才是决定"用哪个适配器"的东西；把 `vendor` 做成受支持厂商枚举，会让"同一协议族下接入第三家厂商"必须改代码（新增枚举常量），直接违反"新增 Provider 仅通过配置与凭证完成"。因此：**协议族**必须已实现（枚举，否则加载期报错），**厂商**可以是任意标签——只要显式给出 `base_url`。

**校验规则（全部在加载期执行，任一失败即拒绝启动）**：

| 规则 | 违反后果 |
|------|---------|
| `providers` 存在且非空 | 报错：注册表为空，无可用 Provider |
| 每个条目 `name` 非空且符合命名正则 | 报错并指出条目序号 |
| `name` 全局唯一（大小写敏感） | 报错并列出冲突名（FR-007） |
| `protocol` 属于**已实现**协议族枚举（`openai-compatible` / `dashscope`） | 报错并列出已实现协议族；`anthropic` 属**已声明未实现**，明确报"该协议族尚未实现"（FR-007） |
| `vendor` 非空且长度 ≤64 | 报错 |
| 未知 `vendor` 且未提供 `base_url` | 报错并提示"未知厂商请显式提供 base_url"（FR-007 的"不受支持"落点） |
| `model` 非空 | 报错 |
| `api_key` 匹配 `^\$\{[A-Za-z_][A-Za-z0-9_]*\}$` | 报错：凭证必须用环境变量占位（FR-002） |
| `base_url` 缺省或为合法 `http/https` URL | 报错并指出非法值（FR-003） |
| `timeout_ms` 缺省或为正整数 | 报错 |
| `defaults.*` 在合法区间（temperature ∈ [0,2]、max_tokens > 0、top_p ∈ [0,1]） | 报错；**不静默夹取** |
| 未知字段（含拼写错误） | 报错并指出未知字段名（防拼写静默失效） |

**凭证解析时机**：加载期只记录环境变量**名**，不读取其值。值在**每次调用时**读取 → 支持零凭证启动（FR-020）；值缺失或为空串 → 该次调用以 `CREDENTIAL_MISSING` 失败，不发出外部请求。

---

## 2. 领域值对象（`oryxos-core`，跨模块契约）

| 类型 | 形态 | 关键字段与不变量 |
|------|------|-----------------|
| `ProviderName` | 值对象（包装 String） | 非空、匹配命名正则；作为映射键与审计字段 |
| `ProtocolFamily` | 枚举 | 已实现的协议族：`OPENAI_COMPATIBLE`、`DASHSCOPE`；`ANTHROPIC` 为**已声明未实现**（构造即拒并给出明确信息）。决定用哪个适配器 |
| `ProviderVendor` | 值对象（自由标签） | 非空、长度 ≤64；用于默认端点查表（已知厂商表）与运维展示。**不是枚举**——未知厂商只要显式给出 `base_url` 即可用（FR-019/US5） |
| `ProviderEntry` | record（不可变） | `name, protocol, vendor, model, credentialEnvVar, baseUrl(可空), timeoutMs, defaults(SamplingParameters)` |
| `SamplingParameters` | record | `temperature(可空), maxTokens(可空), topP(可空)`；`null` 语义 = "未设置"，等价于交给下一优先级；提供 `merge(defaults, overrides)` |
| `ProviderRegistry` | 只读接口 | `Optional<ProviderEntry> find(ProviderName)`、`List<ProviderEntry> all()`、`boolean contains(ProviderName)`。**唯一真相源**；无写方法 |
| `ToolSpec` | record | `name, description, inputSchemaJson`；工具声明的传输形态（后续由 `OryxTool` 转换而来，本特性不定义 `OryxTool`） |
| `LlmRequest` | record | `ProviderName, List<ChatMessage> messages, List<ToolSpec> tools, SamplingParameters overrides, String sessionId(可空), String traceId(可空)`。**不含 model / 凭证**（FR-009） |
| `ChatMessage` | record | `role(SYSTEM/USER/ASSISTANT/TOOL), content, toolCallId(可空)`；本特性只定义够用的最小形态 |
| `LlmResult` | sealed interface | `LlmSuccess` \| `LlmFailure` |
| `LlmSuccess` | record implements `LlmResult` | `content(可空), List<ToolCallRequest> toolCalls, TokenUsage usage, ProviderName provider, String model, long durationMs` |
| `LlmFailure` | record implements `LlmResult` | `FailureCategory category, ProviderName provider, String model, String message, long durationMs`；**不含任何凭证值** |
| `ToolCallRequest` | record | `id(可空), name, argumentsJson`（原样透传，合法性与工具名有效性由上层判定） |
| `TokenUsage` | record | `promptTokens/completionTokens/totalTokens` 为可空 `Integer`；`null` 表示**厂商未返回**（不等价 0）；提供 `unknown()` |
| `FailureCategory` | 枚举 | 见 §4 |
| `ProviderService` | 接口 | `LlmResult invoke(LlmRequest)`（同步阻塞）；`ProviderEntry requireEntry(ProviderName)` 供启动校验复用 |
| `LlmCallAudit` | record | 一条审计记录的全部字段（见 §3） |
| `LlmCallAuditStore` | 接口 | `void append(LlmCallAudit)`；由 `oryxos-storage` 实现（依赖倒置，provider 模块不接触 JPA） |
| `ProviderStatus` | record | `ProviderName, lastSuccessAt(可空), lastFailureAt(可空), lastFailureCategory(可空), consecutiveFailures(int), observedCalls(int), windowSaturated(boolean)`。**原始事实，不含健康判定**（健康状态机属扩展阶段） |
| `ProviderStatusReader` | 接口 | `ProviderStatus read(ProviderName)`、`List<ProviderStatus> readAll()`；由 storage 从审计表派生实现 |
| `ProviderConfigException` | 受检异常 | 携带人类可读的加载期错误说明（含条目名/字段名），用于拒绝启动 |
| `AgentProviderReference` | record | `String agentName, ProviderName providerName(可空), SamplingParameters overrides, String problem(可空)`；本特性只读 AGENT.md frontmatter |

**命名空间与包名**：`io.oryxos.core.provider`、`io.oryxos.core.agent`（暂定；Apache 化前可整体重命名，不影响契约语义）。

---

## 3. 持久化：`llm_calls` 审计表

**单一职责**：每次 LLM 调用尝试（成功或失败）恰好一行（FR-016）。它同时是本特性**运行态势数据的唯一真相源**（§5）。

**SQLite（`db/migration/sqlite/V1__create_llm_calls.sql`）**：

```sql
CREATE TABLE llm_calls (
  id                INTEGER PRIMARY KEY AUTOINCREMENT,
  session_id        VARCHAR(128),
  provider          VARCHAR(64)  NOT NULL,
  model             VARCHAR(128) NOT NULL,
  prompt_tokens     INTEGER,                    -- NULL = 厂商未返回（未知）
  completion_tokens INTEGER,                    -- NULL = 未知
  total_tokens      INTEGER,                    -- NULL = 未知
  trace_id          VARCHAR(64),
  duration_ms       BIGINT       NOT NULL,
  success           BOOLEAN      NOT NULL,
  error_category    VARCHAR(48),                -- 失败时的 FailureCategory 名
  error_message     TEXT,                       -- 失败摘要（截断存储，禁含凭证）
  created_at        TIMESTAMP    NOT NULL
);
CREATE INDEX idx_llm_calls_provider_created ON llm_calls(provider, created_at);
CREATE INDEX idx_llm_calls_trace            ON llm_calls(trace_id);
```

**PostgreSQL（`db/migration/postgresql/V1__create_llm_calls.sql`）**：同结构，`id BIGSERIAL PRIMARY KEY`、`created_at TIMESTAMPTZ`，索引同名。两 vendor 同版本号 `V1`，只增不改。

**写入语义**：
- 写入发生在调用返回之后、结果交还调用方之前（同步）。
- 凭证值、完整请求体、完整响应体**不入库**；`error_message` 只存失败摘要，且写入前做敏感串过滤（`FR-002`/`SC-007`）。
- 写入失败：记录 ERROR 级结构化日志并继续返回调用结果（FR-018），**不**把调用改判为失败，也不吞掉。

**与 CLAUDE.md 既有 schema 的差异（需同步文档）**：本特性新增 `success / error_category / error_message` 三列。原 schema 无失败信息，无法满足 FR-017；`tool_invocations` 表不在本特性创建（本特性不存在工具执行，无内容可审计）。

---

## 4. 失败类别（`FailureCategory`）

| 常量 | 触发条件 | 审计落库 |
|------|---------|---------|
| `PROVIDER_NOT_REGISTERED` | Agent 引用的名称不在注册表中 | 是（不发外部请求） |
| `CREDENTIAL_MISSING` | 环境变量未设置或为空串 | 是（不发外部请求） |
| `AUTH_FAILED` | 上游 401/403 或鉴权错误 | 是 |
| `RATE_LIMITED` | 上游 429 / 配额耗尽 | 是 |
| `MODEL_NOT_FOUND` | 上游 404 或模型标识不被接受 | 是 |
| `BAD_REQUEST` | 上游 400 类（含参数越界、上下文超窗） | 是 |
| `TIMEOUT` | 超过 `timeout_ms` | 是 |
| `UNREACHABLE` | DNS/连接失败、TLS 失败 | 是 |
| `MALFORMED_RESPONSE` | 上游返回无法解析的响应体/工具调用结构 | 是 |
| `UPSTREAM_ERROR` | 其它 5xx 或未归类上游错误 | 是 |
| `INTERNAL` | 本模块内部未预期异常 | 是 |

**不变量**：映射函数必须是**纯的、可测的**（同一上游信号 → 同一类别）；未识别信号一律落 `UPSTREAM_ERROR`，禁止抛出未捕获异常（spec Edge Cases）。

---

## 5. 只读派生视图：`ProviderStatus`

**来源**：`llm_calls` 表，**不做独立计数存储**（避免双真相源；同时满足宪法"运行实例无状态、状态外置"）。

**读取口径**：按 `provider` 倒序取最近 `WINDOW = 100` 行（`ORDER BY created_at DESC, id DESC LIMIT 100`），在应用层计算：
- `lastSuccessAt` = 最近一行 `success = true` 的 `created_at`
- `lastFailureAt` / `lastFailureCategory` = 最近一行 `success = false` 的 `created_at` / `error_category`
- `consecutiveFailures` = 结果序列**从最新往前**连续 `success = false` 的条数
- `observedCalls` = 窗口内行数；`windowSaturated` = 窗口取满 100 且最新 100 行全为失败（提示"连续失败 ≥ 窗口上限"）

**语义约束**：读取该视图**不得**触发任何外部调用（FR-021/FR-022）；无记录时返回全空的 `ProviderStatus`（不是错误）。

---

## 6. 运行配置（`oryxos-boot`）

| 键 | 默认 | 说明 |
|----|------|------|
| `oryxos.workspace-root` | `${ORYXOS_ROOT:.oryxos}` | 工作区根；注册表与 SQLite 库文件的基准目录 |
| `oryxos.provider.registry-file` | `providers.yaml` | 相对工作区根；必须解析到工作区根**真实路径**之内（软连接逃逸即拒绝，宪法原则 VI） |
| `oryxos.provider.default-timeout-ms` | `60000` | 条目未声明 `timeout_ms` 时的单次调用上限 |
| `oryxos.provider.status-window` | `100` | §5 的读取窗口 |
| `oryxos.provider.scan-agent-references` | `true` | 是否在启动期扫描 `.oryxos/agents/*/AGENT.md` 的 provider 引用 |
| `spring.threads.virtual.enabled` | `true` | 虚拟线程（宪法原则 VII） |
| `spring.jpa.hibernate.ddl-auto` | `none` | 表结构只由 Flyway 管理（宪法原则 VIII） |
| `spring.flyway.locations` | `classpath:db/migration/{vendor}` | 双轨迁移目录 |

**禁配项**（必须显式排除，见 plan）：任何 Spring AI 自动装配（含 DashScope）。
**classpath 禁项**：`spring-boot-starter-webflux` / Reactor（宪法原则 VII）。

---

## 7. Agent 定义中的 Provider 引用（`.oryxos/agents/<name>/AGENT.md`）

本特性**只读 frontmatter**，且只读下列字段；正文与 `skills/` 完全不参与（宪法原则 IV 的正文注入属后续特性）。

```markdown
---
name: ops-agent
provider:
  name: deepseek-chat     # 必填；唯一引用字段
  temperature: 0.3        # 可选覆盖
  max_tokens: 2048        # 可选覆盖
---

（正文，本特性不读取）
```

**扫描语义**（启动期，`scan-agent-references=true` 时）：
- 目录不存在 → 无引用，静默通过（零 Agent 工作区可启动）。
- `provider.name` 已注册 → 通过。
- `provider.name` 未注册 → **WARN 日志**（含 agent 名与缺失名称），实例照常启动；该 Agent 的调用在发出任何外部请求前以 `PROVIDER_NOT_REGISTERED` 失败（FR-006）。
- frontmatter 缺失/无法解析/`provider` 段缺失 → WARN，跳过该 Agent，不影响启动。
- 出现 `model` / `api_key` / `base_url` 等实现细节字段 → WARN（配置漂移提示），值被忽略；**不**据此路由。

> **设计边界**：这是"Agent 只持有名称"这一契约能提前验收所需的最小读取器。完整的 Agent 加载（`deriveProfile`、正文注入、skills 扫描）属后续特性，届时本读取器由 `AgentLoader` 取代，`AgentProviderReference` 形态保持兼容。
