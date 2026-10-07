# Quickstart Validation: 核心能力一 · 对接 LLM（Provider 抽象）

**Feature**: `001-llm-provider-abstraction` | **Date**: 2026-10-07
**目的**：不依赖任何真实厂商账号与公网，用本地桩端点在 15 分钟内端到端验证本特性全部验收项（SC-001 ~ SC-010）。

> 本文件是**验证指南**，不是实现文档。契约细节见 [contracts/](./contracts/)，数据形态见 [data-model.md](./data-model.md)。

---

## 0. 前置条件

| 项 | 要求 | 本环境实测 |
|----|------|-----------|
| JDK | 21（`JAVA_HOME` 指向 Temurin 21） | ✅ `/Library/Java/JavaVirtualMachines/temurin-21.jdk/Contents/Home` |
| Maven | 3.9+ **且** `JAVA_HOME=jdk21`（否则 mvn 跑在 JDK 27 上） | ✅ Maven 3.10.0（默认 JDK 27，需显式覆盖） |
| 依赖仓库 | 工作区内 `.m2repo`（沙箱可写）+ 阿里云镜像 | ⚠️ `~/.m2` 在沙箱下不可写（实测），已由 `.mvn/maven.config` 覆盖 |
| 网络 | 首次构建需下载依赖 | ✅ `repo1.maven.org` 实测可达（`repo.maven.apache.org` 不可达），**无需放宽沙箱权限** |
| 真实 API key | **不需要**（全部用桩端点；凭证缺失路径本身是验收项） | ✅ |

```bash
export JAVA_HOME=/Library/Java/JavaVirtualMachines/temurin-21.jdk/Contents/Home
export PATH="$JAVA_HOME/bin:$PATH"
cd <repo-root>
mvn -q -DskipTests package        # 首次需网络；仓库/镜像已在 .mvn/maven.config 指定
```

---

## 1. 启动桩端点（验收用假厂商）

`tools/stub-llm-server/StubLlmServer.java` 是一个 JDK 21 单文件程序（零依赖，`HttpServer`），模拟 OpenAI 兼容协议：

```bash
# 三个身份可区分的桩：A / B / C（用于路由归因）
java tools/stub-llm-server/StubLlmServer.java --port 18081 --id A &
java tools/stub-llm-server/StubLlmServer.java --port 18082 --id B &
java tools/stub-llm-server/StubLlmServer.java --port 18083 --id C &
```

| 参数 | 作用 |
|------|------|
| `--id <X>` | 桩身份；成功响应的文本里带 `[stub-X]`，用于**归因路由** |
| `--shape openai\|dashscope` | 模拟的协议形态（默认 `openai`）；`dashscope` 用于覆盖第二个协议适配器 |
| `--mode text\|tool\|401\|429\|404\|400\|500\|badjson` | 返回形态（默认 `text`） |
| `--slow <ms>` | 延迟响应（验证超时） |
| `--no-usage` | 响应中省略 `usage`（验证"未知 token"） |
| `--with-tools` | 以工具调用形态响应（验证"只下发、不执行"） |
| `/stats` | `GET` 返回 `{"id":"A","requests":N}`（验证"零流量"） |
| `/healthz` | 存活探针 |

> S1~S8 的路由与故障场景统一用 `--shape openai`；`--shape dashscope` 用于自动化测试中覆盖 DashScope 原生适配器（见 `ProviderProtocolAdapterTest`）。

---

## 2. 场景 S1：零凭证启动（FR-020 / SC-009）

```bash
mkdir -p .oryxos
cat > .oryxos/providers.yaml <<'YAML'
providers:
  - name: a-chat
    protocol: openai-compatible
    vendor: deepseek
    model: deepseek-chat
    api_key: ${DEEPSEEK_API_KEY}
    base_url: http://127.0.0.1:18081
  - name: b-chat
    protocol: openai-compatible
    vendor: moonshot
    model: moonshot-v1-32k
    api_key: ${KIMI_API_KEY}
    base_url: http://127.0.0.1:18082
  - name: c-chat
    protocol: openai-compatible
    vendor: dashscope
    model: qwen-plus
    api_key: ${DASHSCOPE_API_KEY}
    base_url: http://127.0.0.1:18083
YAML
unset DEEPSEEK_API_KEY KIMI_API_KEY DASHSCOPE_API_KEY     # 零凭证

java -jar oryxos-boot/target/oryxos-boot-*-exec.jar
```

