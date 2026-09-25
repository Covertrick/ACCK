# ACCK 功能

| 字段 | 内容 |
|------|------|
| 文档版本 | v1.0 |
| 更新日期 | 2026-09-25 |
| 范围 | `docs/REQUIREMENTS.md` |
| 实现 | `docs/DEVELOPMENT.md` |

本文是行为的唯一说明。表、算法步骤、YAML 键和测试文件不在这里重复。

---

## 1. 插件

流水线不能重排，也不能绕过 `Kernel.commit` 写订单。插件只替换门后面的实现。HTTP、进程内调用、MCP 使用同一进程里的同一份注册表。未注册的名字在创建任务时返回 422。同名第二次注册失败，不覆盖。

| 扩展点 | 本版 |
|--------|------|
| 工具 | `read_order`、`place_order`、`refund`、`send_receipt` |
| 冲突策略 | `abort`、`refresh_and_replan`、`merge_if_disjoint` |
| 文档约束 | `order`：初值、状态迁移、角色路径 |
| 编排适配器 | LangGraph。只按角色启动第 14 节的协作循环 |
| 模型 | 协作层调用。`order_v1` 与内核单测用脚本。演示和 `agent_v1` 在 `ACCK_LLM=live` 时用真实模型 |

模型不进入 `Kernel.commit`。决定 JSON、循环和交接见第 14 节。

---

## 2. 订单与任务

一个任务只写一份订单。同一 `doc_id` 已有任务时，再建任务返回 422。

初值（`version = 0`，`order_id` 等于 `doc_id`）：

```json
{
  "order_id": "ord_demo",
  "status": "draft",
  "assignee": "",
  "items": [],
  "note": "",
  "total": 0,
  "payment_id": null
}
```

`total` 是 JSON 数字，不能是布尔。`items` 是数组，不检查元素。`payment_id` 是字符串或 `null`。`""` 允许，与 `null` 不同。

状态只允许：`draft→confirmed`、`confirmed→paid`、`draft→cancelled`、`confirmed→cancelled`。`paid→cancelled` 和写成相同状态都是 `SCHEMA`。不要求 `paid` 时已有 `payment_id`。不按角色再限制迁移。

| 角色 | 可写 | 不可写 |
|------|------|--------|
| `planner` | `/items`、`/note`、`/assignee`、`/total`、`/status` | `/order_id`、`/payment_id` |
| `cashier` | `/payment_id`、`/status`、`/note` | `/order_id`、`/items`、`/total`、`/assignee` |

不在可写列表中即拒绝。禁止优先于可写。按路径分段匹配：规则等于路径，或路径以「规则 + `/`」开头。`/note` 不匹配 `/notebook`。`/items/0` 匹配 `/items`。

新建任务为 `running`、`LIVE`。截止时间是创建时刻 UTC 加上 `wall_clock_seconds`。任务状态只有 `running` 和 `aborted`。

---

## 3. 提交

Agent 提交 `PatchIntent`：`task_id`、`doc_id`、`agent_id`、`base_version`、`idempotency_key`、1 到 32 条 op、`evidence`、`effect_ids`。冲突策略用任务上保存的名字，不在请求里再传。op 只支持 `add`、`remove`、`replace`、`test`。空指针 `""` 拒绝。`remove` 带 `value` 拒绝。其余语义遵循 RFC 6902 与 RFC 6901。`test` 用与内容哈希相同的 canonical JSON 比较。

`doc_id` 必须等于任务上的文档，否则 `SCHEMA`。`base_version < 0` 为 `SCHEMA`。`base_version` 大于当前版本时按版本落后处理，原因码仍是 `OCC`。

检查顺序：

1. 结构。任务不存在则 404，不写账本。
2. 任务不是 `running` → `TASK_ABORTED`。
3. 幂等。见下一节。
4. 锁住订单行。
5. 路径权限。失败 `UNAUTHORIZED_PATH`。Agent 未绑定也是这个码。
6. 预算。失败 `BUDGET`，任务变为 `aborted`。
7. `effect_ids` 必须都属于本任务且为已执行。否则 `EFFECT`。
8. 版本。相等则继续。否则按冲突策略。
9. 在内存副本上应用 patch。失败 `SCHEMA`。
10. 文档约束。失败 `SCHEMA`。
11. 通过则版本加 1，写入订单、账本、检查点，并把引用的效应标为已提交。

