# ACCK 开发文档

| 字段 | 内容 |
|------|------|
| 文档版本 | v1.0 |
| 更新日期 | 2026-09-25 |
| 行为 | `docs/FEATURES.md` |
| 数据 | `docs/DATABASE.md` |
| 评测 | `docs/HARNESS.md` |

行为以功能文档为准。本文只写如何实现那一份行为。

---

## 1. 技术与布局

Python 3.11+、FastAPI、Pydantic v2、PostgreSQL 16、OpenTelemetry SDK 内存 exporter。隔离级别用 Postgres 默认的 READ COMMITTED。

`docker-compose.yml`：Postgres 映射 5432，API 映射 8000，进程为 `uvicorn acck.api.app:app`。

```text
src/acck/
  models.py
  errors.py
  canonical.py
  plugins/registry.py
  plugins/protocols.py
  kernel/patch.py
  kernel/contract.py
  kernel/occ.py
  kernel/effect.py
  kernel/budget.py
  kernel/commit.py
  store/schema.sql
  store/db.py
  store/state.py
  store/ledger.py
  store/checkpoint.py
  store/cassette.py
  store/effects.py
  replay/recorder.py
  replay/player.py
  telemetry.py
  api/app.py
  api/routes.py
  api/mcp.py
  runtime/langgraph_adapter.py
tools/
harness/run.py
harness/suites/order_v1.yaml
demo/story.py
tests/
```

`kernel` 不 import LangGraph、FastAPI、MCP、`tools/`。`api/app.py` 启动时注册插件，并挂上 `POST /mcp`。不另起 MCP 进程。`Kernel.commit` 是订单的唯一写入口。`no_kernel` 不调用它。组件图见 `docs/ARCHITECTURE.md`。`store/schema.sql` 与 `docs/DATABASE.md` 一致。

---

## 2. 扩展接口

```python
class ToolPlugin(Protocol):
    contract: ToolContract  # name, description, effect_class, fields, compensation, requires_approval
    def invoke(self, args: dict, ctx: InvokeCtx) -> dict: ...

class ConflictPolicyPlugin(Protocol):
    name: str
    def on_conflict(self, ctx: ConflictCtx) -> Literal["reject", "replan", "rebase"]: ...

class DocumentSpec(Protocol):
    name: str
    def initial(self, doc_id: str) -> dict: ...
    def paths(self, role: str) -> tuple[list[str], list[str]]: ...  # allow, deny
    def check(self, before: dict, after: dict) -> None: ...  # 失败抛 SchemaError

class LlmClient(Protocol):
    name: str
    def complete(self, messages: list, prompt_hash: str) -> LlmOutput: ...

class OrchestratorAdapter(Protocol):
    name: str
    def run(self, task_id: str, kernel: Kernel, llm: LlmClient) -> None: ...
```

`InvokeCtx`：`task_id`、`agent_id`、`idempotency_key`、`approval_token`、`mode`。`ConflictCtx`：`base_version`、`current_version`、本次路径、已提交路径、`replan_used`、`replan_limit`。`LlmOutput`：`text`、`token_count`。`SchemaError` 只带消息。

`Registry` 按名字保存五类。同名再次 `register` 抛错，不覆盖。查不到则调用方按功能文档返回 422 或 `EFFECT`。

| 返回值 | 内核 |
|--------|------|
| `reject` | 账本 `OCC`，任务保持 `running` |
| `replan` | `replan_used += 1`。未超限则账本 `OCC` 并带回当前文档。超限则 `aborted`，码仍为 `OCC` |
| `rebase` | `occ.disjoint` 为真则把 `base_version` 视为当前版本并继续。否则与 `reject` 相同 |

`abort` 返回 `reject`，`refresh_and_replan` 返回 `replan`，`merge_if_disjoint` 返回 `rebase`。不相交：两条指针互不为前缀。历史路径来自该文档 `decision=commit` 且 `result_version` 在 `(base_version, current]` 的 `ledger.ops`。

`fields` 是显式字段表，不是通用 JSON Schema 求值器。本版四个工具：

