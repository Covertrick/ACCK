# ACCK — Agent Collaboration Commitment Kernel

| 字段 | 内容 |
|------|------|
| 定位 | 可挂到现有编排框架上的协作提交内核 |
| 文档版本 | v1.0 |
| 更新日期 | 2026-09-25 |
| 功能 | `docs/FEATURES.md` |
| 实现 | `docs/DEVELOPMENT.md` |
| 数据 | `docs/DATABASE.md` |
| 决定 | `docs/DECISIONS.md` |

原则：认知可以非确定；后果提交必须确定、可拒绝、可审计、可重放。

行为在 `docs/FEATURES.md`。实现步骤与测试文件在 `docs/DEVELOPMENT.md`。表和事务在 `docs/DATABASE.md`。用例期望在 `docs/HARNESS.md`。已定决定在 `docs/DECISIONS.md`。

---

## 1. 范围

### 要做

共享订单只有经内核校验后才能写入。内核失败码只有：`SCHEMA`、`UNAUTHORIZED_PATH`、`OCC`、`EFFECT`、`BUDGET`、`APPROVAL_REQUIRED`、`REPLAY_DIVERGENCE`、`IDEMPOTENCY_MISMATCH`、`TASK_ABORTED`。成功为 `OK`。`ModelCascade` 只出现在评测报告里，不是内核失败码。

检查顺序以功能文档「提交」一节为准。合约、版本、效应、预算都要做，但预算和效应引用先于版本比较。流水线不能重排，也不能绕过 `Kernel.commit` 写文档。

扩展点用进程内注册表接入：

| 扩展点 | 本版内置 |
|--------|----------|
| 工具 | `read_order`、`place_order`、`refund`、`send_receipt` |
| 冲突策略 | `abort`、`refresh_and_replan`、`merge_if_disjoint` |
| 文档约束 | `order` |
| 编排适配器 | LangGraph |
| 模型客户端 | 脚本模型；`ACCK_LLM=live` 时换真实模型 |

### 不做

自研编排框架；K8s Agent Sandbox、gVisor/Kata、Helm；Go sidecar；Redis、Kafka、Dapr、Service Mesh；完整 A2A；OAuth；通用监控大盘；可重排的门；Soft 评分门；效应状态 `rejected`；把任务标成 `completed`。

---

## 2. 目标

| # | 目标 | 达到 |
|---|------|------|
| 1 | 可约束 | 越权或非法文档不能进入共享状态 |
| 2 | 可仲裁 | `abort`、`refresh_and_replan`、`merge_if_disjoint` 都有测试 |
| 3 | 可治理 | 四类副作用生效；同键不二次扣款；未入账扣款可退 |
| 4 | 可复现 | STRICT 重放可定位第一处分叉 |
| 5 | 可恢复 | 从最后一次成功 commit 的 checkpoint 继续 |
| 6 | 可证明 | 三种模式对照出表 |
| 7 | 可替换 | 换工具、策略、文档约束、适配器、模型时不改提交事务 |

---

## 3. 验收

- [ ] `no_kernel` 的 `same_field` 是整份覆盖且无拒绝码；`full_kernel` 下后者为 `OCC`，备注保持先写
- [ ] `disjoint_field` 在 `no_kernel` 丢掉先写字段，在 `full_kernel` 两字段都在
- [ ] `abort` 与 `merge_if_disjoint` 出现在 `order_v1`；`refresh_and_replan` 达到上限由 `tests/test_occ.py` 覆盖
- [ ] 同键 `place_order` 在全内核只扣一次；未入账扣款在任务 `aborted` 后退一次
- [ ] STRICT 原样重放不分叉；篡改第 2 步停在 `step_id=2`
- [ ] 崩溃后 resume，checkpoint 前的扣款不重复；没有 checkpoint 时为 404
- [ ] 三种模式的报告与 `docs/HARNESS.md` 一致，外加全内核 commit 延迟 p50/p95（含拒绝，单位毫秒）
- [ ] 新工具、新冲突策略通过注册接入；`commit.py` 不出现订单字段判断
- [ ] MCP `tools/list` 返回全部四个工具；`tools/call` 与 HTTP 走同一进程内注册表
- [ ] 提交账本和工具效应上的 `trace_id` 能对上对应 span
