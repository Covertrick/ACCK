# ACCK 已定决定

## 目的

把已经写进规格的决定收成一张表，并标明哪些另有 ADR。这里不再记录未关闭的争论。

## 读者

改规格前要确认「这件事有没有定过」的人。

## 范围

只列出当前 `docs/FEATURES.md`、`docs/DATABASE.md` 和 `docs/HARNESS.md` 里生效的决定。一致性审查的原稿已不在仓库里，不要再恢复那份对照稿。

## 内容

下表是全部已定决定。其中影响骨架的几条另有 ADR：

| 决定 | ADR |
|------|-----|
| A1、A7 | `docs/ADR/002-gate-order.md` |
| B16 | `docs/ADR/001-postgres-row-lock.md` |
| B12、B13、B15 | `docs/ADR/003-strict-replay-input-hash.md` |
| 需求中的不做 Kubernetes | `docs/ADR/004-no-k8s-in-this-version.md` |
| B17，以及插件不能改检查顺序 | `docs/ADR/005-plugin-after-gates.md` |
| B19、B24、B28 | `docs/ADR/006-llm-outside-kernel.md` |

| 编号 | 决定 |
|------|------|
| A1 | 检查顺序是功能文档第 3 节的 11 步。预算和效应引用先于版本策略 |
| A2 | `contract_only` 只做结构、路径、文档约束和账本 |
| A3 | `no_kernel` 整份写回 Agent 读到的旧订单，不进 `Kernel.commit` |
| A4 | `evidence` 存账本。幂等比较只看 ops 哈希 |
| A5 | 写类工具的 `trace_id` / `span_id` 写在 `effects` |
| A6 | 幂等冲突不插入第二行账本 |
| A7 | 动作前用 `>=` 判断预算，含截止时间 |
| A8 | `READ` 不写 `effects`，每次真实调用 |
| A9 | 不保存效应状态 `rejected` |
| A10 | `tools/list` 返回全部四个工具 |
| A11 | 扣款去重在效应记录。桩每次被调用都扣。`double_charge` 四次调用，无内核扣 4 次，全内核扣 2 次 |
| A12 | `same_field` 用 `abort`。重规划上限只在 `tests/test_occ.py` |
| B1 | 订单初值见功能文档第 2 节，`order_id` 等于 `doc_id` |
| B2 | 状态只允许四条迁移。相同状态重写是 `SCHEMA`。不要求 `paid` 已有 `payment_id` |
| B3 | 角色路径见功能文档第 2 节。不在可写列表即拒绝。由 `DocumentSpec.paths` 提供 |
| B4 | 路径按分段匹配，`/note` 不匹配 `/notebook` |
| B5 | 除禁止空指针和带 `value` 的 `remove` 外，遵循 RFC 6901 与 RFC 6902。`test` 用 canonical JSON |
| B6 | `total` 是非布尔数字。`items` 只要求是数组。`payment_id` 为字符串或 `null` |
| B7 | 文档约束注册名 `order`。新任务 `LIVE`、`running`。同一 `doc_id` 再创建为 422。未注册名字为 422。没有 `completed` |
| B8 | 提交体是 `PatchIntent`。策略来自任务。`base_version < 0` 为 `SCHEMA`。大于当前版本走 `OCC`。Agent 未绑定为 `UNAUTHORIZED_PATH` |
| B9 | 四个工具的字段见开发文档第 2 节。用显式字段检查，不用通用 Schema 求值器 |
| B10 | 支付桩和发送计数在进程内，不入库 |
| B11 | 补偿不占预算。失败则效应保持 `executed`，响应 `EFFECT` |
| B12 | 第一条轨迹的 `step_id` 是 1 |
| B13 | 除幂等命中外，拒绝的提交也写 `kind=commit` |
| B14 | 恢复删除 `step_id >= checkpoint.cassette_next`。无检查点为 404 |
| B15 | 重放不改 `tasks.mode`，不改订单。成功体含 `reason_code` 和 `steps` |
| B16 | `READ COMMITTED`。幂等唯一约束冲突后再读已提交行 |
| B17 | 同名插件第二次注册失败，不覆盖 |
| B18 | MCP 与 API 同一进程 |
| B19 | 协作层的模型输出是决定 JSON：`thought`、`action`、`utterance`、`claims`、`tool`、`commit`。`order_v1` 的 `script` 不进入协作循环 |
| B20 | `success` 表示期望成立。`silent_corruption` 只比较订单。`ModelCascade` 不是内核码 |
| B21 | `disjoint_field`：规划者写 `/assignee`，收银写 `/note` |
| B22 | 账本按 `created_at`、`id` 升序。查询字段见 `docs/API.md` |
| B23 | Python 3.11+、FastAPI、Pydantic v2、PostgreSQL 16、OpenTelemetry 内存 exporter。API 端口 8000 |
| B24 | LangGraph 只按角色启动协作循环。规划者直到 `handoff` 或 `stop`，然后收银直到 `stop`。仍可重规划的 `OCC` 由模型重写 patch |
| B25 | 工具成功体含 `reason_code` 与 `result`。`payment_id` 为 `pay_` 加幂等键。422 与 404 的体是 `{"detail":"..."}`。MCP 不包 `jsonrpc` 和 `id` |
| B26 | 不写 `kind=input`。不配置连接池容量。本地提交目标 p50 小于 50 毫秒、p95 小于 200 毫秒，超出只写 `latency_note` |
| B27 | 共同目标在 `tasks.goal`，角色目标在 `role_bindings.goal`。交接句只在 `kind=llm` 的 `output` 里 |
| B28 | 模型不进入 `Kernel.commit`。`order_v1` 与内核单测用脚本。演示和 `agent_v1` 在 `ACCK_LLM=live` 时用真实模型。STRICT 不调用模型 |