| name | 类 | 必填字段 | compensation | requires_approval | description |
|------|----|----------|--------------|-------------------|-------------|
| `read_order` | `READ` | `doc_id: str` | 无 | 否 | 读取当前订单 |
| `place_order` | `NON_IDEMPOTENT_WRITE` | `doc_id: str` | `refund` | 否 | 按当前 total 扣款一次 |
| `refund` | `IDEMPOTENT_WRITE` | `payment_id: str` | 无 | 否 | 按支付单号退款 |
| `send_receipt` | `IRREVERSIBLE` | `doc_id: str` | 无 | 是 | 发送回执 |

支付桩在进程内：`payment_id -> charged | refunded`，另有一个发送计数。`place_order` 每次被调用都把计数加 1，并返回 `pay_` 加幂等键。`refund` 对同一 `payment_id` 只从 `charged` 改为 `refunded` 一次。桩不入库。

未知角色在 `paths` 中不存在。创建任务时因此 422。

---

## 3. 哈希

`content_hash`、`ops_hash`、cassette `input_hash`：UTF-8 canonical JSON（`sort_keys=True`，`separators=(",", ":")`，`ensure_ascii=False`）的 SHA-256 十六进制。`ops_hash` 只哈希 ops 数组。

路径命中：相等，或候选路径以「规则 + `/`」开头。先查 deny，再查 allow。

---

## 4. 提交用到的表

表、索引、隔离级别和初始化见 `docs/DATABASE.md`。一次提交会读写 `documents`、`tasks`、`role_bindings`、`ledger`、`effects`、`checkpoints`、`cassette`。

---

## 5. 提交

`Kernel.commit` 在一个事务内，顺序与功能文档第 3 节相同。`no_kernel` 不进入本函数：用该 Agent 上次读到的整份文档改一个字段后覆盖 `documents.content`，不写拒绝账本。

1. 结构不合法 → 若任务存在则账本 `SCHEMA` 并提交事务；任务不存在则 404，不写账本。
2. `status != running` → 账本 `TASK_ABORTED`。
3. 已有相同 `(doc_id, agent_id, idempotency_key)`：`ops_hash` 相同则返回原结果。不同则返回 `IDEMPOTENCY_MISMATCH`，不插入第二行，追加 `kind=commit` 的 cassette。插入撞上唯一约束时，读已提交行再套用这两句。
4. `SELECT documents … FOR UPDATE`。
5. `intent.doc_id` 不等于任务文档 → `SCHEMA`。Agent 无绑定或路径未通过 → `UNAUTHORIZED_PATH`。
6. `token_used >= token_limit` 或 `tool_calls_used >= tool_call_limit` 或 `now() >= deadline_at` → `status=aborted`，账本 `BUDGET`，然后按第 6 节补偿。
7. `effect_ids` 有任一不是本任务的 `executed` → `EFFECT`。
8. `base_version == version` 则到步骤 9。否则调用策略。`base_version > version` 同样调用策略。
9. 内存中按 RFC 6902 应用。失败 `SCHEMA`。
10. `DocumentSpec.check`。失败 `SCHEMA`。
11. 成功：`version += 1`，更新文档，账本 `OK`（含 `evidence` 与当前 span），引用效应改为 `committed`，`checkpoints.seq = version`，`cassette_next` 取追加本次 `commit` 步之后的值。
12. 其他拒绝：文档不变，追加账本和 `commit` 轨迹，然后提交事务。幂等命中不进入这里。

分配轨迹号：`step_id = cassette_next`，然后 `cassette_next += 1`。第一条是 1。状态图见 `docs/ARCHITECTURE.md`。

---

## 6. 工具、补偿、重放

`invoke_tool`：

