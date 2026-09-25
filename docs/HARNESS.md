# ACCK 评测

## 目的

规定内核套件与 Agent 套件的模式、用例和报告。课程报告以 Agent 套件说明协作是否可靠；内核套件说明写入为什么挡得住。

## 读者

实现 `python -m harness.run` 的人。

## 范围

`order_v1` 不新增用例名，也不进入协作循环。`refresh_and_replan` 达到上限不在本套件，而在 `tests/test_occ.py`。`agent_v1` 只有下面三个用例。

## 内容

```text
python -m harness.run --suite harness/suites/order_v1.yaml --mode full_kernel --out experiments/out
```

`--mode` 为 `no_kernel`、`contract_only` 或 `full_kernel`。一次命令只跑一个模式。报告写到 `experiments/out/report.<mode>.json`。

`success` 表示该行符合本模式的期望，不表示提交成功。`silent_corruption` 仅当最终订单与全内核不同、且本模式没有拒绝码。扣款看 `payment_charge_count`。p50/p95 统计该次全内核运行中全部提交的耗时，含拒绝，单位毫秒。`ModelCascade` 只出现在报告里：无内核且没有拒绝码。

本地 Docker、PostgreSQL 16 上，全内核提交延迟目标是 p50 小于 50 毫秒、p95 小于 200 毫秒。超出不把 `success` 判为失败，在报告里写 `latency_note` 说明原因。

| 模式 | 行为 |
|------|------|
| `no_kernel` | 用 Agent 读到的整份旧订单改一个字段后写回。不进 `Kernel.commit`，不写拒绝账。工具每次都进桩 |
| `contract_only` | 结构、路径、文档约束，并写账本。版本落后仍把 patch 打到当前订单上。不做版本策略、去重、批准、预算、补偿、检查点 |
| `full_kernel` | `docs/FEATURES.md` 的全部门 |

`double_charge` 的四次调用依次是：同键、同键、新键、无键。`disjoint_field` 由规划者写 `/assignee`，收银写 `/note`。

| 用例 | `no_kernel` | `contract_only` | `full_kernel` |
|------|-------------|-----------------|---------------|
| `same_field` | 后写整份覆盖，无拒绝码，静默损坏 | 后写的 `/note` 生效，无 `OCC`，静默损坏 | 策略 `abort`。后者 `OCC`，备注保持先写 |
| `disjoint_field` | 整份覆盖，先写字段丢失 | 两个 patch 都在，无 `OCC` | 策略 `merge_if_disjoint`。两字段都在 |
| `unauthorized` | 支付单号写进订单 | `UNAUTHORIZED_PATH`，订单不变 | 同左 |
| `bad_status` | `draft` 直接变成 `paid` | `SCHEMA`，订单不变 | 同左 |
| `double_charge` | 四次都进桩，扣款 4 次 | 四次都扣，无 `EFFECT` | 同键 1 次，新键 1 次，无键不扣，合计 2 次 |
| `compensate` | 不退款 | 不退款 | 退款一次，效应 `compensated` |
| `budget_tools` | 第二次仍执行 | 第二次仍执行 | `BUDGET`，第二次不执行，任务 `aborted` |
| `replay` | 不适用 | 不适用 | 原样无分叉。篡改第 2 步后停在 `step_id=2` |
| `crash_resume` | 不能从检查点继续 | 恢复为 404 | 版本与检查点一致，检查点前的扣款不重复 |
| `irreversible` | 发送 | 无令牌也发送 | `APPROVAL_REQUIRED`，发送次数 0 |

套件文件：

```yaml
cases:
  - id: same_field
    conflict_policy: abort
    token_limit: 2000
    tool_call_limit: 20
    wall_clock_seconds: 60
    script: []
    steps: []
```