**预期**：进程**正常启动**、无 ERROR、无外部请求；`oryxos provider list` 中三个条目 `READY=no`、`FAILS=0`。
**反向断言**：把凭证写成字面量（`api_key: sk-xxx`）→ 启动**失败**并提示改用 `${ENV_VAR}`。

---

## 3. 场景 S2：多 Provider 显式路由（US1 / SC-001）

```bash
export DEEPSEEK_API_KEY=k1 KIMI_API_KEY=k2 DASHSCOPE_API_KEY=k3

oryxos provider invoke --name a-chat --prompt "ping"     # 期望 stdout 含 [stub-A]
oryxos provider invoke --name b-chat --prompt "ping"     # 期望 stdout 含 [stub-B]
oryxos provider invoke --name c-chat --prompt "ping"     # 期望 stdout 含 [stub-C]

curl -s localhost:18082/stats    # 期望 {"id":"B","requests":1}（B 只被自己的调用打到）
curl -s localhost:18083/stats    # 期望 {"id":"C","requests":1}
```

**交叉验证（SC-001 的 10 次交叉调用）**：

```bash
for i in $(seq 1 4); do
  for n in a-chat b-chat c-chat; do oryxos provider invoke --name "$n" --prompt "ping" >/dev/null; done
done
for p in 18081 18082 18083; do curl -s localhost:$p/stats; echo; done
```

**预期**：每个桩 `requests` 恰好等于打到它的调用次数（4+1=5），**零错投**。

**Agent 只持有名称（US1.2）**：

```bash
mkdir -p .oryxos/agents/ops-agent
cat > .oryxos/agents/ops-agent/AGENT.md <<'MD'
---
name: ops-agent
provider:
  name: a-chat
  temperature: 0.3
---
你是运维助手。
MD
```

**预期**：`oryxos provider show a-chat` 可见条目参数；Agent 定义中**不出现** model / api_key / base_url（契约 [agent-md-provider-reference.md](./contracts/agent-md-provider-reference.md)）。

---

## 4. 场景 S3：失败类别与"不静默切换"（US3 / SC-004）

按行执行，观察 stderr 的 `category=` 与退出码：

| 命令 | 期望类别 | 退出码 | 附加断言 |
|------|---------|-------|---------|
| `unset DEEPSEEK_API_KEY; oryxos provider invoke --name a-chat -p ping` | `CREDENTIAL_MISSING` | 1 | 桩 A `/stats` **请求数不变**（未外呼） |
| `oryxos provider invoke --name ghost -p ping` | `PROVIDER_NOT_REGISTERED` | 1 | 未外呼 |
| 重启桩 A 为 `--mode 401` 后调用 | `AUTH_FAILED` | 1 | — |
| `--mode 429` | `RATE_LIMITED` | 1 | — |
| `--mode 404` | `MODEL_NOT_FOUND` | 1 | — |
| `--mode 400` | `BAD_REQUEST` | 1 | — |
| `--mode badjson` | `MALFORMED_RESPONSE` | 1 | 进程不崩、后续调用仍可用 |
| `--slow 5000` + 条目 `timeout_ms: 1000` | `TIMEOUT` | 1 | 返回耗时 ≈ 1s（SC-008） |
| 杀掉桩 A 后调用 | `UNREACHABLE` | 1 | 桩 B/C **零新增请求**（无 fallback，FR-015） |

---

## 5. 场景 S4：审计与态势（US4 / SC-003 / SC-010）