任一步拒绝：订单与拒绝前相同。除幂等两种情况外，都追加账本。`evidence` 原样存进账本，内核不解释。

成功响应：`decision=commit`、`reason_code=OK`、`version`、`content_hash`。拒绝响应：`decision=reject`、原因码、当前 `version`。重规划还带当前文档和 `replan_remaining`（上限减已用次数）。

业务拒绝 HTTP 200。请求体不合法 HTTP 422。任务或订单不存在，以及没有检查点时的恢复，HTTP 404。

---

## 4. 幂等

比较的是 ops 的哈希，不比较 `evidence` 和 `effect_ids`。

- 同一订单、同一 Agent、同一键、同一 ops 哈希：返回第一次的结果。不追加账本，不追加轨迹。
- 同一键但 ops 哈希不同：响应 `IDEMPOTENCY_MISMATCH`。不插入第二行账本。仍追加一条提交轨迹。

两个相同键同时到达时，后写入的一方读已提交的那一行，再按上面两条处理。

---

## 5. 版本冲突

| 策略 | 版本落后时 |
|------|------------|
| `abort` | `OCC`。任务仍是 `running` |
| `refresh_and_replan` | 未达上限：`OCC`，交回当前订单，重规划次数加 1。达到上限：任务 `aborted`，码仍是 `OCC` |
| `merge_if_disjoint` | 落后期间已提交的路径与本次路径互不为前缀则按当前版本继续并成功。相交则与 `abort` 相同 |

`same_field` 使用 `abort`。`disjoint_field` 使用 `merge_if_disjoint`：规划者写 `/assignee`，收银写 `/note`。重规划达到上限只在 `tests/test_occ.py` 里测，不单列评测用例。

---

## 6. 账本

提交尝试追加一行。幂等命中不追加。幂等冲突不追加第二行。行内有任务、订单、Agent、键、ops 及其哈希、`evidence`、基于的版本、结果版本、决定、原因码、前后内容哈希、效应编号、`trace_id`、`span_id`、时间。

查询按 `created_at` 升序，相同则按 `id` 升序。不相交合并用已成功提交的 ops 判断改过哪些路径。

---

## 7. 工具

未注册为 `EFFECT`。参数不符合该工具的字段要求为 `SCHEMA`。写类没有幂等键为 `EFFECT`，且不产生真实副作用。工具调用不改订单行。门拒绝不写效应行，只返回原因码。效应没有 `rejected` 状态。

| 类 | 规则 |
|----|------|
| `READ` | 可以不带键。每次真实调用，只记轨迹，不写效应表 |
| `IDEMPOTENT_WRITE` | 必须带键。同一键只执行一次，再次返回已记录结果 |
| `NON_IDEMPOTENT_WRITE` | 必须带键。禁止第二次真实调用。未提交的成功调用可在任务中止时补偿 |
| `IRREVERSIBLE` | 无 `approval_token` 则 `APPROVAL_REQUIRED`，不执行。严格重放不真实调用 |

| 工具 | 类 | 参数 | 行为 |
|------|----|------|------|
| `read_order` | `READ` | `doc_id` 字符串 | 返回当前订单 |
| `place_order` | `NON_IDEMPOTENT_WRITE` | `doc_id` 字符串 | 按当时的 `total` 扣款一次。`payment_id` 为 `pay_` 加幂等键。补偿是 `refund` |
| `refund` | `IDEMPOTENT_WRITE` | `payment_id` 字符串 | 该支付单号只退一次。再退返回成功且不再改余额 |
| `send_receipt` | `IRREVERSIBLE` | `doc_id` 字符串 | 无批准令牌则不发送。令牌在调用参数外 |

扣款去重在内核的效应记录，不在支付桩。桩每次被调用都扣一次。`double_charge` 的四次调用在无内核都会进桩；全内核同键只进桩一次。

效应只保存三种状态：

| 状态 | 含义 |
|------|------|
| `executed` | 已真实调用，订单尚未采纳 |
| `committed` | 某笔成功提交引用了它 |
| `compensated` | 已补偿 |

