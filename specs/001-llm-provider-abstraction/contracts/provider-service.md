# Contract: `ProviderService`（模块间 SPI）

**类型**：内部代码契约（`oryxos-core` 定义，`oryxos-provider` 实现，上层推理循环/CLI 消费）
**版本**：v1（本特性） | **数据形态**：[data-model.md §2](../data-model.md)

## 契约

```java
package io.oryxos.core.provider;

public interface ProviderService {
    /** 同步阻塞地完成一次模型调用；任何失败都以 LlmFailure 返回，不抛未捕获异常。 */
    LlmResult invoke(LlmRequest request);

    /** 供启动期引用校验使用：注册表中不存在则抛 ProviderConfigException。 */
    ProviderEntry requireEntry(ProviderName name);
}
```

### 输入契约（`LlmRequest`）

| 字段 | 必填 | 语义 |
|------|------|------|
| `provider` | 是 | Provider 条目名称；**唯一**的实现细节入口 |
| `messages` | 是（可空列表 = 空对话） | 消息序列 |
| `tools` | 否 | 本次可用工具声明；仅下发，**不执行** |
| `overrides` | 否 | 采样参数覆盖（覆盖优先级见下） |
| `sessionId` / `traceId` | 否 | 透传到审计记录，用于关联 |

**禁止入参**：`model`、`api_key`、`base_url`、任何凭证值。契约测试必须断言这些字段无法从调用方传入（FR-009）。

### 输出契约（`LlmResult`）

| 情形 | 返回 | 必含字段 |
|------|------|---------|
| 成功 · 纯文本 | `LlmSuccess` | `content` 非空、`toolCalls` 为空、`usage`、`provider`、`model`、`durationMs` |
| 成功 · 工具调用 | `LlmSuccess` | `toolCalls` 非空（`name` + `argumentsJson` 原样透传）、`provider`、`model`、`durationMs` |
| 任意失败 | `LlmFailure` | `category`、`provider`、`model`、`message`（可定位、**不含凭证值**）、`durationMs` |

### 行为不变量（逐条可测）

1. **显式映射**：`provider` 名 → 唯一模型连接；不存在按类型/顺序/默认回退的路径（宪法原则 III）。
2. **未注册名称**：不发出任何外部请求即返回 `PROVIDER_NOT_REGISTERED`。
3. **凭证缺失**：不发出任何外部请求即返回 `CREDENTIAL_MISSING`，且 `message` 含环境变量名与条目名。
4. **不执行工具**：工具在任何情况下都不得被本模块调用；返回的工具调用请求原样交给调用方（宪法原则 II）。
5. **不自动切换**：任何失败都不得触发向其它 Provider 的调用（无 fallback / hedge racing，FR-015）。
6. **同步**：方法返回即结果已定；不得返回需要调用方"等待"的异步句柄（宪法原则 VII）。
7. **审计即写**：成功与失败都恰好追加一条审计记录；审计写失败只告警、不改判调用结果（FR-016/FR-018）。
8. **采样参数优先级**：`request.overrides` > `entry.defaults` > 不发送（模型自身默认）；逐字段独立生效（FR-005a）。
9. **未知工具名/畸形参数**：原样透传给调用方判定，不抛异常、不改变类别（除非上游响应整体无法解析 → `MALFORMED_RESPONSE`）。
10. **线程安全**：同一 `ProviderService` 被并发调用时，凭证、端点、参数、结果互不串扰（SC-006）。

### 失败类别映射契约

上游信号 → `FailureCategory` 的映射必须是**纯函数**且可被桩端点覆盖验证（见 [data-model.md §4](../data-model.md)）。未识别信号一律 `UPSTREAM_ERROR`。

## 消费方示例（伪代码，非实现）

```java
LlmResult result = providerService.invoke(new LlmRequest(
        ProviderName.of("deepseek-chat"),
        messages,
        tools,
        SamplingParameters.ofTemperature(0.3),
        sessionId, traceId));

switch (result) {
    case LlmSuccess ok   -> ok.toolCalls().isEmpty() ? render(ok.content()) : executeTools(ok.toolCalls());
    case LlmFailure err  -> render(err.category(), err.message());
}
```

> `switch` 模式匹配体现契约闭合性：新增结果形态时必须同时更新所有消费方（Java 21 sealed 类型）。

## 契约测试要求

- 一份**契约测试套件**（参数化）对"文本 / 工具调用 / 11 类失败"逐一断言输出形态（不依赖真实厂商）。
- 一份**禁用行为测试**：断言工具未被执行、失败未触发跨 Provider 调用、审计恰好一条。
