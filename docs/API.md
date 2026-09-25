# ACCK 接口

## 目的

把 REST 和 MCP 的路径、错误码和示例收在一处，便于对照 `docs/openapi.yaml`。

## 读者

调用 API 或写 `api/routes.py` 的人。

## 范围

只描述功能文档和开发文档已经定下的接口。没有健康检查、没有认证。OpenAPI 里因此也不写这两项。

## 内容

基址是 `http://127.0.0.1:8000`。业务拒绝是 HTTP 200 加 `decision=reject`。请求体不合法、未注册的文档约束、冲突策略、角色，以及同一 `doc_id` 再建任务，都是 HTTP 422。任务不存在、订单不存在、没有检查点时的恢复，是 HTTP 404，并且不写账本。

### 错误码

| 码 | HTTP | 何时 |
|----|------|------|
| `OK` | 200 | 提交成功，或重放成功，或工具按规则完成 |
| `SCHEMA` | 200 | 结构、patch、文档约束或工具字段不合法。任务必须已经存在 |
| `UNAUTHORIZED_PATH` | 200 | 路径不在角色可写范围内，或 Agent 未绑定 |
| `OCC` | 200 | 版本落后或 `base_version` 大于当前版本，且策略没有把这次提交变成成功 |
| `EFFECT` | 200 | 工具未注册、写类缺键、效应编号无效，或补偿失败 |
| `BUDGET` | 200 | 动作前已达到 token、工具次数或截止时间 |
| `APPROVAL_REQUIRED` | 200 | `send_receipt` 没有 `approval_token` |
| `REPLAY_DIVERGENCE` | 200 | STRICT 第一步对不上的 `input_hash` |
| `IDEMPOTENCY_MISMATCH` | 200 | 同一订单、同一 Agent、同一键，但 ops 哈希不同。不插入第二行账本 |
| `TASK_ABORTED` | 200 | 任务已经不是 `running` 时又来提交 |

`ModelCascade` 不是本接口的码。422 与 404 的响应体是 `{"detail": "<string>"}`。

`IDEMPOTENCY_MISMATCH` 与其他拒绝一样带当前 `version`。工具成功时插件返回值放在 `result`。`place_order` 的 `payment_id` 为 `pay_` 加上幂等键。

### 创建任务

`POST /v1/tasks`

```json
{
  "task_id": "t-story",
  "doc_id": "ord_demo",
  "document": "order",
  "conflict_policy": "abort",
  "replan_limit": 3,
  "token_limit": 2000,
  "tool_call_limit": 20,
  "wall_clock_seconds": 60,
  "goal": "把订单确认为加急，总价 128，然后收款并出收据",
  "agents": [
    {"agent_id": "a1", "role": "planner", "goal": "把状态改为 confirmed，备注写成加急，总价写成 128，然后交接"},
    {"agent_id": "a2", "role": "cashier", "goal": "用幂等键 pay-1 下单，把 payment_id 写成 pay_pay-1，状态写成 paid，再发送收据"}
  ]
}
```

`goal` 是共同目标，角色上的 `goal` 是该角色目标。省略时按空字符串保存。共同目标为空时，创建任务不会启动协作循环。

成功时订单为功能文档第 2 节的初值，`version` 为 0，任务为 `running`、`LIVE`。截止时间是创建时刻 UTC 加上 `wall_clock_seconds`。任务响应包含 `goal`。

### 提交

`POST /v1/commits`

冲突策略不在请求里传。体是 `PatchIntent`。

```json
{
  "task_id": "t-story",
  "doc_id": "ord_demo",
  "agent_id": "a1",
  "base_version": 0,
  "idempotency_key": "k-note-a",
  "ops": [{"op": "replace", "path": "/note", "value": "from-planner"}],
  "evidence": {},
  "effect_ids": []
}
```

成功：

```json
{"decision": "commit", "reason_code": "OK", "version": 1, "content_hash": "75a91a7bd97c5fc12577125d862c4886f4e3d43450254a124bcbb5ecb411fdc5"}
```

拒绝：

```json
{"decision": "reject", "reason_code": "OCC", "version": 1}
```

`refresh_and_replan` 且未达上限时，拒绝体另有 `content` 和 `replan_remaining`。`replan_remaining` 等于上限减已用次数。

同一键且 ops 相同：返回第一次的结果，不追加账本，不追加轨迹。

### 工具

`POST /v1/tools/invoke`

```json
{
  "task_id": "t-story",
  "agent_id": "a2",
  "name": "place_order",
  "args": {"doc_id": "ord_demo"},
  "idempotency_key": "pay-1"
}
```

`send_receipt` 可另带 `approval_token`。令牌不是 `args` 的字段。`read_order` 可以不带 `idempotency_key`。

同键第二次 `place_order` 返回第一次的 `result`，支付桩不会被再调用。上面这次调用的 `result.payment_id` 是 `pay_pay-1`。

### 补偿、重放、恢复

`POST /v1/tasks/{task_id}/compensate`

只补偿仍为 `executed` 且注册了补偿工具的效应。成功体为 `reason_code=OK` 和已补偿的 `effect_ids`。失败是 `EFFECT`，该行保持 `executed`，`effect_ids` 只含已经补偿成功的编号。

`POST /v1/tasks/{task_id}/replay`

不改任务模式，不改订单。成功：

```json
{"reason_code": "OK", "steps": 3}
```

失败：

```json
{
  "reason_code": "REPLAY_DIVERGENCE",
  "step_id": 2,
  "kind": "commit",
  "expected_hash": "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
  "actual_hash": "bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb"
}
```

`POST /v1/tasks/{task_id}/resume`

没有检查点则 404。有则订单版本和预算回到序号最大的检查点，删掉 `step_id` 大于等于该游标的轨迹，模式变为 `RESUME`，状态变为 `running`。

### 查询

`GET /v1/documents/{doc_id}`：`doc_id`、`version`、`content`、`content_hash`。

`GET /v1/tasks/{task_id}`：`status`、`mode`、`conflict_policy`、`token_used`、`token_limit`、`tool_calls_used`、`tool_call_limit`、`deadline_at`、`replan_used`、`replan_limit`。

`GET /v1/ledger?doc_id=`：按 `created_at` 升序，相同则 `id` 升序。字段与 `ledger` 表一致。

### MCP

`POST /mcp` 与 HTTP 工具入口使用同一进程内注册表。只实现两个方法。

```json
{"method": "tools/list", "params": {}}
```

列表含四个工具的 `name`、`description` 和参数要求。

```json
{
  "method": "tools/call",
  "params": {
    "name": "read_order",
    "arguments": {"doc_id": "ord_demo"},
    "task_id": "t-story",
    "agent_id": "a1"
  }
}
```

写类工具在 `params` 里带 `idempotency_key`。`send_receipt` 可带 `approval_token`。`tools/call` 的结果与 `POST /v1/tools/invoke` 相同。请求只含 `method` 和 `params`，不包 `jsonrpc` 和 `id`。

创建任务后的订单 `content_hash` 是 `a94041fb96478ed97aec08cc140dc83effbfbbed072dd00c1fba98cdc1ac3347`。演示第 3 步重放时，轨迹只有第 1 步的两笔提交和第 2 步的一笔拒绝，所以 `steps` 为 3。本版不写 `kind=input`。
