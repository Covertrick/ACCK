# ACCK

**Agent Collaboration Commitment Kernel。** 测量多 Agent 协作是否可靠：模型在外层决定下一步，共享订单只有经过内核校验的 patch 才能写入。

规划者和收银各自读自然语言目标、当前订单和上一轮拒绝，再在读订单、调工具、提交、交接、停止之间选择。模型可以不确定。写进共享订单的结果必须可拒绝、可审计、可重放。`Kernel.commit` 不调用模型，它是共享订单的唯一写入口。

本版做在一份订单上。规划者确认订单、写加急备注和总价；收银收款并出收据。两人拿着同一个旧版本改同一字段时，后写入的提交被拒绝。越权路径、重复扣款、超预算、轨迹被篡改，都在写入前停住。交接句只留在轨迹里，改不了订单。

原则：认知可以非确定；后果提交必须确定。

## 仓库现状

规格版本 **v1.0**，2026-09-25 已确认。本页索引里的文档都是这一版，没有草稿，也没有待定项。

代码还没开始。仓库里没有 `src/`、`tests/`、`docker-compose.yml`。下面「代码落地之后」的命令，要等对应实现写完才能跑。

验收清单在 [`docs/REQUIREMENTS.md`](docs/REQUIREMENTS.md) 第 3 节。做完没有，对照那一节，不要在本页再抄一份。

## 新成员

1. 读完本页，确认这个仓库现在只有规格。
2. 按下面的阅读顺序读文档。行为以 [`docs/FEATURES.md`](docs/FEATURES.md) 为准。文档和以后的代码不一致时，先改功能文档，再改实现文档和代码。
3. 从实现顺序的第一块开始写。前一块的测试没通过，不要开下一块。
4. 改任何文档之前先看 [`docs/CONTRIBUTING.md`](docs/CONTRIBUTING.md)。需求里写明不做的东西，不要加回验收。

## 阅读顺序

| 你要做的事 | 先读 |
|------------|------|
| 刚加入 | 本页 → [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) → [`docs/FEATURES.md`](docs/FEATURES.md) → [`docs/DEVELOPMENT.md`](docs/DEVELOPMENT.md) |
| 写代码 | [`docs/DECISIONS.md`](docs/DECISIONS.md) → [`docs/DATABASE.md`](docs/DATABASE.md) → [`docs/DEVELOPMENT.md`](docs/DEVELOPMENT.md) |
| 写测试 | [`docs/HARNESS.md`](docs/HARNESS.md) → [`docs/DEVELOPMENT.md`](docs/DEVELOPMENT.md) 第 8 节 |
| 改文档 | [`docs/CONTRIBUTING.md`](docs/CONTRIBUTING.md) |
| 讲清楚某项决定为什么这样定 | [`docs/ADR/`](docs/ADR/) 的 001 到 006 → [`docs/DECISIONS.md`](docs/DECISIONS.md) |

## 做什么

- 共享订单经内核校验后才能写入。失败码只有 `SCHEMA`、`UNAUTHORIZED_PATH`、`OCC`、`EFFECT`、`BUDGET`、`APPROVAL_REQUIRED`、`REPLAY_DIVERGENCE`、`IDEMPOTENCY_MISMATCH`、`TASK_ABORTED`。成功是 `OK`。
- 检查顺序固定为 11 步。预算和副作用引用先于版本比较。插件只能替换门后的实现，不能重排检查，也不能自己 `UPDATE` 订单。
- 内置一块订单文档、四个工具（`read_order`、`place_order`、`refund`、`send_receipt`）、三种冲突策略（`abort`、`refresh_and_replan`、`merge_if_disjoint`）、一个协作层、一个 LangGraph 薄适配器、进程内 MCP（`tools/list`、`tools/call`）。
- 内核评测 `order_v1` 用脚本步骤对照三种模式。Agent 评测 `agent_v1` 用同一套协作循环对照这三种模式，看任务是否成功、非法提议、拒绝后能否修订、交接是否与订单一致。

## 不做什么

自研编排框架；Kubernetes、gVisor、Kata、Helm；Go sidecar；Redis、Kafka、Dapr、Service Mesh；完整 A2A；OAuth；通用监控大盘；可重排的检查；Soft 评分门；效应状态 `rejected`；把任务标成 `completed`。

这些已经在需求里排除。实现时不要补回来。

## 技术栈

Python 3.11+、FastAPI、Pydantic v2、PostgreSQL 16、OpenTelemetry（内存 exporter）。API 端口 8000，Postgres 端口 5432。进程入口是 `uvicorn acck.api.app:app`。

