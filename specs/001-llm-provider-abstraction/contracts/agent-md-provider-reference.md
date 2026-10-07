# Contract: `AGENT.md` 中的 Provider 引用（frontmatter）

**类型**：Agent 作者可见的**定义契约**（本特性只读 frontmatter）
**版本**：v1（本特性） | **数据形态**：[data-model.md §7](../data-model.md)

## 契约

Agent 定义中与 Provider 相关的**全部内容**如下——只有"选哪一个"和"行为策略"，没有任何实现细节：

```markdown
---
name: ops-agent
provider:
  name: deepseek-chat     # 必填：注册表条目名（唯一引用字段）
  temperature: 0.3        # 可选：采样参数覆盖
  max_tokens: 2048        # 可选：采样参数覆盖
---
（正文）
```

### 字段契约

| 字段 | 必填 | 语义 |
|------|------|------|
| `provider.name` | 是 | 注册表条目名；不存在于注册表 → 启动 WARN + 调用前失败 |
| `provider.temperature` | 否 | 覆盖条目默认值 |
| `provider.max_tokens` | 否 | 覆盖条目默认值 |
| `provider.top_p` | 否 | 覆盖条目默认值 |

### 明确禁止（配置漂移门禁）

以下字段出现在 frontmatter 时，本特性**忽略其值并打 WARN**，且**不**据此路由：

- `provider.model` / `provider.api_key` / `provider.base_url` / `provider.vendor`
- 任何凭证字面量

> 理由：实现细节集中在 `.oryxos/providers.yaml`。Agent 目录是业务资产，不应随厂商模型演进或凭证轮换而改动。

### 扫描与错误契约

| 场景 | 行为 |
|------|------|
| `.oryxos/agents/` 不存在 | 静默通过（零 Agent 工作区可启动） |
| 无 frontmatter / frontmatter 无法解析 | WARN，跳过该 Agent，**不阻塞启动** |
| 无 `provider` 段 | WARN，跳过（该 Agent 尚不可调用） |
| `provider.name` 未注册 | WARN（含 Agent 名与缺失名称）；该 Agent 调用前以 `PROVIDER_NOT_REGISTERED` 失败；**其它 Agent 不受影响** |
| 出现禁止字段 | WARN（配置漂移提示），忽略该字段 |

**不变量**：扫描**只读** frontmatter，**不读正文**、**不建 Profile**、**不缓存**、**不写任何文件**。

## 迁移说明（与现行文档的差异）

- 现行 `CLAUDE.md` 的 `AGENT.md` 示例含 `provider.model` / `provider.api_key` / `provider.fallback`，与本契约冲突；本特性交付时同步更新该示例（plan 的文档同步任务）。
- `provider.fallback`（023）未来形态为**Provider 名称列表**，不破坏本契约的"名称唯一引用"性质；本特性不实现。
- 本契约由最小读取器实现；完整 Agent 加载（`AgentLoader.deriveProfile` + 正文注入 + skills 扫描）落地后取代该读取器，契约字段不变。