任务变为 `aborted`，或显式调用补偿时：对仍是 `executed` 且注册了补偿的效应各调用一次，键为 `compensate:{effect_id}`。成功后变为 `compensated`。已 `committed` 的不补偿。补偿不占工具次数，也不因预算已超而跳过。补偿抛错则该行保持 `executed`，响应 `EFFECT`，任务仍是 `aborted`。

因此：扣款已发生但支付单号还没写入订单，任务中止后再退一次。已经随订单一并提交的扣款不自动退。

调用体：`task_id`、`agent_id`、`name`、`args`、`idempotency_key`、可选 `approval_token`。成功响应含 `reason_code` 和 `result`。`result` 是插件返回值。

---

## 8. 预算

动作前检查。`tool_calls_used >= tool_call_limit`，或 `token_used >= token_limit`，或当前时间大于等于截止时间，则 `BUDGET`。本次工具不执行，本次提交不改订单，任务变为 `aborted`。不在单次模型返回中途截断 token。模型调用在允许执行并且成功返回之后，才把本次 `token_count` 加进去。新的真实写类或读类调用成功后，工具次数加 1。

工具路径上的 `BUDGET` 不写账本。提交路径上的 `BUDGET` 写账本。之后的提交是 `TASK_ABORTED`。

---

## 9. 轨迹、重放、恢复

| 模式 | 行为 |
|------|------|
| `LIVE` | 真实执行，并写入轨迹 |
| `STRICT` | 不调用真实模型与工具。逐步对哈希。第一处不一致就停 |
| `RESUME` | 检查点之前只读轨迹；之后按 `LIVE` |

轨迹三类：`llm`（消息与提示词版本的哈希，以及决定 JSON）、`tool`（参数哈希与结果）、`commit`（意图哈希与提交结果）。任务目标存在 `tasks.goal`，角色目标存在 `role_bindings.goal`。不为此写 `kind=input`。`llm` 行由协作层在模型返回后写入。`tool` 与 `commit` 仍由工具路径和提交路径写入。

第一条的 `step_id` 是 1。除幂等命中外，每次提交尝试都追加 `commit`，包括拒绝和幂等冲突。

严格重放不修改任务模式，不改订单。成功响应 `reason_code=OK` 和已比较步数。失败响应 `REPLAY_DIVERGENCE`、`step_id`、`kind`、期望哈希、实际哈希。

每次成功提交写检查点：订单版本、下一可用 `step_id`、已用 token、已用工具次数。恢复时取序号最大的检查点，订单和预算回到该点，删掉 `step_id` 大于等于该游标的轨迹，模式改为 `RESUME`，状态回到 `running`。没有检查点则 404，任务不变。

崩溃点在提交已经成功返回、下一步还没开始。恢复后检查点前的 `place_order` 不会再执行。

---

## 10. 观测

有追踪上下文时，提交写入账本的 `trace_id` / `span_id`，工具的真实调用写入效应行的同名字段。`READ` 不写效应行，追踪留在该步轨迹上。

| 跨度 | 属性 |
|------|------|
| `acck.commit` | 任务、订单、Agent、决定、原因码、基于的版本 |
| `acck.tool` | 任务、工具名、效应类、原因码 |
| `acck.llm` | 任务、步骤号、token 数 |

没有外部收集器时，追踪留在内存里。

---

## 11. 接口

| 功能 | 接口 |
|------|------|
| 创建任务 | `POST /v1/tasks` |
| 提交 | `POST /v1/commits` |
| 调用工具 | `POST /v1/tools/invoke` |
| 补偿 | `POST /v1/tasks/{id}/compensate` |
| 恢复 | `POST /v1/tasks/{id}/resume` |
| 严格重放 | `POST /v1/tasks/{id}/replay` |
| 读订单 | `GET /v1/documents/{doc_id}` |
| 读账本 | `GET /v1/ledger?doc_id=` |
| 读任务 | `GET /v1/tasks/{id}` |
| MCP | `POST /mcp` |

订单响应：`doc_id`、`version`、`content`、`content_hash`。任务响应：状态、模式、策略、任务目标、token 与工具次数的已用和上限、截止时间、重规划已用和上限。