`kernel` 包不引用 LangGraph、FastAPI、MCP、`tools/` 和 `collab`。真实模型使用 OpenAI 兼容接口，由 `ACCK_LLM_BASE_URL` 和 `ACCK_LLM_MODEL` 指定。

## 实现顺序

前一步测试未通过，不开始下一步。

1. 骨架与注册表。同名插件第二次注册必须失败。
2. 提交、路径权限、幂等。
3. 三种冲突策略。
4. 工具与补偿。
5. 预算。
6. 轨迹、STRICT 重放、从检查点恢复。
7. HTTP。
8. OpenTelemetry。提交账本和工具效应上的 `trace_id` 要对上对应 span。
9. 协作循环。模型决定读、工具、提交、交接或停止；拒绝后由模型修订。
10. LangGraph 适配器与 `demo/story.py`。图只按角色启动协作循环。
11. MCP。`tools/list` 返回全部四个工具。
12. 内核评测 `order_v1`。
13. Agent 评测 `agent_v1`。报告与 [`docs/HARNESS.md`](docs/HARNESS.md) 一致。

各步对应的测试文件在 [`docs/DEVELOPMENT.md`](docs/DEVELOPMENT.md) 第 8 节。

## 文档

| 文档 | 读它来 |
|------|--------|
| [`docs/REQUIREMENTS.md`](docs/REQUIREMENTS.md) | 看范围、目标和验收 |
| [`docs/FEATURES.md`](docs/FEATURES.md) | 看行为。这是规格正文 |
| [`docs/DEVELOPMENT.md`](docs/DEVELOPMENT.md) | 看目录、插件、提交步骤和测试文件 |
| [`docs/DATABASE.md`](docs/DATABASE.md) | 看表、索引、事务和初始化 |
| [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) | 看组件和提交顺序 |
| [`docs/DECISIONS.md`](docs/DECISIONS.md) | 看已定决定，以及它们和 ADR 的对应 |
| [`docs/API.md`](docs/API.md) | 看 REST 和 MCP |
| [`docs/openapi.yaml`](docs/openapi.yaml) | 给生成客户端和校验工具读 |
| [`docs/HARNESS.md`](docs/HARNESS.md) | 看内核十例和 Agent 三例 |
| [`docs/DEMO.md`](docs/DEMO.md) | 看协作演示和内核走查 |
| [`docs/GLOSSARY.md`](docs/GLOSSARY.md) | 查术语 |
| [`docs/CONTRIBUTING.md`](docs/CONTRIBUTING.md) | 看改哪一份文档 |
| [`docs/ADR/`](docs/ADR/) | 看六项决定的理由 |

## 代码落地之后

基址是 `http://127.0.0.1:8000`。Postgres 在本机 `5432`。空库用 `src/acck/store/schema.sql` 初始化一次，本版不使用 Alembic。

```text
python -m unittest discover -s tests
python demo/story.py
python -m harness.run --suite harness/suites/order_v1.yaml --mode full_kernel --out experiments/out
python -m harness.run --suite harness/suites/agent_v1.yaml --mode full_kernel --out experiments/out
```

协作演示和内核走查的命令、预期在 [`docs/DEMO.md`](docs/DEMO.md)。另外两种模式把 `--mode` 换成 `no_kernel` 或 `contract_only`。没有设置 `ACCK_LLM=live` 时，演示回放 `demo/fixtures/story_decisions.json`。

本地 Docker、PostgreSQL 16 上，提交延迟目标是 p50 小于 50 毫秒、p95 小于 200 毫秒（含被拒绝的提交）。超出只写入报告里的 `latency_note`，不把用例判失败。

## 一起改这个仓库

行为只改 [`docs/FEATURES.md`](docs/FEATURES.md)。实现步骤和测试文件只改 [`docs/DEVELOPMENT.md`](docs/DEVELOPMENT.md)。表、索引和事务只改 [`docs/DATABASE.md`](docs/DATABASE.md)。用例期望只改 [`docs/HARNESS.md`](docs/HARNESS.md)。接口示例同时改 [`docs/API.md`](docs/API.md) 和 [`docs/openapi.yaml`](docs/openapi.yaml)。已经接受的理由写进 [`docs/ADR/`](docs/ADR/)，并在 [`docs/DECISIONS.md`](docs/DECISIONS.md) 留一行。

协作开发者由仓库所有者在 GitHub 的 Collaborators 里邀请。本仓库不规定分支名和评审人数。