```bash
oryxos provider list
# 期望：a-chat 的 FAILS 与刚刚连续失败次数一致；LAST_FAIL 有值、LAST_OK 为最近一次成功时间

# 可选：直接核对审计表（若本机有 sqlite3）
sqlite3 .oryxos/oryxos.db \
  "select provider, model, success, error_category, prompt_tokens, total_tokens, duration_ms
     from llm_calls order by id desc limit 10;"
```

**断言**：
- 上述每一次 `invoke`（成功与失败）**各留一行**（SC-003）。
- `--no-usage` 桩返回的调用，三列 token 均为 **NULL**（不是 0）。
- 连续 3 次失败后 `FAILS=3`，成功一次后 `FAILS=0`。
- 任意一行都不含测试密钥明文（`grep -c 'k1\|k2\|k3' .oryxos/oryxos.db` → 0，SC-007）。

---

## 6. 场景 S5：弱依赖不扩散（SC-009）

```bash
# 桩 A 已停止（不可达）、B/C 正常 → 重启应用
java -jar oryxos-boot/target/oryxos-boot-*-exec.jar &
oryxos provider invoke --name b-chat --prompt ping     # 期望成功（[stub-B]）
```

**预期**：启动成功；B/C 调用成功率与全健康时一致；`provider list` 中 a-chat 显示最近失败为 `UNREACHABLE`。

---

## 7. 场景 S6：Agent 引用校验不阻塞启动（FR-006）

```bash
mkdir -p .oryxos/agents/ghost-agent
cat > .oryxos/agents/ghost-agent/AGENT.md <<'MD'
---
name: ghost-agent
provider:
  name: not-registered
---
MD
```

**预期**：启动日志出现 **WARN**（含 `ghost-agent` 与 `not-registered`），进程**照常启动**；`ops-agent` 不受影响。
**反向断言**：`ghost-agent` 的调用在任何外部请求之前以 `PROVIDER_NOT_REGISTERED` 失败。

---

## 8. 场景 S7：工具定义只下发、不执行（US2 / SC-005）

```bash
java tools/stub-llm-server/StubLlmServer.java --port 18084 --id T --mode tool &
# 注册一个条目 t-chat 指向 18084
oryxos provider invoke --name t-chat --prompt "call the echo tool" --with-demo-tool
```

**预期**：stdout 打印桩请求的工具调用（工具名 + 参数）作为**结果**交回，而不是被执行的结果；桩 `/stats` 显示仅 1 次请求（无二次回灌）；审计新增 1 行。
**自动化对照**：同一断言在契约测试中固化（`ProviderServiceContractTest`），quickstart 只是人工可复核的等价路径。

---

## 9. 场景 S8：并发无串扰（SC-006）

```bash
seq 1 20 | xargs -P 20 -I{} sh -c 'oryxos provider invoke --name a-chat --prompt "ping-{}" >/dev/null'
curl -s localhost:18081/stats    # 期望 requests = 20（全部落在 a-chat）
```

**预期**：20 并发全部成功、响应身份一致（全为 `[stub-A]`）、无凭证混用、无错投；`provider list` 的 `FAILS=0`。

---

## 10. 一键回归（把上面场景交给自动化）

```bash
mvn verify     # 单元 + 契约 + 集成（含桩端点场景 S1~S8 的自动化等价物）+ 质量门禁
```

| 门禁 | 命令 | 说明 |
|------|------|------|
| 单元/契约/集成 | `mvn test` | 零凭证、路由、11 类失败、审计、态势、并发、工具不执行 |
| 格式 | `mvn spotless:check` | Google 格式 |
| 静态分析 | `mvn pmd:check checkstyle:check spotbugs:check` | P3C + Checkstyle + FindSecBugs |
| 依赖漏洞 | `mvn -Psecurity-scan verify` | OWASP Dependency-Check（首次需下载 NVD 数据，见风险 R1） |

**验收完成的判定**：S1~S8 全部符合预期，且 SC-001 ~ SC-010 均有对应的自动化断言通过。
