# Contract: CLI（`oryxos provider …`）

**类型**：命令行对外契约（Picocli，`oryxos-cli`）
**版本**：v1（本特性） | **范围**：本特性只交付 `provider` 命令组；`chat` / `serve` 属后续能力

## 命令

| 命令 | 作用 | 是否发外部请求 |
|------|------|---------------|
| `oryxos provider list` | 列出注册表所有条目 + 凭证就绪情况 + 运行态势 | 否 |
| `oryxos provider show <name>` | 单条目详情（含生效参数与态势明细） | 否 |
| `oryxos provider invoke --name <name> [--prompt <text>] [--temperature <t>]` | 发起一次真实调用（冒烟/联调） | 是 |

**退出码**：`0` 成功；`1` 调用失败（失败类别打到 stderr）；`2` 参数或注册表错误（启动期已拒绝）。`provider list/show` 不因"某条目不健康"而返回非 0。

## `provider list` 输出契约

固定列（无 TTY 时为纯文本，便于 CI 断言）：

```
NAME             VENDOR      MODEL              CREDENTIAL           READY  LAST_OK               LAST_FAIL             FAILS
deepseek-chat    deepseek    deepseek-chat      DEEPSEEK_API_KEY     yes    2026-10-07T10:00:12Z  -                     0
qwen-plus        dashscope   qwen-plus          DASHSCOPE_API_KEY    no     -                     2026-10-07T10:05:03Z  3
ops-fast         deepseek    deepseek-chat      DEEPSEEK_API_KEY     yes    2026-10-07T09:58:44Z  -                     0
```

| 列 | 语义 |
|----|------|
| `NAME` | 注册表条目名（Agent 引用的键） |
| `VENDOR` / `MODEL` | 实现细节，**仅在此处对运维可见** |
| `CREDENTIAL` | 环境变量**名**（永不打印值） |
| `READY` | 该环境变量当前是否已设置且非空（`yes`/`no`） |
| `LAST_OK` / `LAST_FAIL` | 最近成功/失败时间；无记录 `-` |
| `FAILS` | 连续失败次数；`windowSaturated` 时显示 `≥100` |

**输出禁忌**：任何命令都不得打印凭证值、完整请求体或完整响应体；`--verbose` 亦然（SC-007）。

## `provider invoke` 输出契约

- 成功：stdout 打印模型最终文本，stderr 打印一行结构化摘要 `provider=… model=… duration_ms=… tokens=prompt/completion/total（未知显示 unknown）`。
- 失败：stdout 为空，stderr 打印 `FAILED category=<FailureCategory> provider=… model=… reason=<摘要>`，退出码 `1`。

## 与审计的关系

- 每次 `provider invoke`（含失败）产生 1 条 `llm_calls` 记录，`session_id` 为空、`trace_id` 由 CLI 生成。
- `provider list/show` **不写**审计，也**不**改变态势数据。
