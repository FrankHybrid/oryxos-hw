# Contract: `llm_calls` 审计表与态势读取

**类型**：持久化数据契约 + 只读派生视图契约
**版本**：v1（本特性） | **数据形态**：[data-model.md §3/§5](../data-model.md)

## 1. 写入契约

**承诺**：每一次 LLM 调用尝试（成功、失败、超时、凭证缺失、未注册名称）**恰好**一行。这是宪法原则 V 在核心能力一上的落地形态。

| 列 | 类型 | 空值语义 | 备注 |
|----|------|---------|------|
| `id` | 自增主键 | — | SQLite `INTEGER PK AUTOINCREMENT` / PG `BIGSERIAL` |
| `session_id` | VARCHAR(128) | 无会话上下文时为 NULL | 关联会话 |
| `provider` | VARCHAR(64) | **不可空** | 注册表条目名（不是厂商名） |
| `model` | VARCHAR(128) | **不可空** | 注册表解析出的模型名 |
| `prompt_tokens` | INTEGER | **NULL = 厂商未返回** | 禁止用 0 冒充未知 |
| `completion_tokens` | INTEGER | NULL = 未知 | 同上 |
| `total_tokens` | INTEGER | NULL = 未知 | 同上 |
| `trace_id` | VARCHAR(64) | 无则为 NULL | 单轮处理串联标识 |
| `duration_ms` | BIGINT | **不可空** | 含失败耗时（超时记超时上限） |
| `success` | BOOLEAN | **不可空** | 与调用结果一致 |
| `error_category` | VARCHAR(48) | 成功时为 NULL | `FailureCategory` 名（稳定字符串，可聚合） |
| `error_message` | TEXT | 成功时为 NULL | 失败摘要；**写入前过滤凭证值**，超长截断 |
| `created_at` | TIMESTAMP | **不可空** | 调用发起的时刻 |

**索引**：`(provider, created_at)` 支撑态势读取；`(trace_id)` 支撑单轮串联（后续特性）。

**禁止入库**：消息正文、完整响应体、请求头、凭证值、环境变量值。审计表是"调用元数据"，不是"内容留存"（内容留存属治理阶段策略）。

**写入失败契约**：以 ERROR 结构化日志告警 + 不影响调用结果返回（FR-018）。**不得**因审计失败而重试出第二次外部调用。

## 2. 读取契约（`ProviderStatusReader`）

| 方法 | 语义 |
|------|------|
| `ProviderStatus read(ProviderName)` | 单条目态势；无记录返回全空对象（**不是**错误） |
| `List<ProviderStatus> readAll()` | 注册表中所有条目的态势（注册表顺序），含零调用条目 |

**读取口径**（与 SC-010 一致）：
- 倒序取最近 `window`（默认 100）行 → `lastSuccessAt` / `lastFailureAt` / `lastFailureCategory` / `consecutiveFailures`。
- `consecutiveFailures` = 从最新行往前连续 `success = false` 的条数；窗口取满且最新行仍为失败时置 `windowSaturated = true`（语义："连续失败 ≥ window"）。
- 读取**不得**触发任何外部调用，**不得**写入任何行（FR-021/FR-022）。
- 读取**不得**依赖进程内计数器：态势必须可由审计表独立重建（无状态实例，宪法原则 VIII）。

## 3. 校验断言（验收用）

| 断言 | 依据 |
|------|------|
| 3 次成功 + 1 次失败 → 恰好新增 4 行 | FR-016 / SC-003 |
| 失败行的 `success=false` 且 `error_category` 非空 | FR-017 |
| 厂商未返回用量 → 三列均为 NULL（非 0） | spec Edge Cases |
| 一次超时 → 一行，`duration_ms` ≈ 超时上限、`error_category='TIMEOUT'` | FR-017 / SC-008 |
| `consecutiveFailures` 与表中连续失败行数一致 | SC-010 |
| 表中任何列都不含测试所用凭证明文 | SC-007 |