MCP 只实现 `tools/list` 和 `tools/call`。列表返回注册表里的全部工具（四个都在），含名字、说明和参数要求。`tools/call` 只转给工具入口。不做登录、OAuth、资源订阅。

---

## 12. 评测

内核套件 `order_v1` 不进入协作循环。Agent 套件 `agent_v1` 进入第 14 节。两种报告的字段见 `docs/HARNESS.md`。

---

## 13. 演示

`python demo/story.py` 跑协作演示。任务目标是「把订单确认为加急，总价 128，然后收款并出收据」。规划者把状态写成 `confirmed`、备注写成「加急」、总价写成 128，然后交接。收银用幂等键 `pay-1` 下单，把 `payment_id` 写成 `pay_pay-1`、状态写成 `paid`，再发送收据。最终订单等于 `docs/HARNESS.md` 里 `confirm_and_pay` 的标准结果，扣款次数为 1。脚本打印账本最后一行、最后一条交接，以及扣款次数。

`ACCK_LLM=live` 时调用真实模型。否则按 `demo/fixtures/story_decisions.json` 回放决定，不调用真实模型。

内核走查不经过协作层，六步命令和预期在 `docs/DEMO.md`。

---

## 14. 协作层

协作层决定下一轮动作。内核决定这个动作能不能写入共享订单。`kernel` 不调用模型，不解析自然语言，也不引用协作层。

`tasks.goal` 是交给两个角色的共同目标。`role_bindings.goal` 是该角色自己的目标。两者都可以是空字符串。空的共同目标表示这个任务不启动协作循环。`order_v1` 使用空目标，步骤直接提交或调用工具。

模型每轮只输出一个 JSON：

```json
{
  "thought": "只进入轨迹，内核不读",
  "action": "read",
  "utterance": "",
  "claims": {},
  "tool": null,
  "commit": null
}
```

`action` 只能是 `read`、`tool`、`commit`、`handoff`、`stop`。`tool` 含 `name`、`args`、`idempotency_key`。`commit` 是 `PatchIntent`。`claims` 可含 `total` 和 `status`。解析失败不改订单，该步原因码 `SCHEMA`，并把这段输出交回模型。

模型每轮看到：角色与可写路径、当前订单、共同目标、本角色目标、上一轮原因码、剩余 token 和工具次数、对方最后一条 `handoff` 的 `utterance` 与 `claims`。当前订单以数据库为准。

默认顺序：规划者直到 `handoff` 或 `stop`，或任务不再是 `running`；然后收银直到 `stop`，或任务不再是 `running`。`OCC` 且仍可重规划时，把当前订单和剩余重规划次数交给模型重写 patch。协作层不把上一份被拒绝的 ops 再提交一次。

同一 Agent 连续两轮给出相同 ops 哈希时，不再调用 `Kernel.commit`。该轮记为非法提议，并结束该角色的循环。`SCHEMA`、`UNAUTHORIZED_PATH`、`IDEMPOTENCY_MISMATCH`，以及仍可重规划的 `OCC`，都把原因交回模型再决定一次。

`send_receipt` 的批准令牌不由模型填写。当前订单 `status` 已是 `paid` 时，协作层附上 `ok-receipt`。否则原样调用，内核返回 `APPROVAL_REQUIRED`。

`handoff` 的 `utterance` 和 `claims` 写在该步 `kind=llm` 的 `output` 里，不写入订单。`claims.total` 或 `claims.status` 与交接当时的订单不一致，记为话和状态不一致。收银之后以订单为准。

`ACCK_LLM=live` 时，客户端使用 `ACCK_LLM_BASE_URL` 和 `ACCK_LLM_MODEL`，协议是 OpenAI 兼容的对话补全。`ACCK_LLM=script` 时，按套件中的 `decisions` 顺序弹出上面的 JSON。内核单测和 `order_v1` 不读这两个变量。STRICT 不调用模型，只回放已记下的 `output`。

`note_conflict` 和 `split_fields` 使用 `parallel_first_commit`：两个角色都先只看到版本 0，各产生一次 `commit`，再先提交规划者、后提交收银。其余 Agent 用例走默认顺序。标准结果见 `docs/HARNESS.md`。
