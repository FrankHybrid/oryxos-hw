# Contract: `.oryxos/providers.yaml`（全局 Provider 注册表）

**类型**：运维/Agent 作者可见的**配置契约**（非代码 API）
**版本**：v1（本特性） | **数据形态**：[data-model.md §1](../data-model.md)

## 契约

注册表是**一个文件、一份真相源**。运维在此声明所有可用的 Provider 条目；Agent 只能按 `name` 引用，不能在此文件之外声明任何 Provider 实现细节。

```yaml
providers:
  - name: deepseek-chat
    protocol: openai-compatible
    vendor: deepseek
    model: deepseek-chat
    api_key: ${DEEPSEEK_API_KEY}
    base_url: https://api.deepseek.com
    timeout_ms: 60000
    defaults:
      temperature: 0.7
      max_tokens: 4096
```

### 字段契约

| 字段 | 必填 | 形态 | 语义 |
|------|------|------|------|
| `name` | 是 | `^[a-z0-9][a-z0-9._-]{0,63}$`，全局唯一 | Agent 引用的**唯一标识**；一个名称 = 一个「厂商 + 模型」组合 |
| `protocol` | 是 | 已实现协议族：`openai-compatible` / `dashscope` | **决定用哪个适配器**；`anthropic` 为已声明未实现，写了即加载失败并明确提示 |
| `vendor` | 是 | 自由标签，1..64 字符 | 厂商标签；用于默认端点查表与运维展示。**未知厂商 + 未给 `base_url` → 加载失败**（这正是 FR-007"不受支持"的落点）；未知厂商 + 给了 `base_url` → 可用 |
| `model` | 是 | 非空字符串 | 厂商侧模型标识 |
| `api_key` | 是 | `${ENV_VAR}` | 只接受占位；字面量一律加载失败。**值不在加载期读取** |
| `base_url` | 否（未知厂商时必填） | `http(s)://` URL | 自建网关/代理/私有部署；已知厂商缺省用默认端点 |
| `timeout_ms` | 否 | 正整数 | 单次调用上限；缺省取全局默认 |
| `defaults.temperature` | 否 | [0,2] | 采样参数默认值 |
| `defaults.max_tokens` | 否 | > 0 | 采样参数默认值 |
| `defaults.top_p` | 否 | [0,1] | 采样参数默认值 |

> **设计要点（保证 FR-019/US5"新增厂商仅改配置"）**：把"用哪个适配器"交给 `protocol`（枚举、必须已实现），把"哪家厂商"交给自由标签 `vendor`。若把 `vendor` 做成受支持厂商枚举，"同一协议族下接入第三家"就必须改代码加枚举常量，与本能力的核心承诺直接冲突。

### 行为契约

| 场景 | 必须的行为 |
|------|-----------|
| 文件不存在 | 加载失败并明确报出期望路径（不静默空注册表） |
| 未知字段 / 拼写错误 | 加载失败并指出字段名（禁止静默忽略） |
| 名称重复 | 加载失败并列出冲突名 |
| `api_key` 写成字面量 | 加载失败并提示改用 `${ENV_VAR}` |
| 环境变量未设置/为空 | **加载成功**；调用时以 `CREDENTIAL_MISSING` 失败且不发出请求 |
| 运行期修改本文件 | 不生效、不部分生效；重启后整体生效 |
| 条目指向不可达端点 | **不影响启动**，不影响其它条目调用 |

### 变更契约（兼容性）

- 新增**可选**字段 = 向前兼容；新增**必填**字段 = 破坏性变更，需 spec 更新。
- `fallback` 字段（023 扩展）在本特性不实现；未来形态为**Provider 名称列表**，与"Agent 只持有名称"一致。

## 示例：多条目并存

```yaml
providers:
  - name: deepseek-chat                       # 直达厂商
    protocol: openai-compatible
    vendor: deepseek
    model: deepseek-chat
    api_key: ${DEEPSEEK_API_KEY}

  - name: qwen-plus                           # 走 DashScope 原生协议
    protocol: dashscope
    vendor: dashscope
    model: qwen-plus
    api_key: ${DASHSCOPE_API_KEY}

  - name: ops-fast                             # 业务别名：同厂商不同模型档位
    protocol: openai-compatible
    vendor: deepseek
    model: deepseek-chat
    api_key: ${DEEPSEEK_API_KEY}
    defaults: { temperature: 0.1, max_tokens: 1024 }

  - name: kimi-internal                        # 自建网关 + 自定义端点
    protocol: openai-compatible
    vendor: moonshot
    model: moonshot-v1-32k
    api_key: ${KIMI_INTERNAL_KEY}
    base_url: https://llm-gateway.corp.example/v1

  - name: minimax-abab                         # 全新厂商，零代码接入（未知 vendor + 显式端点）
    protocol: openai-compatible
    vendor: minimax
    model: abab6.5s-chat
    api_key: ${MINIMAX_API_KEY}
    base_url: https://api.minimax.chat/v1
```