1. 未注册 → `EFFECT`。字段不符 → `SCHEMA`。都不写 `effects`。
2. 预算超限 → `aborted`，不调用插件，不写账本，然后补偿。
3. `requires_approval` 且无令牌 → `APPROVAL_REQUIRED`。`contract_only` 与 `no_kernel` 跳过这一步。
4. `STRICT`：不调用插件。按 `(kind=tool, name, input_hash)` 取 cassette。对不上 → `REPLAY_DIVERGENCE`。
5. `READ`：总是调用插件，只追加 cassette，不写 `effects`，成功后 `tool_calls_used += 1`。
6. 写类无键 → `EFFECT`。已有相同 `(task, tool, idempotency_key)` → 返回记录结果，不调用插件。否则调用插件，插入 `executed`，写 cassette 与 span，次数加 1。
7. `RESUME`：`step_id < checkpoint.cassette_next` 的步骤只读 cassette，之后按 `LIVE`。

`compensate`：只处理 `executed` 且带 compensation 的行。不检查预算，不增加 `tool_calls_used`。失败则该行保持 `executed` 并返回 `EFFECT`。

`no_kernel` 直接调用插件，不读 `effects`。`contract_only` 调用插件但不做同键短路，也不写补偿所需的去重。

重放：不改 `tasks.mode`，不改文档。成功体为 `{"reason_code":"OK","steps":N}`。失败体含 `step_id`、`kind`、`expected_hash`、`actual_hash`。

恢复：无检查点则 404。否则写入 checkpoint 上的版本与预算，`DELETE FROM cassette WHERE task_id=$1 AND step_id >= checkpoint.cassette_next`，`mode=RESUME`，`status=running`。

`recorded_llm`：允许调用时才调 `LlmClient`，返回后把 `token_count` 加到 `token_used`。调用前若已 `token_used >= token_limit` 则 `BUDGET`。效应状态图见 `docs/ARCHITECTURE.md`。

---

## 7. HTTP

创建任务：

```json
{
  "task_id": "t1",
  "doc_id": "ord_demo",
  "document": "order",
  "conflict_policy": "abort",
  "replan_limit": 3,
  "token_limit": 2000,
  "tool_call_limit": 20,
  "wall_clock_seconds": 60,
  "agents": [
    {"agent_id": "a1", "role": "planner"},
    {"agent_id": "a2", "role": "cashier"}
  ]
}
```

`deadline_at = now() UTC + wall_clock_seconds`。`document`、`conflict_policy`、`role` 未注册，或 `doc_id` 已有任务，均为 422。`mode=LIVE`，`status=running`，文档用 `DocumentSpec.initial(doc_id)`。

`POST /v1/commits` 的体就是 `PatchIntent`。`POST /v1/tools/invoke` 的体见功能文档第 7 节。

---

## 8. 测试与顺序

套件字段、三种模式和报告格式见 `docs/HARNESS.md`。

| 文件 | 锁住的行为 |
|------|------------|
| `tests/test_plugins.py` | 假工具、假策略不改 `commit.py`；同名注册失败 |
| `tests/test_patch.py` | 四种 op；空指针与带 value 的 `remove` 拒绝；`test` 失败不改副本 |
| `tests/test_contract.py` | deny 优先；分段前缀；`/note` 不命中 `/notebook` |
| `tests/test_commit.py` | version+1；拒绝不改文档但仍有账本；幂等命中不双写；冲突不插第二行 |
| `tests/test_occ.py` | 并发只有一个成功；rebase 仅在不相交时成功；replan 超限 `aborted` |
| `tests/test_effect.py` | 未注册拒绝；同键一次扣款；`READ` 不写 effects；补偿不打已 commit 行；无令牌不发送 |
| `tests/test_budget.py` | `tool_call_limit=1` 时第二次不执行 |
| `tests/test_replay.py` | 原样重放；篡改第 2 步停在 `step_id=2` |
| `tests/test_resume.py` | 版本、预算、游标；无检查点 404 |
| `tests/test_otel.py` | 账本与效应的 `trace_id` 等于对应 span |
| `tests/test_mcp.py` | 列表含四个工具名；`place_order` 产生 effect 与 cassette |
| `tests/test_api.py` | 422、404 与业务 reject |

运行：`python -m unittest discover -s tests`。

实现顺序：骨架与注册表 → 提交、权限、幂等 → 三种 OCC → 工具与补偿 → 预算 → cassette、STRICT、resume → HTTP → OTel → LangGraph 与 `demo/story.py` → MCP → Harness。前一步未绿，不开始下一步。
