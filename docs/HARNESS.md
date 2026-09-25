# ACCK 评测

## 目的

规定三种模式、十个用例、套件文件和报告长什么样。

## 读者

实现 `python -m harness.run` 的人。

## 范围

不新增用例名。`refresh_and_replan` 达到上限不在本套件，而在 `tests/test_occ.py`。

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
