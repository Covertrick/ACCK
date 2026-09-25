# ACCK 演示

## 目的

先跑功能文档第 13 节的协作演示，再给出不经过协作层的六步内核走查，以及每步失败时先查什么。

## 读者

跑 `python demo/story.py` 或自己按下面命令调用 API 的人。

## 范围

协作演示不另加第七个内核步。内核走查的准备任务不算进六步。扣款次数没有查询接口，以脚本打印为准。

## 内容

仓库里还没有可运行代码时，下面的命令会连接失败。API 起来之后基址是 `http://127.0.0.1:8000`。

### 协作演示

```text
python demo/story.py
```

创建任务 `t-story`、文档 `ord_demo`。共同目标、两个角色目标，以及标准结果，与 `docs/HARNESS.md` 的 `confirm_and_pay` 相同。脚本打印账本最后一行、最后一条交接、`payment_charge_count=1`。

`ACCK_LLM=live` 时调用真实模型，并需要 `ACCK_LLM_BASE_URL` 与 `ACCK_LLM_MODEL`。未设置 live 时，按 `demo/fixtures/story_decisions.json` 回放，不调用模型。回放结果应等于标准结果。

失败：最终 `note` 不是「加急」、`total` 不是 128、`status` 不是 `paid` 或 `payment_id` 不是 `pay_pay-1`，说明决定没有落到提交，或提交被拒绝后没有修订。扣款不是 1，说明同键 `place_order` 没有短路。交接改写了订单，说明协作层绕过了 `Kernel.commit`。

下面从「准备」起是内核走查。它不启动协作循环，用来单独看冲突、越权、重放、恢复和同键扣款。`python demo/story.py` 不执行这些 curl。两边都使用 `t-story` 与 `ord_demo`。同一次数据库里只跑其中一条，另一条会在创建任务时得到 422。

### 准备

```text
curl -s -X POST http://127.0.0.1:8000/v1/tasks -H "Content-Type: application/json" -d "{\"task_id\":\"t-story\",\"doc_id\":\"ord_demo\",\"document\":\"order\",\"conflict_policy\":\"abort\",\"replan_limit\":3,\"token_limit\":2000,\"tool_call_limit\":20,\"wall_clock_seconds\":60,\"agents\":[{\"agent_id\":\"a1\",\"role\":\"planner\"},{\"agent_id\":\"a2\",\"role\":\"cashier\"}]}"
```

预期：HTTP 200，`status` 为 `running`，`mode` 为 `LIVE`。`GET /v1/documents/ord_demo` 的 `version` 为 0，`note` 为 `""`，`payment_id` 为 `null`。

失败：422 表示 `ord_demo` 已经有任务，或 `order` / `abort` / 角色未注册。换一个 `doc_id`，或确认进程启动时注册了这些名字。

### 1. 两人改备注

规划者：

```text
curl -s -X POST http://127.0.0.1:8000/v1/commits -H "Content-Type: application/json" -d "{\"task_id\":\"t-story\",\"doc_id\":\"ord_demo\",\"agent_id\":\"a1\",\"base_version\":0,\"idempotency_key\":\"k-note-a\",\"ops\":[{\"op\":\"replace\",\"path\":\"/note\",\"value\":\"from-planner\"}],\"evidence\":{},\"effect_ids\":[]}"
```

收银，仍然基于版本 0：

```text
curl -s -X POST http://127.0.0.1:8000/v1/commits -H "Content-Type: application/json" -d "{\"task_id\":\"t-story\",\"doc_id\":\"ord_demo\",\"agent_id\":\"a2\",\"base_version\":0,\"idempotency_key\":\"k-note-b\",\"ops\":[{\"op\":\"replace\",\"path\":\"/note\",\"value\":\"from-cashier\"}],\"evidence\":{},\"effect_ids\":[]}"
```

预期：第一笔 `decision=commit`、`reason_code=OK`、`version=1`。第二笔 `decision=reject`、`reason_code=OCC`、`version=1`。订单 `note` 仍是 `from-planner`。任务仍是 `running`。账本两行，决定分别是 `commit` 和 `reject`。