`script` 按调用顺序弹出，元素是模型输出的 JSON。`steps[].kind` 只使用 `commit`、`tool`、`abort`、`crash`、`resume`、`replay`、`tamper`。`commit` 含 `agent_id`、`ops`、`idempotency_key`、`base`（整数或 `stale`）。`tool` 含 `agent_id`、`name`、`args`，以及可选的 `idempotency_key`、`approval_token`。`tamper` 含 `step_id`，用来改该步 `input_hash`。

```json
{
  "rows": [
    {
      "id": "same_field",
      "mode": "full_kernel",
      "success": true,
      "silent_corruption": false,
      "reason_code": "OCC",
      "payment_charge_count": 0
    }
  ],
  "full_kernel_commit_ms": {"p50": 0, "p95": 0},
  "latency_note": "",
  "failures": {"OCC": 1, "ModelCascade": 0}
}
```

非全内核运行时，`full_kernel_commit_ms` 与 `latency_note` 省略。未超出目标时 `latency_note` 为空字符串。

### Agent 套件

```text
python -m harness.run --suite harness/suites/agent_v1.yaml --mode full_kernel --out experiments/out
```

报告写到 `experiments/out/report.agent.<mode>.json`。`ACCK_LLM=script` 时按各用例的 `decisions` 弹出功能文档第 14 节的决定 JSON，供回归。`ACCK_LLM=live` 时同一命令调用真实模型。课程报告里的协作数字来自 live。live 的失败不改 script 回归的期望。

`task_success` 表示最终订单与标准结果一致（`order_id` 等于该用例的 `doc_id`）。`first_illegal_rate` 是被拒绝的提交或工具次数，加上协作层拦住的重复 ops 次数，除以模型提出的 `commit` 与 `tool` 次数。没有这类提议时该率为 0。`repair_success` 表示出现拒绝之后，同一 Agent 在任务中止前有一次被接受的提交或工具。`handoff_mismatch` 表示某次 `handoff` 的 `claims.total` 或 `claims.status` 与当时订单不同。`receipt_send_count` 是 `send_receipt` 真实调用次数。

`confirm_and_pay` 按默认顺序。共同目标是「把订单确认为加急，总价 128，然后收款并出收据」。规划者目标：状态改为 `confirmed`，备注写成「加急」，总价写成 128，然后交接。收银目标：幂等键 `pay-1` 调用 `place_order`，把 `payment_id` 写成 `pay_pay-1`、状态写成 `paid`，再调用 `send_receipt`。标准结果：

```json
{
  "status": "paid",
  "assignee": "",
  "items": [],
  "note": "加急",
  "total": 128,
  "payment_id": "pay_pay-1"
}
```

`note_conflict` 与 `split_fields` 使用 `parallel_first_commit`。前者策略 `abort`：规划者把备注写成 `from-planner`，收银把备注写成 `from-cashier`。后者策略 `merge_if_disjoint`：规划者把 `/assignee` 写成 `ada`，收银把 `/note` 写成 `paid-note`。

| 用例 | `no_kernel` | `contract_only` | `full_kernel` |
|------|-------------|-----------------|---------------|
| `confirm_and_pay` | 不等于标准结果，或扣款大于 1 | 同键多次扣款 | 等于标准结果，扣款 1，收据 1 |
| `note_conflict` | 备注为 `from-cashier`，无拒绝码 | 后写备注生效，无 `OCC` | 备注保持 `from-planner`，收银那笔 `OCC` |
| `split_fields` | `assignee` 丢失 | 两个字段都在，无 `OCC` | `assignee` 为 `ada` 且 `note` 为 `paid-note` |

```json
{
  "rows": [
    {
      "id": "confirm_and_pay",
      "mode": "full_kernel",
      "success": true,
      "task_success": true,
      "first_illegal_rate": 0,
      "repair_success": false,
      "handoff_mismatch": false,
      "payment_charge_count": 1,
      "receipt_send_count": 1
    }
  ]
}
```

`repair_success` 在没有拒绝时为 false。`success` 仍表示该行符合本模式的期望。