失败：第二笔也是 `OK`，说明没有行锁，或第二笔的 `base_version` 用了 1。`note` 变成 `from-cashier` 就是静默覆盖，与 `no_kernel` 同类。

### 2. 规划者写支付单号

```text
curl -s -X POST http://127.0.0.1:8000/v1/commits -H "Content-Type: application/json" -d "{\"task_id\":\"t-story\",\"doc_id\":\"ord_demo\",\"agent_id\":\"a1\",\"base_version\":1,\"idempotency_key\":\"k-pay-path\",\"ops\":[{\"op\":\"replace\",\"path\":\"/payment_id\",\"value\":\"pay_example\"}],\"evidence\":{},\"effect_ids\":[]}"
```

预期：`UNAUTHORIZED_PATH`，`version` 仍是 1，`payment_id` 仍是 `null`。账本追加一行 `reject`。

失败：若返回 `OK`，路径门没有执行。若返回 `OCC`，请求里的 `base_version` 不是 1。

### 3. 严格重放

```text
curl -s -X POST http://127.0.0.1:8000/v1/tasks/t-story/replay
```

预期：`reason_code=OK`，`steps` 为 3。这 3 步都是 `commit`：第 1 步两笔，第 2 步一笔拒绝。不写 `kind=input`。订单版本仍是 1。`GET /v1/tasks/t-story` 的 `mode` 仍是 `LIVE`。

失败：`REPLAY_DIVERGENCE` 表示某步 `input_hash` 与轨迹不一致。看响应里的 `step_id`。本步不应出现分叉。`steps` 不是 3，说明创建任务多写了轨迹，或前两步没有按上面的次数提交。

### 4. 从检查点恢复

```text
curl -s -X POST http://127.0.0.1:8000/v1/tasks/t-story/resume
```

预期：HTTP 200，`version` 仍是 1，`status` 为 `running`，`mode` 为 `RESUME`。

失败：404 表示还没有成功提交，第 1 步的第一笔没有写成 `OK`。版本变成 0 表示恢复到了错误的检查点；本演示只有版本 1 这一次成功提交。

### 5. 同一幂等键下单两次

```text
curl -s -X POST http://127.0.0.1:8000/v1/tools/invoke -H "Content-Type: application/json" -d "{\"task_id\":\"t-story\",\"agent_id\":\"a2\",\"name\":\"place_order\",\"args\":{\"doc_id\":\"ord_demo\"},\"idempotency_key\":\"pay-1\"}"
curl -s -X POST http://127.0.0.1:8000/v1/tools/invoke -H "Content-Type: application/json" -d "{\"task_id\":\"t-story\",\"agent_id\":\"a2\",\"name\":\"place_order\",\"args\":{\"doc_id\":\"ord_demo\"},\"idempotency_key\":\"pay-1\"}"
```

预期：两次的 `reason_code` 都是 `OK`，两次 `result.payment_id` 都是 `pay_pay-1`。订单仍没有 `payment_id`，因为工具不写订单行。脚本再打印 `payment_charge_count=1`。

失败：第二次的 `payment_id` 不是 `pay_pay-1`，或脚本打印的次数不是 1，说明同键没有短路，桩被调用了两次。扣款次数没有查询接口，只看脚本这一行。

### 6. 三种模式的对照表

```text
python -m harness.run --suite harness/suites/order_v1.yaml --mode no_kernel --out experiments/out
python -m harness.run --suite harness/suites/order_v1.yaml --mode contract_only --out experiments/out
python -m harness.run --suite harness/suites/order_v1.yaml --mode full_kernel --out experiments/out
```

预期：`experiments/out/report.no_kernel.json`、`report.contract_only.json`、`report.full_kernel.json`。每行的期望以 `docs/HARNESS.md` 为准。`same_field` 在全内核的 `reason_code` 为 `OCC`，在无内核是静默损坏。

失败：只生成一个 `report.json` 且被下一次命令覆盖，说明没有按模式分开文件名。某一格和 `docs/HARNESS.md` 不一致时，先看该模式有没有误走进 `Kernel.commit`。全内核 p95 大于 200 毫秒时，报告里应有 `latency_note`，用例本身仍可按期望判成功。
